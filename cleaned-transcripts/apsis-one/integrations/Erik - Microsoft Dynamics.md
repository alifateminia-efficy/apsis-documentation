---
source_file: Erik - Microsoft Dynamics.txt
domain: Apsis One Integrations
topics: [Microsoft Dynamics 365 CRM Setup and Configuration, OAuth Application Management, Azure/Entra User Administration, Dynamics Environment Creation, Plugin Installation and Security Roles, Real-time Sync Architecture]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Microsoft Dynamics 365 CRM, Azure/Entra AD, Power Platform Admin, Power Apps, OAuth Applications, Dynamics Solutions/Plugins, Application Users, Security Roles, Web Hook System, One API, Integration Manager]
session_type: knowledge-transfer
subdomains: [Architecture, Microsoft Dynamics, Different Types of Connectors]
---

## Session Overview

This session covered the legacy Microsoft Dynamics 365 CRM connector for Apsis One Integrations, with particular focus on setup, administration, and the underlying architecture. Erik walked through the distinction between the legacy connector and the newer generic connector approach, demonstrated the critical Azure/Entra administration pages for managing OAuth applications and user access, and explained the plugin-based architecture that enables real-time synchronization between Dynamics and Apsis. The session emphasized operational risk management—specifically around the single shared OAuth application used across all production Dynamics customers and the importance of maintaining admin access to Azure tenants.

---

## Connector Strategy: Legacy vs. New Approach

### Overview of Two Dynamics Connectors

There are two Microsoft Dynamics connectors available in Apsis One:

1. **Microsoft Dynamics 365 CRM** (legacy) — The original connector, discussed in detail during this session
2. **Dynamics by Sideshop** (new) — A newer implementation managed by Sideshop that uses the **generic connector** pattern

[Erik Andersson]: > "The recommendation is for all new customers to go down this route because this will this is more future proof for new features. It's like everything we add for the generic connector. There is a strong possibility that Sideshop will add this to this connector, whereas we will not add any new functionality to the old connectors."

### Strategic Rationale for Migration

The legacy connector is no longer receiving active feature development. New features are added to the generic connector framework, which Sideshop then backports to the Sideshop-managed Dynamics connector. This creates a strong business and technical case for migrating existing customers away from the legacy connector.

Additionally, there are **operational/security concerns with the legacy approach**: The legacy connector requires Apsis to own and manage two separate Azure tenants (discussed below), creating administrative overhead. When staff leave Apsis, their accounts must be manually removed from these tenants. [Erik Andersson]: > "Right now we we own essentially we own the administration of these two tenants, so whenever someone quits at Apsis, we need to remove them from the tenants here. Like granted, their FSC user will be disabled so they can actually log in, but if someone were to do an audit and look at it, it will not look very nice."

---

## Azure/Entra Infrastructure and Administration

### Two Critical Azure Tenants

The legacy Dynamics connector relies on two separate Azure/Entra organizations:

#### 1. **Apsisut Development (apsisut@onmicrosoft.com)**
- **Purpose**: Development environment for Apsis internal use
- **Owner**: CRM Consultana (the partner who helped develop the solution)
- **Permissions**: Apsis has full administrative control and can create new instances
- **Use case**: Internal QA, development, and testing of the Dynamics integration

#### 2. **Apsis Production (apsis@onmicrosoft.com)**
- **Purpose**: Houses the OAuth client credentials used in all customer installations
- **Critical**: Contains the OAuth application IDs and secrets that must be protected
- **Access requirement**: Developers need access to view application registrations and, if needed, regenerate secrets

[Erik Andersson]: > "This organization we have the OAuth clients that I've previously mentioned. So the apsis@onmicrosoft.com is the environment development environment. Apsis@onmicrosoft.com. Here we have the OAuth clients and their secrets stored."

### Key Azure/Entra Pages

#### **admin.powerplatform.microsoft.com**
- Used to manage **environments** (instances) and tenants
- Where you create new Dynamics environments for testing or new deployments
- Different from Entra — this is specifically for Power Platform/Dynamics instance management

#### **entra.microsoft.com** (formerly Azure AD portal)
- Used to manage **users** for an organization
- Where you add/remove users and assign roles like Global Administrator
- Separate from Power Platform admin page

[Erik Andersson]: > "Here on Admin power platform you manage your micro like your instances and the environments here on the Entra.microsoft.com. This is the old Azure AD. So here you manage the users for your organization. So entra.onmicrosoft.com is for your organization, the admin.powerplatform.microsoft is for your environments or your your tenants."

### User Invitation Best Practice: External Users vs. New Users

When adding team members to Azure organizations, **always invite as external users** rather than creating new local users.

**Why external user invitations are superior:**

- External user invitations leverage Microsoft's SSO flow and tie the Azure account to the person's FC.com SSO identity
- When an FC.com user account is disabled centrally (e.g., upon departure), access to the Azure tenants is **automatically revoked**
- New local user accounts require manual username/password management and must be manually removed when users leave
- External user accounts can be invited as **Members** (with modification rights) or **Guests** (read-only)

[Erik Andersson]: > "If you invite an external user, now you're reusing the SSO flow from Microsoft. If in the same, this is essentially when we add someone to the trust relationship to our Azure accounts. If the root user in Azure is disabled, then there will be no access to your account. And it's the same thing here if you invite an external user and you invite their FC user then when that user is centrally disabled at FC then the access to your tenants is also removed. So there are only advantages of doing that."

---

## OAuth Application Management: The Production Bottleneck

### Single Shared Application Across All Customers

The legacy Dynamics connector uses a **single OAuth application** for all production customer installations. This is a critical architectural decision with significant operational implications.

**The production OAuth applications** (stored in the Apsis production tenant at apsis@onmicrosoft.com under App Registrations):

- **Apsis 1 EU Prod** — Used to access all EU production customer environments
- **Apsis 1 APAC Prod** — Used to access all APAC production customer environments
- **Apsis 1 Staging** — Used for staging environment testing
- **Apsis 1 Beta** — Used for beta customer testing

Each of these applications has:
- A **Client ID** (application identifier) — referenced in configuration and visible to customers
- A **Client Secret** — stored in AVS Secrets Manager and loaded by the Integration Manager service on startup

### Critical Risk: Single Point of Failure

[Erik Andersson]: > "The Apsis one prod EU like this is the single most important application of them all because we use this one to access all customer environments. It might not be the best approach because essentially it's the comparison of having one API secret for every customer account, but that is how it was set up for us."

And: > "If this one were to disappear for whatever reason, then that's a very annoying situation, to say the least."

**Scenario**: If the production OAuth application were accidentally deleted from Azure:
- Every single Dynamics customer in production would lose connectivity to Apsis
- A complete **redeployment would be required** for all customers
- This is why Erik emphasized the importance of team members having access to view and potentially regenerate these credentials

### Where the Client ID Is Configured

The **Client ID** is used during customer setup and is configured in the Apsis frontend application. It's not a secret and is visible in configuration files.

- Frontend repository contains global configuration for product environments
- The Client ID value (e.g., starts with `b6605...`) is stored in environment-specific configuration for setup flow
- Different Client IDs are used for different environments (staging, beta, prod EU, prod APAC)

### Managing Secrets: Creation and Rotation

**Where to manage secrets in Azure:**
1. Navigate to **entra.microsoft.com** → Switch to the production organization
2. Go to **App Registrations** → **All Applications**
3. Select the desired OAuth application (e.g., "Apsis 1 EU Prod")
4. Go to **Certificates & secrets**
5. Create a new secret or view existing ones

**Secret Configuration in Apsis:**
- Secrets are stored in **AVS Secrets Manager** (Apsis's centralized secrets management)
- The Integration Manager service loads these secrets on startup
- Secrets are used in the OAuth flow to request access tokens from Azure
- The **never-expire secret** is the preferred approach (though not a best practice security-wise)

### Secret Rotation and Customer Impact

[Michal Rosikiewicz asked]: "Can you also let us know where to update the secret?"

And: "So whenever customer with old dynamics will request it, we'll have to create a new application for them here create no."

[Erik Andersson]: > "No, no, no. This is one application used by every customer."

**Important distinction:**
- If you **rotate/regenerate the existing secret** in Azure, only Apsis needs to update the secret value in AVS Secrets Manager. Customers do **NOT** need to do anything.
- If you **create a completely new OAuth application** (different Client ID), then the new application must be approved by all customers in their Dynamics instances, requiring reinstallation.

### Real-World Incident: Disabled Applications

[Erik Andersson]: > "We just had a problem with a customer where they on installation had approved us to log in on their instance with that application and then later like someone in their IT had said like what is this, I'll disable it. And then we lost complete access to their environment until they re-enabled it."

This happened because the customer's IT department disabled the OAuth application in their Dynamics instance without understanding its purpose. The solution requires customer action to re-enable the application. This is a warning sign that good customer documentation and communication about the purpose of the OAuth app is essential.

---

## Development vs. Production Environments

### Development Instance Structure

The Apsis lab (development) organization has multiple environments for testing:

- **Integrations QA** — Used by QA team for manual testing and automated test suites. Should not be modified by developers.
- **Pro Dev** — Development environment used by Pro team (internal Apsis team)
- Other environments may exist for specific projects

### Creating a Test Environment

To create a fresh Dynamics environment for testing the installation/uninstallation process:

1. Go to **admin.powerplatform.microsoft.com**
2. In the left menu, select **Manage** → **Environments**
3. Click to create a new environment
4. Fill in the details:
   - **Name**: e.g., "Integrations Development"
   - **Region**: e.g., Europe (should match customer location)
   - **Add Dataverse**: Yes (this creates the database)
   - **Security Group**: Open Access (still requires authentication)
   - **Enable Dynamics 365 apps**: **CRITICAL** — Enable this
5. Click Save and wait for the environment to be created (1-2 minutes)

#### Why Enable Dynamics 365 Apps?

[Erik Andersson]: > "We want to enable the Dynamics 360 apps because this enables more functionality and more precisely it enables Marketing lists and lead handling inside Dynamics. If you don't do this then you can only handle your regular contacts, but the integration supports Marketing lists inside Dynamics like the e-mail list syncing. If you don't enable this then we can't access them like there will be nothing to to be shown there."

The Dynamics 365 apps enable functionality for:
- Marketing lists (e-mail list management)
- Lead handling
- Advanced contact features

Without these apps, the integration can only sync regular contacts, not marketing lists or leads.

### Instance URL Pattern

Once created, instances follow a predictable URL pattern:

```
https://{environment-name}.crm4.dynamics.com/
```

For example: `integrations-development.crm4.dynamics.com`

---

## Plugin/Solution Architecture and Installation

### Plugins as "Solutions" in Dynamics

In Dynamics terminology, a **plugin** is called a **solution**. The Apsis connector requires the customer to import a solution file during installation.

**Where solutions are imported:**
1. Customer logs into their Dynamics instance
2. Clicks the **settings/cog icon** → **Advanced Settings**
3. This opens a Power Apps page (make.powerapps.com)
4. On the left menu, select **Solutions**
5. Click **Import Solution**
6. Provide the solution file (provided by the sales team before customer setup)

### The Apsis Plugin Components

There are **two plugins** that make up the Apsis Dynamics integration:

#### 1. **Apsis [something] Managed** (Main Plugin)

**Primary responsibilities:**
- Creates a **security role** inside the customer's Dynamics instance
- Defines all permissions needed for the integration to function (read contacts, update contacts, etc.) on an organizational level
- Creates **custom database tables** used for background operations
- Implements the **web hook system** for real-time synchronization

**Why it's critical:**
Without this plugin, the integration cannot function because:
- No security permissions exist for the application user
- No custom tables exist for real-time sync operations
- No web hook listeners are registered

#### 2. **[Second plugin name not fully detailed in transcript]**

Referenced but details not fully captured in this session.

### Real-Time Sync Architecture

The plugin implements a **custom web hook system** because Dynamics does not natively support webhooks:

1. **Full Sync Mode** (on-demand):
   - Apsis makes a request to Dynamics: "Give me all your contacts"
   - Dynamics returns all contact records
   - Standard REST API call

2. **Real-Time Sync Mode** (event-driven):
   - The plugin's background process listens for **events** inside Dynamics
   - When a contact is created, updated, or deleted, an event fires
   - The background process checks: "Is this event related to a contact?"
   - If yes, it **sends real-time notification to Apsis** with the profile data
   - This notification uses the **One API client credentials** (see below)

### Application User and Security Role Setup

During the Apsis installation process, the following occurs in the customer's Dynamics instance:

1. **Create an Application User**:
   - A service account that Apsis can authenticate as
   - This user is bound to the OAuth application (Apsis 1 EU Prod, etc.)

2. **Attach the Security Role**:
   - The security role created by the plugin is attached to this application user
   - The role grants permissions: read contacts, update contacts, read leads, etc.
   - Permissions are set at the **organizational level** (across the entire instance)

3. **Generate One API Credentials**:
   - A **One API client ID and client secret** are generated
   - These credentials are sent to Microsoft Dynamics
   - Stored in a **custom table** created by the plugin
   - Used by the background process when sending real-time notifications

[Erik Andersson]: > "So essentially we can access their instance by using an access token generated by our Azure app. Then we attach the security role that we have created with the plugin and that means that we now have a application user inside their tenants that we have can generate access token for and the permissions that this user requires is generated and set up by the plug-in."

### Why the Global Administrator Approval is Required

During setup, the customer must grant the OAuth application permission to access their Dynamics instance. Only a **Global Administrator** in the customer's Entra/Azure directory can approve this authorization.

[Erik Andersson]: > "The reason why you have to be a global admin is because only global admins can approve the authorization of like the app connecting to your organization. If you have done this once, then you don't need to be an admin to install Dynamics, but the first time this happens you need to have a global administrator approving it. This tends to be the biggest challenge for customers to get their IT people to want to or get time to do this for them."

This is often the **biggest bottleneck** in customer onboarding because:
- Requires scheduling time with the customer's IT department
- Customers may be hesitant to approve access by an external application
- Once approved once, reinstallations don't require re-approval

---

## On-Premise Dynamics: Not Supported

[Erik Andersson]: > "We don't support on premise dynamics. At least not if they run like full full on premise because we use resources inside of Azure. I mean like we use web hooks in Azure which we will come to. We have the O off login flow in Azure like if the customer have a like completely locked down on premise which cannot talk with the Internet. Essentially like I mean we we don't support that. So the official standpoint from Apsis is like we we don't support on premise dynamics."

The Apsis Dynamics integration relies on:
- Azure-hosted OAuth flows
- Azure-hosted web hooks for real-time sync
- Internet connectivity to communicate with Apsis services

On-premise Dynamics deployments that cannot communicate with Azure and the internet are **not supported**.

---

## Accessibility and Visibility Across Development and Customer Instances

### Strict Separation Between Development and Production

A key principle that was emphasized:

**Apsis developers NEVER access customer Dynamics instances or their Azure tenants.** 

The development and testing happens entirely within Apsis's own Dynamics environments (in the Apsis lab organization). Developers should assume they will never touch customer-facing Azure pages.

[Lukasz Grabowski]: > "Yeah, nobody is going to give us access to to this, right? So we can only test something, see on our environment and then we, yeah."

[Erik Andersson]: > "Yeah, so this this what we have here like this is our like it is real dynamics or real Azure pages but when it comes to customers, you will never be able, you will never touch their corresponding pages like the customer will have this page with their environments. This is only our development environment that we are looking at now. You will never ever touch this page for the customer."

### UI Parity Assumption

The assumption is that the Dynamics UI is the same across Apsis development instances and customer instances (assuming they're running the same version of Dynamics). However, Microsoft occasionally updates the UI without notice, so developers should be prepared for minor UI differences.

[Erik Andersson]: > "It should be the same and if not, it should not be like it should not be a big difference in terms of versions because like I frequently had the UI in the test instance been updated without me knowing and I had to essentially relearn like where do I look for things."

---

## Administrative User Management

### Cleaning Up Inactive Admins

During the session, the team identified that the Apsis lab Entra organization had Global Administrator accounts for former employees who no longer work at Apsis:
- Zenita Pelko (should remain as she appears to still be associated)
- Gustav Westerlund (must remain — primary contact at CRM Consultana)
- Admin Tool (unclear purpose — should be investigated for removal)

**Action item**: Clean up unused Global Administrator accounts as part of security hygiene.

### Adding New Team Members

When inviting new Apsis team members to the Azure tenants:
1. Invite them as **external users** (not new users)
2. Add their FC.com email address
3. Make them **Members** (not Guests) so they can modify configurations
4. If they need to manage environments, make them **Global Administrators**
5. Only add to organizations they actually need access to

---

## Next Steps (Session Concluded Here)

The session reached a natural break point when the plugin import was initiated in the test Dynamics instance. The following topics were identified as needing continuation in a follow-up session:

1. **Plugin import process** and what happens after importing
2. **Configuration of the integration** within Dynamics after plugin installation
3. **Testing the real-time sync** mechanism
4. **Detailed walkthrough of the solution file** location and contents
5. **Troubleshooting and debugging** common installation issues

---

## Key Takeaways

1. **Legacy vs. New Strategy**: The legacy Microsoft Dynamics connector is no longer receiving feature development. All new customers should be directed to the Sideshop-managed generic connector variant. Existing legacy customers should be prioritized for migration.

2. **Two Azure Tenants**: The legacy connector depends on two Azure organizations:
   - **Apsisut** (development, managed by CRM Consultana)
   - **Apsis** (production, contains critical OAuth credentials)

3. **Single Shared OAuth Application**: All production Dynamics customers use the same OAuth application (Apsis 1 EU Prod or Apsis 1 APAC Prod). If this application is deleted or misconfigured, **all production customers are affected**. Access to these credentials is critical.

4. **External User Invitations**: Always invite new team members as external users tied to their FC.com SSO identity. This ensures automatic access revocation when they leave the company.

5. **Plugin is Mandatory**: The customer must import the Apsis plugin (solution) into their Dynamics instance. Without it:
   - No security permissions exist for the application user
   - No real-time sync capability
   - No contact/lead synchronization

6. **Global Admin Approval Required**: Only once per customer. The customer's Entra Global Administrator must approve the OAuth app. Subsequent installations/reinstalls don't require re-approval.

7. **Web Hook System**: Apsis implements a custom web hook system inside Dynamics because native webhooks aren't available. The plugin's background process listens for contact events and sends real-time notifications to Apsis via One API credentials.

8. **On-Premise Not Supported**: Dynamics instances that cannot reach Azure and the internet are not supported.

9. **No Customer Tenant Access**: Developers never access customer Azure tenants or Dynamics instances. All testing happens in Apsis's own development environments.

10. **Operational Debt**: The need to manage two separate Azure tenants and maintain user accounts there is a burden. This is another reason to migrate legacy customers to the new generic connector approach.

---

## Unresolved Questions and Action Items

1. **Lukasz's Azure Access Issue**: Lukasz encountered "insufficient privileges to view applications" in the Apsis International AB tenant even after being added as a member. This may require explicit admin role assignment or a fresh invitation. Needs troubleshooting in a follow-up session.

2. **Global Administrator Cleanup**: Remove unused Global Admin accounts (specifically "Admin Tool") from Entra. Keep Gustav Westerlund (CRM Consultana) and verify Zenita Pelko's status.

3. **Unused OAuth Application Cleanup**: Erik noted several expired or unused OAuth applications in the app registrations that should be cleaned up. A full list of active vs. deprecated applications should be documented.

4. **Solution File Location**: Erik mentioned he would show where the plugin/solution file is stored in the repository. This documentation was deferred to a follow-up session.

5. **Plugin Import and Configuration**: The actual process of importing the plugin and configuring it for a customer setup needs to be walked through in detail in the next session.

6. **Real-Time Sync Verification**: Procedures for testing and verifying that real-time sync is working after installation need to be covered.