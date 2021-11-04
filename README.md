#Azure Private Link integration Repository**

This repository is meant as a culmination of notes and practices that I have leveraged to help support enterprise customers leverage the Azure Private Link Service.  I will be adding collateral as it fits but wanted to focus more on implemenation and operationalization of the Azure service.
If you are not familiar with the service and why it is critical to securing (the more East - West traffice), please see the Microsoft documents here - [https://aka.ms/privatelink](https://aka.ms/privatelink)

##Implementation

###DNS
One of the biggest barriers to implementation is in understanding how DNS gets implemented to support Private Endpoints.  My recommendation is to review Daniel Mauser's repo on PrivateLink here - [Daniel Mausers Github Repo on Privatelink](https://github.com/dmauser/PrivateLink/blob/master/README.md).  This repository is extensive and covers the different DNS patterns that are not clear from the Azure docs.  I wanted to address one specific scenario by taking a deeper look at a common design pattern: **On Premise DNS forwarding traffic to Azure.  See [here ](https://github.com/dmauser/PrivateLink/tree/master/DNS-Integration-Scenarios#4-on-premises-dns-integration) for a better understanding of how this resolution works (and helps keep the communication secured between the requestor and the Endpoint).  Specifically my client had the following requirements:
- Enable on premise resources to be able to leverage Azure Private Link
- Maintain Azure DNS in Azure (using MS DNS - but this could just as well be Bind or > [!TIP]
> Optional information to help a user be more successfulAzure Private DNS Zones (recommended))
- Enable on premise resources to be able to leverage Azure Private Link (in this case Azure Backup)
- Be able to resolve Private Endpoints without traversing the Internet
- Minimize administration for the onpremise Bind team

Diagram (for what was built)
    :::image type="content" source="Az_PL_DNS_Config.png" alt-text="Example DNS query flow diagram for Azure Private Link.":::
This aligns with Daniels drawing from DNS Integration Scenario 4, the one comment to make here is that his Note on Bind, > [!NOTE]
> There are reports that BIND-based DNS Servers (including Infoblox) work using conditional forwarders towards the privatelink.PaaS-domain zone (example: privatelink.blob.core.windows.net for storage accounts) without any issues. 
Appears to not be an issue with this approach for Bind 9.

Steps to set up the resolution:
1. Network connectivity - DNS is by design minimal traffice, that being said there is definitily a consideration for security here.  Scope and accessibility must be considered in the architecture to minimize bad actors from accessing enterprise services 
2. Assuming that the Bind servers are configured (version 9+) and permitted to communicate across port 53 into the Azure DNS (more on that in a moment).  This can be tested from a bind server by performing a simple: `dig @{IP_ADDRESS_FOR_AZURE_DNS} known_host` Watch for a connectivity error like "connection timed out; no servers could be reached"
3. Offload the management on the bind servers by creating a forward lookup zone for core.windows.net.  > [!CAUTION]
> This zone is important because creating at a higher level domain (i.e. windows.net) will cause issues for DNS users/services that are trying to get to common services such as https://dotnet.microsoft.com/
Using core.windows.net covers the following FQDN Endpoint for the following services:
    blob.core.windows.net
    dfs.core.windows.net
    file.core.windows.net
    queue.core.windows.net
    table.core.windows.net
    web.core.windows.net
    database.windows.net
> [!CAUTION]
> This is not the complete list of services and may require creating individual zone forwarders for services that are NOT included in this list, such as mongo.cosmos.azure.com
Create entry in Binds - /etc/bind/named.conf.local:

```
zone "core.windows.net" {
        type forward;
        forward only;
        forwarders { ***IP_ADDRESS_FOR_AZURE_DNS***; };
    };
```
Substitute the IP_ADDRESS_FOR_AZURE_DNS for the IP address of your DNS in Azure (based by primary and secondary services)
4. DNS in Azure can be of most types. Here are some examples:
    1. [Azure Private DNS ](https://docs.microsoft.com/azure/dns/private-dns-overview)- this is the recommended approach for deployment and operations.  With Azure Private DNS, there isn't the concern of managing the infrastructure, nor managing IP address changes (because it supports dynamic DNS)
    2. DNS Forwarders - this is specifically designed to address eliminate DNS administration in Azure.  The approach is to have a device in Azure that will forward specifically to the Azure [Recursive Resolver](https://docs.microsoft.com/azure/virtual-network/virtual-networks-name-resolution-for-vms-and-role-instances#vms-and-role-instances). That resolver requires having the forwarder's request to originate from within Azure (i.e. we can't directly connect from our onprem Bind server).  This can be a standalone (or VM Scale Set) Bind forwarder (more on that in a minute), or a [container instance in docker](https://github.com/groovy-sky/azure/tree/master/docker-coredns-00#introduction), 
    or could be even [NGINX DNS proxy](https://github.com/Microsoft/PL-DNS-Proxy) - (though I am personally reticent to use NGINX in this way).
        Configuring a standalone Bind forwarder is quite straight forward, it is po
    3. [NRPT and DNS Masq](https://github.com/dmauser/PrivateLink/tree/master/DNS-Client-Configuration-Options) is another approach to minimally managed and deployed services.
    4. [MS DNS](https://github.com/dmauser/PrivateLink/tree/master/DNS-Scenario-Using-AD) and other full DNS services in Azure - The links here should be enough to address custom and configurable options out there.  Keep in mind that without leveraging [Dynamic DNS](https://docs.microsoft.com/azure/virtual-network/virtual-networks-name-resolution-ddns) - consideration should be given to [static IP addresses](https://docs.microsoft.com/azure/virtual-network/ip-services/virtual-network-network-interface-addresses#assignment-methods).
    

