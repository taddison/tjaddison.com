---
layout: post
title: Adding caching to your PowerShell scripts
share-img: https://tjaddison.com/assets/2018/2018-12-24/GetCachedScriptBlockResults.png
tags: [PowerShell]
---
Most of the time when some part of a script takes a long time to run and you want to re-use the result you'll store it in a variable:

```powershell
$databases = Get-ListOfDatabases # assume this is an expensive/long-running query
Invoke-SomeOperation -Server serverOne -Databases $databases
Invoke-SomeOtherOperation -Server serverOne -Databases $databases
```

As long as the function you're calling allows you to pass in the 'expensive' argument you're fine - but what about cases when the computation takes place inside the function:

```powershell
Invoke-SomeOperation -Server serverOne # internally calls Get-ListOfDatabases
Invoke-SomeOtherOperation -Server serverOne # internally calls Get-ListOfDatabases
```

Sometimes it makes sense to rework the script to accept the argument, though other times it can be cleaner to modify the call site to _cache_ the result, rather than changing every function to accept and potentially pass through that argument.

The rest of this post will go through a specific example that motivated caching, share a generic function that implements scriptblock-based caching, and call out a few and gotchas.

<!--more-->

## The Problem

The [SQLChecks][SQLChecks Repo] contains a set of [Pester][Pester Repo] tests that are designed to compare an expected configuration against actual configuration.  For all the database level tests the first thing each invocation does is to get a list of the databases to check.  This check can end up being very slow (tens of seconds) when the server has a lot of availability group databases.

The typical invocation looks something like this (pseudocode):

```powershell
foreach ($config in Get-SqlChecksConfigs -Path "...") {
    foreach ($check in Get-SqlChecks $config) {
        Invoke-Pester -Check $check -Config $config
    }
}
```

>This pattern is used to enable test results to be sent to Log Analytics as each test completes, rather than waiting for the whole batch to finish.  If we weren't looping through tests each Pester test file would only be called once, and the performance issue wouldn't manifest.

For each config-check combination we'll end up calling the `Get-DatabasesToCheck` function once.  The result won't change between checks, so without caching we can end up spending a lot of time waiting to build the list of databases, scaling linearly as we add more checks.

The wrapper code to run the tests is fairly generic, and supports running tests against the server, the SQL Instance, or databases (and potentially other facets in the future).  To keep the wrapper code generic _and_ make things performant, I added caching inside the `Get-DatabasesToCheck` function.

## Adding Caching

I wanted to keep the callsite changes small (so that any tests against the function `Get-DatabasesToCheck` were testing that function, not the caching logic).  The changes in the function itself are minimal:

```powershell
# before
$queryResults = Invoke-Sqlcmd -ServerInstance $serverInstance -query $query

# after
$queryResults = Get-CachedScriptBlockResult -Key $serverInstance -ScriptBlock {
    Invoke-Sqlcmd -ServerInstance $serverInstance -query $query
}
```

And obviously we need to write the `Get-CachedScriptBlockResult` function:

```powershell
Function Get-CachedScriptBlockResult {
    [cmdletbinding()]
    Param(
        [Parameter(Mandatory = $true)]
        $Key,

        [Parameter(Mandatory = $true)]
        [ScriptBlock]
        $ScriptBlock
    )

    $CACHE_VARIABLE_NAME = "SQLChecks_Cache"

    if (-not (Get-Variable -Name $CACHE_VARIABLE_NAME -Scope Global -ErrorAction SilentlyContinue)) {
        Set-Variable -Name $CACHE_VARIABLE_NAME -Scope Global -Value @{}
    }

    $cache = Get-Variable -Name $CACHE_VARIABLE_NAME -Scope Global
    if (-not $cache.Value.ContainsKey($Key)) {
        $cachedValue = &$ScriptBlock
        $cache.Value[$Key] = $cachedValue
    }
    else {
        $cachedValue = $cache.Value[$Key]
    }

    $cachedValue
}
```

>Note that a better name for this function might have been GetOrAdd-ScriptBlockResult - though that isn't an approved verb, so I went with 'Get'.

We use a global variable that holds a hashtable to act as our cache.  The cache key is a string, and the value can be any object.  The flow of the function is:

- Check to see if the global variable exists, if not create it
- Check to see if the cache key exists
  - If it doesn't exist then store the result of invoking the script block in the key
  - If it does exist, return the value in the cache

### Aside: Why ScriptBlock vs. Objects?

The reason we use a `ScriptBlock` rather than an object is to keep the callsite as neat as possible.  If we'd have elected to make the cache store objects then the interaction would have looked like this:

```powershell
# example of using an object cache instead
$databases = Get-CachedValue -Key $serverInstance
if(-not $databases) {
    $databases = Invoke-SqlCmd -ServerInstance $serverInstance -Query $query
    Add-CachedValue -Key $serverInstance -Value $databases
}
```

A tricky edge case here is what if the value cached is _supposed_ to be null?  We could change our `Get-CachedValue` function to return a success flag and instead pass a reference to an object we want our cache to populate.  This, combined with the fact I wanted to make the caching as easy to add (and not require any logic changes) meant `ScriptBlock` was the winner.

## Testing the Cache

To ensure the cache worked as expected I added a few tests of the cache function.  Although basic, these did catch some edge cases and help build the list of caveats.  The below tests leverage Pester's [mocking functionality][Pester Mocking], which allow us to test that the `ScriptBlock` is invoked only once despite repeated calls.

You can view the full set of tests (which cover basic functionality and null caching) [on GitHub][Cache Tests].

```powershell
Function Get-ExpensiveToComputeValue {
    "Non-mocked"
}

Describe "Get-CachedScriptBlockResult" {
    Context "calls script block the correct number of times " {
        BeforeAll {
            Remove-SQLChecksCache
        }

        Mock Get-ExpensiveToComputeValue { return "mocked" }

        Get-CachedScriptBlockResult -Key "test" -ScriptBlock { Get-ExpensiveToComputeValue }
    
        It "calls the function once to populate the cache" {
            Assert-MockCalled -CommandName Get-ExpensiveToComputeValue -Times 1
        }

        Get-CachedScriptBlockResult -Key "test" -ScriptBlock { Get-ExpensiveToComputeValue }
    
        It "doesnt call the function when the value is in the cache" {
            Assert-MockCalled -CommandName Get-ExpensiveToComputeValue -Exactly -Times 1
        }
    }
}
```

## Conclusion and Caveats

As with any piece of code the most important thing is correctness (and then somewhere behind that come performance and maintainability).  If adding caching breaks correctness, you definitely have a problem!  If there is no performance problem then avoid caching, as it will invariably introduce some failure condition.

Once you're sure caching is for you then take some time to understand the [scope of global variables][PowerShell Scopes], and how much is sharing your cache variable's name (perhaps avoid calling it something like `$cache` and making it global!).

There are a few gotchas when it comes to caching and you'll definitely have issues if you expect caching to work via remoting or in a job (serialization of scriptblocks to be passed to jobs/remote sessions won't work).  You will also need to take care with any Pester tests you have that might need the cache resetting between examples (you'll notice above the call to `Remove-SQLChecksCache` - this clears the global variable).

However, with all that said - if you do have a good fit for caching the results can be spectacular.  After implementing the cache our daily Pester run came down from 20 minutes to less than 60 seconds.  That benefit only grows as more tests are added, and so judiciously applied this technique is a boon.

One ancillary benefit you get is the ability to inspect certain state (the cache is just a variable, take a look inside and see what your scripts are up to!).  This is probably not an ideal debugging technique, but it is worth remembering that this isn't magic, it's just a variable.

[SQLChecks Repo]: https://github.com/taddison/SQLChecks
[Pester Repo]: https://github.com/pester/Pester
[Pester Mocking]: https://github.com/pester/Pester/wiki/Mocking-with-Pester
[Cache Tests]: https://github.com/taddison/SQLChecks/blob/master/tests/Get-CachedScriptBlockResult.tests.ps1
[PowerShell Scopes]: https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_scopes