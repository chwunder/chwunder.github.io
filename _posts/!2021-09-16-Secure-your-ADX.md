---
layout: post
title: use a Private Endpoint for your ADX cluster
subtitle: use private endpoints to secure your cluster
thumbnail-img: /assets/img/20210901/thumb.png
cover-img: /assets/img/20210901/banner.png
share-img: /assets/img/20210901/banner.png
tags: [Azure, Time Series Insights, TSI, REST API]
comments: true
---


FRAGEN:

- Create managed private endpoint
PUT https://management.azure.com/subscriptions/39f0d3e1-bdc1-42b1-b970-0b54d1822a83/resourceGroups/ADXDemo/providers/Microsoft.Kusto/Clusters/adxdemoblog/managedPrivateEndpoints/adxdemoblogmpe?api-version=2021-01-01

{
    "properties": {
        "privateLinkResourceId": "/subscriptions/39f0d3e1-bdc1-42b1-b970-0b54d1822a83/resourceGroups/ADXDemo/providers/Microsoft.EventHub/namespaces/adxdemoehns",
        "groupId": "namespace",
        "requestMessage": "Please Approve."
    }
}



- die zwei Storagfe Account sind quasi ein cache?
- Screenshot network isolation nochmal mit DNS private zone machen



Keeping your data secure in the cloud should be one of the most prioritized task of your IT department. Having an ADX cluster in the cloud which processes, stores and analyzes your business critical information should be securely operated.
Microsoft offers, currently to the time of writing this post, a private preview which allows you to create a private endpoint, a managed private endpoint in the backbone virtual network of your ADX cluster and allowing you to completely turn off the public endpoint of your cluster. 

Henning was kind enough to enable my subscription for this private preview so I can share my finding with you. This blogpost should provide you with an common understanding how you can use those features making your ADX clusters even more secure!

The final integration of the private endpoint in ADX will probably look like what we know from other services like storage, cosmos db, ... 
{: .alert .alert-warning}

This private preview is going to be configured differently from what I suspect you will see later in the portal or via CLI. The steps I have to do manually will be all integrated in an dedicated tab in the service! Only the end result will look similar. 

OK, so lets get started ....

# creation of an ADX cluster

before creating a private endpoint for your ADX cluster, you obviously need the ADX cluster itself. Later in the post i will cover a few more tricks on how to make your ADX even more secure, but for the purpose of the demo, I will just use a standard ADX cluster deployment via the portal!

For this purpose you do not need to configure network isolation.

![](/assets/img/20210916/adxcreation.png) 

Once your cluster got created, you should see your resource in the portal!

![](/assets/img/20210916/adxcreation2.png)

# secure ADX and enable a private endpoint

Like I said in the beginning, your implementation of ADX private endpoint will look differently, the end result will be the same!

## creating a private endpoint

creating a private endpoint for ADX will create a dedicated network interface for this auzre service in your VNet. This will enable clients / consumers to securely connect to this service. Network traffic between azure services traverses over the vnet and the private link on the Microsoft backbone network, eliminating exposure from the public internet.

![](/assets/img/20210916/peadx.png)

The resource type *Microsoft.Kusto/clusters* is the private preview feature which is not available for you as of now!

![](/assets/img/20210916/peadx2.png)

After all resources got provisioned, you will see those in the portal. In the next screenshot we compare the newly created resources against the resources created when doing network isolation.

![](/assets/img/20210916/adxpeenabled.png)

Compared to all the resources we get for network isolation, it looks a lot cleaner.

![](/assets/img/20210916/adxcomparenetworkisolation.png)

 # taking a more closer look at those provisioned resources

Lets take a closer look at those provisioned resources to understand how those additional resources interconnect to each other and where you can use levers to strengthen the clusters security!

## network interface card

Like with any other private endpoint you will get a dedicated network interface card, bound to the private endpoint of your selected service type. This will ensure that your service will receive a private IP address of the network you have selected. Instead of having just a public endpoint, your service now has already a foot in your network. But this is for sure not the only thing to get all working.

## private DNS zone

It is quite important to correctly configure your DNS settings to resolve the private IP address to the FQDN of the connection string of your service.

Before using the private preview, in case we wanted to use private links with ADX, you needed to configure the DNS settings on your own. With the private preview and later in the product, you do not need to take care about this at all. (In case you stick to the standard configuration) 
<br>For the purpose of the demo I just wanted to explain the single resources to better understand the process!

![](/assets/img/20210916/dnszonesrg.png)

As you can see, Azure created four private DNS zones for us. The names already reveal what other services are used in the backend. Blob, Table and Queue obviously belong to an azure storage account service. The reason why those are listed there is being discussed later in the blogpost. The other private DNS zone belongs to the actual ADX service itself.

The reason why the private endpoint created those private DNS zones is that you do not change your consumers connection strings to automatically resolve the private IP address of the service.<br>
First Azure will create a CNAME record in the public DNS zone to redirect the resolution to the private domain name. You can see to which private DNS zones azure will configure the CNAME record.
[Azure Services DNS zone configuration](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-dns#azure-services-dns-zone-configuration)
<br>
<br>Now your request e.g. adxdemoblog.westeurope.kusto.windows.net resolves to adxdemoblog.privatelink.westeurope.kusto.windows.net

This configuration is going to happen automatically, but now, your provisioned private DNS zones become functional. 

Your consumer / application now tries to query for the private endpoint IP address in azure provided DNS service. Azure DNS will be responsible for DNS resolution of the private DNS zone. You can take a closer look to the private DNS zone for the kusto service in that case.

![](/assets/img/20210916/pdnszadx.png)

This configuration shows, that there is an A-Type DNS record in the private DNS zone *privatelink.westeurope.kusto.windows.net*. If your client is in the same network than the private dns zones is linked to, it can resolve the A record to its internal, private IP address.

## private endpoint

The private endpoint resource type just does the linking between the NIC address and the private DNS configuration that a private connection can be established!

# Making your ADX more secure

Once the private endpoint is in place, you have enabled to connect securely via the microsoft backbone to the service (ADX) using an internal IP address while keeping the connection strings the same.

## disabling the public endpoint
But what about disabling the public endpoint? This will acutally take your security baseline even a step further. Disabling the public endpoint will eliminate another possible attack surface. As of today, the only way to do so is via the REST API.

In order to disable the endpoint, you can use this URL to query against using the PATCH command and spefifying the below mentioned request body!

```kusto
PATCH https://management.azure.com/subscriptions/xxxxxxxxxxxxxxxxx/resourceGroups/ADXDemo/providers/Microsoft.Kusto/Clusters/adxdemoblog?api-version=2021-08-27
```
The request body should look like this
```kusto
{
    "properties": {
        "publicNetworkAccess": "Disabled"
    }
}
```
here is an example, using postman for this purpose.

![](/assets/img/20210916/adxdisablenetworkaccess.png)

Once the public endpoint is disabled, we can now take it another step further!

## managed private endpoint

What if you would like to connect your Event Hub instance securely via an internal IP address to the ADX cluster instance? What if you want to ensure that the ADX cluster will use a private endpoint and NOT the public service endpoint to fetch data from the event hub and store data securely in your ADX database?

Even though if you would provision a private endpoint of your event hub namespace into the same subnet as the private endpoint of ADX (provisioning explained above), the data fetching of ADX will use the public endpoint of your Event Hub. 

Microsoft already addressed this demand with the offering of managed private endpoints. As of now, this feature is also available as a private preview, but will for sure be released very soon publicly.

A private endpoint is NOT the same as a managed private endpoint.
{: .alert .alert-warning}

With the help of a managed private endpoint, you will enable the ADX cluster to securely access your event hub via its private endpoint. When you create an ADX cluser, Azure will provision an ADX managed virtual network in the microsoft backbone infrastructure. This Vnet is isolated and hidden to you as you just consume ADX as a PaaS resource. You can create a managed private endpoint via the REST API to provision a private endpoint in the same virtual network where the root service resources of the ADX cluster reside in. This will enable the service to access event hub privately.

The REST API call to enable the managed private endpoint, in this case eventhub, is the following:  

```kusto
PUT https://management.azure.com/subscriptions/xxxxxxxxxxxxxxxxx/resourceGroups/ADXDemo/providers/Microsoft.Kusto/Clusters/adxdemoblog/managedPrivateEndpoints/adxdemoblogmpe?api-version=2021-01-01
```

in the event body you specify the resource for which the managed private endpoint should get created. In my test scenario, i have used an eventhub instance.

```kusto
{
    "properties": {
        "privateLinkResourceId": "/subscriptions/xxxxxxxxxxxxxxxxx/resourceGroups/ADXDemo/providers/Microsoft.EventHub/namespaces/adxdemoehns",
        "groupId": "namespace",
        "requestMessage": "Please Approve."
    }
}
```

![](/assets/img/20210916/adxcreatemanagedprivateendpoint.png)

Antother step towards a more secure connection is done. Last step in this case is to approve the private endpoint. You can do this by browsing the event hub namespace and manually approve the managed private endpoint in the network section.

![](/assets/img/20210916/approvemanagedprivateendpoint.png)

If you wanna read more about managed virtual networks as well as managed private endpoints, Azure Data Factory already uses this kind of concept. To learn more, you can use this [link](https://docs.microsoft.com/en-us/azure/data-factory/managed-virtual-network-private-endpoint).

|Link|Description| 
|---|---|
|managed virtual network & managed private endpoint Azure Data Factory  |[https://docs.microsoft.com/en-us/azure/data-factory/managed-virtual-network-private-endpoint](https://docs.microsoft.com/en-us/azure/data-factory/managed-virtual-network-private-endpoint)|