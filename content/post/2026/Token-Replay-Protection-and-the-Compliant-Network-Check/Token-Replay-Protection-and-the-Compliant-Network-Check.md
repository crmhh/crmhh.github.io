---
layout:     post 
title:      "Token Replay Protection and the Compliant Network Check"
subtitle:   "In this part of the series about the Microsoft Traffic Profile in GSA we discuss the Compliant Network check and token replay scenarios"
date:       2026-04-11
author:     "Chris Brumm"
URL:        "/2026/04/11/Token-Replay-Protection-and-the-Compliant-Network-Check/"
tags:
    - Entra
    - Global Secure Access
    - Entra Internet Access
categories: [ Global Secure Access ]
---


This post is part of a series on the Microsoft Traffic Forwarding Profile in Global Secure Access:

1. [Why you should enable the Microsoft Traffic Forwarding Profile](https://chris-brumm.com/...)
2. Token Replay Protection and the Compliant Network Check *(this post)*
3. Universal Tenant Restrictions
4. Coexistence with other Secure Web Gateways
5. Logging

If you haven't read the first post yet, it covers the basics of the Microsoft Traffic Forwarding Profile, how to enable it, and what the four security benefits are: [Why you should enable the Microsoft Traffic Forwarding Profile](https://chris-brumm.com/...).

---

## Why token replay matters

For a long time, the dominant attack vector against Microsoft 365 environments was credential phishing – steal the password, log in as the user. MFA adoption has made this significantly harder, and attackers have adapted accordingly. The focus has shifted to post-authentication attacks: instead of stealing credentials, steal the tokens that are issued after a successful authentication. A valid token already satisfies MFA and device compliance requirements at the time of issuance, so replaying it from a different location or device can bypass both controls entirely.

Microsoft's Digital Defense Report has tracked this shift. Token theft attacks roughly doubled between 2022 and 2023, and AiTM phishing – which captures tokens in transit rather than stealing them from storage – has grown by over 140% year-over-year. Thomas Naunheim and Sami Lamppu have documented the technical detail of these attack scenarios extensively in the [Entra ID Attack & Defense Playbook](https://github.com/Cloud-Architekt/AzureAD-Attack-Defense), which is the best reference I know of for understanding how these attacks work in practice.

---

## The three token types and what attacking them looks like

Entra ID uses three types of tokens that are relevant here:

The **Primary Refresh Token (PRT)** is issued to the device itself and used to silently obtain other tokens without requiring the user to re-authenticate. It is valid for 14 days. On Entra-joined or Hybrid-joined Windows devices, the PRT is protected by the TPM, which makes it significantly harder to extract. On devices without TPM, or on macOS where tokens are stored in the Keychain, the attack surface is considerably larger. Thomas has written in detail about the [macOS token storage scenario](https://www.cloud-architekt.net/abuse-and-replay-azuread-token-macos/) specifically. The [PRT replay chapter of the Attack & Defense Playbook](https://github.com/Cloud-Architekt/AzureAD-Attack-Defense/blob/main/ReplayOfPrimaryRefreshToken.md) covers the full attack scenarios including TPM considerations, detection options and mitigations in depth – it is the most complete public documentation on this topic that I am aware of.

The **Refresh Token (RT)** is issued to applications and used to obtain new access tokens. It is valid for up to 90 days. Unlike the PRT, it is not bound to a device by default, which makes it more portable and therefore more attractive for attackers. The [AiTM chapter of the Attack & Defense Playbook](https://github.com/Cloud-Architekt/AzureAD-Attack-Defense/blob/main/Adversary-in-the-Middle.md) covers RT theft in the context of reverse-proxy phishing tools like Evilginx in detail – including detection approaches using GSA NetworkAccessTraffic logs.

The **Access Token (AT)** is the short-lived bearer token that applications use to authorize requests. The default lifetime is 60–90 minutes, but CAE-enabled applications can issue long-lived tokens with a validity of up to 28 hours. Access tokens are the primary target in AiTM attacks because they are present in every authenticated session and can be replayed directly without requiring the original authentication flow.

---

## Why MFA and device compliance don't stop token replay

When a user successfully authenticates against Entra ID, the resulting tokens carry claims that reflect the security controls that were satisfied at the time of issuance. This includes MFA claims, device compliance state, and authentication strength. These claims are embedded in the PRT and flow into subsequently issued refresh and access tokens.

This is precisely what makes token replay so effective as an attack vector. An attacker who obtains a valid refresh token does not need to bypass MFA or present a compliant device – those requirements were already satisfied by the legitimate user. The token is proof that they were. Replaying it from a different machine or location looks, from Entra's perspective, like a continuation of an already-authenticated session.

Device compliance as a Conditional Access control therefore does not help here. Compliance was verified at the time of token issuance, and the token carries that proof forward. What happens to the token after issuance is outside the scope of Conditional Access as a standalone control.

---

## Where existing controls fall short

Token Protection (formerly Token Binding) addresses part of this problem by cryptographically binding refresh tokens to the device on which they were issued, using Proof-of-Possession via WAM. This is a meaningful improvement for the RT scenario, but it comes with significant current limitations: it only works on Windows with WAM-enabled clients, **browser sessions are explicitly not covered**, and resource coverage is limited to Exchange Online, SharePoint Online, and Teams. The [ConsentFix post](https://www.glueckkanja.com/en/posts/2025-12-31-vulnerability-consentfix) – which Fabian Bader, Thomas Naunheim, and I put together at the end of 2025 – shows in concrete terms where Token Protection helps and where it does not, using the authorization code theft scenario as an example.

Both device compliance and Token Protection share one structural limitation: they operate on the token issuance side. Neither of them can invalidate an access token that has already been issued and is being replayed elsewhere. This is the gap that Continuous Access Evaluation is designed to address – and the Compliant Network check is one of the signals that CAE can act on.

---

## What the Compliant Network check adds

The Compliant Network check in Conditional Access uses the GSA service as a network signal. When enforced, Entra ID requires that token requests originate from a device connected to the GSA service for your tenant. This operates at two levels:

At the **authentication plane**, the check applies at the time of token issuance. If an attacker steals a refresh token and attempts to use it to obtain a new access token from outside the GSA network, Entra ID blocks the request immediately. This closes the RT replay scenario that device compliance and Token Protection alone cannot fully address across all platforms and client types.

At the **data plane**, the Compliant Network check works in combination with Continuous Access Evaluation. CAE-capable applications – currently Exchange Online and SharePoint Online – treat a transition from a compliant network to a non-compliant network as a user condition change, which triggers immediate re-authentication even if the access token is still within its validity period. As I described in my [Global Secure Access in Conditional Access post](https://chris-brumm.com/2024/08/06/Global-Secure-Access-in-Conditional-Access/), this combination provides meaningful protection against access token replay in CAE-capable applications.

The honest limitation is the same as for the other controls: the GSA client must be deployed on the endpoint, and the profile must be active for the user. The Compliant Network check applies to all Entra ID-integrated apps when configured accordingly – not just Microsoft 365 – but practical CAE coverage today is still limited to Exchange and SharePoint.

---

## A comparison of the different approaches

None of these controls is a silver bullet. Each closes different gaps:

| | Compliant Device | Token Protection | Compliant Network + CAE | Universal CAE |
|---|---|---|---|---|
| Prevents AiTM (auth plane) | Yes | No | Yes | No |
| Prevents PRT Cookie replay (auth plane) | No | Yes (WAM/Windows only) | Yes | No |
| Prevents RT replay (auth plane) | No | Yes (WAM/Windows only) | Yes | No |
| Prevents AT replay (data plane) | No | No | Yes, via CAE (CAE-capable apps only) | Yes (GSA access tokens) |
| Non-Windows support | Yes | Limited (preview) | Limited | Yes |
| Requires GSA client | No | No | Yes | Yes |

Universal CAE is worth a separate note. It protects the GSA access tokens themselves – the short-lived tokens the GSA client uses to authenticate to the service tunnels – against theft and replay. If an attacker captures a GSA access token and replays it from a different IP address, Universal CAE's optional Strict Enforcement mode blocks the request. This is a different layer than the Compliant Network check: Compliant Network protects your Entra ID-integrated app tokens, Universal CAE protects the GSA service tokens. Both are worth enabling. The [Microsoft documentation on Universal CAE](https://learn.microsoft.com/en-us/entra/global-secure-access/concept-universal-continuous-access-evaluation) covers the configuration details.

<!-- DIAGRAM: universal-cae-flow.png -->
| ![Picture 1: Universal CAE Flow](/post/2026/Token-Replay-Protection-and-the-Compliant-Network-Check/images/universal-cae-flow.png) |
|:--:|
| *Universal CAE flow: the GSA client (1) uses a Refresh Token to obtain an Access Token from Entra ID, (2) Entra ID issues the Access Token, (3) Entra ID sends a revocation event to the GSA service, (4) the client attempts to use the Access Token at the resource provider, (5) the token is rejected in near real-time.* |

<!-- DIAGRAM: universal-cae-triggers.png -->
| ![Picture 2: Universal CAE Triggers](/post/2026/Token-Replay-Protection-and-the-Compliant-Network-Check/images/universal-cae-triggers.png) |
|:--:|
| *Trigger events for Universal CAE: user account deleted or disabled, password changed or reset, MFA enabled, administrator explicitly revokes all refresh tokens, high user risk detected by Entra ID Protection. The highlighted trigger – explicit token revocation by an administrator – is particularly relevant for incident response scenarios.* |

The practical conclusion is that these controls are complementary rather than alternatives. An environment with device compliance, Token Protection enabled where possible, Compliant Network enforced via GSA, and Universal CAE active covers significantly more of the attack surface than any single control alone. The combination of Compliant Network as a CA signal and CAE as the enforcement mechanism is the only approach in this list that can invalidate access tokens after issuance – relevant in scenarios where an already-issued AT is exfiltrated and replayed from outside the GSA network.

---

## Demo: Blocking a PRT Cookie replay with the Compliant Network check

The following demo shows a token replay attack using [BAADTokenBroker](https://github.com/secureworks/BAADTokenBroker), a post-exploitation tool presented at Black Hat Asia 2024 by Yuya Chudo (Secureworks). The tool extracts the PRT Cookie from a managed Windows device by talking directly to lsass, and passes it to an attacker-controlled system which attempts to use it to authenticate against Microsoft 365.

The setup uses two victim clients with different Conditional Access policies:

- **WaldorfWin11** – protected by MFA and Device Compliance only
- **StadlerWin11** – protected by MFA, Device Compliance, and Compliant Network (GSA)

On both devices, BAADTokenBroker successfully extracts the PRT Cookie – the tool works regardless of the CA policy in place, since it operates at the OS level before any network controls apply. The difference becomes visible when the attacker attempts to replay the cookie from the unmanaged attacker system:

- The token from **WaldorfWin11** works. The attacker lands in Office 365 because Device Compliance and MFA claims are already embedded in the token.
- The token from **StadlerWin11** is blocked. Entra ID enforces the Compliant Network check at the authentication plane and rejects the token request because it does not originate from within the GSA service.

<!-- GIF: EIA-Block-TokenTheft.gif -->
| ![Picture 3: Compliant Network and PRT theft](/post/2026/Token-Replay-Protection-and-the-Compliant-Network-Check/images/EIA-Block-TokenTheft.gif) |
|:--:|
| *BAADTokenBroker PRT Cookie extraction and replay – blocked by Compliant Network on StadlerWin11, successful on WaldorfWin11* |

> 💡 This demo was recorded in January 2025 and reflects the behavior at that time. The attack technique itself remains valid. Since then, comparable PRT Cookie extraction techniques have also been demonstrated for macOS ([DEF CON 33, August 2025](https://infocondb.org/con/def-con/def-con-33/original-sin-of-sso-macos-prt-cookie-theft-entra-id-persistence-via-device-forgery)), which underlines the importance of platform-independent mitigations like the Compliant Network check.

> 💡 Two notes on BAADTokenBroker that are relevant for the threat assessment:
>
> First, the `Request-PRTCookie` command – which extracts the PRT Cookie of the currently logged-in user by calling lsass directly – **does not require administrative rights**. Standard user context is sufficient. This significantly lowers the bar for exploitation, as an attacker with only user-level code execution on a device can extract the token. This has been confirmed independently for the Windows implementation and was also demonstrated for macOS at user-level permissions at TROOPERS 2025.
>
> Second, **Microsoft Defender now detects and blocks BAADTokenBroker** in its current form. This is worth noting as context for the demo – the tool as published is no longer a silent threat on Defender-protected endpoints. However, the underlying technique is well-documented and has been ported into other offensive tooling (including C2 frameworks), so the conceptual threat remains valid.

---

## Enabling CA Signaling

Before the Compliant Network condition becomes available in Conditional Access, one prerequisite needs to be in place: GSA signaling must be enabled in the tenant. Navigate to **Global Secure Access → Session management → Adaptive access** and enable the toggle for GSA signaling in Conditional Access. This creates the named location **All Compliant Network locations** which then becomes selectable as a network condition in CA policies.

---

## The Conditional Access policy

The Compliant Network check is configured as a network condition in CA. The recommended approach is a dedicated block policy:

- **Target resources**: All cloud apps (GSA resources are automatically excluded in the backend – no explicit exclusion needed)
- **Network**: Include → All locations; Exclude → All Compliant Network locations
- **Grant**: Block access

Any authentication not coming through the GSA service is blocked. The automatic exclusion of GSA resources itself ensures the client can always reach the service to establish the compliant network signal – no circular dependency there.

<!-- SCREENSHOT: post2-ca-policy-config.png -->
| ![Picture 4: Conditional Acces Policy config](/post/2026/Token-Replay-Protection-and-the-Compliant-Network-Check/images/ca-policy-config.png) |
|:--:|
| *CA policy "EIA 3 - Require Compliant Network" showing the key configuration elements – Network condition excluding All Compliant Network locations, Grant set to Block access, and Target resources excluding Microsoft Intune and Microsoft Intune Enrollment.*|

> 💡 GSA resources themselves do not need to be manually excluded. When Compliant Network is enabled in a CA policy, Entra ID [automatically excludes the endpoints](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-compliant-network) the GSA client needs to reach the service. Without this automatic exclusion, the client could never establish the compliant network signal in the first place – a circular dependency that Microsoft has resolved in the backend. This is also confirmed by Fabian Bader's [Conditional Access Bypasses](https://cloudbrothers.info/en/conditional-access-bypasses/) research.

> 💡 All experience in this post is based on Windows deployments. The GSA client is available for macOS, iOS, and Android as well, but I have not yet validated the behavior on those platforms in production. Treat any non-Windows statements here as directional rather than confirmed.

---

## Required exclusions

The automatic backend exclusions cover the GSA service itself, but several other services need explicit exclusions to avoid breaking your environment.

### Mandatory exclusions

**Microsoft Intune and Microsoft Intune Enrollment**

This is the most important exclusion and the one most likely to cause problems if missed. Devices need to communicate with Intune before the GSA client is deployed – and in some enrollment scenarios, before the device is even registered in Entra ID. Both **Microsoft Intune** and **Microsoft Intune Enrollment** must be excluded from the Compliant Network policy.

This exclusion carries a security implication worth understanding: it creates the same type of chicken-egg bypass that exists for device compliance policies. The [Compliant Device Bypass post](https://www.glueckkanja.com/en/posts/2025-01-14-compliant-device-bypass) that Fabian Bader, Thomas Naunheim and I wrote covers the broader context of this issue in detail. The short version: the exclusion is necessary and by design, but it means that Intune enrollment flows are not protected by the Compliant Network check.

**Emergency Access Accounts**

Break-glass accounts must always be excluded from Compliant Network policies – as they should be from all CA policies. If something breaks during rollout and the GSA client stops functioning, you need a way back in. Microsoft provides a [break-glass recovery PowerShell script](https://learn.microsoft.com/en-us/entra/global-secure-access/scripts/powershell-break-glass-recovery) specifically for this scenario that re-enables forwarding profiles and CA policies after a GSA service disruption.

### Recommended exclusions

**Microsoft Defender mobile app**

If you use the GSA client on mobile devices via the Defender app, specific Defender resources need to be excluded to avoid blocking the app from reaching the GSA service. Microsoft documents these exclusions separately under [Defender mobile app exclusions from Conditional Access](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-compliant-network).

### Optional exclusions – AVD, W365, and Remote Access scenarios

The following apps are relevant if your users access Azure VMs, Azure Virtual Desktop, Windows 365, or other Remote Desktop scenarios from unmanaged devices:

- Azure Windows VM Sign-In
- Azure Virtual Desktop
- Windows 365
- Microsoft Remote Desktop
- Windows Cloud Login

A common use case is allowing contractors or other users on unmanaged devices to reach a managed Remote Desktop environment, while the endpoint itself has no GSA client. A blanket exclusion from the Compliant Network policy would leave these authentications entirely unprotected.

The recommended approach is a **separate CA policy** that targets these apps specifically and requires **phishing-resistant authentication** as the grant control. This keeps the access path open for unmanaged devices while ensuring the authentication itself meets a high bar. It also avoids weakening the main Compliant Network policy with broad exclusions.

> 💡 It is worth noting that the B2B and BYOD capabilities of GSA do not apply to this scenario, for two distinct reasons.
>
> First, those features – such as cross-tenant access and BYOD support – are specific to **Entra Private Access** and operate at the network tunnel level. The Microsoft Traffic Forwarding Profile has no equivalent capability for external or unmanaged devices.
>
> Second, AVD and W365 scenarios typically involve an **identity switch**: the user authenticates with their own identity to reach the Remote Desktop environment, and then operates under a different identity inside the session. These are two separate authentication flows. The Compliant Network signal from the outer authentication does not carry over to the inner session, and the GSA client on the endpoint has no visibility into what happens inside the remote session.

---

## Rollout considerations

A few practical points before enabling the policy:

**Start in report-only mode.**

> 💡 **Always start in report-only mode.** Before enabling the policy, run it in report-only for at least a week. A misconfigured Compliant Network policy can lock users out of all Entra ID-integrated apps. The report-only phase will surface unexpected blocks – service accounts, non-Windows devices, apps you forgot about – before they become incidents.

**Use platform filters for a phased rollout.** Rather than rolling out the Compliant Network policy to all users at once, use device platform filters in your CA policy to target Windows devices first. This is the most straightforward approach since the Windows GSA client is the most mature and best-tested option. Other platforms can be added incrementally as the client deployment there is validated.

**Service accounts and non-interactive sign-ins.** Any user account configured as a service account that authenticates against Entra ID without a GSA client will be blocked. These should be identified during the report-only phase and either excluded from the policy or – where possible – migrated to managed identities which are not subject to user-facing CA policies. Workload identities – service principals and managed identities – are not affected as they are not included in user-scoped CA policies. 

**Per-tenant control.** The Compliant Network check is enforced per tenant. In B2B scenarios where users from your tenant access resources in another tenant, the Compliant Network signal from your tenant does not automatically satisfy the Compliant Network requirement in the other tenant. This scenario is covered in the admin access section below in the context of PAW scenarios.

---

## Why admin access deserves a separate look

The Compliant Network check applies to all users equally – but the value it provides is asymmetric. For a standard user, it closes token replay scenarios that device compliance alone doesn't cover. For an admin, the stakes are higher: a compromised admin token can lead to tenant takeover, and the existing controls – MFA, device compliance, phishing-resistant authentication – while necessary, each have limitations as described above.

Adding Compliant Network to the controls required for access to administrative interfaces is therefore not just a marginal improvement. It means that even a stolen token from an admin session cannot be replayed from outside the GSA network.

---

## Scenario 1: Privileged Access Workstations

A PAW is a dedicated, hardened device used exclusively for administrative tasks. No email, no general web browsing, no productivity applications – only the tools required for privileged work. The clean-source principle ensures that an attacker who compromises the user's regular workstation cannot reach the admin credentials or sessions.

For the PAW itself, the GSA client should always be deployed. Beyond the token replay protection covered in this series, the Microsoft Traffic Profile unlocks additional capabilities on the PAW that are worth having – but those deserve their own post.

The CA policy design for PAW access to administrative interfaces typically combines:

- Require compliant device (the PAW, enforced via device filter)
- Require phishing-resistant authentication
- Require compliant network

> 💡 Rather than targeting specific admin portals in the CA policy, the cleaner approach is to scope the policy to privileged roles or a dedicated admin group and apply it to **All Cloud Apps**. This ensures coverage is automatic when new administrative interfaces are added, and avoids the maintenance overhead of keeping a portal list up to date.

This combination means that accessing the Entra portal, Azure portal, or Microsoft 365 admin center requires a token that was issued on a managed, compliant device, with a phishing-resistant credential, from within the GSA network. A stolen token satisfying all three conditions simultaneously is significantly harder to achieve than compromising any single control.

---

## Scenario 2: Admin access from a standard company device

Not every organization has a PAW program in place. In practice, many admins perform administrative tasks from the same device they use for daily work – checking email, joining Teams calls, and accessing the Azure portal in the same session.

The pragmatic step up from basic device compliance for these environments is to require both device compliance **and** compliant network for access to administrative interfaces. This is straightforward to implement as a CA policy scoped to privileged roles or specific admin portals.

One observation worth sharing from practice:

> 💡 The Compliant Network signal is device-wide, not user-specific. When the GSA client is active on a Windows device, the signal applies to all users authenticating from that device – including a separate admin account used in a different browser profile or Edge window. The admin account does not need to be the primary account registered with the GSA client. As long as the device has the client running and the Microsoft Traffic Profile active, the compliant network condition is satisfied for admin sign-ins on the same device.

This is particularly useful in environments where the admin uses a separate Entra account for privileged tasks but works on the same physical machine as their regular user account. The GSA client runs under the regular user context, and the admin account benefits from the compliant network signal without any additional configuration.

> 💡 This behavior also means that device-level controls and user-level controls are complementary here. Device compliance and compliant network are both device-level signals. They confirm that the request is coming from a known, managed device connected to the GSA service – regardless of which user account is performing the authentication.

---

## A note on the relationship between the two scenarios

These two scenarios are not mutually exclusive – they represent different points on the same maturity curve. Organizations without a PAW program can start with the company device approach: require compliant network in addition to device compliance for privileged roles, and use that as a forcing function to ensure GSA client deployment reaches admin devices.

The PAW scenario is the more complete answer, but it does not need to apply to every admin in the organization. In practice, a PAW program typically starts with Tier 0 identities – Global Administrators, Privileged Role Administrators, and other roles that can cause tenant-wide impact if compromised. For Tier 1 and Tier 2 admins, the company device approach with Compliant Network and Device Compliance is often the right balance between security and operational overhead. This aligns with the [Microsoft Enterprise Access Model](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-access-model) which recommends tiered controls based on the blast radius of each role rather than a one-size-fits-all approach.

---

## What's next

The next post in this series covers Universal Tenant Restrictions – how to use the Microsoft Traffic Profile to enforce which external Entra tenants your users are permitted to authenticate against.
