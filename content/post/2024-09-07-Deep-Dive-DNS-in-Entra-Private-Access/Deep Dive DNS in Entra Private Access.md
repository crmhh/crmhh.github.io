---
layout:     post 
title:      "Global Secure Access in Conditional Access"
subtitle:   "The blog explores the challenges of DNS in Entra Private Access and provides solutions for split DNS and disconnected environments."
date:       2024-09-07
author:     "Chris Brumm"
URL:        "/2024/09/07/Deep-Dive-DNS-in-Entra-Private-Access/"
tags:
    - Entra
    - Global Secure Access
    - Entra Private Access
categories: [ Global Secure Access ]
---

# Deep Dive DNS in Entra Private Access

A few days ago, Microsoft announced that Global Secure Access is now generally available. Since I have been working with the product for some time now and more and more proof of concepts are being launched, it is high time for me to do a blog series about it.

Here is an overview of the parts (planned so far):

1. [Overview to Global Secure Access](https://chris-brumm.com/2024/07/30/Overview-to-Global-Secure-Access/)
2. [Global Secure Access in Conditional Access](https://chris-brumm.com/2024/08/06/Global-Secure-Access-in-Conditional-Access/)
3. [Deep Dive DNS in Entra Private Access](https://chris-brumm.com/2024/09/07/Deep-Dive-DNS-in-Entra-Private-Access/)
4. Deep Dive SSO in Entra Private Access 

With the extension of Entra Private Access, which introduces both UDP and DNS support, the vast majority of scenarios for which VPN was used in the past can be covered.

But as in VPN projects (at that time), one of the more challenging topics is name resolution via DNS and the fact that we can also connect disconnected environments with Entra Private Access can make it even more complex here.

Based on past projects and various discussions about the introduction of Entra Private Access in companies of different sizes and structures, I took a closer look at the topic.

After explaining how it works and architecture, I look at the two advanced scenarios "Split DNS" and "Disconnected Environments" to show which challenges currently (still) exist and compare solution approaches.

# How it works

An integral part of Private is the Quick Access app, with which Microsoft provides a quick way to configure a VPN replacement. A valid project strategy here is to configure all internal networks and then subdivide them into individual app segments in the further course.

When it comes to DNS, however, the Quick Access app gets an additional role, as DNS suffixes can only be configured there. Name resolution for the configured domains is then done by the client via the internal DNS servers. 

![quick-access-config](/post/2024-09-07-Deep-Dive-DNS-in-Entra-Private-Access/images/quick-access-config.png)

>💡 It is important to note that it is currently not possible to configure exclusions, the name resolution is done exclusively via the Connector Group configured on the Quick Access App and there is only one Quick Access App in a tenant.

On the (Windows) client, this then generates an entry in the Name Resolution Policy Table (NRPT) via the agent, which generates an entry for the suffix with the target 6.6.255.254 - which is obviously a DNS forwarder integrated by Microsoft, which then forwards the request to one of the agents in the stored connector group. The connector itself does not need a dedicated configuration, but it must be able to resolve the desired DNS record and then returns the response to the client.

The client itself does not need a way to reach the DNS server - so the DNS protocol (UDP/53) does not have to be stored in the Quick Access app.

![overview-dns-quick-access](/post/2024-09-07-Deep-Dive-DNS-in-Entra-Private-Access/images/overview-dns-quick-access.png)

>💡 In contrast to a VPN with IP address pools for the clients - as with any DNS forwarder - it is not clear from the DNS server's point of view where the DNS request originally comes from.

## Some background on the Name Resolution Policy Table

Microsoft introduced the NRPT with Server 2012, it played an important role in Direct Access and Always On VPN and is now also used in Microsoft Entra Private Access.

Without the NRPT, it is only possible to specify (two) DNS servers in the network configuration, to which the client then sends all DNS requests, with the NRPT we can define complex rule sets for the question of which DNS server should be asked for which namespace.

The entries in the NRPT contain (in addition to various legacy parameters) the target and the name servers to be used and the client chooses the best match for the selection of the correct name server.

The Global Secure Access Client creates the required rules (with the prefix GlobalSecureAccess_) when connecting and deletes them again when disconnecting.

Here is an example of how to read out the current rule set: 

```powershell
Get-DnsClientNrptRule | select Name, Namespace, NameServers, Comment | fl
```

![read-nrpt](/post/2024-09-07-Deep-Dive-DNS-in-Entra-Private-Access/images/read-nrpt.png)

>💡 The existing CMDlets contain many parameters that were relevant to Direct Access. For Entra Private Access, we technically only need the Namespace and Nameserver parameters.

### What about non-Windows devices?

I'm focusing heavily on Windows clients in this blog, but since Microsoft has announced that they want to support all platforms, here's a quick overview (which I haven't tested):

- MacOS allows you to create separate resolv.conf files for different domains and should therefore have the same range of functions as the NRPT on Windows. Here is a blog about it: https://stanislas.blog/2020/02/different-nameservers-domains-macos/
- On iOS, I also found exactly the right option in the MDM documentation with the parameter "Matching Domains": https://support.apple.com/en-gb/guide/deployment/dep86469ba99/web
- Unfortunately, I couldn't find anything about a similar configuration on Android devices so far.

# Dealing with advanced scenarios

Even with the switch from a VPN to a ZTNA solution, some DNS challenges remain and you are well advised to build up a toolbox for projects in order to be successful in heterogeneous/grown environments. 

## The challenge of split DNS

*means: the same namespace is used internally and externally*

The decisions on the names used in Active Directory were usually made a few years ago (sometimes even before the introduction of VPN 😉) and at that time it was difficult to foresee what consequences each choice would have. In any case, it is not uncommon today for internally to use the same namespace as externally (this was even very popular with Exchange and Skype at the time) and my experience says that no one is seriously considering converting to another namespace - especially not in the short term - so we have to find solutions for this.

Companies that made this decision back then usually maintain 2 zones for the same domain today to cover the following scenarios:

- FQDNs / subdomains that should only be resolvable in the public DNS
- FQDNs / subdomains that should only be resolvable in private DNS
- FQDNs / subdomains that should be resolved differently in the public DNS than in the private DNS

### Include or Exclude Approach

When handling split DNS a fundamental decision has to be made and I want to briefly discuss criteria and implications here.

*Should all requests for the domain in question be sent to the private DNS server and exceptions defined if necessary, or should only the requests for the necessary resources be included?*   

![dns-choice](/post/2024-09-07-Deep-Dive-DNS-in-Entra-Private-Access/images//dns-choice.png)

In order to follow an include approach, in which only the required FQDNs are sent to the private DNS server, I think two conditions must be met:

- There is no need to queries against the domain itself, e.g. to determine domain controllers. → This is only the case if I have Entra ID Joined Devices and do not use SSO or use a KDC proxy.
- The amount of resources accessed and thus resolved is manageable.

![split-dns-flow](/post/2024-09-07-Deep-Dive-DNS-in-Entra-Private-Access/images/split-dns-flow.png)

>💡 **Why don't we use the internal DNS servers exclusively for the entire DNS resolution?** 
This solution was popular when VPN was still operating in "full tunnel" mode and all traffic went through the data center. But I haven't seen a VPN for a long time that doesn't run in "Split Tunnel" mode, where the connections are only routed into the tunnel when needed. 
**In a ZTNA scenario, this is a no go!**

### Solution on the client: Exclusions in the NRPT

Based on the above decision, it may be necessary to make exclusions for FQDNs or subdomains that are only or otherwise resolvable in the public DNS.

To configure the NRPT, there are the following options, which can be used differently in combination with Private Access. Unfortunately, the native configuration option does not offer us the possibility to configure exclusions at the moment:

| Option | **Available** | **Remark** |
| --- | --- | --- |
| GPOs (local and AD) | ❌ | Not usable for modern Entra Joined clients |
| PowerShell CMDlets | ✅ | Universally usable, but must be included in a package, for example  |
| Intune in combination with a VPN configuration | ❌ | Is not usable for Private Access because there is no VPN profile. |
| Private Access Quick Access App  | ✅ | Is very convenient to use, but (still) has functional limitations (exclusions, different DNS servers) |

In order to solve the problem on the client side, another configuration option is (still?) needed, which is sensibly based on Powershell. Everything we need to manage the NRPT via Powershell is included in the DnsClient module, which is installed by default on every Windows client. 

Since the parameter NameServers is a mandatory field and unfortunately does not have a value that instructs the DNS client to use its default DNS resolver, it must first be determined in a somewhat cumbersome way. Here is an example:

```powershell
$NIC = (Get-NetAdapter | where {$_.Status -eq "up"})
$DNSServer = (Get-DnsClientServerAddress -InterfaceIndex $NIC.ifIndex).ServerAddresses
$Rule = @{
    'Namespace'   = 'exclusion.adatum.com'
    'Comment'     = 'App 1'
    'NameServers' = ($DNSServer)
}
Add-DnsClientNrptRule @Rule
```

### Solution on the server: Use of DNS policies

DNS policies are a rather unknown feature of Active Directory, which allows the DNS server to give different answers for a zone according to defined criteria.

- Configuration of the internal DNS for the whole domain on the client
- Creation of one zone per FQDN for exclusion (PinPoint Zones)
- Creation of a policy on the configured DNS servers - Useful criterion: Subnet of the connectors

Depending on the requested name, the following logic then results for the answers:

![nrpt-lookup-flow](/post/2024-09-07-Deep-Dive-DNS-in-Entra-Private-Access/images/nrpt-lookup-flow.png)

Here is a practical example with an ADFS server (from time to time I see one of them and offer to help with the migration to PHS) where the scenario can be shown wonderfully. While the internal client accesses the ADFS server via the private IP, the client connects to the ADFS proxy via the public IP on the Internet. 

![overview-split-dns](/post/2024-09-07-Deep-Dive-DNS-in-Entra-Private-Access/images/overview-split-dns.png)

Unfortunately, there is no graphical configuration option for DNS policies and the configuration is a bit cumbersome in my opinion, but I wrote a [small script](https://github.com/crmhh/GSAHelper/blob/2944969dd9d99f2ac3d8635390bc3715addb1a53/scripts/new-EPASplitDNSZone.ps1) that does the above necessary things for our use case.

The core is these lines:

```powershell
# Prepare the Connector-Subnet config - only one time needed
Add-DnsServerClientSubnet -Name $Connectors -IPv4Subnet "192.168.0.40/32" -PassThru
Set-DnsServerClientSubnet -Name $Connectors -Action ADD -IPv4Subnet "192.168.0.41/32" -PassThru

# Create the Zone for the FQDN
Add-DnsServerPrimaryZone -Name $ZoneName -ReplicationScope "Forest" -PassThru

# Create ZoneScope for the Zone
Add-DnsServerZoneScope -ZoneName $ZoneName -Name $ZoneScopeName -PassThru

# Add IPs for default and EPA-Exclusion
Add-DnsServerResourceRecord -ZoneName $ZoneName -A -Name "@" -IPv4Address $PrivIP -PassThru
Add-DnsServerResourceRecord -ZoneName $ZoneName -A -Name "@" -IPv4Address $PubIP -ZoneScope $ZoneScopeName -PassThru

# Set Policy for the ZoneScope
Add-DnsServerQueryResolutionPolicy -Name $ZoneName -Action ALLOW -ClientSubnet "eq,$($Connectors)" -ZoneScope "$($ZoneScopeName),1" -ZoneName $ZoneName -PassThru
```

Since the DNS policies are not stored in Active Directory, the configuration must be done on each DNS server that is stored on the connector servers (or on all of them). Since this can get a bit confusing, here is a script to read out the relevant settings per server: https://github.com/crmhh/GSAHelper/blob/2944969dd9d99f2ac3d8635390bc3715addb1a53/scripts/get-EPASplitDNSZones.ps1

### Comparison of both solutions

Since I assume that both solutions can work well and it depends on many factors and preferences, I have described both here. Here is a little orientation:

Pro Client-Side Solution:

- Microsoft has chosen this path and it can be assumed that it will be expanded.
- No adjustments to the infrastructure (individually on each DNS server) necessary
- Different configuration possible for different clients
- More transparent for the client and no dependency on the OnPrem DNS servers

Per server-side solution:

- Independent of the client OS
- Client configuration can be done completely via GSA
- No separate tool required for the NRPT config

## The Challenge of Disconnected Environments

*means: there are several network segments in which connector groups are located without a common name resolution*

Many companies have a distributed environment that has never been consolidated for historical reasons or through acquisitions and consists of several active directories. From a security perspective, I also don't think it's advisable to build trusts between environments (with different security), because the forest boundary is by no means a security boundary.

However, a common tenant as a common target environment for users, files/mails, clients and apps has proven to be a good plan (especially when using Entra Cloud Sync) and Entra Private Access with its lightweight connectors and the termination of the clients in the cloud fits very well into this architecture.

The only problem at the moment is the limitation that only the one Quick Access app can use the integrated DNS resolver, which requires that the Connector Group stored there can resolve all domains.

### Solution on the client: Extension of the NRPT

A possible solution to this problem is to include DNS servers in the app segment for all environments (which are not accessible via the Quick Access app) and to set additional entries in the NRPT.

![overview-disconnected](/post/2024-09-07-Deep-Dive-DNS-in-Entra-Private-Access/images/overview-disconnected.png)

>💡 By the way, this strategy is also great for accessing private endpoints in Azure.

Setting a rule for a (sub)domain:

```powershell
$Rule = @{
    'Namespace'   = '.adatum.com'
    'Comment'     = 'App 1'
    'NameServers' = ('10.0.1.10', '10.0.1.11')
}
Add-DnsClientNrptRule @Rule
```

### Solution on the server: Conditional Forwarder (or Stub Zone)

If a network connection from the DNS servers used by the Connector Group on the Quick Access App to the other DNS servers is possible, the problem can also be easily solved with conditional forwarders or possibly stub zones ([here is a comparison](https://www.linkedin.com/advice/1/what-differences-similarities-between-stub-zones-conditional)).   

![conditional-forwarder](/post/2024-09-07-Deep-Dive-DNS-in-Entra-Private-Access/images/conditional-forwarder.png)

### Comparison of both solutions

Even though both approaches have their advantages and disadvantages, the main criterion is whether it is possible and reasonable to establish a network connection between the DNS servers.

Pro Client-Side Solution:

- Also works for completely disconnected environments.
- No dependence of the DNS servers on each other
- No infrastructure changes necessary (establish VPN tunnels...)

Per server-side solution:

- Easy and transparent to implement
- If necessary, also helps for non Entra Private Access accesses
- Independent of the client OS
- Client configuration can be done completely via GSA
- No separate tool required for the NRPT config

## Summary

As already described in the introduction, the topic of DNS is also of great importance in ZTNA environments and poses some challenges. However, since DNS is often the basis for encryption, load balancing and authentication, we have to face these challenges. 

I hope to have made a small contribution to this with this blog by trying to show the functionality, problems and solutions - knowing full well that we do not yet have ready tools for e.g. a central NRPT configuration (maybe something for a future blog...).

Regardless of this, I am curious to see how Entra Private Access develops and makes the workarounds described above unnecessary. 

*My next blog will be a deep dive to Single Sign On in Entra Private Access*

## **Attribution and References**

- [Microsoft Learn: The NRPT](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/dn593632(v=ws.11))
- [Callan Halls-Palmer: Report on DNS Policy with PowerShell](https://hallspalmer.wordpress.com/2020/04/22/report-on-dns-policy-with-powershell/)
- [VCLOUDNINE: Setting up split DNS using Windows DNS server](https://vcloudnine.de/setting-up-split-dns-using-windows-dns-server/)
- [Microsoft Learn: Use DNS Policy for Split-Brain DNS Deployment](https://learn.microsoft.com/en-us/windows-server/networking/dns/deploy/split-brain-dns-deployment)
- [4sysops: Split-brain DNS deployment using Windows Server DNS policy](https://4sysops.com/archives/split-brain-dns-deployment-using-windows-server-dns-policy/)
- [Practical365: Use the same internal and external HTTPS names with Exchange Server](https://practical365.com/use-the-same-internal-and-external-https-names-with-exchange-server/)