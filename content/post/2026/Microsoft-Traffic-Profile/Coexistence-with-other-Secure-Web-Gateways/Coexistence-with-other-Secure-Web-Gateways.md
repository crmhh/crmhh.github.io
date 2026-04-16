---
layout:     post 
title:      "Coexistence with other Secure Web Gateways"
subtitle:   "In this part of the series about the Microsoft Traffic Profile in GSA we discuss how to run GSA alongside a third-party SWG and what breaks if you don't"
date:       2026-04-29
draft:      true
author:     "Chris Brumm"
URL:        "/2026/04/Coexistence-with-other-Secure-Web-Gateways/"
tags:
    - Entra
    - Global Secure Access
    - Entra Internet Access
categories: [ Global Secure Access ]
---

This post is part of a series on the Microsoft Traffic Forwarding Profile in Global Secure Access:

1. [Why you should enable the Microsoft Traffic Forwarding Profile](https://chris-brumm.com/2026/04/Why-you-should-enable-the-Microsoft-Traffic-Forwarding-Profile/)
2. [Token Replay Protection and the Compliant Network Check](https://chris-brumm.com/2026/04/Token-Replay-Protection-and-the-Compliant-Network-Check/)
3. [Universal Tenant Restrictions](https://chris-brumm.com/2026/04/Universal-Tenant-Restrictions/)
4. Coexistence with other Secure Web Gateways *(this post)*
5. Logging

---

## The problem you probably have

Some organizations that are evaluating the Microsoft Traffic Forwarding Profile are not starting from scratch. They already have a Secure Web Gateway in place – Zscaler, Netskope, or one of the other major SSE vendors – and have been routing internet traffic through it for years. The natural question is whether adding GSA on top creates conflicts, and if so, how to handle them.

The short answer is: yes, there is a real conflict, but it only manifests if you route Microsoft 365 traffic through your existing SWG. If you are already using Microsoft 365 optimized routing – which most modern SWG deployments do – you are likely bypassing M365 traffic at the SWG level already. In that case, the Microsoft Traffic Profile slots in cleanly. If you are not, this post explains what is breaking and why.

---

## What routing M365 through a SWG actually breaks

When Microsoft 365 traffic flows through a cloud-based SWG, Microsoft's services see the SWG's egress IP as the source of the request – not the user's actual IP address. This is not a minor logging detail. It has concrete consequences across three areas:

**Entra ID Protection**

Entra ID Protection's risk detections depend heavily on IP-based signals. Impossible Travel, Atypical Travel, and Unfamiliar Sign-in Properties all use the source IP of the authentication request to assess whether a sign-in looks anomalous. When that IP is a SWG egress address shared by hundreds or thousands of users across your organization, the signal is degraded significantly. Impossible Travel alerts fire when two users sign in from the same SWG egress IP in different cities. Atypical Travel stops working because all sign-ins appear to originate from the same location. The detections don't disappear, but their accuracy drops to the point where alert fatigue sets in quickly.

**Conditional Access Named Locations**

If you have Conditional Access policies that reference named locations – trusted IP ranges that determine whether a sign-in is considered on-network or off-network – routing M365 authentication through a SWG breaks this logic. Entra ID sees the SWG's egress IP, not the user's corporate egress IP. Users authenticating from your office appear to come from a Zscaler or Netskope datacenter. Your trusted location policies either fire incorrectly or stop being meaningful at all.

This also affects Continuous Access Evaluation's location-based enforcement. CAE can revoke access when a user moves from a trusted to an untrusted network – but if all authentication looks like it's coming from a SWG IP, CAE's location signal is useless.

**Sign-in log fidelity**

Beyond the active security controls, the sign-in logs themselves lose their value for investigation and forensics. When an incident requires you to correlate authentication events with network activity, a log full of SWG egress IPs makes that correlation significantly harder. The user's actual network path is invisible.

---

## The additional problem: Tenant Restrictions degrade

There is a second, less obvious consequence that connects directly to the previous post. SWGs can enforce Tenant Restrictions v1 by injecting a custom HTTP header on Microsoft login traffic. This requires TLS inspection on the Microsoft login endpoints, which is operationally expensive and brings its own problems – particularly on Android, where SSL certificate handling for non-system trusted roots creates persistent issues.

More importantly, this approach only covers the authentication path. It does not protect against token infiltration or anonymous access scenarios at the data path. The Universal Tenant Restrictions via GSA that [Post 3](https://chris-brumm.com/2026/04/Universal-Tenant-Restrictions/) covered provides both authentication plane and data plane coverage – but that capability requires the Microsoft Traffic Profile to be active. If M365 authentication is flowing through the SWG instead, Universal TR cannot do its job.

---

## Why the Microsoft Traffic Profile solves this

The GSA client uses a Lightweight Filter (LWF) driver to intercept traffic at the network stack level, before VPN tunnels or SWG tunnel adapters see it. This means when both a SWG client and the GSA client are installed on the same device, the GSA client captures M365 traffic first. The SWG sees what's left. The [Microsoft documentation on the GSA deployment guide for Microsoft Traffic](https://learn.microsoft.com/en-us/entra/architecture/gsa-deployment-guide-microsoft-traffic) covers the coexistence architecture in more detail.

For M365 traffic that goes through GSA, Microsoft's services see the user's actual source IP – restored and verified by the GSA service. This is not just the traffic reaching Microsoft 365 endpoints: it covers Entra ID authentication for M365 apps as well, which is what matters for Identity Protection, Named Locations, and CAE.

The mechanism is called **[Source IP Restoration](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-source-ip-restoration)**. The GSA client cryptographically communicates the original egress IP of the endpoint to the GSA service, which in turn passes it to Entra ID and Microsoft Graph. From Microsoft's perspective, the authentication and traffic appear to originate from the user's actual IP – not from a GSA or SWG datacenter.

<!-- DIAGRAM: source-ip-restoration.png -->
> 📸 *Diagram: Source IP Restoration flow – GSA client captures M365 traffic before the SWG tunnel, passes original source IP to GSA service, which communicates it to Entra ID*

Source IP Restoration is not enabled by default. It requires explicit configuration in the GSA portal under **Global Secure Access → Session management → Adaptive access → Source IP restoration**. It also has a licensing dependency: it requires Microsoft Entra ID P2 rather than P1.

> 💡 Source IP Restoration only applies when the Microsoft Traffic Profile is active on the device. For devices without the GSA client, the SWG egress IP problem remains. This makes client deployment coverage a prerequisite for the fix to be effective – another reason to treat GSA client rollout as a dependency, not an afterthought.

---

## Performance: a practical argument

Beyond the security implications, there is a straightforward performance argument for routing M365 traffic through GSA rather than a third-party SWG.

SWGs process traffic by decrypting it at their cloud nodes, inspecting it, and re-encrypting it before forwarding it to the destination. For Microsoft 365 traffic – which is already encrypted end-to-end and originates from endpoints that Microsoft operates with known, verifiable certificates – this inspection adds latency and CPU overhead without meaningful security benefit. [Microsoft's own network connectivity principles](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-network-connectivity-principles) have long recommended bypassing M365 traffic at the SWG for exactly this reason, categorizing M365 endpoints into Optimize, Allow, and Default tiers specifically to guide selective bypass decisions.

When GSA handles M365 traffic instead, SSL termination happens at Microsoft's own Office front-door endpoints. The traffic takes the shortest path into Microsoft's global WAN from the nearest GSA point of presence, without additional decryption/re-encryption overhead. For latency-sensitive workloads like Teams calls or real-time collaboration in SharePoint, this difference is measurable.

---

## Forward vs. Bypass: controlling what GSA acquires

The Microsoft Traffic Profile's rules have two possible actions: **Forward** and **Bypass**.

- **Forward** means the GSA client acquires the traffic and routes it through the GSA service.
- **Bypass** means the GSA client deliberately skips the traffic, leaving it to the normal routing path – which in a SWG environment means the SWG tunnel picks it up.

This is the primary configuration lever for coexistence. If you have a SWG and want it to continue handling specific M365 traffic while GSA handles the rest, you set the corresponding rules to Bypass in the Microsoft Traffic Profile.

One behavior worth understanding: when Microsoft adds new rules to the profile – which happens automatically as new M365 services are onboarded – they appear in **Forward mode by default**. This means new services are automatically covered without manual action, but it also means that in a selective configuration where you are relying on Bypass for certain rules, newly added rules will start being acquired by GSA until you explicitly set them to Bypass. Monitoring the profile for rule additions is therefore relevant if you are running a selective coexistence setup. The [Microsoft Traffic Profile concept documentation](https://learn.microsoft.com/en-us/entra/global-secure-access/concept-microsoft-traffic-profile) describes the full rule set and the Forward/Bypass behavior in detail.

---

## Coexistence with Zscaler

Microsoft and Zscaler have documented three deployment scenarios depending on which product handles which traffic category. For environments where the Microsoft Traffic Profile is the focus, the relevant scenario is:

- **GSA handles M365 and internet traffic** (Entra Internet Access), Zscaler handles private application access only (ZPA enabled, ZIA disabled)

In this configuration, Zscaler needs to be configured to exclude the GSA service endpoints from the ZPA tunnel, and the Zscaler Internet Access module should be disabled or its M365 rules set to bypass.

The Microsoft documentation on [Zscaler coexistence](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-zscaler-coexistence) covers the required FQDN and IP exclusions in detail.

> 💡 A common issue in Zscaler coexistence deployments is that ZPA's split tunneling configuration interferes with GSA's LWF driver acquiring the correct traffic. Verify the acquisition order by checking the GSA client's Advanced Diagnostics panel – it will show which rules are active and which traffic is being forwarded vs. bypassed.

---

## Coexistence with Netskope

Support for Netskope coexistence was added to the GSA Windows client in [July 2024](https://learn.microsoft.com/en-us/entra/global-secure-access/reference-windows-client-release-history). The coexistence approach follows the same logic as Zscaler: GSA's LWF driver takes priority for M365 traffic, and Netskope handles internet traffic that falls outside the Microsoft Traffic Profile scope.

Netskope publishes its own coexistence documentation alongside Microsoft's. The key configuration steps involve ensuring that Netskope's steering configuration does not capture the GSA service endpoints, and that M365 destinations are excluded from Netskope's SSL inspection policy to avoid conflicts with GSA's traffic acquisition.

The Microsoft documentation on [Netskope coexistence](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-netskope-coexistence) and [Netskope's own guide](https://docs.netskope.com/en/microsoft-and-netskope-sse-coexistence-1/) cover the required exclusions from both sides.

---

## What to keep in the SWG

Running the Microsoft Traffic Profile for M365 does not mean replacing your SWG. The two products cover different scopes, and the SWG remains the right tool for general internet traffic.

What typically stays in the SWG:

- **General internet traffic** – URL filtering, web content categories, DLP for non-Microsoft destinations
- **Private application access** – if the organization is not using Entra Private Access, the SWG's ZTNA capability continues to handle this
- **Non-M365 SaaS applications** – other cloud services that benefit from the SWG's inspection and DLP policies

What moves to GSA:

- **Microsoft 365 traffic** – Exchange Online, SharePoint, Teams, and M365 Common endpoints
- **Entra ID authentication for M365** – this is the critical piece for Source IP Restoration, Identity Protection, and Named Locations
- **All Entra-integrated apps** if the [Compliant Network check from Post 2](https://chris-brumm.com/2026/04/Token-Replay-Protection-and-the-Compliant-Network-Check/) is enforced – since that requires all authentication to flow through GSA

The practical takeaway is that M365 already tends to be bypassed at the SWG in most well-configured environments. Enabling the Microsoft Traffic Profile means GSA picks up that bypassed traffic and adds the security capabilities described in this series, rather than having it go directly to Microsoft unmanaged.

---

## What's next

The final post in this series covers the Network Access Traffic Logs – the dedicated log table that GSA provides, how it correlates with Entra sign-in logs and M365 audit logs, and what it enables for your SOC team.

## **Attribution and References**

References for the series are [here](/post/2026/microsoft-traffic-profile/microsoft-traffic-profile-series-references/)