---
layout: post
title: Simplifying alert configuration in the OMS to Slack function app
share-img: http://tjaddison.com/assets/2017-11-07/ExcelConfiguration.png
---

In the [last post](http://tjaddison.com/2017/08/29/Building-an-OMS-metric-alert-to-Slack-bridge-with-Azure-functions) we had a fairly complete (if limited) solution for the routing or arbitrary OMS metric alerts to Slack.  After having used this in production for a while I can report that:
- When configured correctly it works as intended (we get the right alerts in the right place)
- Explaining to people how they need to configure a new alert is extraordinarily difficult
- There is a seemingly arbitrary separation of alert configuration (some in the OMS webhook payload, some in the function app itself)

I was planning on extracting the configuration from the application, and rather than extract as-is I've made changes that make configuring an alert and any overrides pretty straightforward.

Inspired by a recent [Hanselman post](https://www.hanselman.com/blog/CloudDatabaseNoSQLNahJustUseCSVsAndCsvHelper.aspx) I've opted to use CSVs and CSVHelper to store the alert configuration data.  The tabular format and lack of references between config files leant itself to this task, and I wanted to start with a format that ensured it was easy to source control the configuration (so they can live and be versioned with the project).

If you want to look at the finished repo you can see it [on GitHub](https://github.com/taddison/blog-oms-to-slack/tree/master/GenericFunctionExternalConfig) - the rest of the post will focus on the key changes to configuring an alert and how I've implemented those change.

![Excel Configuration](/assets/2017-11-08/ExcelConfiguration.png)

<!--more-->

## Configuring an alert

One of the most significant changes is in the OMS payload.  I've included the old & new payloads for a CPU alert below.

```json
// Old
{
  "IncludeSearchResults": true,
  "WarningThreshold": 0.35,
  "CriticalThreshold": 0.45,
  "ValueMultiplier": 0.01,
  "Channel": "#oms-alerts",
  "LessThanThresholdIsBad": false,
  "AlertMessage": "Infra - CPU",
  "MetricName": "Processor Usage %",
  "FormatString": "P0",
  "ObservationThreshold": 3
}
// New
{
  "IncludeSearchResults": true,
  "MetricName": "Processor Usage %"
}
```

The OMS queries themselves remain unchanged.

The only additional piece of information we provide in the payload (IncludeSearchResults is needed to get OMS to send the metrics to our function) is the *MetricName*, and this acts as the key which maps to the configuration data in our app.  The *MetricName* in the OMS payload needs to match the MetricName in all configuration CSVs.

The configuration data for each alert requires a minimum of a *DefaultAlertConfig* and a *DefaultAlertNotificationConfig* file.  These are CSVs and are stored in the Configuration directory of the application (these need a CopyToOutputDirectory setting of CopyIfNewer to ensure they get deployed).

The *DefaultAlertConfig* file contains the details needed to convert from OMS (the *ValueMultiplier*), as well as the default alerting thresholds.

As a reminder the *ValueMultiplier* is needed to convert values which might be reported as whole numbers in OMS (e.g. 99% is reported as 99) into values appropriate for formatting (if we want to format as P0 we need the value to be 0.99).  The value post-multiplication is also used when comparing to any alert thresholds.

The other column which got reworked here is minimum violations to alert - which is hopefully better named than its previous incarnation 'ObservationThreshold'.  If in our example CSV below the processor query had 5 data points, at least 3 would need to be at least 35% to trigger a warning alert (and 3 would need to be at least 50% for critical).

**defaultAlertConfig.csv**
```csv
metricName,warningThreshold,criticalThreshold,lessThanThresholdIsBad,minimumViolationsToAlert,valueMultiplier
Processor Usage %,0.35,0.5,false,3,0.01
Free Space %,0.2,0.1,true,1,0.01
Free Megabytes,10000,5000,true,1,1
```

The DefaultAlertNotificationConfig file is used to determine how to format the alert, the message that should be used, and the default channel to route the alert to.

**defaultAlertNotificationConfig.csv**
```csv
metricName,channel,formatString,alertMessage
Processor Usage %,#alerts,P0,Infra - CPU
Free Space %,#alerts,P0,Infra - Drive
Free Megabytes,#alerts,N0,Infra - Memory
```

## Customising alert thresholds or routing

Setting per-server rules for alerts is now done by adding records to the OverrideAlertConfig CSV file, which has the following format:

**overrideAlertConfig.csv**
```csv
metricName,machineName,warningThreshold,criticalThreshold,minimumViolationsToAlert
Processor Usage %,Server 2,0.3,0.4,
```

Metric and Machine are both mandatory, and every other field is optional.  The way the above record reads is:
- For Processor Usage % on Server 2
- Override warning to 30%
- Override critical to 40%
- Inherit the minimum violations to alert from the default

For routing the configuration file is OverrideAlertNotificationConfig, and it allows us to override based on either metric, machine, or both.

**overrideAlertNotificationConfig.csv**
```csv
metricName,machineName,channel
,Server1,#Server1Team
Free Space %,,#memory-monitors
Processor Usage %,Server2,#server2-cpu
```

In this example you can see we have one rule that applies to a whole server, one that applies to a metric, and one which is targeted at a specific combination of a server and a metric.

The current behaviour for routing is that an exact match (metric + machine) will replace the default channel, and a partial match will add a channel to the list.

## Reading configuration

The [CSVHelper](https://joshclose.github.io/CsvHelper/) was used to read CSV files, and this required minimal code.  Once the CSV definition was complete using the defaults was all that was required to build the config population methods.  Using the example below will read the CSV file using the headers and call the constructor to build each object.

```csharp
// OverrideAlertNotificationConfig.cs
public class OverrideAlertNotificationConfig
{
    public OverrideAlertNotificationConfig(string metricName = null, string machineName = null, string channel = null)
    {
        MetricName = string.IsNullOrWhiteSpace(metricName) ? null : metricName;
        MachineName = string.IsNullOrWhiteSpace(machineName) ? null : machineName;
        Channel = channel;
    }

    public string MetricName { get; private set; }
    public string MachineName { get; private set; }
    public string Channel { get; private set; }
}

// ConfigHelper.cs
private List<OverrideAlertNotificationConfig> GetOverrideAlertNotificationConfigs()
{
    using (var tr = File.OpenText(_context.FunctionAppDirectory + "/Configuration/overrideAlertNotificationConfig.csv"))
    {
        var csv = new CsvReader(tr);
        return csv.GetRecords<OverrideAlertNotificationConfig>().ToList();
    }
}
```
The _context is a Microsoft.Azure.WebJobs.ExecutionContext and is needed to get the root folder of our function deployment (as we're deploying the configuration files along with the function app we need to get at them, and the runtime doesn't execute from that folder in a function app).  More information can be found on the [WebJobs Github wiki](https://github.com/Azure/azure-webjobs-sdk-script/wiki/Retrieving-information-about-the-currently-running-function).

The defaults of CSVHelper automatically map an empty string in our CSV to null, so we don't have any additional configuration/maps required.

One area that does need some attention on the function app is doing a better job of reading/tolerating malformed CSVs.

## Using configuration

As we now have the configuration isolated in a ConfigHelper class we're able to make the AlertProcessor method a little simpler.  The start of the ProcessAlert method (that deals with determining whether or not an alert sent from OMS should actually be considered an alert) is now a little simpler.

```csharp
var alertConfig = _configHelper.GetAlertConfig(alert);

// Is this a < or > alert?
var comparison = alertConfig.LessThanThresholdIsBad ? LessThan : MoreThan;

// Aggregate metrics to produce a single summary record
var processedValues = alert.MetricValues.Select(mv => mv.Value * alertConfig.ValueMultiplier);

var totals = new
{
    Average = processedValues.Average()
    ,Min = processedValues.Min()
    ,Max = processedValues.Max()
    ,Critical = processedValues.Count(v => comparison(v, alertConfig.CriticalThreshold))
    ,Warning = processedValues.Count(v => comparison(v, alertConfig.WarningThreshold))
};

// Determine alert criticality
var isWarning = totals.Warning >= alertConfig.MinimumViolationsToAlert;
var isCritical = totals.Critical >= alertConfig.MinimumViolationsToAlert;

// If the alert doesn't cross the warning threshold return
if(!isWarning)
{
    return;
}
```

The final part of the method has also been simplified somewhat, so we're now getting closer to truly separating the alert logic from the notification logic.

```csharp
// Where should the alert go
var notificationConfig = _configHelper.GetAlertNotificationConfig(alert);

// Build message
// ...elided
    
// Send message
foreach(var channel in notificationConfig.Channels)
{
    await SlackHelper.SendSlackMessage(channel, message);
}
```

## Other changes

In order to keep the alerts as terse as possible stripping the domain name from a server proved useful, so rather than seeing Server1.foo.corp was alerting we'd just see Server1.  This is applied all the way in the function that receives the server name, and is hardcoded:

```csharp
machineName = machineName.Replace(".foo.corp", string.Empty);
```