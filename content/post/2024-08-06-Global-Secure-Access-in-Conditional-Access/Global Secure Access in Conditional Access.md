---
layout:     post 
title:      "Global Secure Access in Conditional Access"
subtitle:   "The blog discusses the integration of Global Secure Access in Conditional Access."
date:       2024-08-06
author:     "Chris Brumm"
URL:        "/2024/08/06/Global-Secure-Access-in-Conditional-Access/"
#image:      "/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/conditional-access-central-policy-engine-zero-trust.png"
tags:
    - Entra
    - Global Secure Access
    - Conditional Access
categories: [ Global Secure Access ]
---

A few days ago, Microsoft announced that Global Secure Access is now generally available. Since I have been working with the product for some time now and more and more proof of concepts are being launched, it is high time for me to do a blog series about it.

Here is an overview of the parts (planned so far):

1. [Overview to Global Secure Access](https://chris-brumm.com/2024/07/30/Overview-to-Global-Secure-Access/)
2. [Global Secure Access in Conditional Access](https://chris-brumm.com/2024/08/06/Global-Secure-Access-in-Conditional-Access/)
3. [Deep Dive DNS in Entra Private Access](https://chris-brumm.com/2024/09/07/Deep-Dive-DNS-in-Entra-Private-Access/)
4. [Deep Dive SSO in Entra Private Access](https://chris-brumm.com/2024/09/14/Deep-Dive-SSO-in-Entra-Private-Access/) 
5. [Entra Private Access and the future of the Entra App Proxy](https://chris-brumm.com/2025/04/06/Entra-Private-Access-and-the-future-of-the-Entra-App-Proxy/)

In the overview to Global Secure Access, I particularly emphasized the good integration in Conditional Access for both Microsoft Entra Internet Access and Microsoft Entra Private Access. Accordingly, my first detailed blog in the Global Secure Access series also deals with this.  

In this blog, we will first take a look at the Conditional Access controls introduced with Global Secure Access and the Global Secure Access-related resources.

After that, I will discuss a few use cases that result from Global Secure Access.

# Configuration elements in Conditional Access

With Global Secure Access, Conditional Access has been enriched by several configuration elements, some of which can be integrated into the existing policies and some of which are necessary for new special policies.

## Compliant Network Locations

Conditional Access can now use GSA as a Compliant Network Location Condition in policies. 

On this occasion, the network-relevant conditions were also moved from the Conditions section to the first level of the policy.

![Untitled](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled.png)

### Protection provided by the Compliant Network Control

Even if access to Entra ID integrated apps can then be linked to the use of GSA, Conditional Access cannot, of course, restrict access at network level. Instead, it only controls the issuing of OAuth2/SAML tokens. We are therefore not automatically protected against token replay by GSA, but are still dependent on features such as Continuous Access Evaluation and Token Protection. 

>💡 The restriction to Compliant Network Control acts as a [User Condition Change](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation#user-condition-change-flow) for CAE-capable applications and therefore forces instant re-authentication - so the combination of both features provides very effective protection against token replay!



### Incompatible Conditional Access control configuration elements

At the moment, the following conditions cannot be selected when GSA is selected as the target:

- [Compliant network check is currently not supported for Private Access applications](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-compliant-network#:~:text=Compliant%20network%20check%20is%20currently%20not%20supported%20for%20Private%20Access%20applications.).

At the moment, the following controls cannot be selected when Compliant Network is selected as the condition:

- Require app protection policy
- Require approved client app
- Use Global Secure Access security profile

## Global Secure Access as target resource type

In the Target Resources, it is now possible to switch from Cloud Apps to Global Secure Access and then select one or more desired traffic profiles in order to define conditions - such as a compliant device - for access.

![Untitled](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%201.png)

![Untitled](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%202.png)

The selection of Global Secure Access as a target is also necessary in order to use the security profiles described below in the session controls.

>💡 Policies with Global Secure Access as a target are of course only effective if we can enforce them through the client. If the user is able to deactivate the client (which requires admin rights), the policies have no effect.



### Incompatible Conditional Access configuration elements

At the moment, the following conditions cannot be selected when GSA is selected as the target:

- Locations
- Client Apps

At the moment, the following controls cannot be selected when GSA is selected as the target:

- Require approved client app
- Require app protection policy
- Terms of Use
- Use app enforced restrictions (expected as only usable for O365)
- Use Conditional Access App Control
- Customize continuous access evaluation
- Disable resilience defaults

## Global Secure Access security profiles as a Session Control

Security profiles are used in Global Secure Access to group policies together and define priorities in the processing of these policies. Using them as session controls provides a very high degree of flexibility when assigning user groups in particular situations.

Only one profile can be selected per Conditional Access Policy, but as it is possible for several CA policies with security profiles to affect the session, each profile also has a priority that decides in the event of a conflict. 

![Untitled](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%203.png)

*As the use of Security Profiles requires scoping on Global Secure Access Traffic Profiles, the incompatibilities described there also apply here.*

## Microsoft Entra Private Access Apps

When Microsoft Entra Private Access is set up, the Quick Access app is directly available and further Global Secure Access Applications for defining application segments can be created by the admin after setup and are automatically assigned to the Private Access Traffic Forwarding Profile. 

A Global Secure Access application is a new type of enterprise application that is included in All Cloud apps and can be included or excluded as a target in Conditional Access. 

*I have not yet found any incompatible conditions or controls*

# Global Secure Access Use Cases in Conditional Access

With Global Secure Access, there are various scenarios for conditional access policies and I would like to give you a brief overview of use cases that I think make sense at the moment.

With the extension to the network level, the importance of Conditional Access continues to grow and it makes a lot of sense to have a concept. I would like to recommend the frameworks by [Daniel Chronlund](https://danielchronlund.com/2020/11/26/azure-ad-conditional-access-policy-design-baseline-with-automatic-deployment-support/), [Claus Jespersen](https://github.com/clajes/ConditionalAccessforZeroTrustResources) and [Kenneth van Surksum](https://www.vansurksum.com/2022/12/15/december-2022-update-of-the-conditional-access-demystified-whitepaper-and-workflow-cheat-sheet/).

## Access to Internet resources

With the use of Internet Access, we have the possibility to control the outgoing traffic from the client. The effective policies result from the user logged on to the OS - regardless of which user logs on to resources later.

### Blocking websites and categories

The classic Secure Web Gateway use case is the filtering of URLs and categories.

Microsoft provides us with a flexible configuration tool in which rules with FQDNs or categories are combined into policies and these are then combined into profiles and prioritized if several profiles are applied to a session.

The profiles are attached to Conditional Access Policies as described above and thus allow an extremely flexible configuration, e.g. to allow or block websites only depending on the device or location used.

![Untitled](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%204.png)

>💡 **Baseline Security Profile**: As remote networks also allow devices without the GSA client and without user awareness to run through the solution, Microsoft has created a solution to enforce certain rules for everyone as a baseline. It makes sense to be economical with rules here, as these also apply to all devices with a client and would otherwise have to be overwritten in other profiles.



### Comparison of Global Secure Access and Microsoft Defender for Endpoint

With Web Content Filtering, Microsoft for Endpoint also has a feature for blocking categories and can block URLs as custom indicators (which can also be managed by MDA), which raises the question of the relationship between the two functions.

Both products (like also the Azure Firewall) use the same engine for categorization and I think Microsoft will keep both functions, as it is not to be expected that both functions will always be used.

>💡 Because the same lists are used, [the lookup function of the Azure Firewall](https://learn.microsoft.com/en-us/azure/firewall/premium-features#web-category-search) can also be used to find out which category a URL belongs to.



On closer inspection, however, there is a significant difference: while MDE enforces filtering completely independently of the logged-in user at device level, GSA works at the level of the logged-in user.

*In my opinion, GSA has a clear advantage when it comes to modeling complex rule sets!*

### Forcing special controls when accessing traffic profiles

In addition to controlling which pages and categories should generally be accessible as described above, we can also define conditions for certain or all accesses to an (Internet access) traffic profile.

In this example, a compliant device is required to access M365 and Internet Access:

![Untitled](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%205.png)

![Untitled](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%206.png)

## Access to Entra Private Access resources

As described above, we can use Private Access to group resources - i.e. the combination of IP/IP range/FQDN, protocol and port - into app segments and include or exclude them as targets in Conditional Access Policies, thereby setting different conditions for access to different app segments.

![Untitled](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%207.png)

Like all other enterprise apps, these apps can also be selected in several Conditional Access policies and, because they are included in All Cloud Apps, this is very likely to be the case in every environment. This fact should also ensure good basic protection - consisting of Compliant Device and MFA - with a solid rule set.

>💡 This diagram also clearly shows the relationship between the GSA Apps, the App Segments and the Connector Groups. Both the app (assignment and CA) and the Connector Group (routing) are decided on the basis of an addressed App Segment (IP+Port).



### Use of custom security attributes

Instead of direct selection in Conditional Access Policies, I also think it makes a lot of sense to use Custom Security Attributes for this, as this reduces the likelihood of misconfigurations and makes it easier to implement a separation of duties in administration.

![Untitled](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%208.png)

In the example configuration shown here, I have created a new attribute set "GlobalSecureAccess" and created the two attributes "Criticality" and "Datacenter" with predefined values and the following rules: 

- The conditional access rule set is controlled via the "Criticality" attribute, which should be maintained for all apps.
- To avoid a gap, the behavior for an attribute that is not set and the default value is the same.
- For the other values, additional tightenings are added to ensure (for Critical) Phishing Resistant Credentials or (for PAW-only) restrict the access to Admin Workstations.

![Untitled](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%209.png)

![Untitled](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%2010.png)

This procedure means that Global Secure Access and Conditional Access can be maintained by different people / teams with different rights and the creation of further apps or adaptation of criticality does not require any adaptation of the CA policies.

Unfortunately, relatively high rights are still required at the moment, e.g. to enable an employee from the network team to act, and it is to be hoped that MS will enable more granularity here in the future. These are the rights required to create a new GSA app and assign a security attribute to it:

- Global Secure Access Administrator
- Cloud Application Administrator
- Attribute Assignment Administrator

>💡 In the model of division of work described above, I recommend assigning the role of Cloud Application Administrator not at tenant level but only for the respective apps, since this role is really powerful and should be avoided (at the tenant level) .

The procedure for the assignment is described [here](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/assign-roles-different-scopes#assign-roles-scoped-to-an-app-registration).

![Untitled](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%2011.png)



## Access to Microsoft 365 resources

When accessing M365, the CA Compliant Networks feature described above already offers some interesting new capabilities that help us prepare our environments for a changing threat situation.

Although we have seen a further increase in password-based attacks in the [Microsoft Digital Defense Report](https://www.microsoft.com/en-us/security/security-insider/microsoft-digital-defense-report-2023) for 2023, we can already see that we are also increasingly having to deal with more complex attacks.   

>💡 Not directly Conditional Access features but further advantages of using Global Secure Access for M365 are the features Tenant Restrictions, IP restoration and M365 log enrichment which I described in the first blog of this series.



### Defense against AiTM attacks

The last few months have been characterized by the increasing prevalence of AiTM attacks (e.g. with Evilginx2) and I am very curious to see the next figures on this topic, because my personal experience has often shown me how effective this attack is. 

The existing defense mechanisms of phishing-resistant authentication methods (caution: Evilginx can now automatically downgrade) and device compliance are now joined by the Compliant Networks Condition.

![[https://www.microsoft.com/en-us/security/security-insider/microsoft-digital-defense-report-2023](https://www.microsoft.com/en-us/security/security-insider/microsoft-digital-defense-report-2023)](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%2012.png)

[https://www.microsoft.com/en-us/security/security-insider/microsoft-digital-defense-report-2023](https://www.microsoft.com/en-us/security/security-insider/microsoft-digital-defense-report-2023)

I recommend [this blog post](https://derkvanderwoude.medium.com/microsoft-entra-internet-access-to-prevent-aitm-attack-s-31171db43a83) in which Derk van der Woude describes the scenario and configuration in detail. (described here by Derk)

### Strengthening of Continuous Access Evaluation

The token replay attack has also seen considerable growth (doubling in 2023). Due to the fact that access/bearer tokens are stolen here, it cannot be stopped with conditional access alone and, in my view, represents the next evolutionary stage for identity attacks, as it can also be successful if (phishing-resistant) MFA and device compliance are effective.

One of the most effective countermeasures against this attack at the moment is Continuous Access Evaluation and I would like to recommend this sensational [blog](https://cloudbrothers.info/continuous-access-evaluation) by Fabian, in which he not only explains how CAE works, but also describes the various scenarios in which it works.

Even if you have probably all read the blog, I would like to briefly summarize how the location-based scenarios work: Whenever there is a change to or from a named location, CAE-enabled applications reject the access token and require reauthentication even if the token is still valid

This is a very powerful feature but - especially in modern environments - it has a crucial weakness: We don't want to route traffic from clients to cloud services through the data center and we don't want to use IP-based conditional access policies.

![[https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation#user-condition-change-flow](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation#user-condition-change-flow)](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%2013.png)

[https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation#user-condition-change-flow](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation#user-condition-change-flow)

And this is precisely where the strength of Global Secure Access lies. A Compliant Network behaves like a Named Location (currently only in SharePoint Online) and triggers the Continuous Access Evaluation trigger.

### Combination with the remote network feature

To integrate systems that do not support device compliance, the Trusted Network feature can be used for Microsoft 365 services (and I hope in the future for all cloud apps) to replace Trusted Locations.

![Untitled](/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/Untitled%2014.png)

## Access to Entra ID integrated apps

It is now possible to include the Compliant Network Condition in the Conditional Access rule set for Entra integrated apps, but at the moment it makes little sense as the devices must be managed (and CAE is currently only supported for Exchange and Sharepoint). We can therefore also use Device Compliance directly in the rule set.

>💡 But this is just a snapshot and can of course change quickly with additional features.



# Summary

As I hope to show in this blog, Global Secure Access has really good conditional access integration - much more than the SAML integrations I've seen in other vendors' products. 

It also became very clear to me during the preparation of this blog that the importance of conditional access will continue to grow significantly. It is vitally important

- to have a clear strategy for the policy design → see my recommendations above
- to implement release management and automation → the gold standard so far is [the work of Thomas Naunheim on this topic](https://www.cloud-architekt.net/aadops-conditional-access/)

*In any case, I am convinced that this is just the beginning and that we will see various other use cases and improvements in the combination of Global Secure Access and Conditional Access.*

*My next blog in this series will be a deep dive regarding DNS in Entra Private Access!*

## **Attribution and References**

- [Manage access to custom security attributes in Microsoft Entra ID
](https://learn.microsoft.com/en-us/entra/fundamentals/custom-security-attributes-manage?tabs=admin-center)
- [How to configure per-app access using Global Secure Access applications
](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-configure-per-app-access)
- [AADOps: Operationalization of Azure AD Conditional Access](https://www.cloud-architekt.net/aadops-conditional-access/)
- [User condition change flow
](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation#user-condition-change-flow)
- [Entra ID Conditional Access Policy Design Baseline with Automatic Deployment Support](https://danielchronlund.com/2020/11/26/azure-ad-conditional-access-policy-design-baseline-with-automatic-deployment-support/)
- [Claus Jesperson: ConditionalAccessforZeroTrustResources
](https://github.com/clajes/ConditionalAccessforZeroTrustResources)
- [Kenneth van Surksum: December 2022 update of the conditional access demystified whitepaper and workflow cheat sheet](https://www.vansurksum.com/2022/12/15/december-2022-update-of-the-conditional-access-demystified-whitepaper-and-workflow-cheat-sheet/)
- [How to use Microsoft Entra | Internet Access to prevent AiTM attack(s)](https://derkvanderwoude.medium.com/microsoft-entra-internet-access-to-prevent-aitm-attack-s-31171db43a83)
- [Fabian Bader: Continuous access evaluation](https://cloudbrothers.info/continuous-access-evaluation)
 
>🙏 A big thank you to [Peter Lenzke](https://www.linkedin.com/in/peter-lenzke-bb95813a/) for proofreading this blog.
