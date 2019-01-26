---
layout: post
title: Overhead of EventHub outputs with Azure Function Apps
share-img: https://tjaddison.com/assets/2019/2019-01-21/EventHubComparison.png  
tags: [Azure, PowerShell, Function Apps, EventHubs]
---

In a world where you're billed for what you use, it pays to really understand what _exactly_ it is you are using.  The [pricing model][Function app pricing] of the Azure Function Apps consumption plan sounds pretty simple (pay based on execution count and execution time) - though as always the devil is in the detail.

In a recent project where we've migrated a workload from a dedicated (on-prem) server to a function app, and in trying to deconstruct the costs of the serverless deployment someone asked what sounded like a fairly simple question:

- **Do we pay (on the function app side) for the output to an Event Hub, and if so how much does it cost?**

This post explores the answer to that question for a trivial C# function, as well as providing a few pointers to help get your head around consumption billing.

If you're wondering why we wanted to deconstruct the function's cost when an _execution unit_ costs a mere 16 picdollars (16x10^-12), consider what we saw after one week under full load:

![A lot of executions](/assets/2019/2019-01-21/Trillions.png)

5 trillion execution units...  Interesting.

>Execution units vs. what you see on the pricing page is covered later in the post.

<!--more-->

## Consumption billing

Functions hosted on the consumption plan charge you for:

- Every function execution (currently $0.20 per million execution)
- Every GB-s used ($0.000016)

If the function runs for one second and uses 1GB of memory, your function has used 1GBs.  The smallest quantum is 128MB for memory, and 1 millisecond for duration.  Billing rounds up, so a function using 1MB for 10us is billed at 0.128GB x 1ms, a function using 129MB for 1001us is billed at 0.256GB for 2ms, etc.

Execution time is measured from the start of the function to the end of the function - and is _not_ CPU time.  If your function does any calls to external systems (like an Event Hub) that is billed.  This includes the input/output bindings.

> This is one reason why your functions shouldn't call other functions - you'll get billed for both!  If you need to do this then [Durable Functions] might be a better fit, though prepare for a whole host of new billing complexities to wrap your head around.

Billing information is provided at the function-app level, and not at a per-function level.

Azure Monitor doesn't expose the GB-s billing metric directly, and instead exposes a metric called *Function Execution Units*.  One of these is equal to 1MB-ms.  Converting from GB-s to MB-ms can be done by dividing through by 1,024,000 (GB -> MB is 1024, s -> ms is 1000).

All of these together tell us that:

- Any optimisation that doesn't help us cross a boundary of 128MB won't impact cost
- Any optimisation that doesn't help us cross a boundary of 1ms execution time wont' impact cost
- Adding an Event Hub output is only 'free' if it takes <1ms and has a memory impact that won't take us over a 128MB memory boundary

> For more details check out the [Consumption FAQ].  Precise details on how the memory usage is billed is pretty hard to pin down - there are a handful of outstanding issues that may need to be resolved before it's straightforward to examine (see [here][memory issue 1], [here][memory issue 2], and [here][memory issue 3]).

## Test harness

Our test harness is going to take a stock function (that returns) and compare it to a funciton which outputs a message to an Event Hub.

The functions below were deployed as separate function apps (to enable parallel testing and still get the granular cost metrics needed).  You can view the [sample project on GitHub][function app project].  The deployment was a straightforward 'Publish' from Visual Studio, with each function getting it's own storage account/app service plan (on the consumption tier), and application insights resource.  An Event Hub was created to act as the output binding for the second test.

### No-Op Test
```csharp
[FunctionName("NoOp")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = null)] HttpRequest req,
    ILogger log)
{
    return new OkResult();
}
```

### Event Hub Test
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

### Running the Test

This PowerShell script uses [runspaces][runspace blog] to execute the tests in parallel.  We're interested in executing the function a set number of times and examining cost (as opposed to load testing), and so any delay or 'unfairness' due to the client-driver being slow isn't an issue.

Once the code has finished executing we can then go to Azure Monitor and pull the stats on execution count for both function apps (with/without an Event Hub output binding), and see what overhead there is.

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
        $timings += (Measure-Command {
            Invoke-WebRequest -Uri $test.Uri -Method Post | Out-Null
        }).TotalMilliseconds
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

## Results

After running the test multiple times the numbers were fairly consistent - the function with an Event Hub output binding cost about *18% more* to run than a stock function.

![With and without EventHub output](/assets/2019/2019-01-21/EventHubComparison.png)

After crunching the raw numbers (which you can export from Azure Monitor to a CSV) I settled on the following:

|Function|Execution Units|
|---|---:|
|Stock|17,371|
|Event Hub Output|20,500|

The addition of an Event Hub output was costing *3,129 execution units*, or about **$0.000000049 per execution**
 (from the function app side - don't forget there's a whole host of billing associated with the hub itself!).

To put that into context here is what that translates to per-week in **additional cost for the Event Hub output binding** for various requests per second:

|RPS|Additional Weekly Cost|
|---:|---:|
|100|$2.96|
|500|$14.78|
|1000|$29.57|

It's worth noting that is an additional cost, and that at high RPS the execution count starts to dominate.  For a function being executed at 1000 RPS you'd pay:

- $164.15 in GBs for the stock function
- $29.58 in GBs for the Event Hub output binding
- $120.96 for the execution count (1000 RPS is 605 million executions per week)
- **$314.69** in total

Although these numbers are almost certain to be different every time (and hopefully they'll keep going down as improvements are made to the hosting/runtime/bindings), it's probably safe to say that when looking into cost that the presence of a binding itself is _not_ the thing to worry about.

## Closing Thoughts

Cost-control in the cloud can be very interesting.  Knowing exactly how much we're spending is easy, but knowing what we're spending it on (and if that's the most efficient way to solve the problem)...another matter entirely.

This post has scratched the surface of a whole bunch of questions, and we haven't even got into considerations such as dedicated vs. consumption (some excellent coverage on the [Azure from the trenches blog][dedicated vs consumption]).

In this specific example we came to realise pretty quickly that the real issue is the app sending messages needs re-architecting to reduce the number of messages sent (via batching) rather than chasing down micro-optimisations in the function app.  Some napkin math reveals we can probably get a 5-10x reduction in call count, which directly translates to a reduction in cost.

Understanding how the cloud services are built and how you're charged for using them is key in building the mental models required to architect a cost-effective deployment - the savings at scale can be quite significant.

[Function app pricing]: https://azure.microsoft.com/en-us/pricing/details/functions/
[Consumption FAQ]: https://github.com/Azure/Azure-Functions/wiki/Consumption-Plan-Cost-Billing-FAQ
[Durable Functions]: https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview
[memory issue 1]: https://github.com/Azure/azure-functions-host/issues/3852
[memory issue 2]: https://github.com/Azure/Azure-Functions/issues/726
[memory issue 3]: https://github.com/Azure/Azure-Functions/issues/762
[function app project]: https://github.com/taddison/csharp-coffre/tree/master/function-app-eventhub-test
[runspace blog]: https://blog.netnerds.net/2016/12/runspaces-simplified/
[dedicated vs consumption]: https://www.azurefromthetrenches.com/azure-functions-scaling-with-a-dedicated-app-service-plan/