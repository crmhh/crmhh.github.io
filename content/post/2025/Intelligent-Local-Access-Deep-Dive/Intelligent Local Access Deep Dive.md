---
layout:     post 
title:      "Intelligent Local Access Deep Dive"
subtitle:   "This blog post is about the Intelligent Local Access feature of Entra Private Access that allows to bypass local traffic while doing pre-authentication in Entra ID"
date:       2025-11-19
author:     "Chris Brumm"
URL:        "/2025/11/19/Intelligent-Local-Access-Deep-Dive/"
tags:
    - Entra
    - Global Secure Access
    - Entra Private Access
categories: [ Global Secure Access ]
---

Global Secure Access (GSA) enforces that all client traffic is routed through the cloud before reaching the target resource via Private Network Connectors—even if both endpoints are in the same building or network. This design ensures that security controls are consistently applied.

However, not every location has the connectivity of Coruscant; some sites feel more like the Outer Rim—and in Germany, bandwidth limitations can appear quickly. To cope, many users have resorted to disabling the GSA client when on the corporate LAN, a behavior familiar from traditional VPN clients. In some cases, IT teams even automated this process (shoutout to Morten).

This workaround assumes that client-to-resource connections are not blocked by firewalls and it comes at a cost: we lose visibility and control over access. **This means we have no logging and no Entra Authentication including MFA and Device Compliance at the network layer with a disabled GSA client.** So much for Zero Trust Network Access (ZTNA).

Microsoft now offers a middle ground with **Intelligent Local Access (ILA)**. ILA promises to address bandwidth and latency challenges while maintaining strong security controls and delivering an excellent user experience. That’s reason enough to take a closer look at this feature.

## How **Intelligent Local Access works**

| ![Picture 1: Intelligent Local Access Overview](/post/2025/Intelligent-Local-Access-Deep-Dive/images/ILA-overview.png) |
|:--:|
| *Picture 1: Intelligent Local Access Overview* |

In short, the flow is as follows:

1. The client recognizes that it is at a specific location (in this case: private network).
2. Before establishing a network connection to a resource, the client checks whether traffic acquisition is included in the profile for private traffic.
3. If this is the case, the client attempts to obtain an access token and blocks the traffic if it does not receive one.
4. The client then checks whether the resource is local. If so, the traffic is not sent to GSA via a tunnel but runs directly.

| ![Picture 2: Intelligent Local Access Decision Flow](/post/2025/Intelligent-Local-Access-Deep-Dive/images/ILA-decisions.png) |
|:--:|
| *Picture 2: Intelligent Local Access Decision Flow* |

### How does the client know the location?

The following is defined for each private network:

- the IPs of the usable DNS servers
- the A record that is queried
- the expected response(s)

>💡The DNS server configured in the OS, e.g. via DHCP, is not used for the determination, but rather the one stored here.

| ![Picture 3: Private Network Config](/post/2025/Intelligent-Local-Access-Deep-Dive/images/ILA-config.png) |
|:--:|
| *Picture 3: Private Network Config* |

Of course, multiple private networks can be configured, which can then contain different apps.

| ![Picture 4: Mulitple Private Networks](/post/2025/Intelligent-Local-Access-Deep-Dive/images/ILA-multiple.png) |
|:--:|
| *Picture 4: Mulitple Private Networks* |

The DNS queries can be viewed (after removing the default filter on Tunneled) on the client in Advanced Diagnostics, starting from the `GlobalSecureAccessEngineService.exe` process. In the Microsoft `Global Secure Access Client/Operational` event log, event IDs 217 and 218 show which private networks the client is currently connected to.

| ![Picture 5: ILA Network Traffic](/post/2025/Intelligent-Local-Access-Deep-Dive/images/ILA-traffic.png) | ![Picture 6: ILA Events](/post/2025/Intelligent-Local-Access-Deep-Dive/images/ILA-events.png) |
|:--:|:--:|
| *Picture 5: ILA Network Traffic* | *Picture 6: ILA Events* |

> 💡 In my tests, detection was very fast (< 1 minute).

### A brief excursus: Storing the forwarding profile on the client

In my flowchart above, I described that the client checks whether:

- the addressed destinations are included in the forwarding
- the addressed destinations are assigned to a private network
- the private networks are reachable

This information (and much more) is contained in the registry key `forwarding-profile` under `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Global Secure Access Client`.

| ![Picture 7: Forwarding-Profile in Registry](/post/2025/Intelligent-Local-Access-Deep-Dive/images/ILA-registry.png) |
|:--:|
| *Picture 7: Forwarding-Profile in Registry* |

This key contains a JSON structure in which we are interested in the node `policy` with the child nodes `networkDefinitions` and `rules` for this matter.  

Here is an example for a Private Network, including its Id and Name , the FQDN, the expected response and the DNS server with all [IPs in decimal.](https://tools.base64decode.net/decimal-to-ip-converter)

```json
"policy": {
	"networkDefinitions": [
		{
			"networkId": "702408bb-d401-4d96-81ff-0547c0dd7d7c",
			"networkIdentification": [
				{
					"dnsResolution": {
						"expectedIpResolution": [
							{
								"end": 3232235619,
								"start": 3232235619
							}
						],
						"fqdnToResolve": "ILA.gkfelucia.net",
						"serverAddresses": [
							-1062731766
						]
					}
				}
			],
			"networkName": "GKFeluciaDC"
		}
	],
}	
```

And here is an example of an EPA app, including its Application segments (matchingCriteria), the mapping to the Enterprise App (clientAppId) and the mapping to the Private Network (networkId)

```json
"policy": {
		"rules": [		
		{
		    "action": "Tunnel",
		    "appAuthorizationTokenContext": {
		        "audienceScope": "api://478434b9-dd76-4d77-b32b-cc9348df02da/user_impersonation",
		        "clientAppId": "760282b4-0cfc-4952-b467-c8e0298fee16",
		        "clientRedirectUri": "https://login.microsoftonline.com/common/oauth2/nativeclient"
		    },
		    "channelId": "f3ceb6cc-1706-4817-a2d7-2a8ff07f474c",
		    "hardening": "Block",
		    "id": "356f1ba6-9083-4b33-8547-ff38963b18d3",
		    "matchingCriteria": {
		        "address": {
		            "fqdns": [],
		            "ips": [
		                {
		                    "end": 3232235775,
		                    "start": 3232235520
		                }
		            ]
		        },
		        "ports": [
		            {
		                "end": 22,
		                "start": 22
		            },
		            {
		                "end": 3389,
		                "start": 3389
		            },
		            {
		                "end": 5985,
		                "start": 5985
		            },
		            {
		                "end": 5986,
		                "start": 5986
		            }
		        ],
		        "protocol": "All"
		    },
		    "networkId": "702408bb-d401-4d96-81ff-0547c0dd7d7c",
		    "order": 69.0,
		    "tunnelingPrerequisite": false
		},
	],
}
```

### How do Private Networks and Private Access Apps relate to each other?

Each private network has a list of apps that are directly accessed when the client successfully detects the private network. 

| ![Picture 8: Private Network Structure](/post/2025/Intelligent-Local-Access-Deep-Dive/images/ILA-PrivateNetworks.png) |
|:--:|
| *Picture 8: Private Network Structure* |

## How to use Intelligent Local Access

### Should I use Intelligent Local Access for all my Private Apps?

No - please don’t do this! In my opinion ILA is a feature you only want to use if you have a good reason to do so - like performance or latency.

The reason for this is quite simple: Using ILA requires an open Firewall for this connection between the client and the resource. This means everyone who manages to turn off the Private Access channel or the whole  GSA client, is able to access your resources without Entra Authentication and Conditional Access. Same for clients without a GSA client.

My idea of Zero Trust Network Access (ZTNA) is that a connection is only possible after comprehensive verification. This is not compatible with open firewalls between clients and servers.

> 💡Please use ILA granular for services that generate a lot of traffic and are not particularly security-critical. Printers and print servers are a good example.

### How many Private Networks do I want/need?

When I saw that I could create multiple private networks, I immediately started thinking about how I could use this for the most efficient configuration possible. After looking at a few environments I am familiar with, I came to the conclusion that having this flexibility is generally a good thing, but it also comes with a few hurdles and additional complexity.

The simplest scenario is to create just one private network for all locations:

| ![Picture 9: Single Private Network](/post/2025/Intelligent-Local-Access-Deep-Dive/images/ILA-SinglePrivateNetwork.png) |
|:--:|
| *Picture 9: Single Private Network* |

In this case, it makes sense to configure multiple DNS servers:

| ![Picture 10: Single Private Network with multiple DNS](/post/2025/Intelligent-Local-Access-Deep-Dive/images/ILA-SinglePrivateNetworkMultiDNS.png) |
|:--:|
| *Picture 10: Single Private Network with multiple DNS* |

Alternatively, you can also set up a separate private network for each location. However, it makes sense to have a separate DNS server at each location for this purpose.

| ![Picture 11: Multiple Private Networks](/post/2025/Intelligent-Local-Access-Deep-Dive/images/ILA-MultiPrivateNetworks.png) |
|:--:|
| *Picture 11: Multiple Private Networks* |

> 💡 If the connection from the clients to the DNS servers at the other locations is possible, it can quickly happen that these connections also go directly, possibly unintentionally. It may therefore be necessary to block certain DNS connections…

I consider the “one private network for all” scenario to be the most reasonable for most environments, but I am curious to see what will be decided in the projects over the next few months.

### Should I set special settings for client hardening?

By default, any user can disable the GSA Client even without administrator rights. This is often useful in the early stages of proof of concept and rollout, as instability can sometimes occur and users can use this as a workaround. 

After the rollout and with the introduction of ILA, it makes sense to restrict user options and implement hardening measures.  We have the following options for this:

| RegKey | explanation |
| --- | --- |
| [HideDisableButton](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-install-windows-client#hide-or-unhide-system-tray-menu-buttons) | Hides the option to deactivate the client. |
| [HideDisablePrivateAccessButton](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-install-windows-client#hide-or-unhide-system-tray-menu-buttons) | Hide the option to disable Private Access only |
| [RestrictNonPrivilegedUsers](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-install-windows-client#restrict-nonprivileged-users)  | Requires admin rights with UAC to enable/disable the client. |

Based on the ILA configuration, I think it makes sense to either hide both buttons or set [RestrictNonPrivilegedUsers](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-install-windows-client#restrict-nonprivileged-users) (and hide the Disable Private Access button) to prevent Entra authentication and conditional access from being bypassed.

| ![Picture 12: Client Hardening](/post/2025/Intelligent-Local-Access-Deep-Dive/images/ClientHardening.png) |
|:--:|
| *Picture 12: Client Hardening* |

## What about logging and reporting?

### Client-side logging

We can see the traffic in the Advanced Diagnostic tool (look for Action: local) and there are events in the `Microsoft-Global Secure Access Client/Operational` log:

| ![Picture 13: ILA logging](/post/2025/Intelligent-Local-Access-Deep-Dive/images/ILA-logging.png) |
|:--:|
| *Picture 13: ILA logging* |

> 💡 Unfortunately, in my tests, I was unable to find any interesting events related to ILA (and the GSA client in general) in the MDE tables of Advanced Hunting.

### Logging and Reporting in Entra and Sentinel/XDR

At the time of the writing we don’t have any centralized logging or reporting, but I’m looking forward to it and will update this section when it is available.

## Summary

My first impression is really very positive. The feature is easy to configure, works quickly and reliably, and solves a real problem in many environments. I think it will make many rollouts much easier and solve bandwidth shortages.

Nevertheless, I would like to point out that the use of ILA weakens security posture and undermines the zero trust architecture. It should only be used if there is a good reason for doing so and the targets do not have high security requirements. The most secure option is to route all traffic through GSA and not allow any direct connections from the client.

## **Attribution and References**

https://learn.microsoft.com/en-us/entra/global-secure-access/enable-intelligent-local-access

>🙏 Big thanks to [Peter Lenzke](https://www.linkedin.com/in/peter-lenzke-bb95813a/) for proofreading this blog.