---
source_file: Erik - Dynamics setup part 2 & code review of event listeners for campaigns.txt
domain: Apsis One Integrations
topics: [Outbound Manager Refactoring, Event Listener Registration Flow, CRM System Integration Patterns, Microsoft Dynamics Setup, User Permissions and Licensing, Azure AD Configuration, OAuth Application Management, Integration Environment Cleanup]
speakers: [Erik Andersson, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [Outbound Manager, Generic Connector, Event Listeners, Child Activities, E-deal (Efficy Corporate), Microsoft Dynamics 365, Entra ID, Azure AD App Registrations, OAuth Applications, Power Platform Admin Center]
session_type: knowledge-transfer
subdomains: ['Architecture', 'Different Types of Connectors', 'Generic Connector', 'Outbound Flow', 'Microsoft Dynamics', 'Efficy Enterprise 12.0', 'Efficy Enterprise 12.1']
---

## Session Overview

This session covered two main areas: (1) a code review discussion of the refactoring required for the **outbound manager and patch campaigns** story, focusing on reordering event listener registration versus CRM system requests; and (2) an extensive hands-on troubleshooting and setup session for Microsoft Dynamics integration access. The team discovered and resolved critical permission issues related to the distinction between "Administrative" and "Read Write" access modes in Dynamics, identified the prerequisite licensing requirements, and performed significant cleanup of user accounts and Azure AD applications in the Apsis One integration environment.

---

## Outbound Manager Refactoring and Event Listener Registration

### The Current Problem with Child Activities

[Erik Andersson]: Currently, when an email campaign is created, the email tool makes a request to the outbound manager, which then requests the CRM system to create a corresponding campaign. For **Event tool activities**, there is a concept of **child activities**—sub-activities like email send-outs and form registration activities that are created alongside the main event activity.

Today, the system registers all child activity event listeners in **Apsis One first**, before making the request to the CRM system. The CRM then responds confirming which activities were created.

### The E-deal Use Case and the Core Issue

[Erik Andersson]: **E-deal (Efficy Corporate)** has stated they are only interested in receiving the main event tool activity—not the child activities like email send-outs or form submissions. However, because listeners are registered in Apsis before the CRM request, the system still sends all these child activity events to E-deal even though E-deal never created corresponding activities for them.

> "They don't want to have these sent to them and hearing now it suddenly becomes a problem because... we will still send these events to the CRM system, despite them not saying yeah yeah, but I didn't create any activity for that."

This creates events for activities that don't exist in the customer's CRM system.

### The Required Refactoring Solution

[Erik Andersson]: The solution is to **reverse the order of operations**:

1. **First**: Make the request to the CRM system
2. **Wait for the response**: The CRM tells us which activities it actually created and is interested in receiving events for
3. **Then**: Register event listeners only for those activities

This way, we only create listeners for activities that actually exist in the customer's system.

### Pull Request Status and Implementation Notes

[Erik Andersson]: A branch exists for this refactoring, but the implementation goes "far and beyond" what's actually needed. In essence, only one code block needs to move before the request, and the output of the CRM request should become the input for listener registration. Instead of using a complete activity list for registration, we should use only the CRM response.

> "It's almost to the level. It's easier to start from the beginning, unfortunately... it's not going to be any pointing us going through all of this here and now."

The PR involves unnecessary refactoring beyond the minimal scope. The recommendation is to move this story to the integration backlog and potentially rebuild it rather than try to salvage the current implementation.

---

## Microsoft Dynamics Setup and Permissions Troubleshooting

### Initial Access Issues

[Erik Andersson]: Tomasz did not have **read/write access** in Dynamics, only administrative access. Additionally, he was not a **System Administrator** user but had administrative access in a different form—described as a "nitpicky" distinction that Microsoft seems to have introduced.

### The Administrative vs. Read Write Discovery

Through extensive troubleshooting, the team discovered a critical distinction in Dynamics/Entra ID:

- **Administrative mode**: Allows you to manage the Dynamics instance (settings, configurations, user management) but does **not** grant access to view or interact with the actual data in the system
- **Read Write mode**: Grants actual data access and is what's needed to use Sales Hub and other applications

[Erik Andersson]: "I notice one difference between our users... I don't know if this is the course of it or not."

The issue manifested as a **"Security group not set"** error or **HTTP 404** errors when trying to access Sales Hub, even though the user had administrative permissions to manage the environment.

### The Licensing Prerequisite

The team discovered that **you cannot change a user from Administrative to Read Write mode without first assigning appropriate Microsoft licenses**. If you attempt to change the mode without licenses, the system reverts the user back to Administrative.

Required licenses for basic data access include:
- **Basic User** license
- **Sales Team Member** license (or **Sales Enterprise Edition** if you need access to leads and marketing lists)
- **Power Automate Free** (or premium variant)
- **Power Apps** licenses (if needed)

[Erik Andersson]: "So the only thing the sales one should do is if you have the sales you can see leads here and you can see marketing lists, but that's the only difference."

### The Fix Process

The correct sequence to enable a user:

1. Assign required licenses in **Microsoft 365 Admin Center** (admin.microsoft.com)
2. Change the user's Dynamics role assignment from **Administrative** to **Read Write** in Power Platform Admin
3. Refresh/disable and re-enable the user in Entra ID to force synchronization
4. The user's access is now enabled with proper data permissions

[Michal Rosikiewicz]: After applying the fix and logging back in, "I'm able to go to integrations development board and then when I go to sales hub I still have..." [confirms access is restored]

### External User Complications

The team had invited an external FCC user that was causing permission issues. External users invited to the organization cannot complete the initial OAuth setup because:

- When an external user attempts to authorize the Apsis One application, it tries to add permissions to FCC's tenant (the external organization)
- The external user doesn't have global administrator rights in the FCC tenant to grant those permissions
- The authorization fails

**Solution**: Use dedicated internal users in the Dynamics instance for initial setup. Once setup is complete, the OAuth flow can potentially work with external users, but the initial application authorization step must be performed by someone with global admin rights in the customer's organization.

[Erik Andersson]: "You cannot do the initial setup with the external FCC users because... when you do that with your external FEC user, it is trying to add permission to FEC for the application and that of course we are not allowed to do."

---

## Azure AD and OAuth Application Configuration

### OAuth Application Structure

The Apsis One integration uses multiple OAuth applications in Azure AD (now Entra ID), one for each environment:

- **Apsis One Sandbox 2**: For local development with redirect URI `http://localhost:4200`
- **Apsis One Staging 2**: For staging environment with callback URL `https://cloud-integration-stage.apsis-ip.com/...`
- **Apsis One Beta**: For beta environment
- **Apsis One EU Prod**: For production

Each application has:
- **Application (client) ID**: Remains constant across secret rotations
- **Client secrets**: Expire and can be rotated without affecting customer installations
- **Redirect URIs**: Define which URLs the OAuth flow can callback to after authentication

### Key Learning: Secret Rotation Without Reinstallation

[Erik Andersson]: If a client secret expires or is compromised, a new secret can be generated and updated in the Apsis One flow/secret manager without requiring customers to reinstall the application or re-authorize it.

> "If App ID doesn't change, new secret only requires to update the secret value, not the whole app ID."

This is because customers authorize the application by its Application ID, not by the secret. Changing only the secret doesn't require customers to re-authorize.

However, if the entire application is deleted and a new one created with a different Application ID, **all customers must reinstall and re-authorize**.

### Expired and Unused Applications

The team identified several applications that could be deleted or disabled:

- **Apsis One Sandbox**: Environment no longer exists; can be deleted
- **Apsis One Sandbox 2**: Environment no longer exists; can be deleted
- **Apsis One APEC Beta**: Environment deleted during cost reduction; can be disabled
- **Web API Test**: Has no callback URLs and appears to be a test artifact; can be deleted
- **Permission Test Client**: No credentials; can be deleted
- **Temp Beta**: No credentials; can be deleted
- **Hassan Action Adapter** and **Hassan Integration Tests**: Have only localhost endpoints; likely local testing artifacts

### OAuth App Ownership and Disaster Recovery

[Erik Andersson]: One critical issue discovered: the owner of the production OAuth applications is a disabled user account (Nicholas). This means:

- Only the owner can delete or modify the app
- If the secret is lost and cannot be recovered, a new application would have to be created
- All customers would need to reinstall with the new application ID

[Erik Andersson]: "Because now the customers will have to reinstall and that's not gonna be fun for anyone."

As a precaution, Erik saved the credentials for this disabled account specifically for disaster recovery scenarios. This allows him to log in and perform critical operations if needed.

**Future Note**: When the application owner account is no longer accessible, adding a global administrator as a co-owner would provide continuity, though historically the team has never needed to use this.

---

## User Management and Environment Cleanup

### User Permission Audit

The team conducted a comprehensive review of all users in both the development/staging Dynamics environment and the Apsis One production Azure AD:

**Users Disabled or Deleted**:
- **Benjamin**: Global administrator; no longer needed; disabled
- **Janita**: Disabled (was already disabled)
- **Felix**: Disabled and deleted; was temporary for managerial access
- **Admin Tool**: Disabled and deleted; was for testing installation processes without global admin requirement

**Users Kept Active**:
- **Tomasz Kowalski (TTK)**: Development/integration team member; now has read/write access with proper licenses
- **Michal Rosikiewicz**: Development/integration team member; now has read/write access
- **Luke Grabowski (Lukas Lab)**: New developer user created; assigned basic licenses
- **Najeeb**: CRM consultant from external firm; disabled but kept (can be re-enabled when needed)
- **Gustav Estlund (Gus Lab)**: Consultant from CRM firm; main architect of the Dynamics solution design; disabled but kept
- **Shraddha (Sravidya)**: On parental leave; kept active as still an employee

**Unused Test Accounts**:
- **Test Not Admin**: Disabled; was used for investigating whether global admin requirement could be bypassed (it cannot)

### The Global Admin Requirement for Dynamics OAuth Setup

[Erik Andersson]: Apsis One cannot overcome the Microsoft requirement that **a global administrator must authorize the Apsis One OAuth application to the organization**. This is a hard constraint because OAuth application authorization to a tenant is an extremely sensitive operation.

> "It is a hard requirement on dynamics that whenever you add this like oaf user to the organisation, it has to be a global admin... the whole flow is dependent on a oaf user being added to the organisation."

This is why customers often struggle with implementation—they must provide a global administrator or high-level IT person to complete the initial setup, which is cumbersome in large organizations.

### The Generic Connector Advantage

[Erik Andersson]: This limitation applies only to the **legacy Dynamics connector**. The **generic connector** avoids this problem entirely because:

- Apsis One connects to an intermediate service (Site Shop)
- Apsis One never directly touches the Dynamics environment
- The intermediate service handles all Dynamics authentication and integration
- A global admin is only needed at the intermediate service level, not at the customer's Dynamics level

> "Apsis is will never touch dynamics in that flow, we only communicate with the intermediate service and the intermediate service handles everything with dynamics."

---

## Environment Architecture and Licensing Constraints

### Staging Environment (Integration Development)

The Dynamics instance used for development and testing is the **Integration Development CRM** environment. This is a shared environment for:
- New feature development
- Testing of integration flows
- User onboarding for the development team
- CRM consultant involvement (Najeeb and Gustav from the external CRM firm)

### Sales Enterprise Edition Licensing Limitation

The most constrained resource is the **Sales Enterprise Edition** license. Only **three licenses are available**, and they are currently assigned to:
- Erik Andersson (being handed over)
- Tomasz Kowalski
- Michal Rosikiewicz

This license tier provides access to leads and marketing lists in addition to basic contacts and other records.

[Erik Andersson]: "We have only three available" for the Sales Enterprise Edition, making it the limiting factor for adding new team members with full access.

---

## Key Takeaways

1. **Event Listener Registration Refactoring**: The outbound manager must request CRM system activity creation first, then register listeners only for activities the CRM actually created. This prevents sending events for non-existent activities to customers like E-deal who only want specific activity types.

2. **Administrative vs. Read Write Access**: In Microsoft Dynamics/Entra ID, Administrative mode allows management but not data access. Read Write mode is required for actual data interaction. This distinction is not immediately obvious and caused significant confusion.

3. **Licensing is a Prerequisite**: Users cannot be switched from Administrative to Read Write mode without appropriate Microsoft licenses assigned first. Attempting this without licenses reverts the user to Administrative mode.

4. **OAuth Setup Requires Global Admin**: External users cannot complete the initial Apsis One OAuth authorization because they lack global admin rights in the customer's organization. Use dedicated internal users for setup.

5. **Secret Rotation is Non-Destructive**: Client secrets can be rotated without requiring customer reinstallation, as long as the Application ID remains unchanged. Only changing the Application ID requires customer action.

6. **Application Ownership Risk**: Currently, the production OAuth applications are owned by a disabled user account. This is a disaster recovery risk, though it's been mitigated by saving the credentials for that account. Future best practice: ensure multiple active global administrators are owners.

7. **Generic Connector Eliminates Global Admin Requirement**: The generic connector using an intermediate service (Site Shop) avoids the global admin requirement entirely, making it a better option for customers than the legacy Dynamics connector for this reason.

8. **Sales Enterprise Edition is Constrained**: Only three licenses available; careful allocation needed as the team grows.

---

## Unresolved Questions and Action Items

### Pending Investigation
- **Hassan Action Adapter**: Unknown purpose; endpoints suggest local testing; needs confirmation before deletion
- **Web API Test Application**: Unknown purpose; has expired secrets but no clear usage; needs determination of whether it's still needed
- **Apsis Lead Integration User**: Created in 2015; likely for the Pro or legacy lead system; unclear if still in use; recommend not deleting until confirmed

### Follow-up Actions
- **Monday or next week** [Erik]: Check application logs in Apsis to understand why Michal's initial integration connection failed
- **Monday/Tuesday handover sessions** [Erik]: Evaluate team's theoretical knowledge of Dynamics setup and determine if additional guides or sessions needed for handover quality goals
- **Monday morning** [Erik]: Access and manage Nicholas's disabled account to address OAuth application ownership issues; potentially add global admins as co-owners
- **Cost-benefit review**: Determine if Hassan Action Adapter and other mystery applications should be officially documented or deleted
- **External user workflow documentation**: Formalize the process that external users cannot perform initial OAuth setup and must use internal accounts
- **Disaster recovery plan**: Document the procedure for rotating secrets and handling lost application ownership

### User Access Backlog
- **Lucas Grabowski**: New user created with temporary password; he will need to change password on first login; needs to be notified
- **Najeeb and Gustav**: Both disabled but needed for future work; enable when their involvement is required

---

## Technical Details Preserved

**File Paths / Configuration**:
- Power Platform Admin Center: Admin Power Platform navigation → Manage Environments
- Microsoft 365 Admin Center: admin.microsoft.com → Users → Active Users → Licenses and Apps
- Entra ID / Azure AD: portal.azure.com → App Registrations → All Applications

**OAuth Application IDs** (not specific values, but referenced):
- Apsis One Sandbox 2 (http://localhost:4200)
- Apsis One Staging 2 (https://cloud-integration-stage.apsis-ip.com/...)
- Apsis One Beta
- Apsis One EU Prod (must stay)
- Apsis One APEC Beta (environment no longer exists)

**License Terminology**:
- Basic User
- Sales Team Member
- Sales Enterprise Edition (3 available)
- Power Automate Free
- Power Apps Premium
- Microsoft Fabric (trial)
- Dynamics Customer Service

**URL Patterns**:
- Integration Development environment: [environment-url].dynamics.com
- OAuth redirect: https://cloud-integration-stage.apsis-ip.com/[path]