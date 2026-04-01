---
source_file: "Erik - Dynamics setup part 2 & code review of event listeners for campaigns.txt"
domain: Apsis One Integrations
topics:
  - Outbound manager refactor for CRM patch campaigns
  - Event listener registration order bug and fix
  - Microsoft Dynamics 365 user access setup
  - Dynamics user permissions (Administrative vs Read/Write access mode)
  - Microsoft Entra (Azure AD) licence requirements for Dynamics access
  - Azure App Registrations for Apsis One OAuth applications
  - External vs dedicated user accounts in Dynamics environments
  - Environment and user cleanup (deprovisioning)
  - Global administrator role requirements for initial Dynamics plugin setup
  - Secret rotation for OAuth app registrations
speakers:
  - "Erik Andersson (departing senior integration developer, knowledge transferor)"
  - "Tomasz Kowalski (integration developer, knowledge recipient)"
  - "Michal Rosikiewicz (integration developer, knowledge recipient)"
key_components:
  - Outbound Manager
  - Apsis One (Staging, Beta, EU Prod, APAC environments)
  - Microsoft Dynamics 365 / Sales Hub
  - Microsoft Power Platform Admin Center
  - Microsoft Entra (Azure AD / Entra ID)
  - Azure App Registrations (OAuth applications)
  - CRM Plugin (Apsis One Dynamics plugin)
  - Generic Connector / Siteshop
  - Apsis Pro (legacy)
  - MA (Marketing Automation component)
session_type: knowledge-transfer
---

# Session Overview

This session covered two main areas. Erik first explained the background and rationale for a pending refactor of the Outbound Manager's event listener registration logic, specifically the bug introduced when a CRM partner (E-deal / FCC Corporate) declared it only wanted top-level event tool activities — not child activities — yet the current code was still sending all of them. The bulk of the session then became a hands-on troubleshooting and remediation exercise to get Tomasz and Michal proper access to the Dynamics 365 test instance, which revealed a non-obvious Microsoft requirement: users need specific licences assigned in Entra before their access mode can be changed from "Administrative" to "Read/Write." The session ended with a walkthrough of the Azure App Registrations used for Apsis One's OAuth flow across all environments, plus user/environment cleanup across both the Dynamics instance and Entra.

---

## Outbound Manager Refactor: Event Listener Registration Order Bug

### Background and Business Context

**[Erik Andersson]:** The story that went through grooming was titled "Refactor the Outbound Manager and Patch Campaigns." This was triggered by a request from a specific CRM system — **E-deal (formerly FCC Corporate)** — which asked for a modification to the generic connector flow.

### How the Current Flow Works (and Why It's Broken)

**Current flow:**

1. An email campaign is created in the email tool.
2. The email tool makes a request to the **Outbound Manager**.
3. The Outbound Manager sends a request to the CRM system: "A campaign has been created with this Activity ID and name."
4. The CRM system responds confirming it created a campaign entry on its side.

For **Event Tool** and **MA**, the concept of **child activities** exists — sub-activities attached to a parent event tool activity. Examples:
- Email send-outs
- Form activities (e.g., registration forms)

**Current behavior:** Before making the request to the CRM system, the Outbound Manager registers **event listeners in Apsis for all child activities** (email send-outs, forms, events, etc.). The CRM system's response then tells us which activities it actually created — but by then, listeners are already registered for everything.

**The problem with E-deal specifically:** E-deal stated they are **not interested in child activities at all** — only the top-level event tool activity. However, because listeners were already registered for all child activities before the CRM response came back, those events are still being sent to E-deal even though E-deal never created any corresponding activity to attach them to.

### The Required Fix

**[Erik Andersson]:** The fix is conceptually simple — shuffle the order of operations:

1. **First** make the request to the CRM system.
2. **Receive their response** indicating which activities they actually created / are interested in.
3. **Then** register listeners only for those activities.

Instead of using the full child activity list as input to listener registration, use only the CRM system's response as the input.

### Status of the Existing Pull Request

An existing PR was created for this story, but Erik's assessment is that it goes far beyond what's needed — it performs extensive unnecessary refactoring.

**[Erik Andersson]:**
> "It's almost to the level where it's easier to start from the beginning. The actual change needed is essentially moving one code block earlier and using the CRM response as the input for listener registration instead of the full list."

**Decision:** The existing PR will not be reviewed in this session. The story will be moved to the integrations to-do board and re-implemented together as a learning exercise. The PR may provide some salvageable pieces, but will largely be rewritten.

---

## Microsoft Dynamics 365 Access Setup: Root Cause Investigation

### The Core Problem

Tomasz and Michal were unable to access the Dynamics 365 integration development environment (`integration development` environment in Power Platform). Symptoms included:
- HTTP 404 on the environment URL
- "Security group not set" error
- "Does not have a valid authorization to complete your request" on plugin installation
- Permission errors referencing Dynamics 365 Conversations table (a table the plugin doesn't even use)

### Root Cause Discovered: "Administrative" vs "Read/Write" Access Mode

**[Erik Andersson]:**
> "I believe all of these problems have been because of the Administrative access mode — you could manage the instance but not actually read the data in the instance."

In Dynamics / Power Platform user settings, there are two distinct access modes:
- **Administrative** — can manage the environment/instance but **cannot read or interact with actual data records**
- **Read/Write** — full access to data; required for normal use and for the Apsis plugin installation flow

Erik had "Read/Write" on his own account; Tomasz and Michal had been set to "Administrative."

### Prerequisite to Setting Read/Write: Licences Must Be Assigned First

**Critical gotcha discovered during the session:**

> Simply changing the access mode to "Read/Write" in the Power Platform admin reverts back to "Administrative" unless the user has the required Microsoft licences assigned in **Entra (Azure AD)**.

**Required licences (based on Erik's account as reference):**
- **Microsoft Fabric** (trial)
- **Power Automate Free**

**Optional / role-dependent:**
- **Microsoft 365 Sales Enterprise Edition** — grants access to Leads and Marketing Lists views in Dynamics. Only 3 licences available in the lab tenant; not strictly required for core plugin functionality.

**Workflow to correctly set up a new Dynamics lab user:**
1. Assign the required licences to the user in the Microsoft 365 Admin Center (`admin.microsoft.com` → Active Users → Licences and Apps).
2. In Power Platform Admin Center, navigate to the environment → Users → select the user.
3. Change access mode from "Administrative" to "Read/Write."
4. Save and refresh the user record.
5. The user should now be able to access Sales Hub and the Dynamics data.

**[Erik Andersson]:** "I have learned something new today — I never had to do this nitpicky step before. They seem to have introduced this requirement relatively recently."

### Additional Factor: Disabled User Status

Separately, one of the users (Michal's lab account) was found to have a "Status: Disabled" flag. Enabling it was a prerequisite before any permission changes would take effect.

### External User Conflict

An earlier hypothesis was that a stale **external FCC user** (an external invitation that had been added to the environment) was conflicting with the lab user accounts. Michal deleted this external user entry from Power Platform during the session. While this alone did not resolve the access issue, removing stale external users is considered good hygiene.

---

## Dynamics Plugin Installation: Global Admin Requirement for Initial Setup

### Why a Global Admin is Required

**[Erik Andersson]:** The initial Dynamics integration setup in Apsis One requires granting an OAuth application permission to the customer's organisation. This is a hard Microsoft requirement:

> "It is a hard requirement on Dynamics that whenever you add an OAuth user to the organisation, it has to be a global admin — because it is such a sensitive operation. The whole flow is dependent on an OAuth user being added to the organisation."

When a team member tried to perform this step using their **external FCC user** (not a global admin in the customer tenant), Microsoft attempted to add the permission to the FCC tenant instead — which failed because team members are not global admins of the FCC tenant.

**Implication for customers:** Getting the initial plugin installation done requires involving a high-level IT person (global admin) at the customer's organisation. This is a known friction point and cannot be worked around for the legacy Dynamics integration.

### Exception: Generic Connector / Siteshop Flow

**[Erik Andersson]:** This global admin requirement **does not apply** to the generic connector (Siteshop) flow:

> "For the generic connector, we connect to an intermediate service. Apsis will never touch Dynamics directly in that flow — we only communicate with the intermediate service, and the intermediate service handles everything with Dynamics."

Authentication with Siteshop uses a standard **X-API key**. If all existing customers are migrated to the generic connector, the OAuth app registrations in Azure become irrelevant.

### Workaround: Use Dedicated Lab Users, Not External FCC Users

Once the dedicated lab users (e.g., `ttko-lab`, `mirror-lab`) have been properly set up (licences + Read/Write access mode), they should be used for all setup and testing. The external FCC user accounts can only be used after the initial OAuth consent step has been completed with a global admin account.

---

## Azure App Registrations: OAuth Applications for Apsis One

### Location

App registrations are in the **Apsis International AB** Azure/Entra tenant (accessed via `portal.azure.com`, switching to the Apsis directory). The relevant tenant is identified as `apsis[something].microsoft.com`.

⚠️ *[Note: There are two Apsis-related Azure tenants. One holds the Azure app infrastructure used by the integration platform; the other is used for code repositories. Confirm which is which before making changes.]*

### Active App Registrations (Must Be Preserved)

| App Name | Purpose |
|---|---|
| Apsis One Staging 2 | Current staging environment OAuth app |
| Apsis One Beta | Beta environment |
| Apsis One EU Prod | EU production environment |
| Apsis One APAC | APAC environment |

The `Authentication` blade of each app contains the **redirect/callback URLs** that are permitted for the OAuth flow. For example, the staging app contains the `apsis-one-stage-cloud-integration-callback` entry.

**[Erik Andersson]:** "If you were ever forced to set up Dynamics for a disaster recovery environment, you would need to create a new app registration here and add the redirect URL for that environment."

### App Registrations That Can Be Cleaned Up

| App Name | Status / Notes |
|---|---|
| Apsis One Sandbox | Can be disabled/deleted — sandbox environment was deleted during cost reduction exercise |
| Apsis One Sandbox 2 | Same as above |
| Apsis One APAC Beta | Can be deleted — APAC beta environment no longer exists |
| Disaster Recovery Beta | Likely created by Erik; can be deleted |
| Apsis One Staging 1 (expired) | Can be deleted — reason for replacement unclear but a new staging app was created |
| Permission Test Client | No credentials; can be deleted |
| Temp Beta Cleanup | No credentials; can be deleted |
| Test / Test 2 / Test to Remove | No credentials; can be deleted |
| Online Dev | No certificate; presumed inactive |
| Hassan Action Adapter / MSB Action Adapter | Unclear purpose; redirect URLs are `localhost` — likely a local test setup. Investigate before deleting. |

**Important note on why the sandbox was deleted:**
> "During cost reductions we were asked/ordered to delete it because the cost was way too high for what it provided. We used it to play around with completely new CloudFormation structures."

### Secret Rotation Procedure (Disaster Recovery)

If an OAuth app secret is lost or needs rotation (e.g., AVS Ireland backup fails):

1. Go to the App Registration → **Certificates & Secrets**.
2. Add a new **Client Secret**.
3. Store the new secret value in the **AWS Secrets Manager** (AVS secrets manager).
4. **Customers do not need to reinstall** — because the App ID (client ID) remains unchanged, and customer consent was granted to the application, not the secret. Only the secret value in the backend flow needs updating.

**[Erik Andersson]:** "We have never had to do this in practice. We did once have to upload a completely new application for staging, and every test environment stopped working as a result."

### Critical Ownership Issue: Nicholas's Account

**[Erik Andersson]:** Nicholas (a former team member) is set as **owner** of several app registrations. His account is disabled, but as owner he may be the only one able to perform certain actions on those apps.

> "Luckily, I have the credentials for his account saved for this specific reason. I'll fix this at the beginning of next week."

**[Tomasz Kowalski]:** Flagged that Nicholas is owner of "almost all" apps — this is a significant bus-factor risk.

**Action item:** Erik to use Nicholas's saved credentials to transfer ownership of app registrations to active team members.

---

## Entra / Azure AD: Global Administrator Role for App Registration Management

### Requirement

To be able to **delete, modify, or manage app registrations** in the Apsis Entra tenant (beyond just viewing), users need the **Global Administrator** role assigned in that tenant.

Tomasz and Michal were external users in the Apsis tenant and did not initially have this role. Erik added the Global Administrator role to Michal's account during the session, which allowed Michal to perform app registration operations.

**[Erik Andersson]:** "At least one of you has it now, and that means you can go in and modify permissions for others too. That's good enough for now."

### Access Method for Team Members

Team members access the Apsis Entra tenant using their **FCC user accounts** (not the lab accounts):
1. Log into Azure Portal with FCC credentials.
2. Switch directory to the Apsis International AB tenant.

---

## Dynamics and Entra Environment User Cleanup

### Philosophy

**[Erik Andersson]:**
> "I would suggest that at a later stage we just delete everyone who shouldn't be here anymore. For now, disabling is good enough."
>
> "This is why developers should not own and maintain Dynamics instances — situations like this happen."

### Users Actioned During This Session

| User | Action | Reason |
|---|---|---|
| Benjamin (lab tenant) | Disabled | No longer with company; had Global Admin role |
| Janita (lab tenant) | Already disabled by Erik | No longer with company |
| Sravidya (lab tenant) | Disabled | No longer with company |
| Felix (lab tenant) | Deleted | No longer with company; only had managerial access |
| Admin2 / Join Admin Tool (lab tenant) | Deleted | Test accounts, no longer needed |
| Lucas Grabowski external user (Dynamics) | Deleted | External invitation no longer needed |
| FCC external user for Michal (Dynamics) | Deleted | Stale external invitation; potentially causing access conflicts |
| Sravidya (Apsis production Entra) | Should be disabled — deferred | FCC account being disabled automatically revoked her external access |

### Users to Leave Untouched

| User | Reason |
|---|---|
| Najeeb (lab tenant, disabled) | External CRM consultant; developed the Apsis One Dynamics plugin and Apsis Pro plugin. Re-enable on request. Company: "CRM Consultants" (Swedish: CRM-konsulterna) |
| Gustav Stalund (lab tenant, disabled) | CRM consultant; **main architect/designer** of the Dynamics solution. Najeeb handles implementation; Gustav handles design decisions. Re-enable on request. |
| Shraddha (lab tenant) | On parental leave — still an active employee, access intentionally retained |
| Apsis Lead Integration User (created 2015) | Predates the integration platform by ~4 years. Likely connected to Apsis Pro or legacy lead system. **Do not touch until confirmed.** |
| Najeeb external (Apsis Entra) | Do not delete; may be needed for production Pro access |
| Stan (Apsis Entra) | CTO; has Global Admin — leave as-is |

### New User Created: Lucas (Lab Tenant)

A dedicated lab user was created for Lucas during the session:
- Username: `lukas-lab@[apsis-lab-domain]`
- Initial password set to a simple value; Lucas should change on first login
- Licences assigned: Microsoft Fabric (trial), Power Automate Free
- Global Administrator role added by Erik

---

## Key Takeaways

1. **Outbound Manager bug:** The current code registers Apsis event listeners for all child activities *before* asking the CRM what it wants. The fix is to reverse this: ask first, then register only for what the CRM confirms it created. This is a small code change but an existing over-engineered PR needs to be scrapped and redone cleanly.

2. **Dynamics "Administrative" vs "Read/Write" access mode is the critical gotcha:** New lab users land in "Administrative" mode, which blocks data access entirely. You cannot change this to "Read/Write" without first assigning the correct licences (Microsoft Fabric trial + Power Automate Free) in the Microsoft 365 Admin Center. If you try to change the mode without licences, it silently reverts.

3. **Global Admin is a hard requirement for initial Dynamics plugin installation** — cannot be worked around for the legacy Dynamics integration. The generic connector (Siteshop) does not have this requirement.

4. **If the OAuth app secret is lost**, generate a new one in Azure App Registrations and update AVS Secrets Manager. Customers do not need to reinstall as long as the App ID is unchanged.

5. **Nicholas's account ownership** over most app registrations is a critical risk. Erik has credentials saved and will transfer ownership before his departure.

6. **External FCC user accounts** automatically lose access to the Apsis Entra environment when their FCC account is disabled — this is the safer model. Dedicated lab users require manual deprovisioning and should be cleaned up explicitly when team members leave.

7. **When migrating all customers to the generic connector (Siteshop)**, the OAuth app registrations in Entra become entirely obsolete and the whole Azure app registration infrastructure can be retired.

---

## Unresolved Questions and Action Items

- **[Erik — before departure]:** Transfer ownership of Azure App Registrations away from Nicholas's disabled account using saved credentials.
- **[Erik — Monday/Tuesday]:** Begin formal KPI-based handover checklist review with Tomasz and Michal; identify gaps requiring additional sessions or written guides.
- **[Michal/Tomasz]:** Clean up expired/unused Azure App Registrations (Sandbox, Sandbox 2, APAC Beta, Disaster Recovery Beta, Staging 1, test apps). Defer until confirmed safe — do not delete in a Friday afternoon session.
- **[Open]:** Identify what the "Hassan Action Adapter" / "MSB Action Adapter" app registration is and whether it is still in use. Redirect URLs point to `localhost` — likely a local dev test, but confirm before deleting.
- **[Open]:** Identify what the "Apsis Lead Integration User" (created 2015) account is connected to. Suspected: Apsis Pro or legacy lead system. Do not touch until confirmed.
- **[Open]:** Determine whether to assign Global Administrator role to Michal in the lab Entra tenant (separate from the Apsis production Entra). Currently only Michal has it in the Apsis production Entra; Lucas and Tomasz's lab users may need it added by Michal.
- **⚠️ [Warning — ambiguous]:** The "Security group not set" error in Dynamics was observed alongside the access mode issue but it is not fully confirmed whether it was a *symptom* of the access mode problem or a separate issue. The fix (licences + Read/Write mode) resolved access in practice; monitor for recurrence.