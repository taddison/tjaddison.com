---
layout: post
title: Understanding space usage in Log Analytics
share-img: https://tjaddison.com/assets/2019/TBDTBDTBD
tags: [Azure, Log Analytics, Azure Monitor]
---

Data ingested to Azure Monitor logs is billed per-Gigabyte ingested.  As a workspace will typically grow to have data coming from many different sources and solutions it is helpful to have a set of queries that allow you to quickly drill into where exactly the GBs (or TBs!) of data you have stored comes from.

I've found the below queries very helpful starting points for three main scenarios:

- Regular monitoring (once/month) to see how data volumes are trending
- Reacting to a monitoring alert based on overall ingestion volumes
- Testing out a configuration change/new solution and observing the impact on data ingested

The latter is particularly important before rolling a change to a workspace with long retention - you wouldn't want (hypothically :)) to accidentally ingest 100GB of IIS logs and then be forced to retain them for 2 years...

```kql

```

The [official docs][1] are a great place to start reading more about space usage and monitoring for Azure Monitor logs.

[1]: https://docs.microsoft.com/en-us/azure/azure-monitor/platform/manage-cost-storage
[Wire Data Solution]: https://docs.microsoft.com/en-us/azure/azure-monitor/insights/wire-data