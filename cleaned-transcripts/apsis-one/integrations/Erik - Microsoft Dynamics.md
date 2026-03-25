---
source_file: Erik - Microsoft Dynamics.txt
domain: Apsis One Integrations
topics: [Microsoft Dynamics 365 CRM Setup, Legacy vs. New Connectors, Azure AD and Entra Configuration, OAuth Applications Management, Dynamics Environment Creation, Plugin Installation, Security Roles and Permissions]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Microsoft Dynamics 365 CRM, Azure AD/Microsoft Entra, Power Platform Admin Center, Power Apps, Dynamics Solution/Plugin, OAuth Application Registration, Dataverse, API Client Credentials]
session_type: knowledge-transfer
subdomains: [Architecture, Microsoft Dynamics, Inbound Flow, Outbound Flow]
---

## Session Overview

Erik Andersson conducted a comprehensive knowledge transfer session on the legacy Microsoft Dynamics 365 CRM integration within the Apsis One platform. The session covered the architectural distinction between the legacy connector (which Apsis owns and maintains) versus the new Sideshop-managed generic connector approach, the setup of Azure AD organizations and environments, OAuth application management, environment provisioning in Dynamics, and the critical plugin installation process that enables real-time synchronization. This is foundational knowledge for any developer working with customer Dynamics integrations.

---

## Legacy vs. New Microsoft Dynamics Connectors

### Strategic Direction: Moving to Generic Connector

[Erik Andersson]: The current integration platform has two Microsoft Dynamics connectors available to customers:

1. **Legacy Connector (Microsoft Dynamics 365 CRM)** — the native, first-implemented connector that Apsis owns and maintains
2. **New Connector (Dynamics by Sideshop)** — managed by an external partner, built on the generic connector framework

> "The recommendation is for all new customers to go down this route because this will this is more future proof for new features. It's like everything we add for the generic connector. There is a strong possibility that Sideshop will add this to this connector, whereas we will not add any new functionality to the old connectors."

**Key implication**: The legacy connector is in maintenance mode. New feature development will not occur for the old connector, making the Sideshop generic approach the future standard.

---

## Azure Organizations and Multi-Tenancy Architecture

### Two Critical Azure Organizations

Apsis maintains two separate Microsoft Entra (Azure AD) organizations for Dynamics integration:

#### 1. **Apsis Utveck Apsis Development** (`apsisut.onmicrosoft.com`)
- **Purpose**: Development and testing environment
- **Owner**: Created and maintained by CRM Consultans (external partner)
- **Permissions**: Apsis team can create new instances and manage licenses freely
- **Use cases**: QA testing, development environment setup, plugin testing

#### 2. **Apsis Organization** (`apsis.onmicrosoft.com`)
- **Purpose**: Credentials and OAuth client storage
- **Content**: Stores all OAuth application registrations and secrets used for customer integrations
- **Critical data**: Application IDs and secrets for every customer's Dynamics instance

[Lukasz Grabowski]: "So development is up."

[Erik Andersson]: "Yes. And we are invited to Apsis dot on Microsoft.com."

### Access via Directory Switching

When logging into Azure services with FEC (Apsis) credentials:
- Visit `entra.microsoft.com` (Microsoft Entra/Azure AD) or `admin.powerplatform.microsoft.com` (Power Platform Admin)
- Use the "Switch Directory" option in the top-right to toggle between organizations
- External users are invited via Microsoft's external user model, which ties their access to their centralized FEC account
- When a user's FEC account is disabled centrally, their access to both organizations is automatically revoked

**Best practice for user invitations**: Always choose "Invite external user" rather than creating new users, as this maintains SSO integration with the FEC account lifecycle.

---

## Critical OAuth Application Management

### Single OAuth Application for All Customers

Apsis uses **one shared OAuth application** for all customer Dynamics integrations:

- **Application Name**: `Apsis 1 EU prod` (primary production application)
- **Location**: Registered in `apsis.onmicrosoft.com` organization
- **Scope**: Handles authentication for every customer's Dynamics instance

[Erik Andersson]: > "The Apsis one prod EU like this is the single most important application of them all because we use this one to access all customer environments. It might not be the best approach because essentially it's the comparison of having one API secret for every customer account, but that is how it was set up for us."

[Michal Rosikiewicz]: "So this one application has access to multiple user environments."

[Erik Andersson]: "Yes, we use this to access any customer dynamics instances so we can access multiple, but of course like they they cannot access anything else because we have the secret for it."

### Why This Single Application Matters

If this application or its credentials were lost or compromised:
- **Incident severity**: CRITICAL — all customer integrations would immediately fail
- **Recovery requirement**: Every customer would need to re-authorize access and reinstall the integration
- **Business impact**: Potential outage affecting all Dynamics customers simultaneously

This is why access to the OAuth application secrets in Azure is tightly controlled and why team members need visibility into where these credentials are stored and managed.

### OAuth Application Variants by Environment

```
Apsis 1 APAC prod     — Production, Asia-Pacific region
Apsis 1 EU prod       — Production, Europe region (primary)
Apsis 1 staging       — Staging environment (current)
Apsis 1 beta          — Beta features environment
Apsis 1 Integration tests  — Integration test environment
```

[Erik Andersson]: "There are a lot of them. Some are expired, some are old one which I have not really dared touch but the only ones that you will ever have to worry about is the Apsis staging 2. This is the new one... The only ones that you will ever have to worry about is the Apsis staging 2 [for staging]... Apsis 1 EU prod and Apsis one APAC prod [for production]."

Note: Multiple legacy applications exist from earlier iterations; cleanup is ongoing but deferred due to uncertainty about their purpose.

### Secret Management and Updates

The OAuth application secrets are stored in AVS Secrets Manager and loaded into the Integration Manager at boot time. These secrets are used internally by Apsis to:
1. Request access tokens from Azure
2. Authenticate with customer Dynamics instances

**Critical distinction**: Customers do NOT store the secret. When Apsis regenerates a secret:
- Apsis updates its own configuration and services
- Customers do NOT need to reinstall or update anything
- The secret is only used in the backend OAuth flow

However, if Apsis had to create an entirely NEW application with a NEW application ID:
- The new application would need customer re-authorization
- Every customer would need to go through re-installation

### Where to Update Secrets

Location in Azure:
```
Microsoft Entra > App Registrations > [Application Name] > Certificates & secrets > Client secrets
```

From this page, new secrets can be created with custom expiration policies. Current production secrets are set to "never expire."

### Customer Authorization and The Consent Flow

During installation, customers grant permission (via OAuth consent screen) for the Apsis application to access their Dynamics instance. This is a one-time approval.

**Real-world incident**: One customer had their IT team disable the application mid-production after questioning what it was. This completely severed Apsis's access to that customer's environment until the application was re-enabled. This was a highly disruptive situation that required customer IT involvement to resolve.

> "I really, really hope this is something you would never ever have to deal with, but at least it's good that you know where you do this if needed to be."

---

## Managing Users in Azure Organizations

### Global Admin Requirements

- **Global Administrator** role is required to approve OAuth application access for Dynamics during initial setup
- This is the **biggest challenge with customers** — getting their IT departments to allocate time and authorization to grant permissions
- After the first installation, Global Admin privileges are not required for subsequent installations

### Current Global Admins

During the session, the team reviewed the global admin list in the development organization and noted several outdated entries:
- Some admins no longer work with the organization and should be removed
- Others like Gustav Westerlund (CRM Consultans) are critical stakeholders and must remain
- Cleanup of inactive accounts was deferred to later

### User Types and Access Levels

- **Member**: Can modify and manage resources
- **Guest**: Read-only access, cannot make changes
- **External User**: Invited users who authenticate via SSO (recommended for Apsis team members)

---

## Dynamics Environment Creation

### Step-by-Step Environment Provisioning

Using the Power Platform Admin Center (`admin.powerplatform.microsoft.com`), environments are created within the Apsis lab organization (development):

**Environment creation parameters**:
```
Name: Integration Development (example)
Region: Europe
Dataverse storage: Enabled (required for database)
Security group: Open Access (still requires authentication)
Dynamics 365 apps: Enabled (critical for marketing lists and lead handling)
```

**Why Dynamics 365 apps must be enabled**:

[Erik Andersson]: > "We want to enable the Dynamics 360 apps because this enables more functionality and more precisely it enables Marketing lists and lead handling inside Dynamics. If you don't do this then you can only handle your regular contacts, but the integration supports Marketing lists inside Dynamics like the e-mail list syncing. If you don't enable this then we can't access them like there will be nothing to to be shown there."

### Behind-the-Scenes Infrastructure

When an environment is created, Microsoft automatically:
1. Spins up infrastructure in Azure
2. Creates a Dataverse database instance
3. Registers DNS records for the environment URL

**Example environment URL pattern**:
```
integrations-development.crm4.dynamics.com
```

### Environment State Transitions

- **Initial state**: "Preparing" (spinning up resources, 1-2 minutes)
- **Ready state**: Environment is accessible, can be logged into with appropriate credentials

### Customer vs. Development Environments

The development environments visible in the Apsis lab organization are **never touched by customers**:

[Erik Andersson]: > "You will never ever touch this page for the customer... the customer will have this page with their environments. This is only our development environment that we are looking at now. You will never ever touch this page for the customer."

Apsis always works with customers' own Dynamics instances, which are in the customers' own Azure organizations.

---

## Dynamics Instance UI and Advanced Settings

Once an environment is provisioned, users log in with their Azure AD credentials and access the Dynamics interface. The critical setup steps occur in the Advanced Settings section.

### Accessing Power Apps and Solutions

From within a Dynamics instance:
1. Click the **Settings cog wheel** (top-right)
2. Navigate to **Advanced Settings**
3. In the left menu, find **Solutions**
4. This opens the **Power Apps** platform interface where plugins (called **solutions** in Dynamics terminology) are imported

[Erik Andersson]: > "What Power Apps is, is like it's a tool to have lots of cool like automations running inside the Dynamics instances or your organization like essentially you can have like Ron jobs that trigger once every something and then access some Azure APIs and do like and do some magic."

The Power Apps interface is also **never accessed by customers directly** for plugin installation — that is handled by Apsis during the implementation process.

---

## The Apsis Plugin (Solution)

### What the Plugin Does

The plugin is packaged as a **Dynamics Solution** file and provides three critical capabilities:

#### 1. **Security Role Creation**
The plugin automatically creates an Apsis-specific security role with comprehensive permissions:
- Full read/write/delete permissions on **Contacts** at the organizational level
- Full read/write/delete permissions on **Marketing Lists**
- Any other permissions required for the integration to function

[Erik Andersson]: > "The plugin creates a security role inside of the customers dynamics instance and this security role has all of the permissions that we need in order to handle all our services. So this security role will have every permission available for contacts like reading contacts, updating contacts and also have the permission to do this on an organizational level."

#### 2. **Application User and Authentication Setup**
During installation, the plugin enables:
- Creation of an **application user** in the customer's Dynamics instance
- Binding of the Apsis OAuth application to this application user
- The application user has the security role created above attached

This allows Apsis to:
- Generate access tokens via the Apsis OAuth app
- Authenticate as the application user in the customer's environment
- Access resources with the appropriate permissions

#### 3. **Real-Time Sync Infrastructure (Web Hooks)**

Dynamics natively does NOT support web hooks. The plugin creates a custom infrastructure to simulate webhook functionality:

**The problem without webhooks**:
- Apsis could only perform periodic full syncs: "Please give me all your contacts"
- Real-time or near-real-time updates would be impossible
- High latency between Dynamics changes and Apsis updates

**The plugin's solution**:
- Creates **custom database tables** to track events
- Implements an **event listener** within the customer's Dynamics instance
- When an event occurs (contact created, updated, deleted), the listener checks if it's contact-related
- If relevant, it retrieves the one API credentials (stored in a custom table during installation)
- Sends real-time notification to the one API with the profile data

**How the credentials are stored**:
- During installation, Apsis generates a **one API client ID and client secret** specific to the customer
- These credentials are stored in the custom table within Dynamics
- The background process uses these credentials to authenticate when sending real-time notifications to one API

### Plugin Installation Process

1. Sales/support provides the customer with a **solution file** (developed by CRM Consultans)
2. Customer (or Apsis on behalf of customer with permission) imports the solution via Power Apps
3. Solution installs all components: security role, application user binding, custom tables, event listeners

**Failure scenarios**:
- If the plugin is NOT installed: Integration is completely non-functional
- Without permissions from the plugin's security role: Apsis cannot read any contacts
- Without the custom tables: Real-time sync infrastructure doesn't exist
- Without one API credentials: Real-time sync cannot send data back to Apsis

[Erik Andersson]: > "If you don't have this plugin like nothing will work like you will fail to uninstall and even if you had managed to install we would not have permission to read any contacts and no data that would be sent to Apsis, so it would be completely useless."

### Plugin Variants

Two related plugins are mentioned:
- **Apsis [Something] Managed** — The main plugin described above
- A second plugin variant (specifics not detailed in this session)

---

## On-Premises Dynamics (Not Supported)

Apsis does NOT support on-premises Dynamics installations because:

1. The integration relies on **Azure web hooks** and **Azure OAuth flows**
2. These require internet-facing Azure infrastructure that on-premises installations cannot access
3. Enterprise firewall restrictions at on-premises customers (fully locked-down, no internet access) are incompatible with Apsis's architecture

[Erik Andersson]: > "We use web hooks in Azure which we will come to. We have the O off login flow in Azure like if the customer have a like completely locked down on premise which cannot talk with the Internet. Essentially like I mean we we don't support that. So the the official standpoint from Apsis is like we we don't support on premise dynamics."

---

## Development and Production Environments

### The Assumption of Consistency

[Lukasz Grabowski]: "The assumption that the the dynamics looks the same for our development and for customers is is valid, right?"

[Erik Andersson]: > "It should be the same and if not, it should not be like it should not be a big difference in terms of versions because like I I frequently had the UI in the test instance been updated without me knowing and I had to essentially relearn like where do I look for things."

**Reality**: Microsoft occasionally updates the Dynamics UI without notice. The team discovers changes when something is no longer where they remember it being. Despite these updates, the fundamental structure and capability remain consistent between test and production environments.

---

## Key Takeaways

1. **Legacy connector is in maintenance mode**: New customers should be directed to the Sideshop generic connector. No new features will be added to the legacy Microsoft Dynamics connector.

2. **Two Azure organizations are critical infrastructure**:
   - `apsisut.onmicrosoft.com` (development/testing)
   - `apsis.onmicrosoft.com` (credentials and OAuth apps)
   - Team members must understand how to navigate between them via directory switching

3. **Single OAuth application for all customers is a single point of failure**: The `Apsis 1 EU prod` application secret, if lost, would require every customer to reinstall. This secret is the most critical credential in the entire system.

4. **OAuth secrets are internal to Apsis**: When Apsis rotates a secret, customers do NOT need to update anything. Secrets are only used in backend OAuth flows. A NEW application creation would require customer re-authorization.

5. **Global Admin approval is required for first-time setup**: Customer IT must approve the Apsis application's access once during initial setup. This is often the biggest implementation blocker.

6. **The plugin is absolutely essential**: Without the Apsis solution/plugin installed:
   - Security roles don't exist
   - Application user binding doesn't occur
   - Real-time sync infrastructure is missing
   - Integration is non-functional

7. **Real-time sync via custom web hook implementation**: Because Dynamics doesn't natively support webhooks, the plugin creates custom tables and event listeners. The one API credentials are stored in Dynamics itself and used by the background process to send real-time notifications.

8. **Dynamics UI updates are frequent**: Be prepared for the interface to change between test and production, or to change in test environments without warning. Core functionality remains consistent despite UI changes.

9. **On-premises Dynamics is not supported**: Any customer asking about on-premises deployment should be advised that Apsis only supports cloud-based Dynamics 365 due to Azure infrastructure dependencies.

10. **User cleanup is an ongoing operational concern**: When team members leave, their accounts must be manually removed from both Azure organizations. This is a reason Sideshop's managed generic connector approach (where customers own the Azure organization) is strategically preferred.

---

## Unresolved Issues and Follow-Up Actions

### Access Issues to Be Resolved
- Lukasz Grabowski does not yet have sufficient privileges to view OAuth applications in the `apsis.onmicrosoft.com` organization
- Lukasz is not yet added as an admin to the development organization (`apsisut.onmicrosoft.com`)
- **Action**: Erik needs to grant Lukasz admin privileges or ensure proper group membership in both organizations

### Global Admin List Cleanup
- Several users are listed as Global Administrators who no longer work with the organization
- **Action**: Michal Rosikiewicz noted that at least one user (possibly Zenita Pelko) can be removed; others need verification
- Gustav Westerlund (CRM Consultans) must remain as a global admin

### Plugin Installation Demonstration
- The session ended before the plugin import could complete
- **Action**: Session to continue (possibly recorded) to show the full plugin installation flow and verification steps

### Documentation of OAuth Applications in Use
- Erik noted that many expired and obsolete OAuth applications are cluttering the list
- **Action**: Create a clean reference list of applications actually in use and decommission the rest

### Continued Session
- **Timing**: Potentially later today or tomorrow
- **Option**: Record the remainder without requiring all attendees' presence (they can review async)
- **Duration**: One additional hour recommended to cover plugin installation, configuration verification, and any remaining topics