---
source_file: Erik - Microsoft Dynamics.txt
domain: Apsis One Integrations
topics: [Microsoft Dynamics 365 CRM Integration, Legacy vs. New Connector Architecture, Azure/Entra Setup and User Management, OAuth Application Registration, Dynamics Environment Creation, Plugin/Solution Installation, Real-time Sync Mechanisms]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Microsoft Dynamics 365 CRM, Azure AD/Entra, Admin Power Platform, Power Apps, OAuth Applications, Dynamics Solutions/Plugins, Security Roles, Application Users, One API Integration, Web Hooks]
session_type: knowledge-transfer
subdomains: [Architecture, Microsoft Dynamics Integration]
---

## Session Overview

This knowledge transfer session covered the **legacy Microsoft Dynamics 365 CRM connector** within the Apsis One integration platform. Erik Andersson walked the team through the infrastructure setup (Azure/Entra organizations, OAuth applications, admin portals), explained the distinction between the old Dynamics connector and the newer Sideshop-managed generic connector (with a recommendation to migrate new customers to the latter), demonstrated environment creation in the Apsis development tenant, and introduced the plugin architecture that enables real-time synchronization and permissions management. The session ended with the beginning of a plugin installation process on a newly created test environment.

---

## Legacy vs. New Dynamics Connector Architecture

### Strategic Direction: Migration to New Connector

The integration platform supports two Microsoft Dynamics connectors:

1. **Legacy Microsoft Dynamics 365 CRM connector** — the original implementation, discussed in this session
2. **Dynamics by Sideshop** — the new connector, managed by Sideshop and implementing the generic connector framework

> For all new customers, the recommendation is to go down the Sideshop route because it's more future-proof. Everything we add to the generic connector has a strong possibility that Sideshop will add to their connector, whereas we will not add any new functionality to the old connectors.

[Erik Andersson]

**Why this matters:**
- The legacy connector is in maintenance mode only
- New feature development goes into the generic connector, which Sideshop adopts
- Existing legacy customers should be migrated away from this connector over time

---

## Azure Organization and Tenant Structure

### Two Key Organizations

The integration relies on two separate Microsoft Entra (Azure AD) organizations:

#### 1. **Apsis Lab (apsisut.onmicrosoft.com)** — Development Organization
- Created and managed by CRM Consultana (our development partner)
- Purpose: Internal development, testing, and QA
- Permissions: Apsis team can create new instances, manage licenses, perform unrestricted testing
- Contains multiple environments:
  - **Integrations QA instance** — Used by QA team for manual testing and automated test suites
  - **Pro Dev** — Development environment for the Pro integration (managed separately)
  - Additional test/development instances

#### 2. **Apsis International AB (apsis.onmicrosoft.com)** — Credential Organization
- Production credential storage and OAuth client management
- Contains all OAuth application registrations used for customer deployments
- Stores secrets for authentication flows
- **Critical:** This is where customer OAuth application IDs and secrets are registered and stored

[Erik Andersson]: > These are the two organizations that are very important to remember. One is the Apsis development environment where we can create new instances. The second organization is where we have the OAuth clients and their secrets stored.

### Key Distinction

- **Admin Power Platform** (admin.powerplatform.microsoft.com) — Manage Dynamics instances and environments
- **Microsoft Entra** (entra.microsoft.com, formerly Azure AD) — Manage users and organizational members for each organization

---

## User Access and Provisioning Strategy

### External User Invitation (Recommended)

When adding team members to Azure organizations, **always use "Invite External User"** rather than creating a new user:

**Why external user invitation is superior:**
1. Reuses existing SSO flow from Microsoft
2. Leverages Apsis FC (federated credential) authentication
3. When a user is centrally disabled at FC (company-wide), access to the tenants is automatically revoked
4. No need to manually manage separate usernames and passwords in Azure
5. Maintains security alignment with corporate identity management

**Process:**
- Select "Invite external user" during user addition
- Set the member level to "Member" (not "Guest") if they need to modify resources
- Guests have read-only access; Members can modify depending on their assigned role

[Erik Andersson]: > If you invite a user, you will create a completely new user for your organization that is disconnected from the FC world. And you would need to add a username and password. If you invite an external user and reuse their FC user, then when that user is centrally disabled at FC, access to your tenants is also removed. There are only advantages to doing that.

### Global Administrator Role Requirement

Only **Global Administrators** can approve OAuth application authorization requests during customer setup. This is a common friction point:

> The reason why you have to be a global admin is because only global admins can approve the authorization of the app connecting to your organization. If you have done this once, then you don't need to be an admin to install Dynamics again, but the first time this happens you need to have a global administrator approving it. This tends to be the biggest challenge for customers — getting their IT people to want to or get time to do this.

[Erik Andersson]

---

## OAuth Application Registration and Management

### Critical Application: Apsis One EU Prod

Located in **Apsis International AB > App Registrations > All Applications**

The **single most important application** is **Apsis One EU Prod** (also referenced as "Apsis 1 EU prod"):
- Used to authenticate and access **all customer Dynamics instances** in production
- One application ID and secret pair is shared across every customer deployment
- Stores application ID (visible in registrations) and secret (stored in AVS Secrets Manager)

**Risk Profile:**
If this application or its secret is lost or compromised:
- Every Dynamics customer would lose connectivity
- Complete re-installation would be required unless the secret is recovered or rotated
- This is described as being "completely stuck out of luck"

[Erik Andersson]: > If we were to lose access to these ones and there would be an incident and you would need to essentially redeploy integration and you did not have access to the secret, then every Dynamics customer would need to reinstall their integration.

### Other Application Registrations

The organization also maintains environment-specific applications:

| Application | Purpose | Status/Notes |
|---|---|---|
| Apsis One EU Prod | Production EU customers | CRITICAL — in active use |
| Apsis One APAC Prod | Production APAC customers | In active use |
| Apsis 1 Staging 2 | Staging environment | Active (older version expired) |
| Apsis 1 Beta | Beta/testing | For non-production validation |
| Various expired/unused apps | Historical registrations | Subject to cleanup |

[Erik Andersson]: > There are a lot of them. Some are expired, some are old ones which I have not really dared touch. The only ones that you will ever have to worry about is Apsis Staging 2, Apsis 1 Beta, Apsis 1 EU Prod, and Apsis 1 APAC Prod. I'll do some cleanup of the others, but I'll make a list of the ones actually in use for our system.

### Secret Management and Rotation

**Where secrets are stored:**
- Azure App Registration > Certificates & Secrets > Client Secrets
- Secrets are set to never expire
- Production secret is stored in **AVS Secrets Manager**
- Integration Manager loads the secret at boot time

**What happens when a secret is rotated:**
- Only Apsis services need to be updated (they fetch the new secret from AVS)
- **Customers do NOT need to reinstall** if only the secret is rotated
- The OAuth application **ID** remains unchanged; customers approved access to that ID
- Customers WOULD need to re-approve if the application ID itself is changed

[Erik Andersson]: > They don't need to reinstall if you just change the secret. We request permission for our application to access them using the specific application ID. The secret is for us when we authenticate to Microsoft. But if you were to create a completely new application, then this new application would have to be approved by the customer again.

### How Secrets Are Used

1. **Integration Manager boots** and loads the client secret from AVS Secrets Manager
2. **OAuth flow to Azure** using the application ID and secret to generate an access token
3. **Access token used to communicate** with the customer's Dynamics instance
4. Customer must have granted access approval to the application ID during setup

### Real-World Issue: Disabled Applications

[Erik Andersson]: > We just had a problem with a customer where on installation they approved us to log in to their instance with that application, and then later someone in their IT had said, 'What is this? I'll disable it.' And then we lost complete access to their environment until they re-enabled it.

---

## Front-End Configuration and Client IDs

The front-end repository contains global configuration that specifies which OAuth application ID to use for different environments.

**File/Location:** Global configuration setup in front-end repository  
**Configuration Key:** Dynamics client ID

The front-end uses these IDs during customer setup:
- **Production EU:** One application ID (visible in app registrations)
- **Staging:** Different application ID (e.g., B6605...)
- **Beta:** Separate application ID

[Erik Andersson]: > If we search in the front-end repository for the client ID configuration, you can see that the Dynamics client ID for different environments is set in the global configuration.

---

## Dynamics Environment Creation

### Step-by-Step Environment Setup

**Access Point:** Admin Power Platform (admin.powerplatform.microsoft.com)

**Navigation:**
- Switch to the correct organization (Apsis Lab for development)
- Menu > Manage > Environments
- Click "New Environment"

**Configuration Options:**

| Field | Value | Purpose |
|---|---|---|
| Environment Name | `integration-development` (example) | Identifies the instance |
| Region | Europe | Data residency |
| Add Dataverse | Yes | Creates database |
| Dataverse Language | English | Default language for records |
| Security Group | Open Access | Anyone in org can access (authentication still required) |
| Enable Dynamics 365 Apps | **Yes** | CRITICAL for contact/marketing list handling |

**Why Enable Dynamics 365 Apps:**
Without this setting:
- Only standard Contacts can be managed
- Marketing Lists are not available
- Email list syncing from Dynamics is not possible

[Erik Andersson]: > You want to enable the Dynamics 360 apps because this enables more functionality and more precisely it enables Marketing lists and lead handling inside Dynamics. If you don't do this then you can only handle your regular contacts, but the integration supports Marketing lists inside Dynamics like the e-mail list syncing. If you don't enable this then we can't access them.

### Environment Provisioning Process

After saving, the environment enters **"Preparing"** state:
- Microsoft spins up compute resources in the background
- Creates and registers a database
- Registers DNS records

**Example DNS Result:** `integrations-development.crm4.dynamics.com`

Provisioning typically takes 1–2 minutes.

### First Access

Once ready, access the instance at the provided URL and authenticate with an FC user account (e.g., UTV account = "Apsis Development" in Swedish).

---

## Advanced Settings and Solutions/Plugins

### Accessing Advanced Settings

Within a Dynamics instance:
- Click the **Settings (gear) icon** (top right)
- Select **Advanced Settings**
- Left navigation menu appears

### Solutions: Dynamics Terminology

A **Solution** in Dynamics is equivalent to a **plugin** or **package**. It contains:
- Custom code and configurations
- Security roles and permissions
- Custom database tables
- Event handlers and automations

### Make Power Apps Portal

From Advanced Settings > Solutions, you're redirected to **Power Apps** (make.powerapps.com):
- This is Microsoft's low-code automation platform
- Enables custom workflows, triggers, and integrations
- Contains the **Import Solution** button

[Erik Andersson]: > Power Apps is a tool to have automations running inside the Dynamics instances or your organization. You can have jobs that trigger once every something and then access some Azure APIs. But the only thing that is important for us is that you have a button called import solution.

---

## Apsis Dynamics Plugin Architecture

### Overview: Two Plugins

The integration provides two managed solutions (plugins):

1. **Apsis Main Plugin** ("Apsis [something] Managed")
2. Supporting plugin (additional configuration)

### Main Plugin: Core Responsibilities

#### 1. Security Role Creation

Creates a custom security role with permissions for all operations the integration requires:
- **Contact-level permissions:** Read, Create, Update, Delete contacts
- **Organizational-level access:** Operations apply across the entire organization, not just user-assigned records

[Erik Andersson]: > This security role will have every permission available for contacts like reading contacts, updating contacts, and also have the permission to do this on an organizational level.

#### 2. Application User and OAuth Integration

During customer installation, the process:
1. Creates an **application user** within the customer's Dynamics instance
2. Attaches the Apsis OAuth application (registered in Apsis International AB)
3. Connects the OAuth app with the application user via an access token flow
4. Assigns the custom security role to the application user

**Result:** Apsis can authenticate to the customer's instance using an access token generated from the OAuth secret, with full permissions for contacts and related operations.

#### 3. Custom Database Tables

The plugin creates custom tables used for behind-the-scenes operations:
- Store OAuth client credentials generated during setup
- Store configuration and state information
- Support real-time synchronization mechanisms

#### 4. Real-Time Sync Infrastructure (Web Hook System)

Dynamics does not natively support webhooks for real-time change notifications. The plugin implements a workaround:

**Problem:** Dynamics doesn't natively expose change events in real-time  
**Solution:** The plugin creates a background process that:
- Listens to events inside Dynamics
- Filters for contact-related events
- Sends contact data to One API in real-time
- Uses the OAuth client credentials stored in the custom table to authenticate to One API

[Erik Andersson]: > Dynamics does not natively support web hooks. So we would have to have a flow where we say we only download good profiles from Dynamics at will like the full sync. The full sync makes a request to Dynamics: 'Please give me all of your contacts.' The background process that this package creates implements a web hook system. Essentially it listens to events inside dynamics and then checks was this event for a contact. If it is for a contact then it will contact One API with the profile data.

### Why the Plugin is Non-Negotiable

[Erik Andersson]: > If you don't have this plugin like nothing will work. You will fail to uninstall and even if you had managed to install, we would not have permission to read any contacts and no data would be sent to Apsis, so it would be completely useless.

**Without the plugin:**
- No security role = no permissions = cannot read/write contacts
- No OAuth client registration in custom tables = real-time sync fails
- No custom tables = web hook system cannot function
- Complete integration failure

### Plugin Installation

**Process:**
1. Access the customer's Dynamics instance
2. Navigate to Advanced Settings > Solutions
3. Click **Import Solution**
4. Upload the solution file (provided by sales team during pre-installation)
5. Import process runs (can take several minutes)

This session ended at the start of the plugin import process.

---

## Development vs. Customer Environments

### Critical Boundary

**Development environments (shown in this session):**
- Run on Apsis Lab organization (apsisut.onmicrosoft.com)
- Team has full administrative access
- Used for testing plugin updates, environment creation, configuration testing

**Customer environments:**
- Run on customer's own Azure/Entra organization (not Apsis-controlled)
- Apsis never has access to customer Azure admin portals
- Customers manually create their own Dynamics instances
- Customers install the provided solution file themselves or request Apsis assistance
- Customers control all permissions and access

[Erik Andersson]: > You will never be able to touch the customer's corresponding pages. The customer will have their own page with their environments. You will never touch this page for the customer. We can have hope that it looks the same for the customers, and it should be the same if there are no variances of Dynamics.

### Platform Assumptions

**On-Premises Dynamics:**
- Not officially supported
- Requires internet connectivity to Azure for webhooks and OAuth flows
- Completely locked-down on-premises installations (no internet) are incompatible with the integration

[Erik Andersson]: > We use resources inside Azure. We use webhooks in Azure. We have the OAuth login flow in Azure. If the customer has a completely locked down on-premises instance which cannot talk with the Internet, we don't support that. The official standpoint from Apsis is we don't support on-premises Dynamics.

---

## Key Takeaways

1. **Two Organizations, Two Purposes:**
   - **Apsis Lab (apsisut.onmicrosoft.com):** Internal development and testing
   - **Apsis International AB (apsis.onmicrosoft.com):** Credential and OAuth storage for customer deployments

2. **Always Invite External Users:** Use the external user invitation flow to leverage centralized FC authentication and automatic revocation on termination.

3. **One Critical OAuth App:** The Apsis One EU Prod application is shared across all production customers. Loss of this credential means customer re-installation. Protect the secret in AVS Secrets Manager.

4. **Plugin is Non-Negotiable:** The Apsis solution must be installed in every customer Dynamics instance. It provides permissions, real-time sync, and OAuth credential storage. Without it, the integration cannot function.

5. **Real-Time Sync via Web Hook Emulation:** Dynamics lacks native webhooks, so the plugin implements a background process that listens for events and forwards them to One API.

6. **Enable Dynamics 365 Apps:** This is essential for marketing list support. Always enable during environment creation.

7. **Development != Customer:** The development environment shown is on Apsis infrastructure. Customers manage their own Dynamics instances; Apsis only provides the solution file.

8. **Sideshop Migration Path:** New customers should use the Sideshop-managed generic connector, not the legacy connector.

---

## Unresolved Questions and Action Items

### Access Issues to Resolve (Post-Session)

- **Lukasz Grabowski:** Unable to access the Apsis International AB organization or view app registrations (insufficient privileges error). Erik to provide individual access or alternative setup.
- **Michal Rosikiewicz & Tomasz Kowalski:** Successfully accessed Entra and Admin Power Platform, but may need role elevation for full admin functions.

**Action Items:**
1. **Erik Andersson** — Add Lukasz as member/admin to Apsis International AB (app registrations organization)
2. **Erik Andersson** — Provide admin role assignments to all team members as needed
3. **Erik Andersson** — Clean up unused/expired OAuth applications from the registry (Apsis lab identified several unused entries)
4. **Erik Andersson** — Create documentation list of in-use vs. obsolete OAuth applications
5. **Erik Andersson** — Remove obsolete global admins from Entra (identified during session cleanup)

### Continuation

The knowledge transfer will resume in a follow-up session to cover:
- Plugin import and configuration validation
- Real-time sync flow testing
- Customer installation process walkthrough
- Troubleshooting and debugging scenarios