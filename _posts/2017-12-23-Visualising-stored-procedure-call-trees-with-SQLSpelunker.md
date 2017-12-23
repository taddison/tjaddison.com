---
layout: post
title: Visualising stored procedure call trees with SQLSpelunker
share-img: http://tjaddison.com/assets/2017-12-23/StoredProcedureParser.png
---
Whether debugging a problem in an existing system or planning out some changes, knowing how the call graph for a bit of SQL looks has been invaluable in quickly understanding the domain.  If you've worked in any systems where a lot of logic ends up in the code you'll probably be nodding your head and remembering that procedure which started simple and then went down the rabbit hole (the longest I've looked at so far branches out from 1 into ~100 procedures).

Taking inspiration from [What's in the box? Validating SQL Scripts with Powershell](http://port1433.com/2017/12/04/whats-in-the-box-validating-sql-server-scripts-with-powershell/) and [Get Started with the ScriptDom](https://the.agilesql.club/blog/Ed-Elliott/2015-11-07/Get-Started-With-The-ScriptDom) I built [SQLSpelunker](https://www.github.com/taddison/SQLSpelunker) to allow you to go from a stored procedure name to a visual of the call tree for that procedure:

```code
c:\SQLSpelunker>dotnet run "server=localhost;initial catalog=tempdb;integrated security=SSPI" "exec dbo.ProcOne;"

tempdb.dbo.ProcOne
-tempdb.dbo.ProcTwo
--tempdb.dbo.ProcThree
```

You can get started right away by grabbing the [source from Github](https://github.com/taddison/SQLSpelunker) or download the latest build from the [releases page](https://github.com/taddison/SQLSpelunker/releases).Note that to build or run the code you'll need the [.Net Core 2.0 SDK](https://www.microsoft.com/net/download).

Read on for more details about what is currently supported, as well as details of how it works under the hood.

<!--more-->

## Features

### Visualising simple call trees

Given the following schema:

```sql
use tempdb
go
create procedure dbo.ProcOne
as
exec dbo.ProcTwo;
go
create procedure dbo.ProcTwo
as
exec dbo.ProcThree;
go
create procedure dbo.ProcThree
as
select @@servername as ServerName;
go
```

Running SQLSpelunker against that database and the script `exec dbo.ProcOne` will generate the following output:

```code
tempdb.dbo.ProcOne
-tempdb.dbo.ProcTwo
--tempdb.dbo.ProcThree
```

ProcOne calls ProcTwo calls ProcThree.  Fairly straightforward.

### Infinite loop detection

The current implementation will report _every procedure_ called in the code, even if that procedure isn't reachable (e.g. hidden behind an always-false statement).  

While walking the procedure tree if a procedure is encountered that exists anywhere as a parent, the tree is stopped.  This will catch both simple cases (ProcA calls ProcA) as well as multi-step infinite loops (ProcA calls ProcB calls ProcA).

If we now add one more procedure to our schema:

```sql
create procedure dbo.NeverCallsItself
as
exec dbo.NeverCallsItself;
exec dbo.ProcOne;
go
```

Running SQLSpelunker against the same database with the script `exec dbo.NeverCallsItself` will return the following:

```code
tempdb.dbo.NeverCallsItself
-tempdb.dbo.NeverCallsItself [*]
-tempdb.dbo.ProcOne
--tempdb.dbo.ProcTwo
---tempdb.dbo.ProcThree
```

The asterisk (`[*]`) signifies the procedure has previously been called by a parent procedure, and SQLSpelunker will stop trying to walk the call tree.

### Default schema

If a procedure is executed without a schema name then it will be assumed that the default schema of the calling user is dbo.

### Cross-database calls

The procedures can span multiple databases.  Given the following new procedures:

```sql
use tempdb
go
create procedure dbo.CrossDBCall
as
exec master.dbo.ILiveInMaster;
go
use master
go
create procedure dbo.ILiveInMaster;
as
exec tempdb.dbo.CrossDBCall
go
```

We'd see the following call chain:

```code
tempdb.dbo.CrossDBCall
-master.dbo.ILiveInMaster
--tempdb.dbo.CrossDBCall [*]
```

## Limitations

### Exec @procedure

Only stored procedures executed by identifier are supported.  If your statement contains any SQL in the following form it won't be reported:

```sql
declare @procName = 'dbo.WontBeSeen';
exec @procName;
```

### Every procedure is output

The script is parsed but not executed - if there is unreachable code it will still be reported.  In most cases runtime values determine which codepath is executed, so this is unlikely to be a major issue (why would you have compile-time unreachable code hanging around?)


```sql
create procedure dbo.ThisMaybeDoesNothing
as
if 1 = 0
begin
    /* Never reachable */
    exec dbo.ProcOne;
end

if rand() < 0.1
begin
    /* Sometimes reachable */
    exec dbo.ProcTwo;
end
go
```

This produces:

```code
tempdb.dbo.ThisMaybeDoesNothing
-dbo.ProcOne
-dbo.ProcTwo
```

### Missing procedures are still reported

When looking up the definition of a procedure if the definition can't be retrieved the call will still be reported.  This is a semi-feature (some procs which do exist won't have their definition reported, one example being msdb.dbo.sp_send_dbmail).  In the future 'missing' procs may get an annotation.

## Usage

The command line arguments for the SQLSpelunker console app are:

- A connection string to the initial database (which must contain a database)
- The SQL fragment that should be parsed and walked

An example connection string that connects to tempdb on the localhost instance with windows authentication is:

```code
server=localhost;initial catalog=tempdb;integrated security=SSPI
```

## How it works

The flow of the program looks something like this:

- Parse the SQL fragment using ScriptDOM and extract all stored procedure calls
- For each stored procedure call get the definition if it isn't already in cache
- For each stored procedure parse the definition using ScriptDOM and extract all stored procedure calls
- Repeat until there are no more unique calls (e.g. also break on infinite loops)

SQL fragments are parsed using TransactSql ScriptDom - the [official docs](https://msdn.microsoft.com/en-us/library/microsoft.sqlserver.transactsql.scriptdom.aspx?f=255&MSPPError=-2147217396) are a bit barebones, so I'd suggest starting with [Ed Elliot's excellent blog category](https://the.agilesql.club/taxonomy/term/26).

Stored procedure definitions are extracted using the [object_definition](https://docs.microsoft.com/en-us/sql/t-sql/functions/object-definition-transact-sql) function.

## The future of SQLSpelunker

I was pleasantly surprised at how little code was needed to solve the core problem of getting the definition and parsing out additional cores.  The console output is helpful but not particularly easy to use (having to start a demo with 'just download .net core, clone the repo, and then run this - no there isn't just an exe you can download...').  A web version or powershell cmdlet are ideas I'm playing with.

The other feature which would be great is 'what calls this procedure?'.  Obviously that is a very different challenge to walking a tree (you're forced to go an get every procedure - potentially from every database! When you've got thousands of procedures that starts to take time, and keeping it snappy/interactive was a goal, and one of the reasons I didn't just recommend some of the existing tools out there for exploring schema).

## The longest proc I've seen so far

Just for fun - curious to see what else is out there!

![Namespace configuration](/assets/2017-12-23/ARatherLongProcedure.png)

