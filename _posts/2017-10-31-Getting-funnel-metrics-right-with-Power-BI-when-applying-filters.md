---
layout: post
title: Getting funnel metrics right with Power BI when applying filters
share-img: http://tjaddison.com/assets/2017-10-31/UpdatedDataModel.png
---
In a [previous post](/2017/09/10/Creating-recruitment-funnel-metrics-in-Power-BI) we built a simple set of recruitment funnel metrics that allowed us to plot the hiring funnel from a new application to (hopefully) a hire.  The core measure we built was *inclusive applications*, which works fine until you start to filter your data, after which point you go from a set of data like this:

![Wrong Funnel](/assets/2017-10-31/ApplicationData.png)

To a funnel that looks like this when filtered for the DBA role:

![Wrong Funnel](/assets/2017-10-31/DBAFunnel.png)

Which is kind of odd as we'd like to reflect the fact we phone screened two DBAs.

The reason the formula stops working when we filter is because none of the applications are in the stage 'Phone'.  Once you start to slice the data on multiple dimensions (e.g. What percent of my DBAs in London got past the Phone Screen in Q1 17?) it is very easy to end up in a situation where you'll be missing some data.

The good news is this is an easy fix.

<!--more-->

The core issue we have is with our data model - we've got a dimension that isn't captured by its own dimension table.  Extracting this is the first thing we should do, giving us a new data model:

![Wrong Funnel](/assets/2017-10-31/UpdatedDataModel.png)

Now we need to update our function to reference the dimension table (which as an added benefit simplifies the formula, as now we can use All(Table) rather than having to enumerate the columns):

```dax
Corrected Inclusive Applications = 
CALCULATE (
    [Applications],
    FILTER (
        ALL (Stages),
        Stages[StageOrder] >= MAX ( Stages[StageOrder] )
    )
)
```

This gives us the chart we expect:

![Wrong Funnel](/assets/2017-10-31/UpdatedDBAFunnel.png)

You can download a workbook with the data in [here](/assets/2017-10-31/Data.xlsx), and the Power BI file with data and measures [here](/assets/2017-10-31/FunnelSample.pbix).