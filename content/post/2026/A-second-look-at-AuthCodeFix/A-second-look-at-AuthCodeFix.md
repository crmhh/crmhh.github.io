---
layout:     post
title:      "A second look at AuthCodeFix (ConsentFix)"
subtitle:   "OAuth authorization-code phishing against Entra ID: a one-minute window, the Global Secure Access control that stops it, and where to see it in the logs"
date:       2026-09-01
draft:      true
author:     "Chris Brumm"
URL:        "/2026/09/A-second-look-at-AuthCodeFix/"
tags:
    - Entra
    - Global Secure Access
    - Conditional Access
categories: [ Global Secure Access ]
---

The trigger for this post is a session on the current state of advanced identity attacks that I
co-presented at the Workplace Ninja Summit. One of the four techniques we covered was ConsentFix — or
AuthCodeFix, which is the better name, since the attack has nothing to do with consent and everything to
do with the authorization code. I'll use AuthCodeFix from here. It has been described many times since
Push Security first published it as ConsentFix at the end of 2025, so I won't add another walkthrough. I
want to look at three narrower questions instead: how you would see it happen, which control actually stops
it, and where the existing guidance — including a post of mine from earlier this year — is now out of
date.

## AuthCodeFix in one paragraph

AuthCodeFix abuses a legitimate flow. The attacker builds an authorization request for a trusted
first-party client, usually Azure CLI, with a `redirect_uri` pointing at `http://localhost`. The victim
opens the link and signs in on the real Entra endpoint, MFA and all. The localhost redirect then fails,
because there is no listener, and the browser shows an error page whose URL contains the authorization
code. The victim is talked into copying that URL and handing it over, and the attacker exchanges the code
for an access and a refresh token. Nothing about the authentication is faked; the victim performs it,
which is why the usual anti-phishing controls stay quiet. [Fabian Bader](https://www.linkedin.com/in/fabianbader/), [Thomas Naunheim](https://www.linkedin.com/in/thomasnaunheim/) and I wrote up the
technique and its mitigations at the [end of 2025](https://www.glueckkanja.com/en/posts/2025-12-31-vulnerability-consentfix),
and the mechanics are covered there, in [Push Security's original research](https://pushsecurity.com/blog/consentfix),
and in [John Hammond's video walkthrough](https://www.youtube.com/watch?v=AAiiIY-Soak).

The delivery is worth a note, because it isn't tied to one channel. The link can arrive by email or a
Teams message, and in the campaigns Push observed it came through compromised, search-ranked pages fronted
by a fake Cloudflare Turnstile check, a browser-native trick close to ClickFix. What these share is that
they lean on the trust the browser and the real Entra endpoint already carry, and they don't depend on the
mail gateway to succeed. That is part of why the controls that matter for this attack sit at the token
layer.

| ![Picture 1: the AuthCodeFix flow](/post/2026/A-second-look-at-AuthCodeFix/images/consentfix-flow.png) |
|:--:|
| *Picture 1: 1 attacker sends link · 2 victim clicks (or scans a QR) · 3 redirect and login (often SSO) · 4 Conditional Access check · 5 redirect to localhost, error message · 6 user transmits URL · 7 attacker sends the auth code · 8 Conditional Access check (location only). Steps 4 and 8 are the two points where the check applies.* |

## The one-minute handoff

At some point Microsoft shortened the lifetime of an authorization code, and the change matters more than
its size suggests. The identity platform documentation used to give the lifetime as about ten minutes; a
[documentation change](https://github.com/MicrosoftDocs/entra-docs/commit/da8b8502fc7e45b6941452612aa793d6a1f204b3)
merged on 22 January 2026 revised it to about one minute, and that is what the platform enforces today. I
could not find an announcement anywhere — no *What's new* entry, no blog post, no release note. The
one-line documentation edit is the whole communication.

Because that is only a documented value, I measured it. In a lab tenant I redeemed codes against Azure CLI
at increasing ages: still accepted at 55, 58 and 59 seconds, then rejected at 61 and 70 with `AADSTS70008`
("expired due to inactivity"). The real lifetime is a hard sixty seconds.

That minute falls on the one step that still needs a person: the handoff, the victim moving the code URL to
the attacker. It happens in one of two ways. In the first, everything stays in the browser — the phishing
page shows a form, the victim pastes the URL into it, and the exchange never leaves the tab. In the second,
the victim sends the URL through a separate channel, most often a Teams chat, sometimes email or another
messenger.

Either way, the whole sequence has to fit inside a minute: the error page appears, the victim finds and
copies the URL, gets it to the attacker, and the attacker redeems it. Someone watching for a URL and
pasting it into a terminal by hand does not reliably make that window, which explains how the toolkits
developed — the later versions take the human off the attacker's side of the exchange. That automation is
not limited to the web form: Eric Woodruff built a demo in which the attacker side of a Teams handoff is
fully automated, redeeming the moment the victim pastes. So the sixty-second window only pressures the
fully manual case; as soon as one side is automated, it is no longer the obstacle.

| ![Picture 2: the AuthCodeFix timeline](/post/2026/A-second-look-at-AuthCodeFix/images/consentfix-timeline.png) |
|:--:|
| *Picture 2: from Push's first manual version through the automated toolkits, with the quietly shortened code lifetime as the pressure that drove the automation.* |

> 💡 The access token that comes out of the redemption is not short-lived in the same way. In my tests it
> came back with a lifetime of roughly 65 to 90 minutes. The sixty-second limit is on the code, not on what
> the code buys once it is redeemed.

Which channel the code takes decides where you can *see* the attack, though not how you *block* it: the
redemption is the same however it arrived. Start with seeing it.

## Catching it in the logs

Take the Teams path first. The attack is visible there, but not in the obvious place. The natural hunt is to
search Teams message URLs for a localhost address carrying a `code=` parameter:

```kql
MessageUrlInfo
| where Url has "localhost" and Url has "code="
```

It looks right and finds nothing. Defender for Office 365 does not treat a `localhost` URL as a URL, so it
never lands in the Teams message tables. I confirmed the same across tenants: ordinary URLs appear in
`MessageUrlInfo`, localhost ones don't.

The same message, taken from Defender for Cloud Apps instead, does keep the URL. A short query over
`CloudAppEvents` returns the handoff with the full URL and code, and it needs nothing beyond the Defender for
Cloud Apps data that comes with an E5 Security plan, no extra configuration:

```kql
CloudAppEvents
| extend Data = parse_json(RawEventData)
| mv-expand Url = Data.MessageURLs to typeof(string)
| where Url has "localhost" and Url has "code="
| project Timestamp, ActionType, Sender = AccountDisplayName, Url, HasForeignTenantUsers = RawEventData.ParticipantInfo.HasForeignTenantUsers, CommunicationType = RawEventData.CommunicationType, ParticipatingDomains = RawEventData.ParticipantInfo.ParticipatingDomains
```

| ![Picture 3: the CloudAppEvents query result](/post/2026/A-second-look-at-AuthCodeFix/images/cloudappevents-messageurls.png) |
|:--:|
| *Picture 3: the CloudAppEvents query returns the Teams message with the full localhost URL and code, where the mail-side tables show nothing.* |

This is the low-cost path: no extra setup, and a Teams message carrying an AuthCodeFix URL has no benign
reason to exist, so the signal stays clean.

That hunt runs on data an E5 Security plan already carries. Microsoft Purview data-loss-prevention sits on
the other side of the E5 stack, in the compliance licences, and where you have it you can act on the content
instead of only watching it. Data-loss-prevention matches sensitive patterns and takes action on them; a
policy scoped to Teams can match the same auth-code URL in the message text. Purview inspects the raw
message, so the localhost address that Defender for Office 365 skipped is no obstacle here, and it can block
the message before it reaches the attacker, with a warning to the sender. Detecting the handoff needs only
E5 Security; blocking it needs the compliance side.

| ![Picture 4: the Teams DLP block](/post/2026/A-second-look-at-AuthCodeFix/images/dlp-block-teams.png) |
|:--:|
| *Picture 4: the Teams DLP block, and the policy tip that lands at the moment of the handoff.* |

The warning is as much the point as the block. AuthCodeFix works by talking someone into sharing something
they don't realise is dangerous, and the policy tip lands at the one moment that can still change the
outcome. It is Teams-only, though: data-loss-prevention inspects the channel you point it at, and the code
can travel others. The web-form variant, and pushing this down to the endpoint, are a larger topic for their
own post.

On the redemption side there is a second signal, and it is the one we described in the
[glueckkanja post](https://www.glueckkanja.com/en/posts/2025-12-31-vulnerability-consentfix). A single
authorization leaves two sign-in events for the same session, application and user: an interactive one when
the victim signs in, and a non-interactive one when the attacker redeems the code. The tell is that the two
come from **different IP addresses** — the victim's and the attacker's — seconds apart. Legitimate Azure CLI
use produces the same pair from a single address; the attack is the version where the two addresses differ.
The shortened code lifetime tightens the correlation window with it: the original guidance allowed roughly
ten minutes between the two events, and it is now about one.

These share a limit. The handoff query and the handoff block only reach the channel you inspect; the
redemption correlation reaches any channel, but only once the code has been exchanged. The control that
stops the attack on any channel, before the exchange completes, works at the redemption — and that is
prevention.

## What Token Protection covers

Token Protection is the control most people reach for against token theft. It binds the sign-in session
token to the device it was issued on, using proof-of-possession through the Web Account Manager, so a
stolen token can't be used from another machine. Where it applies, that guarantee is cryptographic, and it
is the strongest option available.

The question is where it applies. For native applications it is generally available for Exchange Online,
SharePoint Online and Teams. Browser-based access used to be out of scope entirely, and I said so when I
wrote about Token Protection in April. That has changed: browser support is now in preview, for a single
resource, Azure Resource Manager (the "Windows Azure Service Management API" in Conditional Access), and
only for a short list of web applications.

For AuthCodeFix that preview helps less than it looks. The attack can't use just any client: it needs a
first-party public client that is pre-consented and registers a localhost redirect, which is a short list —
Azure CLI, Azure PowerShell, Visual Studio Code, Visual Studio, the Teams PowerShell module. But that short
list is the problem, not the safeguard. These are native clients, and their tokens go to resources like
Azure Resource Manager and Microsoft Graph, which Token Protection doesn't reach: its native coverage is
Exchange, SharePoint and Teams, and the browser preview is ARM, but for a handful of web apps, not a CLI.
Azure CLI is also part of a client family, so a refresh token from it can be swapped for others. The
attacker never needed much room; the clients that already work sit in the gap.

> 💡 PKCE is not the answer here either, although it looks like it should be. PKCE binds an authorization
> code to the client that started the flow, but in AuthCodeFix the attacker starts the flow, so the attacker
> holds both the code verifier and the challenge. PKCE defends against a code being intercepted; AuthCodeFix
> is a code handed over voluntarily.

## Blocking the redemption

The control that does stop AuthCodeFix is the Compliant Network check in Conditional Access. It requires
that token requests reach Entra ID through the Global Secure Access service for your tenant, and blocks
requests from anywhere else.

The point that is easy to miss is where the check applies in the flow. The interactive sign-in happens on
the victim's own, compliant network, so it passes; you might expect that to be the end of it. But the
redemption is evaluated as well. When the attacker exchanges the authorization code for tokens, that
request is a POST to the token endpoint from the attacker's network, not yours, and Entra ID evaluates the
Compliant Network condition at that point and refuses to issue the token.

I tested it. Redeeming a captured code for a protected user from outside the GSA network, the token
endpoint returned `AADSTS53003`, "Access has been blocked by Conditional Access policies. The access policy
does not allow token issuance." The same user's requests through GSA were issued normally. The block lands
on the redemption, at the moment the token would be issued.

| ![Picture 5: the redemption blocked](/post/2026/A-second-look-at-AuthCodeFix/images/signin-block.png) |
|:--:|
| *Picture 5: the same user, allowed when the request comes through GSA and blocked with AADSTS53003 when it does not.* |

Two details decide how far this reaches. First, it is the Compliant Network grant doing the work, not
Continuous Access Evaluation. A plain "Require Compliant Network" policy, without strict CAE, blocks the
redemption in exactly the same way, and the sign-in record shows the token was not a CAE token. CAE acts
later, at the resource; this block is at issuance. Second, it extends what I described in April for
refresh-token replay: the same authentication-plane enforcement that rejects a replayed refresh token also
rejects the AuthCodeFix authorization-code redemption. When we published the mitigations at the end of 2025,
we rated the Compliant Network check as medium effectiveness. This result argues for higher, at least where
GSA is deployed. It does not slow the attack down; it stops the redemption outright.

## One prerequisite you can drop

Rolling out a Compliant Network policy is real work — every user, the right exclusions, staged carefully —
and I covered that groundwork in the [April post](https://chris-brumm.com/2026/04/Token-Replay-Protection-and-the-Compliant-Network-Check/).
What this result changes is one of the prerequisites.

Reaching GSA does not mean the Microsoft traffic forwarding profile. Entra's authentication endpoints are
carried by a separate system profile, "Microsoft Entra traffic", that is on whenever any forwarding profile
is active and never shows up in the portal. Any GSA profile is enough: in the lab a user with only Entra
Private Access is blocked off-GSA and allowed through it, while a user with every profile but no policy
isn't blocked at all — the policy does the blocking, not the profile. The [documentation](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-compliant-network)
says the same, listing the GSA client and Conditional Access signaling as prerequisites, never the Microsoft
traffic profile.

| ![Picture 6: the GSA client's Entra rules](/post/2026/A-second-look-at-AuthCodeFix/images/policy-entra-rules.png) |
|:--:|
| *Picture 6: on a Private-Access-only device, the GSA client tunnels login.microsoftonline.com through the always-on Microsoft Entra traffic rules.* |

> 💡 I need to correct myself here: in April I wrote that the condition is satisfied "as long as the client
> is running and the Microsoft Traffic Profile is active." Any active profile does. The Microsoft traffic
> profile is still worth having, just not for this.

The personal-device pivot that usually saves a phishing attack does not help here. Where the policy
covers all users and resources, the victim's sign-in from a non-compliant network is blocked exactly as the
redemption is — there is no off-network device to fall back to.

## What to do

AuthCodeFix works because nothing about the authentication is faked, and that bounds what stops it: controls
that inspect the sign-in don't help, because the sign-in is genuine. What actually stops it is a control on
the redemption — but that is also the most work, so it's worth starting with what stands up quickly and
building toward it.

1. **Watch for it, on both sides.** In Teams, a `CloudAppEvents` hunt finds the code URL that the mail-side
   tables drop — E5 Security, no DLP required. On the redemption, the two-sign-in correlation — one session
   redeemed from a different IP within a minute — catches it on any channel. This is the quickest to stand
   up, and most tenants already have what it needs. With Purview data-loss-prevention you can escalate the
   Teams signal to a block.
2. **Require user assignment on the first-party apps** the attack rides on — Azure CLI, Azure PowerShell,
   Visual Studio Code, the Teams PowerShell module. This breaks the attack at the first step: an unassigned
   user can't complete the authorization at all. [sentinel.blog](https://sentinel.blog/consentfix-securing-your-tenant-against-oauth-authorisation-code-theft/)
   walks through the configuration.
3. **Require Compliant Network** in Conditional Access, for all users and all resources. This is the control
   that blocks the redemption — Entra evaluates the condition when the code is exchanged and rejects it from
   outside your network. It works on any GSA profile; the cost is the rollout, not the profile.
4. **Enable Token Protection** where it applies. It is the strongest control where it reaches, even if its
   reach against this attack is narrow today.

Detection is the quick win; user assignment and Compliant Network are what stop the attack — one before it
begins, the other at the moment the attacker tries to cash in the code.

## Sources

**Primary research and disclosure**

- [Push Security — ConsentFix](https://pushsecurity.com/blog/consentfix) — original research; named the technique
- [John Hammond — video analysis](https://www.youtube.com/watch?v=AAiiIY-Soak) — hands-on demonstration of the technique and the v2 drag-and-drop handoff
- [glueckkanja — ConsentFix: how a new OAuth attack bypasses Conditional Access](https://www.glueckkanja.com/en/posts/2025-12-31-vulnerability-consentfix) — Fabian Bader, Christopher Brumm, Thomas Naunheim, 31 December 2025; the mitigation matrix

**Microsoft Learn — authorization code and Token Protection**

- [OAuth 2.0 authorization code flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow) — code lifetime now "about 1 minute"
- [entra-docs commit `da8b850`](https://github.com/MicrosoftDocs/entra-docs/commit/da8b8502fc7e45b6941452612aa793d6a1f204b3) — the 10 min → 1 min change, merged 22 January 2026
- [Token Protection in Conditional Access](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-token-protection) — platform availability and supported resources
- [Token Protection deployment guide — Web apps (preview)](https://learn.microsoft.com/en-us/entra/identity/conditional-access/deployment-guide-token-protection-web-apps) — browser/ARM preview scope

**Microsoft Learn — Global Secure Access and Compliant Network**

- [Enable the Compliant Network check with Conditional Access](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-compliant-network) — prerequisites: GSA client and CA signaling
- [Traffic forwarding profiles](https://learn.microsoft.com/en-us/entra/global-secure-access/concept-traffic-forwarding) — the always-on "Microsoft Entra traffic" system profile

**Microsoft Learn — advanced hunting**

- [`CloudAppEvents` table](https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-cloudappevents-table) — keeps `MessageURLs`, including localhost
- [`MessageEvents` table](https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-messageevents-table) — Teams message metadata; does not record localhost URLs

**Community — detection and mitigation**

- [NVISO — ConsentFix (a.k.a. AuthCodeFix): detecting OAuth2 authorization-code phishing](https://blog.nviso.eu/2026/01/29/consentfix-a-k-a-authcodefix-detecting-oauth2-authorization-code-phishing/)
- [MSEndpointMgr — ConsentFix: the quickfix](https://msendpointmgr.com/2026/01/08/consentfix-quickfix/)
- [sentinel.blog — Securing your tenant against OAuth authorisation-code theft](https://sentinel.blog/consentfix-securing-your-tenant-against-oauth-authorisation-code-theft/) — user-assignment hardening
- [valar12/ConsentFix](https://github.com/valar12/ConsentFix) — mitigation scripts
- [Socura](https://socura.co.uk/threat-alerts/consentfix-weaponising-first-party-trust-to-bypass-phishing-resistant-mfa) · [Mitiga](https://www.mitiga.io/blog/consentfix-oauth-phishing-explained-how-token-based-attacks-bypass-mfa-in-microsoft-entra-id) — defense-in-depth and hunting signals

**Reporting and toolkit evolution**

- [Arctic Wolf](https://arcticwolf.com/resources/blog/new-attack-technique-consentfix-hijacks-oauth-consent-grants/) — threat alert, dates the first observation to 11 December 2025
- [CSO Online](https://www.csoonline.com/article/4105230/meet-consentfix-a-new-twist-on-the-clickfix-phishing-attack.html) — framing as a browser-native ClickFix variant
- [KnowBe4](https://blog.knowbe4.com/new-consentfix-technique-tricks-users-into-handing-over-oauth-tokens) · [SC Media](https://www.scworld.com/brief/microsoft-account-takeovers-eased-by-new-consentfix-attack) — delivery via the Azure CLI OAuth app and the Turnstile lure
- [Push Security — ConsentFix v3: analyzing a new toolkit](https://pushsecurity.com/blog/consentfix-v3-analyzing-a-new-toolkit) · [BleepingComputer — v3](https://www.bleepingcomputer.com/news/security/consentfix-v3-attacks-target-azure-with-automated-oauth-abuse/) — the automated backend
- [Ciphers Security](https://cipherssecurity.com/consentfix-v3-azure-oauth-mfa-bypass/) — v3 attack chain and FOCI abuse

**Own prior work**

- [Token Replay Protection and the Compliant Network Check](https://chris-brumm.com/2026/04/Token-Replay-Protection-and-the-Compliant-Network-Check/) — the Compliant Network deployment groundwork, and the statement corrected here

**MITRE ATT&CK**

- [T1528 — Steal Application Access Token](https://attack.mitre.org/techniques/T1528/)
- [T1566.002 — Spearphishing Link](https://attack.mitre.org/techniques/T1566/002/) · [T1566.003 — Spearphishing via Service](https://attack.mitre.org/techniques/T1566/003/)
