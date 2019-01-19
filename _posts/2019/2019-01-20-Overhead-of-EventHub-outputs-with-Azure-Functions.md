---
layout: post
title: Overhead of EventHub outputs with Azure Function Apps
share-img: https://tjaddison.com/assets/2019/.....
published: false
tags: [Azure, PowerShell, Functions Apps, EventHubs]
---

In a world where you're billed for what you use, it pays to really understand what _exactly_ it is you are using.  The [pricing model][Function app pricing] of Azure Function Apps sounds pretty simple (pay based on execution count and execution time) - though the devil is really in the detail.

In a recent project where we've migrated a workload from a dedicated server to a function app, someone asked what sounded like a fairly simple question:

-  **Do we pay (on the function app side) for the output to an event hub, and if so how much does it cost?**

This post explores one answer to that question (accurate as of today, for this specific example - no guarantees for the future!), and contains a few pointers to help understand the cost model for Azure functions.

If you're wondering why we wanted to decompose our funciton and understand what drives pricing when an _execution unit_ costs a mere 16 picdollars (16x10^12), consider what we saw after one week under full load:

![A lot of executions](/assets/2019/2019-01-20/Trillions.png)

5 trillion execution units...  Interesting.

>Execution units vs. what you see on the pricing page is covered later in the post.

<!--more-->

[Function app pricing]: https://azure.microsoft.com/en-us/pricing/details/functions/