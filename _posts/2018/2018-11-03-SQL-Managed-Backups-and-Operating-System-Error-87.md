---
layout: post
title: SQL Managed Backups and Operating System Error 87
share-img: https://tjaddison.com/assets/2018/2018-11-03/SQLManagedBackup.png
tags: [SQL, Azure]
---

We use [SQL Managed backups] for our on-premises SQL Servers, and have been very impressed with it (from a speed, management, and cost perspective).  Shortly after deploying the solution though the SQL error logs started to log errors when attempting to read managed backups:

```
BackupIoRequest::ReportIoError: read failure on backup device 'https://allthebackups.blob.core.windows.net/ServerOne/DatabaseOne_ac7caecc997f4cf192d3befba2ed00c4_11b995a1cb9a49d4a40d2bc2f181e17a_20180202100928+00.log'. Operating system error 87(The parameter is incorrect.).
```

Nothing suggested there were any issues - backups were still being taken, our backup chain wasn't broken (restores were fine) - but this error was being logged all the time.

Understanding where the error came from and how to fix it required a better understanding of exactly how managed backup works.
<!--more-->

## How does managed backup know what backups are available?

This doesn't appear to be documented anywhere, but we were able (through reasoning and monitoring) to establish the following set of operations happens to maintain the list of 'what backups are in Azure':

- Managed backup gets a list of files in the target container
- Managed backup performs a [restore headeronly] on each file, to get metadata about that file
- Managed backup uses this information to determine what it needs to do (delete old backups, create a new full backup, create a transaction log backup)

## Why do we get operating system error 87?

After downloading the file and attempting to restore it, we realised the backup was corrupt.  The file size tipped us off to the fact this was probably a partially complete backup.  We were able to reproduce this by killing an in-flight backup request, which generated a partial (and corrupt) backup.

Managed backup will only delete backups that are outside of the retention period, which it can only determine by using the information from the `restore headeronly` command.  Because the backup was corrupt, it would never be eligible for deletion.

## How do I fix it?

The fix is to delete the backup file from Azure storage.  We currently do this manually (as it happens so rarely) but it could be automated to react to the message in the SQL error log.  Looking forward something like [automated storage lifecycle management] would allow us to set a policy to automatically delete blobs that exceed our maximum retention period.

The root cause of the corrupt backup for us was a (planned!) failover.  It's also feasible a network blip could terminate an in-progress backup, or perhaps good old-fashioned storage corruption (which is something we really would care about!).

Outside of some rather sparse documentation (if we'd known how managed backup worked we'd have figured this out a lot faster!) managed backup is something we've been really impressed with - so if you're hitting this scenario hopefully you now know how to fix it.

>A big thanks to my colleague Jose for getting to the bottom of this one

[SQL managed backups]: https://docs.microsoft.com/en-us/sql/relational-databases/backup-restore/sql-server-managed-backup-to-microsoft-azure
[restore headeronly]: https://docs.microsoft.com/en-us/sql/t-sql/statements/restore-statements-headeronly-transact-sql
[automated storage lifecycle management]: https://docs.microsoft.com/en-us/azure/storage/common/storage-lifecycle-managment-concepts