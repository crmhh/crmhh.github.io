---
layout:     post
title:      "Next Steps Toward ZTNA — Sign-in Frequency Every Time and Universal CAE Device Signals"
subtitle:   "What each of them improves, how they divide the work, and why Private Access needs both."
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
[a DNS lookup can trigger an MFA prompt](https://chris-brumm.com/2026/07/When-DNS-Lookups-Trigger-MFA/) — name resolution
through the private DNS path needs a token just like a TCP connection does.

Both features act on that token:

| | Sign-in frequency | Continuous Access Evaluation |
|---|---|---|
| Question | Has enough time passed? | Has something changed? |
| Direction | proactive — refuses to reuse a token past an age | reactive — invalidates a token that is still valid |
| Acts at | token **issuance** | token **use**, after issuance |
| Mode | **pulled** — runs when something asks for a token | **pushed** — does not wait to be asked |
| Token lifetime | shortens it | lengthens it, and pays with near-real-time revocation |

| ![Picture 1: sign-in frequency and Universal CAE on the same access token](/post/2026/Sign-in-Frequency-and-CAE-on-Private-Access/images/sif-vs-cae-timeline.svg) |
|:--:|
| *Picture 1: sign-in frequency walls in from the left, driven by the clock and evaluated whenever a token is requested. Universal CAE arrives from above, when something changes. One shortens the token's life, the other revokes it mid-flight.* |

## Every time strict and the five-minute window

The control to pick is **Every time strict (preview)**, and this is the part most people will get wrong.
Microsoft's configuration guide is unusually direct about it:

> Select **Every time strict (preview)**. The **Every time** option does not apply to Global Secure
> Access private applications.

Read that as guidance on which control to configure, which is clearly how it is meant. I would not read
it as a promise that a plain **Every time** policy pointed at a private application has no effect — those
are different statements, and the documentation does not say which one it intends. If you still have
such a policy lying around from earlier experiments, the safe assumption is that it may still be
prompting people, and the cheap move is to check rather than to assume it went inert.

`Every time strict` is a new, general-purpose Conditional Access control — you can use it without GSA
at all. It was first documented on 2 November, in the same change that added the Private Access pages.
Its documented purpose is applications that need genuinely fresh authentication for a sensitive action:
e-prescribing, payment approval. Microsoft is explicit that it should target specific applications, and
that it must not be applied to Office 365 apps.

Then there is a detail you only find by reading the Conditional Access page against the Global Secure
Access page. The control itself has a **ten-second** validity window; Private Access has a **five-minute
reauthentication window**, per application. So on a private application you do not get the ten-second
freshness the control is built around.

The ten seconds are not overridden by the five minutes. They are only consulted when the client asks
for a token, and the client holds one access token per private application and reuses it. The interval
that governs a private application is therefore the client's, not the policy's — which is the difference
between a control you can tighten and one you cannot.

Microsoft's own worked example: a user authenticates at 09:01:15, closes
the terminal at 09:01:55, reopens it at 09:03:45 and is *not* prompted. At 09:10:34 they are, because
more than five minutes have passed since 09:01:15. Note what the clock runs from — the sign-in, not the
connection.

In my lab a connection opened a little over five minutes after the previous token was issued was refused
with `70043` and prompted; one opened **twenty-six seconds** after a token was issued produced no request
at all — no sign-in row of any kind. I did not probe the boundary itself, so the five minutes is
Microsoft's figure rather than mine. What the measurements do show is that no policy setting moves it.

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
check to happen at all. I measured this against the shipped control: an established RDP session stayed up
for **more than two hours**, across better than twenty of those five-minute boundaries, with no drop and
no prompt — while a second connection opened moments after one of them, to a different host in the same
application, was gated. The live session is not what is being checked, and nothing about it exempts the
next connection either.

The second is that the token is held **per application**, not per application segment — worth knowing
if you segment finely. A connection to a host in a different segment of the same application reuses the
same token, so it is never evaluated. Splitting an application into segments buys you no additional
re-authentication: the application is the unit, and a separate gate needs a separate application.

| ![Picture 2: what sign-in frequency reaches on an RDP session, and what it does not](/post/2026/Sign-in-Frequency-and-CAE-on-Private-Access/images/rdp-session-sif-vs-cae.svg) |
|:--:|
| *Picture 2: an admin session to a domain controller. New connections are gated, but only when the client actually asks for a token — at most once every five minutes, per application. The session already running never asks, which is why until October nothing could end it.* |

So sign-in frequency governs *starting* work on an application; Continuous Access Evaluation reaches
work already in progress. That is the real answer to the question I get every time I show this: why
would I still need CAE if I have re-authentication every time?

Microsoft now documents one more consequence:

> Windows devices joined to an on-premises domain periodically check in with a domain controller, which
> may trigger a sign in frequency related authentication prompt without an explicit user action.

Background traffic raising an authentication prompt is exactly the
effect I described in [the DNS posts](https://chris-brumm.com/2026/08/Private-DNS-Suffix-and-Segment-Hygiene/), and it is now Microsoft's own
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

A gate has a constant head: four `70043` rows, then a `70046`. `LogonType` separates those two codes more
reliably than the endpoint does — the `70043` rows are `nonInteractiveUser` and the `70046` is
`interactiveUser`. The tail varies. Three of my gates ended in a single success, a `nonInteractiveUser`
row carrying `ErrorCode 0`; a fourth ended in two, an `interactiveUser` success followed by the
`nonInteractiveUser` one. Count gates on the head rather than on the success rows. End to end, from the
first refused request to the connection being established, a gate cost between 25 and 29 seconds, about
nine of which were the user approving.

Assembling those rows into one event is harder than it looks, and `CorrelationId` is the trap. In my
tenant 84 of these rows carried only **four** `CorrelationId` values, one of them spanning four hours
and several unrelated re-authentications — it is stable per application, not per flow. `RequestId` and
`UniqueTokenId` are per request, which makes them right for counting and for dropping duplicate rows (I
have seen the same `70046` row twice under an identical `UniqueTokenId`), but they group nothing either.
What delimits one gate is time: a burst inside about twenty seconds, with the next gate at least half a
minute away.

```kusto
EntraIdSignInEvents
| where Timestamp > ago(7d)
| where Application startswith "ZTNA Network Access Client"
| where ErrorCode in (70043, 70046)
| distinct Timestamp, AccountUpn, ResourceDisplayName, EndpointCall, ErrorCode, UniqueTokenId
| order by AccountUpn asc, ResourceDisplayName asc, Timestamp asc
| extend Gate = row_window_session(Timestamp, 1h, 30s,
      AccountUpn != prev(AccountUpn) or ResourceDisplayName != prev(ResourceDisplayName))
| summarize Requests = dcount(UniqueTokenId), Codes = make_set(ErrorCode),
            Start    = min(Timestamp),
            Seconds  = datetime_diff('second', max(Timestamp), min(Timestamp))
    by Gate, AccountUpn, ResourceDisplayName
| order by Start asc
```

That gives one row per re-authentication, with the number of token requests it took to get through.

If you only have the Entra diagnostic settings rather than the Defender XDR table, use
`ResultType` in `SigninLogs` — but include `AADNonInteractiveUserSignInLogs`, because the client's token
requests are non-interactive and will otherwise be missing entirely.

One caveat about that query, and it cost me a wrong conclusion before I noticed: **drop the error-code
filter when you want to prove a negative.** The telling evidence for how this control behaves is usually
a row that is *not* there, and filtering on 70043 and 70046 hides the difference between "Conditional
Access evaluated this and allowed it" and "nothing ever asked for a token".

```kusto
EntraIdSignInEvents
| where Timestamp > ago(1h)
| where Application startswith "ZTNA Network Access Client"
| project Timestamp, ResourceDisplayName, LogonType, ErrorCode, UniqueTokenId, SessionId
| order by Timestamp asc
```

An empty stretch in that output, while connections are being opened, is the mechanism from
*Token acquisition, not connections* in its most direct form.

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
| where TrafficType == "private"
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

The same rows also tie back to the sign-in that issued the token, though not on the column you would
reach for first. `NetworkAccessTraffic` carries two token identifiers: `Token3PUniqueId`, the token used
against the private application, and a plain `UniqueTokenId`, documented in exactly the same words as
the sign-in column of that name. The identically named pair is the trap — `UniqueTokenId` was **empty on
every Private Access row I looked at**. The join runs on `Token3PUniqueId`:

```kusto
NetworkAccessTraffic
| where TimeGenerated > ago(1d)
| where TrafficType == "private"
| where UserPrincipalName =~ "someone@contoso.com"
| summarize Connections = dcount(ConnectionId),
            FirstSeen   = min(ConnectionCreatedTime),
            LastSeen    = max(TimeGenerated)
    by Token3PUniqueId, Token3PIssuedAt, Token3PExpiry
| join kind=inner EntraIdSignInEvents on $left.Token3PUniqueId == $right.UniqueTokenId
| order by Token3PIssuedAt asc
```

That gives every reused token the sign-in that minted it, which is the whole chain in one result set: the
re-authentication, the token it produced, and the connections that then ran on it without any further
evaluation.

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

`AgentVersion` is documented only as "the version of the agent connecting", and it sits among a group of
fields that are otherwise about AI agents, so I checked: it carries the Global Secure Access client
version, matching the installed build exactly. If you would rather have a second source, or your users
are not generating Private Access traffic yet, Defender for Endpoint's software inventory answers the
same question from another direction:

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

## The device signal in Universal CAE

Universal CAE extends Continuous Access Evaluation to anything reached through Global Secure Access,
without the application having to be CAE-aware — an SSH jump box or an old file share is never going to
implement OIDC, and here it does not have to. Tokens live 60 to 90 minutes and support near-real-time
revocation; the mechanics are in
[Token Replay Protection and the Compliant Network Check](https://chris-brumm.com/2026/04/Token-Replay-Protection-and-the-Compliant-Network-Check/).

Until 30 October, it had no device signal — neither classic CAE nor Universal CAE listed a
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

| ![Picture 3: the MDE to Intune to Entra chain, and where it takes effect](/post/2026/Sign-in-Frequency-and-CAE-on-Private-Access/images/device-signal-mde-chain.svg) |
|:--:|
| *Picture 3: the detection chain — Defender for Endpoint machine risk, an Intune compliance policy that consumes it, the Entra device state — has worked for years. What changed in October is that the result reaches a session that is already running.* |

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
than the work itself, at a five-minute granularity you cannot tighten, and per application rather than
per application segment. It is Windows-only, it needs the
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

Neither control is complete on its own, and together they are the first time a Private Access session
is governed per application both when it starts and while it runs.

| ![Picture 4: three ways to interrupt one access token](/post/2026/Sign-in-Frequency-and-CAE-on-Private-Access/images/sif-cae-combined.svg) |
|:--:|
| *Picture 4: one access token and three ways it stops being good: the clock, an identity event, and — since October — the device. The bottom two panels are the same policy and the same compliance state; what changed is where it takes effect.* |

## Attribution and References

[THANKS] Thanks to … for input, discussion and proofreading.

- [Microsoft Learn: Microsoft Entra Private Access sign-in every time (preview)](https://learn.microsoft.com/entra/global-secure-access/concept-private-access-sign-in-every-time)
- [Microsoft Learn: Configure sign-in every time for Microsoft Entra Private Access (preview)](https://learn.microsoft.com/entra/global-secure-access/how-to-configure-sign-in-every-time)
- [Microsoft Learn: Conditional Access adaptive session lifetime policies](https://learn.microsoft.com/entra/identity/conditional-access/concept-session-lifetime) — where **Every time strict (preview)** is specified
- [Microsoft Learn: Configure adaptive session lifetime policies](https://learn.microsoft.com/entra/identity/conditional-access/howto-conditional-access-session-lifetime) — the older article, still carrying the Known issue
- [Microsoft Learn: Universal Continuous Access Evaluation](https://learn.microsoft.com/entra/global-secure-access/concept-universal-continuous-access-evaluation)
- [Microsoft Learn: Known limitations for Global Secure Access](https://learn.microsoft.com/entra/global-secure-access/reference-current-known-limitations)
- [Microsoft Learn: Conditional Access device platform conditions](https://learn.microsoft.com/entra/identity/conditional-access/concept-conditional-access-conditions#device-platforms)
- [Microsoft Learn: NetworkAccessTraffic table reference](https://learn.microsoft.com/azure/azure-monitor/reference/tables/networkaccesstraffic) — the traffic-log columns used in the queries above
- [Fabian Bader: Other Entra ID / Azure AD SignIn errors](https://cloudbrothers.info/en/entra-id-azure-ad-signin-errors/) and [f-bader/EntraID-ErrorCodes](https://github.com/f-bader/EntraID-ErrorCodes)
- [Chris Brumm: When DNS Lookups Trigger MFA (2026)](https://chris-brumm.com/2026/07/When-DNS-Lookups-Trigger-MFA/)
- [Chris Brumm: Private DNS Suffix and Segment Hygiene (2026)](https://chris-brumm.com/2026/08/Private-DNS-Suffix-and-Segment-Hygiene/)
- [Chris Brumm: Token Replay Protection and the Compliant Network Check (2026)](https://chris-brumm.com/2026/04/Token-Replay-Protection-and-the-Compliant-Network-Check/)
- [Chris Brumm: Global Secure Access in Conditional Access (2024)](https://chris-brumm.com/2024/08/06/Global-Secure-Access-in-Conditional-Access/)
