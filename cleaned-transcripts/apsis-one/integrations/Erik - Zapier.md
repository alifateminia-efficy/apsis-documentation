---
source_file: Erik - Zapier.txt
domain: Apsis One Integrations
topics: [Zapier Integration Architecture, OAuth2 Authentication Flow, Apsis One API Integration, Dynamic Field Configuration, Zapier App Development and Deployment, Customer Support and Debugging, Token Refresh Middleware, Version Management]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Apsis One Zapier App, Zapier Platform, Apsis One API, OAuth2 Token Exchange, Sections/Key Spaces, Attributes, Events, Consents, Middleware Error Handling]
session_type: knowledge-transfer
subdomains: [Architecture, Lead creation]
---

## Session Overview

This knowledge transfer session covered the **Apsis One Zapier integration**, a custom-built JavaScript application that enables customers to create marketing automation workflows connecting Apsis One with external systems via Zapier. Erik Andersson walked through the integration architecture, authentication mechanisms, how the app is built and deployed, and troubleshooting of a customer-reported issue. The integration supports creating or updating profiles, adding events, and managing consents—all configured dynamically based on customer permissions. Key discussion points included the OAuth2 token refresh middleware that was added to handle expired tokens gracefully, the versioning and promotion workflow in Zapier's platform, and strategies for tracking integration usage and debugging customer issues.

---

## Integration Overview and Use Case

### What Zapier Integration Does

The Apsis One Zapier integration allows customers to build **marketing automation workflows that span multiple systems** using Zapier as the orchestration layer, rather than building flows entirely within Apsis One itself.

**Example workflow:** A customer maintains a Google Sheet with prospects interested in their product. Using Zapier:
- **Trigger:** When a new row is added to the spreadsheet
- **Action:** Data from that row is automatically sent to Apsis One to create or update a profile

This extends beyond Google Sheets—customers can trigger from **Microsoft Dynamics CRM contacts/leads**, Salesforce, email applications, and other systems supported by Zapier's ecosystem. [Erik Andersson]

### Supported Actions in the Apsis One App

The custom Apsis One Zapier app (not a standard Zapier integration) supports three main operations: [Erik Andersson]

1. **Create or Update Profile** — Uses the `create_or_update_profile` API endpoint to add or modify profile data, mapping external data fields to Apsis One attributes
2. **Update Consents** — Manage subscription opt-in/opt-out for profiles, configurable by section and key space
3. **Add Events to Profile** — Attach events to existing profiles (note: external event *creation* is not supported via the API, only adding to profiles)

> "We cannot create [events] externally because that's not supported in one API, but we built the functionality to add events to a profile." [Erik Andersson]

---

## Authentication and API Integration

### OAuth2 Flow with Apsis One API

The Zapier app uses **OAuth2** to authenticate with Apsis One:

1. Customer specifies their region (**APAC or EU**)
2. Customer provides **Client ID** and **Client Secret** (configured via Apsis One API Management)
3. Zapier exchanges these credentials for an **access token** by making a POST request to the OAuth token endpoint
4. If successful, the connection is established and the app can proceed with API requests

[Erik Andersson]

The region selection is critical because the API endpoint URL varies by region (e.g., EU vs. APAC). The credentials and region are stored as part of the Zapier account connection configuration.

### Token Refresh Middleware (Critical Fix)

A **critical issue** was discovered and fixed between app versions:

**Version 100 (Initial Release):** Did NOT include middleware to handle token expiration. When an access token expired:
- Zapier received a `403` response with the error message: `"user does not have permission to access these resources"`
- The request failed immediately without retry
- Customers saw confusing errors in their logs, even though the issue was recoverable

**Version 101+:** Includes middleware that:
1. Detects the specific `403` error indicating token expiration
2. Automatically triggers a token refresh using the stored Client ID and Client Secret
3. Retries the original request with the new access token
4. Returns the result to Zapier without exposing the error to the customer

[Erik Andersson]

> "What happened is that instead this results in an error in the code, so it looks like something has gone completely wrong for the customer. We are in fact handling this expected behavior. So we added this middleware so we are catching that error and instead making Zapier do a refresh of the token and do a retry on the failed request before that error reaches Zapier." [Erik Andersson]

This issue is suspected in the current customer case being debugged, but verification requires checking which app version the customer is using.

---

## Data Configuration and Field Mapping

### Sections and Key Spaces

To write data to Apsis One attributes, the **section and key space must be specified and whitelisted**. [Erik Andersson]

- **Section:** The data collection/segment in Apsis One (e.g., "Integration Handover", "FCC Enterprise")
- **Key Space:** The namespace controlling write permissions for specific attributes within that section (e.g., "email", "crm_id")

These cannot be controlled from Zapier itself—they must be **pre-configured in Apsis One** by the customer or professional services. The Zapier app only enforces these boundaries.

### Dynamic Field Population

The Zapier app uses **dynamic fields** to populate dropdown lists based on what's available in the customer's Apsis One account:

1. **Static fields** are always shown:
   - Section selector
   - Key space selector
   - Identifier for incoming data (which profile key to use for matching: email, CRM ID, phone number, etc.)

2. **Dynamic fields** appear conditionally based on selections:
   - **Attributes dropdown:** Only displayed after a section is selected. Fetches all writable attributes for that section/key space combination
   - **Events dropdown:** Only displayed after a section is selected. Lists only events the customer has permission to write to
   - **Subscription list:** Only populated if subscriptions exist for the selected section

[Erik Andersson]

**Rationale for dynamic approach:** This ensures customers only see options they actually have permissions for, preventing failed requests downstream. The app calls the Apsis One API to fetch these lists in real-time.

### Data Mapping

Once fields are configured, the customer can:
- **Map external data to attributes:** If the incoming trigger has a field called `email`, map it directly to Apsis One's email attribute
- **Use static values:** Hardcode values in the field mapping (e.g., always set a certain flag to `true`)
- **Match profiles:** The app identifies which profile to update using a specified identifier (email, CRM ID, etc.)

[Erik Andersson]

> "If I have an incoming field called CRM ID, then I would add the CRM ID to the CRM ID field. If I have email here, I can utilize the data from the incoming events to the email and as you can see, just as in [other systems], if I have an incoming trigger from the Google Sheet event here, then I can utilize the data from the triggering events." [Erik Andersson]

---

## App Architecture and Codebase

### Repository Structure

The Apsis One Zapier app code resides in the **FCC code repository** under `Apsis Integrations/Zapier/`, but **outside the main Justin application** since it is not a Justin component. [Erik Andersson]

The app is a **completely separate application** built in **JavaScript** using Zapier's framework. It is developed locally, then uploaded to Zapier's infrastructure for testing and promotion.

### Zapier Framework: Creates and Triggers

The code is organized into two main folders: [Erik Andersson]

**Creates** — Define the **actions** the app can perform:
- `add_consent.js` — Update consents for a profile
- `add_events.js` — Add events to a profile
- `update_profile.js` — Create or update a profile

Each create defines:
- Static metadata (description, label, logical key)
- Input fields (both static and dynamic)
- The `perform()` function — the main entry point executed by Zapier

**Triggers** — Functions that **populate dropdown lists** dynamically:
- `sections.js` — Fetches available sections; returns list of section names and IDs
- `keyspaces.js` — Fetches available key spaces for the selected section
- `identifiers.js` — Lists available profile identifiers (email, CRM ID, phone, etc.)
- `event_definitions.js` — Fetches event types available for a section; filters out events without write permission

[Erik Andersson]

> "Triggers is a bit weird name, but essentially you can consider triggers as like the different values that you can pick from the drop down." [Erik Andersson]

### Authentication Configuration

The `authentication.js` file defines:
- Required fields for account setup (region, Client ID, Client Secret)
- The OAuth2 token exchange configuration:
  - **Grant type:** `client_credentials`
  - **Token endpoint:** Constructed from region + hardcoded path
  - **Request body:** Includes Client ID, Client Secret, and grant type

[Erik Andersson]

---

## Versioning and Deployment Workflow

### Three-Stage Release Process

Zapier has a distinct workflow for changes:

1. **Develop locally:** Make code changes on your machine
2. **Publish:** Upload to Zapier infrastructure with a new version number (e.g., 102)
   - App is available to the development team only
   - Accessible to anyone with access to the Zapier project/organization
   - Can be tested internally without affecting customers
3. **Promote:** Make the version available to all Zapier users
   - Only promoted versions are discoverable in the public Zapier app marketplace
   - Customers can now select this version in their flows

[Erik Andersson]

**Current versions:**
- Version 100 — Initial release (lacks token refresh middleware)
- Version 101 — Likely intermediate version
- Version 102 — Current public version (includes middleware fix)

> "When you publish it, it will be 102. This is as far as I am aware, that's how it works whenever you publish a new... the customer will need to switch to the 102 version. I don't think you can do in place changes of existing versions. You can absolutely make the changes and you can push them, but I don't think it will automatically apply to the customer's nodes." [Erik Andersson]

### Version Update Implications

- **Non-disruptive changes** (e.g., adding a header to requests) could theoretically be applied to all versions, but Zapier does not support automatic rollouts
- **Customers must manually update** their flows to use a newer version; the old version continues to work unless explicitly deprecated
- There is **no known forced upgrade mechanism** in Zapier, so older versions with bugs remain active until customers act
- **Deprecation strategy:** Mark old versions as deprecated, forcing customers to choose another version when they next edit their flows [Michal Rosikiewicz]

**Action item:** Explore whether Zapier has tooling to deprecate versions or notify customers of updates.

---

## Monitoring, Debugging, and Error Handling

### Monitoring via Zapier Developer Dashboard

The Zapier developer platform provides a **monitoring dashboard** for each app version: [Erik Andersson]

- **Request history:** View all API requests made by the app
- **Error tracking:** See HTTP status codes and error counts (4XX, 5XX categories)
- **Time filtering:** Can view data within the last week (limited time range)
- **Log inspection:** Click on errors to see detailed logs

**Limitation:** The monitoring dashboard is less sophisticated than CloudWatch and provides limited filtering/search capabilities. Date range selection is restricted to recent periods (not queryable back months).

### Common Error Patterns and Debugging

**409 Conflict Error (Expected and Handled):**
- Occurs when Zapier attempts to create a profile that already exists
- The app is configured to accept both 201 (Created) and 409 (Conflict) responses as success
- Conflict is logged but not treated as a failure; the profile is simply not duplicated

[Erik Andersson]

> "If it is 201 or 409 then we then we proceed. This is something I can clean up for like logs sake, but it is not a problem per se." [Erik Andersson]

**403 Forbidden with Token Expiration:**
- Indicates the access token is no longer valid
- Middleware catches this and triggers a token refresh
- If customers are on version 100, this error is not retried and appears as a hard failure

**Other Potential Failure Causes:**
- Incorrect attribute/event write permissions configured in Apsis One (customer-side)
- Improperly formatted phone numbers in incoming data
- Permissions changed in Apsis One after the flow was created, invalidating previously working configurations

[Erik Andersson]

### Current Customer Issue

A customer has reported errors in their Zapier flow logs. The integration team suspects:

1. **Most likely:** Customer is using app **version 100**, which lacks the token refresh middleware. When a token expires, they see hard errors instead of automatic retry.
   - **Resolution:** Customer must upgrade to version 102

2. **If on version 102:** The error may indicate a permission issue:
   - Missing write permission on a subscription being updated
   - Missing write permission on an attribute or event
   - These permissions may have been revoked after the flow was created

**Next steps:** [Lukasz Grabowski, Erik Andersson]
- Contact customer (via Cristina) to determine which app version they are using (ask for a screenshot of the flow showing the Apsis One app node)
- If on version 100, instruct them to update to version 102
- If on version 102, request them to trigger the flow again so the error can be captured in Zapier's monitoring dashboard
- Cross-reference Apsis One API logs to identify the actual error (e.g., 403, permission denied)

**Challenge:** Apsis One integration team has no visibility into which customers use Zapier since the integration card is a README with a link, not a built-in integration. All traffic goes through the customer directly to Zapier, unlike other integrations where installation is tracked in the database.

---

## Implementation Details and Code Patterns

### Example: Sections Trigger

The `sections.js` trigger makes an HTTP GET request to fetch all sections:

```javascript
GET {api_region}/sections
```

Response is converted to Zapier format:
```javascript
[
  { name: "Integration Handover", id: "integration_handover" },
  { name: "FCC Enterprise", id: "fcc_enterprise" },
  ...
]
```

The **name** is displayed to the customer; the **id** is stored for subsequent API calls.

[Erik Andersson]

### Example: Dynamic Attributes Fetch

The `get_attribute_fields()` function in the update_profile action:

1. **Checks if a section is selected** — If not, returns empty (field hidden)
2. **Fetches all attributes** for that section from Apsis One API
3. **Filters results:**
   - Excludes deprecated attributes
   - Excludes attributes without write permission in the selected key space
4. **Converts to field list:** Returns only writable attributes to display in the dropdown

[Erik Andersson]

> "We only show attributes that are not deprecated. We only show attributes that we have right permission for and then we create the list dynamically from that." [Erik Andersson]

### Example: Perform Function in Update Profile

The main `perform()` function:

1. **Extracts configuration:** Section, key space, profile identifier, incoming data
2. **Verifies profile exists:** GET request to check if profile matches the identifier
3. **Updates profile:** POST to the `create_or_update_profile` endpoint with mapped data
4. **Handles token expiration:** Middleware detects 403 with "permission denied" and triggers refresh

[Erik Andersson]

---

## Request Tracking and Feature Requests

### Origin of the Zapier Integration

The Apsis One Zapier integration was **requested by the product team** (Rose) as a portfolio feature, not initiated by a specific customer request. [Erik Andersson]

### Current and Future Feature Requests

The integration team has received **new feature requests from CRM partners** that are considered "straightforward" to implement and would add value. These are prioritized for upcoming work.

Possible Golang-related implementation work is being considered, and team members are assessing their comfort level with Golang for future features. The use of GitHub Copilot and AI tools is expected to ease development for team members less familiar with the language. [Lukasz Grabowski]

### Platform Version Management

A **warning was noted** about the Zapier platform core package being outdated. [Michal Rosikiewicz]

Currently, the app uses an older major version of the Zapier platform core. However, this is **not an immediate blocker** because:
- The app is very static with few feature requests
- Previous version updates (e.g., 15 → 17) required only changing the dependency number, no logic changes
- Updates may become problematic only after multiple major version changes without upgrades

[Erik Andersson]

---

## Access and Team Coordination

### Zapier Developer Account Access

The Zapier developer platform is managed via a **shared organization account**. [Erik Andersson]

**Prior situation:** Benjamin managed access, which has since been updated.

**Current state:** Team members (Lukasz, Michal, Tomasz) have been invited to the Zapier project and should now have access to:
- View the app source code
- Publish new versions
- Promote versions to public
- Monitor app usage and errors
- Make changes to the app

This access is critical for the integration team to independently troubleshoot issues and deploy fixes without external dependencies.

---

## Key Takeaways

1. **Zapier integration is custom-built** — Not a standard Zapier integration; it's a JavaScript app that must be explicitly maintained and deployed by the integration team.

2. **OAuth2 with region awareness** — Authentication requires specifying EU or APAC region; token refresh is automatic but failed on old versions without middleware.

3. **Dynamic field population prevents errors** — Sections, attributes, events, and subscriptions are fetched from Apsis One at configuration time, filtered by customer permissions, reducing downstream failures.

4. **Versioning requires manual customer action** — New app versions must be published and promoted, but customers must manually update their flows; there is no automatic rollout mechanism.

5. **Token expiration was a critical issue** — Version 100 failed silently on expired tokens; version 102 includes middleware to handle refresh transparently. Customers on old versions must upgrade.

6. **Monitoring is limited but functional** — Zapier's dashboard shows error counts and logs but lacks sophisticated filtering; Apsis One API logs are the primary source for detailed troubleshooting.

7. **Permission mismatches are common failure points** — Customers must whitelist attributes, events, and subscriptions in Apsis One before use in Zapier flows. Permissions changed after flow creation cause failures.

8. **Team needs access to Zapier account** — All team members should have invites to the Zapier project to troubleshoot, review code, and deploy fixes independently.

---

## Unresolved Questions and Action Items

1. **Customer version and issue resolution** [In Progress]
   - **Action:** Lukasz to contact Cristina to ask the customer which version of the Apsis One Zapier app they are using (screenshot of the flow node)
   - **Outcome:** If version 100, customer must upgrade to 102. If 102, request flow execution to capture error in monitoring dashboard.

2. **Zapier version deprecation mechanism** [To Investigate]
   - **Question:** Can Zapier automatically deprecate old versions or force customers to upgrade?
   - **Implication:** If possible, could address the challenge of many customers still using version 100.
   - **Owner:** Erik or team to explore Zapier platform capabilities

3. **Apsis One API logging for Zapier source** [To Implement]
   - **Question:** Can a source identifier (header or other tracking mechanism) be added to Zapier API requests to distinguish them from other integrations in logs?
   - **Implication:** Would enable tracking token refresh frequency and identifying which customer accounts are using Zapier.
   - **Owner:** Michal raised; Erik to coordinate with Apsis One API team

4. **Zapier platform core version update** [To Prioritize]
   - **Question:** Should the app be updated to the latest Zapier platform core version proactively?
   - **Outcome:** Likely not urgent, but should be monitored for compatibility as platform matures.
   - **Owner:** Michal to track; consider for next maintenance window

5. **CRM partner feature requests** [Backlog]
   - **Description:** New feature requests from CRM partners are ready to implement; deemed straightforward and valuable.
   - **Next step:** Lukasz to review calendar and structure work priority; assess team Golang readiness.