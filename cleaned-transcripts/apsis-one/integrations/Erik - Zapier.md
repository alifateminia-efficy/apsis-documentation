---
source_file: Erik - Zapier.txt
domain: Apsis One Integrations
topics: [Zapier Integration Architecture, Authentication and OAuth2 Flow, Dynamic Field Configuration, App Versioning and Deployment, Middleware for Token Refresh, Monitoring and Debugging, Customer Issue Troubleshooting]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Apsis One Zapier App, Zapier Platform, One API, OAuth2 Token Exchange, Create/Update Profile Endpoint, Add Events to Profile, Update Consents, Google Sheets, Microsoft Dynamics]
session_type: knowledge-transfer
subdomains: [Zapier Integration, Architecture]
---

## Session Overview

This knowledge transfer session covers the Apsis One Zapier integration, a custom-built JavaScript application that enables customers to create marketing automation flows between Zapier and Apsis One. Erik Andersson walked through the integration's architecture, authentication mechanism, dynamic field population, versioning and deployment process, and troubleshooting a customer issue involving token refresh behavior. The session included hands-on demonstration of the integration UI, code structure, monitoring capabilities, and best practices for development and deployment.

## Zapier Integration Overview

### Purpose and Use Cases

The Apsis One Zapier integration functions as a marketing automation tool that allows customers to trigger Apsis One actions based on events from external systems. Rather than managing profiles and workflows directly in Apsis One, customers can leverage existing data sources connected to Zapier.

**[Erik Andersson]:** The integration concept is similar to marketing automation flows, but executed through integrations from other systems instead of native Apsis profiles.

### High-Level Flow Example

A typical workflow involves:

1. **Trigger**: Data arrives in an external system (e.g., Google Sheets row added, Microsoft Dynamics contact created)
2. **Action**: Apsis One Zapier app processes the data and updates profiles or consents in Apsis One

**[Erik Andersson]:** With premium Zapier accounts, customers can trigger on events like contacts added in Microsoft Dynamics, leads added in Salesforce, or email application events. Non-paying users are limited to simpler triggers like Google Sheets rows.

### Supported Operations

The Apsis One Zapier app currently supports three primary actions:

- **Create or Update Profile**: Add or modify profile attributes using the One API `create_or_update_profile` endpoint
- **Add Events to Profile**: Append events to existing profiles (external event creation is not supported by the One API)
- **Update Consents**: Modify subscription/consent status for profiles (opt-in or opt-out)

## Authentication Mechanism

### OAuth2 Flow Configuration

The Zapier app implements OAuth2 token exchange with One API. Authentication requires:

1. **Region Selection**: Customer specifies EU or APAC region
2. **Credentials**: Client ID and Client Secret from One API

**[Erik Andersson]:** The app extracts the region from authentication data and uses it to construct the correct One API endpoint. Zapier exchanges the client credentials for an access token using the One API credential endpoint.

### Token Management

Tokens have finite lifespans. When an access token expires during a request, the middleware (discussed below) intercepts the 403 error and triggers Zapier's built-in refresh flow:

1. Middleware detects 403 with specific error message indicating expired token
2. Throws `RefreshOAuthError` to signal Zapier
3. Zapier makes new request to OAuth token endpoint
4. Fresh access token obtained
5. Original failed request retried with new token

**[Erik Andersson]:** This middleware was added in version 101 because the initial version 100 didn't handle expired tokens gracefully, resulting in visible errors for customers even though the system was working correctly.

## Dynamic Field Population and Configuration

### Static vs. Dynamic Fields

The Zapier app uses both static and dynamic fields to guide customer configuration:

**Static Fields** (always visible):
- Section (CRM context)
- Key space (attribute domain)
- Identifier type (profile key: email, phone, CRM ID, etc.)

**Dynamic Fields** (shown conditionally):
- Available attributes for write
- Available event definitions
- Available subscriptions/consents

### Trigger System for Dynamic Dropdowns

The codebase uses **triggers** to populate dynamic dropdowns. Triggers are JavaScript functions that fetch data from One API endpoints and format responses for Zapier's UI.

**[Erik Andersson]:** Triggers are different from the marketing automation concept of "triggers." In Zapier code, triggers are functions that populate dropdown menus. If I'm configuring a section dropdown, a section trigger executes to fetch all available sections.

### Key Triggers Implemented

**Sections Trigger** (`triggers/sections.js`):
```
GET request to One API sections endpoint
Returns: [{name: "Section Name", id: "section-id"}, ...]
```

**Key Spaces Trigger** (`triggers/keyspaces.js`):
```
Fetches all accessible key spaces
Returns: [{name: "Email", id: "email"}, ...]
```

**Event Definitions Trigger** (`triggers/event_definitions.js`):
```
GET request to One API event definitions endpoint
Filters: Only returns events where customer has write permission
Reason: Prevents displaying events they cannot write to
Returns: [{name: "Event Name", id: "event-id"}, ...]
```

**[Erik Andersson]:** This filtering logic was specifically requested. We asked the Apsis One team to add an endpoint that returns permission metadata for events because previously we could retrieve event definitions but couldn't check write permissions, causing all customer requests to fail when they tried to write to restricted events.

**Attributes Trigger** (`get_attributes_fields()`):
```
Conditional function - only executes after section is selected
GET request to One API attributes endpoint for that section
Filters: 
  - Only non-deprecated attributes
  - Only attributes with write permission for selected key space
Returns: Dynamic form fields for each writable attribute
```

**[Erik Andersson]:** This is important because we need to know the section first. If we don't have a section, we can't call the attributes endpoint. So this function only runs after you've selected a section.

### Configuration Workflow Example

**Updating a Profile:**
1. Customer selects section (e.g., "Integration Handover")
2. Attributes trigger fires, fetches all writable attributes for that section
3. Customer selects key space (e.g., "Email")
4. Customer selects identifier type (e.g., profile key = email address)
5. Customer maps incoming data to target attributes:
   - Incoming "CRM_ID" → writes to "CRM ID" attribute
   - Incoming "email" → writes to "Email" attribute
   - Static value "newsletter-signup" → writes to some field

**Adding Consent:**
1. Section and key space configuration as above
2. Subscription trigger fetches available subscriptions for selected section/key space
3. Customer selects subscription name
4. Customer specifies opt-in or opt-out value
5. Incoming data mapped to consent value

**[Erik Andersson]:** If a customer wants to handle both opt-in and opt-out in one flow, they must branch the flow in Zapier and compare the incoming consent value. We put the filtering responsibility on the customer because we cannot know all their possible consent value formats (0=opt-in, 1=opt-out vs. true=opt-in, false=opt-out, etc.).

## File Structure and Code Organization

### Repository Location

```
Apsis Integrations / Zapier
```

The codebase resides within the FC (Efficy corporate) repository but **outside** the Justin component structure, as it is a completely separate application not part of the core Apsis One product.

**[Erik Andersson]:** It's separate because it's not a Justin component. It's an external application that lives in the FC code repository.

### Directory Structure

```
apsis-one/
├── creates/              # Action definitions (what the app can do)
│   ├── add_consent.js
│   ├── add_events.js
│   └── update_profile.js
├── triggers/             # Dynamic dropdown data loaders
│   ├── event_definitions.js
│   ├── key_spaces.js
│   ├── sections.js
│   └── subscriptions.js
├── authentication.js     # OAuth2 configuration
└── middleware.js         # Error handling and token refresh
```

### Create Actions Structure

Each action file (e.g., `creates/update_profile.js`) contains:

```javascript
{
  key: "update_profile",
  noun: "Profile",
  display: { label: "Update Apsis One Profile" },
  operation: {
    // Static fields always shown
    inputFields: [
      { key: "section", ... },
      { key: "keyspace", ... },
      { key: "identifier", ... },
      // Dynamic field - function fetches available attributes
      { key: "get_attributes_fields", altersDynamicFields: true }
    ],
    perform: (z, bundle) => {
      // Main function - executes when action runs
      // Uses section, keyspace, identifier from configuration
      // Posts data to One API update_profile endpoint
    }
  }
}
```

**[Erik Andersson]:** The `perform` function is what Zapier actually executes when the action runs. That's the main entry point.

### Middleware for Error Handling

A critical addition to version 101 is error handling middleware:

```javascript
// All responses pass through middleware
if (response.status === 403 && 
    response.body.includes("user not authorized to access these resources")) {
  throw new RefreshOAuthError(z);
}
```

**[Erik Andersson]:** When One API returns a 403 with that specific message, it means the access token expired. We throw `RefreshOAuthError`, which signals Zapier to refresh the token and retry. In version 100, we didn't have this, so customers saw big red errors even though the system was working correctly. Every request that hit an expired token would show an error in their logs.

## Versioning and Deployment

### Three-Step Version Lifecycle

**1. Development (Local)**
- Engineer develops feature locally
- Code is built and tested locally

**2. Publishing (Private)**
```
Developer → Upload to Zapier account → Publish version 102
```
- New version created (auto-incremented)
- Version available only to team members with project access
- Not visible to public customers
- Can be tested, fixed, re-published without affecting customers

**[Erik Andersson]:** Publishing is different from promoting. When you publish, it creates a version number but it's still private to your project.

**3. Promotion (Public)**
```
Promote version 102 → Available to all customers
```
- Makes version available in public Zapier marketplace
- Customers can now add the app to new flows or switch existing flows to new version

### Customer Update Mechanism

**Critical caveat**: Existing customer flows do **not** automatically update to new versions.

**[Michal Rosikiewicz]:** If we fix a bug in version 101 and release version 102, does the customer automatically get it?

**[Erik Andersson]:** No. The customer's existing nodes stay on version 101. They must manually reconfigure their flow node to use version 102. We can deprecate old versions to encourage migration, but we cannot force it without risking breaking their workflows.

### Impact of Updates

When making breaking changes, a new version is required:
- Cannot modify version 101 after promotion (would break all existing flows)
- New version number allows parallel operation: some customers on 101, some on 102
- Backups-compatible changes (like adding header tracking) can be pushed as new version
- Non-backwards-compatible changes require customer intervention

## Current Support Limitations

### Lack of Native Usage Tracking

Unlike other integrations where Apsis One logs connections via the "Connect" button in the admin UI, Zapier integration is **not tracked internally**.

**[Erik Andersson]:** The Zapier integration card is just a README that links to Zapier. There's no install button in Apsis One, so no row is created in our installation database. All traffic goes through the customer's Zapier account.

**[Lukasz Grabowski]:** Do you know how many customers use Zapier?

**[Erik Andersson]:** I don't have direct metrics because we don't control the connection point. Usage increased significantly after a period of inactivity, as evidenced by support tickets appearing.

### Who Manages Customer Configuration

The integration is primarily managed by:
- **Professional Services** or **external consultants** (initial setup)
- **Customers** (ongoing management within Zapier)
- **Integration team** (app code and bug fixes)

**[Lukasz Grabowski]:** Should we expose this capability in our marketing?

**[Erik Andersson]:** Possibly, but coordination with Sales and Professional Services would be needed to ensure they can support customer implementations.

## Debugging Customer Issue: Token Refresh Errors

### Problem Statement

A customer reported large numbers of errors in their Zapier logs. The error indicated authentication failures during flow execution.

### Investigation Process

**Step 1: Version Check Required**

The first diagnostic is determining which app version the customer is using (100, 101, or 102).

- **Version 100**: Lacks middleware; expired tokens show as hard errors
- **Version 101+**: Includes middleware; expired tokens handled transparently with automatic retry
- **Latest**: Version 102 (or newer when promoted)

**[Erik Andersson]:** If they're using version 100, the solution is simple: update to version 101 or 102. The errors are actually being handled; they're just being logged as failures.

### Root Causes of Failures (Priority Order)

1. **Expired/Invalid Access Token**
   - Symptom: 403 with "user not authorized" message
   - Fixed in v101+ via middleware
   - Previous behavior: Hard error that blocks flow execution

2. **Missing Write Permissions**
   - Symptom: 403 or 401 responses
   - Cannot be fixed by updating app version
   - Requires action: Verify Apsis One key space and attribute write permissions are configured
   - Customer may have changed permissions after flow setup

3. **Incorrect Event Write Permissions**
   - Symptom: Requests fail when trying to add specific events
   - Fixed: App filters events by permission in v101+
   - Version 100: Would show all events, but fail at execution time if no permission

4. **Incorrect Subscription/Consent Mapping**
   - Symptom: Consent updates fail
   - Cause: Customer's incoming consent value doesn't match expected opt-in/opt-out format
   - Fixed in v101+: App only shows subscriptions customer has permission for

5. **Invalid Phone Number Format**
   - Symptom: Phone number updates fail silently
   - Cause: Phone numbers must be properly formatted per One API requirements
   - Note: No validation in app; validation happens at API level
   - **[Erik Andersson]:** One caveat: phone numbers need to be correctly formatted or they won't update. The customer is responsible for formatting.

### Monitoring and Logging

**Zapier Developer Dashboard** provides request/error monitoring:

```
Zapier Dashboard → Monitoring → Request logs
```

Available filters:
- Date range (limited to ~1 week in UI)
- HTTP status code (200s, 400s, etc.)
- Error categories

Limitations:
- UI is not as sophisticated as CloudWatch
- Limited date range selection (dragging/date selection didn't work as expected)
- Errors logged but sometimes not detailed enough to identify root cause

**[Lukasz Grabowski]:** Can we look back one month for errors?

**[Erik Andersson]:** Only one week is available in this interface. The monitoring tools are basic compared to CloudWatch.

### Observation from Investigation

During the session, monitoring showed:
- Version 100: Several 404/409 status codes in recent week
- 404 errors: Present but details not retrievable from UI
- 409 Conflict errors: Appeared in logs but didn't match reported issue
- Access token refresh events visible in logs

**[Erik Andersson]:** We saw some 409 conflict errors which is weird, but that's a separate issue I should fix. The errors we're looking for would be 403s related to token expiration.

### Next Steps to Resolve

1. **Contact customer** via Christina (support liaison):
   - Request screenshot showing Zapier node version
   - Or ask them to re-trigger the flow so error appears in Zapier monitoring

2. **If version is 100**:
   - Direct customer to update to version 101 or 102
   - Explain that middleware now handles token refresh transparently

3. **If version is 101+**:
   - Check Apsis One API logs for the actual error
   - Verify key space and attribute write permissions
   - Verify subscription/consent configurations
   - Confirm phone numbers are properly formatted (if applicable)

**[Erik Andersson]:** The tricky part is that either the customer hasn't configured the right write permissions, they're using expired credentials, or they're trying to write to attributes/subscriptions/events they don't have permission for. It could be many things, which is why we need to see the actual error.

## Code Structure: Detailed Walkthrough

### Create Action: Update Profile

**File**: `creates/update_profile.js`

**Static Configuration Block:**
```javascript
{
  key: "update_profile",
  noun: "Profile",
  display: {
    label: "Update Apsis One Profile",
    description: "Updates an existing profile with new attribute values"
  }
}
```

**Input Fields with Dynamic Loading:**

The action defines both static and dynamic input fields:

```javascript
operation: {
  inputFields: [
    // Static fields - always visible
    {
      key: "section",
      label: "Section",
      required: true,
      helpText: "The CRM section to write to",
      search: "section_trigger"  // populated by triggers/sections.js
    },
    {
      key: "keyspace",
      label: "Key Space",
      required: true,
      helpText: "Attribute namespace (Email, Phone, etc.)",
      search: "keyspace_trigger"
    },
    {
      key: "identifier",
      label: "Identifier (Profile Key)",
      required: true,
      choices: ["email", "phone", "crm_id", ...],
      helpText: "Which field uniquely identifies this profile"
    },
    // Dynamic fields - shown after section is selected
    {
      key: "attributes",
      label: "Attributes to Update",
      dynamic: "get_attributes_fields.id.label",  // depends on section
      altersDynamicFields: true  // changing this re-triggers dynamic fields
    }
  ]
}
```

**The Perform Function** (main execution logic):

```javascript
perform: (z, bundle) => {
  const section = bundle.inputData.section;
  const keyspace = bundle.inputData.keyspace;
  const identifier = bundle.inputData.identifier;
  const profileKey = bundle.inputData.profile_key_value;  // incoming email, phone, etc.
  
  // First: verify profile exists (or create if doesn't exist)
  const verifyUrl = `${z.env.apsis_one_url}/rest/v2/${section}/profiles?${identifier}=${profileKey}`;
  z.request({ url: verifyUrl });
  
  // Second: build attribute update payload from incoming data
  const attributeMap = {};
  Object.keys(bundle.inputData).forEach(key => {
    if (key.startsWith("attr_")) {
      // Map incoming field to attribute name
      attributeMap[key] = bundle.inputData[key];
    }
  });
  
  // Third: POST to update_profile endpoint
  return z.request({
    url: `${z.env.apsis_one_url}/rest/v2/${section}/profiles/${keyspace}/${profileKey}`,
    method: "POST",
    json: attributeMap
  }).then(response => {
    if (response.status === 201 || response.status === 409) {
      return response.json;  // Success
    }
    throw new Error(`Failed to update profile: ${response.status}`);
  });
}
```

**[Erik Andersson]:** We accept both 201 (created) and 409 (conflict) as success because 409 means the profile already existed. We just log it and continue.

### Trigger: Event Definitions

**File**: `triggers/event_definitions.js`

This trigger populates the "Event to Add" dropdown in the `add_events` action:

```javascript
perform: (z, bundle) => {
  const section = bundle.inputData.section;
  const keyspace = bundle.inputData.keyspace;
  
  if (!section) {
    return [];  // Can't fetch without knowing the section
  }
  
  // GET all event definitions for this section
  const response = z.request({
    url: `${z.env.apsis_one_url}/rest/v2/${section}/events`,
    headers: {
      "Authorization": `Bearer ${bundle.authData.access_token}`
    }
  });
  
  // Filter to only events we have write permission for
  return response.json.events
    .filter(event => {
      // Check permission for this keyspace
      return event.permissions[keyspace]?.includes("write");
    })
    .map(event => ({
      label: event.name,
      value: event.id
    }));
}
```

**[Erik Andersson]:** The key insight here is that we filter by write permission. Without this filtering, every event would be offered to the customer, but they'd get failures when Zapier tried to write to restricted events.

### Authentication Configuration

**File**: `authentication.js`

Defines OAuth2 token exchange:

```javascript
const authentication = {
  type: "oauth2",
  config: {
    // User specifies region (EU or APAC)
    authorize: {
      url: "https://login.apsis1.com/oauth/authorize"  // template varies by region
    },
    refresh: {
      url: "https://api.apsis1.com/oauth/token",  // varies by region
      params: {
        client_id: "{{bundle.authData.client_id}}",
        client_secret: "{{bundle.authData.client_secret}}",
        grant_type: "client_credentials"
      }
    }
  },
  connectionLabel: (z, bundle) => bundle.authData.client_id
};
```

**Input Fields for Authentication:**
```javascript
fields: [
  {
    key: "region",
    type: "choice",
    choices: ["EU", "APAC"],
    required: true,
    helpText: "Which region is your Apsis One instance?"
  },
  {
    key: "client_id",
    type: "string",
    required: true,
    helpText: "From One API management"
  },
  {
    key: "client_secret",
    type: "string",
    required: true,
    sensitive: true  // Don't echo in logs
  }
]
```

**[Erik Andersson]:** The region field is critical because the token endpoint URL changes. If EU, it's one URL; if APAC, it's another.

### Middleware: Token Refresh Handling

**File**: `middleware.js`

Intercepts all API responses to handle expired tokens:

```javascript
middleware: (request, z, bundle) => {
  return request(z, bundle).catch(error => {
    const response = error.response;
    
    // Check for expired token error from One API
    if (response.status === 403 && 
        response.body.message.includes("user not authorized to access these resources")) {
      
      // Signal Zapier to refresh the token and retry
      throw new z.errors.RefreshOAuthError(z);
    }
    
    // Re-throw other errors
    throw error;
  });
}
```

**Behavior**:
1. Request is made with current access token
2. If response is 403 with the specific error message:
   - Middleware throws `RefreshOAuthError`
   - Zapier intercepts this error
   - Zapier calls OAuth refresh endpoint to get new token
   - Zapier automatically retries the original request
   - Second attempt succeeds (in most cases)
3. If response is any other error, it propagates normally

**[Erik Andersson]:** This was a game-changer in version 101. Without it, customers saw errors for every expired token, which made their logs look terrible. With it, token expiration is handled silently and only logged if the retry also fails.

## Ownership and Origin

### Team Responsible

**Integration Team** at Apsis built and maintains the Zapier app.

**[Erik Andersson]:** We built this Apsis One app as a custom JavaScript application. We develop it locally, upload it to Zapier, and publish it.

### Product Request

The integration was originally requested by **Product (Rose)** as a feature request.

**[Erik Andersson]:** It was a request from Rose from the product team, not from a specific customer.

### Knowledge Holders

- **Erik Andersson**: Primary developer and maintainer
- **Agneta and Christina**: Originally involved in development/deployment process

## Maintenance and Technical Considerations

### Platform Dependency

The app depends on `zapier-platform-core` as specified in `package.json`.

**Current Status**: Using version 17 (as of last update), but the latest may be newer.

**[Michal Rosikiewicz]:** We have a warning that we're not using the latest platform version. Should we update?

**[Erik Andersson]:** Not critical. The app is very static with few feature requests. Last time I updated it (adding middleware), I went from version 15 to 17 just by changing the dependency number. No breaking changes.

### Low Feature Request Volume

Maintenance is minimal because:
- App supports the three core operations needed (update profile, add events, add consent)
- Customers can handle complex logic within Zapier itself
- Most requests are handled by professional services or customer integration teams

**[Erik Andersson]:** This app doesn't get heavy feature requests. Last time we did significant work was when we added the middleware. Everything else has been stable.

## Future Considerations and Open Questions

### Potential Enhancements

1. **Source Tracking Header**
   - **Proposal** [Michal Rosikiewicz]: Add custom HTTP header to identify Zapier requests in Apsis One API logs
   - **Value**: Track Zapier integration usage and token refresh frequency
   - **Implementation**: Add to OAuth token request and/or API calls
   - **Trade-off**: Would require new app version

2. **Customer Update Mechanism**
   - **Challenge**: How to force customers onto new versions without breaking flows?
   - **Options**:
     - Deprecate old versions (customers must manually switch)
     - Automated version bump (risky)
     - Deprecation notice (graceful)
   - **Status**: Not yet implemented

3. **Platform Core Version Upgrade**
   - **Status**: Optional, not blocking
   - **Effort**: Minimal (update package.json)

### Known Gaps

1. **Usage Analytics**
   - Cannot easily track how many customers use Zapier
   - No built-in installation metrics
   - Only visible through support tickets and proactive inquiry

2. **Permission Error Clarity**
   - Difficult to distinguish between token expiration, missing write permissions, and invalid consent mappings
   - Errors often reported as generic failures

3. **Monitoring UI Limitations**
   - Zapier's built-in monitoring has limited date range and filtering
   - Complex to debug historical issues

## Key Takeaways

1. **The Apsis One Zapier app is a custom-built JavaScript application**, not a built-in feature, maintained by the Integration Team and residing in the FC code repository.

2. **Three core operations are supported**: create/update profile, add events to profile, and update consents. Each requires careful configuration of section, key space, and identifier fields.

3. **Dynamic field population is critical**: The app uses Zapier's trigger system to fetch available sections, key spaces, attributes, events, and subscriptions from One API, filtering by customer write permissions to prevent runtime failures.

4. **OAuth2 token refresh is handled via middleware**: Version 100 lacked proper error handling for expired tokens, causing visible errors. Version 101+ added middleware to catch 403 errors and trigger automatic token refresh and retry.

5. **Versioning is non-disruptive**: Publishing creates a new version for internal testing, but promotion makes it public. Existing customer flows must manually upgrade; automatic updates do not occur.

6. **Customer configuration is customer-owned**: Professional Services or external consultants typically set up the initial flow, but configuration decisions (opt-in/opt-out values, attribute mapping, consent definitions) rest with the customer.

7. **Usage is not directly tracked**: Unlike native Apsis One integrations, Zapier connections don't appear in Apsis One's installation database, making it difficult to measure adoption.

8. **Debugging requires version and error context**: When customers report failures, the first step is determining app version (100 vs. 101+) and what error appears in Zapier's monitoring. Permission and token issues require API-level investigation.

9. **Permission filtering prevents runtime errors**: By filtering events and attributes by write permission, the app prevents customers from configuring flows that would fail at execution time.

10. **Phone number formatting is customer's responsibility**: The app does not validate phone number format; One API rejects improperly formatted numbers at update time.

## Unresolved Questions and Action Items

### Outstanding Debugging Issue

**Status**: In Progress

- **Problem**: Customer reporting large error counts in Zapier logs (session referenced Christina contacting customer)
- **Next Step**: Determine customer's app version (100 vs. 101 vs. 102) via screenshot or re-trigger
- **Expected Outcome**: If version 100, customer upgrades to 101+ and errors resolve. If version 101+, deeper investigation needed.
- **Owner**: Christina (support) + Erik (technical investigation if needed)

### Potential Enhancements Under Consideration

1. **Source Header for Tracking** [Michal Rosikiewicz]
   - Proposal: Add custom HTTP header to OAuth token requests to identify Zapier calls in One API logs
   - Benefit: Measure integration usage and token refresh frequency
   - Effort: Requires new app version

2. **Customer Update Strategy**
   - Question: Can we force customers to use new versions or automate deprecation?
   - Status: To be explored (Michal mentioned discovering Zapier's capabilities)

3. **Platform Core Update**
   - Question: Should we proactively update `zapier-platform-core` to latest version?
   - Status: Not urgent; requires testing to confirm no breaking changes

---

**Session recorded**: November 18, 2025  
**Duration**: 1h 22m 42s