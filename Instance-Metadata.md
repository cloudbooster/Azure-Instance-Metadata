
# Azure's Instance Metadata Service --Preview

> [!NOTE] 
> Previews are made available to you on the condition that you agree to the terms of use. For more information, see [Microsoft Azure Supplemental Terms of Use for Microsoft Azure Previews.] (https://azure.microsoft.com/en-us/support/legal/preview-supplemental-terms/)
>

Instance Metadata service provides information regarding your running virtual machine instances. This information can be used to manage and configure your instances on Azure. Azure's instance metadata service is a RESTful endpoint available to all IaaS VMs created via new [Azure Resource Manager](https://docs.microsoft.com/en-us/rest/api/resources/?redirectedfrom=MSDN). The endpoint is available at a well-known non routable IP address (*169.254.169.254*) that can be accessed only from within the VM. This service would provide important data regarding virtual machine instance, its network configuration, and upcoming maintenance events. For additional information See [Metadata categories](#instance-metadata-data-categories).

### Important information
Currently this service is in **preview** and hosts information regarding virtual machine instance and upcoming maintenance events. Service continues to receive updates and this page reflects the specific [data categories](#instance-metadata-data-categories) available.


## Retrieving instance metadata 

Instance metadata is available for running VMs created/managed using [Azure Resource Manager](https://docs.microsoft.com/en-us/rest/api/resources/?redirectedfrom=MSDN). 
Access all data categories for an instance use the following URI

```
$ curl -H Metadata:true http://169.254.169.254/metadata/instance?api-version=2017-03-01
```

The default output for all instance metadata is of json format(content type Application/JSON)

## Usage Examples
Following are set of examples and usage semantics for instance metadata service

### Versioning 
Instance metadata service is versioned. Versions are mandatory and the current preview version is 2017-03-01.


```
curl -H Metadata:true http://169.254.169.254/metadata/instance?api-version=2017-03-01
```

As we add newer versions, earlier version is made available in case your scripts has dependencies on data formats. Note, current preview version may not be available once the service is generally available


### Data output
By default instance metadata returns data in JSON (content type=application/json).Different node elements can return data in different default format as applicable, following table is a quick reference for data formats 

Element | default data format | Other formats
--------|---------------------|--------------
/instance | Json | text
/scheduledevents | Json | None

For text format use format=text in the request URL, for example 

```
curl -H Metadata:true http://169.254.169.254/metadata/instance?api-version=2017-03-01\&format=text
```
### Security
Instance metadata endpoint is accessible only from within the running instance on a non-routable 169.254.169.254 IP address. In addition, any request with **X-Forwarded-For header** is rejected by the metadata service. 
We also require requests to contain **Metadata:true** header being sent to ensure that the actual request was directly intended and not a part of unintentional redirection. 
### Error
If there is a data element not found or malformed requests Metadata Service returns standard HTTP Error, following are the few examples of return codes 

HTTP Return Code | Reason
----------------|-------
200 | OK
400 | Bad Request, missing header, pass -H Metadata:true
404 | Not Found, requested element does’t exist 
405 | Method not supported 
429 | Too many requests, currently we only support maximum of 5 queries per second

### Examples
#### Retrieving the network information 


```
curl -H Metadata:true http://169.254.169.254/metadata/instance/network?api-version=2017-03-01

{
  "interface": [
    {
      "ipv4": {
        "ipaddress": [
          {
            "ipaddress": "10.0.0.4",
            "publicip": "<>.<>.<>.<>"
          }
        ],
        "subnet": [
          {
            "address": "10.0.0.0",
            "dnsservers": [
              {
                "ipaddress": "10.0.0.2"
              },
              {
                "ipaddress": "10.0.0.3"
              }
            ],
            "prefix": "24"
          }
        ]
      },
      "ipv6": {
        "ipaddress": []
      },
      "mac": "000D3A00FA89"
    }
  ]
}
```

#### Retrieving public IP address

```
curl -H Metadata:true "http://169.254.169.254/metadata/instance/network/interface/0/ipv4/ipaddress/0/publicip?api-version=2017-03-01&format=text"
```

#### Retrieving all instance metadata 

```
curl -H Metadata:true http://169.254.169.254/metadata/instance?api-version=2017-03-01

{
"compute": {
    "location": "CentralUS",
    "name": "IMDSCanary",
    "offer": "RHEL",
    "osType": "Linux",
    "platformFaultDomain": "0",
    "platformUpdateDomain": "0",
    "publisher": "RedHat",
    "sku": "7.2",
    "version": "7.2.20161026",
    "vmId": "5c08b38e-4d57-4c23-ac45-aca61037f084",
    "vmSize": "Standard_DS2"
  },
  "network": {
    "interface": [
      {
        "ipv4": {
          "ipaddress": [
            {
              "ipaddress": "10.0.0.4",
              "publicip": "X.X.X.X"
            }
          ],
          "subnet": [
            {
              "address": "10.0.0.0",
              "dnsservers": [
                {
                  "ipaddress": "10.0.0.2"
                },
                {
                  "ipaddress": "10.0.0.3"
                }
              ],
              "prefix": "24"
            }
          ]
        },
        "ipv6": {
          "ipaddress": []
        },
        "mac": "000D3A00FA89"
      }
    ]
  }
}

```


## Instance Metadata data categories
Following table has a list of all data categories available via Instance Metadata

Data | Description
-----|------------
location | Azure Region the VM is running 
name | Name of the VM 
offer | Offer information for the VM image, these values are present only for images deployed from Azure image gallery 
publisher | Publisher of the VM image
sku | Specific SKU for the VM image  
version | Version of the VM Image 
osType | Linux or Windows 
platformUpdateDomain |  [Update domain](virtual-machines-windows-manage-availability.md) the VM is running in. 
platformFaultDomain | [Fault domain](virtual-machines-windows-manage-availability.md) the VM is running in.
vmId | Unique identifier for the VM, more info [here](https://azure.microsoft.com/en-us/blog/accessing-and-using-azure-vm-unique-id/)
vmSize | [VM size](virtual-machines-windows-sizes.md)
ipv4/Ipaddress | Local IP address of the VM 
ipv4/publicip | Public IP address for the Instance 
subnet/address | Address for subnet 
subnet/dnsservers/ipaddress1 | Primary DNS server
subnet/dnsservers/ipaddress2 | Secondary DNS server
subnet/prefix | Subnet prefix, example 24
ipv6/ipaddress | IPv6 address for the VM
mac | VM mac address 
scheduledevents | see [scheduledevents](virtual-machines-scheduled-events.md)

## Availability 
The current Preview is available in **West US Central** Region of Azure. 
For trying out Instance Metadata Service, create a VM from [Azure Resource Manager](https://docs.microsoft.com/en-us/rest/api/resources/?redirectedfrom=MSDN) or [Azure portal](http://portal.azure.com) and follow the examples in the following section access metadata service. 


## Example Scenarios for usage  

### Tracking VM running on Azure 

As a service provider, you may require to track the number of VMs running your software or have agents that need to track uniqueness of the VM. To be able to get a unique ID for a VM use vmId field from Instance Metadata service 

```
curl -H Metadata:true http://169.254.169.254/metadata/instance/compute/vmId?api-version=2017-03-01\&format=text

5c08b38e-4d57-4c23-ac45-aca61037f084

```

### Placement of containers, data-partitions based Fault/Update domain 

For certain scenarios where placement of different replicas is of prime important. For example [HDFS replica placement](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html#Replica_Placement:_The_First_Baby_Steps), or for container placement via an [orchestrator](https://kubernetes.io/docs/user-guide/node-selection/) you require to know the platformFaultDomain and platformUpdateDomain the VM is running on. You can query this data point directly via instance metadata

```
curl -H Metadata:true http://169.254.169.254/metadata/instance/compute/platformFaultDomain?api-version=2017-03-01\&format=text 

0
```

### Getting more information about the VM during support case 

As a service provider you may get a support call where you would like to know more information about the VM. Asking the customer to share the compute metadata can provide basic information for the support professional to know about the kind of VM on Azure. 


```
curl -H Metadata:true http://169.254.169.254/metadata/instance/compute?api-version=2017-03-01

{
"compute": {
    "location": "CentralUS",
    "name": "IMDSCanary",
    "offer": "RHEL",
    "osType": "Linux",
    "platformFaultDomain": "0",
    "platformUpdateDomain": "0",
    "publisher": "RedHat",
    "sku": "7.2",
    "version": "7.2.20161026",
    "vmId": "5c08b38e-4d57-4c23-ac45-aca61037f084",
    "vmSize": "Standard_DS2"
  }

}
```

## FAQ
1. I am getting Bad request. Required metadata header not specified?
    ##### Instance Metadata Service requires header of Metadata:true to be passed in the request. Passing header allows access
2. I am not getting compute information for my VM?
    ##### Currently Instance Metadata Service supports Azure Resource Manager created instances only, in future we may add support for Cloud Services VMs 
3. I created my Virtual Machine through Azure Resource Manager a while back, I do not see compute metadata information?
    ##### For any VMs created after Sep 2016, add a [Tag](../azure-resource-manager/resource-group-using-tags.md) to start seeing compute metadata. For older VM (created before Sep 2016), add/remove extensions or data disks to the VM to refresh metadata 
4. I am getting 500 - Internal server error?
    ##### Currently Instance Metadata Preview is available only in West US Central Region. Deploy your VMS in the supported regions.  
4. Additional questions/comments?
    ##### Send your comments on feedback.azure.com 
    
## Next Steps

Learn more about [scheduledevents] (virtual-machines-scheduled-events.md)
