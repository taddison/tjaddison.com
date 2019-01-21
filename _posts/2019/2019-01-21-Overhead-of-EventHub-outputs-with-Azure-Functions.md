---
layout: post
title: Overhead of EventHub outputs with Azure Function Apps
share-img: https://tjaddison.com/assets/2019/2019-01-21/EventHubComparison.png
published: false
tags: [Azure, PowerShell, Functions Apps, EventHubs]
---

In a world where you're billed for what you use, it pays to really understand what _exactly_ it is you are using.  The [pricing model][Function app pricing] of Azure Function Apps sounds pretty simple (pay based on execution count and execution time) - though the devil is really in the detail.

In a recent project where we've migrated a workload from a dedicated server to a function app, someone asked what sounded like a fairly simple question:

-  **Do we pay (on the function app side) for the output to an event hub, and if so how much does it cost?**

This post explores one answer to that question (accurate as of today, for this specific example - no guarantees for the future!), and contains a few pointers to help understand the cost model for Azure functions.

If you're wondering why we wanted to decompose our funciton and understand what drives pricing when an _execution unit_ costs a mere 16 picdollars (16x10^12), consider what we saw after one week under full load:

![A lot of executions](/assets/2019/2019-01-21/Trillions.png)

5 trillion execution units...  Interesting.

>Execution units vs. what you see on the pricing page is covered later in the post.

<!--more-->

## Recap of function app billing
- Consumption - link to FAQs, etc.
- Aside: consumption vs. dedicated?
- Pricing vs. Monitor metrics

## Functions under test
- Deployment notes (one app per function - easier to measure (monitor doesn't cut by function))
- Online https://github.com/taddison/csharp-coffre/tree/master/function-app-eventhub-test

```csharp
[FunctionName("NoOp")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = null)] HttpRequest req,
    ILogger log)
{
    return new OkResult();
}
```

```csharp
[FunctionName("ToEventHub")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = null)] HttpRequest req,
    ILogger log,
    [EventHub(eventHubName: "fatesteventhub", Connection = "eh-connection")] ICollector<string> outMessages)
{
    outMessages.Add("Hello from the function app");
    return new OkResult();
}
```

## Load testing? Bit of a distraction...maybe skip?

## Powershell to drive, Azure monitor to measure

## Results

## Conclusion




--
```powershell
$BASELINE_URI = "https://fa-eh-test-appinsights.azurewebsites.net/api/noop"
$EH_OUTPUT_URI = "https://fa-eh-test-output.azurewebsites.net/api/toeventhub"
$REPEATS = 10000

$tests = @(
    @{ Name = "Baseline"; Uri = $BASELINE_URI },
    @{ Name = "EH Output"; Uri = $EH_OUTPUT_URI }
)
 
$scriptblock = {
    Param (
        $test,
        $repeats
    )
    Write-Output "Testing $($test.Name) for $repeats repeats at $([DateTime]::UtcNow)"
    $timings = @()
    for($i = 0; $i -lt $repeats; $i++) {
        $timings += (Measure-Command {Invoke-WebRequest -Uri $test.Uri -Method Post | Out-Null}).TotalMilliseconds
    }

    $summary = $timings | Measure-Object -Average -Maximum
    Write-Output "Finished $($test.Name) - Average $($summary.Average) - Max $($summary.Maximum) at $([DateTime]::UtcNow)"
}

## Thank you https://blog.netnerds.net/2016/12/runspaces-simplified/ !

$pool = [RunspaceFactory]::CreateRunspacePool(1,2)
$pool.Open()
$runspaces = @()

$tests | ForEach-Object {
    $runspace = [PowerShell]::Create()
    $null = $runspace.AddScript($scriptblock)
    $null = $runspace.AddArgument($_)
    $null = $runspace.AddArgument($repeats)
    $runspace.RunspacePool = $pool
    $runspaces += [PSCustomObject]@{ Pipe = $runspace; Status = $runspace.BeginInvoke() }
}

while ($runspaces.Status -ne $null)
{
    $completed = $runspaces | Where-Object { $_.Status.IsCompleted -eq $true }
    foreach ($runspace in $completed)
    {
        $runspace.Pipe.EndInvoke($runspace.Status)
        $runspace.Status = $null
    }
}

$pool.Close()
$pool.Dispose()
```


![With and without EventHub output](/assets/2019/2019-01-21/EventHubComparison.png)

[Function app pricing]: https://azure.microsoft.com/en-us/pricing/details/functions/