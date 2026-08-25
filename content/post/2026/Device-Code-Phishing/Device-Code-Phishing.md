---
layout:     post
title:      "Device Code Phishing: Prevention, Detection and Response"
subtitle:   "A look at device code phishing from the defender's seat — how the attack works, what Microsoft detects by now, whether that is really enough, and what you have to build yourself."
date:       2026-08-30
draft:      true
author:     "Chris Brumm"
URL:        "/2026/08/Device-Code-Phishing/"
tags:
    - Entra
    - ITDR
    - Detection
categories: [ ITDR ]
---

The trigger for this post is my session with [Eric Woodruff](https://www.linkedin.com/in/ericonidentity/) at the [Workplace Ninja Summit 2026](https://summit.wpninjas.global/sessions/) on 16 September 2026: *"The 2026 state of advanced identity attacks"*. The premise is one I like a lot — every time one of these advanced identity attacks makes the news, the same questions come up (are we vulnerable? has Microsoft fixed this? how likely is it really?), and then quietly move to the sidelines as business has to continue. So we go back and check what state a handful of recent Entra ID attacks are actually in, with a red and a blue lens.

Device code flow is one of them — next to ConsentFix/AuthCodeFix, the Device Compliance Bypass and Authentication Downgrade — and preparing my part pulled me in deep enough that a whole blog post fell out of it. This is that post: the blue-team deep dive on device code phishing. Most of what I write here is about Global Secure Access, but identity security is where I spend a good part of my community time, and my day job is on the defender side.

So I go through those same questions for device code flow — the focus is the defender side, not the attack mechanics.

# How it works

The [OAuth 2.0 device authorization grant](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code) exists for devices without a proper keyboard — conference room systems, digital signage, kiosk displays. The device shows a short code, the user types it in on `microsoft.com/devicelogin`, and the device receives its token.

The attack reverses the roles. The attacker starts the flow, sends the code to the victim, and the victim authenticates on the **real** Microsoft page. There is no fake login, no suspicious domain, no password that runs through the attacker. MFA is completed cleanly — by the victim. At the end, the attacker holds a valid token.

So the controls that normally stop phishing simply don't fire here, because nothing about the authentication is fake.

>💡 This is also why user awareness only gets you so far with this one. The usual advice — don't type your password into a strange-looking site — does not apply here, because there is no strange-looking site. The user is doing everything right, on the genuine page.

And the token is not always the end of it: run against the Microsoft Authentication Broker (`29d9ed98-a469-4536-ade2-f981bc1d605e`), the same flow can yield a Primary Refresh Token (PRT) and register a device in the tenant — a short-lived token turned into durable access ([see Dirk-jan](https://dirkjanm.io/phishing-for-microsoft-entra-primary-refresh-tokens/)). That escalation is what makes device code phishing more than a stolen session, and it is why the response section later insists on removing the device, not just revoking the session.

# A short history

Device code phishing is not new. The technique itself goes back to research work around 2020, and there has been open-source tooling for it for years. What changed is not the mechanism but who uses it, and at what scale.

| ![Picture 1: Device code phishing timeline, 2019–2026](/post/2026/Device-Code-Phishing/images/device-code-phishing-timeline.png) |
|:--:|
| *Picture 1: from a research technique to commodity phishing-as-a-service.* |

Roughly, there are three threads, and they build on each other:

- **Research and open-source tooling.** The idea of abusing the device authorization grant for phishing was described by Nestori Syynimaa ([AADInternals](https://aadinternals.com/)) around 2020, followed by tools like Secureworks' [PhishInSuits](https://github.com/secureworks/PhishInSuits) and [SquarePhish](https://github.com/secureworks/squarephish). In 2023, Dirk-jan Mollema published the [step from a phished token to a Primary Refresh Token](https://dirkjanm.io/phishing-for-microsoft-entra-primary-refresh-tokens/) — the escalation from above. In 2024, Dennis Kniep's [DeviceCodePhishing](https://github.com/denniskniep/DeviceCodePhishing) automated the flow with a headless browser, and in 2025 [GitPhish](https://www.praetorian.com/blog/gitphish-automating-enterprise-github-device-code-phishing/) brought the same idea to GitHub.
- **Nation-state actors.** In-the-wild use became visible in 2024, when Microsoft attributed an active campaign to [Storm-2372](https://www.microsoft.com/en-us/security/blog/2025/02/13/storm-2372-conducts-device-code-phishing-campaign/) (assessed as Russia-linked). In 2025, [Volexity](https://www.volexity.com/blog/2025/04/22/phishing-for-codes-russian-threat-actors-target-microsoft-365-oauth-workflows/) documented several Russian actors, and Microsoft named a China-based actor ([Storm-1249](https://www.microsoft.com/en-us/security/blog/2025/05/29/defending-against-evolving-identity-attack-techniques/)).
- **Commodity phishing-as-a-service.** This is the part that changed the picture in 2026. With kits like [EvilTokens](https://blog.sekoia.io/new-widespread-eviltokens-kit-device-code-phishing-as-a-service-part-1/) (and its affiliate panels), and later additions to platforms like Tycoon2FA, device code phishing turned into a product you can rent. [Push Security](https://pushsecurity.com/blog/device-code-phishing) tracks more than twenty-five distinct device code kits by now, and the [FBI warned](https://www.ic3.gov/PSA/2026/PSA260521) about one of them (Kali365) specifically.

The individual names change all the time, so I would not put too much weight on them. The direction is what matters: a technique that used to be a hands-on tool for a few actors is now something many can simply buy, cheaply and at scale.

# Where things stand today

## The delivery channels — and one myth

When people think of device code phishing, they often think of Microsoft Teams, because the Storm-2372 reporting made the "Teams invite" famous. That is worth putting into perspective, because the channels do not all carry the same weight.

- **Email is the dominant channel.** This is where the commodity kits deliver, and it is what you will see most of. QR codes and attachments (HTML, SVG, PDF) are not separate channels here, they are ways to get the link past inspection inside a mail. Delivery from already-compromised accounts is common on top of that.
- **Messaging apps are the notable "besides mail" channel** — and, importantly, mostly on the targeted / nation-state side. The Volexity campaigns ran the outreach over [Signal and WhatsApp](https://thehackernews.com/2025/04/russian-hackers-exploit-microsoft-oauth.html), with "live support" to walk the victim through it. Low volume, high value, and deliberately outside your corporate telemetry (personal devices, encrypted apps).
- **The mobile pivot (QR, SMS)** is growing, for exactly one reason: it moves the click onto a private device, away from your endpoint and mail controls.
- **Teams gets a lot of attention here, but in the data it barely shows up.** In the environments I have looked at, very few device code URLs actually arrive over Teams, and none of the in-the-wild reporting I found treats it as a major channel. A good part of the "Teams invite" story is really a mail that was *styled* as a Teams invite. That does not mean Teams is not a vector — it is, and I will come to detecting it — but it is a gap worth closing for completeness, not the front line.

## What Microsoft detects by now

Microsoft has added quite a lot since early 2026. The following alerts exist across Defender for Identity, Defender XDR and Defender for Office. Only the two MDI entries appear in the [alert reference](https://learn.microsoft.com/en-us/defender-for-identity/alerts-xdr); the rest come from the [blog post of 6 April 2026](https://www.microsoft.com/en-us/security/blog/2026/04/06/ai-enabled-device-code-phishing-campaign-april-2026/).

| Alert | Product | Documented |
|---|---|---|
| [Anomalous OAuth device code authentication activity](https://learn.microsoft.com/en-us/defender-for-identity/alerts-xdr#anomalous-oauth-device-code-authentication-activity) | Defender for Identity | Feb 2026 |
| [Suspicious sign-in from an unusual user agent and IP address using device code flow](https://learn.microsoft.com/en-us/defender-for-identity/whats-new) | Defender for Identity | Mar 2026 |
| [Suspicious Azure authentication through possible device code phishing](https://www.microsoft.com/en-us/security/blog/2026/04/06/ai-enabled-device-code-phishing-campaign-april-2026/) | Defender XDR | Apr 2026 |
| [User account compromise via OAuth device code phishing](https://www.microsoft.com/en-us/security/blog/2026/04/06/ai-enabled-device-code-phishing-campaign-april-2026/) | Defender XDR | Apr 2026 |
| [User account compromised by device code phishing](https://www.microsoft.com/en-us/security/blog/2026/04/06/ai-enabled-device-code-phishing-campaign-april-2026/) | Defender for Identity | Apr 2026 |
| [Suspicious device code authentication following a URL click in an email from rare sender](https://www.microsoft.com/en-us/security/blog/2026/04/06/ai-enabled-device-code-phishing-campaign-april-2026/) | Defender XDR | Apr 2026 |
| Device registration after potential device code phishing | Defender XDR | undocumented |
| [Predelivery protection for device code phishing emails](https://www.microsoft.com/en-us/security/blog/2026/04/06/ai-enabled-device-code-phishing-campaign-april-2026/) | Defender for Office | Apr 2026 |

One point that is often misunderstood: these Defender for Identity alerts do not need an on-premises MDI sensor. They are driven by Entra ID sign-in behaviour, so they fire in cloud-only tenants as well — the prerequisite is MDI licensing for the users, not a deployed sensor.

Entra ID Protection has [no dedicated device code risk detection](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-risks) — coverage there only happens indirectly, through generic signals.

>💡 One thing worth keeping in mind: the date in the documentation is not the date a detector actually went live. When I lined the alert names up against real telemetry, *Suspicious Azure authentication through possible device code phishing* had already been firing about five months before it was documented. That matters for any time-series reading: if you interpret a rise in alert volume without knowing when each detector actually went live, you read the rollout of new detectors as an escalation of the threat.

# Prevention — the more effective route

It is worth ordering the prevention measures by how much they actually help, from weakest to strongest — so that is the order I use here.

**1. User awareness.** The weakest step for this particular attack, for the reason above — the user is on the genuine page, doing the right thing. Awareness is not worthless (recognising an unexpected code request is something), but do not expect it to carry the load.

**2. Filter the transport (Defender for Office).** Microsoft's predelivery protection catches device code lures in mail, and you can build your own mail detection on top (more on that below). This is real value, but it is a lower bound: only mail, only known patterns, and everything that arrives over Teams, a QR code or a messaging app goes around it.

**3. Block device code flow in Conditional Access.** The [authentication flows condition](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-block-authentication-flows) blocks the flow directly. The pattern is *block with an exception*, not *don't block*: `includeUsers = All`, and in `excludeUsers` the handful of accounts that genuinely need it (room systems), optionally narrowed further to a trusted network location. Microsoft has a [dedicated guide for Teams devices](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-teams-devices-device-code-flow).

**4. Require a compliant device in Conditional Access.** The strongest step. Requiring a compliant or hybrid-joined device blocks device code flow as well. The reason is a bit indirect: the flow runs on an unregistered device that can never satisfy the grant, so the sign-in simply fails with `AADSTS53000`. Fabian Bader put it bluntly, and I agree: this barrier is, as far as anyone knows, impossible to fake, so the attacker cannot get a usable token in the first place.

There are two honest caveats to step 4, and both are worth stating:

- **The device registration itself is not covered.** Requiring a compliant device stops device-code access to your resources, but it does not protect the device-registration escalation — I come back to that in its own section below.
- **It is strong, but not absolute.** The ways to get around the compliance requirement itself are their own topic and out of scope here — worth knowing they exist, but they do not change the recommendation.

>💡 These build on each other, they do not replace each other — treat the ladder as measures to stack, not a menu to pick the strongest from. The two Conditional Access policies especially are not an either/or: blocking device code flow and requiring a compliant device cover *different* parts of the attack (the block reaches the device-registration/PRT path, the compliant-device requirement covers resource access — more on that below), so run both.

## Where the block actually takes effect

This is not an academic question. It decides what a blocked attack leaves behind, and what the victim notices.

**At the start, Conditional Access cannot do anything.** The [device authorization request](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code) contains only `client_id` and `scope`:

```http
POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/devicecode
client_id=...&scope=user.read openid profile
```

No user, no authentication, no identity. Conditional Access is a policy over user sign-ins, so at this point there is nothing to evaluate. The attacker can generate a code in any tenant, regardless of any policy — that is not a gap, it follows from the protocol. And it leaves no trace in the target's sign-in log: for the defender, the attack starts out invisible.

**At the end, it does take effect.** Microsoft enforces Conditional Access at token issuance. For device code flow that is the moment the polling client fetches its token. The error fits — `AADSTS53003: Access has been blocked by Conditional Access policies. The access policy does not allow token issuance.` names token issuance as the point where the policy applies.

**In between, the victim authenticates all the way through.** They go to the real Microsoft page, enter username and password, confirm MFA. Those factors are really exercised before anything is blocked.

Whether the victim then sees the block in the browser, or the attempt only fails on the attacker's side, is not cleanly documented — the sources I found contradict each other. If you want to know for sure, you have to reproduce it in a tenant with an active policy.

So Conditional Access here is a **late** control. It prevents the compromise, but only in the last step, and that has three consequences:

- The campaign works up to the second-to-last step. The victim was successfully social-engineered. "CA blocks device code" does not mean the users saw through the trick — awareness still matters.
- Password and MFA were exercised. The attacker gets no token, but learns that the target is reachable and plays along. For the next attempt over a different vector, that is useful.
- There is no early warning from the policy, and the block is barely visible in the log, because `AuthenticationProtocol` is not set on failure.

None of this argues against the block — it argues against treating it as the whole story. Conditional Access does not replace the mail and Teams detection, it complements it: the detection fires at the start of the chain, where CA structurally cannot see anything, and CA takes effect at the end, when the detection missed the vector.

## Two points that are often overlooked

**Report-only blocks nothing.** Obvious when you say it, less obvious in reporting: a policy in report-only mode looks, in many overviews, like an active one. If you measure the protection state through a policy inventory, filter on `state`.

**Protocol tracking produces surprising blocks.** A session that once came up via device code keeps `Original transfer method = Device code flow` across refreshes. As a result, later sign-ins that were not themselves device code can get blocked too — the error for that is `AADSTS530036`. If you suddenly see Teams clients with that code in support, this is the explanation.

## Making the state measurable

The Maester test [`Test-MtCaDeviceCodeFlow`](https://maester.dev/docs/commands/Test-MtCaDeviceCodeFlow) checks whether an *enabled* policy with `authenticationFlows.transferMethods = deviceCodeFlow` exists that covers all users and all client app types and either blocks or requires compliant/domain-joined devices. Report-only policies are correctly excluded.

Two things it does not check, which you should look at yourself. First, it never inspects the target resources or their exclusions at all — so a policy passes the test even if it excludes the Device Registration Service (more on why that matters in its own section below), and even if it does not target all resources. Second, it does not look at how large `excludeUsers`/`excludeGroups` actually are. A passing test is necessary, not sufficient — with six room accounts in the exception all is well; with a five-hundred-member group the passing test is worth very little.

>💡 There is a nice second use for the policy state. If a tenant blocks device code flow, the attack cannot succeed, so a mail alert can be closed deterministically with the policy state as the justification. That turns a posture test from a reporting artefact into an input for alert handling — checked once per tenant instead of argued per incident.

## Building the exception list

Before you cut that exception, you need to know who actually uses device code flow legitimately — otherwise you either break something or exclude too much. This is the one question the interactive field answers well — *who signed in via device code?* — and those successful sign-ins are low-volume, so you can look back over a long window cheaply. If you work in Defender advanced hunting rather than Sentinel, you also avoid pulling the high-volume non-interactive logs in at all.

```kql
SigninLogs
| where AuthenticationProtocol == "deviceCode"   // success-only, which is exactly what we want here
| extend Direction = case(
      tostring(HomeTenantId)     != "<your-tenant-id>", "inbound",   // external guest
      tostring(ResourceTenantId) != "<your-tenant-id>", "outbound",  // own user, external
                                                        "internal")
```

`HomeTenantId` against `ResourceTenantId` separates the three cases cleanly — much more reliably than filters on UPN patterns. What remains internal, you classify. In practice two groups stand out: room and kiosk systems (many active days, one or two IPs, only Teams apps), and admin tooling (Azure CLI, Azure PowerShell, Graph Command Line Tools against ARM, Key Vault, Storage — legitimate, but see the note on admin tooling below).

Only reach for the non-interactive log (`OriginalTransferMethod == "deviceCodeFlow"`) if you need to catch a long-lived session whose original device-code sign-in predates your window — protocol tracking keeps that flag set across refreshes — knowing that table is far larger and more expensive to query.

>💡 That result is your exclusion list for the Conditional Access policy — and, if you later build your own detection, the same list is its allowlist. Hang the exception on a group, not on individual accounts, otherwise new room systems are missed and the policy rots quietly.

## Don't forget your own admin tooling

One pattern that is easy to overlook: consultants and administrators use `az login --use-device-code` because it is convenient in cross-tenant or headless contexts. Those accounts are typically highly privileged, and a successful phish against them is especially valuable. The answer is not to exempt them from the block — that would be the wrong direction — but to get the flow out of the daily routine: `az login` and `Connect-AzAccount` use the browser flow by default, and pipelines belong on managed identity or workload identity federation. Once you no longer need device code yourself, you can recommend the block to customers without reservation, too.

# Device registration and Conditional Access

Around device registration and Conditional Access there are two exceptions that get mixed up regularly. One you configure yourself, the other Microsoft has set in the background. Both are relevant for device code phishing, but in completely different ways.

## The explicit one: the Device Registration Service

Since early September 2024, Microsoft also enforces authentication flows policies on the Device Registration Service — but only for policies that target **all resources**. The [documentation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-authentication-flows) adds:

> If your organization currently uses Device Code Flow for device registration purposes, and you have an authentication flows policy targeting all resources, you need to exempt the Device Registration Resource from the scope of your Conditional Access policy to avoid impact.

That is often read as a blanket instruction, but it has two conditions that *both* have to hold — an all-resources policy **and** actually using device code flow for registration — and its purpose is *"to avoid impact"*: an availability note, not a security recommendation.

So the consequence is the other way round. If you do **not** use device code flow for registration — the majority — **do not exclude** the Device Registration Service, and the block covers exactly the path the PRT chain runs over. The real risk is the exclusion being set preventively just because it is in the docs: the tenant then looks blocked in every overview while staying open at the one spot that turns a short-lived token into durable access.

For a review it comes down to one question: *do you use device code flow for device registration? If not — why is the Device Registration Service excluded?* If it really is needed, the exception sits under **Target Resources → Exclude → Select excluded cloud apps → Device Registration Service** (client ID `01cb2876-7ebd-4aa4-9cc9-d28bd4d359a9`).

## The implicit one: compliance cannot apply during registration

The second exception is one no administrator sees, because it is not in any policy. To enrol a device in Intune you cannot satisfy *"require device to be marked as compliant"* — the device is not compliant yet at that point — so Microsoft exempts the registration path in the backend.

This is the concrete reason behind the *run both policies* point from the ladder above. A compliance requirement is not a reliable backstop against the PRT chain, because the attacker registers a device on exactly the path that is exempt by design; what actually closes that leg is the authentication flows **block** (all resources, Device Registration Service not excluded). Requiring a compliant device *on the device code flow itself* is not a substitute — it runs into the same exemption. And for response, the registered device object has to be removed; you cannot rely on compliance to lock it out afterwards. (How far this exemption reaches, and how it can be abused more broadly, is its own topic — the *Device Compliance Bypass* in our talk — out of scope here.)

## Hardening the registration action

One more lever, not watertight but worth having: scope a Conditional Access policy to the **"Register or join devices" user action** and require a phishing-resistant [authentication strength](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strengths) (with a Temporary Access Pass for legitimate onboarding). The phish hands the attacker a token off the victim's *ordinary* MFA, so demanding a phishing-resistant factor at registration asks for something that token does not carry — and it helps against anything that registers a device off a phished token, not just this attack.

The caveat is the familiar one: it only bites if the registration path actually evaluates the user action — the same exemptions that weaken the compliance requirement apply here — and it does nothing if the phished session was already phishing-resistant. A layer, not a guarantee.

# Detection — what the documentation doesn't tell you

This is the part where I spent the most time, and where the practically useful findings sit. Two of them are about single fields in the sign-in logs, and both surprised me.

A couple of practical notes on the queries first:

- **Combining the two sign-in tables.** A `union` of `SigninLogs` and `AADNonInteractiveUserSignInLogs` collides on column types — you end up with `_string`/`_dynamic` suffixes and silently empty results. Use Fabian Bader's [helper for unified sign-in logs](https://cloudbrothers.info/unified-sign-logs-advanced-hunting/) instead of harmonising by hand.
- **`CloudAppEvents` naming.** The time column is `Timestamp` in Defender Advanced Hunting but `TimeGenerated` in the Sentinel copy, and the actor is `AccountObjectId`, not a UPN.

## Two fields, two questions

The obvious query is this:

```kql
SigninLogs
| where AuthenticationProtocol == "deviceCode"
```

The part that is not written down anywhere: **this field is only populated for successful sign-ins.** Blocked or failed device code attempts do not appear here at all.

This observation originally comes from [Fabian Bader](https://cloudbrothers.info/en/protect-users-device-code-flow-abuse/), and it held up in real telemetry: over ninety days, this query returned only rows with `ResultType 0` — not a single failure.

>💡 As so often, Fabian had already written it down long before I looked. By now it is a bit of a running gag — wherever I start looking into an identity topic, Fabian has been there before me, and more often than not his older blog posts turn out to be the only, or simply the best, source on the exact detail I need. Given how long and how closely we actually work together, there is a certain irony in me learning from his archive again and again.

It has two consequences:

1. **A tenant with a hard Conditional Access block looks identical to one where nobody ever clicked.** Both show zero device code sign-ins. If you want to measure the effect of a block through this field, you measure nothing.
2. A query looking for *successful* device code sign-ins needs **no** additional `ResultType` filter. The filter is implicit.

That is one field. The second — `OriginalTransferMethod` in `AADNonInteractiveUserSignInLogs` — is the one that keeps working after that first moment: if a session was built via device code, the value stays at `deviceCodeFlow` across token refreshes, which Microsoft calls [protocol tracking](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-authentication-flows).

The difference in volume is drastic. In an environment I looked at, `AuthenticationProtocol == "deviceCode"` produced a few hundred events over ninety days, against **more than two hundred times** as many over `OriginalTransferMethod == "deviceCodeFlow"`. And in all the non-interactive rows, `AuthenticationProtocol` is set to `none`.

The two fields answer different questions:

| Field | Question | What it shows |
|---|---|---|
| `AuthenticationProtocol == "deviceCode"` | Did someone sign in via device code? | only the successful interactive moment |
| `OriginalTransferMethod == "deviceCodeFlow"` | What is this session doing afterwards? | all follow-up tokens, across refreshes |

For the initial detection, the interactive path is the right one. For **incident response**, `OriginalTransferMethod` is the field that shows what the token touched after it was issued — resources, apps, IPs. In my experience it is almost never used for that. As a side effect, `AADSTS530036` shows up here too — the error code when Conditional Access rejects a protocol-tracked refresh token.

## Mail detection

A mail detection on `EmailUrlInfo` is straightforward, with one thing that is easy to get wrong: a host-based filter on "microsoft" quietly misses **`aka.ms/devicelogin`** — the short link Microsoft uses in its own instructions — and `login.microsoft.com/device`. Make sure the rule matches the concrete patterns:

```kql
Url has_any ("microsoft.com/devicelogin", "login.microsoft.com/device", "aka.ms/devicelogin", "oauth2/deviceauth")
```

`oauth2/deviceauth` is host-independent and also covers the US Gov and China clouds.

One limit to be clear about: this only catches the variant where the device-login URL is actually in the mail. The automated kits deliver a plain attacker link and only redirect to device-login *on click*, so nothing device-login-shaped is in the delivered mail — those you catch on the sign-in side, which does not care how the code was delivered.

## Teams — a real, but low-volume, gap

For device code phishing over Teams chats I currently see no coverage, neither from Microsoft nor in common rule sets — and Safe Links does not close the gap, because the device-login URL is legitimate and there is nothing for it to flag. As I said above, the volume I see over Teams is small, so I would not put this first — but a gap with zero coverage is still worth closing.

The build is more involved than for mail, because three tables come together: `MessageUrlInfo` for the same URL patterns, `MessageEvents` filtered on delivered and external threads, and `CloudAppEvents` (`Application == "Microsoft Teams"`, `ActionType == "MessageSent"`, `IsExternalUser`). The third join is the decisive one: `IsExternalThread` only tells you that external participants are in the chat, not that the suspicious message came *from* an external user. It is also sensible to exempt the sender domain `microsoft.com`, because legitimate Microsoft communication contains such links. MITRE-wise this is [`T1566.003`](https://attack.mitre.org/techniques/T1566/003/) (Spearphishing via Service), as opposed to [`T1566.002`](https://attack.mitre.org/techniques/T1566/002/) for the mail path.

## Making blocked attempts visible anyway

Because `AuthenticationProtocol` never shows blocked attempts, you need a different angle if you want to see them. There are three, and all of them avoid the protocol field:

- **Via the policy:** `ConditionalAccessStatus == "failure"`, then expand `ConditionalAccessPolicies` and filter on the name of your authentication-flows policy. That counts the blocks without depending on the protocol field.
- **Via the error code:** Microsoft's own hunting query correlates `ErrorCode in (0, 50199)` under the same `CorrelationId` — deliberately without `AuthenticationProtocol`. `50199` is the CMSI interrupt of the device code flow.
- **Via the non-interactive log:** `OriginalTransferMethod == "deviceCodeFlow"` — as above, and the cleanest substitute, since it also surfaces `AADSTS530036`.

## Why no alert is not proof of safety

There is a thought error I have seen more than once: *"users are getting phishing mail, but no alert comes in — so the detector is blind."* That is not right. The mail detection fires on delivery, the sign-in detector fires on authentication. **In between sits the click.** Without a click, silence is the expected behaviour.

The most likely explanation for missing sign-in alerts is not a broken detector, either. If the detector evaluates *successful interactive* device code sign-ins, then the population in most tenants is small and stable — room systems, occasional admin tooling, grown habits. None of that is anomalous. No alert is then the correct answer to "there was no successful device code phish", not a failure.

But the reverse is just as true: missing alerts are **not** proof of security. They are only "no indication of a successful compromise".

# Response

**First, the clock.** A device code is short-lived — the user has around fifteen minutes to enter it before it expires. So if a lure was delivered and twenty minutes have passed with no successful device code sign-in for that user, that particular code is dead and the immediate risk from it is over. The useful action then is on the mail side — pull the message, warn the user, and block further mail from the sender — so the attacker cannot walk them into a fresh code later. The caveat: this holds for a *static* code in a mail. The automated kits mint a new code the moment the victim clicks the link, so a lure pointing at a live kit resets the fifteen minutes on every click — there, the mail-side action matters more than the clock.

Beyond that, a few things that often go wrong once a compromise is confirmed.

The escalation from *How it works* — a phished token turned into a Primary Refresh Token and a registered device — is what containment has to undo, and it is why the usual revoke-and-reset reflex falls short here.

**A session revoke alone is not enough.** [`revokeSignInSessions`](https://learn.microsoft.com/en-us/graph/api/user-revokesigninsessions) invalidates the refresh tokens. Existing **access tokens stay valid for up to an hour.** For a confirmed compromise, the temporary disabling of the account belongs to the response — not just revoke plus password reset.

**Registered devices have to go.** If a device was registered in the relevant window, the Primary Refresh Token survives every session revoke. The device object itself has to be removed — and, as the device-registration section explains, you cannot rely on compliance policies to keep a registered attacker device out after the fact. Without that step containment is incomplete — and that is exactly the reason attackers take this path. For the search: device registrations show up in the `AuditLogs` under operations like `Add device` or `Add registered owner`, filtered on the affected initiator.

**Look at the registered authentication methods, too.** The same access lets an attacker add their own sign-in method — a fresh Authenticator, a FIDO key, a phone number — which survives a password reset and hands them a way back in. Check the account's methods for anything registered in the incident window and remove it; in the `AuditLogs` these show up under the authentication-method operations (for example `User registered security info`), again filtered on the affected user.

# Summary

**What Microsoft covers:** mail predelivery, anomalous device code authentication, compromise follow-up alerts, and since mid-2026 also device registration. Solid — but anomaly-based, undocumented in its prerequisites, and not available everywhere.

**What you have to build yourself:**

- Mail detection with **all** URL variants, `aka.ms/devicelogin` in particular
- Teams as a vector — low volume, but currently almost no coverage
- A baseline over `OriginalTransferMethod` to cut the Conditional Access exception cleanly — the same query doubles as a detection allowlist
- `OriginalTransferMethod` as a standard step in incident response

**What helps most:** Conditional Access with a cleanly cut exception for room systems — and requiring a compliant device where you can, with the two caveats in mind. Plus getting your own admin tooling off device code flow. And in the response: remove the device, disable the account temporarily, don't just revoke sessions.

## Sources

**Microsoft**

- [Storm-2372 conducts device code phishing campaign](https://www.microsoft.com/en-us/security/blog/2025/02/13/storm-2372-conducts-device-code-phishing-campaign/) — 13 Feb 2025
- [Defending against evolving identity attack techniques](https://www.microsoft.com/en-us/security/blog/2025/05/29/defending-against-evolving-identity-attack-techniques/) — 29 May 2025
- [Inside an AI-enabled device code phishing campaign](https://www.microsoft.com/en-us/security/blog/2026/04/06/ai-enabled-device-code-phishing-campaign-april-2026/) — 6 Apr 2026
- [Defender for Identity: XDR security alerts](https://learn.microsoft.com/en-us/defender-for-identity/alerts-xdr) · [What's new](https://learn.microsoft.com/en-us/defender-for-identity/whats-new) · [Technical FAQ](https://learn.microsoft.com/en-us/defender-for-identity/technical-faq)
- [Authentication flows as a Conditional Access condition](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-authentication-flows) · [Block authentication flows](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-block-authentication-flows) · [Device code flow for Teams devices](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-teams-devices-device-code-flow)
- [Conditional Access: Grant](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-grant) · [OAuth 2.0 device authorization grant](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code)
- [signIn resource type](https://learn.microsoft.com/en-us/graph/api/resources/signin) · [user: revokeSignInSessions](https://learn.microsoft.com/en-us/graph/api/user-revokesigninsessions) · [AADSignInEventsBeta (deprecated)](https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-aadsignineventsbeta-table)

**Community**

- [Fabian Bader — Protect your users from Device Code Flow abuse](https://cloudbrothers.info/en/protect-users-device-code-flow-abuse/) · [Working with unified sign-in logs in Advanced Hunting](https://cloudbrothers.info/unified-sign-logs-advanced-hunting/)
- [Dirk-jan Mollema — Phishing for Microsoft Entra Primary Refresh Tokens](https://dirkjanm.io/phishing-for-microsoft-entra-primary-refresh-tokens/)
- [Volexity — Phishing for Codes](https://www.volexity.com/blog/2025/04/22/phishing-for-codes-russian-threat-actors-target-microsoft-365-oauth-workflows/)
- [Push Security — Analyzing the rise in device code phishing attacks in 2026](https://pushsecurity.com/blog/device-code-phishing) · [Sekoia — EvilTokens](https://blog.sekoia.io/new-widespread-eviltokens-kit-device-code-phishing-as-a-service-part-1/) · [FBI IC3 — Kali365](https://www.ic3.gov/PSA/2026/PSA260521)
- [Maester: Test-MtCaDeviceCodeFlow](https://maester.dev/docs/commands/Test-MtCaDeviceCodeFlow)

**MITRE ATT&CK**

- [T1528 — Steal Application Access Token](https://attack.mitre.org/techniques/T1528/) · [T1078.004 — Valid Accounts: Cloud Accounts](https://attack.mitre.org/techniques/T1078/004/)
- [T1566.002 — Spearphishing Link](https://attack.mitre.org/techniques/T1566/002/) · [T1566.003 — Spearphishing via Service](https://attack.mitre.org/techniques/T1566/003/)
