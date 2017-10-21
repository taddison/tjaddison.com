---
layout: post
title: Troubleshooting an SSRS slowdown - portal delays, PREEMPTIVE_OS_LOOKUPACCOUNTSID, and bad plans
share-img: http://tjaddison.com/assets/2017-10-21/BadPlan.png
---
_This tale takes place on an SSRS 2016 Enterprise instance running on Server 2012 R2._

Users had reported that the SSRS instance was 'slow', and after opening the portal I could see what they meant.  In addition to the home page taking a very long time to load (minutes), sometimes after loading I'd be presented with a menu bar and no folders/reports (despite knowing I should have seen a bunch of folders and reports).

![Loading](/assets/2017-10-21/Loading.png)

After quickly querying the execution catalog to see if any reports were running at all (as well as interactive rendering we have subscriptions and rendering happening via the API).

```sql
select		top 100 el.TimeStart
			,el.ItemAction
			,el.ItemPath
			,el.Status
			,el.*
from		ReportServer.dbo.ExecutionLog3 as el
order by	el.TimeStart desc
```
This confirmed that subscriptions were working, and some users were using still browsing & running reports interactively.  Knowing that the whole instance wasn't broken I was able to proceed with troubleshooting in a slightly calmer manner.

<!--more-->
## Looking into the portal issues

To summarise the issue after a few minutes of checks:
	- The portal is very slow to load (no errors observed in the browser console)
	- Sometimes after the portal loads there are no reports/folders
	- Looking at user/folder settings (e.g. security) everything appears normal (permissions are intact)
	- The report API is unaffected (can browse folders + render reports quickly and without any issues)
	- The servers are not under high load (not capped on CPU/Memory/No space left)
	- No configuration changes had been made recently

The portal looked to be the misbehaving piece, so the next stop was the error logs.  You can read more information about these logs online ([Report Server Service Trace Log](https://docs.microsoft.com/en-us/sql/reporting-services/report-server/report-server-service-trace-log)) - and in our case we're after the portal log file (Microsoft.ReportingServices.Portal.WebHost).  Browsing these logs is always a little painful - someone somewhere has always left a misconfigured report, a report that belongs to someone who is no longer with the company - and shame on me we don't put these logs somewhere centrally queryable and so dealing with those issues tends to come in response to problems only.

Ignoring all of the alerts which can get fixed later, there were some unfamiliar alerts all of which have identical stack traces.  The final error message was:

***An error occurred within the report server database.  This may be due to a connection failure, timeout or low disk condition within the database.***

Which was immediately preceded by:

***System.Data.SqlClient.SqlException: Timeout expired.  The timeout period elapsed prior to completion of the operation or the server is not responding.***

This tells us our next stop should be the ReportServer database.  The call stack of the error indicated it was part of GetFavoriteItems - which is why only the portal was having issues and not the API/reports themselves.

## ReportServer database

In this case the diagnosis was pretty fast.

```sql
exec sp_whoisactive @get_plans = 1;
```

A few queries running for over a minute, with a wait type of [PREEMPTIVE_OS_LOOKUPACCOUNTSID](https://www.sqlskills.com/help/waits/preemptive_os_lookupaccountsid/).  The query in question was ***dbo.GetAllFavoriteItems***, and the offending statement was:

```sql
SELECT 
    C.Type,
    C.PolicyID,
    SD.NtSecDescPrimary,
    C.Name, 
    C.Path, 
    C.ItemID,
    DATALENGTH( C.Content ) AS [Size],
    C.Description,
    C.CreationDate, 
    C.ModifiedDate,
    SUSER_SNAME(CU.Sid), 
    CU.[UserName],
    SUSER_SNAME(MU.Sid),
    MU.[UserName],
    C.MimeType,
    C.ExecutionTime,
    C.Hidden, 
    C.SubType,
    C.ComponentID,
    CAST(1 AS bit)
FROM
   Catalog AS C 
   INNER JOIN [dbo].[Catalog] AS P ON C.ParentID = P.ItemID
   INNER JOIN [dbo].[Users] AS CU ON C.CreatedByID = CU.UserID
   INNER JOIN [dbo].[Users] AS MU ON C.ModifiedByID = MU.UserID
   INNER JOIN [dbo].[Favorites] F ON C.ItemID = F.ItemID AND F.UserID = @UserID
   LEFT OUTER JOIN [dbo].[SecData] SD ON C.PolicyID = SD.PolicyID AND SD.AuthType = @AuthType
```

The wait type is coming from the calls into [SUSER_SNAME](https://docs.microsoft.com/en-us/sql/t-sql/functions/suser-sname-transact-sql).  What this function does is takes the Sid and asks the local machine for the user name associated with it (by calling the [LsaLookupSid](https://technet.microsoft.com/en-us/library/ff428139(v=ws.10).aspx#BKMK_LsaLookupSIDs) function).  If the mapping between Sid and user name isn't on the local machine, the machine will then ask the domain controller for the mapping.

Running the following query took over 2 minutes:

```sql
select suser_sname(u.Sid)
from dbo.Users as u
```

The only wait type observed was PREEMPTIVE_OS_LOOKUPACCOUNTSID, and each wait was between 70 and 150 milliseconds.  In this case there are quite a few users with credentials in a domain where the DC is located on a different continent (and the wait times correlate with the round-trip time from server to DC).

At this point I started looking at things like caching of the lookups (and it turns out there is a cache but only for 128 entries, and given we have 1000s of user records the cache will constantly get cycled if we scan the table).  I was about to start working on the registry to update the setting (after learning about the post from [this SO question](https://stackoverflow.com/questions/31969101/ssrs-users-table), [this support article](https://support.microsoft.com/en-us/help/946358/the-lsalookupsids-function-may-return-the-old-user-name-instead-of-the), and [these docs](https://msdn.microsoft.com/en-us/library/ms721799.aspx)), when I realised that the DC hadn't moved overnight and the report server had been working fine up until today.

## Plan regression

The problem (and the eventual solution) was the query plan.  Looking again at the query it was clear we were pulling all favourites for a single user, so we shouldn't be scanning the user table and looking up all the account name for every user.

The current plan was scanning the user table twice, and for every row coming from the user table it was evaluating the suser_sname function for every row (represented by the compute scalars in the image below).

![Bad Plan](/assets/2017-10-21/BadPlan.png)

Looking in the table which holds user favourites I could see there was only a small number of rows, and so it seemed likely there was a better plan (especially as the portal had been performing well until recently, and there was no user who had favourite'd every report on the server…).

After running the procedure with a few users (including the user who had the most favourites) and only observing good plans I recompiled the procedure, and usual service was instantly returned (the query that had been running for minutes and timing out now finished in less than a second).  The plan is as expected - get the favourites and then perform the Sid lookups only the users who own one of the favourite'd reports.

![Good Plan](/assets/2017-10-21/GoodPlan.png)

What caused the plan to tip in the first place and when I don't know, as our ReportServer database didn't have one of the best features of 2016 enabled at the time (querystore).  That has now been rectified, and going forward the data about execution times/timeouts will now feed into our querystore aggregation system and make spotting this kind of issue easy.

## Bonus: Missing users

One additional issue we were facing is what happens when a Sid mapping isn't found - rather than cache the miss nothing is cached.  Some of our reports have been around for a long time and their authors are no longer with the company, but SSRS still faithfully attempts to resolve the username of the report creator, sends the request on to the remote DC, and gets nothing back (the suser_sname function returns null).

This is easy to test - assume you've got two users (Tim and Tom).  Tim still exists, but Tom doesn't.  We're querying a remote DC for these.  To test yourself you'll need to get the Sid of an expired user account - querying the Users table on an old ReportServer database might be the easiest way (the table also stores the account name, which is how SSRS is able to display information even if the Sid lookup returns null).

```sql
declare @timSid varbinary=0x00…
declare @tomSid varbinary=0x00…

select suser_sname(@timSid); -- takes 90ms
select suser_sname(@timSid); -- takes 0ms - cached

select suser_sname(@tomSid); -- takes 90ms
select suser_sname(@tomSid); -- takes 90ms - not cached
```

The Sid contains information about which domain the user belongs to - if you test with users removed from local domains you'll probably still be able to observe the result though the delta is much smaller (as LsaLookupSid is cached on the machine that makes the query - in this case our SQL Server).

Based on this, my recommendation is that you ensure the owner and 'lasted updated by' users of all reports are set to a user that currently exists, preferably in a local domain.  Clearing old users out of the User table is something else you can do (so at least if the plan does tip you are minimising the impact).