---
layout:     post 
title:      "Overview to Global Secure Access"
subtitle:   "Global Secure Access is a comprehensive solution for secure network access."
date:       2024-07-30
author:     "Chris Brumm"
URL:        "/2024/07/30/Overview-to-Global-Secure-Access/"
#image:      "/post/2024-07-30-Overview-to-Global-Secure-Access/images/GSA-Overview.png"
tags:
    - Entra
    - Global Secure Access
    - ZTNA
categories: [ Global Secure Access ]
---

# Overview to Global Secure Access

A few days ago, Microsoft announced that Global Secure Access is now generally available. Since I have been working with the product for some time now and more and more proof of concepts are being launched, it is high time for me to do a blog series about it.

Here is an overview of the parts (planned so far):

1. [Overview to Global Secure Access](https://chris-brumm.com/2024/07/30/Overview-to-Global-Secure-Access/)
2. [Global Secure Access in Conditional Access](https://chris-brumm.com/2024/08/06/Global-Secure-Access-in-Conditional-Access/)
3. Deep Dive DNS in Entra Private Access
4. Deep Dive SSO in Entra Private Access 

# What is Global Secure Access?

Microsoft calls Global Secure Access (GSA) its Secure Service Edge (SSE) solution. SSE is a term created by Gartner and SSE solutions include - preferably as a cloud solution - the provision of security functions for users and user device traffic.

In addition, broad platform support and good integration with both the identity provider (IDP) and common security solutions such as XDR and SIEM are expected. Gartner also defines the components Secure Web Gateway (SWG), Zero Trust Network Access (ZTNA) and Cloud App Security Broker (CASB) as components of an SSE solution.

***Global Secure Access has all these features and I would like to give you an overview of the various functions in this blog.***

![Untitled](/post/2024-07-30-Overview-to-Global-Secure-Access/images/GSA-Overview.png)

>💡 Of course, Microsoft is keeping Microsoft Defender for Cloud Apps (MDA) as a CASB and is not building this function into Global Secure Access again. An integration between MDA and GSA has been announced but is not yet concrete at this time.

## Isn't that actually called SASE?

> SSE can be seen as the little brother of Secure Access Service Edge (SASE). However, while SSE is limited to the connection of user devices, SASE also includes functions for site networking such as SD-WAN that allow bandwidths to be controlled/optimized.
> 

## Activation and exceptions

The Microsoft documentation contains very good deployment guides, so I will only show a few excerpts here to give you a first impression. 

The different services are activated via traffic forwarding profiles. In addition to a variety of other settings (which currently vary for each profile), the profiles can be assigned to individual users or non-nested security groups.

> 💡 The profiles are processed in the following order:
Microsoft 365 access profile → Private access profile → Internet access profile
   

![[https://learn.microsoft.com/en-us/entra/architecture/media/sse-deployment-guide-intro/traffic-forwarding-profile-enabled-expanded.png#lightbox](https://learn.microsoft.com/en-us/entra/architecture/media/sse-deployment-guide-intro/traffic-forwarding-profile-enabled-expanded.png#lightbox)](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Traffic-Forwarding.png)

[https://learn.microsoft.com/en-us/entra/architecture/media/sse-deployment-guide-intro/traffic-forwarding-profile-enabled-expanded.png#lightbox](https://learn.microsoft.com/en-us/entra/architecture/media/sse-deployment-guide-intro/traffic-forwarding-profile-enabled-expanded.png#lightbox)

Which traffic is acquired for the Microsoft 365 profile is controlled via a (configurable) list managed by Microsoft and the selection for Private Access is made via the app segments configured there.

After activating the Internet Access Traffic Forwarding Profile for a user, all HTTP/HTTPs packets are routed through Global Secure Access. Microsoft maintains a default bypass list with private IP ranges and some Microsoft FQDNs and it is possible to create your own entries. 

![Untitled](/post/2024-07-30-Overview-to-Global-Secure-Access/images/EIA-policies.png)

# Microsoft Entra Internet Access

Microsoft Entra Internet Access is an identity-centric, device-aware, cloud-delivered Secure Web Gateway (SWG) that secures access to both the Internet and SaaS services such as Microsoft 365.

In addition to the typical features such as web filtering based on categories as a baseline or on a user or group basis, the solution is characterized by strong integration with Conditional Access and good log integration with Entra and Office 365.

## Microsoft Entra Internet Access for all apps

For non-M365 traffic, the web content filter is currently the relevant use case. We have a [broad selection of categories](https://learn.microsoft.com/en-us/entra/global-secure-access/reference-web-content-filtering-categories), can build very flexible policies and assign them as session control via conditional access.

> 💡 You can find a detailed description of the configuration in my blog on the Conditional Access integration of GSA.

## Microsoft Entra Internet Access for M365

> [Use of the Microsoft traffic profile is included with the Secure Access Essentials license, which will soon be included in the Microsoft 365 E3 license.](https://learn.microsoft.com/en-us/entra/global-secure-access/overview-what-is-global-secure-access#:~:text=Use%20of%20the%20Microsoft%20traffic%20profile%20is%20included%20with%20the%20Secure%20Access%20Essentials%20license%2C%20which%20will%20soon%20be%20included%20in%20the%20Microsoft%20365%20E3%20license)
> 

Microsoft provides its own traffic profile for accessing M365 not only for licensing purposes but also because there are some interesting unique use cases for M365 that I will only briefly discuss here, with the intention of examining them in more detail in one of the next parts:

### Tenant Restriction

In simple terms, this allows a client to be bound to a tenant as a data loss prevention measure. 

After setting up a policy (1), a login (3) with a user from a tenant that has not been authorized in the policy cannot be made from a managed device (2). Entra ID will block the requests while Token request at the authentication plane (4) and the M365 services will block copied external token from outside the network at the data plane (5).

![Source: [https://learn.microsoft.com/en-us/entra/global-secure-access/media/how-to-universal-tenant-restrictions/tenant-restrictions-v-2-universal-tenant-restrictions-flow.png](https://learn.microsoft.com/en-us/entra/global-secure-access/media/how-to-universal-tenant-restrictions/tenant-restrictions-v-2-universal-tenant-restrictions-flow.png)](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Tenant-Restriction.png)

Source: [https://learn.microsoft.com/en-us/entra/global-secure-access/media/how-to-universal-tenant-restrictions/tenant-restrictions-v-2-universal-tenant-restrictions-flow.png](https://learn.microsoft.com/en-us/entra/global-secure-access/media/how-to-universal-tenant-restrictions/tenant-restrictions-v-2-universal-tenant-restrictions-flow.png)

>💡 Especially in combination with the (Outbound) Guest Restriction controls and the other filter capabilities of Entra Internet Access this is a very powerful Data Loss Prevention Feature.



### Source IP Restoration

Source IP Restoration ensures that the public IP address of the client is displayed in the Entra sign-in logs instead of the address of the Secure Web Gateway to ensure that the usual problems with Secure Web Gateways do not occur with Entra Internet Access.

The first thing that comes to mind here is of course Entra Identity Protection, as with the other popular solutions the accesses do not arrive at Entra and the service with the public IP address of the end device, but with an IP address of the service provider. If all relevant connections are not excluded from the proxy configuration, this always leads to problems, like:

- alarms such as [Atypical Travel](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-risks#atypical-travel) or [Impossible Travel](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-risks#impossible-travel), as the user comes from several places at the same time.
- User irritation, as the [Additional Context](https://learn.microsoft.com/en-us/entra/identity/authentication/how-to-mfa-additional-context) feature cannot be used in the Authenticator app.
- difficult situations during troubleshooting, due to the modification of the IP address.

>💡  Why is this a problem at all? Why can't the proxy simply set the [X-Forwarded-For](https://en.wikipedia.org/wiki/X-Forwarded-For) header, for example?

In Identity Protection (and often also in Conditional Access), the IP address is security-relevant information and a high level of trust is required. At the moment, Microsoft can only ensure this trust relationship within its own infrastructure and therefore has a unique selling point with this feature!



### M365 Log Enrichment

In addition to the sign-in logs, the various audit logs are of course the most important source of information for finding out who accessed what, when and under what circumstances, and in terms of end users, the M365 Audit Logs are by far the most important here.

With regard to the question with which devices and under which circumstances the accesses were made, it is very good news that Microsoft offers a log enrichment with Global Secure with which the M365 audit logs (so far only for Sharepoint) are enriched with the following:

- Improved latency
- Additional information added to original logs
- Accurate IP address

In summary, this feature provides us with more detailed audit logs more quickly in relation to access to the M365 service.

>💡 A more detailed overview can be found in the [MS documentation](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-view-enriched-logs) and [this blog](https://www.invictus-ir.com/news/the-mystery-of-the-enrichedoffice365auditlogs-solved).



# Microsoft Entra Private Access

Microsoft Private Access is an identity-centric, device-aware, cloud-delivered Zero Trust Network Access, which enables access to all non-public networks in data centers or IaaS/PaaS. Clients can only be connected via an agent on the user device.

![Source: [https://techcommunity.microsoft.com/t5/image/serverpage/image-id/500458i7281440D06C510D1/image-size/large?v=v2&px=999](https://techcommunity.microsoft.com/t5/image/serverpage/image-id/500458i7281440D06C510D1/image-size/large?v=v2&px=999) ](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Entra-Private-Access.png)

Source: [https://techcommunity.microsoft.com/t5/image/serverpage/image-id/500458i7281440D06C510D1/image-size/large?v=v2&px=999](https://techcommunity.microsoft.com/t5/image/serverpage/image-id/500458i7281440D06C510D1/image-size/large?v=v2&px=999) 

## Comparison to a VPN

In my opinion, the biggest advantage over a VPN solution is the integration with Conditional Access, which allows each individual session to be checked by its engine and the level of access control on M365 to be transferred to each network session.

To access the private networks, Windows Server-based connectors are combined into groups and distributed to suitable locations in the network, which establish an outgoing HTTPs connection to Entra and tunnel the traffic. 

In contrast to classic VPN solutions, there is no IP address pool that is assigned by the gateway or via DHCP. Instead, all sessions are established by the connectors from the perspective of the resources.

Another important difference to VPN solutions is that the client's connections do not terminate at the gateways in one or more of the data centers, but in Microsoft's globally distributed network and the Entra Private Network Connectors configured for the target network establish a connection (also outgoing HTTPs) to the Microsoft network to fetch the packets. This allows you to transparently integrate destinations from different environments that are not connected to the headquarters.

>💡 If you have already worked with the Entra App Proxy in the past, you will recognize the connector used. With version [1.5.3829.0 from 2.4.2024](https://learn.microsoft.com/en-us/entra/global-secure-access/reference-version-history#version-1538290), the App proxy connector becomes the Entra private network connector and, in addition to the customization of various names (of files, processes, folders, ...), also includes UDP and DNS support, which is currently in preview and makes Entra Private Access really usable for most people.



Here is a non-exhaustive list of the differences:

| **Topic** | **VPN** | **Entra Private Access** |
| --- | --- | --- |
| What is authenticated? | The complete VPN tunnel | Every connection to an App Segment |
| How is authentication done? | Variies by vendor, model and environment. Radius or SAML are frequently used. | OIDC / OAuth2 with Entra ID |
| How can Sessions from compromised users/devices been revoked? | Via GUI. Maybe the VPN appliance offers an API and you can build a runbook. | Continuous Access Revokation support is already included on the diagrams and will follow. |
| Where is the connection terminated? | At gateways in one or more of the data centers. | In Microsoft's globally distributed network. |
| Where does the session come from from the perspective of the resource? | Usually an IP address from a dedicated pool is assigned to the client. | The sessions are originated from the Entra Network Connector. |
| How can we integrate multiple locations/environments? | #1 Via WAN 
#2 Deployment of gateways at each location/environment and different configs for the client. (One connection at a time) | Deployment of Entra Network Connectors at each location/environment. The infrastructure is transparent to the client. |

## Representation in Entra ID

For Entra Private Access, Microsoft has defined a new type of Enterprise Application that contains network locations as app segments and can be used like all other Enterprise Apps for user assignment and can be used as a target in Conditional Access.

![Untitled](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Enterprise-Apps.png)

Each tenant contains a special app called Quick Access. This is intended for a quick start (e.g. as a VPN replacement) in which all relevant networks are entered and the app segments are successively moved to dedicated enterprise apps in order to control access more granularly. An Application Discovery section has already been prepared in the portal to support this workflow and the Quick Access app is the only one allowed to overlap with other apps.

>💡 You can find a detailed description how to use the Per-App access in my blog on the Conditional Access integration of GSA.



## **Private Access for on-prem users**

Not yet available, but definitely worth mentioning, is the version of Entra Private Access [already presented](https://techcommunity.microsoft.com/t5/microsoft-entra-blog/microsoft-entra-private-access-for-on-prem-users/ba-p/3905450), which links the retrieval of Kerberos tickets to the use of Entra Private Access via an agent on the domain controller and thus subjects it to the conditional access rule set, while the application traffic can take place on-prem.

For more information on this topic I can recommend this video: https://www.youtube.com/watch?v=_p4bzmPl7MY

![Untitled](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Untitled%206.png)

# Client Support

Separate agents for Windows and Android are currently available for download, but you can already see in the Entra portal that agents for Mac and iOS will also be available soon.

![Source: My lab Tenant](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Untitled%207.png)

Source: My lab Tenant

The Windows client is (currently) an agent that has to be installed separately and for most users will probably never be more than a tray icon that displays the connectivity status. For the admin the agent offers very good possibilities for client-side troubleshooting, but the real highlight is the [WAM support](https://learn.microsoft.com/en-us/entra/identity-platform/scenario-desktop-acquire-token-wam), which allows the primary refresh token of the device to be accessed and a Windows Hello or FIDO login to be passed through to Entra ID in addition to the device ID. The client thus becomes virtually invisible to the user through the SSO and we can use the full scope of conditional access.

Here is an overview of the troubleshooting features:

## Health Check

The client includes a detailed health status on configuration and connectivity. This allows you to easily detect connectivity or configuration problems. 

![Untitled](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Untitled%208.png)

>💡 If you install the client on a standard Windows client, you will immediately see some errors. The necessary steps to disable [IPv6 and secure DNS](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-install-windows-client#disable-ipv6-and-secure-dns) and [QUIC](https://learn.microsoft.com/en-us/troubleshoot/azure/entra/global-secure-access/troubleshoot-global-secure-access-client-windows-issues#:~:text=Disable%20QUIC%20in%20a%20web%20browser) can be found in the documentation.



## The effective configuration

The separate configuration for every traffic/forwarding profile is full visible at the client. While this is a managed list for the Microsoft 365 and Internet profiles the Private Access rules are the summary of some builtin rules and all configured targets in the Quick Access and the Enterprise Apps.

![Untitled](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Untitled%209.png)

As you can see, each rule has a priority. These are processed in ascending order (to find the corresponding app) and the rules from Quick Access always come last. 

>💡 Note that all configured app segments currently end up in the client configuration - regardless of whether the app is assigned to the logged-in user.



## Real-time logging for name resolution and network traffic

For interactive troubleshooting on the client, we can activate logging in Advanced Diagnostics for both hostname acquisition and network traffic and then filter or export them there. This makes it easy to see which packets were acquired by the GSA client and sent into the tunnel and which IP addresses (from the mysterious 6.6.x.x range) were used for this.

![Untitled](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Untitled%2010.png)

![Untitled](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Untitled%2011.png)

## Log-package collection

As the GSA admin is not always physically present at the client, it makes sense to be able to generate a package with the most important logs. There are currently two options for this:

- Via the "Collect logs" item in the context menu of the GSA client
- In the " Advanced log collection" section of the advanced diagnostics tool

![Untitled](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Untitled%2012.png)

![Untitled](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Untitled%2013.png)

In a direct comparison, I noticed that the advanced logs contain some additional ETL and Wireshark files.

## The Android client

On Android, the client is integrated into the Microsoft Defender for Endpoint app.

Nothing needs to be configured centrally, the user only needs to activate Global Secure Access and the same policies as for the Windows client are active.

>💡 At the moment, the range of functions (no DNS, no SSO) is still somewhat limited - I will look into it again at a later date…



![Source: [https://learn.microsoft.com/en-us/entra/global-secure-access/media/how-to-install-android-client/defender-global-secure-access-disabled.png](https://learn.microsoft.com/en-us/entra/global-secure-access/media/how-to-install-android-client/defender-global-secure-access-disabled.png)](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Untitled%2014.png)

Source: [https://learn.microsoft.com/en-us/entra/global-secure-access/media/how-to-install-android-client/defender-global-secure-access-disabled.png](https://learn.microsoft.com/en-us/entra/global-secure-access/media/how-to-install-android-client/defender-global-secure-access-disabled.png)

## What about legacy client / IOT support?

In addition to the client option, Global Secure Access offers the option of using a Site2Site IPSec VPN for Microsoft Entra Internet Access with your own routers (CPEs) with Remote Network Connectivity. Microsoft has an extensive list of [validated configurations with various vendors](https://learn.microsoft.com/en-us/entra/global-secure-access/reference-remote-network-configurations). Here is an example with a Virtual Network Gateway in Azure:

![Source: [https://learn.microsoft.com/en-us/entra/global-secure-access/media/how-to-simulate-remote-network/simulate-remote-network.png#lightbox](https://learn.microsoft.com/en-us/entra/global-secure-access/media/how-to-simulate-remote-network/simulate-remote-network.png#lightbox)](/post/2024-07-30-Overview-to-Global-Secure-Access/images/Untitled%2015.png)

Source: [https://learn.microsoft.com/en-us/entra/global-secure-access/media/how-to-simulate-remote-network/simulate-remote-network.png#lightbox](https://learn.microsoft.com/en-us/entra/global-secure-access/media/how-to-simulate-remote-network/simulate-remote-network.png#lightbox)

>💡 As of today the Remote Networks feature is only supported for Entra Internet Access for M365.



# Summary

Global Secure Access already has a lot to offer in the initial version and I am sure we will see a lot more features in the coming months. For me, this finally completes the Zero Trust story by allowing me to extend the control capabilities of Conditional Access to the network layer and the OnPrem access.

*Therefore, I will also discuss the integration of Global Secure Access in Conditional Access next.* 

## **Attribution and References**

- [Microsoft Entra Private Access: An Identity-Centric Zero Trust Network Access Solution](https://techcommunity.microsoft.com/t5/microsoft-entra-blog/microsoft-entra-private-access-an-identity-centric-zero-trust/ba-p/3905451)
- [Microsoft Entra Private Access for on-prem users
](https://techcommunity.microsoft.com/t5/microsoft-entra-blog/microsoft-entra-private-access-for-on-prem-users/ba-p/3905450)
- [Microsoft Security Service Edge now generally available
](https://techcommunity.microsoft.com/t5/microsoft-entra-blog/microsoft-security-service-edge-now-generally-available/ba-p/3847828)
- [Microsoft Entra Suite now generally available
](https://techcommunity.microsoft.com/t5/microsoft-entra-blog/microsoft-entra-suite-now-generally-available/ba-p/2520427)
- [Simplified Zero Trust security with the Microsoft Entra Suite and unified security operations platform, now generally available](https://www.microsoft.com/en-us/security/blog/2024/07/11/simplified-zero-trust-security-with-the-microsoft-entra-suite-and-unified-security-operations-platform-now-generally-available/)
- [Microsoft Entra Internet Access: An identity-centric Secure Web Gateway solution
](https://techcommunity.microsoft.com/t5/microsoft-entra-blog/microsoft-entra-internet-access-an-identity-centric-secure-web/ba-p/3922548)

> 🙏 A big thank you to [Peter Lenzke](https://www.linkedin.com/in/peter-lenzke-bb95813a/) for proofreading this blog.
