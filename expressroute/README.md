# Monitoring Traffic on ER circuit 

Azure provides default metrics for ER circuits and ER connections on ER Gateways. ER circuit metrics provides total ingress and egress throughput metrics on a ER circuit. ER connection metric on ER Gateway provides ingress and egress throughput metrics on a ER Gateway which is connected to a ER circuit. 

To monitor ER throughput more granularly, e.g. by session, NSG Flow log can be utilized. NSG Flow log collects traffic statistics every minutes and saves it to Azure Blob Storage. 

Below link shows how to use flow log integrating with Logstash/ElasticSearch/Grafana to analysis and to build dashboard.

https://docs.microsoft.com/en-us/azure/network-watcher/network-watcher-nsg-grafana

Azure Traffic Analyzer processes flow log every 10 minutes or every hour depending on the configuration and provides aggregated analytics data on Log Analytics workspace. Log Analytics provides KUSTO, powerful query langugage, which can be used to further analysis. Although the data granularity is less than flow log, it gives easier tool to build a dashboard. 

Below is KUSTO query statements to build dashboard for ER throughput by top 10 sessions (sourceIP/destIP/destport tuple) by transferred bytes.

### Log Analytics query for ER Ingress monitoring

```
let min_t = toscalar(
AzureNetworkAnalytics_CL | summarize min(TimeGenerated)
);
let max_t = toscalar(
AzureNetworkAnalytics_CL | summarize max(TimeGenerated)
);
let tuples = toscalar(
    AzureNetworkAnalytics_CL
    | where SubType_s == 'FlowLog' and FlowType_s == 'S2S' and ConnectionType_s == 'ExpressRoute'
    | extend tuple = strcat(SrcIP_s, '-', DestIP_s, ':', DestPort_d)
    | extend // ER Ingress
    IngressBytesAtDest = iff(FlowDirection_s == 'I', tolong(OutboundBytes_d), 0),
    IngressBytesAtSrc = iff(FlowDirection_s == 'O', tolong(InboundBytes_d), 0)
    | summarize IngressBytesAtDest = sum(IngressBytesAtDest), IngressBytesAtSrc = sum(IngressBytesAtSrc)  by tuple
    | extend IngressBytes = IngressBytesAtDest + IngressBytesAtSrc
    | top 10 by IngressBytes desc
    | summarize makeset(tuple)
    | project set_tuple
);
AzureNetworkAnalytics_CL
| where SubType_s == 'FlowLog' and FlowType_s == 'S2S' and ConnectionType_s == 'ExpressRoute'
| extend tuple = strcat(SrcIP_s, '-', DestIP_s, ':', DestPort_d)
| extend // ER Ingress
    IngressBytesAtDest = iff(FlowDirection_s == 'I', tolong(OutboundBytes_d), 0),
    IngressBytesAtSrc = iff(FlowDirection_s == 'O', tolong(InboundBytes_d), 0)
| where tuple in (tuples)
| make-series IngressBytes = sum(IngressBytesAtDest + IngressBytesAtSrc) default = 0 on TimeGenerated from min_t to max_t step 10m by tuple 
| render timechart with (ytitle = "Ingress Bytes per 10 min")
```

### Log Analytics query for ER Egress monitoring

```
let min_t = toscalar(
AzureNetworkAnalytics_CL | summarize min(TimeGenerated)
);
let max_t = toscalar(
AzureNetworkAnalytics_CL | summarize max(TimeGenerated)
);
let tuples = toscalar(
    AzureNetworkAnalytics_CL
    | where SubType_s == 'FlowLog' and FlowType_s == 'S2S' and ConnectionType_s == 'ExpressRoute'
    | extend tuple = strcat(SrcIP_s, '-', DestIP_s, ':', DestPort_d)
    | extend // ER Egress
    EgressBytesAtDest = iff(FlowDirection_s == 'I', tolong(InboundBytes_d), 0),
    EgressBytesAtSrc = iff(FlowDirection_s == 'O', tolong(OutboundBytes_d), 0)
    | summarize EgressBytesAtDest = sum(EgressBytesAtDest), EgressBytesAtSrc = sum(EgressBytesAtSrc)  by tuple
    | extend EgressBytes = EgressBytesAtDest + EgressBytesAtSrc
    | top 10 by EgressBytes desc
    | summarize makeset(tuple)
    | project set_tuple
);
AzureNetworkAnalytics_CL
| where SubType_s == 'FlowLog' and FlowType_s == 'S2S' and ConnectionType_s == 'ExpressRoute'
| extend tuple = strcat(SrcIP_s, '-', DestIP_s, ':', DestPort_d)
| extend // ER Egress
    EgressBytesAtDest = iff(FlowDirection_s == 'I', tolong(InboundBytes_d), 0),
    EgressBytesAtSrc = iff(FlowDirection_s == 'O', tolong(OutboundBytes_d), 0)
| where tuple in (tuples)
| make-series EgressBytes = sum(EgressBytesAtDest + EgressBytesAtSrc) default = 0 on TimeGenerated from min_t to max_t step 10m by tuple 
| render timechart with (ytitle = "Egress Bytes per 10 min")
```


mint_t, max_t variables are used to get time range values from Azure Portal UI. (Currently Log Analytics UI doesn't provides feature to get time range values from UI. When using Grafana dashboard, however, Grafana provides $__timeFrom(),$__timeTo() macros which can be used to get range values from grafana UI.

### Reference



https://grafana.com/docs/grafana/latest/datasources/azuremonitor/

https://techcommunity.microsoft.com/t5/azure-monitor/use-time-range-value-in-kusto-query-to-calculate-uptime/m-p/390582

https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/make-seriesoperator

https://www.cloudsma.com/2021/04/kusto-make-series-vs-summarize/

https://docs.microsoft.com/en-us/azure/network-watcher/traffic-analytics-schema

https://docs.microsoft.com/en-us/azure/network-watcher/traffic-analytics-faq#how-do-i-check-which-vms-are-receiving-most-on-premises-traffic-

https://docs.microsoft.com/en-us/azure/network-watcher/network-watcher-nsg-flow-logging-overview



