---
layout: page
multilingual: false
---

## References

### Thomas Naunheim / Entra ID Attack & Defense Playbook

- [Entra ID Attack & Defense Playbook – PRT/Token Replay Chapter](https://github.com/Cloud-Architekt/AzureAD-Attack-Defense/blob/main/ReplayOfPrimaryRefreshToken.md)
  *Thomas Naunheim & Sami Lamppu, 2022 (updated 2023). Core reference for PRT, RT, AT token types and attack scenarios. Chris Brumm listed as reviewer.*

- [Entra ID Attack & Defense Playbook – AiTM Chapter](https://github.com/Cloud-Architekt/AzureAD-Attack-Defense/blob/main/Adversary-in-the-Middle.md)
  *Thomas Naunheim & Sami Lamppu, Sept. 2024 (updated Dec. 2024). Covers AiTM attacks, GSA/Compliant Network as mitigation, and KQL hunting queries using NetworkAccessTraffic logs.*

- [Abuse and replay of Azure AD refresh token from Microsoft Edge in macOS Keychain](https://www.cloud-architekt.net/abuse-and-replay-azuread-token-macos/)
  *Thomas Naunheim, 2022. Deep dive into macOS token storage and replay attack paths.*

- [Analyzing Workload Identity Activity Through Token-Based Hunting](https://www.cloud-architekt.net/token-hunting-workload-identity-activity/)
  *Thomas Naunheim, Jan. 2026. Token hunting for non-human identities; notes that Compliant Network and Token Protection are not available for workload identities.*

- [MicrosoftCloudActivity KQL Function](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Hunting%20Queries/EID-TokenHunting/MicrosoftCloudActivity.func)
  *Thomas Naunheim. KQL function for hunting token-based activity across Microsoft Cloud.*

- [ConsentFix Hunting Query – Confidence on Token and Network Signals](https://github.com/Cloud-Architekt/AzureSentinel/blob/main/Hunting%20Queries/EID-Authentication/ConsentFix-HuntingConfidenceOnTokenAndNetworkSignals.kusto)
  *Thomas Naunheim. Hunting query leveraging GSA NetworkAccessTraffic logs, WAM signals, and token binding state.*

- [AADOps: Operationalization of Conditional Access](https://www.cloud-architekt.net/aadops-conditional-access/)
  *Thomas Naunheim. Gold standard for CA lifecycle management and automation.*

---

### glueckkanja (Co-Authored Posts)

- [ConsentFix: How a New OAuth Attack Bypasses Microsoft Entra Conditional Access](https://www.glueckkanja.com/en/posts/2025-12-31-vulnerability-consentfix)
  *Fabian Bader, Chris Brumm, Thomas Naunheim, Dec. 2025. Best available comparison of Compliant Network vs. Token Protection as mitigations. Includes concrete ConsentFix/AuthCodeFix scenario and hunting queries.*

- [Compliant Device Bypass – All you need to know](https://www.glueckkanja.com/en/posts/2025-01-14-compliant-device-bypass)
  *Fabian Bader, Chris Brumm, Thomas Naunheim, Jan. 2025. Relevant for Post 2 comparison of Compliant Device vs. Compliant Network.*

---

### Fabian Bader / cloudbrothers.info

- [Continuous Access Evaluation](https://cloudbrothers.info/continuous-access-evaluation)
  *Fabian Bader. Best community reference for CAE mechanics and location-based scenarios.*

- [EntraScopes.com](https://entrascopes.com/)
  *Fabian Bader. Reference for first-party app permissions; includes ConsentFix-relevant apps.*

---

### Chris Brumm (Own Posts – Cross-References)

- [Overview to Global Secure Access](https://chris-brumm.com/2024/07/30/Overview-to-Global-Secure-Access/)
  *GSA client prerequisites (IPv6, Secure DNS, QUIC). Referenced in Post 1.*

- [Global Secure Access in Conditional Access](https://chris-brumm.com/2024/08/06/Global-Secure-Access-in-Conditional-Access/)
  *Compliant Network + CAE combination explained. Referenced in Post 2 Part 1.*

- [Using Global Secure Access in Cross-Tenant scenarios](https://chris-brumm.com/2025/12/21/Cross-Tenant-Global-Secure-Access/)
  *B2B capabilities of GSA. Referenced in Post 2 Part 3 (Admin/PAW scenario) and Post 3 (Tenant Restrictions).*

- [Advanced Workbooks for Conditional Access](https://chris-brumm.medium.com/advanced-workbooks-for-conditional-access-9efa73b4a575)
  *Customized CA workbooks that extend the Microsoft-provided versions to also cover non-interactive sign-ins. Referenced in Post 2 rollout section.*

---

### Jan Bakker

- [Prevent AiTM with Microsoft Entra Global Secure Access and Conditional Access](https://janbakker.tech/prevent-aitm-with-microsoft-entra-global-secure-access-and-conditional-access/)
  *Early (2023) walkthrough of AiTM prevention using GSA Compliant Network. Good configuration reference.*

---

### Derk van der Woude

- [Microsoft Entra Internet Access to prevent AiTM attacks](https://derkvanderwoude.medium.com/microsoft-entra-internet-access-to-prevent-aitm-attack-s-31171db43a83)
  *Scenario and configuration walkthrough for AiTM prevention with GSA. Referenced in Chris' CA post.*

---

### BAADTokenBroker & PRT Cookie Theft Research

- [BAADTokenBroker – Secureworks GitHub](https://github.com/secureworks/BAADTokenBroker)
  *Original PowerShell post-exploitation tool by Yuya Chudo (Secureworks). Extracts PRT Cookie by talking directly to lsass, or creates one from credentials/WHfB keys. No updates since initial release.*

- [Black Hat Asia 2024 – Bypassing Entra ID Conditional Access Like APT](https://www.blackhat.com/asia-24/briefings/schedule/index.html#bypassing-entra-id-conditional-access-like-apt-a-deep-dive-into-device-authentication-mechanisms-for-building-your-own-prt-cookie-37344)
  *Presentation by Yuya Chudo introducing BAADTokenBroker. Covers device authentication internals, attack scenarios, and mitigations.*

- [Ballpoint – Pass the PRT](https://ballpoint.fr/en/blog/pass-the-prt)
  *Independent confirmation that `Request-PRTCookie` does not require administrative rights – standard user context is sufficient. Referenced in Post 2 Part 1 demo notes.*

- [BAADTokenBroker – BOF Fork (C2 Integration)](https://github.com/temp43487580/BAADTokenBroker)
  *Community fork that ports BAADTokenBroker as a Beacon Object File for C2 frameworks like Sliver. Indicates the technique has been adopted into offensive tooling.*

- [DEF CON 33 (2025) – Original Sin of SSO: macOS PRT Cookie Theft & Entra ID Persistence via Device Forgery](https://infocondb.org/con/def-con/def-con-33/original-sin-of-sso-macos-prt-cookie-theft-entra-id-persistence-via-device-forgery)
  *Extends the BAADTokenBroker attack concept to macOS via the Intune Company Portal SSO extension. Demonstrates PRT Cookie extraction under user-level permissions by bypassing process validation. DEF CON 33, August 2025.*

- [TROOPERS 25 – Breaking Down macOS Intune SSO: PRT Cookies Theft and Platform Comparison](https://troopers.de/troopers25/talks/ap8zxs/)
  *Companion research to the DEF CON 33 talk. Compares Windows and macOS SSO authentication flows and security controls. Relevant context for the Windows-only disclaimer in Post 2.*

- [Microsoft TechCommunity – Addressing data exfiltration: Token theft](https://techcommunity.microsoft.com/blog/microsoft-entra-blog/addressing-data-exfiltration-token-theft-talk/3915337)
  *Official Microsoft statement confirming that Compliant Network check via GSA protects apps not yet covered by Token Protection. Key reference for the "what GSA adds" argument in Post 2 Part 1.*

---

### PowerShell & Tooling

- [Migrate2GSA](https://github.com/microsoft/Migrate2GSA)
  *Community project maintained by Microsoft employees. PowerShell-based migration toolkit for transitioning from other SSE solutions to GSA. Also useful for scripting GSA provisioning from scratch.*

- [Entra PowerShell Beta Module – GSA Cmdlets](https://microsoft.github.io/GlobalSecureAccess/Entra%20Private%20Access/powershell/)
  *`Microsoft.Graph.Entra` Beta module with GSA-specific cmdlets for managing Private Access, profile assignments, and more. Referenced in Post 1 for scripting profile enablement.*

- [GSATool](https://github.com/microsoft/GSATool)
  *PowerShell-based troubleshooting tool from Microsoft. Runs 50+ tests across all GSA components without requiring module installation. Relevant for Post 4 (Coexistence) and Post 5 (Logging).*

- [Microsoft Graph Beta PowerShell – GSA PowerShell Samples](https://learn.microsoft.com/en-us/entra/global-secure-access/powershell-samples)
  *Official PowerShell samples for GSA using Microsoft.Graph.Beta module 2.10+. Includes break-glass recovery scripts relevant for Post 2 Part 2 (Exclusions/Rollout).*

---

### Official Microsoft Documentation

- [Enable Compliant Network Check with Conditional Access](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-compliant-network)
  *Post 2 – configuration reference.*

- [Learn about the Microsoft Traffic Profile](https://learn.microsoft.com/en-us/entra/global-secure-access/concept-microsoft-traffic-profile)
  *Post 1 – endpoint list, rule behavior, forward/bypass mode.*

- [Traffic forwarding profiles](https://learn.microsoft.com/en-us/entra/global-secure-access/concept-traffic-forwarding)
  *Post 1 – profile overview and processing order.*

- [Continuous Access Evaluation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)
  *Post 2 – CAE mechanics and user condition change flow.*

- [Token Protection in Conditional Access](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-token-protection)
  *Post 2 – Token Protection/Token Binding reference.*

- [Universal Continuous Access Evaluation (Preview)](https://learn.microsoft.com/en-us/entra/global-secure-access/concept-universal-continuous-access-evaluation)
  *Post 2 – Universal CAE for GSA access tokens, Strict Enforcement mode.*

- [Universal Tenant Restrictions](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-universal-tenant-restrictions)
  *Post 3.*

- [GSA Client for Windows](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-install-windows-client)
  *Post 1 – client prerequisites (IPv6, Secure DNS, QUIC).*

- [Enriched Microsoft 365 Logs](https://learn.microsoft.com/en-us/entra/global-secure-access/how-to-view-enriched-logs)
  *Post 5 – logging.*

- [Microsoft Digital Defense Report 2024](https://www.microsoft.com/en-us/security/security-insider/microsoft-digital-defense-report-2024)
  *Post 2 – token theft growth statistics.*

---
