---
layout:     post
title:      "Has Enough Time Passed, or Has Something Changed?"
subtitle:   "Sign-in frequency and Continuous Access Evaluation are two answers to the same question about the same token — and together they cover a span no VPN concentrator does"
date:       2099-01-01
draft:      true
author:     "Chris Brumm"
URL:        "/2026/11/Sign-in-Frequency-and-CAE-on-Private-Access/"
tags:
    - Entra
    - Global Secure Access
    - Entra Private Access
    - Continuous Access Evaluation
    - Conditional Access
categories: [ Global Secure Access ]
---

Two dates, a day apart. On 30 October the device signals for Universal Continuous Access Evaluation
quietly lost their preview label. On 1 November, sign-in frequency "every time" arrived for Microsoft
Entra Private Access — as a new Conditional Access control, with three new documentation pages.

Taken together they change what a Global Secure Access session is. For the first time, access to a
private application can be re-authenticated per application, and a session can be ended because a
*device* changed. Neither of those is something a VPN concentrator does. A concentrator authenticates
you to a tunnel once and does not look again; everything behind it stays reachable for as long as the
tunnel lives. These are the per-resource controls that ZTNA was supposed to bring, and they arrive at
the network layer — for applications that know nothing about identity.

## Two controls on one access token

In Private Access, access to a private application needs an access token. That is also why
[a DNS lookup can trigger an MFA prompt](/2026/07/When-DNS-Lookups-Trigger-MFA/) — name resolution
through the private DNS path needs a token just like a TCP connection does.

Both features act on that token:

| | Sign-in frequency | Continuous Access Evaluation |
|---|---|---|
| Question | Has enough time passed? | Has something changed? |
| Direction | proactive — refuses to reuse a token past an age | reactive — invalidates a token that is still valid |
| Acts at | token **issuance** | token **use**, after issuance |
| Mode | **pulled** — runs when something asks for a token | **pushed** — does not wait to be asked |
| Token lifetime | shortens it | lengthens it, and pays with near-real-time revocation |

## Every time strict and the five-minute window

The control to pick is **Every time strict (preview)**, and this is the part most people will get wrong.
Microsoft's configuration guide is unusually direct about it:

> Select **Every time strict (preview)**. The **Every time** option does not apply to Global Secure
> Access private applications.

Read that as guidance on which control to configure, which is clearly how it is meant. I would not read
it as a promise that a plain **Every time** policy pointed at a private application has no effect — those
are different statements, and the documentation does not say which one it intends. [R8-1b] If you still
have such a policy lying around from earlier experiments, the safe assumption is that it may still be
prompting people, and the cheap move is to check rather than to assume it went inert.

`Every time strict` is a new, general-purpose Conditional Access control — you can use it without GSA
at all. It was first documented on 2 November, in the same change that added the Private Access pages.
Its documented purpose is applications that need genuinely fresh authentication for a sensitive action:
e-prescribing, payment approval. Microsoft is explicit that it should target specific applications, and
that it must not be applied to Office 365 apps.

Then there is a detail you only find by reading the Conditional Access page against the Global Secure
Access page. The control itself has a **ten-second** validity window; Private Access has a **five-minute
reauthentication window**. So on a private application you do not get the ten-second freshness the
control is built around — you get a five-minute floor, per application.

Microsoft's own worked example: a user authenticates at 09:01:15, closes
the terminal at 09:01:55, reopens it at 09:03:45 and is *not* prompted. At 09:10:34 they are, because
more than five minutes have passed since 09:01:15. Note what the clock runs from — the sign-in, not the
connection.

[R8] Our own cadence measurement goes here once the lab has run against the shipped control.

> 💡 The older session-lifetime article still lists "Microsoft Entra Private Access doesn't support
> setting sign-in frequency to every time" under Known issues. Microsoft shipped a second, parallel
> page rather than correcting that one, so if you search for this feature you can easily land on the
> page that says it does not exist.

## Token acquisition, not connections

This is the part I found most useful to understand, because once you see it the feature stops being
surprising.

Sign-in frequency is attached to **token acquisition**. It is evaluated when something asks Entra for a
token. Microsoft states the general principle on the Conditional Access page:

> Applying the control alone isn't sufficient if the application continues to use its existing session
> without requesting a new token.

And for Private Access specifically:

> This feature is triggered when a new network connection is established to the protected private
> application. It doesn't guarantee a fresh prompt for every command, transaction, or patient record.

Two things follow.

The first is that a session already in progress is not re-evaluated. No new token is requested, so
nothing reaches Conditional Access. The configuration guide concedes this in the shape of a test
instruction — when it walks you through validating the five-minute window, it tells you to close the
connection and *"ensure that the application process which initiated the connection is completely
closed (this ensures that the network flow is terminated)"*. You have to terminate the flow for the
check to happen at all. [R8] Our own measurement of this goes here once the lab has run against the
shipped control.

The second is why that ten-second window cannot bind here. The control only looks when a token is
requested, and the client asks at most once per five minutes per application — a tighter bar cannot
tighten anything it does not drive.

So sign-in frequency governs *starting* work on an application; Continuous Access Evaluation reaches
work already in progress. That is the real answer to the question I get every time I show this: why
would I still need CAE if I have re-authentication every time?

Microsoft now documents one more consequence:

> Windows devices joined to an on-premises domain periodically check in with a domain controller, which
> may trigger a sign in frequency related authentication prompt without an explicit user action.

Background traffic raising an authentication prompt is exactly the
effect I described in [the DNS posts](/2026/08/Private-DNS-Suffix-and-Segment-Hygiene/), and it is now Microsoft's own
stated reason to prefer per-app segmentation over a broad Quick Access configuration. If you put this
control on Quick Access, you have put it on name resolution.

## Supported clients, and what happens to the others

Sign-in every time is Windows only, and it needs the Global Secure Access client **2.34.222 or later**.
Below that version, or on any other operating system, access to an enforced application is **blocked**.
Microsoft's own test table says "Access is blocked" for both cases, and it calls these "blocking
policies" when it reminds you to exclude your emergency access accounts.

The documented mitigation is a Conditional Access **Device platforms** condition scoped to Windows,
which leaves you two options:

| Approach | What you get |
|---|---|
| Scope to Windows | macOS, iOS and Android "aren't subject to this policy and can connect if their other access requirements are satisfied" — the application stays reachable from those platforms, without the step-up |
| Leave platforms unscoped | those devices are blocked |

The first one fails quietly: the policy reports success and the application looks protected.

Client versions are the sharper edge. A Device platforms condition filters by platform, not by client
version, so a Windows device carrying an older client sits inside a Windows-only policy and is blocked
with nothing to rescue it — your client update pipeline becomes an availability dependency for that
application. Microsoft effectively acknowledges this by recommending Report-only mode plus the traffic
logs to audit which versions your users actually run before you enable anything.

For a step-up control on an administrative jump box, denying an under-versioned client is arguably the
right answer, and because the platform condition exists the choice is yours. Universal CAE resolves the
same kind of dependency the other way: it needs the Windows client 1.8.239.0 or later, and on other
platforms it simply "uses regular access tokens". One control denies access when it cannot apply, the
other stops protecting without saying so.

## Seeing it in your own tenant

Three things I needed while working on this are not in the documentation, so here they are. The column
names come from a real tenant, but you should still sanity-check each query against yours.

**Catching a sign-in frequency event.** The event emits *two* different error codes at two different
endpoints in the same flow: **70043** at the token endpoint, when the silent refresh is refused, and
**70046** at the authorize endpoint, when the interactive leg opens. The canonical AADSTS reference
carries only 70043; 70046 lives in a different article. So a rule keyed on either one alone sees half
the event, and may see nothing.

```kusto
EntraIdSignInEvents
| where Timestamp > ago(7d)
| where Application startswith "ZTNA Network Access Client"
| where ErrorCode in (70043, 70046)
| mv-apply d = parse_json(AuthenticationProcessingDetails) on (
    where tostring(d.key) == "Is CAE Token"
    | project CaeToken = tostring(d.value)
  )
| project Timestamp, AccountUpn, ResourceDisplayName, EndpointCall, ErrorCode, CaeToken, CorrelationId
| order by Timestamp asc
```

Group by `CorrelationId` and you see one re-authentication as one flow rather than as four unrelated
failures. If you only have the Entra diagnostic settings rather than the Defender XDR table, use
`ResultType` in `SigninLogs` — but include `AADNonInteractiveUserSignInLogs`, because the client's token
requests are non-interactive and will otherwise be missing entirely.

The same query answers a question the portal does not: whether Universal CAE is actually in effect.
`Is CAE Token` is a readable field inside `AuthenticationProcessingDetails`, and on the Private Access
application it reads **False** — so from the application an administrator would think to check,
Universal CAE is invisible.

**Watching a token get reused.** This is the one I would start with, because it shows the mechanism from
*Token acquisition, not connections* directly, with no lab setup. The Global Secure Access traffic logs carry the access
token's identity and lifetime per transaction:

```kusto
NetworkAccessTraffic
| where TimeGenerated > ago(1d)
| where AccessType == "PrivateAccess"
| where UserPrincipalName =~ "someone@contoso.com"
| summarize Connections = dcount(ConnectionId),
            FirstSeen   = min(ConnectionCreatedTime),
            LastSeen    = max(TimeGenerated)
    by Token3PUniqueId, Token3PIssuedAt, Token3PExpiry
| order by Token3PIssuedAt asc
```

One `Token3PUniqueId` against many `ConnectionId` values is token reuse, visible in your own data. That
is sign-in frequency not being consulted, because nothing asked for a token. `Token3PExpiry` minus
`Token3PIssuedAt` also gives you the real token lifetime instead of the documented range.

**Auditing client versions before you enable anything.** The configuration guide tells you, twice, to
"use the Global Secure Access Traffic Logs to validate the versions of GSA client that these users are
using" — but the documented portal view of those logs shows only a device *category*, not a version. The
version is in the Log Analytics table:

```kusto
NetworkAccessTraffic
| where TimeGenerated > ago(7d)
| where DeviceCategory == "Client"
| summarize Devices = dcount(DeviceId), Users = dcount(UserPrincipalName), LastSeen = max(TimeGenerated)
    by AgentVersion, DeviceOperatingSystem
| order by AgentVersion asc
```

One caveat, and I would rather flag it than pretend: `AgentVersion` is documented only as "the version
of the agent connecting", and it sits among a group of fields that are clearly about AI agents. Check in
your tenant that it carries the Global Secure Access client version before you rely on it for a go/no-go
decision. If it does not, Defender for Endpoint's software inventory gives you the same answer from
another direction:

```kusto
DeviceTvmSoftwareInventory
| where SoftwareName has "secure_access"
| summarize Devices = dcount(DeviceId) by SoftwareVendor, SoftwareName, SoftwareVersion
| order by SoftwareVersion asc
```

> 💡 For error codes in general, Fabian Bader maintains
> [a list of Entra ID error codes with their messages](https://cloudbrothers.info/en/entra-id-azure-ad-signin-errors/),
> published as JSON in [f-bader/EntraID-ErrorCodes](https://github.com/f-bader/EntraID-ErrorCodes),
> together with a KQL pattern that joins it onto your sign-in logs so the generic "Other" descriptions
> are replaced by readable ones. It is the fastest way to stop guessing what a code means, and it covers
> far more than the canonical reference does.

## Universal CAE and the device signal

Universal CAE extends Continuous Access Evaluation to anything reached through Global Secure Access,
without the application having to be CAE-aware. That is the part I keep coming back to: an SSH jump box
or an old file share is never going to implement OIDC, and here it does not have to. Tokens live 60 to
90 minutes and support near-real-time revocation. I wrote about the mechanics in
[Token Replay Protection and the Compliant Network Check](/2026/04/Token-Replay-Protection-and-the-Compliant-Network-Check/) and will
not repeat the diagrams.

Until 30 October, CAE had never had a device signal — neither classic CAE nor Universal CAE listed a
device event. A device that fell out of compliance, got disabled or got deleted did not touch a Global
Secure Access session that was already running. Device state was enforced at issuance, which meant the *next* token request was blocked while
the current one ran to expiry. That was the gap in the "ZTNA replaces the VPN" story:
Conditional Access per application segment is true at the moment of issuance and silent for up to a
token lifetime afterwards.

That gap is now closed, and it is generally available. The chain runs from Defender for Endpoint's
machine risk score, through an Intune compliance policy that consumes it, into the Entra device state,
and from there into CAE. In our lab the user-facing sequence was a toast in the Global Secure Access
client followed by the tunnels disconnecting **2 minutes 3 seconds** later. For comparison, revoking a
user's refresh tokens produced the same effect with a **2 minute 9 second** gap.

Two caveats worth knowing. A device with no registered owner does not get device-compliance revocation —
that is a documented limitation and it will catch some Hybrid-joined estates. And Microsoft states that
the signals are meant to be used together with the corresponding Conditional Access policy; without one,
the re-authentication is silent and there is nothing to act on.

One note on novelty: user risk has been in the public Universal CAE signal list since the December 2024
state of the documentation, so of the "new signals" only the device one is genuinely new.

> 💡 The known-limitations page — edited two days *after* this feature shipped — still says Conditional
> Access policies for Private Access traffic are not supported, and still describes the CAE disconnect
> as three prompts with a six-minute grace period, roughly ten minutes in total. The concept page says
> two minutes, which is what I measured. [R7] I also only ever saw one prompt — but both runs had two
> traffic forwarding profiles connected, and I haven't finished testing whether the prompt count tracks
> the number of profiles. If it does, the two pages aren't in conflict after all.

## Summary

Sign-in frequency is attached to token acquisition, so it gates starting work on an application rather
than the work itself, at a five-minute granularity you cannot tighten. It is Windows-only, it needs the
Global Secure Access client 2.34.222 or later, and it denies access below that — audit your client
versions and decide the platform question before you enable anything, or the default decides it for you.

Universal CAE is the one that reaches a session already in progress, at no cost to the user when nothing
is wrong, and since 30 October it knows about devices too.

So: Universal CAE as the default session-security mechanism for Global Secure Access, and sign-in every
time used surgically — `Every time strict`, per application, where a fresh authentication at the start
of the work is worth the interruption. An administrative jump box is the canonical case. Never as a
blanket policy, and never on Quick Access, which is where DNS lives. Used where it belongs, it does
exactly what it says. Microsoft's own warning still holds: prompting too often "can affect their
productivity and increase the risk of users approving MFA requests they didn't initiate" — though on
Private Access the five-minute window caps that for you.

Together they cover a span a concentrator does not: authentication freshness scoped to a single
application and enforced by the network layer, plus identity and posture changes propagating into a
session that is already running, in about two minutes.

## References

Microsoft Learn: *Microsoft Entra Private Access sign-in every time (preview)*, *Configure sign-in every
time for Microsoft Entra Private Access (preview)*, *Conditional Access adaptive session lifetime
policies*, *Universal Continuous Access Evaluation*, *Known limitations for Global Secure Access*.

Earlier posts in this series: [When DNS Lookups Trigger MFA](/2026/07/When-DNS-Lookups-Trigger-MFA/),
[Private DNS Hygiene](/2026/08/Private-DNS-Suffix-and-Segment-Hygiene/), Token Replay Protection and the Compliant Network
Check, Global Secure Access in Conditional Access.

