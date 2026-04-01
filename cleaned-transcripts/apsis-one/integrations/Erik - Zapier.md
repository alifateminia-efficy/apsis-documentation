---
source_file: Erik - Zapier.txt
domain: Apsis One Integrations
topics: [Zapier integration, Apsis One app architecture, OAuth2 authentication, token refresh middleware, versioning and deployment, dynamic field configuration, profile update API, consent management, event management, monitoring and debugging]
speakers: ["Erik Andersson (Integration Team)", "Lukasz Grabowski", "Michal Rosikiewicz", "Tomasz Kowalski"]
key_components: [Apsis One Zapier App, One API, OAuth2 token exchange, Zapier platform-core, FCC code repository, Zapier Developer Platform, Google Sheets trigger, Microsoft Dynamics trigger]
session_type: knowledge-transfer
---

# Apsis One Zapier Integration — Knowledge Transfer Session

## Session Overview

Erik Andersson from the Integration Team walked the incoming team (Lukasz, Michal, Tomasz) through the Apsis One Zapier integration: what it does, how it is structured, and how it is maintained. The session covered the end-user experience of setting up a Zap using the Apsis One app, a deep-dive into the JavaScript source code structure, the OAuth2 authentication flow, dynamic field population, and the publish/promote deployment lifecycle. The session also included a live debugging investigation into an active customer issue related to token expiry handling, which revealed that customers on version 1.0.0 (pre-middleware) are likely the source of the problem. Access to the Zapier developer account was transferred to the new team during the session.

---

## What the Zapier Integration Does and Why It Exists

The **Apsis One Zapier app** allows customers to connect external data sources (e.g., Google Sheets, Microsoft Dynamics, Salesforce, email applications) to Apsis One without writing custom code. It was built by the Integration Team at the request of the product team (Rose).

A typical use case: a customer has a Google Sheet of contacts. Whenever a new row is added, a Zap is triggered that creates or updates a profile in Apsis One.

The integration card for Zapier inside Apsis One is simply a **readme page that links to Zapier** — no component is installed in Apsis One itself. This means there is no installation database entry, and the team has no direct visibility into how many customers are using it.

> "All the traffic goes through Zapier in this case." — Erik Andersson

Usage tracking: Agneta and Christina are the people within the organisation who would have the best visibility into adoption. Adoption reportedly increased after a slow start, evidenced by growing support ticket volume.

---

## App Capabilities — The Three Supported Actions ("Creates")

The Apsis One Zapier app currently supports three actions, referred to as **"creates"** in Zapier terminology:

1. **Create or Update Profile** — Uses the One API `create or update profile` endpoint. Requires specifying a section, a key space, an identifier for the incoming data (e.g., CRM ID, email, phone number), and a mapping of incoming fields to Apsis One attributes.

2. **Update Consents** — Updates a subscription/consent for a profile (opt-in or opt-out). Requires specifying section, key space, and subscription. Opt-in vs. opt-out branching is left to the customer to handle in their Zap logic, because normalising all possible boolean/string representations of consent state would be intractable.

   > "We chose to put that filtering on the customer instead of us having to handle like an infinite amount of different possibilities — if it is 0 then opt in, if it is 1 then something, if it is true then something." — Erik Andersson

3. **Add Events to Profile** — Adds existing event definitions to a profile. Events **cannot be created externally** because that is not supported by the One API. The event and its field values are configured in the Zap; the profile is looked up by the configured identifier/key space.

---

## Code Repository Location and Structure

The Zapier app is a **custom-built JavaScript application** developed and uploaded to a Zapier developer account.

**Repository location:**
```
FCC code repository → Apsis Integrations → Zapier
```

It is **not** inside the Justin component tree — it is a standalone app that resides outside of Apsis proper, within the FCC code repository.

### Directory Structure

```
apsis-one/
├── creates/
│   ├── add_consent.js
│   ├── add_events.js
│   └── update_profile.js
└── triggers/
    ├── sections.js
    ├── key_spaces.js
    └── event_definitions.js
```

- **`creates/`** — Contains the three action implementations (the features listed above).
- **`triggers/`** — Despite the name, these are not workflow triggers. They are functions that **populate drop-down menus** in the Zapier UI (sections list, key spaces list, event definitions list). Each trigger makes a GET request to the One API and transforms the response into a `{ label, value }` list format that Zapier requires.

> "Triggers in Zapier is a bit of a weird name, but essentially you can consider triggers as the different values you can pick from the drop down." — Erik Andersson

---

## Authentication Flow — OAuth2 via One API Credentials

Authentication uses the **One API OAuth2 client credentials flow**.

### User-Facing Configuration Fields

When a customer connects their Apsis One account in Zapier, they fill in:
- **One API Region** — `EU` or `APAC` (controls which base URL is used for all subsequent API calls)
- **Client ID**
- **Client Secret**

Zapier then makes a token exchange request to the One API OAuth token endpoint. If a valid access token is returned, the connection is marked as successful.

### How Region Is Used at Runtime

The region value is stored in the auth data and referenced at runtime in every trigger and create function:

```javascript
// Conceptual example from sections trigger
const region = bundle.authData.one_api_region;
const response = await z.request({
  method: 'GET',
  url: `https://${region}-api.apsis.one/...`
});
```

### Token Refresh Middleware (Added in v1.0.1)

A **middleware function** was added that intercepts every API response. If the response is a `403` containing a specific error message indicating the token has expired, it throws a `RefreshAuthError`. This causes Zapier to:
1. Automatically request a new access token
2. Retry the failed request

**This middleware was NOT present in v1.0.0.** Without it, an expired token results in a visible, unhandled error in the customer's Zap history — even though no data is actually lost, since the operation would eventually succeed on retry. This is the most likely cause of the current customer support issue.

> "The customer's app would make a request to you with an expired token and because we didn't have this middleware, this resulted in a big red error in their logs." — Erik Andersson

---

## Dynamic Field Population in the "Create or Update Profile" Action

The update profile action demonstrates Zapier's **dynamic fields** pattern:

### Static Fields (always shown)
- Section selector (drop-down populated by `sections` trigger)
- Key space selector (drop-down populated by `key_spaces` trigger)
- Identifier field (which attribute to use as the profile key, e.g., CRM ID, email)

These static fields have the property `altersDynamicFields: true`, meaning a change to any of them causes the dynamic fields to reload.

### Dynamic Fields (`getAttributeFields` function)
- Only executes after a section has been selected (guards against calling the One API without enough context)
- Makes a GET request to One API to retrieve all attributes for the selected section
- Filters the results: only displays attributes that are **not deprecated** AND where the API key has **write permission** for that key space
- The resulting list is rendered as configurable input fields — one per attribute — allowing the customer to map incoming data fields to Apsis One attributes

The same pattern applies to **`getEventsFields`** for the "Add Events" action: first pick which event definition to add, then the fields for that event are dynamically populated.

### Caveat: Phone Number Formatting

> "One caveat with this is of course phone numbers — the customer needs to make sure that the phone numbers are correctly formatted or those will not be able to be updated." — Erik Andersson

### Note on Event Definitions Permission Filtering

The event definitions trigger filters out events where the API key does not have write permission. This relies on a One API endpoint that was specifically requested and implemented by the backend team to expose event-level permissions.

> "We could get the key space permissions for attributes, we could not get it for events. But because you were kind enough to add that for us, we can utilise that here in Zapier and only display events to the customer where they have write permission for — otherwise every request in Zapier would fail with a permission denied error." — Erik Andersson

---

## Versioning, Publishing, and Deployment Lifecycle

The Zapier developer platform uses a **publish → promote** two-step deployment model.

### Current Versions
- **1.0.0** — Initial release; no token refresh middleware. **Do not encourage customers to stay on this version.**
- **1.0.1** — Added token refresh middleware.
- **1.0.2** — Current promoted/public version (confirmed as latest during session).

### Deployment Steps

1. **Develop** — Make changes locally in the FCC repository.
2. **Publish** — Upload and create a new version (e.g., 1.0.3). This version is **not public**. It is only visible to users who are members of the Zapier developer project.
3. **Test** — Internal team and stakeholders (e.g., Agneta, Rose) can test using the unpublished version.
4. **Promote** — Makes the version public. All Zapier users can now find and select it.

> ⚠️ **Important:** Promoting a new version does **not** automatically migrate existing customers. Customers must manually update their Zap nodes to use the new version. There is currently no known mechanism to force-update all customers.

> ⚠️ **Important:** Once a version is promoted and public, you **cannot make in-place changes** to that version, as this would break existing customer flows.

### Idea: Adding Source Header for Tracking

[Michal Rosikiewicz] raised the idea of adding a `source: zapier` header to authentication requests so One API logs could be used to track Zapier usage and identify customers.

[Erik Andersson] confirmed this is technically feasible but would require publishing a new version (e.g., 1.0.3). Adding it to already-promoted versions is not possible.

> "It would not be a very big thing to publish a new version that has that in the header." — Erik Andersson

This remains an open action item.

### Platform Core Version

[Michal Rosikiewicz] noted a warning in the developer console that the **Zapier platform-core** package is not at the latest version. Erik's assessment:

- This is not currently a blocker.
- The last update only required changing a version number in the dependency (went from platform-core 15 to 17 with no other code changes).
- It could become a blocker if the app goes unmaintained for a long time and multiple major versions accumulate.

---

## Monitoring and Observability

The Zapier developer console has a **Monitoring** section that shows:
- Request counts per version
- Error breakdown by HTTP status code (4xx, 5xx)
- Limited to approximately **one week** of history

### Limitations Discovered During Session
- The date range for monitoring data is limited to roughly one week — cannot query one month back.
- Error counts may be displayed (e.g., "11 404XX errors") but clicking through sometimes shows no detail — apparent UI bug or data lag in Zapier's console.
- No free-text search or filter by specific status code (e.g., cannot filter to show only 403s).

### 409 Conflict Errors Observed in Logs
During the session, 409 Conflict responses were visible in the monitoring logs. Erik confirmed this is a known, non-critical issue:

> "If it is 201 or 409, then we proceed. We are just logging it. This is something I can clean up for logs' sake, but it is not a problem per se." — Erik Andersson

The app does not treat 409 as a failure — it proceeds regardless. However the log noise could be cleaned up in a future version.

### One API Logs as Alternative Debugging Tool
Because Zapier's monitoring is limited, the One API logs (accessible by the backend team) can serve as a complementary debugging source. When a customer reports errors, the backend team can look up what endpoints were being called and what responses were returned.

---

## Active Customer Issue — Token Expiry Errors

### Problem Description
A customer is experiencing visible errors in their Zap history. The suspected root cause is that they are using **version 1.0.0**, which lacks the token refresh middleware.

### Debugging Steps Taken During Session
- Checked Zapier monitoring logs — found evidence of access token expiry events (consistent with normal token refresh cycles, not indicative of a broken flow).
- Could not confirm which version the customer is using from logs alone.
- Could not find a 403 in the monitoring view within the available one-week window.

### Resolution Path

1. **Contact the customer (via Cristina)** and ask them to provide a screenshot of their Zap showing which version of the Apsis One app node they are using (1.0.0, 1.0.1, or 1.0.2).
2. **If they are not on 1.0.2**: ask them to upgrade the node to 1.0.2. This should resolve the issue.
3. **If they are already on 1.0.2**: ask them to trigger the Zap so that errors appear in the monitoring console within the one-week window, allowing the team to see at which step the failure occurs.
4. **Additional possibility**: the customer may have had correct write permissions when the Zap was set up, but permissions were subsequently changed inside Apsis One. If so, this is outside the Zapier integration's control and must be corrected in Apsis One.

---

## Access Handover

During the session, Erik invited the new team members to the **Zapier developer project** (the administrative account where the Apsis One app is managed). Benjamin (previous team member) no longer has access.

> ⚠️ Confirm that invites were received and accepted by Lukasz, Michal, and Tomasz.

---

## Key Takeaways

1. The Apsis One Zapier app is a **custom JavaScript app** maintained by the Integration Team, stored in the **FCC code repository under `Apsis Integrations/Zapier`**, not in Justin.
2. It supports three actions: **create/update profile**, **update consents**, and **add events to profile**.
3. Authentication uses **One API client credentials OAuth2**. Region (EU/APAC) must be specified by the user and determines all API endpoint base URLs.
4. The **token refresh middleware** (added in v1.0.1) is critical. Customers on v1.0.0 will see spurious errors when access tokens expire. The middleware catches the 403 and triggers a transparent retry.
5. Deployment follows a **publish → promote** model. New versions are not public until promoted, and promotion does **not** auto-migrate existing customers.
6. There are currently **three versions** live: 1.0.0, 1.0.1, 1.0.2. Customers should be on 1.0.2.
7. Monitoring in the Zapier console is limited to ~one week of history with limited filtering capability. **One API logs are a useful complement** for debugging customer issues.
8. Dynamic fields (attribute lists, event lists) are **filtered at runtime** by write-permission to avoid surfacing options that would result in permission-denied errors.
9. Phone number formatting is the customer's responsibility — no normalisation is performed by the integration.

---

## Unresolved Questions and Action Items

| Item | Owner | Notes |
|------|-------|-------|
| Confirm which Zapier app version the customer with the active support issue is using | Lukasz / Cristina | Ask customer for screenshot of their Zap node showing version number |
| If customer is on v1.0.0 or 1.0.1, ask them to upgrade to 1.0.2 | Cristina / customer | Should resolve token expiry error visibility |
| Investigate feasibility of adding `source: zapier` header to token requests for usage tracking | Michal / Erik | Would require publishing v1.0.3; also explore whether Zapier supports force-migrating customers to new versions |
| Confirm team access to Zapier developer account (invites sent during session) | Lukasz, Michal, Tomasz | Verify receipt of invites |
| Investigate whether there is a Zapier mechanism to force-update all customers to a new version | TBD | Would be valuable for non-breaking changes like adding a source header |
| Consider updating `zapier-platform-core` package version | TBD | Not blocking now; last upgrade (v15→v17) required only a version bump in dependencies |
| Clean up 409 Conflict log noise in the app code | Erik | Not breaking, but adds unnecessary noise to monitoring logs |
| Scope and schedule new CRM partner feature request | Lukasz + Erik | Erik noted it is straightforward and high value; pending Lukasz reviewing the calendar |