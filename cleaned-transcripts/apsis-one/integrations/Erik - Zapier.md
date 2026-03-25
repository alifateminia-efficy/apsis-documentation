---
source_file: Erik - Zapier.txt
domain: Apsis One Integrations
topics: [Zapier Integration Architecture, Custom App Development, OAuth2 Authentication, Dynamic Field Configuration, API Permissions, Token Refresh Handling, Monitoring and Debugging, Feature Implementation]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Apsis One Zapier App, Zapier Platform, One API, OAuth2 Flow, Dynamic Dropdowns, Middleware Error Handling, Monitoring Dashboard]
session_type: knowledge-transfer
subdomains: [Architecture, Zapier Integration, Generic Connector, Outbound Flow]
---

## Session Overview

Erik Andersson conducted a comprehensive knowledge transfer session on the **Apsis One Zapier Integration**, a custom-built application that enables customers to create marketing automation flows using Zapier as a no-code integration platform. The session covered the architectural design of the integration, how it handles authentication via OAuth2, the framework structure for defining actions and triggers, dynamic field configuration based on API permissions, and debugging challenges with token refresh handling. The team also discussed current customer issues, monitoring capabilities, and future feature requests. Access to the Zapier developer account was provisioned to team members during the session.

---

## Understanding Zapier and the Apsis One Integration

### What Zapier Does

[Erik Andersson]: Zapier can be conceptualized as a marketing automation flow builder, but instead of building flows within a platform itself, it orchestrates integrations between external systems. Rather than creating profiles directly in a system, Zapier acts as a connector that enables data movement between applications.

A typical use case: A customer has a Google Sheets spreadsheet containing leads or event invitees. When a new row is added to this spreadsheet, Zapier can automatically trigger an action—for example, creating or updating an Apsis One profile with that data.

### The Apsis One Zapier App Overview

The **Apsis One Zapier App** is a custom-built application developed by the integration team. It is hosted separately from the main Apsis One codebase and is not installed as a traditional plugin within Apsis One itself.

**Three Core Capabilities:**
1. **Create or Update Profile** — Add or modify profile attributes in Apsis One
2. **Add Events to Profile** — Attach events to existing profiles (events themselves cannot be created externally via this integration)
3. **Update Consents** — Manage subscription/consent status for profiles (opt-in/opt-out)

[Erik Andersson]: The reason we cannot create events externally is because that functionality is not supported in the One API. However, we built functionality to add pre-existing events to profiles.

---

## Architecture and Code Organization

### Code Repository Structure

The Zapier app code resides in the **Apsis Integrations repository** under a dedicated `Zapier` folder:
```
Apsis Integrations/Zapier/
```

Although it lives within the FC (Apsis One) code repository, it is a **completely separate application** and is not a Justin component. The application is built in **pure JavaScript** and follows the Zapier framework's specific conventions.

[Erik Andersson]: It's important to know that while the code lives in our repository, it's structured as a standalone application that gets built locally and uploaded to a Zapier account.

### Zapier Framework Structure

The Zapier app uses a specific directory structure to define functionality:

#### **Creates** Folder
Defines the **actions** the app can perform. Each action is a `.js` file:
- `add-consent.js` — Logic for updating consents
- `add-events.js` — Logic for adding events to profiles
- `update-profile.js` — Logic for creating or updating profiles

#### **Triggers** Folder
Defines **dynamic dropdown populations** that appear in the Zapier UI. Despite the name "triggers," these are not the same as Zapier flow triggers (which come from external systems). Instead, they populate configuration options:
- Section selector trigger (lists available CRM sections)
- Key space selector trigger (lists available key spaces)
- Identifier selector trigger (lists profile identifier options like email, phone, CRM ID)
- Event definition trigger (lists events the user has permission to write)

[Erik Andersson]: These triggers execute `perform` functions that make HTTP calls to the One API to fetch available options and convert them into key-value pairs suitable for Zapier's UI.

#### **Authentication** File
Defines authentication fields and OAuth2 flow configuration:
- Fields for API region selection (EU or APAC)
- Client ID and Client Secret configuration
- OAuth token endpoint configuration
- Token refresh logic

---

## Authentication Flow: OAuth2 Integration

### Initial Setup

When a customer configures the Apsis One Zapier app in their flow, they must first authenticate:

1. **Select Region**: Specify whether they are using the EU or APAC region of Apsis One
2. **Provide Credentials**: Enter Client ID and Client Secret (obtained from the Apsis One API Management interface)

[Erik Andersson]: These credentials are obtained from the Apsis One dashboard under API Management, where you create a new API credential set.

### Token Exchange Process

Once credentials are provided:

1. Zapier exchanges the Client ID and Client Secret for an **access token** using the One API OAuth endpoint
2. If successful, the authentication is marked as valid and the user can proceed with configuration
3. The access token is stored and used for subsequent API calls

### Token Refresh and Middleware

**Critical Issue (Version 100 vs. Version 101+):**

In the original release (version 100), there was no middleware to handle expired tokens. When an access token expired:
- Zapier would attempt to use the expired token
- The One API would return a `403 Forbidden` error with a message about insufficient access
- This error would bubble up to the customer's Zapier logs as a critical failure, even though the issue was merely an expired token

**Solution Implemented (Version 101+):**

A **middleware layer** was added to catch authentication errors:

```javascript
// Pseudocode of middleware logic
if (response.status === 403 && response.body.contains("user does not have permission to access these resources")) {
    throw RefreshAuthError(); // Signals Zapier to refresh the token
}
```

[Erik Andersson]: When a 403 error with the specific "user does not have permission" message is detected, we throw a `RefreshAuthError`. This tells Zapier to automatically:
1. Request a new access token using the stored Client Secret
2. Retry the failed request with the fresh token
3. All before the error reaches the customer's logs

This makes the token refresh transparent to the customer—the request eventually succeeds, but without the alarming error messages.

---

## Configuration: Mandatory vs. Dynamic Fields

### Static Fields (Always Visible)

Every action has static configuration fields that must be completed first:

**For all actions:**
- **Section** — Which CRM section (e.g., "Integration Handover", "Efficy Enterprise")
- **Key Space** — Which attribute namespace to write to (e.g., "email", "crm_id")
- **Identifier for Incoming Data** — How to identify the profile being updated (email address, phone number, CRM ID, etc.)

[Erik Andersson]: These fields must be selected first because the app needs them to make API calls to fetch the list of available attributes and events.

### Dynamic Fields (Conditional Display)

After selecting a section, the app triggers a **dynamic field fetch**:

1. A function called `get_attributes_fields` or similar executes
2. This function checks: "Do we have a section selected?"
3. If yes, it makes an API call to One API to fetch:
   - All attributes available on that section
   - Only those with write permissions for the specified key space
   - Filtered to exclude deprecated attributes

4. The API response is converted into form fields the user can fill

**Example Flow:**

```
User selects: Section = "Integration Handover", Key Space = "email"
  ↓
get_attributes_fields() executes
  ↓
One API call: GET /attributes?section=integration_handover&keyspace=email
  ↓
Response: [{ id: "email", label: "Email" }, { id: "first_name", label: "First Name" }, ...]
  ↓
Dynamic fields appear: [Email field] [First Name field] ...
```

### Why Permissions Matter

[Erik Andersson]: This approach was essential because we cannot control attribute whitelisting from outside Apsis One. The whitelisting (which key spaces and attributes a credential can write to) is configured within Apsis One's settings. By fetching the available attributes dynamically from the One API, we only show the customer options they actually have permission to use. Otherwise, every request would fail with a permission denied error.

---

## Detailed Action: Update Profile

### Static Configuration
The user specifies:
- Section
- Key Space
- Identifier for incoming data (email, phone, CRM ID)

### Dynamic Configuration
After selection, available attributes appear as form fields.

### Data Mapping

The customer can now configure how incoming data should be mapped:

**Option 1: Dynamic Mapping**
Map data from the triggering event (e.g., a new row in Google Sheets) to profile attributes:
```
Incoming field "first_name" → Attribute "first_name"
Incoming field "email" → Attribute "email"
Incoming field "phone" → Attribute "phone"
```

[Erik Andersson]: If the trigger is a new contact created in Microsoft Dynamics, the incoming fields would be the Dynamics contact properties like first_name, last_name, middle_name, etc. If it's a Google Sheet row, the incoming fields are the column names in that sheet.

**Option 2: Static Values**
Hardcode a fixed value for a field:
```
"category" → "webinar_attendee" (static)
```

### Execution (The `perform` Function)

When the Zapier flow is triggered:

1. The `perform` function in `update-profile.js` executes
2. It verifies the profile exists using the specified identifier (email, phone, CRM ID)
3. It constructs the profile update payload using the mapped data
4. It POSTs the update to the One API endpoint:
   ```
   POST /profile?section=<section>&keyspace=<keyspace>
   ```
5. If a 403 error occurs, the middleware catches it and triggers a token refresh + retry

### Important Caveat: Phone Number Formatting

[Erik Andersson]: One critical gotcha: phone numbers must be correctly formatted before being sent to the One API. If a customer sends an improperly formatted phone number, the update will fail silently or be rejected by validation. The customer is responsible for ensuring phone numbers are in the correct format in their source data.

---

## Detailed Action: Add Events to Profile

### Configuration Steps

1. **Static Selection**: Section, Key Space, Identifier for incoming data
2. **Dynamic Selection**: After selecting these, a dropdown appears listing all events available on this section (filtered to only those with write permission)

[Erik Andersson]: We filter events based on the endpoint we implemented at the One API level. Previously, we couldn't retrieve event permissions, so we asked the One API team to add an endpoint that returns whether we have write permission for each event. This prevents the flow from failing when trying to add an event the customer doesn't have permission for.

### Mapping Event Fields

Once an event is selected, the user configures which fields on that event should be populated:

```
Event: "email_opened"
  - timestamp (required)
  - click_type (optional)

Mapping:
  timestamp ← incoming_timestamp (dynamic)
  click_type ← "email" (static)
```

### Execution

The app POSTs the event data to the One API for the profile identified by the selected identifier (email, phone, etc.) in the selected key space.

---

## Detailed Action: Update Consents

### Configuration

**Static Fields:**
- Section
- Key Space
- Identifier for incoming data

**Dynamic Dropdown:**
After selection, the app fetches all available subscriptions for this section and displays them.

### Consent Selection and Branching Challenge

[Erik Andersson]: Users can select a subscription and specify whether it should be opt-in or opt-out. However, if the incoming data contains the opt-in/opt-out decision as a value (rather than hardcoded), we have a challenge.

**The Problem:**
Different customers may use different value conventions:
- Some use `0` for opt-in and `1` for opt-out
- Some use `true` for opt-in and `false` for opt-out
- Some use the string `"opt_in"` and `"opt_out"`

**Our Solution:**
We decided to **place this burden on the customer**. Rather than building logic in the app to handle every possible convention, the customer must:

1. Use Zapier's native branching/conditional logic to check the incoming value
2. Route opt-in requests to one node (configured for opt-in)
3. Route opt-out requests to another node (configured for opt-out)

[Erik Andersson]: We chose not to support both opt-in and opt-out in a single node because it would require us to handle an infinite number of different value interpretations. By having the customer do the branching themselves, they ensure their specific convention is respected.

---

## Version Management and Release Process

### Three Version Stages

1. **Development (Private)**
   - Developer builds the app locally
   - Publishes to Zapier with a new version number (e.g., 102)
   - Marked as **private** — only visible to team members with access to the Zapier project
   - Used for testing and internal QA

2. **Testing Phase**
   - Team can add the private version to Zapier flows for testing
   - Multiple team members can access and validate functionality
   - No customer impact

3. **Promotion (Public)**
   - Developer clicks "Promote" on the published version
   - Version becomes available to all Zapier users
   - Customers can now select this version in their flows

### Current Versions in Production

- **Version 100**: Original release (lacks middleware for token refresh)
- **Version 101**: Intermediate version
- **Version 102**: Latest release (includes middleware)

[Erik Andersson]: Version 102 is currently promoted and publicly available. If we make changes now, we'd publish version 103 in private first, test it, and only promote it when ready.

### Updating Existing Versions: Not Possible In-Place

[Michal Rosikiewicz]: What happens if we want to fix something in version 101?

[Erik Andersson]: You cannot modify an existing promoted version. If you make changes, they must be published as a new version. You cannot apply changes retroactively to version 101. This is by design to prevent breaking customer flows.

### Future Enhancement: Forced Updates

[Michal Rosikiewicz]: Is there a way to force all customers to upgrade to a new version?

[Erik Andersson]: To my knowledge, there's no automatic forced update mechanism in Zapier. If we need all customers to upgrade (e.g., for a security fix), we would need to deprecate the old version or contact customers directly. We could investigate whether there's a deprecation feature in Zapier that would prompt customers to upgrade.

---

## Monitoring and Debugging

### Zapier Developer Dashboard

The Zapier platform provides a monitoring interface accessible from the developer account:

**Path**: Developer Platform → Application → Monitoring

### Available Metrics

- **Request Count**: Number of requests per action and version
- **Error Codes**: Breakdown of HTTP responses (4XX, 5XX errors)
- **Time Range Filtering**: Default last week, limited historical data

[Erik Andersson]: The interface is not as sophisticated as AWS CloudWatch. It's limited in filtering options, but it does show you which endpoints are being called and what errors are occurring.

### Current Issue: Limited Historical Data

[Lukasz Grabowski]: Can we see errors from further back than one week?

[Erik Andersson]: The interface defaults to one week. It doesn't appear to support arbitrary date range selection via drag-and-drop. If we need historical data, it's not readily available through this UI.

### Challenge: Identifying Customers Using Zapier

[Lukasz Grabowski]: Do you know how many customers use Zapier?

[Erik Andersson]: We don't have good visibility into this because the Zapier integration isn't installed as a traditional Apsis One plugin. Normally, when a customer connects an integration, it creates a record in our installation database. But for Zapier, the integration card is just a README that links to Zapier. All traffic goes through Zapier's infrastructure, so we don't get a direct installation record.

[Erik Andersson]: What we do know is that suddenly after half a year of no support tickets, we started getting multiple reports. This suggests adoption is increasing, but we don't have precise numbers.

---

## Current Customer Issue: Token Refresh Errors

### The Problem

A customer is reporting errors in their Zapier flow logs, likely related to authentication or permissions.

### Initial Diagnosis

Based on the symptoms, the issue is probably one of:

1. **Using version 100** — Missing middleware to handle token refresh gracefully
2. **Expired token** — Token refresh is failing
3. **Missing permissions** — Customer doesn't have write permission for the attributes/events they're trying to write
4. **Changed permissions** — Permissions were revoked in Apsis One after the flow was created

### Debugging Steps

[Erik Andersson]: We need to ask the customer:
1. Which version of the Apsis One Zapier app are they using? (100, 101, or 102?)
2. Can they provide a screenshot showing the app version in their flow?

**Most Likely Scenario**: If they're using version 100, they need to update to version 102 (or 101), which includes the middleware to handle token expiration transparently.

### If Version Is Already Latest

[Erik Andersson]: If they're already on version 102 and still seeing errors, then:
1. We need them to trigger the flow again to capture the error in real-time
2. We can then check the Zapier monitoring dashboard for the actual error response
3. The error is likely a permissions issue, not a middleware/token issue

[Tomasz Kowalski]: We could also check the One API logs on our side to see what requests are coming from this customer and what errors we're returning.

### Potential Non-Blocking Issue: 409 Conflict Errors

During monitoring exploration, the team noticed **409 (Conflict) errors** in the logs. Investigation revealed:

[Erik Andersson]: A 409 is being logged, but the request is being retried and succeeding. The middleware catches it and continues. This is not blocking the customer's flow, but it's creating unnecessary log noise. I can clean this up in the next version.

---

## Code Example: Triggers (Dynamic Dropdown Population)

### Sections Trigger

```javascript
// Fetch all CRM sections available to this customer
const perform = async (z, bundle) => {
  const regionUrl = bundle.authData.region; // EU or APAC
  const response = await z.request({
    url: `${regionUrl}/sections`,
    method: 'GET'
  });
  
  // Convert to Zapier format: { label: "...", value: "..." }
  return response.data.map(section => ({
    label: section.name,      // Display name
    value: section.id         // ID used in subsequent calls
  }));
};
```

### Why This Matters

[Erik Andersson]: When the user opens the dropdown, this `perform` function is called, fetches the real list of sections from the One API, and dynamically populates the dropdown. It's not a static list—it reflects the current state of their Apsis One account.

---

## Code Example: Dynamic Fields in Create Actions

### Attribute Fields Retrieval

```javascript
// Executed when "section" and "keyspace" are selected
const getAttributeFields = async (z, bundle) => {
  // Check: do we have required prerequisites?
  if (!bundle.inputData.section) {
    return []; // Don't show anything yet
  }
  
  if (!bundle.inputData.keyspace) {
    return [];
  }
  
  // Fetch attributes from One API
  const response = await z.request({
    url: `${bundle.authData.region}/attributes`,
    params: {
      section: bundle.inputData.section,
      keyspace: bundle.inputData.keyspace
    }
  });
  
  // Filter: only non-deprecated, writable attributes
  const writableAttributes = response.data.filter(attr => 
    !attr.deprecated && attr.permissions.includes('write')
  );
  
  // Convert to Zapier fields
  return writableAttributes.map(attr => ({
    key: attr.id,
    label: attr.label,
    type: attr.dataType === 'string' ? 'string' : 'text'
  }));
};
```

### How This Works in the UI

```
User selects Section: "Integration Handover"
  ↓
[Section field is populated]
  ↓
User selects Key Space: "email"
  ↓
getAttributeFields() is triggered
  ↓
Dynamic fields appear: [Email field] [First Name field] [Last Name field] ...
```

---

## Authentication Configuration (Detailed)

### Auth Data Fields

The `authentication.js` file defines what credentials are needed:

```javascript
const fields = [
  {
    key: 'region',
    label: 'API Region',
    helpText: 'Is your Apsis One account in EU or APAC?',
    required: true,
    type: 'choice',
    choices: { 'eu': 'EU', 'apac': 'APAC' }
  },
  {
    key: 'clientId',
    label: 'Client ID',
    required: true,
    type: 'string'
  },
  {
    key: 'clientSecret',
    label: 'Client Secret',
    required: true,
    type: 'password'
  }
];

// OAuth token request configuration
const oauth = {
  authorization_url: '...',
  access_token_url: '${region}/oauth/token',
  refresh_url: '${region}/oauth/token',
  scope: 'integration'
};
```

---

## Product and Business Context

### Feature Request Origin

[Lukasz Grabowski]: Who requested that we build the Zapier integration?

[Erik Andersson]: This was a request from product (Rose). It wasn't for a specific customer initially; it was a strategic decision to add Zapier to our integration portfolio.

### Go-to-Market: Sales and Professional Services Role

[Lukasz Grabowski]: Who typically shows customers how to set up Zapier flows?

[Erik Andersson]: It's typically either professional services or sometimes the customer brings in external help. It's definitely not the integration team. Professional services likely owns customer enablement for this integration.

[Lukasz Grabowski]: This is something we should expose in our marketing—that we have Zapier integration capabilities. That's a sales and professional services discussion though.

---

## Code Maintenance and Dependencies

### Code Repository Location

```
Repository: Apsis Integrations
Path: /Zapier/
```

The code is version-controlled here, separate from the Justin codebase.

### Platform Version Dependencies

[Michal Rosikiewicz]: I notice we're not using the latest Zapier platform version. The `zapier-platform-core` package is outdated. Should we update it?

[Erik Andersson]: It's not a current blocker. The app is very static—we haven't had many feature requests. The last time I updated it (from version 15 to 17), the change was trivial: just update the version number in `package.json`. Nothing else broke.

[Erik Andersson]: It could become a blocker if we go years without updates and they release multiple major versions. But we can cross that bridge when we get there. I'd recommend updating when we next make other changes to the app.

---

## Key Takeaways

1. **Zapier as a No-Code Integration Layer**: Zapier enables customers to create marketing flows triggered by external systems (Google Sheets, Microsoft Dynamics, etc.) that write data to Apsis One without custom code.

2. **Custom JavaScript App**: The Apsis One Zapier integration is a custom-built JavaScript application, not a traditional plugin. It's stored in the Apsis Integrations repository but deployed and managed separately through Zapier's platform.

3. **Three Core Capabilities**: Update Profile, Add Events, and Update Consents. These map directly to One API endpoints and respect all attribute/event permissions configured in Apsis One.

4. **Dynamic Configuration is Key**: The app uses triggers to dynamically fetch sections, key spaces, attributes, and events from the One API. This ensures customers only see options they have permission to use—preventing downstream failures.

5. **OAuth2 with Token Refresh Middleware**: Authentication uses OAuth2 with the One API. Version 101+ includes middleware to transparently handle token expiration and refresh, whereas version 100 does not—this is likely the cause of current customer issues.

6. **Version Management**: Versions are published privately for testing, then promoted for public availability. There's no in-place updating of existing versions; all changes require a new version number.

7. **Limited Observability**: We can't easily track how many customers use Zapier because there's no installation record. Adoption is increasing (support tickets were zero for 6 months, then suddenly started appearing). Monitoring is available but limited to one week of history.

8. **Customer Support Path**: The current customer issue likely requires asking them which app version they're using. If version 100, they need to upgrade. If already on latest version, we need real-time error capture or One API-side logging to diagnose.

9. **Phone Number Formatting**: A critical caveat—phone numbers must be correctly formatted before being sent to One API, or updates will fail silently.

10. **Professional Services Ownership**: Sales and professional services likely own customer enablement and go-to-market for this integration.

---

## Unresolved Questions and Action Items

### Immediate Actions

- [ ] **Contact Customer**: Ask them to provide a screenshot showing which version of Apsis One Zapier app they're using (100, 101, or 102)
  - Assigned to: Cristina (via Lukasz's message)
  - If version 100: Direct them to upgrade to latest version
  - If version 102: Proceed to next diagnostic step

- [ ] **Check One API Logs**: If customer is on latest version, review One API logs for their account to see what endpoints are being called and what errors are returned
  - Assigned to: Tomasz Kowalski

- [ ] **Verify Monitoring Dashboard**: Determine if the Zapier monitoring dashboard can show errors further back than one week, or if we need another approach

### Future Enhancements

- [ ] **Update Zapier Platform Core**: Update the `zapier-platform-core` dependency when making next changes to the app (not urgent)

- [ ] **Investigate Forced Updates**: Research whether Zapier has a mechanism to force customers to update to a newer version (relevant for critical fixes)

- [ ] **Add Tracking Header**: Explore adding a source/identifying header to token requests so we can track which customers are refreshing tokens and how often (Michal's suggestion)

- [ ] **Clean Up 409 Logging**: In next version, clean up unnecessary 409 conflict logging—these are being handled but creating noise

- [ ] **Team Access Provisioning**: Erik has already invited Lukasz, Michal, and Tomasz to the Zapier developer account for future maintenance

### Future Feature Requests

- [ ] A new feature request from CRM partners has been received (Erik mentioned this). Decision pending on whether to implement immediately or add to backlog. Lukasz to review calendar and structure work.

---

## Technical Gaps and Learning Notes

- The team collectively doesn't have deep Zapier platform expertise, but Erik is the primary expert
- Lukasz, Michal, and Tomasz are less familiar with the integration but now have access to the developer account
- The team notes that using Copilot and AI tools should ease onboarding for future Golang-related work (not directly related to Zapier but mentioned as context)