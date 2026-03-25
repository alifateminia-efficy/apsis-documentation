---
source_file: Erik - Dynamics setup part 2 & code review of event listeners for campaigns.txt
domain: Apsis One Integrations
topics: [Outbound Manager Refactoring, Campaign Event Listeners, Microsoft Dynamics Access Control, Azure AD Configuration, OAuth Application Management, User Licensing and Permissions]
speakers: [Erik Andersson, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [Outbound Manager, Event Tool Activities, Email Send Outs, Apsis One Platform, Microsoft Dynamics 365, Azure Entra ID, OAuth Applications, Power Platform Admin Center]
session_type: knowledge-transfer
subdomains: [Architecture, Microsoft Dynamics Integration]
---

## Session Overview

This session covered two main areas: (1) a code review of the outbound manager refactoring story related to campaign event listener registration logic, and (2) extensive troubleshooting of Microsoft Dynamics 365 access permissions for integration team members. The team discovered a critical permission configuration issue where users had administrative access but could not read data due to missing licenses and incorrect access mode settings. The session also included Azure AD and OAuth application management review.

---

## Outbound Manager Refactoring Story

### Problem Statement: Child Activities and CRM System Requests

[Erik Andersson]: The refactoring addresses an issue with how child activities are handled when creating campaigns in connected CRM systems. When an email campaign is created, the email tool makes a request to Apsis One's outbound manager, which then communicates with the CRM system to register the campaign activity.

**Child activities concept**: When you create an event tool activity, it can contain sub-activities such as:
- Email send outs
- Form registration activities
- Event-related activities

Currently, all child activities are being sent to the CRM system regardless of whether the CRM system wants them.

### The E-deal/FCC Corporate Issue

[Erik Andersson]: E-deal (formerly called FCC Corporate) has explicitly stated they are not interested in receiving child activities. They only want the parent event tool activity registered in their system. However, the current flow creates event listeners for all child activities (emails, forms, etc.) in Apsis before requesting data from the CRM system.

This creates a logical problem:
- Apsis registers listeners for emails, forms, and other child activities
- CRM system responds: "I only created an activity for the event tool"
- Apsis still sends child activity events to the CRM system for non-existent activity records

### Proposed Solution: Reverse the Request Order

[Erik Andersson]: The solution is to reverse the order of operations:

**Current flow (problematic)**:
1. Register all event listeners in Apsis
2. Send request to CRM system
3. CRM system responds with created activities
4. Listeners fire for all activities (including unwanted ones)

**New flow (proposed)**:
1. Send request to CRM system asking which activities they created
2. Receive list of activities CRM is interested in
3. Register listeners only for those activities

### Code Review Findings

[Erik Andersson]: The pull request implementing this change goes far beyond what's necessary. The minimal implementation should involve:
- Moving one code block (the CRM request) before the listener registration
- Using the CRM response as input for listener registration instead of the full list

> "In essence, we should only need to move like one code block before and then take the output of this request as the input for what when we register the listener. But this pull request... goes way and beyond and does some more refactoring."

[Erik Andersson]: The current PR implementation is overly complex and would be easier to rewrite from scratch. This will be moved to the integration backlog for reimplementation.

---

## Microsoft Dynamics 365 Access Troubleshooting

### Initial Access Issues

[Tomasz Kowalski] and [Michal Rosikiewicz] were unable to access the Dynamics integration development environment despite having administrative credentials. They received:
- HTTP 404 errors on the Dynamics instance
- "Security group not set" errors
- "You don't have access to these records" messages

### Root Cause Discovery: Administrative vs. Read-Write Access Modes

[Erik Andersson]: The critical discovery was that Dynamics has two different access modes:
- **Administrative access**: Can manage the instance and configurations but cannot READ DATA
- **Read-Write access**: Can both manage and view/edit data

Users were set to "Administrative" mode, which prevented them from viewing actual records in the system, even though they had the ability to configure the instance.

[Erik Andersson]: This is a Microsoft requirement that was new:

> "You weren't administrator, but you only had administrative access and this is something I don't know. This is something new. They seem to have introduced because I never had to do this nitpicky before."

### The License Prerequisite

To change a user from Administrative to Read-Write access mode, the user must first have appropriate licenses assigned in Azure AD/Entra. Without licenses, the system automatically reverts the access mode to Administrative.

### Solution Steps

1. **Assign licenses in Entra ID**: The following licenses were required:
   - Basic User license
   - Sales Enterprise Edition license (limited availability—only 3 available)
   - Power Apps/Power Automate licenses
   - Customer Fabric license (if applicable)

2. **Change access mode to Read-Write**: Once licenses are assigned, the access mode can be changed from Administrative to Read-Write

3. **Refresh user session**: Users must log out and log back in for the changes to take effect

4. **Verify in Dynamics**: User can now access Sales Hub and view/interact with records

### License Allocation Issues

The Sales Enterprise Edition license is a scarce resource (only 3 available). The team had to audit existing users and reassign licenses:
- Removed Benjamin from Sales Enterprise Edition (he only needed viewing access)
- Removed Janita from Sales Enterprise Edition (disabled user)
- Allocated licenses to [Tomasz Kowalski] and [Michal Rosikiewicz] for active development work

---

## Azure Entra ID and OAuth Application Management

### External User Removal Issue

[Michal Rosikiewicz] and [Tomasz Kowalski] identified that an external FCC user (from external invitation) was blocking proper integration connection attempts. This user was preventing the plugin installation flow from working correctly.

**Action taken**: Deleted the external FCC user from the Entra tenant after confirming their credentials were no longer needed for integration testing.

[Erik Andersson]: External FCC users cannot be used for the initial OAuth setup because:
> "You cannot do the initial setup with the external FCC users because when you do that with your external FEC user, it is trying to add permission to FEC for the application and that of course we are not allowed to do because we are not global administrators."

Using dedicated internal users (created directly in the lab tenant) avoids this permission delegation issue.

### OAuth Application Configuration

The team reviewed the OAuth applications configured in Azure:

**Key OAuth applications in use**:
- **Apsis One Sandbox 2**: Local development on localhost:3000
- **Apsis One Staging 2**: Staging environment redirect URL
  - Redirect URI: `https://stagecloud.appsis.co/integrations/callback`
- **Apsis One Beta**: Beta environment
- **Apsis One APEC Prod**: Production environment
- **Apsis One EU Prod**: Production environment

**Decommissioned applications** (should be deleted):
- APEC Beta (environment no longer exists)
- Sandbox One (sandbox environment was removed during cost reductions)
- Sandbox Two (sandbox environment was removed during cost reductions)
- Various test applications without redirect URIs configured

### Disaster Recovery and Secret Management

[Erik Andersson]: Discussed the importance of maintaining backup access to critical OAuth accounts. When credentials are needed:
- Saved credentials for Nicholas's account (who owns most OAuth applications) for disaster recovery scenarios
- If the original app secret is lost, a new client secret can be generated without changing the app ID
- Customers would only need to update the secret value, not reinstall the application

### Expired and Orphaned Applications

Several applications were found with expired credentials or unclear purposes:
- **Hassan Action Adapter**: Unknown purpose, no redirect URIs, uses localhost endpoints
- **Integration Tests**: Local testing application, can likely be deleted
- **Web API Test**: Has expired credentials, unclear if still in use
- **Permission Test Client**: No credentials configured, candidate for deletion

[Erik Andersson]: Before deleting applications, verify that no customers are using them, as deletion would require customer reinstallation.

---

## User Account Audit and Cleanup

### Disabled Accounts

The team systematically reviewed and cleaned up user accounts:

**Disabled users** (should remain disabled or deleted):
- **Benjamin**: Former manager, disabled, had Global Administrator role
- **Janita**: Former employee, disabled, had Global Administrator role  
- **Shraddha**: On parental leave, should remain enabled (still employed, may return)
- **Sravidya**: Former employee, account disabled

**Test/temporary accounts** (deleted):
- **Felix**: Old account with managerial access, deleted
- **Admin Tool**: Test account, disabled and deleted
- **External FCC User (Wukash Grabowski)**: External invitation no longer needed, deleted

### Active Development Users

**Users requiring access**:
- [Tomasz Kowalski]: Internal developer, assigned Read-Write access with appropriate licenses
- [Michal Rosikiewicz]: Internal developer, assigned Read-Write access with appropriate licenses
- **Najeeb** (CRM Consultant at Sierra Consultancy): Disabled by default, can be re-enabled as needed
- **Gustav** (CRM Consultant at Sierra Consultancy): Disabled by default, can be re-enabled. Main architect behind Dynamics solution design
- **Lucas** (pending): New user account created, password set, awaiting first login

### User Information Documentation

[Michal Rosikiewicz]: Added company information to user profiles to clarify their roles:
- **Najeeb**: Marked as "CRM Consultant - Sierra Consultancy" (Swedish: "CRM konsult")
- **Gustav**: Marked as "CRM Consultant - Sierra Consultancy"
- **Legacy user note**: "Apsis Lead Integration User" created in 2015, predates the integration platform by 4 years, likely connected to legacy Pro system

### Critical Dependency: Nicholas's Account

[Erik Andersson]: The Apsis One OAuth applications are owned by Nicholas, an account with Global Administrator role. His credentials are saved for disaster recovery because:

> "I saved the access to his account because I am extremely paranoid by nature or things like this and so far I have only been proven right. So one time when MA had crashed for essentially 2 weeks, I was super paranoid... Luckily, because of that, we didn't lose 300,000 messages in MA because we could recreate it from the logs."

If Nicholas's account is ever permanently removed, new OAuth applications would need to be created and all customers would need to reinstall.

---

## Known Challenges and Workarounds

### Global Administrator Requirement for Setup

[Erik Andersson]: The initial OAuth setup in Dynamics requires a Global Administrator user. This is a hard technical requirement that cannot be bypassed for the legacy Dynamics connector:

> "Like it is a hard requirement on dynamics that whenever you add this like oauth user to the organisation, it has to be a global admin because it is such a sensitive operation and the whole flow is dependent on a oauth user being added to the organisation."

**Contrast with Generic Connector**: The generic connector avoids this issue by communicating with an intermediate service (Siteshop) instead of directly with Dynamics:
> "For the generic connector, we connect to an intermediate service. So apsis will never touch dynamics in that flow, we only communicate with the intermediate service and the intermediate service handles everything with dynamics."

### Permission Cascades and Access Mode Confusion

The confusion between Administrative and Read-Write access modes is a recent Microsoft change. Users may have all the right roles assigned but still cannot access data if in Administrative mode. This requires both:
1. Correct license assignment
2. Explicit access mode change to Read-Write
3. Session refresh

---

## Key Takeaways

1. **Outbound Manager Refactoring**: The current PR implementing child activity listener changes is over-engineered. The solution requires only reversing the request order (query CRM for desired activities first, then register listeners) rather than extensive refactoring. This story will be rewritten.

2. **Dynamics Access Control**: Users need both correct licenses AND Read-Write access mode to view data. Administrative access alone only allows configuration, not data access. This is a Microsoft requirement.

3. **License Scarcity**: Sales Enterprise Edition is limited (only 3 seats). Regular audits needed to ensure licenses are allocated to active users who need them.

4. **External Users for Setup**: External FCC users cannot be used for initial OAuth setup due to permission delegation issues. Use dedicated internal users instead.

5. **OAuth Application Ownership Risk**: Applications owned by a single external consultant (Nicholas) represent a single point of failure. His credentials are maintained for disaster recovery.

6. **Sandbox Cost**: Integration sandbox environments were removed during cost reductions. Current architecture uses staging as the test environment instead.

7. **User Hygiene**: Regular cleanup of disabled/departed users is important. Dedicated internal users should be preferred over external guest accounts for long-term integration access, as their access is not automatically revoked when external credentials are disabled.

8. **Generic Connector Advantage**: The generic connector approach via Siteshop eliminates the Global Administrator requirement for customers, significantly improving deployment experience compared to legacy Dynamics direct integration.

---

## Unresolved Questions and Action Items

- **Hassan Action Adapter application**: Purpose unclear, appears to be a legacy test application. Needs investigation before deletion.
- **Web API Test application**: Contains expired credentials and endpoints. Verify it's not in use before deletion.
- **Shraddha's account**: Currently enabled despite being on parental leave. Clarify company policy for parental leave access retention.
- **Nicholas's account management**: When Erik departs, there must be a formal handover of the OAuth application ownership or credentials to another developer. This should be documented in the handover process.
- **Apsis Lead Integration User (2015)**: Purpose unclear, likely connected to legacy Pro system. Do not modify without confirming current dependencies.

**Upcoming**: Erik plans to formalize knowledge transfer checkpoint reviews with Tomasz and Michal to measure handover quality against explicit KPIs before his departure (targeting February).