---
source_file: Erik - Microsoft Dynamics.txt
domain: Apsis One - Integrations
topics: [Microsoft Dynamics 365 CRM Setup, OAuth Applications & Secrets Management, Azure Entra ID Administration, Plugin Installation, Security Roles, Real-time Synchronization, Legacy vs. Generic Connectors]
speakers: [Erik Andersson (Integration Platform Expert), Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Microsoft Dynamics 365, Azure Entra ID, Power Platform Admin, Power Apps, OAuth Apps, Plugin/Solution Files, Security Roles, CRM Consultana, Sideshop Generic Connector]
session_type: knowledge-transfer
---

## Session Overview

This KT session covers the legacy Microsoft Dynamics 365 CRM connector for Apsis One, with particular focus on the administration and setup process. Erik walks through the critical infrastructure components: the two Microsoft organizations hosting Dynamics credentials (one for development, one for production), the OAuth application architecture, user management via Azure Entra ID, environment provisioning, and the mandatory plugin installation that enables real-time synchronization. A key strategic note: **new customers should use the Sideshop-managed generic connector going forward**, as the legacy connector will receive no new feature development.

---

## Legacy vs. Generic Connector Strategy

**Current State**: Apsis offers two Dynamics connectors:
1. **Microsoft Dynamics 365 CRM** (legacy) — the original implementation, the subject of this session
2. **Dynamics by Sideshop** (generic) — managed by Sideshop, implements a generic connector pattern

[Erik Andersson]: > "The recommendation is for all new customers to go down this route [Sideshop generic] because this is more future proof for new features. Everything we add for the generic connector, there is a strong possibility that Sideshop will add this to this connector, whereas we will not add any new functionality to the old connectors."

**Rationale for migration**: The team owns full administration of the legacy connector's tenant and OAuth apps, which creates operational overhead. When employees leave Apsis, they must be manually removed from these Microsoft organizations — even though their FSC (Apsis) user accounts are disabled, audit findings look unfavorable. The generic approach transfers this burden to Sideshop.

---

## Microsoft Organization & Environment Architecture

### Two Separate Organizations

The infrastructure relies on two distinct Microsoft organizations, each serving different purposes:

**1. Development/Credentials Organization: `Apsisut@on.microsoft.com`**
- **Owner**: CRM Consultana (the external firm that helped develop the solution)
- **Purpose**: Development environment; contains the OAuth applications and secrets
- **Access Model**: Apsis has full administrative permissions and can create new instances
- **Usage**: Team creates test/QA environments here; this is where all OAuth app management occurs

**2. Production Customers Organization: `Apsis@on.microsoft.com`**
- **Purpose**: Stores OAuth client credentials that customers use
- **Access Model**: Contains credential data; less frequently accessed by developers
- **Critical Note**: This organization should contain the OAuth app secrets; where customers' credential data is stored centrally

[Lukasz Grabowski]: > "So we are invited to Apsis dot on Microsoft.com."
[Erik Andersson]: > "Exactly."

### Admin Power Platform vs. Entra ID

[Erik Andersson]: > "On the Admin Power Platform here you manage your micro[soft] instances and the environments. Here on the entra.microsoft.com, this is the old Azure AD. So here you manage the users for your organization. So entra.onmicrosoft.com is for your organization, the admin.powerplatform.microsoft is for your environments or your tenants."

- **`admin.powerplatform.microsoft.com`** — Manages Dynamics environments (instances), where you provision new environments and configure them
- **`entra.microsoft.com`** (formerly Azure AD) — Manages user identity and access control for the organization

---

## User Management & Access Control

### External Users vs. Internal Users

When adding team members to the Microsoft organizations, **always choose "invite external user"**, not "create new user":

[Erik Andersson]: > "If you invite a user, you will create a completely new user for your organization that is disconnected from the FSC world. And you would need to add like a username and password. If you invite an external user, now you're reusing the SSO flow from Microsoft. If the root user in AVS is disabled, then there will be no access to your account. And it's the same thing here — if you invite an external user and you invite their FC user, then when that user is centrally disabled at FC, the access to your tenants is also removed."

**Guest vs. Member**:
- **Guest**: Read-only access; cannot modify anything
- **Member**: Can modify resources depending on role assignment

**Global Administrator Role**:
Only Global Administrators can approve authorization requests when an OAuth app first connects to a customer's organization. This is the biggest customer-facing bottleneck during setup, as it requires their IT team to grant explicit consent.

[Erik Andersson]: > "The reason why you have to be a global admin is because only global admins can approve the authorization of like the app connecting to your organization. If you have done this once, then you don't need to be an admin to install Dynamics, but the first time this happens you need to have a global administrator approving it. This tends to be the biggest challenge for customers — to get their IT people to want to or get time to do this for them."

---

## OAuth Application Architecture & Secrets Management

### Single Shared Application for All Customers

All legacy Dynamics connector installations use **one shared OAuth application** in the production organization. This is architecturally questionable but intentional:

[Erik Andersson]: > "This is the single most important application of them all because we use this one to access all customer environments. It might not be the best approach because essentially it's the comparison of having one API secret for every customer account, but that is how it was set up for us."

**Application List** (from `Apsis@on.microsoft.com` > App Registrations > All Applications):
- `Apsis 1 APAC Prod` — Production APAC region
- `Apsis 1 EU Prod` — Production EU region (most critical)
- `Apsis 1 Staging 2` — Staging environment
- `Apsis 1 Beta` — Beta testing
- Various expired/legacy apps to be cleaned up

[Michal Rosikiewicz]: > "So this one application has access to multiple customer environments."
[Erik Andersson]: > "Yes, we use this to access any customer dynamics instances. Of course, they cannot access anything else because we have the secret for it. If this one were to disappear for whatever reason, then that's a very annoying situation, to say the least."

### Secret Storage & Rotation

Secrets are stored in **AVS Secrets Manager** and loaded by the Integration Manager at boot time.

**Where to create/update secrets**:
1. Navigate to `Apsis@on.microsoft.com` > App Registrations > [Select app, e.g., "Apsis 1 EU Prod"]
2. Under **Certificates & Secrets** section
3. Click **New client secret**
4. Copy the secret value and store in AVS Secrets Manager

[Erik Andersson]: > "We load this in the Integration Manager whenever it boots up and that is then utilized in the request to Azure to generate an access token. In the same fashion, we use that access token to communicate with the actual instance."

**Secret Expiration**: The current secrets are set to never expire, which simplifies management but requires careful handling.

### Application ID Usage (Frontend Configuration)

The Application ID (not the secret) is embedded in **frontend configuration** for the setup flow. This is one of the few OAuth operations handled by the frontend rather than the backend.

[Erik Andersson]: > "This is like one of the few things which is handled by the front end and not handled by our back end. If we go to the front end repository and search... in their global configuration setup for the product environments, the front end should utilize this Dynamics client ID."

Example mapping:
- **Staging**: `B6660...` (Apsis 1 Staging 2)
- **Staging**: `B6605...` (similar staging configuration)

### Implications of Secret Changes

[Michal Rosikiewicz]: > "You said when refreshing a secret, customer wouldn't have to do anything. Wouldn't they have to update the secret value?"

[Erik Andersson]: > "They don't. The way it works is that when we install Dynamics, the customer gives us permission to log in to their environment using our specific application. The secret is for us. The important thing for them is that they have given access to the specific application. If we had to change the secret, that's only used by us when we talk with Microsoft. But if you had to create a completely new application, then this new application would once again have to be approved by the customer."

---

## Creating & Configuring Dynamics Environments

### Environment Provisioning Process

Access `admin.powerplatform.microsoft.com` and navigate to **Manage > Environments**.

**Parameters for new environment**:
- **Name**: e.g., "Integrations Development"
- **Region**: Select appropriate geography (e.g., Europe)
- **Add Dataverse data store**: Yes — this creates the backend database
- **Security Group**: Typically set to "Open Access" (authentication is still required)
- **Enable Dynamics 365 apps**: Critical — enables Marketing Lists and Lead handling

[Erik Andersson]: > "We want to enable the Dynamics 365 apps because this enables more functionality and more precisely it enables Marketing lists and lead handling inside Dynamics. If you don't do this, then you can only handle your regular contacts, but the integration supports Marketing lists inside Dynamics."

### Behind-the-Scenes Setup

Once saved, the environment moves to "Preparing" state and Azure provisions:
1. A Dynamics instance with a auto-generated URL (e.g., `integrations-development.crm4.dynamics.com`)
2. A Dataverse database backend
3. DNS registration

This typically takes 1-2 minutes.

---

## Customer vs. Development Environments

**Critical operational boundary**: Developers **never directly access customer Dynamics instances**. 

[Erik Andersson]: > "When it comes to customers, you will never be able to touch their corresponding pages like the customer will have this page with their environments. This is only our development environment that we are looking at now. You will never ever touch this page for the customer."

[Lukasz Grabowski]: > "Nobody is going to give us access to this, right? So we can only test something on our environment and then we have hope that it looks the same for the customers, right?"

[Erik Andersson]: > "It should be the same. If they run like full on-premise, we don't support that. We use web hooks in Azure which we will come to. We have the OAuth login flow in Azure. If the customer has a completely locked down on-premise which cannot talk with the Internet, essentially we don't support that."

**Supported deployments**: Cloud-based (Azure-hosted) Dynamics 365 only. On-premise Dynamics is not supported because the integration relies on:
- Azure-hosted web hooks
- OAuth flows that require internet connectivity
- Cloud resources that on-premise installations cannot access

---

## The Plugin/Solution: Core Installation Component

### What is a Solution?

In Dynamics terminology, a **solution** is a packaged plugin that adds functionality to an instance. The Apsis integration requires plugin installation.

**Access point**: Customer logs into their Dynamics instance > Advanced Settings > Solutions menu > Power Apps page (make.powerapps.com) > Import Solution

[Erik Andersson]: > "What we are looking for is to install a plugin inside of Dynamics, and in the Dynamics lingo this is called a solution. You can see here, to the left after you went to advanced setting, you have a menu called Solutions. Then here, now we come to a third page called Make Power Apps."

### Two Critical Plugins

**1. Apsis [Something] Managed (Main Plugin)**
- **Purpose**: Creates a security role with all required permissions
- **Permissions granted**:
  - Read contacts (organizational level)
  - Update contacts (organizational level)
  - List operations and other contact-related operations
  - Custom database table access for internal operations

**2. Secondary Plugin**
- Supports the main plugin's functionality
- Details not fully elaborated in this session

[Erik Andersson]: > "This plugin is the main plugin because what it does is it creates a security role inside of the customers Dynamics instance, and this security role has all of the permissions that we need in order to handle all our services."

### Why Plugin Installation is Mandatory

Without the plugin:
1. **No security role**: Cannot create an application user with necessary permissions
2. **No custom tables**: Cannot store credentials or operational metadata
3. **No real-time sync**: Cannot implement web hook system for event-driven synchronization
4. **Installation fails**: The process cannot complete
5. **Zero data access**: Even if partially installed, no contacts can be read

[Erik Andersson]: > "If you don't have this plugin, like nothing will work. You will fail to uninstall, and even if you had managed to install, we would not have permission to read any contacts, and no data would be sent to Apsis, so it would be completely useless."

---

## Installation Process & Application User Creation

### High-Level Flow

1. **Pre-installation**: Customer provides solution file (delivered by sales team) and grants IT approval
2. **OAuth Approval**: Global Admin in customer org approves our OAuth app
3. **Plugin Import**: Solution file imported via Power Apps interface
4. **Application User Creation**: Team creates an application user tied to our OAuth app
5. **Security Role Assignment**: Attach the plugin-created security role to the application user
6. **OAuth Credentials**: Generate Azure app client ID and secret; store in plugin's custom table
7. **Real-time Sync Setup**: Background process uses stored credentials to send real-time notifications

### The Application User & Token Generation

An **application user** is a service account inside the customer's Dynamics instance:

[Erik Andersson]: > "One step we do in our installation process is that we create an application user in the customer's instance. We attach our OAuth app that we have in their organization, so essentially we can access their instance by using an access token generated by our Azure app. Then we attach the security role that we have created with the plugin."

Once created, Apsis can generate access tokens using the shared OAuth app credentials (`Apsis 1 EU Prod`, etc.) and use those tokens to call Dynamics APIs on behalf of the application user.

---

## Real-Time Synchronization Architecture

### Webhook System for Event Listening

Dynamics does not natively support webhooks. The plugin implements a workaround:

[Erik Andersson]: > "The dynamics does not natively support like web hooks. So let's say we could support a flow where we only download profiles from Dynamics at will — the full sync. The full sync makes a request to Dynamics: 'Please give me all of your contacts.' The background process that this package creates implements like a web hook system. Essentially it listens to events inside dynamics and then checks: was this event for a contact? If it is for a contact, then it will contact one API with the profile data."

### Event-Driven Sync Flow

1. **Event occurs** in Dynamics (contact created, updated, etc.)
2. **Background process** (running in Dynamics or via scheduled job) detects the event
3. **Custom table lookup**: Retrieves stored One API client ID and secret
4. **REST call**: Sends contact data to One API (`https://one.apsis.com/...`) via HTTP POST
5. **Apsis ingestion**: One API receives and processes the profile data

[Erik Andersson]: > "As part of the installation process, we generate a One API client ID and client secret, send it to Microsoft Dynamics, and it is stored in one of these custom tables. Then the background process will utilize that secret when it sends us real-time notification."

### Contrast: Full Sync

Alternatively, Apsis can perform a **full sync** — polling Dynamics at scheduled intervals for all contacts without relying on event detection.

---

## Critical Gotchas & War Stories

### Disabled OAuth App = Complete Outage

[Erik Andersson]: > "We just had a problem with a customer where on installation they had approved us to log in on their instance with that application, and then later, like someone in their IT had said, 'What is this? I'll disable it.' And then we lost complete access to their environment until they re-enabled it."

**Prevention**: Document the OAuth app in the customer's IT security policies and communicate why it's essential.

### Organizational Housekeeping

Because Apsis owns the Microsoft organizations (`Apsisut@on.microsoft.com` and `Apsis@on.microsoft.com`), when employees leave:
- FSC user accounts are automatically disabled
- But they remain in the Entra ID directory
- Audits will show inactive users with admin roles
- Manual cleanup is required

This is a strong incentive to migrate customers to the Sideshop generic connector, where Sideshop owns the tenants.

### UI Changes Between Versions

[Erik Andersson]: > "I frequently had the UI in the test instance been updated without me knowing and I had to essentially relearn where to look for things."

Expect Dynamics UIs to shift without notice. Screenshots and documentation can become stale quickly.

---

## Repository & Configuration

### Frontend Configuration

The **frontend repository** contains global configuration for product environments. Search the codebase for references to:
- `Dynamics client ID`
- Application IDs (e.g., `B6660...`)
- Environment-specific settings (Staging, Beta, Prod EU, Prod APAC)

[Erik Andersson]: > "In their global configuration set up for the product environments, the front end should utilize this Dynamics client ID."

### Solution File Location

The plugin/solution file is provided by the **sales team** to the customer and contains all necessary components (security roles, custom tables, background processes).

(Details on where this file is stored in the repository were not fully covered in this session.)

---

## Access Troubleshooting

During the session, several team members encountered access issues when attempting to view app registrations or switch between organizations:

- **Insufficient privileges error**: Even after being added as members to organizations, some users couldn't view app registrations
- **Directory switch not reflecting**: Switching directories in Entra sometimes didn't update the current context immediately
- **Possible causes**: 
  - Insufficient role assignments (may need Global Admin or Application Administrator)
  - Cache/browser state issues
  - Not yet accepted invitation emails

[Erik Andersson]: > "It might actually be because you are not admins. That is a strong possibility."

**Workaround**: Check Entra ID user roles; assign appropriate roles if needed. Ensure invitations are accepted.

---

## Key Takeaways

1. **Legacy connector is sunset**: New customers should use Sideshop generic connector; no new features will be added to the legacy connector.

2. **Two Microsoft organizations**: 
   - `Apsisut@on.microsoft.com` (development, managed by Apsis/CRM Consultana)
   - `Apsis@on.microsoft.com` (production credentials/secrets, managed by Apsis)

3. **One shared OAuth app for all customers**: `Apsis 1 EU Prod` (and regional variants) are the critical apps; their secrets are stored in AVS and loaded by Integration Manager at boot.

4. **Plugin is non-negotiable**: The Dynamics solution file must be imported; without it, zero permissions and zero data sync are possible.

5. **Global Admin approval is the customer bottleneck**: The first OAuth consent request requires their Global Administrator; plan extra time for this during implementations.

6. **Real-time sync is event-driven**: The plugin implements a webhook system that listens for contact changes and pushes data to One API in real time.

7. **Cloud-only**: On-premise Dynamics is not supported; integration relies on Azure-hosted resources and OAuth flows.

8. **Always invite external users**: Reuse FC.com SSO rather than creating independent Microsoft users; ensures access is revoked when employees leave.

9. **Security role setup is plugin-dependent**: The plugin creates the security role; the application user is then attached to it. This is how Apsis gains the permissions it needs (contact read/write at organizational level).

10. **Customer instances are off-limits**: Developers only work in their own test environments. Customers manage their own instances. Configuration is validated in dev, then replicated in customer orgs via the plugin solution file.

---

## Unresolved Questions & Action Items

1. **Access for Lukasz**: Lukasz encountered persistent issues viewing app registrations in Entra ID despite being added to the organization. Needs follow-up:
   - Verify role assignments (may need Global Admin role)
   - Test switching back to `Apsis@on.microsoft.com` after signing out/in
   - May require Erik to manually assign stronger permissions

2. **Cleanup of unused OAuth apps**: The app registrations list contains many expired/legacy applications. Erik committed to:
   - Clean up clearly expired apps
   - Document which apps are actively in use (Prod EU, Prod APAC, Staging 2, Beta)
   - Create a reference list for the team

3. **Solution file location**: The repository location for the plugin/solution file to be installed was not explicitly covered. Needs documentation:
   - Where is the solution file stored?
   - How is versioning handled?
   - How do customers obtain it during implementation?

4. **Continuation of session**: Erik needs to leave for a meeting with Ali; session to continue later (potentially recorded) to cover:
   - Plugin import process in detail
   - Credential setup and testing
   - Troubleshooting common installation failures
   - Migration strategy from legacy to generic connector

5. **Role assignment for team members**: Verify that all team members have appropriate roles (Global Admin, Application Administrator) to manage the organizations and OAuth apps without requiring Erik for every task.