---
layout: post
title: Keeping Application Insights Costs Under Control
share-img: https://tjaddison.com/assets/2019/2019-05-31/XXXXXTBDXXXXXX
tags: [Azure, Application Insights, PowerShell]
---

https://azure.microsoft.com/en-us/pricing/details/monitor/
https://docs.microsoft.com/en-us/azure/azure-monitor/app/pricing

```powershell
Import-Module Az
Connect-AzAccount

$CAP_CUTOFF = 10 ## Free tier = 0.161 - 5GB/month
$CAP_TO = 1
$ignoreResources = @("Subscription.Resource Group.Resource")

$subscriptions = Get-AzSubscription

$actions = @()
foreach($sub in $subscriptions) {
    $subscriptionName = $sub.Name
    $sub | Set-AzContext

    $appInsightResources = Get-AzApplicationInsights

    $aiResources = @()
    foreach($ai in $appInsightResources) {
        $aiResources += Get-AzApplicationInsights -ResourceGroupName $ai.ResourceGroupName -Name $ai.Name -IncludeDailyCap
    }

    foreach($ai in $aiResources) {
        $identifier = "$($subscriptionName).$($ai.ResourceGroupName).$($ai.Name)"

        if($ai.Cap -gt $CAP_CUTOFF) {
            if($ignoreResources -contains $identifier) {
                $action = "Ignore"
            } else {
                $action = "Reduce to $CAP_TO"
                Set-AzApplicationInsightsDailyCap -ResourceGroupName $ai.ResourceGroupName -Name $ai.Name -DailyCapGB $CAP_TO
            }
        } else {
            $action = "Below $CAP_CUTOFF"
        }
        
        $actions += [PSCustomObject]@{
            Subscription = $subscriptionName
            ResourceGroup = $ai.ResourceGroupName
            ResourceName = $ai.Name
            Cap = $ai.Cap
            MonthlyCost = $ai.Cap * 31 * 2.3
            Action = $action
        }
    }
}

$actions | Sort-Object -Property MonthlyCost -Descending | Format-Table

```