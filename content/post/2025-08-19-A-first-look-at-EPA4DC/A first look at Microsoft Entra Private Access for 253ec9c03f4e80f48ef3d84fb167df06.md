---
layout:     post 
title:      "A first look at Microsoft Entra Private Access for Active Directory domain controllers"
subtitle:   "This blog post is about the new Private Access Sensor for Domain Controller and the option to restrict Kerberos SSO to clients using Entra Private Access"
date:       2025-08-19
author:     "Chris Brumm"
URL:        "/2025/08/19/Deep-Dive-SSO-in-Entra-Private-Access/"
tags:
    - Entra
    - Global Secure Access
    - Entra Private Access
categories: [ Global Secure Access ]
---

In many environments - often for historical reasons - there is no strict separation of client and server networks. And if there is a firewall between the networks, the rule sets often allow direct communication with the domain controllers in the environment. Although a conversion makes a lot of sense, it is often not possible quickly, because various services like GPOs or Kerberos rely on this communication and a client modernization project takes time and effort.

This means for all clients in the local network:

- Any client can attack Active Directory, including unmanaged or even known malicious ones.
- Any user can log on to all services without strong authentication and without managed clients.

Accordingly, in recent years I have often heard justified calls for "OnPrem MFA", for example.  

This is where **Microsoft Entra Private Access for Active Directory domain controllers** (From here on, I will refer to it as **EPA4DC**.) comes in and addresses a very important part of this requirement with the Kerberos service by installing the **Private Access Sensor** on the domain controllers. This sensor is (as of August 2025 in the preview) able to control the retrieval of Kerberos tickets based on the combination of the following criteria

- Which user is requesting?
- Which service principal name is requested?
- From which IP address is he requesting?

| ![Picture 1: Overview from Microsoft](/post/2025-08-19-A-first-look-at-EPA4DC/images/Overview-from-Microsoft.png) |
|:--:|
| *Picture 1: Overview from Microsoft* |

## What does this look like for the user?

Let's start with how this ideally affects the user. In the following video you can see how the user Bert tries to access a web server in the local network from his client. As the web server requires a Kerberos login, EPA4DC is used and we can see that it is only possible to retrieve a ticket (TGS) if the GSA client is switched on.

| ![Picture 2: Demo](/post/2025-08-19-A-first-look-at-EPA4DC/images/EPA4DCs-new.gif) |
|:--:|
| *Picture 2: Demo* |

For this to be possible, all domain controllers involved must be published via Entra Private Access (from now referred as EPA), the Private Access Sensor must be installed on them and the Service Principal Names (SPNs) involved must be stored in the Quick Access App:

| ![Picture 3: My test environment](/post/2025-08-19-A-first-look-at-EPA4DC/images/Test-Environment.png) |
|:--:|
| *Picture 3: My test environment* |

## Installation

Let's go through the necessary steps **for a test setup** (instructions for production environments are given below). I am assuming that you already have EPA running.

### Step 1: Installing the Private Access Sensor on the DCs

Overall quite simple and well documented. Don't forget to set Edge as default browser since an authentication at Entra ID is necessary to register the sensor.

>‼️ Unfortunately, this requires a login with an admin account. It is not yet documented which role is required as a minimum. In many environments it will unfortunately come down to a login from the DC without phishing-resistant MFA and device compliance. In my opinion, there is an urgent need for action for the installation of the agents and an alignment with e.g. the MDI procedure would be desirable. 

### Step 2: Configuration of an SPN

SPNs can only be configured in the Quick Access app.

| ![Picture 4: SPNs in the Quick Access app](/post/2025-08-19-A-first-look-at-EPA4DC/images/SPNinQuickAccess.png) |
|:--:|
| *Picture 4: SPNs in the Quick Access app* |

We can then also view the result in the `cloudpolicy.json` file at the sensor installation path (on my DC it is: `C:\Program Files\Private Access Sensor`) on the domain controller: 

| ![Picture 5: cloudpolicy.json](/post/2025-08-19-A-first-look-at-EPA4DC/images/cloudpolicy.png) |
|:--:|
| *Picture 5: cloudpolicy.json* |

### Step 3: Allowlisting the Entra Network Connector

Right next to the `cloudpolicy.json` is the file `localpolicy.json` in which we can store the IP addresses of the Entra Network Connectors in the `SourceIpAllowList`. Since every system in this list is excluded from the the requirement to use GSA to get a Kerberos ticket you can (and should) include other systems such as servers or terminal services here.

| ![Picture 6: localpolicy.json](/post/2025-08-19-A-first-look-at-EPA4DC/images/localpolicy.png) |
|:--:|
| *Picture 6: localpolicy.json* |

### Step 4: Test

You can get a first impression of whether the Private Access Sensor is running correctly and the configuration is as intended in the event log.

It doesn't include the username (SamAccountName, UPN, ObjectID,...) and it doesn't include the requesting client (device, IP) - so it doesn't help me to prepare the enforcement...

| ![Picture 7: Block events](/post/2025-08-19-A-first-look-at-EPA4DC/images/Block-Event.png) |
|:--:|
| *Picture 7: Block events* |

But if you have Microsoft Defender for Identity (MDI) running on the DCs (which I strongly recommend) you can use this query, for example, to get a simple overview of how big the impact of an activation would be:

```sql
let EntraNetworkConnectors = dynamic(['192.168.0.40','192.168.0.41']);
let SPNsInScope = dynamic(['http/blog.gkfelucia.net','http/gkfeluciaapp1.gkfelucia.net']);
IdentityLogonEvents
//| where Application == @"Active Directory"
| where Protocol == @"Kerberos"
| where LogonType == @"Resource access" 
| extend ThroughGSA = iff(IPAddress in(EntraNetworkConnectors), true, false)
| extend SPN = split(AdditionalFields.Spns,',')
| mv-expand SPN
| extend SPN = tolower(tostring(SPN))
| where SPN in (SPNsInScope)
| summarize count() by SPN, AccountUpn, ThroughGSA, DeviceName, DestinationDeviceName, ActionType //, IPAddress, DestinationIPAddress
| sort by count_
``` 

In this example you can clearly see that I first have to take care of the user Waldorf before I disable the audit mode (Bert had his appearance in the demo above):

| ![Picture 8: sample output](/post/2025-08-19-A-first-look-at-EPA4DC/images/Sample-Output.png) |
|:--:|
| *Picture 8: sample output* |

### Step 5: Disabling AuditMode

The agent starts in audit mode and can be switched under HKLM:\SOFTWARE\Microsoft\PrivateAccessSensor. 

| ![Picture 9: RegKeys](/post/2025-08-19-A-first-look-at-EPA4DC/images/RegKeys.png) |
|:--:|
| *Picture 9: RegKeys* |

## How to do a proof of concept

If you want to do a proof of concept in a production environment outside of the test environment, it unfortunately becomes apparent that the maturity level of the solution is not yet really high.

At the moment, I think the most promising approach is to start with a small test group, gradually expand the protected SPNs, review the logs from the MDI table IdentityLogonEvents (alternatively, the event log in e.g. Sentinel/Monitor is also possible) and test intensively, as we have no logging to show us the blocked requests.

In addition, scoping for individual users currently cannot be configured centrally in the Entra portal; instead, the `localpolicy.json` config file must be used locally on the DCs. This has priority over the `cloudpolicy json` and you can simply copy the structure for the SPNs from there - here is an example:

```json
{
    "ProtectedSpns": [
        {
            "Spn": "HTTP/blog.gkfelucia.net",
            "Exclusions": {
                "Ips": ["192.168.0.40"],
                "Users": []
            },
            "Inclusions": {
                "Users": ["Bert"]
            }
        }
    ],
    "SourceIpAllowList": ["192.168.0.40"]
}
```

Here are some important observations from me that may save you a little time:

- Usernames are case-sensitive (why?) and f[irst part of the OnPrem Userprincipalname](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-configure-domain-controllers#:~:text=On%2Dpremises%20username%2C%20which%20is%20the%20first%20part%20of%20a%20User%20Principal%20Name%20(UPN)%20such%20as%20username%40domain) (why?)
- If something else (like a User include) is configured for an SPN the `SourceIpAllowList`is ignored and you have to add the IPs to the exclusion section.

### Using wildcards

In the `localpolicy.json` it is also possible to use wildcards, but unfortunately this is only limited to the service classes of the SPNs without the possibility to exclude individual SPNs. Here is an example:

```json
{
    "ProtectedSpns": [
        {
            "Spn": "HTTP/*",
            "Exclusions": {
                "Ips": ["192.168.0.40"],
                "Users": []
            },
            "Inclusions": {
                "Users": ["Bert"]
            }
        }
    ],
    "SourceIpAllowList": ["192.168.0.40"]
}
```

## The NTLM problem

Okay now let's talk about the elephant in the room: NTLM 

In the prerequisites, there is this note that disabling NTLM in the environment would make a lot of sense for EPA4DCs to deliver the expected results:

| ![Picture 10: The NTLM prereq](/post/2025-08-19-A-first-look-at-EPA4DC/images/NTLM-Prereq.png) |
|:--:|
| *Picture 10: The NTLM prereq* |

The primary reason for this is that the sensor today only supports Kerberos and it is standard in Active Directory for systems to fall back to NTLM when Kerberos is unavailable. In my demo environment, I have configured the web server to only accept Kerberos (Picture 11). As soon as I also allow NTLM, I see a password prompt (Picture 12) and can log in. 

| ![Picture 11: Kerberos-only at the IIS](/post/2025-08-19-A-first-look-at-EPA4DC/images/Kerberos-Only-IIS.png) |
|:--:|
| *Picture 11: Kerberos-only at the IIS* |

| ![Picture 12: NTLM password prompt](/post/2025-08-19-A-first-look-at-EPA4DC/images/NTLM-Prompt.png) |
|:--:|
| *Picture 12: NTLM password prompt* |

There are many good reasons to switch off NTLM today and if we take a look at how authentication via NTLM works (Picture 13), it also shows that the solution for securing Kerberos that Entra Private Access uses cannot simply be transferred.

However, since disabling NTLM is no easy task - as Steve Syfuhs explains [here](https://www.youtube.com/watch?v=zlhoAYsSd4c) - we will need a solution. I am excited to see how this turns out! 

| ![Picture 13: NTLM auth flow](/post/2025-08-19-A-first-look-at-EPA4DC/images/NTLM-Auth-Flow.png) |
|:--:|
| *Picture 13: NTLM auth flow*|

## Excursus: EPA4DC and KDC proxy

In addition to the question of how proofs of concept and rollouts of EPA4DC can be implemented, I am also very interested in the question of how EPA4DC and the KDC proxy, which I described in my blog on SSO in GSA, relate to each other.  

If we look at the 3 core advantages of the KDC proxy and add the ability of the EPA agent to filter connections based on the criteria described above, the following picture emerges:

|  | **KDC proxy** | **EPA 4 DC** |
| --- | --- | --- |
| Strong Auth Enforcement | Windows Hello / Cert-based Authentication | Full Conditional Access Support |
| Direct connection to a domain controller | no | yes |
| Timeouts because of Negative DC locator caching | no | yes |
| Blocking of Kerberos TGS Requests for specific combinations of users/SPNs/IPs | no | yes |

So it's worth taking a look at whether and how the two can be combined!

### Requirements for coexistence

For the KDC proxy to work in combination with EPA4DCs, all proxies must also be included in the `SourceIpAllowList`. If the KDC proxies are running on the Entra Network Connectors of the Quick Access Connector Group, this is already the case. My extended test environment looks like this:

| ![Picture 14: My test environment with KDC proxy](/post/2025-08-19-A-first-look-at-EPA4DC/images/Test-Environment-KDC.png) |
|:--:|
| *Picture 14: My test environment with KDC proxy* |

### Hardening:

If we enforce with EPA4Dcs that Kerberos traffic must go through EPA, it also makes sense to ensure that the KDC proxy is only accessible via GSA so that it is not possible to subvert EPA4DCs.

When I set up a KDC proxy, I always use this [script](https://cloudbrothers.info/windows-business-cloud-trust-kdc-proxy/#kdc-proxy) from Fabian. The last step in it is to create an inbound firewall rule and with a small adjustment there you can easily restrict access to the AppProxies:

```powershell
...
**# Create an incoming firewall rule
$AppProxies =   @("192.168.0.40", "192.168.0.41")
New-NetFirewallRule -DisplayName "Allow KDC proxy TCP/443 for AppProxies" -Direction Inbound -Protocol TCP -LocalPort 443 -RemoteAddress $AppProxies** 

# Set the KDC proxy service to automatic
Set-Service -StartupType Automatic -Name kpssvc
# Start the KDC proxy 
Start-Service kpssvc

```

## Summary (so far)

I have been recommending to my customers for years that they remove their clients from both AD and internal networks, and over the past year I have helped several companies test and implement Entra Private Access. But of course, I have also seen that for many, this is still a long way off. I see Microsoft Entra Private Access for Active Directory domain controllers as a great innovation that has the potential to significantly improve the security of many on-premises environments. 

But I think this blog has also shown that the feature is still quite new and needs to grow a little more. I would like to see the following aspects improved in the short term:  

- A centralized configuration incl. support of groups for the users and the SPNs, use of connector groups in the configuration.
- The logs contain user, SPN, IP/client and action and can be viewed in the portal and queried via KQL.

I will update this blog when new features are added!

## **Attribution and References**

- Early video: [https://www.youtube.com/watch?v=_p4bzmPl7MY](https://www.youtube.com/watch?v=_p4bzmPl7MY)
- Early blog: [https://techcommunity.microsoft.com/blog/microsoft-entra-blog/microsoft-entra-private-access-for-on-prem-users/3905450](https://techcommunity.microsoft.com/blog/microsoft-entra-blog/microsoft-entra-private-access-for-on-prem-users/3905450)
- Documentation: [https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-configure-domain-controllers](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-configure-domain-controllers)
- SPNs: [https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names)
- NTLM: [https://learn.microsoft.com/en-us/windows/win32/secauthn/microsoft-ntlm](https://learn.microsoft.com/en-us/windows/win32/secauthn/microsoft-ntlm)
- more NTLM: [https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-apds/5bfd942e-7da5-494d-a640-f269a0e3cc5d](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-apds/5bfd942e-7da5-494d-a640-f269a0e3cc5d)
- Steve Syfuhs on KDC proxy: [https://syfuhs.net/windows-hello-cloud-trust](https://syfuhs.net/windows-hello-cloud-trust)
- Fabian Bader on KDC proxy: [https://cloudbrothers.info/windows-business-cloud-trust-kdc-proxy/](https://cloudbrothers.info/windows-business-cloud-trust-kdc-proxy/)

>🙏 Big thanks to [Fabian Bader](https://www.linkedin.com/in/fabianbader/) for proofreading this blog.