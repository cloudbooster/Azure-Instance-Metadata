# Azure's Instance Metadata Service --Preview

Instance Metadata service provides information regarding your running instance which can be used to manage and configure your instances on Azure. Azure's instance metadata service is a RESTful endpoint available to all IaaS VMs created via new [Azure Resource Manager](https://docs.microsoft.com/en-us/rest/api/resources/?redirectedfrom=MSDN). The endpoint is available at a well-known non routable IP address ( 169.254.169.254 ) that can be accessed only from within the VM. This service would provide important data regarding instance , its network configuration and upcoming maintenance events. For additional information See [Metadata categories](#instance-metadata-data-categories).

### Important information
Currently this service is in **preview** and hosts information regarding instance and upcoming maintenance events. Service will continue receiving updates and this page will reflect the specific [data categories](#instance-metadata-data-categories) available.

# Retrieving instance metadata 

Instance metadata is available for running VMs created/managed using [Azure Resource Manager](https://docs.microsoft.com/en-us/rest/api/resources/?redirectedfrom=MSDN). 
To access all data categories for an instance use the following URI

```
$ curl -H Metadata:true http://169.254.169.254/metadata/latest
```
The default output for all instance metadata is of text format (content type text/plain)

## Usage Examples
Below is set of examples and usage semantics for instance metadata service
### Versioning 
Instance metadata service is versioned and list of versions can be retrieved by the following 
```
$ curl -H Metadata:true http://169.254.169.254/metadata

v2/
latest/
```
Latest version will always point to last release version, currently this points to V2 version.
Earlier version will be made available incase your scripts has dependencies on data formats.
### Data output
By default instance metadata returns data in text (content type=text/plain), however different node elements can return data in different default format as applicable , below table is a quick reference for data formats 

Element | default data format | Other formats
--------|---------------------|--------------
/instance | text | Json
/scheduledevents | Json | None

To get data in json format for instance ?format=Json can be appended to the request , for example 
```
curl -H Metadata:true http://169.254.169.254/metadata/v2/instance?format=json
```
### Security
Instance metadata endpoint is accessible only from within the running instance on a non-routable 169.254.169.254 IP address. In addition, any request with **X-Forwarded-For header** is rejected by the metadata service. 
We also require requests to contain **Metadata:true** header being sent to ensure that the actual request was directly intended and not a part of unintentional redirection. 
### Error
In case of a data element not found or malformed requests IMDS will return standard HTTP Error, few examples below

HTTP Return  Code | Reason
----------------|-------
200 | OK
400 | Bad Request, missing header , please pass -H Metadata:true
404 | Not Found, requested element doesn’t exist 
405 | Method not supported 
429 | Too many requests

### Examples
#### Retrieving  the network information 
```
curl -H Metadata:true http://169.254.169.254/metadata/latest/instance/network/interface/0
ipv4/
ipv6/
mac
```
#### Retrieving  public IP address
```
curl -H Metadata:true http://169.254.169.254/metadata/latest/instance/network/interface/0/ipv4/ipaddress/0/publicip
```
# Instance Metadata data categories
Following table has a list of all data categories available via Instance Metadata

Data | Description
-----|------------
ipv4/Ipaddress | Local IP address of the VM 
ipv4/publicip | Public IP address for the Instance 
subnet/address | Address for subnet 
subnet/dnsservers/ipaddress1 | Primary DNS server
subnet/dnsservers/ipaddress2 | Secondary DNS server
subnet/prefix | Subnet prefix , example 24
ipv6/ipaddress | IPv6 address for the VM
mac | VM mac address 
scheduledevents | see [scheduledevents](#scheduled-events)

# Availability 
The current Preview is available in **West US Central** Region of Azure. 
For trying out IMDS simply create a VM from [Azure Resource Manager ](https://docs.microsoft.com/en-us/rest/api/resources/?redirectedfrom=MSDN) or [Azure Portal](http://portal.azure.com) and follow the [examples](#retrieving-instance-metadata)
# Scheduled Events
Azure Metadata service enables you to discover information about your Virtual Machine hosted in Azure. The Azure Metadata Service provides information about upcoming maintenance events (for example, reboot) so your service can prepare for them and limit disruption. It's available for all Azure Virtual Machine types including PaaS as well as IaaS. The service gives your Virtual Machine time to perform preventive tasks and minimize the effect of an event. For example, your service might drain sessions, elect a new leader, or copy data after observing that an instance is scheduled for reboot to avoid disruption. The Service enables your Virtual Machine to notify Azure that it can continue with the event (ahead of time). This is useful for expediting the impact when your service has successfully complete the graceful shutdown sequence.
## Introduction - Why Scheduled Events?
With Scheduled Events, you can learn of (discover) upcoming events which impact the availability of your VMs and take proactive operations to limit the impact on your service. Multi-instance workloads, which use replication techniques to maintain state, may be vulnerable to frequent outages happening across multiple instances. Such outages may result in expensive tasks (for example, rebuilding indexes) or even a replica loss. In many other cases, using graceful shutdown sequence improves the overall service availability. For example, completing (or canceling) in-flight transactions, reassigning other tasks to other VMs in the cluster (manual failover), remove the Virtual Machine from a load balancer pool. There are cases where notifying an administrator about upcoming event or even just logging such an event help improving the serviceability of applications hosted in the cloud.
Azure Metadata Service surfaces scheduled events in the following use cases:     
•	Platform initiated ‘impactful’ maintenance (for example, Host OS rollout)
•	Platform initiated ‘impact-less’ maintenance (for example, In-place VM Migration)
•	Interactive calls (for example, user restarts or redeploy a VM)
Scheduled Events – The Basics
Azure Metadata service expose information about running Virtual Machines using a REST Endpoint from within the VM. The information is available via a Non-routable IP so that it is not exposed outside the VM.
### Scope
Scheduled events are surfaced to all VMs in a cloud service or to all VMs in an Availability Set. As a result, you should check the *Resources field in the event to identify which VMs are going to be impacted.
### Discover the Endpoint
In the case where a Virtual Machine is created within a Virtual Network (VNet), the metadata service is available from the non-routable IP of: 169.254.169.254
In the case where a Virtual Machine is used for cloud services (PaaS), metadata service endpoint could be discovered using the registry.
{HKEY_LOCAL_MACHINE\Software\Microsoft\Windows Azure\DeploymentManagement}
### Versioning
The Metadata Service uses a versioned API in the following format: http://{ip}/metadata/{version}/scheduledevents It is recommended that your service consumes the latest version available at: http://{ip}/metadata/latest/scheduledevents
### Using Headers
When you query the Metadata Service, you must provide the following header *Metadata:true *.
### Enable Scheduled Events
The first time you make the call to the scheduled events endpoint, Azure will implicitly enable the feature on your Virtual Machine. As a result, you should expect a delayed response in your first call of up to a minute.

# Using the API
## Query for events
You can query for Scheduled Events simply by making the following call
curl -H Metadata:true http://169.254.169.254/metadata/latest/scheduledevents
A response contains an array of scheduled events. An empty array means that there are currently no events scheduled. In the case where there are scheduled events, the response will look like this:

```
{
"Events":[
      {
            "EventId":{eventID},
            "EventType":"Reboot" | "Redeploy" | "Pause",
            "ResourceType":"VirtualMachine",
            "Resources":[{resourceName}],
            "EventStatus":"Scheduled" | "Started",
            "NotBefore":{timeInUTC},              
     }
]
}
```
EventType Captures the expected impact on the Virtual Machine where:
* Pause: The Virtual Machine will be paused for few seconds. There is no impact on memory, open files or network connections
* Reboot: The Virtual Machine is scheduled for reboot (memory will be wiped).
* Redeploy: The Virtual Machine is scheduled to move to another node (ephemeral disk will be lost).
Note that when an event is scheduled (Status = Scheduled) Azure shares the time after which the event can start (specified in the NotBefore field).

## Starting an event (expedite)
Once you have learned of an upcoming event, and completed your logic for graceful shutdown, you can start the event indicating Azure to move faster (when possible) by making a POST call
# Samples

## PowerShell Sample
The following sample reads the metadata server for scheduled events and record them in the Application event log before acknowledging.

```
$localHostIP = "169.254.169.254"
$ScheduledEventURI = "http://"+$localHostIP+"/metadata/latest/scheduledevents"

# Call Azure Metadata Service - Scheduled Events 
$scheduledEventsResponse =  Invoke-RestMethod -Headers @{"Metadata"="true"} -URI $ScheduledEventURI -Method get 

if ($json.Events.Count -eq 0 )
{
    Write-Output "++No scheduled events were found"
}

for ($eventIdx=0; $eventIdx -lt $scheduledEventsResponse.Events.Length ; $eventIdx++)
{
    if ($scheduledEventsResponse.Events[$eventIdx].Resources[0].ToLower().substring(1) -eq $env:COMPUTERNAME.ToLower())
    {    
        # YOUR LOGIC HERE 
         pause "This Virtual Machine is scheduled for to "+ $scheduledEventsResponse.Events[$eventIdx].EventType

        # Acknoledge the event to expedite
        $jsonResp = "{""StartRequests"" : [{ ""EventId"": """+$scheduledEventsResponse.events[$eventIdx].EventId +"""}]}"
        $respbody = convertto-JSon $jsonResp

        Invoke-RestMethod -Uri $ScheduledEventURI  -Headers @{"Metadata"="true"} -Method POST -Body $jsonResp 
    }
}

```
## C# Sample

```
static async Task RunAsync()
{
    try
    {
        using (var client = new HttpClient())
        {
            client.BaseAddress = new Uri("http://169.254.169.254/");
            client.DefaultRequestHeaders.Accept.Clear();
            client.DefaultRequestHeaders.Accept.Add(new\
            MediaTypeWithQualityHeaderValue("application/json"));
            HttpResponseMessage response = await client.GetAsync(\
            "metadata/latest/scheduledevents");

            if (response.IsSuccessStatusCode)
            {
                JSonHelper helper = new JSonHelper();
                CloudControlMessage msg =
                helper.ConvertJSonToObject&lt;CloudControlMessage&gt;\
                (response.Content.ReadAsStringAsync().Result);
                await AnalyzeCloudControlEvents(msg);
            }

            else
            {
                Debug.WriteLine("REST Call returned failed {0}",
                response.IsSuccessStatusCode);
            }
        }
    }

    catch (Exception ex)
    {
        Debug.WriteLine("Exception caught {0}", ex);
    }
}
```

# FAQ
1. I am getting Bad request. Required metadata header not specified?
    #### IMDS requires header of Metadata:true to be passed in the request. Passing header will allow access
2. I am not getting information for my VM 
    #### Currently IMDS supports Azure Resource Manager created instances only(including VM Scale Sets), in future we  will add support for Cloud Services VMs 
3. I am getting  500 - Internal server error
    #### Currently Instance Metadata Preview is available only in West US Central Region, Please deploy your VMS there.  
