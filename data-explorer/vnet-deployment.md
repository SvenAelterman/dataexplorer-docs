---
title: Deploy Azure Data Explorer into your Virtual Network
description: Learn how to deploy Azure Data Explorer into your Virtual Network
author: orspod
ms.author: orspodek
ms.reviewer: basaba
ms.service: data-explorer
ms.topic: how-to
ms.date: 10/31/2019
---

# Deploy Azure Data Explorer cluster into your Virtual Network

This article explains the resources that are present when you deploy an Azure Data Explorer cluster into a custom Azure Virtual Network. This information will help you deploy a cluster into a subnet in your Virtual Network (VNet). For more information on Azure Virtual Networks, see [What is Azure Virtual Network?](/azure/virtual-network/virtual-networks-overview)

:::image type="content" source="media/vnet-deployment/vnet-diagram.png" alt-text="diagram showing schematic virtual network architecture"::: 

Azure Data Explorer supports deploying a cluster into a subnet in your Virtual Network (VNet). This capability enables you to:

* Enforce [Network Security Group](/azure/virtual-network/security-overview) (NSG) rules on your Azure Data Explorer cluster traffic.
* Connect your on-premises network to Azure Data Explorer cluster's subnet.
* Secure your data connection sources ([Event Hub](/azure/event-hubs/event-hubs-about) and [Event Grid](/azure/event-grid/overview)) with [service endpoints](/azure/virtual-network/virtual-network-service-endpoints-overview).

## Access your Azure Data Explorer cluster in your VNet

You can access your Azure Data Explorer cluster using the following IP addresses for each service (engine and data management services):

* **Private IP**: Used for accessing the cluster inside the VNet.
* **Public IP**: Used for accessing the cluster from outside the VNet for management and monitoring, and as a source address for outbound connections started from the cluster.

The following DNS records are created to access the service: 

* `[clustername].[geo-region].kusto.windows.net` (engine) `ingest-[clustername].[geo-region].kusto.windows.net` (data management) are mapped to the public IP for each service. 

* `private-[clustername].[geo-region].kusto.windows.net` (engine) `ingest-private-[clustername].[geo-region].kusto.windows.net`\\`private-ingest-[clustername].[geo-region].kusto.windows.net` (data management) are mapped to the private IP for each service.

## Plan subnet size in your VNet

The size of the subnet used to host an Azure Data Explorer cluster can't be altered after the subnet is deployed. In your VNet, Azure Data Explorer uses one private IP address for each VM and two private IP addresses for the internal load balancers (engine and data management). Azure networking also uses five IP addresses for each subnet. Azure Data Explorer provisions two VMs for the data management service. Engine service VMs are provisioned per user configuration scale capacity.

The total number of IP addresses:

| Use | Number of addresses |
| --- | --- |
| Engine service | 1 per instance |
| Data management service | 2 |
| Internal load balancers | 2 |
| Azure reserved addresses | 5 |
| **Total** | **#engine_instances + 9** |

> [!IMPORTANT]
> Subnet size must be planned in advance since it can't be changed after Azure Data Explorer is deployed. Therefore, reserve needed subnet size accordingly.

## Service endpoints for connecting to Azure Data Explorer

[Azure Service Endpoints](/azure/virtual-network/virtual-network-service-endpoints-overview) enables you to secure your Azure multi-tenant resources to your virtual network.
Deploying Azure Data Explorer cluster into your subnet allows you to setup data connections with [Event Hub](/azure/event-hubs/event-hubs-about) or [Event Grid](/azure/event-grid/overview) while restricting the underlying resources for Azure Data Explorer subnet.

> [!NOTE]
> When using EventGrid setup with [Storage](/azure/storage/common/storage-introduction) and [Event Hub](/azure/event-hubs/event-hubs-about), the storage account used in the subscription can be locked with service endpoints to Azure Data Explorer's subnet while allowing trusted Azure platform services in the [firewall configuration](/azure/storage/common/storage-network-security), but the Event Hub can't enable Service Endpoint since it doesn't support trusted [Azure platform services](/azure/event-hubs/event-hubs-service-endpoints).

## Private Endpoints

[Private Endpoints](/azure/private-link/private-endpoint-overview) allow private access to Azure resources (such as [Storage/Event Hub](vnet-endpoint-storage-event-hub.md)/Data Lake Gen 2), and use private IP from your Virtual Network, effectively bringing the resource into your VNet.
Create a [private endpoint](/azure/private-link/private-endpoint-overview) to resources used by data connections, such as Event Hub and Storage, and external tables such as Storage, Data Lake Gen 2, and SQL Database from your VNet to access the underlying resources privately.

 > [!NOTE]
 > Setting up Private Endpoint requires [configuring DNS](/azure/private-link/private-endpoint-dns), We support [Azure Private DNS zone](/azure/dns/private-dns-privatednszone) setup only. Custom DNS server isn't supported. 

## Dependencies for VNet deployment

### Network Security Groups configuration

[Network Security Groups (NSG)](/azure/virtual-network/security-overview) provide the ability to control network access within a VNet. Azure Data Explorer automatically applies the following required network security rules. For Azure Data Explorer to operate using the [subnet delegation](/azure/virtual-network/subnet-delegation-overview) mechanism, before creating the cluster in the subnet, you must delegate the subnet to **Microsoft.Kusto/clusters** .

#### Inbound NSG configuration

| **Use**   | **From**   | **To**   | **Protocol**   |
| --- | --- | --- | --- |
| Management  |[ADX management addresses](#azure-data-explorer-management-ip-addresses)/AzureDataExplorerManagement(ServiceTag) | ADX subnet:443  | TCP  |
| Health monitoring  | [ADX health monitoring addresses](#health-monitoring-addresses)  | ADX subnet:443  | TCP  |
| ADX internal communication  | ADX subnet: All ports  | ADX subnet:All ports  | All  |
| Allow Azure load balancer inbound (health probe)  | AzureLoadBalancer  | ADX subnet:80,443  | TCP  |

#### Outbound NSG configuration

| **Use**   | **From**   | **To**   | **Protocol**   |
| --- | --- | --- | --- |
| Dependency on Azure Storage  | ADX subnet  | Storage:443  | TCP  |
| Dependency on Azure Data Lake  | ADX subnet  | AzureDataLake:443  | TCP  |
| EventHub ingestion and service monitoring  | ADX subnet  | EventHub:443,5671  | TCP  |
| Publish Metrics  | ADX subnet  | AzureMonitor:443 | TCP  |
| Active Directory (if applicable) | ADX subnet | AzureActiveDirectory:443 | TCP |
| Certificate authority | ADX subnet | Internet:80 | TCP |
| Internal communication  | ADX subnet  | ADX Subnet:All Ports  | All  |
| Ports that are used for `sql\_request` and `http\_request` plugins  | ADX subnet  | Internet:Custom  | TCP  |

### Relevant IP addresses

#### Azure Data Explorer management IP addresses

> [!NOTE]
> For future deployments, use AzureDataExplorer Service Tag

| Region | Addresses |
| --- | --- |
| Australia Central | 20.37.26.134 |
| Australia Central 2 | 20.39.99.177 |
| Australia East | 40.82.217.84 |
| Australia Southeast | 20.40.161.39 |
| Brazil South | 191.233.25.183 |
| Brazil Southeast | 191.232.16.14 |
| Canada Central | 40.82.188.208 |
| Canada East | 40.80.255.12 |
| Central India | 40.81.249.251, 104.211.98.159 |
| Central US | 40.67.188.68 |
| Central US EUAP | 40.89.56.69 |
| China East 2 | 139.217.184.92 |
| China North 2 | 139.217.60.6 |
| East Asia | 20.189.74.103 |
| East US | 52.224.146.56 |
| East US 2 | 52.232.230.201 |
| East US 2 EUAP | 52.253.226.110 |
| France Central | 40.66.57.91 |
| France South | 40.82.236.24 |
| Germany West Central | 51.116.98.150 |
| Japan East | 20.43.89.90 |
| Japan West | 40.81.184.86 |
| Korea Central | 40.82.156.149 |
| Korea South | 40.80.234.9 |
| North Central US | 40.81.43.47 |
| North Europe | 52.142.91.221 |
| Norway East | 51.120.49.100 |
| Norway West | 51.120.133.5 |
| South Africa North | 102.133.129.138 |
| South Africa West | 102.133.0.97 |
| South Central US | 20.45.3.60 |
| Southeast Asia | 40.119.203.252 |
| South India | 40.81.72.110, 104.211.224.189 |
| Switzerland North | 51.107.42.144 |
| Switzerland West | 51.107.98.201 |
| UAE Central | 20.37.82.194 |
| UAE North | 20.46.146.7 |
| UK South | 40.81.154.254 |
| UK West | 40.81.122.39 |
| USDoD Central | 52.182.33.66 |
| USDoD East | 52.181.33.69 |
| USGov Arizona | 52.244.33.193 |
| USGov Texas | 52.243.157.34 |
| USGov Virginia | 52.227.228.88 |
| West Central US | 52.159.55.120 |
| West Europe | 51.145.176.215 |
| West India | 40.81.88.112 |
| West US | 13.64.38.225 |
| West US 2 | 40.90.219.23 |
| West US 3 | 20.40.24.116 |

#### Health monitoring addresses

| Region | Addresses |
| --- | --- |
| Australia Central | 191.239.64.128, 52.163.244.128 |
| Australia Central 2 | 191.239.64.128, 52.163.244.128 |
| Australia East | 191.239.64.128,52.163.244.128 |
| Australia Southeast | 191.239.160.47,52.163.244.128 |
| Brazil South | 23.98.145.105, 23.101.115.123 |
| Canada Central | 168.61.212.201, 23.101.115.123 |
| Canada East | 168.61.212.201, 23.101.115.123 |
| Central India | 23.99.5.162, 52.163.244.128 |
| Central US | 168.61.212.201, 23.101.115.123 |
| Central US EUAP | 168.61.212.201, 23.101.115.123 |
| China East 2 | 40.73.96.39 |
| China North 2 | 40.73.33.105 |
| East Asia | 168.63.212.33,52.163.244.128 |
| East US | 137.116.81.189, 52.249.253.174 |
| East US 2 | 137.116.81.189, 104.46.110.170 |
| East US 2 EUAP | 137.116.81.189, 104.46.110.170 |
| France Central | 23.97.212.5, 40.127.194.147 |
| France South | 23.97.212.5, 40.127.194.147 |
| Japan East | 138.91.19.129, 52.163.244.128 |
| Japan West | 138.91.19.129, 52.163.244.128 |
| Korea Central | 138.91.19.129, 52.163.244.128 |
| Korea South | 138.91.19.129, 52.163.244.128 |
| North Central US | 23.96.212.108, 23.101.115.123 |
| North Europe | 191.235.212.69, 40.127.194.147 |
| South Africa North | 104.211.224.189, 52.163.244.128 |
| South Africa West | 104.211.224.189, 52.163.244.128 |
| South Central US | 23.98.145.105, 104.215.116.88 |
| South India | 23.99.5.162, 52.163.244.128 |
| Southeast Asia | 168.63.173.234, 52.163.244.128 |
| UK South | 23.97.212.5, 40.127.194.147 |
| UK West | 23.97.212.5,40.127.194.147 |
| USDoD Central | 52.238.116.34 |
| USDoD East | 52.238.116.34 |
| USGov Arizona | 52.244.48.35 |
| USGov Texas | 52.238.116.34 |
| USGov Virginia | 23.97.0.26 |
| West Central US | 168.61.212.201, 23.101.115.123 |
| West Europe | 23.97.212.5, 213.199.136.176 |
| West India | 23.99.5.162, 52.163.244.128 |
| West US | 23.99.5.162, 13.88.13.50, 104.210.32.14 |
| West US 2 | 23.99.5.162, 104.210.32.14, 52.183.35.124 |

## Disable access to Azure Data Explorer from the public IP

If you want to completely disable access to Azure Data Explorer via the public IP address, create another inbound rule in the NSG. This rule has to have a lower [priority](/azure/virtual-network/security-overview#security-rules) (a higher number). 

| **Use**   | **Source** | **Source service tag** | **Source port ranges**  | **Destination** | **Destination port ranges** | **Protocol** | **Action** | **Priority** |
| ---   | --- | --- | ---  | --- | --- | --- | --- | --- |
| Disable access from the internet | Service Tag | Internet | *  | VirtualNetwork | * | Any | Deny | higher number than the rules above |

This rule will allow you to connect to the Azure Data Explorer cluster only via the following DNS records (mapped to the private IP for each service):
* `private-[clustername].[geo-region].kusto.windows.net` (engine)
* `private-ingest-[clustername].[geo-region].kusto.windows.net` (data management)

## ExpressRoute setup

Use ExpressRoute to connect on premises network to the Azure Virtual Network. A common setup is to advertise the default route (0.0.0.0/0) through the Border Gateway Protocol (BGP) session. This forces traffic coming out of the Virtual Network to be forwarded to the customer's premise network that may drop the traffic, causing outbound flows to break. To overcome this default, [User Defined Route (UDR)](/azure/virtual-network/virtual-networks-udr-overview#user-defined) (0.0.0.0/0) can be configured and next hop will be *Internet*. Since the UDR takes precedence over BGP, the traffic will be destined to the Internet.

## Securing outbound traffic with firewall

If you want to secure outbound traffic using [Azure Firewall](/azure/firewall/overview) or any virtual appliance to limit domain names, the following Fully Qualified Domain Names (FQDN) must be allowed in the firewall.

```
prod.warmpath.msftcloudes.com:443
gcs.prod.monitoring.core.windows.net:443
production.diagnostics.monitoring.core.windows.net:443
graph.windows.net:443
*.update.microsoft.com:443
login.live.com:443
wdcp.microsoft.com:443
login.microsoftonline.com:443
azureprofilerfrontdoor.cloudapp.net:443
*.core.windows.net:443
*.servicebus.windows.net:443,5671
shoebox2.metrics.nsatc.net:443
prod-dsts.dsts.core.windows.net:443
ocsp.msocsp.com:80
*.windowsupdate.com:80
ocsp.digicert.com:80
go.microsoft.com:80
dmd.metaservices.microsoft.com:80
www.msftconnecttest.com:80
crl.microsoft.com:80
www.microsoft.com:80
adl.windows.com:80
crl3.digicert.com:80
```

> [!NOTE]
> If you're using [Azure Firewall](/azure/firewall/overview), add **Network Rule** with the following properties: <br>
> **Protocol**: TCP <br> **Source Type**: IP Address <br> **Source**: * <br> **Service Tags**: AzureMonitor <br> **Destination Ports**: 443

You also need to define the [route table](/azure/virtual-network/virtual-networks-udr-overview) on the subnet with the [management addresses](#azure-data-explorer-management-ip-addresses) and [health monitoring addresses](#health-monitoring-addresses) with next hop *Internet* to prevent asymmetric routes issues.

For example, for **West US** region, the following UDRs must be defined:

| Name | Address Prefix | Next Hop |
| --- | --- | --- |
| ADX_Management | 13.64.38.225/32 | Internet |
| ADX_Monitoring | 23.99.5.162/32 | Internet |

## Deploy Azure Data Explorer cluster into your VNet using an Azure Resource Manager template

To deploy Azure Data Explorer cluster into your virtual network, use the [Deploy Azure Data Explorer cluster into your VNet](https://azure.microsoft.com/resources/templates/kusto-vnet/) Azure Resource Manager template.

This template creates the cluster, virtual network, subnet, network security group, and public IP addresses.
