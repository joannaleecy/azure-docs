---
title: Azure Files networking considerations | Microsoft Docs
description: An overview of networking options for Azure Files.
author: khdownie
ms.service: storage
ms.topic: overview
ms.date: 07/02/2021
ms.author: kendownie
ms.subservice: files
---

# Azure Files networking considerations 
You can connect to an Azure file share in two ways:

- Accessing the share directly via the Server Message Block (SMB), Network File System (NFS), or FileREST protocols. This access pattern is primarily employed when to eliminate as many on-premises servers as possible.
- Creating a cache of the Azure file share on an on-premises server (or on an Azure VM) with Azure File Sync, and accessing the file share's data from the on-premises server with your protocol of choice (SMB, NFS, FTPS, etc.) for your use case. This access pattern is handy because it combines the best of both on-premises performance and cloud scale and serverless attachable services, such as Azure Backup.

This article focuses on how to configure networking for when your use case calls for accessing the Azure file share directly rather than using Azure File Sync. For more information about networking considerations for an Azure File Sync deployment, see [Azure File Sync networking considerations](../file-sync/file-sync-networking-overview.md).

Networking configuration for Azure file shares is done on the Azure storage account. A storage account is a management construct that represents a shared pool of storage in which you can deploy multiple file shares, as well as other storage resources, such as blob containers or queues. Storage accounts expose multiple settings that help you secure network access to your file shares: network endpoints, storage account firewall settings, and encryption in transit. 

> [!Important]  
> Azure Files supports multiple network routing options. The default option, Microsoft routing, works with all Azure Files configurations. The internet routing option does not support AD domain join scenarios or Azure File Sync.

We recommend reading [Planning for an Azure Files deployment](storage-files-planning.md) prior to reading this conceptual guide.

:::row:::
    :::column:::
        <iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/jd49W33DxkQ" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
    :::column-end:::
    :::column:::
        This video is a guide and demo for how to securely expose Azure file shares directly to information workers and apps in five simple steps. The sections below provide links and additional context to the documentation referenced in the video.
   :::column-end:::
:::row-end:::

## Applies to
| File share type | SMB | NFS |
|-|:-:|:-:|
| Standard file shares (GPv2), LRS/ZRS | ![Yes](../media/icons/yes-icon.png) | ![No](../media/icons/no-icon.png) |
| Standard file shares (GPv2), GRS/GZRS | ![Yes](../media/icons/yes-icon.png) | ![No](../media/icons/no-icon.png) |
| Premium file shares (FileStorage), LRS/ZRS | ![Yes](../media/icons/yes-icon.png) | ![Yes](../media/icons/yes-icon.png) |

## Accessing your Azure file shares
SMB Azure file shares are immediately accessible via the storage account's public endpoint with SMB 3.1.1 and SMB 3.0. This means that authenticated requests, such as requests authorized by a user's logon identity, can originate securely from inside or outside of Azure. NFS Azure file shares are only accessible through the storage account's public endpoint if the public endpoint is restricted to Azure virtual networks.

For many environments, you may wish to apply additional network configuration to their Azure file shares:

- With respect to SMB file shares, many organizations and internet service providers (ISPs) block the port that SMB uses to communicate, port 445. This practice originates from legacy security guidance about deprecated and non-internet safe versions of the SMB protocol. Although SMB 3.x is an internet-safe protocol, organizational or ISP policies may not be possible to change. 

- With respect to NFS file shares, restricted public endpoint access restricts mounts to inside of Azure only.

- Some organizations require traffic to Azure to follow a deterministic path.

### Tunneling traffic over a virtual private network or ExpressRoute
When you establish a network tunnel between your on-premises network and Azure, you are peering your on-premises network with one or more virtual networks in Azure. A [virtual network](../../virtual-network/virtual-networks-overview.md), or VNet, is similar to a traditional network that you'd operate on-premises. Like an Azure storage account or an Azure VM, a VNet is an Azure resource that is deployed in a resource group. 

Azure Files supports the following mechanisms to tunnel traffic between your on-premises workstations and servers and Azure SMB/NFS file shares:

- [Azure VPN Gateway](../../vpn-gateway/vpn-gateway-about-vpngateways.md): A VPN gateway is a specific type of virtual network gateway that is used to send encrypted traffic between an Azure virtual network and an alternate location (such as on-premises) over the internet. An Azure VPN Gateway is an Azure resource that can be deployed in a resource group along side of a storage account or other Azure resources. VPN gateways expose two different types of connections:
    - [Point-to-Site (P2S) VPN](../../vpn-gateway/point-to-site-about.md) gateway connections, which are VPN connections between Azure and an individual client. This solution is primarily useful for devices that are not part of your organization's on-premises network, such as telecommuters who want to be able to mount their Azure file share from home, a coffee shop, or hotel while on the road. To use a P2S VPN connection with Azure Files, a P2S VPN connection will need to be configured for each client that wants to connect. To simplify the deployment of a P2S VPN connection, see [Configure a Point-to-Site (P2S) VPN on Windows for use with Azure Files](storage-files-configure-p2s-vpn-windows.md) and [Configure a Point-to-Site (P2S) VPN on Linux for use with Azure Files](storage-files-configure-p2s-vpn-linux.md).
    - [Site-to-Site (S2S) VPN](../../vpn-gateway/design.md#s2smulti), which are VPN connections between Azure and your organization's network. A S2S VPN connection enables you to configure a VPN connection once, for a VPN server or device hosted on your organization's network, rather than doing for every client device that needs to access your Azure file share. To simplify the deployment of a S2S VPN connection, see [Configure a Site-to-Site (S2S) VPN for use with Azure Files](storage-files-configure-s2s-vpn.md).
- [ExpressRoute](../../expressroute/expressroute-introduction.md), which enables you to create a defined route between Azure and your on-premises network that doesn't traverse the internet. Because ExpressRoute provides a dedicated path between your on-premises datacenter and Azure, ExpressRoute may be useful when network performance is a consideration. ExpressRoute is also a good option when your organization's policy or regulatory requirements require a deterministic path to your resources in the cloud.

Regardless of which tunneling method you use to access your Azure file shares, you need a mechanism to ensure the traffic to your storage account goes over the tunnel rather than your regular internet connection. It is technically possible to route to the public endpoint of the storage account, however this requires hard-coding all of the IP addresses for the Azure storage clusters in a region, since storage accounts may be moved between storage clusters at any time. This also requires constantly updating the IP address mappings since new clusters are added all the time.

Rather than hard-coding the IP address of your storage accounts into your VPN routing rules, we recommend using private endpoints, which give your storage account an IP address from the address space of an Azure virtual network. Since creating a tunnel to Azure establishes peering between your on-premises network and one or more virtual network, this enables the correct routing in a durable way.

### Private endpoints
In addition to the default public endpoint for a storage account, Azure Files provides the option to have one or more private endpoints. A private endpoint is an endpoint that is only accessible within an Azure virtual network. When you create a private endpoint for your storage account, your storage account gets a private IP address from within the address space of your virtual network, much like how an on-premises file server or NAS device receives an IP address within the dedicated address space of your on-premises network. 

An individual private endpoint is associated with a specific Azure virtual network subnet. A storage account may have private endpoints in more than one virtual network.

Using private endpoints with Azure Files enables you to:
- Securely connect to your Azure file shares from on-premises networks using a VPN or ExpressRoute connection with private-peering.
- Secure your Azure file shares by configuring the storage account firewall to block all connections on the public endpoint. By default, creating a private endpoint does not block connections to the public endpoint.
- Increase security for the virtual network by enabling you to block exfiltration of data from the virtual network (and peering boundaries).

To create a private endpoint, see [Configuring private endpoints for Azure Files](storage-files-networking-endpoints.md).

### Private endpoints and DNS
When you create a private endpoint, by default we also create a (or update an existing) private DNS zone corresponding to the `privatelink` subdomain. Strictly speaking, creating a private DNS zone is not required to use a private endpoint for your storage account, but it is highly recommended in general and explicitly required when mounting your Azure file share with an Active Directory user principal or accessing from the FileREST API.

> [!Note]  
> This article uses the storage account DNS suffix for the Azure Public regions, `core.windows.net`. This commentary also applies to Azure Sovereign clouds such as the Azure US Government cloud and the Azure China cloud - just substitute the the appropriate suffixes for your environment. 

In your private DNS zone, we create an A record for `storageaccount.privatelink.file.core.windows.net` and a CNAME record for the regular name of the storage account, which follows the pattern `storageaccount.file.core.windows.net`. Since your Azure private DNS zone is connected to the virtual network containing the private endpoint, you can observe the DNS configuration when by calling the `Resolve-DnsName` cmdlet from PowerShell in an Azure VM (alternately `nslookup` in Windows and Linux):

```powershell
Resolve-DnsName -Name "storageaccount.file.core.windows.net"
```

For this example, the storage account `storageaccount.file.core.windows.net` resolves to the private IP address of the private endpoint, which happens to be `192.168.0.4`.

```Output
Name                              Type   TTL   Section    NameHost
----                              ----   ---   -------    --------
storageaccount.file.core.windows. CNAME  29    Answer     csostoracct.privatelink.file.core.windows.net
net

Name       : storageaccount.privatelink.file.core.windows.net
QueryType  : A
TTL        : 1769
Section    : Answer
IP4Address : 192.168.0.4


Name                   : privatelink.file.core.windows.net
QueryType              : SOA
TTL                    : 269
Section                : Authority
NameAdministrator      : azureprivatedns-host.microsoft.com
SerialNumber           : 1
TimeToZoneRefresh      : 3600
TimeToZoneFailureRetry : 300
TimeToExpiration       : 2419200
DefaultTTL             : 300
```

If you run the same command from on-premises, you'll see that the same storage account name resolves to the public IP address of the storage account instead; `storageaccount.file.core.windows.net` is a CNAME record for `storageaccount.privatelink.file.core.windows.net`, which in turn is a CNAME record for the Azure storage cluster hosting the storage account:

```Output
Name                              Type   TTL   Section    NameHost
----                              ----   ---   -------    --------
storageaccount.file.core.windows. CNAME  60    Answer     storageaccount.privatelink.file.core.windows.net
net
storageaccount.privatelink.file.c CNAME  60    Answer     file.par20prdstr01a.store.core.windows.net
ore.windows.net

Name       : file.par20prdstr01a.store.core.windows.net
QueryType  : A
TTL        : 60
Section    : Answer
IP4Address : 52.239.194.40
```

This reflects the fact that the storage account can expose both the public endpoint and one or more private endpoints. To ensure that the storage account name resolves to the private endpoint's private IP address, you must change the configuration on your on-premises DNS servers. This can be accomplished in several ways:

- Modifying the hosts file on your clients to make `storageaccount.file.core.windows.net` resolve to the desired private endpoint's private IP address. This is strongly discouraged for production environments, since you will need make these changes to every client that wants to mount your Azure file shares and changes to the storage account or private endpoint will not be automatically handled.
- Creating an A record for `storageaccount.file.core.windows.net` in your on-premises DNS servers. This has the advantage that clients in your on-premises environment will be able to automatically resolve the storage account without needing to configure each client, however this solution is similarly brittle to modifying the hosts file because changes are not reflected. Although this solution is brittle, it may be the best choice for some environments.
- Forward the `core.windows.net` zone from your on-premises DNS servers to your Azure private DNS zone. The Azure private DNS host can be reached through a special IP address (`168.63.129.16`) that is only accessible inside virtual networks that are linked to the Azure private DNS zone. To workaround this limitation, you can run additional DNS servers within your virtual network that will forward `core.windows.net` on to the Azure private DNS zone. To simplify this set up, we have provided PowerShell cmdlets that will auto-deploy DNS servers in your Azure virtual network and configure them as desired. To learn how to set up DNS forwarding, see [Configuring DNS with Azure Files](storage-files-networking-dns.md).

## Storage account firewall settings
A firewall is a network policy which controls which requests are allowed to access the public endpoint for a storage account. Using the storage account firewall, you can restrict access to the storage account's public endpoint to certain IP addresses or ranges or to a virtual network. In general, most firewall policies for a storage account will restrict networking access to one or more virtual networks. 

There are two approaches to restricting access to a storage account to a virtual network:
- Create one or more private endpoints for the storage account and restrict all access to the public endpoint. This ensures that only traffic originating from within the desired virtual networks can access the Azure file shares within the storage account.
- Restrict the public endpoint to one or more virtual networks. This works by using a capability of the virtual network called *service endpoints*. When you restrict the traffic to a storage account via a service endpoint, you are still accessing the storage account via the public IP address.

> [!NOTE]
> NFS file shares can only access the storage account's public endpoint via virtual networks. NFS shares may freely access the storage account's private endpoints.

To learn more about how to configure the storage account firewall, see [configure Azure storage firewalls and virtual networks](../common/storage-network-security.md?toc=%2fazure%2fstorage%2ffiles%2ftoc.json).

## See also
- [Azure Files overview](storage-files-introduction.md)
- [Planning for an Azure Files deployment](storage-files-planning.md)
