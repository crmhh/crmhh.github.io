---
layout:     post 
title:      "Microsoft Entra MFA Fraud Deep Dive"
subtitle:   "Microsoft's new feature 'Report suspicious activity' enhances Entra ID Protection by integrating user-reported fraudulent attempts, providing automated responses and generating high-risk alerts, improving the detection and response to MFA fraud."
date:       2023-10-07
author:     "Chris Brumm"
URL:        "/2023/10/07/Microsoft-Entra-MFA-Fraud-Deep-Dive/"
#image:      "/post/2024-08-06-Global-Secure-Access-in-Conditional-Access/images/conditional-access-central-policy-engine-zero-trust.png"
tags:
    - Entra
    - ITDR
    - MFA
categories: [ ]
---

# Microsoft Entra MFA Fraud Deep Dive

Tags: Entra, ITDR, MFA
Published at: October 7, 2023
Summary: 

Recently, Microsoft released the new feature [**Report suspicious activity**](https://learn.microsoft.com/en-us/azure/active-directory/authentication/howto-mfa-mfasettings#report-suspicious-activity) for Entra ID. Since I see this feature as a significant improvement and have faced some challenges with the old feature in the past, I have decided to delve deeper into the topic and share my findings here.

Now, you might be wondering what makes this feature special. After all, Microsoft has long provided the ability to reject authentication through the Authenticator app and phone call and report fraudulent attempts in Entra ID. 

The difference is that the new feature integrates the valuable signal of a user reporting a fraudulent attempt into Entra ID Protection and the Detection and Response environment, which can trigger automated responses. Microsoft is currently driving the integration of [identity and access management (IAM)](https://www.microsoft.com/en-us/security/business/solutions/identity-access) features and [extended detection and response (XDR)](https://www.microsoft.com/en-us/security/business/solutions/extended-detection-response-xdr) features under the term [identity threat detection and response (ITDR)](https://www.microsoft.com/en-us/security/business/solutions/identity-threat-detection-response).

# A brief comparison of the old and new solutions

Since we are dealing with a compromised password in the case of a reported fraudulent attempt, I have always recommended the following over the years:

- [ ]  Enable the feature.
- [ ]  Enable automatic blocking of users for further MFA attempts.
- [ ]  Provide an email address of your SOC or identity operations team for notifications.

![/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled.png](/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled.png)

However, this feature does have its weaknesses, which is why I am glad that Microsoft has recently made improvements.

The following issues have always bothered me:

- When a user is blocked, there is no notification or adjustment of the error message. It simply appears as if MFA failed.
- The blocking only applies to further second-factor MFA challenges with the Authenticator app or phone call. The user itself is not blocked, and logins without MFA or with Windows Hello or FIDO are still possible.
- Unlocking can only be done by admins with the Global Admin or Authentication Policy Admin role. No self-service option for the user.
- The email notification to the IT was not integrated with other systems like Microsoft 365 Defender or Microsoft Sentinel to directly generate alerts/incidents. However, it could be built in Sentinel through the audit log.
- It was also possible to report a fraudulent attempt during a self-service password reset. This resulted in someone else being able to trigger the MFA challenge without authentication, putting a well-trained user in a situation where they locked themselves out.

The new feature "Report suspicious activity" addresses these weaknesses, along with the ability to scope the feature. Now we have an integration into Entra ID Protection, which generates a high-risk alert for the user when a fraud is reported, giving this signal much more influence. If an Entra ID P2 license is available and the appropriate configurations are enabled, the following will occur:

- The user receives an error message clearly stating that they have been blocked.
- The user is effectively blocked, Refresh Tokens will be revoked and existing sessions are also terminated by Continuous Access Evaluation (see below).
- Self-remediation is possible for the user via self-service password reset.
- We have multiple options for generating alerts in Entra ID Protection, which can also be integrated into M365 Defender and MS Sentinel (and should be).
- If only the new feature is used, fraud cannot be triggered during a self-service password reset.

![/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%201.png](/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%201.png)

This means that the eager reader (who has enabled the policies for User Risk) can now stop reading and simply enable the new feature and disable the old one.

# MFA Fraud and Entra ID Protection

Entra ID Protection is a very powerful feature, and describing it in full detail would exceed the scope of this blog. Therefore, we will focus on the concrete use case of the impact of a high-risk alert generated by the fraud alert.

First and foremost, the most important effect of the high-risk alert is that the user risk is set to high:

![Risk Detection: User reported suspicious activity](/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%202.png)

Risk Detection: User reported suspicious activity

![Risky User Details after a User reported suspicious activity alert](/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%203.png)

Risky User Details after a User reported suspicious activity alert

By the way, the suspicious activity reported by the user is not detected as a risky login, but as an offline detection in Entra ID Protection. We have to establish a connection with a sign-in (if there is one) ourselves (see below). There may also be an "Unfamiliar sign-in properties" alert generated in this context with the failure reason "Authentication failed during strong authentication request," but these are two completely separate detections.

# Protection: Risk-based Access Control

For User Risk to have an impact on the issuance of new tokens in Entra ID, two things are needed:

- An Entra ID P2 license (this is also necessary to generate the alarm).
- A User Risk policy or a Conditional Access policy.

The Conditional Access policy to enforce a secure password change for a high-risk user may look like this:

![/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%204.png](/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%204.png)

However, in heterogeneous environments, the impact of a high user risk status for a user depends heavily on the architecture and configuration of the environment. In short: The more modern your access management is and the better it is connected to Entra ID, the greater the impact!

Here are a few examples:

|  | Impact with High Risk | No Impact with High Risk |
| --- | --- | --- |
| App Sign-in | App connected to EID via OAuth2 or SAML | App connected to AD via Kerberos |
| Publishing Web Apps | AppProxy with pre-authentication against EID | 3rd Party Reverse Proxy with pre-authentication against AD |
| Access to On-Premises Resources | Global Secure Access - Entra Private Access | VPN solution with hardware token |

## Session Disruption:

In addition, there is the possibility to disrupt sessions before the expiration of the access token lifetime using Continuous Access Evaluation, as reporting a fraud attempt via Authenticator is one of the triggers for the critical event [High user risk detected by Azure AD Identity Protection](https://learn.microsoft.com/en-us/azure/active-directory/conditional-access/concept-continuous-access-evaluation#:~:text=High).

Currently, this only applies to Office 365 services, but the announcement that Microsoft's SSE solution will support Global Secure Access for private and public access and Microsoft's efforts to establish the Continuous Access Evaluation Protocol as a general standard will greatly enhance this feature.

[Fabian Bader](https://www.linkedin.com/in/fabianbader/) has written a wonderful deep dive blog on CAE: [https://cloudbrothers.info/en/continuous-access-evaluation/](https://cloudbrothers.info/en/continuous-access-evaluation/)

## Self-Remediation

The password reset enforced by the policy solves the problem of the compromised password, resets the user risk, and allows the user to regain access.

# Coexistence

According to the [documentation](https://www.notion.so/Review-Azure-RBAC-5371927920d2418fa792c3aed6bfd19f?pvs=21), the two features can be operated in parallel:

> **Report suspicious activity** and the legacy **Fraud Alert** implementation can operate in parallel. You can keep your tenant-wide **Fraud Alert** functionality in place while you start to use **Report suspicious activity** with a targeted test group.

If **Fraud Alert** is enabled with Automatic Blocking, and **Report suspicious activity** is enabled, the user will be added to the blocklist and set as high-risk and in-scope for any other policies configured. These users will need to be removed from the blocklist and have their risk remediated to enable them to sign in with MFA.
> 

Since the two settings of the old feature are dependent on each other, the following configuration options and resulting effects arise:

| Allow users to submit fraud alerts | Automatically block users ... | Report suspicious activity | Block MFA for 90 days | Set User Risk to high | Fraud Submission at SignIn | Fraud Submission at SSPR |
| --- | --- | --- | --- | --- | --- | --- |
| Enabled | Enabled | Enabled | Yes | Yes | Yes | Yes |
| Enabled | Enabled | Disabled | Yes | No | Yes | Yes |
| Enabled | Disabled | Disabled | No | No | Yes | Yes |
| Enabled | Disabled | Enabled | No | Yes | Yes | Yes |
| Disabled | Disabled | Enabled | No | Yes | Yes | No |
| Disabled | Disabled | Disabled | No | No | No | No |

And the graphical version: 

![Untitled](/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%205.png)

# Detection

In theory, with a configured User Risk policy, there is no need for IT intervention. However, as is often the case, the detection capabilities are especially important when the protection capabilities are weaker. Especially in imperfect environments where, for example, there are access/applications with single-factor authentication, the detection component is very important for the possibility of manual intervention if necessary.

## Built-In Alerting

As mentioned above, an EID P2 license is required to use the feature, as the "User Reported Suspicious activity" alarm is a premium alert.

Entra ID Protection distinguishes between risk detections that occur:

- Within the context of a sign-in (online)
- Independently of a sign-in (offline)

If these detections are not directly mitigated, they will increase the current user risk level, which can take on the values none, low, medium, and high.

Identity Protection has a built-in notification feature that is automatically activated for some roles but can also have other email addresses from ticketing, monitoring, or SIEM systems added.

![/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%206.png](/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%206.png)

### Integration with M365D and Sentinel

Attacks are usually complex and not limited to identity alerts alone. Therefore, we should do everything possible to contextualize the alarms that occur and use tools such as the [Cyber Kill Chain](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html) and the MITRE ATT&CK Framework: our scenario falls under [**Multi-Factor Authentication Request Generation**](https://attack.mitre.org/techniques/T1621/) in [Credential Access](https://attack.mitre.org/tactics/TA0006).

It makes sense to integrate the alerts from Entra ID Protection into M365D and Microsoft Sentinel in order to respond appropriately.

<aside>
💡 For me, regarding the Entra ID Protection alerts, an important realization was:
*While the notifications occur directly in* Entra ID Protection *based on reaching a defined user risk level, detections are passed on to M365D and Sentinel as alerts.*

</aside>

In M365D, you can choose how many alerts from Identity Protection should be integrated. The default (and sensible value in most production environments) is *High-impact alerts only*.

![/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%207.png](/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%207.png)

![/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%208.png](/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%208.png)

For this use case, I would like to point out that in the default configuration, automatic alert creation in M365D unfortunately does not occur because the list of alerts is not based on the risk level but curated by Microsoft: **High Impact ≠ High Severity**

The integration with Sentinel can now be done either directly through the dedicated Identity Protection Connector or through the M365D Connector. For the reasons mentioned above, I agree with Microsoft's recommendation to use the M365D Connector.

[data:image/gif;base64,R0lGODlhAQABAIAAAP///wAAACH5BAEAAAAALAAAAAABAAEAAAICRAEAOw==](data:image/gif;base64,R0lGODlhAQABAIAAAP///wAAACH5BAEAAAAALAAAAAABAAEAAAICRAEAOw==)

It is important to note that even when using the M365D Connector, the Identity Protection Connector must be enabled to retrieve all metadata. However, the Analytic Rule (ANR) should be disabled for alert generation to avoid duplicate alerts.

For those who want to understand the integration of Entra ID Protection and M365D in detail, I recommend reading this excellent blog by Sami Lamppu: [Azure AD Identity Protection Integrations with Microsoft Security Solutions](https://samilamppu.com/2022/11/22/azure-ad-identity-protection-integration-with-microsoft-security-solutions/)

## **Custom Detection**

If you have been reading attentively up to this point, you may have noticed the following: We now have Entra ID Protection alerts in M365D and Sentinel, but we may not be aware of the reported MFA fraud because it is not considered a high-impact detection by Microsoft, and no alert is generated for switching to high risk.

Fortunately, Sentinel is very flexible in creating alarms. Let's take a look at some useful approaches to build our own detections to fill this gap.

### **Sentinel Alerts for High Risk Users**

In addition to the mentioned connectors, it is possible and recommended to integrate the Risk Data tables into Sentinel. The table [**AADRiskyUsers**](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/aadriskyusers) provides us with an overview of which users currently have an active risk level of "high".

```sql
AADRiskyUsers
| where RiskLevel == "high"
| where RiskState in("atRisk", "confirmedCompromised")
```

### **Sentinel Alerts for the Risk Event "User Report suspicious activity"**

In the table [**AADUserRiskEvents**](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/aaduserriskevents), we can directly filter for the event.

```sql
AADUserRiskEvents
| where RiskEventType == "userReportedSuspiciousActivity"
| where RiskState == "atRisk"
```

Please note that my queries here are just a starting point. You can find more information on how to effectively work with tables, for example, here: [https://learnsentinel.blog/2021/08/05/streaming-azure-ad-risk-events-to-azure-sentinel/](https://learnsentinel.blog/2021/08/05/streaming-azure-ad-risk-events-to-azure-sentinel/)

[data:image/gif;base64,R0lGODlhAQABAIAAAP///wAAACH5BAEAAAAALAAAAAABAAEAAAICRAEAOw==](data:image/gif;base64,R0lGODlhAQABAIAAAP///wAAACH5BAEAAAAALAAAAAABAAEAAAICRAEAOw==)

### **Sentinel Alerts for the old and new Fraud Report variants**

Things get more interesting when the old feature is active in addition to the new one, as we only receive events from Entra ID Protection for the new feature. (This can be useful, for example, if only a portion of users have a P2 license).

In this case, the [Audit Log](https://learn.microsoft.com/en-us/azure/active-directory/reports-monitoring/reference-audit-activities#azure-ad-mfa-azure-mfa) can help us. There are events that can be used to generate alerts:

| Audit Category | Activity |
| --- | --- |
| UserManagement | Fraud reported - no action taken |
| UserManagement | Fraud reported - user is blocked for MFA |
| UserManagement | Suspicious activity reported |

The event generated depends on the configuration affecting the user:

| Enabled Setting | Event |
| --- | --- |
| Allow users to submit fraud alerts | Fraud reported - * |
| Automatically block users … | Fraud reported - user is blocked for MFA |
| Report suspicious activity | Suspicious activity reported |

If both features are active, both a "Fraud reported - *" and a "Suspicious activity reported" event will occur.

```sql
// Fraud Detected?
AuditLogs
		| where OperationName has_any ("Fraud reported", "Suspicious activity reported")
		| mv-expand TargetResources
		| evaluate bag_unpack(TargetResources)
		| extend _UserPrincipalName = tolower(tostring(column_ifexists("userPrincipalName", "")))
		| extend UserId = tostring(column_ifexists("id", ""))
		| project-rename FraudTimestamp=TimeGenerated
```

### **In what context was the fraud attempt reported?**

It is also interesting to know if the fraud report was triggered in response to a sign-in or a self-service password reset (SSPR), as we can infer a compromised password in the case of a sign-in, while SSPR can occur completely unauthenticated until the fraud report is made.

| Trigger | Log | Indicator Event |
| --- | --- | --- |
| Sign In | SigninLogs | Authentication requirement: Multifactor authentication
Status: Failure
Sign-in error code: 500121
Failure reason: Authentication failed during strong authentication request. |
| SSPR | AuditLog | Activity Type: Self-service password reset flow activity progress
Category: UserManagement
Status reason: User started the mobile app notification verification option |

Note: This clarification is only necessary if the old feature is still active, as only then can a fraud report occur during SSPR. You can check in your Logs what was the trigger:

```sql
// Fraud Report while SignIn?
SigninLogs
    | where Status.errorCode == 500121
    | extend _UserPrincipalName = tolower(UserPrincipalName)
    | project-rename SignInTimestamp=TimeGenerated, RelatedActorIP=IPAddress

// Fraud Report while SSPR?
AuditLogs
    | where ResultReason == "User started the mobile app notification verification option"
    | extend _UserPrincipalName = tolower(tostring(TargetResources[0].userPrincipalName))
    | extend RelatedActorIP = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
    | project-rename SSPRTimestamp=TimeGenerated
```

The three queries can then be combined to quickly assess whether the trigger was a SignIn or an SSPR - and adjust the severity of the incident if desired.

```sql
// Fraud Detected?
let FraudEvent= AuditLogs
| where OperationName has_any ("Fraud reported", "Suspicious activity reported")
| mv-expand TargetResources
| evaluate bag_unpack(TargetResources)
| extend _UserPrincipalName = tolower(tostring(column_ifexists("userPrincipalName", "")))
| extend UserId = tostring(column_ifexists("id", ""))
| project-rename FraudTimestamp=TimeGenerated;
// Fraud Report while SignIn?
let SignInFraudEvent = FraudEvent
| join kind=leftouter (SigninLogs
    | where Status.errorCode == 500121
    | extend _UserPrincipalName = tolower(UserPrincipalName)
    | project-rename SignInTimestamp=TimeGenerated, RelatedActorIP=IPAddress
    ) on _UserPrincipalName
| where SignInTimestamp between ((FraudTimestamp - 3m) .. (FraudTimestamp + 3m))
| extend FraudType = "SignIn"
| summarize RelatedEvents = make_set(CorrelationId1) by FraudTimestamp, _UserPrincipalName, OperationName, ResultDescription, FraudType, RelatedActorIP;
// Fraud Report while SSPR?
let SSPRFraudEvent = FraudEvent
| join kind=leftouter (AuditLogs
    | where ResultReason == "User started the mobile app notification verification option"
    | extend _UserPrincipalName = tolower(tostring(TargetResources[0].userPrincipalName))
    | extend RelatedActorIP = tostring(parse_json(tostring(InitiatedBy.user)).ipAddress)
    | project-rename SSPRTimestamp=TimeGenerated
    )on _UserPrincipalName
| where SSPRTimestamp between ((FraudTimestamp - 3m) .. (FraudTimestamp + 3m))
| extend FraudType = "SSPR"
| summarize RelatedEvents = make_set(CorrelationId1) by FraudTimestamp, _UserPrincipalName, OperationName, ResultDescription, FraudType, RelatedActorIP;
union SignInFraudEvent, SSPRFraudEvent
```

![Untitled](/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%209.png)

## **Response**

The following questions arise when considering the response:

Do we assume the password has been compromised?

- Is the new feature active? Has the password already been changed through SSPR?
- Have there been any other alerts regarding the user or their device?
- Are there endpoints that can be accessed with single-factor authentication?

Based on the answers to these questions and the assessment by the analyst, the following responses are recommended:

- Ignore the alarm as it was triggered within the context of a failed SSPR.
- Force a password reset in the AD for the user, for example, through an MDI action.
- Take extensive measures for a compromised user (e.g., locking the account, isolating the device).

## **Summary**

In summary, I wanted to say:

- As long as we have passwords, they can be stolen or guessed, which is why an user reporting a fraud attempt is a very valuable signal that should be addressed.
- The new feature has several advantages and addresses several weaknesses of the old feature.
- The seriousness of a compromised password depends largely on whether additional factors are required for access everywhere.
- Enable alerting and be prepared to act on those alerts.

For a better understanding and as a future reference I’ve also created a graphical overview of the different alerting options:

![Untitled](/post/2023-10-07-Microsoft-Entra-MFA-Fraud-Deep-Dive/images/Untitled%2010.png)