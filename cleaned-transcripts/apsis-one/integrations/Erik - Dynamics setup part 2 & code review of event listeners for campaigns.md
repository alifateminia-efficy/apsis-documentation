---
source_file: Erik - Dynamics setup part 2 & code review of event listeners for campaigns.txt
domain: Apsis One Integrations
topics: [
  "Outbound Manager and Campaign Refactoring",
  "Event Listener Architecture",
  "Microsoft Dynamics 365 Setup and Access Control",
  "User Permissions and Licensing in Azure Entra",
  "OAuth Application Management",
  "Generic vs Legacy Connector Flow"
]
speakers: [
  "Erik Andersson",
  "Tomasz Kowalski",
  "Michal Rosikiewicz"
]
key_components: [
  "Outbound Manager",
  "Event Listeners",
  "Microsoft Dynamics 365",
  "Azure Entra (formerly Azure AD)",
  "OAuth Applications",
  "Apsis One Integration Platform",
  "Generic Connector",
  "E-deal (Efficy Corporate)"
]
session_type: knowledge-transfer
subdomains: [
  "Architecture",
  "Different Types of Connectors",
  "Generic Connector",
  "Outbound Flow",
  "Microsoft Dynamics"
]
---

## Session Overview

This session covers two major topics: (1) a high-level architectural discussion of a refactoring story for the outbound manager's campaign event listener registration flow, particularly how different CRM systems (E-deal/Efficy Corporate) have different requirements for which child activities they want to receive, and (2) a deep technical troubleshooting and hands-on walkthrough of Microsoft Dynamics 365 user access configuration, Azure Entra permissions, OAuth application management, and environment setup. The refactoring discussion emphasizes moving listener registration to *after* CRM system acknowledgment rather than before. The Dynamics troubleshooting revealed that users with administrative access but not read/write permissions cannot actually view data—a critical discovery that required assigning Microsoft licenses before changing the administrative role to read/write access.

---

## Campaign Refactoring: Event Listener Registration Flow

### The Current Problem with Child Activities

[Erik Andersson]: The refactoring story exists because one CRM system requested a modification to the generic connector flow. Currently, when an email campaign is created, the email tool makes a request to the outbound manager, which then makes a request to the CRM system saying:

> "Hello CRM system, someone has created an email campaign here with this Activity ID and this name"

The CRM system responds confirming they created a campaign.

However, for event tool activities, there's a concept of **child activities**—these are sub-activities within an event tool activity. For example, an event tool activity might have:
- Email send-outs
- Form activities for registration
- Other related activities

Today, all of these child activities are sent to the CRM system as well. The registration flow works like this:

1. All event listeners are registered in Apsis first
2. Only then is the request sent to the CRM system

### The E-deal (Efficy Corporate) Constraint

[Erik Andersson]: E-deal (also called Efficy Corporate) has stated they are **not interested in these child activities at all**—they only want the event tool activity itself. They don't want the email send-outs, form submissions, or other sub-activities sent to them.

This creates a problem: because listeners are registered in Apsis *before* asking the CRM system what they actually want, Apsis still registers listeners for all child activities. Then when E-deal sends events for form submissions and emails, Apsis tries to forward them to the CRM system, but E-deal never created activities for those—there's nothing to connect the events to.

### The Proposed Solution: Reorder the Flow

[Erik Andersson]: The fix is to shuffle the logic:

**Before (Current):**
1. Register all listeners in Apsis for all activities
2. Send request to CRM system
3. CRM system responds (but some activities weren't actually created)
4. Events arrive and have nowhere to attach

**After (Proposed):**
1. Send request to CRM system
2. CRM system responds with list of activities they actually created
3. Register listeners *only* for the activities the CRM system confirmed
4. Events only arrive for activities that exist on the CRM side

The code change should be minimal—move one code block before the listener registration, then use the CRM system's response as the input for which listeners to register instead of registering listeners for everything.

### Pull Request Scope Issue

[Erik Andersson]: When reviewing the implementation branch, it was discovered that the PR goes far beyond what's actually needed—it includes additional refactoring that isn't required for this fix. The logic shuffle could be accomplished by moving one code block and changing the input source for listener registration, but the PR does much more. The assessment was that it would be easier to start the implementation from scratch rather than salvage the PR. This will be moved to the integration board for a complete reimplementation as a learning opportunity.

---

## Microsoft Dynamics 365 Access Control Troubleshooting

### The Core Issue: Administrative vs Read/Write Access

During the session, Tomasz and Michal were experiencing persistent access problems to the Dynamics 365 sandbox environment (`TKO lab`), despite having what appeared to be sufficient permissions. The errors included:

- HTTP 404 errors when trying to access the integration development environment
- "Security group not set" errors
- Permission denied messages even after being assigned various roles

[Erik Andersson]: After investigation, the root cause was discovered: **users with administrative role but no read/write access cannot actually see or access data in Dynamics, even though they can manage the instance itself**. This is a Microsoft feature (or "feature," as Erik sarcastically noted) that prevents data access without explicit read/write permissions.

### The Licensing Requirement

The critical discovery: **you cannot change a user from administrative to read/write without first assigning the appropriate Microsoft licenses in Azure Entra (formerly Azure AD)**.

The process is:

1. Go to `admin.microsoft.com` → Active Users
2. Select the user and click "Licenses and apps"
3. Assign the required licenses:
   - **Power Automate Free** (required)
   - **Dynamics 365 Sales Enterprise Edition** (if sales access needed; limited to 3 available licenses)
   - **Microsoft Fabric** (for data analytics; optional)
   - **Customer Service** or other Dynamics roles as needed
4. In Dynamics user settings, change role from "administrative" to **read/write**
5. Refresh the user profile to apply changes

[Erik Andersson]: Once the licenses are assigned and read/write is set, users can then access data and see tables like contacts, leads, and marketing lists. Without the licenses, attempting to set read/write access causes the system to revert to administrative.

### The External User Complication

Early in the troubleshooting, an **external FCC (Efficy) user** named "mural app" had been invited to the environment. This external user was causing permission issues and preventing proper OAuth setup.

[Michal Rosikiewicz]: When trying to set up the Dynamics integration in Apsis One, the external user invitation was blocking the process. The solution was to remove the external FCC user from the Entra directory entirely, which allowed the internal dedicated users to properly authenticate and authorize the OAuth flow.

[Erik Andersson]: This is a key insight: **external users cannot perform the initial OAuth setup** because the system tries to grant permissions to the external organization (FCC in this case), but since Apsis is not a global admin for FCC, the permission grant fails. Internal dedicated users in the tenant must perform the initial OAuth setup, after which the integration works.

### User Roles and Permissions Setup

The correct permission set for developers needing full access includes:

**In Entra/Microsoft 365 Admin:**
- Global Administrator (needed to manage Azure applications and OAuth setups)
- Global reader (for viewing-only audit)
- Power Platform Administrator (for Power Automate and canvas apps)
- Dynamics 365 Administrator (if managing Dynamics-specific settings)

**In Dynamics 365 itself:**
- System Administrator role (read/write access)
- Basic User role (foundational permissions)
- Sales Team Member role (if working with sales data)

[Erik Andersson]: The "administrative" access mode in Dynamics is distinct from these roles—it's a user mode that lets you *manage* the instance but not *view* its data. This is counterintuitive and was a source of significant confusion.

### License Availability and Constraints

The staging environment has **3 available Dynamics 365 Sales Enterprise Edition licenses**, which is the most restrictive license type. Other licenses like Power Automate and Fabric have more availability.

During the session, licenses were redistributed:
- **Benjamin** (no longer needed, now disabled): was removed from Sales Enterprise Edition
- **Janita** (another old test user, now disabled): was removed from all licenses
- **Lukas** (new developer): created and assigned with appropriate licenses
- **Erik** and **Michal**: verified to have correct licenses for their roles

---

## Azure Entra and OAuth Application Management

### OAuth Application Registration Structure

Apsis One maintains multiple OAuth applications in Azure Entra for different environments:

```
App Registrations in Azure (apsis-on.microsoft.com tenant):
├── Apsis One Sandbox 2 (local development)
│   └── Redirect URIs: http://localhost:xxxx/callback
├── Apsis One Staging 2 (current staging environment)
│   └── Redirect URIs: https://[stage].apsis-one.cloud/callback
├── Apsis One Beta
│   └── Redirect URIs: https://[beta].apsis-one.cloud/callback
├── Apsis One Production
│   └── Redirect URIs: https://apsis-one.cloud/callback
├── Apsis One APEC (no longer used, should be deleted)
├── Web API Test (from old development, should be deleted)
├── Integration Test (old, not actively used)
└── [other legacy and test apps]
```

### Critical Application Ownership Issue

[Erik Andersson]: A significant issue was discovered: **the production OAuth applications are owned by a disabled user account** (Nicholas's account is disabled but listed as owner of most production apps). This creates a single point of failure—if there's ever an issue with the app, only a global admin can fix it since the owner is disabled.

The solution requires using saved credentials for the disabled owner's account to make changes. In theory, a new credential secret can be generated and stored in Azure Key Vault (AVS) without customers needing to reinstall, as long as the Application ID remains the same.

[Michal Rosikiewicz]: If a secret expires or must be rotated:
1. Generate a new client secret
2. Store it in the Azure Key Vault
3. Update the configuration in the integration flow
4. Customers do **not** need to reinstall because the app registration ID hasn't changed—only the secret changes

### Cleaning Up Unused Applications

During the session, several applications were identified for cleanup:

**Can be deleted immediately:**
- **Sandbox** apps (no longer needed, sandbox environments were shut down during cost reduction)
- **Web API Test** (local development artifact, not in use)
- **Apsis One APEC Beta** (environment no longer exists)
- **Integration Tests** (old test app with no active endpoints)
- **Permission Test Client** (no credentials, not in use)
- **Temp Beta** (no credentials, likely old POC)
- **Test** app (unused)

**Must stay:**
- **Apsis One Production** (active customers)
- **Apsis One Beta** (active testing)
- **Apsis One Staging 2** (development and testing)
- **Apsis One Dev** (development, though certificate is missing)

### The Hassan/MSD Action Adapter Mystery

[Erik Andersson]: There's an application called "Hassan action adapter" or "MSD action adapter" with no clear purpose. It has no redirect URIs configured and appears to be associated with a URL endpoint that serves a generic sign-in page. It may be a proof-of-concept from an earlier experiment but should be investigated before deletion.

---

## Generic Connector vs Legacy Dynamics Connector

### OAuth User Requirement Difference

[Erik Andersson]: A critical architectural difference exists between the legacy Dynamics connector and the new generic connector regarding OAuth user requirements:

**Legacy Dynamics Connector (current production):**
- Requires a **global administrator** to approve the OAuth consent screen when setting up
- This is a hard Microsoft requirement because Apsis adds an OAuth user to the organization, which is a sensitive operation
- Global admins are notoriously hard to get in large organizations—usually requires high-level IT approval
- This is a major pain point for customer implementations

**Generic Connector (future direction):**
- Does **not** require Apsis to touch Dynamics directly
- Apsis connects to an **intermediate service** instead
- The intermediate service (likely Siteshop or similar) handles all Dynamics interactions
- Once the generic connector fully replaces the legacy connector, customers won't need to approve Apsis as an organization user

### Test Account for Non-Admin OAuth

[Erik Andersson]: The team had attempted to investigate whether OAuth setup could work without a global admin using a test account called "test_not_admin". The conclusion was that **it's impossible**—Microsoft's Dynamics 365 OAuth flow absolutely requires global admin approval for the organization-level user grant.

This is why the generic connector architecture is so important: it moves the OAuth integration point to an intermediate service, removing the organizational user requirement entirely.

---

## User Management and Security Cleanup

### Disabled and Removed Users

During the session, the following users were disabled or deleted to clean up the staging environment:

| User | Status | Reason |
|------|--------|--------|
| Benjamin | Disabled | Old test account, no longer needed |
| Janita | Disabled | Old test account, disabled by Erik previously |
| Sravidya (Shraddha) | Disabled | Was on parental leave; no longer with company |
| Felix | Deleted | Left team long ago; only had managerial access |
| Admin Tool | Deleted | Test account for admin flow testing |
| External Mural App (FCC) | Deleted | Was blocking OAuth setup; external users can't perform initial setup |
| Wukash Grabowski (external) | Deleted | External user; confirmed not needed |

[Erik Andersson]: When removing external users, make sure they're actually the external invitations, not dedicated internal accounts. External users invited via another organization's tenant get disabled automatically when their home organization account is disabled, making them safer from the "disgruntled former employee" scenario.

### Still-Active Users

| User | Purpose | Status |
|------|---------|--------|
| Erik Andersson | Developer/Architect | Global Admin, all licenses |
| Tomasz Kowalski | Developer | Read/Write access, being configured |
| Michal Rosikiewicz | Developer | Global Admin (just added), selected licenses |
| Lukas | New Developer | User created, needs password setup |
| Najeeb | CRM Consultant (external) | Disabled (on demand, can be re-enabled) |
| Gustav Eklund | CRM Consultant/Architect | Disabled (on demand, can be re-enabled) |
| Masha | Another developer | Accessible via external invite; not global admin |
| Stan | CTO | Global Admin |
| Apsis Lead Integration User | Unknown (created 2015) | **Do not touch** — predates integration platform, may be critical for Pro/legacy systems |

[Erik Andersson]: The "Apsis Lead Integration User" is a legacy account created in 2015, before the integration platform existed. It's almost certainly tied to the old Apsis Pro lead system. Without knowing its exact purpose, it should never be deleted or disabled.

---

## Key Takeaways

### Campaign Listener Refactoring
1. The outbound manager's event listener registration order needs to be reversed: get CRM confirmation first, register only for what they want
2. The current PR does too much refactoring; a cleaner implementation is needed
3. E-deal (Efficy Corporate) requirements drove this architectural change

### Dynamics 365 Access Control
1. **Administrative access ≠ read/write access**—administrative users can manage instances but not see data
2. **Licenses must be assigned in Entra *before* setting read/write access**—without licenses, read/write reverts to administrative
3. The key licenses needed are Power Automate Free and Sales Enterprise Edition (if sales data access needed)
4. External FCC users cannot perform initial OAuth setup—internal dedicated users are required
5. Once OAuth setup is done by internal users, the integration works

### Azure and OAuth Management
1. Multiple OAuth apps exist for different environments (sandbox, staging, beta, prod)
2. Production apps are owned by a now-disabled user account—a risk that requires careful secret rotation procedures
3. Unused/test apps should be cleaned up to reduce attack surface
4. App IDs persist across secret rotations, so customers don't need to reinstall if only the secret changes

### Architecture Insight
1. The generic connector's design (connecting through an intermediate service instead of directly to Dynamics) will eliminate the global admin requirement for future customers
2. Legacy Dynamics connector is stuck with the global admin requirement due to Microsoft's OAuth architecture

---

## Unresolved Questions and Action Items

### Pending Investigations
1. **Hassan/MSD Action Adapter**: Purpose unknown, should investigate before deciding to delete
2. **Lukas's password**: Should be set and communicated (placeholder was discussed but not finalized)
3. **Masha's role**: Should be clarified whether they need global admin access

### Planned Follow-up Actions
- **Erik**: Monday/Tuesday will begin assessing handover progress on theoretical knowledge
- **Michal/Tomasz**: May need additional sessions or written guides on specific areas before Erik's departure
- **Nicholas's OAuth App Ownership**: Secret rotation and access procedures should be formally documented
- **Cleanup of unused OAuth apps**: Schedule deletion of confirmed unused applications
- **User Account Maintenance**: Establish policy for removing access when employees leave (currently some manual processes)

### General Guidance
[Erik Andersson]: The team should **not** maintain these Azure/Entra/Dynamics instances without clear documentation and runbooks. Developers should not be the primary maintainers of security-sensitive infrastructure. Future considerations:
1. Document all user roles, their purposes, and their required permissions
2. Create a runbook for onboarding and offboarding users
3. Implement a regular audit schedule for unused applications and disabled accounts
4. Consider whether a dedicated IT operations or DevOps team should manage these Azure resources