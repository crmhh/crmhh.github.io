---
layout:     post 
title:      "Entra Private Access and the future of the Entra App Proxy"
subtitle:   "The blog compares Entra Private Access and Entra App Proxy and helps to decide which to use when."
date:       2025-04-06
author:     "Chris Brumm"
URL:        "/2025/04/06/Entra-Private-Access-and-the-future-of-the-Entra-App-Proxy/"
tags:
    - Entra
    - Global Secure Access
    - Entra Private Access
    - Entra App Proxy
categories: [ Global Secure Access ]
---

Since the release of Entra Private Access, I have been getting more and more questions about the future of the Entra App Proxy. Will it still be needed? Should I still use it? Are there synergies or incompatibilities?  

This blog post is dedicated to these very questions and is part of my series on Global Secure Access 

1. [Overview to Global Secure Access](https://chris-brumm.com/2024/07/30/Overview-to-Global-Secure-Access/)
2. [Global Secure Access in Conditional Access](https://chris-brumm.com/2024/08/06/Global-Secure-Access-in-Conditional-Access/)
3. [Deep Dive DNS in Entra Private Access](https://chris-brumm.com/2024/09/07/Deep-Dive-DNS-in-Entra-Private-Access/)
4. [Deep Dive SSO in Entra Private Access](https://chris-brumm.com/2024/09/14/Deep-Dive-SSO-in-Entra-Private-Access/) 
5. [Entra Private Access and the future of the Entra App Proxy](https://chris-brumm.com/2025/04/06/Entra-Private-Access-and-the-future-of-the-Entra-App-Proxy/)

## Do I still need the App Proxy?

At the moment, there is no indication that Entra Private Access will completely replace the App Proxy and I believe that it still has a right to exist.

Web applications can currently be published in two different ways.

- as an app in Entra App Proxy
- as an app(-segment) in Entra Private Access

And since both variants have their strengths in different scenarios, here is a brief comparison based on a few use cases / subject areas.

## What was the App Proxy again?

The App Proxy has been available for quite some time as part of the Entra ID Premium license. It is a reverse proxy like NGinx (or Microsoft WAP, Citrix Netscaler, Kemp LoadMaster,...) with the crucial difference that it includes a cloud component in addition to the component that is installed on one or more servers.

![app-proxy-architecture](/post/2025/Entra-Private-Access-and-the-future-of-the-Entra-App-Proxy/images/app-proxy-architecture.png)

Source: [https://learn.microsoft.com/en-us/entra/identity/app-proxy/media/what-is-application-proxy/azure-ad-application-proxy-architecture.png](https://learn.microsoft.com/en-us/entra/identity/app-proxy/media/what-is-application-proxy/azure-ad-application-proxy-architecture.png)

From the client's perspective, the pages/applications are not published at the perimeter of the data center, but in the Microsoft cloud. As the OnPrem component sets up an outgoing tunnel based on HTTPS, we do not need any incoming firewall connections from the Internet and can use Microsoft's protection mechanisms with regard to DDOS, for example.

In the field, however, I have generally not used the App Proxy for publishing websites that are freely accessible from the Internet, such as blogs and webshops, but for authenticated access as an alternative to a VPN. 

The strong integration in Entra ID and thus also Conditional Access results in some clear advantages in the area of access management, but the restriction to web servers meant that a complete replacement of a VPN was out of the question - this will only change with the use of Entra Private Access.

## Entra Private Access as a further development of the App  Proxy

Entra Private Access is a fully-fledged Zero Trust Network Access solution and is part of the Microsoft Security Service Edge solution Global Secure Access. As it offers a transparent connection for all TCP and UDP-based protocols, it is ideally suited to replace existing VPN solutions and achieve a significantly higher level of security.

![private access architecture](/post/2025/Entra-Private-Access-and-the-future-of-the-Entra-App-Proxy/images/private-access-architecture.png)

Source: [https://learn.microsoft.com/en-us/entra/global-secure-access/media/concept-private-access/private-access-diagram.png](https://learn.microsoft.com/en-us/entra/global-secure-access/media/concept-private-access/private-access-diagram.png)

Since Entra Private Access is based on the Entra App Proxy, there are some similarities that I would like to briefly describe here:

- With Entra Private Access, all sessions are also set up in the direction of the Microsoft Cloud and no incoming connections etc. are required in the data center.
- Entra Private Access also has a very strong integration with Entra ID and Conditional Access
- Both solutions use Entra Network Connectors combined into groups and can even use the same connectors and groups at the same time.

## What are the differences and when do I use what?

As described above, Entra App Proxy is a reverse proxy, i.e. limited to web applications and APIs. All connections are terminated at the proxy in order to then establish new connections to the applications to the actual target. 

Entra Private Access, on the other hand, tunnels the connections (like a VPN) and is therefore largely protocol-independent. Although UDP/TCP connections are established from the connector to the target, the sessions at protocol level (HTTP, SMB,...) are not interrupted.

***So as soon as I need a protocol other than HTTPs (such as SMB, RDP or SSH), I need Entra Private Access.*** 

### Licenses

Another important difference is the licensing: While Entra App Proxy only requires an Entra ID Premium (P1) license, Entra Private Access (additionally) also requires the license of the same name (individually or in the Entra Suite Bundle).

### Name resolution

When making your selection, it is important to understand that the two solutions work on different levels with regard to name resolution:

- With the app proxy, a website is published in Entra, which requires a publicly resolvable name (plus certificate). This can either come from a publicly resolvable custom domain of the client or a name from the [msappproxy.net](http://msappproxy.net/) domain is used. The client's default DNS server is used for resolution.
- With Entra Private Access there is no restriction on publicly resolvable names. For app segments that are configured as FQDN, the resolution is redirected to the connector and it is possible to resolve entire DNS suffixes internally in the NRPT.
    
>💡 I recommend reading the DNS Deep Dive in this series.

There are different scenarios for an application depending on the name chosen:

| Scenario | App Proxy | Entra Private Access |
| --- | --- | --- |
| **Private names**: Here the (web) application is accessible in the LAN under a private name (such as .local). | The URLs must be rewritten to a public URL (URL translation).
***Possible, but depends heavily on the application*** | Name resolution takes place at OS level (via DNS Config in Quick Access) or network level (FQDN as app segment)
***Simpler option but only available for managed devices (see B2B+BYOD)*** |
| **Split DNS:** Here, the (web) application can be accessed in the LAN under the same (publicly resolvable) name as from the Internet. Depending on the location, the name is resolved to different IP addresses. | ***This scenario works very well with the App Proxy. No URL translation needed and no limitations!*** | In some environments, the same (publicly resolvable) namespace is used both externally and internally (as the name of the Active Directory). 
***Since Private Access currently does not support exclusions in the DNS configuration this can be problematic*** |

> #### URL-Translation
> If URL translation is necessary because the application is operated with private names such as contoso.local, a few challenges arise:
> - The App Proxy can rewrite URLs both in headers and in the body. For some time now, the App Proxy has also been able to map wildcard applications with complex applications via several web servers, including CORS.
> - However, there is a list of restrictions on the supported headers and formats in the body and all components must be configured in App Proxy.
> - In order to mitigate the restrictions, Microsoft has also ensured over time that URL translation can be completely dispensed with for managed devices by using the Edge browser or the MyApps plugin.

***If the applications have publicly resolvable names, e.g. due to a split DNS scenario, and no URL translation has to take place at the reverse proxy App Proxy, this is a really good choice.***

***In my experience, a detailed check of the application to be published is absolutely essential in more complex scenarios. This requires detailed knowledge of the applications and a test environment is also a great advantage. Particularly when the applications are legacy systems, neither of these are often available and using Entra Private Access is the simpler solution.***

### Single-Sign On

The login to the Entra App Proxy is done with the browser via Entra ID with OIDC. The App Proxy also supports the following SSO scenarios for logging in to the published application: 

- Password-based sign-on
- Integrated Windows authentication
- Header-based sign-on
- SAML single sign-on

In my experience, Integrated Windows Authentication through the use of Kerberos Constrained Delegation is particularly practical, as this enables Kerberos login for platforms (such as Android and iOS) that do not support Kerberos themselves.

The app proxy is also a good choice for publishing web services completely without pre-authentication, such as a certificate revocation list.

Login to Entra Private Access also takes place via Entra ID but with the Global Secure Access Client. Entra Private Access does not offer its own SSO functions for logging on to the application, but leaves the login - through a transparent tunnel - to the client. This makes it possible, for example, to use old (NTLM) or proprietary protocols.

>💡In this series there is a part that only deals with the SSO of Entra Private Access.

***Both variants offer full conditional access support, including device authentication and phishing-resistant authentication and have their advantages in some scenarios.*** 

### B2B and BYOD

Currently, Entra Private Access does not support B2B accounts. So if I want to publish web applications that are to be accessed B2B accounts, **only the App Proxy is an option**.

Especially [in B2B scenarios](https://learn.microsoft.com/en-us/entra/external-id/hybrid-cloud-to-on-premises), I have seen (and built) environments in the past that used a combination of App Proxy, Kerberos Constrained Delegation and a script to sync B2B accounts into the OnPrem AD (https://github.com/Azure-Samples/B2B-to-AD-Sync) and thus achieved a high level of security and usability.

![B2B-2-AD](/post/2025/Entra-Private-Access-and-the-future-of-the-Entra-App-Proxy/images/B2B-2-AD.png)

>🚧In my opinion, access from unmanaged environments should be avoided as far as possible. So if you want to let guests access your environment (whether on-prem or in the cloud), take a look at one of my favorite features in Entra ID: the [cross-tenant access settings](https://learn.microsoft.com/en-us/entra/external-id/cross-tenant-access-settings-b2b-collaboration), which can be used to enforce device compliance for B2B access, for example.

In addition it is required by Entra Private Access that all accessing devices need to be registered in Entra ID and have the Global Secure Access Client installed. Since the client is not MAM-capable this means in the field for all device types (except Android with its Work Profile) the access is limited to full managed devices. 

***If you need B2B support or want to grant access to private devices App Proxy is the better option at the moment.*** 

### Logging, Monitoring and Alerting

If pre-authentication is configured, there are events in the Entra sign-in logs for access via the App Proxy. In addition, there are local logs on the Entra Network Connector, which can be ingested into Sentinel via Azure Monitoring Agent, for example. 

>💡There are no built-in dashboards, workbooks or analytics rules from Microsoft for the App Proxy and apps published via it, but it is of course feasible to build this yourself. If you - or someone you know - have done this, I would be delighted if you could get in touch with me.  

Entra Private Access is still working on this topic, but it is already clear that the result will be good. There are various built-in dashboards and pre-built workbooks, for example for usage profiling, the top used destinations or the status of the devices to get a quick overview.

![Client-Activity-Workbook](/post/2025/Entra-Private-Access-and-the-future-of-the-Entra-App-Proxy/images/Client-Activity-Workbook.png)

In addition to the sign-in logs in Entra ID, Entra Private Access offers the [NetworkAccessTraffic](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/networkaccesstraffic) table, which can be [ingested](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-view-traffic-logs) with the Entra ID logs via the Diagnostic Settings to Log Analytics / Sentinel. The log provides us with a lot of additional information that can be used for the following evaluations, for example:

- Usage & Traffic
- Investigation of anomalies or suspicious behavior
- Correlation with the MDE logs of client, connector and server

![NetworkAccessTraffic-Table](/post/2025/Entra-Private-Access-and-the-future-of-the-Entra-App-Proxy/images/NetworkAccessTraffic-Table.png)

### Performance

The topic of performance is also interesting. While the app proxy uses HTTPs for transport, Entra Private Access uses the high-performance protocol [gRPC](https://grpc.io/).

>💡For an intro to gRPC, I recommend Thomas Detzner's explanation in the 425 Show: **[Troubleshooting Deep Dive on Microsoft Entra Internet Access](https://youtu.be/-gdaqLAwVt4?t=525&si=Erh0_D9hYN-tb1NR)**

On the other hand - at the time of writing this blog - unfortunately only the App Proxy has the feature of being able to [select a dedicated region for a connector group](https://learn.microsoft.com/en-us/entra/identity/app-proxy/application-proxy-network-topology#optimize-connector-groups-to-use-closest-application-proxy-cloud-service). In some scenarios, this can make a big difference. However, based on this roadmap slide from Ignite 2024, it can be assumed that this will soon also be possible for Entra Private Access.

![Ignite-GSA-Roadmap](/post/2025/Entra-Private-Access-and-the-future-of-the-Entra-App-Proxy/images/Ignite-GSA-Roadmap.png)

[Ignite 2024 - **Accelerate your Zero Trust Journey: Unify Identity and Network Access**](https://ignite.microsoft.com/en-US/sessions/BRK326)

### (Universal) Continuous Access Evaluation (CAE)

Global Secure Access recently got CAE support, which means that it is able to interrupt already authenticated sessions if the corresponding trigger is activated. This is a huge advantage over both a VPN and the App Proxy.  

***I will publish a separate blog on CAE and GSA in the next weeks.***

## Summary and decision-making aid

*Due to the restrictions of the App Proxy to web applications, the question arises for all other applications / protocols whether VPN or Entra Private Access is the right choice - I am in favor of Entra Private Access and have written down my reasons for this here: LINK*

*There is no clear answer for web applications, as both variants have their strengths and limitations. As is so often the case: know your tools!*

***However, if there are no hard criteria such as the need for B2B access, I would always give Entra Private Access the preference in case of doubt due to its advantages in the areas of performance, logging and security.***

|  | **App Proxy** | **Entra Private Access** |
| --- | --- | --- |
| Pre-Authentication | optional | always |
| Conditional Access | full featured 
(if Pre-Authentication is enabled) | full featured |
| BYOD Support | BYOD is possible, incl. MAM with Edge Browser. Conditional Access can be used to enforce specific devices.  | all devices have to be registered or joined to the tenant and need the GSA agent |
| B2B Support | yes | no |
| Name Resolution | needs to be public resolvable with (limited) support for translation | private DNS via publishing via FQDN  or complete suffixes in the NRPT |
| Complex Apps | feature set of a reverse proxy, incl. CORS and rewriting   | works like you are in the local network |
| Licensing | included in Entra ID P1  | needs separate license |
| Logging / Monitoring | only Entra Sign-In logs and local logging on the connector | additional dedicated logs, prebuild dashboards and alerts |
| Continuous Access Evaluation | no | yes |

![EPA-EAP-Comparison](/post/2025/Entra-Private-Access-and-the-future-of-the-Entra-App-Proxy/images/EPA-EAP-Comparison.png)

## Better together

Depending on the environment, it is therefore not unlikely that both will be used together - and I think this is one of the biggest advantages of the solution!

We can not only use the same type of connector for both services, but also the very same connectors and connector groups. This reduces the minimum number of systems required and allows us to scale both services together. 

It is also possible to publish the same websites via both solutions in parallel and benefit from the advantages of both solutions or to allow a smooth migration. 

### Running both features side by side

For the general coexistence between Entra App Proxy and Entra Private Access it is important to understand that whenever there is the same URL for an application published to both App Proxy and Entra Private Access, and the user has the Global Secure Access Client installed and running, **Entra Private Access will take precedence over App Proxy and capture the traffic accordingly**.

This behavior allows the administrator to have both configurations active at the same time. Whenever the user accesses the web application from a device with the Global Secure Access client installed, Private Access will take care of the traffic forwarding and whenever the user accesses the web application from a device without the Global Secure Access client, Entra App Proxy will take care of the traffic forwarding.

This combines both scenarios and allows administrators to slowly onboard more users to Entra Private Access while maintaining the existing Entra App Proxy functionality for their user base.

The tables below will explain the expected behavior in different user rollout stages of Entra Private Access. Admins can use the assignment of the Private Access traffic forwarding profile to their users to gradually onboard more users to use Private Access over App Proxy.

Scenario A: Internal URL is different from external App Proxy URL, User is outside of corp. network

|  | **Using external App Proxy URL** | **Using Internal URL** |
| --- | --- | --- |
| **App is not created in Private Access, user is not assigned to the Private Access traffic profile** | Access via App Proxy | No Access |
| **App is created in Private Access, user is not assigned to the Private Access traffic profile** | Access via App Proxy | No Access |
| **App is created in Private Access, user is assigned to the Private Access traffic profile** | Access via App Proxy | Access app via Private Access |

Scenario B: Internal and external App Proxy URL are the same, user is outside of corp. network

|  | **Using external / Internal**               **App Proxy URL** |
| --- | --- |
| **App is not created in Private Access, user is not assigned to the Private Access traffic profile** | Access via App Proxy |
| **App is created in Private Access, user is not assigned to the Private Access traffic profile** | Access via App Proxy |
| **App is created in Private Access, user is assigned to the Private Access traffic profile** | Access via Private Access |

# **Attribution and References**

- [Microsoft Learn: Using Microsoft Entra application proxy to publish on-premises apps for remote users](https://learn.microsoft.com/en-us/entra/identity/app-proxy/overview-what-is-app-proxy)
- [Microsoft Learn: Redirect hard coded links for apps published with Microsoft Entra application proxy](https://learn.microsoft.com/en-us/entra/identity/app-proxy/application-proxy-configure-hard-coded-link-translation)
- [Microsoft Learn: How to configure single sign-on to an application proxy application](https://learn.microsoft.com/en-us/entra/identity/app-proxy/how-to-configure-sso)
- [Microsoft Learn: Global Secure Access dashboard](https://learn.microsoft.com/en-us/entra/global-secure-access/concept-traffic-dashboard)
- [Ignite 2024: Accelerate your Zero Trust Journey: Unify Identity and Network Access](https://ignite.microsoft.com/en-US/sessions/BRK326)
- [Youtube: Troubleshooting Deep Dive on Microsoft Entra Internet Access](https://youtu.be/-gdaqLAwVt4?t=525&si=Erh0_D9hYN-tb1NR)

> 💡 The inspiration for this blog was a discussion with [Peter Lenzke](https://www.linkedin.com/in/peter-lenzke-bb95813a/) on the subject and he played a very important role in the Better Together area in particular. Thank you Peter for your support and the good discussions 🙏