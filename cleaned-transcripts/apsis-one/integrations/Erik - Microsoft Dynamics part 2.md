---
source_file: Erik - Microsoft Dynamics part 2.txt
domain: Apsis One Integrations
topics: [Microsoft Dynamics 365 integration setup, OAuth2 authentication flow, Azure Entra app registration, Dynamics plugin installation, security roles and application users, field mappings, subscription mappings, feature flags, multi-account limitations, real-time sync architecture, environment management]
speakers: ["Erik Andersson (Integration domain expert/owner)", "Lukasz Grabowski (KT recipient, team lead)", "Michal Rosikiewicz (KT recipient, developer)", "Tomasz Kowalski (KT recipient, developer)"]
key_components: [Apsis One, Microsoft Dynamics 365, Azure Entra (formerly Azure AD), Power Platform Admin, Global Discovery Service, Integration Manager service, CloudWatch, One API, Apsis Marketing List plugin, Apsis Integration User security role]
session_type: knowledge-transfer
---

# Session Overview

This is part 2 of a knowledge transfer session on the Microsoft Dynamics 365 integration within the Apsis One platform. Erik Andersson walks the team (Lukasz, Michal, Tomasz) through the complete OAuth2-based installation flow for the Dynamics integration, using a live staging environment. The session was highly interactive — the team encountered and debugged real permission/authentication issues in real time (external user SSO conflicts, missing security role assignments, browser session conflicts). Key topics covered include: the Azure app consent step, the Global Discovery Service, how application users are created in Dynamics, how security roles are attached, field and subscription mappings, and architectural limitations around real-time syncs across multiple accounts. The session also touched on environment management, feature flag cleanup, and the "consent roleplay" workaround in Dynamics using Boolean fields.

---

## Session Environment and Access Setup (Azure Entra / Directory Switching)

### Issue: External User Cannot See Correct Azure Directory

Lukasz could not see the correct Azure directory ("Apsis International AB" with domain `apsis...microsoft.com`) in the Switch Directory list. He could see a directory with the same display name but a different domain (`apsisint...microsoft.com`), which is the wrong one.

Erik confirmed Lukasz was already added as a user inside the correct directory, but the directory was not appearing in the switch list. This was not resolved during the session and was flagged for offline follow-up.

> "It is important that you are here, but it's not something we need to sit everyone together here now." — Erik Andersson

Erik had also just added Lukasz as a **Global Admin** in the correct directory before the session started, which he had previously omitted.

### Staging Environment Access

The correct environment for this KT is accessed at:

```
admin.ourplatform.com
```

The account used is the **Apsis Lab** account, not the FEC (team's main company) account.

---

## Recap: What Was Completed Before This Session (Part 1)

Before this session, the following was completed in the Microsoft Dynamics tenant:

1. Created a new Microsoft Dynamics **tenant** (development environment).
2. Navigated to **Advanced Settings → Solutions** and installed:
   - The core **Apsis integration plugin**
   - The **Apsis Marketing List plugin** (only available/usable if the customer has the Dynamics Marketing app enabled)
3. Created a **Security Role** named **"Apsis Integration User"** with the following permissions at Organization level:
   - **Create, Read, Update, Delete** on **Contacts** (needed because Apsis downloads contacts from Dynamics)
   - **Read** on **Apsis Authentication** data (needed for the real-time sync flow)
   - **Read** on **Lists** / Marketing Lists (from the marketing list plugin)

> The Dynamics environment is fully prepared at this point for the Apsis-side installation to proceed.

---

## Feature Flags for Dynamics, Magento, and Lime Integrations

### Current State

There is a **feature flag system in back office** that gates installation of certain integrations. Only three integrations still require a feature flag:

- **Dynamics**
- **Lime**
- **Magento**

All other integrations (e.g., Sleek Note, Super Office, Google Data Studio, Tessitura) have flags that are effectively redundant — they are already open to all users, or the external service itself controls access via API keys.

To check/enable the Dynamics flag: navigate to the account in back office → **Feature Toggles → Integrations → Dynamics**.

### Why These Flags Exist (Historical Context)

> "The whole idea previously was that you had to go in and enable all of this because it was a cost thing — you couldn't use it unless you paid for it. Then that requirement disappeared and it should be available for anyone because you need to have everything set up already." — Erik Andersson

### Current Rationale for Keeping Dynamics/Magento/Lime Flags

Even though the flags are arguably redundant (because you can't do anything without the plugin file from Apsis anyway), they haven't been removed yet.

> "Even if you were to try and install Dynamics, if you haven't paid Apsis extra, you would not have gotten the plugin file from us. If you don't have the plugin file from us, you can't do anything — so it doesn't make much sense for this to require a feature toggle." — Erik Andersson

### Cleanup Action Item

Erik flagged that all redundant integration feature flags (everything except Dynamics, Magento, Lime — and possibly those too) **should be deleted**. This cleanup task is on Erik's To-Do list. He noted it will also be a good opportunity to review how user permissions are evaluated before allowing actions.

---

## OAuth2 Installation Flow — Step-by-Step

### Overview

The installation flow is initiated from the **Integrations** section in Apsis One (bottom-left navigation). There is no on-demand Azure app creation — a **central OAuth application** already exists per environment:

- One app for **staging**
- One app for **production**

These are shared across all customers.

### Step 1: Organization-Level App Consent (One-Time Only)

**This step only appears the first time** the Dynamics integration is installed for a given Microsoft organization. It requires a **Global Admin** of the Azure/Entra organization.

What happens: The user approves Apsis's Azure application to be added to their organization. The permissions requested include: read user email addresses, sign in users, etc.

> "This has to be done once. If you reinstall this integration on any future section, any one of you — you don't need to do that step again because it's there now." — Erik Andersson

**Important:** In the staging environment, this step had been previously completed ~6 years ago, so the team had never seen it. Erik deliberately removed the app from the organization before the session so the team could observe this step.

### Step 2: List Organizations (Global Discovery Service)

The flow makes a request to Azure asking: for the account the user has selected, which Azure organizations does this account have access to?

This uses an endpoint Erik highlighted:

> "It's called the **Global Discovery Service** — and like, Global Discovery is... you discover global things. Which organization does this user have access to, which environments, which Dynamics environments exist in this specific organization." — Erik Andersson

The user selects their organization (e.g., Apsis Lab).

### Step 3: List Dynamics Environments

After selecting the organization, the flow retrieves all Dynamics instances/environments within that organization. For the Apsis Lab org, the available environments are:

- `Integrations Development` ← **use this for developer work**
- `Integrations QA` ← used by QA, do not use for development
- `Apsis Pro Dev` ← used by CM Consult for plugin development

**Only `Integrations Development` has the Apsis plugin installed** (as of this session).

### Step 4: Plugin Verification

The flow calls the selected Dynamics environment to retrieve installed solution metadata. The endpoint queried returns a large JSON document listing installed solutions.

```
https://<environment>.crm4.dynamics.com/api/data/v9.<x>/...
```

The code searches this response for a solution with the name matching the Apsis plugin. If not found, the error returned is: "plugin is not installed." 

> **Gotcha:** This same error appears both when the plugin genuinely isn't installed AND when the user running the flow doesn't have permission to access the environment. The root cause must be distinguished via CloudWatch logs.

### Step 5: Create Application User and Attach Security Role

If the plugin is verified, the flow:

1. Uses the logged-in user's Dynamics session to make a request to Dynamics to **create an Application User** in the specific instance.
2. **Links that Application User to the Apsis OAuth application** registered in Azure (e.g., "Apsis One Staging 2").
3. **Attaches the "Apsis Integration User" security role** (created during plugin installation) to that Application User.

After this step, the logged-in user's credentials are **no longer used** for any further Dynamics communication. All subsequent requests go through the Application User + OAuth app.

> "Your dedicated Azure user will not be utilized when we make requests to Dynamics [after setup]. It is only used to set it up. When we make a request to Azure saying we have this application that has access to this organization — please give us an access token for this environment — that's the Application User." — Erik Andersson

You can verify this in Power Platform Admin: **Manage → [Environment] → Settings → Users + Permissions → Application Users**.

### Step 6: Generate and Send One API Credentials

The flow generates a **One API client secret and client key** and sends them to the **plugin endpoint** inside the Dynamics instance. These credentials are stored in the plugin.

Purpose: When something happens to a contact in Microsoft Dynamics, the plugin uses these stored credentials to send **real-time updates** to Apsis One via the One API OAuth flow.

### Step 7: Default Country Code (Deprecated/Legacy)

There is a step to select a **default country code**. This is a legacy remnant:

- Originally, Apsis would format phone numbers to international format if they lacked a country prefix, using this default.
- This was **stopped by a higher-level decision** because Apsis was actively modifying customer data, which is not acceptable.
- The default country code is now only used in phone number **validation**, not modification.

> "This is also something we should remove in theory." — Erik Andersson

**Cleanup action item:** Remove the default country code step from the installation UI.

---

## Installation Flow — Summary Boiled Down

Erik summarized the conceptual steps, stripping away the Azure complexity:

1. List organizations → user selects one
2. List environments → user selects one
3. Verify plugin is installed in the selected environment
4. Create Application User + attach security role
5. Generate and send One API credentials to the plugin
6. Done

> "What is complicated is all of this Azure ritual setup — the user having global admin, the user having access to the specific environment, the user having system administrator access, and all of these other permissions. That is what is actually complicated with it." — Erik Andersson

---

## Prerequisite Requirements for Installation (Global Admin User)

The person performing the installation must:

- Be a **Global Admin** of the Azure/Entra organization
- Have **access to the specific Dynamics environment** (not just the organization)
- Have **System Administrator** role within that specific Dynamics environment

### War Story: External User SSO Conflicts During Session

The team discovered a significant practical issue: Michal is an **external user** in the Apsis Lab Azure tenant (his primary identity is in the FEC tenant via SSO). This caused cascading problems:

1. Could not use SSO credentials to log in as the Apsis Lab user during the consent step (no password exists — SSO only).
2. Attempts to reset password for the external user from within the Apsis Lab tenant were blocked: `"Main tenant can reset password of this user"` — the home tenant (FEC) controls it.
3. Browser session conflicts: Chrome was authenticated to both FEC and Apsis Lab simultaneously, causing Dynamics API calls to hit the wrong user's context.
4. **Resolution:** Created a **dedicated member user** (non-SSO, username/password) directly inside the Apsis Lab Azure tenant with Global Admin role, then used Firefox in private/incognito mode to avoid session bleed.

> **Warning:** Dedicated member users created this way are NOT tied to the company's SSO/directory. If a person leaves the company, this user will NOT be automatically disabled — manual cleanup required.

Erik noted: "Every time I am doing this, I am doing it with a user that is a dedicated user inside the organization." This is the expected/supported path. External SSO users are not the intended actor for this flow, and the process for handling them was unknown to Erik during the session.

**Unresolved:** How to properly handle this flow when the person performing setup is only an external/guest user in the customer's Azure tenant.

### War Story: `service appointment` Permission Error

During the second installation attempt, the flow failed with a CloudWatch error (found in the **Integration Manager** service logs):

```
CRM Security Exception: User [...] is missing privilege: read activity on entity: service appointment
```

- The Application User was created correctly and had the Apsis Integration User security role attached.
- The `service appointment` entity is **not part of the Apsis integration's intended data model** — the plugin does not knowingly use it.
- Erik had not seen this error before and flagged it for investigation.

> "I have no idea what that entity is even doing. I need to check that." — Erik Andersson

**Unresolved action item:** Investigate why `service appointment` read privilege is required and whether it needs to be added to the "Apsis Integration User" security role in the plugin.

---

## Uninstalling the Integration

When you uninstall the Dynamics integration from Apsis One:

- The **Application User inside Dynamics is deactivated** (not deleted, at least not immediately — there may be a small delay).
- The integration can be reinstalled freely on the same or different environments.

On reinstallation, **Step 1 (org-level app consent) is skipped** — it was already approved and persists. The user goes directly to organization/environment selection.

---

## Post-Installation: Field Mappings

After installation, navigate to the integration → **Field Mappings** (middle menu).

- By default, one mapping exists: `Contact` entity → `email` field.
- Clicking **Add New Mapping** shows all fields available from Dynamics — these are fetched live from the connected tenant using the Application User credentials.
- Customers map Dynamics CRM fields to Apsis fields here.

---

## Post-Installation: Subscription Mappings and the Consent Problem

### How Consent Works (or Doesn't) in Dynamics

Dynamics has **no native concept of consent/subscription opt-in** as Apsis understands it.

**Workaround:** Apsis "roleplays" consent using **Boolean fields** in Dynamics. If a Boolean field is `true`, it is treated as consent granted for the mapped Apsis subscription.

Navigate to **Subscription Mappings → Add New Mapping → select field** to see all Boolean fields available in the Dynamics instance.

### Critical Gotcha: Reverse-Worded Boolean Fields

Dynamics has many built-in fields that are worded in the negative, e.g.:

- `Do Not Consent to Email`
- `Do Not Contact`

> "If they have selected `true` on that field in Dynamics, this would be like 'I don't consent' — but in Apsis terms, it suddenly does become a consent because it is `true`. So it's important for customers not to use those specific fields that are worded that way, because it will become completely impossible to keep track of." — Erik Andersson

**Recommendation to customers:** Create a **custom Boolean attribute** with clear positive naming, e.g., `apsis_newsletter_consent` or `technical_newsletter_consent`. Do not use Dynamics' built-in negative-consent fields for subscription mapping.

---

## Multi-Account / Multi-Section Installation Limitations

### Question from Michal: Can multiple Apsis One accounts connect to the same Dynamics instance?

**Answer:** Yes, with an important limitation on **real-time syncs**.

### Full Syncs (Apsis pulling from Dynamics)

Work on **all sections/accounts** where Dynamics is installed. No limitation — Apsis downloads data, so it is pull-based and independent per account.

### Real-Time Syncs (Dynamics pushing to Apsis via plugin + One API)

**Design flaw in the current version:** The plugin in Dynamics can only hold credentials for **one** Apsis One installation target. Real-time events are sent only to the **last account/section where the integration was installed**.

**Workaround implemented by Apsis:** When a real-time event arrives at one section, the Integration Manager checks which other sections within **the same account** also have Dynamics installed, and replicates the event to all of them.

> **Limitation of the workaround:** It only works within a single Apsis account. If the same Dynamics instance is connected to multiple different Apsis accounts (e.g., Account A and Account B), real-time syncs will only reach all sections of whichever account was installed last.

### Real-World Example

> "We have a very big Dynamics customer called Vänsterpartiet — it's the left party in Sweden. They have like 25 sections with Dynamics. Every member and every contact in whole of Sweden is synced to different sections. So if Dynamics sends it to one section, we replicate it to everything so they don't have to do a full sync on every section every day." — Erik Andersson

### Practical Implication for the Team

For developer/QA work: real-time sync correctness is not critical. Full sync and field mapping functionality works regardless. The dedicated QA instance handles real-time sync verification.

---

## Environment Management and Constraints

### Available Environments in Apsis Lab Azure Organization

| Environment | Purpose | Notes |
|---|---|---|
| `Integrations Development` | Developer sandbox | Can be freely reinstalled, torn down, recreated |
| `Integrations QA` | Automated QA test suites | Do not use for development work |
| `Apsis Pro Dev` | Plugin development by CM Consult | CM Consult owns and uses this |

### Storage Quota Constraint

All environments share a **single database quota** under the Apsis Lab organization. The quota was previously large enough for 4 environments; it has since been reduced. Currently only 3 environments fit. Every contact stored in any environment also consumes quota.

> "It doesn't cost anything for us as far as I know — Apsis is not paying anything for this. But there are limits as to how much you can have." — Erik Andersson

### Recommendation for QA Environment

Erik recommended the team consider:
1. Tearing down the existing `Integrations QA` environment.
2. Recreating a fresh QA environment.
3. Installing the Dynamics integration there once and leaving it permanently for automated tests only (real-time sync checks, full sync verification, field mapping sanity checks).
4. Never using that environment for development/reinstallation work.

---

## CloudWatch / Debugging Integration Issues

Errors from the installation flow are logged in the **Integration Manager** service in CloudWatch.

To find them:
1. Go to CloudWatch (staging account, correct region — note: region switching was having issues during the session).
2. Navigate to **CloudWatch Logs** (not CloudFormation).
3. Filter for the **Integration Manager** log group.
4. Filter by last 5 minutes (or relevant time window).
5. Look for `WARNING` level entries — Azure/Dynamics errors appear here.

Example error format seen in session:
```
CRM Security Exception: User with ID [GUID] has not been assigned any roles
```
or:
```
User [GUID] is missing privilege: read_activity on entity: service_appointment
```

The GUID in the error corresponds to the **Application User's object ID** in Azure/Dynamics, which can be cross-referenced in the Entra users list to confirm which user is being used.

---

## Key Takeaways

1. **The installation flow has 6 logical steps**: org consent (one-time) → list orgs → list environments → verify plugin → create Application User + attach security role → send One API credentials to plugin.
2. **The installing user must be a Global Admin AND have System Administrator access to the specific Dynamics environment** — not just org-level admin. This is the most common source of installation failures.
3. **The plugin must be installed in Dynamics before the Apsis installation flow is run.** Customers install the plugin themselves; Apsis does not and should not have access to customer Dynamics tenants.
4. **External/SSO users from another tenant cannot easily perform the installation flow** — a dedicated member user with a password must be created in the target Azure tenant.
5. **After setup, the installing user's credentials are no longer used** — all ongoing communication goes through the Application User + OAuth app.
6. **Subscription consent in Dynamics is simulated via Boolean fields.** Customers must use positively-worded custom fields — never Dynamics' built-in negative-consent fields (e.g., "Do Not Contact").
7. **Real-time syncs only go to the last account that installed the integration.** Apsis has a workaround that replicates within the same account across sections, but cross-account real-time sync is not supported.
8. **Feature flags for Dynamics/Lime/Magento** still exist and must be enabled in back office for those integrations to appear as installable. All other integration feature flags are redundant and should be cleaned up.
9. **Only use `Integrations Development`** for developer testing. Do not touch `Integrations QA` (for QA automated tests) or `Apsis Pro Dev` (owned by CM Consult).
10. **The default country code step in the installation wizard is a legacy remnant** and should be removed — it no longer modifies data, only used in phone number validation.

---

## Unresolved Questions and Action Items

| Item | Owner | Notes |
|---|---|---|
| Fix Lukasz's Azure directory access issue (cannot see correct Apsis International AB tenant in Switch Directory) | Erik + Lukasz (offline) | Not resolved during session |
| Investigate `service appointment` read privilege error during Application User setup — why is this entity required, and should it be added to the "Apsis Integration User" security role? | Erik | Never seen before; needs investigation |
| Determine correct process for running the OAuth installation flow as an external/guest user in a customer's Azure tenant | Erik | Currently only works with dedicated member users |
| Lukasz and Tomasz to attempt full Dynamics installation on their own accounts as homework | Lukasz, Tomasz | After Erik resolves the `service appointment` error |
| Clean up redundant integration feature flags in back office (keep only Dynamics, Lime, Magento, possibly remove those too) | Erik | Confirm with "Rose" first |
| Remove the default country code step from the Dynamics installation wizard | Erik | Legacy remnant, no longer serves meaningful purpose |
| Decide on QA environment strategy — tear down and recreate `Integrations QA`, assign ownership to the team | Team + Erik | Relevant now that the team is also handling QA |