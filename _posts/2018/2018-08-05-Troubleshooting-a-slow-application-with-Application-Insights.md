---
layout: post
title: Troubleshooting a slow application with Application Insights
share-img: https://tjaddison.com/assets/2018/2018-08-05/Endpoints.png
tags: [Azure, "Application Insights", DevOps]
---

[Application Insights] (AI) is a fantastic instrumentation framework that with minimal/zero configuration will start giving you rich data about your applications performance.

We recently got some reports that one of our website solutions was 'slow' when developing locally, and as much as we'd like to turn to the DBA to blame the database (you know [what DBA stands for], right? I quite like Database Blamed Always...) when you have all of your dependencies instrumented it is easy enough to find the problem.

From our starting point of 'it runs slow locally, I think it is the database' we'll figure out precisely how slow it is, and whether it really is the database or not.

<!--more-->

## Investigating with the portal

After navigating to the AI resource that holds our development telemetry, we select the *performance* blade.  This lets us to navigate to the time period where the application was misbehaving, and then filter out all of the roles and instances we don't care about.  

![Role Selection](/assets/2018/2018-08-05/RoleSelection.png)

In our case we drilled down to select the frontend website application, and the development machine.

We now get an overview of the performance of each endpoint (e.g. /Items, /Account, /Login), and can quickly see that a few endpoints had _really poor_ performance - 4 minutes to load one page.  Discrepancies between production and local development are to be expected (none of our development team run any 64 core machines as far as I know!), but 4 minutes is pretty extreme.

![Endpoints](/assets/2018/2018-08-05/Endpoints.png)

From here we can either drill into that single slow call, or take a look at overall dependency performance.

## Overall dependencies

Flipping to the dependencies view we see a breakdown of every call as well as the call count, and average duration.  You'll need to know what 'normal' looks like for your application (and modulate normal for the environment - I wouldn't expect dev to be as performant as production), and in this case it was immediately clear something was up.  Call times I was expecting to be single-digit/low double-digit milliseconds were running at almost 100ms.

![Slow Calls](/assets/2018/2018-08-05/SlowCalls.png)

Flipping between a machine that wasn't suffering the problems and the problematic machine revealed the 'slow' machine was almost 5-10x slower than the normal machine.  This can be done via the portal, or by dropping into the [Analytics] portal and querying by a few different machines:

```
let start=datetime("2018-07-30T15:00:00.000Z");
let end=datetime("2018-07-30T17:00:00.000Z");
dependencies
| where timestamp > start and timestamp < end
| where (cloud_RoleName == 'Website' and cloud_RoleInstance == 'SLOW_PC_HOSTNAME')
| summarize count_=sum(itemCount), avg(duration), percentiles(duration,50,95) by target, name
```

Looking at averages as well as the median and 95th percentile shows us that this isn't some long-tail of slow requests, but that all of our requests are slow.  This isn't just the database, but is also impacting the GRPC calls to other microservices.

![Slow Calls in Analytics](/assets/2018/2018-08-05/SlowCallsAnalytics.png)

> Application Insights doesn't capture GRPC dependencies out of the box, but it does capture SQL and HttpClient calls.

## Looking at an individual request

Rather than looking at all dependencies we can see how a single operation performed.  By clicking on the slow operation (e.g. /Login) we can then see a list of sample operations AI has captured.  Clicking on any of those shows all the dependencies associated with that individual operation in a timeline.  We can see again this isn't one slow dependency - they're _all_ slow.

![Slow Dependencies](/assets/2018/2018-08-05/SlowDependencies.png)

> AI by default turns every request into an operation.  If you're instrumenting a windows service you'll need to create and track your own operations if you want to see them show up and tag all dependencies to a parent operation.  See more in the [Telemetry Correlation docs].

As with the overall view we can drop down to Analytics to query the raw data for a single operation:

```
dependencies
| where operation_Id == "oPerAtionId="
| summarize sum(itemCount), avg(duration) by target, name
| order by sum_itemCount desc
```

> Note that we sum(itemCount) rather than count() because of [automatic sampling].  If there is no sampling this has no impact on your results, but if your data is being sampled and you don't sum(itemCount), your numbers will be off!

![Slow Dependencies in Analytics](/assets/2018/2018-08-05/SlowDependenciesAnalytics.png)

This tells us the same story as the overall view - that all dependencies seem to be slow.  This view additionally tells us that we might have an [N+1 problem] somewhere.

## The verdict

AI is fantastic for pinpointing issues, but it still isn't capable of telling you how to fix them (yet!).  In this case I already had a pretty good idea of what the problem was, and we used AI to confirm it.  The slow development machine was located on the other side of the Atlantic from the development services it was configured to talk to!  While some of our engineering team are working on increasing the speed of light, the short term fix was to target services local to the development machine.

The examples above are just the tip of the iceberg when it comes to diagnosing application performance (whether through the portal, using workbooks, or via Analytics - you can quickly ask and answer some incredibly detailed questions).  AI is moving pretty quickly (every few months a new feature or experience pops up), and in the last 12 months we've seen Log Analytics and AI Analytics converge to support the [Kusto Query Language] (KQL), and so it is worth keeping an eye on the [AI Blog].

[Application Insights]: https://docs.microsoft.com/en-gb/azure/application-insights/app-insights-overview
[what DBA stands for]: http://michaelcorey.com/blog/what-does-dba-really-mean/
[Analytics]: https://docs.microsoft.com/en-gb/azure/application-insights/app-insights-analytics
[Telemetry Correlation docs]: https://docs.microsoft.com/en-us/azure/application-insights/application-insights-correlation
[automatic sampling]: https://docs.microsoft.com/en-us/azure/application-insights/app-insights-sampling
[N+1 problem]: https://www.brentozar.com/archive/2018/07/common-entity-framework-problems-n-1/
[Kusto Query Language]: https://www.pluralsight.com/courses/kusto-query-language-kql-from-scratch
[AI Blog]: https://azure.microsoft.com/en-gb/blog/tag/application-insights/