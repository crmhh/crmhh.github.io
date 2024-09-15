---
layout:     post 
title:      "Deep Dive SSO in Entra Private Access"
subtitle:   "This blog post explores the deep dive into Single Sign-On (SSO) in Entra Private Access, discussing the features, configuration, and benefits of SSO with Active Directory and Entra ID."
date:       2024-09-14
author:     "Chris Brumm"
URL:        "/2024/09/14/Deep-Dive-SSO-in-Entra-Private-Access/"
tags:
    - Entra
    - Global Secure Access
    - Entra Private Access
categories: [ Global Secure Access ]
---

A few days ago, Microsoft announced that Global Secure Access is now generally available. Since I have been working with the product for some time now and more and more proof of concepts are being launched, it is high time for me to do a blog series about it.

Here is an overview of the parts (planned so far):

1. [Overview to Global Secure Access](https://chris-brumm.com/2024/07/30/Overview-to-Global-Secure-Access/)
2. [Global Secure Access in Conditional Access](https://chris-brumm.com/2024/08/06/Global-Secure-Access-in-Conditional-Access/)
3. [Deep Dive DNS in Entra Private Access](https://chris-brumm.com/2024/09/07/Deep-Dive-DNS-in-Entra-Private-Access/)
4. [Deep Dive SSO in Entra Private Access](https://chris-brumm.com/2024/09/14/Deep-Dive-SSO-in-Entra-Private-Access/) 

With the addition of both UDP and DNS support to Entra Private Access, the vast majority of scenarios that VPN has been used for in the past can be covered - including Single Sign On with Kerberos. To enable a high level of user-friendliness and security, single sign-on is an important component and Global Secure Access really has a lot to offer here!

So I decided to write a blog post on this topic to show how the different SSO features of Active Directory and Entra ID can work together in a Entra Private Access scenario. After that I will show how a KDC proxy can be used to improve the security and stability of this scenario even more and compare both deployment variants.

But before I compare the different variants, I would first like to briefly discuss the most important components. If you want to delve deeper into these basics, you should definitely read this Deep Dive blog by Fabian (which is also the foundation of this blog 🙏):

[Windows Hello for Business Cloud Trust and KDC proxy](https://cloudbrothers.info/windows-business-cloud-trust-kdc-proxy/)

## SSO for Entra ID through WAM Support

The Global Secure Access Client is able to use the [Windows Web Account Manager (WAM](https://learn.microsoft.com/en-us/entra/identity-platform/scenario-desktop-acquire-token-wam)). By using this built-in authentication broker, access to the already stored key material such as the Primary Refresh Token is possible and makes further password entries on the clients unnecessary. In addition to very good usability, the integration enables support for Windows Hello, Conditional Access, and FIDO keys.

## SSO for Active Directory through Cloud Kerberos Trust

Cloud Kerberos Trust is not a GSA component, but it is such an important piece of the puzzle for the overall construct that I would like to briefly discuss it again here.

During setup, Entra ID is registered as a read-only domain controller in the Active Directory and can therefore also issue (3) a rudimentary TGT for the user after authentication (1) based on the user's domain (2) with the PRT, which can then be exchanged for a new TGT with all SIDs at a domain controller (or a KDC proxy) (4+5). 

![Cloud-Kerberos-Trust](/post/2024-09-14-Deep-Dive-SSO-in-Entra-Private-Access/images/Cloud-Kerberos-Trust.png)

This procedure also enables SSO against AD with Entra Joined Devices without logging in with a password - i.e. with Windows Hello for Business or FIDO2. 

>💡 Of course, the Key Trust variant can also perform the same task, but I would advise anyone still using Key Trust today to migrate, as this variant is much more robust. 
Good comparison of both variants: [https://msendpointmgr.com/2023/03/04/cloud-kerberos-trust-part-1/](https://msendpointmgr.com/2023/03/04/cloud-kerberos-trust-part-1/)

## Domain Controller Connectivity

For single sign-on via Kerberos against an Active Directory, in addition to the connection to the resource, access to at least one of the domain controllers is also required. To select a suitable DC, the DC Locator Process is used, in which a prioritized list of domain controllers is first queried via DNS (1), then the IP address of the preferred DC is queried (2) and then first a so-called LDAP ping and then a NetLogon query (3) is performed to check which domain controller is the closest.

![DCLocator](/post/2024-09-14-Deep-Dive-SSO-in-Entra-Private-Access/images/DCLocator.png)

This results in the following requirements for SSO to work:

| **Protocol** | **Port** | L4-Protocol | Target | Notes |
| --- | --- | --- | --- | --- |
| DNS | 53 | UDP | DCs at the site of the connector | Optional: is usually done by the Built-In Resolver of the Quick Access App. |
| LDAP | 389 | TCP+UDP | DCs at the site of the connector | The LDAP ping takes place via UDP. LDAP itself uses TCP |
| Kerberos | 88 | TCP | DCs at the site of the connector | / |

*In addition, it must be ensured in the NRPT of the client that the subdomain _msdcs of the AD domain is resolved internally in addition to the DNS entries of the resources.*  

Here is an example in which the DC(s) are made available as a separate app segment via Entra Private Access: 

![DCPublish](/post/2024-09-14-Deep-Dive-SSO-in-Entra-Private-Access/images/DCPublish.png)

The configuration in the portal is really easy. Just create 2 new enterprise applications, give them proper names and select your connector group. For the application segments ensure that there is no overlapping in the IP ranges and configure the needed ports. 

![NewApp](/post/2024-09-14-Deep-Dive-SSO-in-Entra-Private-Access/images/NewApp.png)

![App Segment for a Domain Controller](/post/2024-09-14-Deep-Dive-SSO-in-Entra-Private-Access/images/DCAppSegment.png)

App Segment for a Domain Controller

![App Segment for a Member/Web Server](/post/2024-09-14-Deep-Dive-SSO-in-Entra-Private-Access/images/WebAppSegment.png)

App Segment for a Member/Web Server

## Extending the scenario with the KDC Proxy

Instead of direct communication with a DC, the entire configuration can also run via a proxy. This is already available on every Windows server and only needs to be configured and activated. 

### How it works

The KDC supports all common Kerberos scenarios, such as initial registration and issuance of a TGT (a), renewal of a TGT (b) or issuance of a TGS (c)

![KDCProxy](/post/2024-09-14-Deep-Dive-SSO-in-Entra-Private-Access/images/KDCProxy.png)

1. To decide whether a KDC proxy is used, the client checks whether a proxy configuration is available.
2. Then the client establishes a connection to the proxy via HTTPs (if it trusts the certificate) and
    1. logs on there to obtain a TGT
    2. renews its TGT
    3. requests a TGS
3. The proxy determines the best domain controller if needed and establishes a connection to it via Kerberos
4. The domain controller
    1. checks the credentials / certificate and issues a TGT
    2. renews valid TGTs
    3. looks up the SPN and issues a TGS
5. The respective ticket is transmitted to the KDC proxy
6. and transmitted to the client in the HTTPs session.

>💡 At this point I would like to refer once again to Fabian's blog (see above), which also describes the configuration parameter DisallowUnprotectedPasswordAuth of the KDC proxy, which makes it possible to switch off the more insecure variant (a).

### Combination with Cloud Kerberos Trust

In the schematic representation of the functionality above, I have also described the scenario for renewing the TGT (b). This scenario is used - as described above - by Cloud Kerberos Trust to be able to use SSO to AD after Windows Hello login on the client.    

The entire process is identical to the above, with the addition that the TGT issued by Entra ID is sent to the KDC proxy (4), which then establishes the connection to a suitable DC (5) and transmits the renewed TGT to the client (6). 

![Cloud-Kerberos-Trust-KDC](/post/2024-09-14-Deep-Dive-SSO-in-Entra-Private-Access/images/Cloud-Kerberos-Trust-KDC.png)

>💡 In addition to the improved user experience, we also have the opportunity to massively increase security by making use of the DisallowUnprotectedPasswordAuth configuration parameter to restrict strong login methods!

### Usage in combination with Private Access

The systems on which the Entra Private Network Connectors run are very suitable for providing the KDC proxies and there are two options for making the KDC proxies accessible to the clients.

Publishing can take place within Entra Private Access as a separate GSA app or as an app segment in an existing GSA app. This gives us the full control options of Conditional Access and can ensure, for example, that only compliant devices can establish a connection to the KDC proxy.   

![KDCAppSegment](/post/2024-09-14-Deep-Dive-SSO-in-Entra-Private-Access/images/KDCAppSegment.png)

#1 KDC proxy in Entra Private Access

Alternatively, a separate publication as an Entra App Proxy App is also possible. This must be done without pre-authentication, as the client component does not support this, but the disabling of password-based login described above means that there is only a very small attack surface. 

![KDCAppProxy](/post/2024-09-14-Deep-Dive-SSO-in-Entra-Private-Access/images/KDCAppProxy.png)

#2 KDC published via App Proxy

### How to deploy?

- For the KDC proxy config follow the guidance [here](https://cloudbrothers.info/windows-business-cloud-trust-kdc-proxy/#kdc-proxy)
- For option #1: Create an App Segment with the IPs of the KDC proxies and TCP/443
- For option #2: Publish the KDC proxies (you should have two of them) via App Proxy following the guidance [here](https://learn.microsoft.com/en-us/entra/identity/app-proxy/application-proxy-add-on-premises-application#add-an-on-premises-app-to-microsoft-entra-id) and ensure to select “Passthrough” for Pre Authentication since your Kerberos client is not able to do Pre Authentication.
    
    ![AppProxyConfig](/post/2024-09-14-Deep-Dive-SSO-in-Entra-Private-Access/images/AppProxyConfig.png)
    
- For the client config follow the guidance [here](https://cloudbrothers.info/windows-business-cloud-trust-kdc-proxy/#client) and use the (for #2 external) URLs of the KDC proxies. 
The [proxy entries are just separated by a blank character](https://admx.help/?Category=Windows_10_2016&Policy=Microsoft.Policies.Kerberos::KdcProxyServer), like:
    
    ```powershell
    Value: <https [proxy1.contoso.com](http://proxy1.contoso.com/) [proxy2.contoso.com](http://proxy2.contoso.com/) />
    ```
    
# Choosing your variant

First of all, by combining the features discussed, a solution can be implemented that combines a passwordless SSO experience for the user with a previously impossible level of security:    

- Usage of the PRT enables Device Authentication, Windows Hello and FIDO
- Device compliance and MFA enforcement through conditional access
- Continuous Access Evaluation (not yet available)
- Separate tunnels with their own authentication per app
- Restriction of access to the accessed IPs and ports per user
- No direct connection to a domain controller necessary


![SecurityFeatures](/post/2024-09-14-Deep-Dive-SSO-in-Entra-Private-Access/images/SecurityFeatures.png)

>💡 Some of the features discussed here can also be implemented with hybrid-joined clients, but due to the various necessary connections to the Active Directory, Entra joined clients are required for the final step towards extensive separation in the sense of Zero Trust Network Access.

I would like to take a closer look at the last point "No direct connection to a domain controller necessary" and compare the variants with and without KDC proxy from there. 

| **Topic** | **Domain Controller** | **Kerberos Proxy** |
| --- | --- | --- |
| Deployment | no additional components needed | At least 2 proxy instances should be configured but are easy to enable on the Entra Network Connector |
| Attack Surface | ***Connections to DCs are*** ***needed***
but can be restricted to Kerberos+LDAP | No direct connections to any Domain Controller, only HTTPs to the proxies

In addition: With the parameter DisallowUnprotectedPasswordAuth the authentication can be restricted to FIDO2, WHfB or certificate based authentication only |
| Redundancy and Scalability | Both is Builtin in AD - you don’t need to do anything | At least 2 proxy instances should be configured, but you don’t need Loadbalancer, Sync or similar… |
| Flexibility | Because of the usage of the DC locator process the local DC at the connector group can be used.  | It is only possible to deploy one Kerberos Proxy config per client |
| Stability | Timeouts because of Negative DC locator caching can happen | Since no DC Locator process is used, the solution should be more stable. |

### Timeouts because of Negative DC locator caching

A common problem with VPN and also ZTNA solutions are timeouts because of Negative DC locator caching - described in combination with Entra Private Access [in this blog by Morten Knudsen](https://mortenknudsen.net/?p=2965).
In summary, it can happen that Windows does not use Kerberos for a period of 10 minutes (by default) if the DC locator process has failed. To mitigate the problem, the time period for negative caching can be reduced.

## Summary

The SSO capabilities of and with Global Secure Access really impressed me and the agent will probably be invisible to most users. Entra Private Access fits really well into the existing architecture in many environments with Windows Hello and Cloud Kerberos Trust and allows a high level of usability and security.

## Attribution and References

- [Fabian Bader: Windows Hello for Business Cloud Trust and KDC proxy](https://cloudbrothers.info/windows-business-cloud-trust-kdc-proxy/#disallowunprotectedpasswordauth)
- [Ben Whitmore + Michael Mardahl: Simplify Windows Hello for Business SSO with Cloud Kerberos Trust – Part 1](https://msendpointmgr.com/2023/03/04/cloud-kerberos-trust-part-1/)
- [Microsoft Learn: How domain controllers are located in Windows](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/how-domain-controllers-are-located)
- [Microsoft Learn: Understand the Microsoft Entra private network connector](https://learn.microsoft.com/en-us/entra/global-secure-access/concept-connectors)
- [Microsoft Learn: Use Kerberos for single sign-on (SSO) to your resources with Microsoft Entra Private Access](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-configure-kerberos-sso)
