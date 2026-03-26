---
source_file: Erik - Microsoft Dynamics.txt
domain: Apsis One Integrations
topics: [Microsoft Dynamics 365 CRM Setup, Legacy vs. Generic Connectors, OAuth Application Management, Azure Entra Administration, Environment Creation, Plugin Installation, Security Roles, Real-time Synchronization]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Microsoft Dynamics 365 CRM, Azure Entra (formerly Azure AD), Admin Power Platform, Power Apps, OAuth Applications, Apsis One Plugin/Solution, Dataverse, Web Hooks, Application Users, Security Roles]
session_type: knowledge-transfer
subdomains: ["Different Types of Connectors"]
---

## Session Overview

This session covers the Microsoft Dynamics 365 CRM integration for Apsis One, focusing on the legacy connector implementation. Erik walks through the architectural landscape (legacy vs. generic connectors), administration of OAuth applications and Azure tenants, environment setup in the development org, and the critical plugin installation process required for customer deployments. The session includes hands-on navigation of Admin Power Platform, Microsoft Entra, and Power Apps, with emphasis on understanding why certain admin privileges and configurations are necessary.

---

## Microsoft Dynamics Connector Landscape: Legacy vs. Generic Implementation

### Two Connectors Available

Apsis One offers two distinct Microsoft Dynamics connectors:

1. **Legacy Connector (Apsis One - Microsoft Dynamics 365 CRM)**
   - Built in-house by Apsis
   - First connector implemented for the integration platform
   - No longer receiving new features
   - Requires customer ownership of Azure tenants and OAuth application management

2. **Generic Connector (Dynamics by Sideshop)**
   - Managed and maintained by Sideshop
   - Implements the standardized generic connector interface
   - **Recommended for all new customers** because it is more future-proof
   - Any new features added to the generic connector framework will likely be added by Sideshop
   - Cleaner separation of concerns—Sideshop handles their connector, Apsis maintains the platform

[Erik Andersson]: > "The recommendation is for all new customers to go down this route because this will this is more future proof for new features. It's like everything we add for the generic connector. There is a strong possibility that Sideshop will add this to this connector, whereas we will not add any new functionality to the old connectors."

### Strategic Importance of Migration

One critical operational reason to migrate legacy customers to the generic connector is **tenant administration overhead**. With the legacy connector, Apsis owns two Microsoft Azure tenants (`apsisutvekling.onmicrosoft.com` and `apsis.onmicrosoft.com`). When employees leave Apsis, their accounts must be manually removed from these tenants. While their FC.com SSO users are centrally disabled, lingering accounts in the Azure directory create audit and compliance issues.

---

## Azure Tenant and OAuth Application Architecture

### Two Development Organizations

The legacy connector requires management of two distinct Microsoft Azure organizations:

1. **`apsisutvekling.onmicrosoft.com`** (Apsis Development)
   - Created and managed by CRM Consultana (the third-party partner who helped develop this solution)
   - Contains development and QA instances for internal testing
   - Has its own licenses and environment creation permissions
   - **Key environments**:
     - `Integrations QA` — for QA manual testing and automated test suites
     - `pro dev` — development instance for the "pro" team
     - Additional development instances

2. **`apsis.onmicrosoft.com`** (Apsis Production)
   - Contains OAuth client applications and their secrets
   - Contains the stored credentials used during customer installations
   - **Critical data stored here**: OAuth client IDs and secrets for all production connector instances

### Admin Power Platform vs. Microsoft Entra

Two separate Azure administration surfaces manage different concerns:

- **Admin Power Platform** (`admin.powerplatform.microsoft.com`)
  - Manages **environments** and **instances** within an organization
  - Where you create new Dynamics environments, manage capacity, configure settings
  - Organization-specific (e.g., `apsisutvekling` vs. `apsis`)

- **Microsoft Entra** (`entra.microsoft.com`)
  - Manages **users** and **organization-level settings**
  - Where you invite external users, assign roles (Global Administrator, etc.), manage group memberships
  - Also organization-specific

### User Access via SSO

When team members are invited to these organizations:

- **External User Invitation (Recommended)**
  - Reuses Microsoft's SSO flow; users log in with their `@fc.com` FC account
  - When a user's FC account is centrally disabled, access to the Azure tenants is automatically revoked
  - Eliminates the need to manually clean up accounts when people leave
  - Users are made **Members** (not Guests) so they can modify resources according to their assigned roles

- **Regular User Creation (Not Recommended)**
  - Creates a new, disconnected user account in the Azure organization
  - Requires a separate username and password
  - Does not benefit from central SSO management
  - Creates orphaned accounts that must be manually cleaned up

[Erik Andersson]: > "If you invite an external user and you invite their FC user then when that user is centrally disabled at FC then the access to your tenants is also removed. So there are only advantages of doing that."

---

## OAuth Application Management and Secret Handling

### The Critical Single Application for Production

For the legacy connector, **all customer production instances share a single OAuth application**:

- **Application Name**: `Apsis 1 EU prod`
- **Application ID**: (varies by environment; search in front-end code for "Dynamics client ID")
- **Stored Secret**: Managed in AVS Secrets Manager and loaded by the Integration Manager at boot time
- **Scope**: This single application is used to authenticate and access every legacy customer's Dynamics instance

[Erik Andersson]: > "The Apsis one prod EU like this is the single most important application of them all because we use this one to access all customer environments. It might not be the best approach because essentially it's the comparison of having one API secret for every customer account, but that is how it was set up for us."

### Why This Design Has Risk

If this application's secret is lost or the application is deleted:
- **Full Incident**: Every customer running the legacy connector loses integration capability
- **Recovery Requires Secret Regeneration**: A new secret must be created and deployed across the entire service
- **Customers Don't Need to Reinstall** if only the secret changes—the application ID remains the same
- **Customers Must Reinstall** if a completely new application is created (because they must re-approve access)

### Where Secrets Are Updated

**Location**: Microsoft Entra → App Registrations → [Application Name] → Certificates & secrets

**Process**:
1. Navigate to the specific application registration
2. Go to **Certificates & secrets**
3. Create a new secret (can be set to never expire)
4. The new secret value is stored in AVS Secrets Manager
5. Integration Manager loads this secret at startup and uses it for OAuth token generation with Microsoft

[Erik Andersson]: > "The secret is for us. Like we we give, we are we request permission for our application to access them, but like they don't store the secret or anything like the secrets is used by us when we authenticate like we do the O off flow to the Microsoft service."

### Application IDs in Front-End Code

The application ID (not the secret) is embedded in front-end configuration during the Dynamics installation flow because this step is handled by the UI, not the backend:

- **Staging**: Application ID starts with `B666660...` (e.g., `Apsis staging 2`)
- **Beta**: Application ID for `Apsis 1 beta`
- **Production**: Application IDs for `Apsis 1 EU prod` and `Apsis 1 APAC prod`

Each environment (staging, beta, EU prod, APAC prod) has its own application and corresponding secret.

### Cleanup Needed

The app registrations list contains legacy and expired applications that should be removed, but Erik defers this cleanup because some very old applications lack clear documentation of their purpose (created before his tenure at Apsis). A documented list of in-use applications should be maintained for operational clarity.

---

## Customer Authorization and the Global Administrator Requirement

### The Installation Flow Requires Customer Approval

During legacy Dynamics connector setup:

1. The customer is provided with an **application ID** (the Apsis OAuth app)
2. The customer's **Global Administrator** must explicitly approve access for this application to connect to their Dynamics organization
3. Once approved, the application can generate access tokens to read, write, and manage customer data
4. The secret used to generate these tokens is stored only on Apsis's side (in AVS Secrets Manager)

### Why Global Administrator?

Only a **Global Administrator** in the customer's Azure organization can authorize third-party applications to access their tenant. This is a Microsoft security boundary.

[Erik Andersson]: > "The reason why you have to be a global admin is because only global admins can approve the authorization of like the app connecting to your organization. If you have done this once, then you don't need to be an admin to install Dynamics, but the first time this happens you need to have a global administrator approving it. This tends to be the biggest challenge for customers to get their IT people to want to or get time to do this for them."

### Incident: Customer IT Disables the App

Real-world issue encountered: A customer's IT department discovered the Apsis OAuth application and disabled it without informing Apsis or the customer's business team. Result: **Complete loss of access** to the customer's Dynamics environment until IT re-enabled it.

This is a known gotcha—customers must be briefed during onboarding that the integration relies on maintaining the authorization for this specific application.

---

## Environment Creation in Admin Power Platform

### Creating a Fresh Dynamics Instance

When setting up a test or development instance, the process in Admin Power Platform involves:

1. **Navigate to**: Manage → Environments
2. **Click**: Create new environment
3. **Configure**:
   - **Name**: (e.g., "Integrations Development")
   - **Region**: (e.g., Europe)
   - **Dataverse**: Enable (creates a database for the instance)
   - **Security Group**: Set to "Open Access" (still requires authentication but doesn't restrict by group)
   - **Dynamics 365 Apps**: **Must Enable** to support full CRM functionality

### Why Enable Dynamics 365 Apps?

If you do not enable Dynamics 365 apps during environment creation:
- Only standard **Contacts** are available in Dynamics
- **Marketing Lists** are not available
- Apsis One integration cannot sync marketing lists (email lists) from Dynamics
- If this is disabled after creation, it may be difficult to retrofit the functionality

[Erik Andersson]: > "We will want to enable the Dynamics 360 apps because this enables more functionality and more precisely it enables marketing lists and lead handling inside Dynamics. If you don't do this then you can only handle your regular contacts, but the integration supports marketing lists inside Dynamics like the e-mail list syncing."

### Environment Provisioning Timeline

After creation, the environment enters a **Preparing** state for approximately 1-2 minutes while Microsoft:
- Allocates compute resources
- Creates the Dataverse database
- Registers DNS records (e.g., `integrations-development.crm4.dynamics.com`)

### Admin Portal vs. Customer Instances

**Critical distinction**: The Admin Power Platform and Entra experiences shown in this session are **Apsis development and test environments only**. Customers have their own separate Microsoft Azure organizations and will never provide direct access to them. When troubleshooting, you must:
- Test on your own development instances
- Verify configuration on Apsis-owned test environments
- Hope that the UI and behavior are consistent with customer environments (they should be, barring version differences)

---

## The Plugin/Solution Architecture and Installation Process

### What Is a "Solution"?

In Microsoft Dynamics terminology, a **solution** is a packaged bundle of customizations, configurations, and code. For Apsis One:
- The plugin/solution file is created by **CRM Consultana** to Apsis specifications
- It is provided to customers before deployment
- It must be imported into each customer's Dynamics instance via Power Apps

### Two Plugins/Solutions Installed

#### 1. Main Apsis Solution (e.g., "Apsis [something] managed")
**Creates Security Role**:
- Establishes a custom security role within the customer's Dynamics instance
- This role grants all permissions required for Apsis integration:
  - Read contacts (organizational level)
  - Update contacts (organizational level)
  - Create contacts
  - Delete contacts (if needed)
  - Access marketing lists and related entities

**Creates Application User**:
- During installation, Apsis creates an **application user** in the customer's Dynamics instance
- This application user is linked to the Apsis OAuth app (the one stored in `apsis.onmicrosoft.com`)
- The security role created by the plugin is attached to this application user
- Apsis generates an access token using the OAuth app's secret; this token is used to impersonate the application user

**Creates Custom Database Tables**:
- Stores integration metadata and configuration
- Most critically: stores the **Apsis One API client ID and secret** (generated during installation and sent to Dynamics)

**Implements Web Hook System**:
- Dynamics does not natively support webhooks
- The plugin creates a custom background process that:
  - Listens to contact events in Dynamics (create, update, delete)
  - Filters for contact-related changes
  - Retrieves the Apsis One API credentials from the custom table
  - Sends real-time notifications to Apsis One with profile data

[Erik Andersson]: > "The dynamics does not natively support like web hooks. So let's say like we would have we could support a flow where we say we only download good profiles from Dynamics at will like the full sync. The full sync makes a request to Dynamics. Please please give me all of your contacts. The background process that this package also creates it implements like a web hook system."

#### 2. Secondary Solution (if applicable)
Details not extensively covered, but implied to be a supporting solution or dependency.

### Installation Flow

1. **Customer receives** the solution file (e.g., `Apsis_managed.zip`) from Apsis sales/onboarding
2. **Customer navigates** to their Dynamics instance → Advanced Settings → Solutions
3. **Power Apps interface** appears (separate from the main Dynamics UI)
4. **Click** "Import solution"
5. **Upload** the `.zip` file
6. **Microsoft applies** the solution, creating the security role, custom tables, and background processes
7. **Apsis backend** then configures the OAuth app, creates the application user, and attaches the security role

### Why the Plugin Is Mandatory

Without the plugin installed:
- No security role exists; Apsis has no permissions
- No custom tables exist; no way to store integration credentials
- No webhook system; only manual/scheduled syncs possible
- **Installation would fail** or the integration would be non-functional

[Erik Andersson]: > "Yes, the very short version is if you don't have this plugin like nothing will work like you will fail to uninstall and even if you had managed to install we would not have permission to read any contacts and no data that would be sent to Apsis, so it would be completely useless."

---

## Dynamics Configuration: Advanced Settings and Solutions

### Accessing Advanced Settings

Within a Dynamics 365 instance:
1. Click the **Settings cog** (gear icon, top right)
2. Select **Advanced Settings**
3. This opens a separate admin portal (distinct from the main Dynamics UI)

### Solutions Menu

In Advanced Settings:
- Left sidebar menu contains **Solutions**
- This is where imported solutions are listed
- From here, you can view installed solutions, their versions, and dependencies

### Power Apps (make.powerapps.com)

When you click to manage or import a solution, you are redirected to **Power Apps**:
- This is Microsoft's low-code/no-code platform
- Contains **Import solution** button
- Allows uploading `.zip` solution files
- Shows solution details, version history, components (security roles, custom tables, etc.)

**Critical note**: Customers will also have access to this Power Apps interface in their own organizations. Apsis engineers will never have direct access to customer Power Apps instances—only customers and their admins can manage their solutions.

---

## On-Premises Dynamics: Not Supported

Apsis One does not officially support on-premises Dynamics deployments.

**Why**:
- The integration relies on **Azure resources** (webhooks, OAuth flows)
- If a customer runs Dynamics completely on-premises with no internet connectivity, the integration cannot function
- Real-time sync requires the ability to reach Azure services and Apsis APIs

[Erik Andersson]: > "If the customer have a like completely locked down on premise which cannot talk with the Internet, essentially like I mean we we don't support that. So the official standpoint from Apsis is like we we don't support on premise dynamics."

**Assumption**: All Apsis Dynamics customers run **cloud-based (SaaS) Dynamics 365**, so the UI and behavior observed in development instances should match customer environments (modulo version updates).

---

## Key Takeaways

1. **Legacy Connector Is Deprecated**: New customers should be directed to the Sideshop generic connector; the legacy connector receives no new features and creates operational overhead (tenant administration, single-secret risk).

2. **OAuth Application Management Is Critical**: The production OAuth app (`Apsis 1 EU prod`) is a single point of failure for all legacy customers. Its secret is stored in AVS Secrets Manager and must be carefully managed. If lost, recovery requires regeneration; if the app is deleted, all customers must reinstall.

3. **Global Administrator Approval Required**: During setup, the customer's Global Administrator must explicitly authorize the Apsis OAuth app. This is often the biggest blocker in real-world deployments.

4. **Two Azure Management Portals**: Admin Power Platform manages environments/instances; Microsoft Entra manages users and roles. Both are essential for development work.

5. **External User Invitation via SSO**: Always invite team members as external users using their `@fc.com` accounts, not by creating disconnected local accounts. This ensures automatic revocation when they leave.

6. **Plugin Installation Is Mandatory**: The solution file must be imported into every customer's Dynamics instance. It creates the security role, application user, custom tables for credentials, and the webhook system. Without it, the integration is non-functional.

7. **Real-Time Sync Requires Custom Web Hooks**: Dynamics doesn't natively support webhooks, so the plugin implements a background process that listens to Dynamics events and sends real-time notifications to Apsis One.

8. **Enable Dynamics 365 Apps During Environment Setup**: Failure to enable this option during instance creation will prevent syncing of marketing lists and full CRM functionality.

9. **Customers Own Their Instances**: Apsis engineers never have direct access to customer Azure organizations or Power Apps instances. All work is done in Apsis-owned development and test environments, with the assumption that customer environments are configured similarly.

---

## Unresolved Questions and Action Items

- **Access Issues**: Lukasz Grabowski encountered insufficient privileges error when trying to view applications in the `apsis.onmicrosoft.com` organization. This may require admin role assignment or acceptance of an outstanding Entra invitation. Should be resolved in a follow-up session.

- **Solution File Location**: Erik mentioned the solution file location will be shown "during the setup instead" but did not provide a concrete path in this session. Needs documentation.

- **Cleanup of Legacy Applications**: Multiple expired OAuth applications exist in app registrations. Erik plans to document which applications are in use and remove obsolete ones, but this work is not complete.

- **Complete Plugin Installation Walkthrough**: The session ends before the plugin import completes. A follow-up is needed to show the full installation process, post-import configuration, and how Apsis sets up the application user and API credentials.