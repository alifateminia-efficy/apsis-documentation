---
source_file: "Apsis x Membrane technical discussion.txt"
domain: Apsis One Integrations
topics: [authentication, workspace configuration, tenant management, CRM connections, field mapping, subscription mapping, full sync, webhooks, real-time event flows, pagination, rate limits, retry logic, custom connectors, CI/CD, POC planning]
speakers: ["Lukasz Grabowski (Apsis, lead engineer)", "Vlad Ursul (Membrane, solutions engineer)", "Kyle (Membrane, account/sales)", "Michal (Apsis, mentioned/minor contributor)", "Daniel (Apsis, mentioned)"]
key_components: [Membrane platform, Apsis One, Workspace API, Tenant/Customer objects, Universal Actions, Connection-specific Actions, Flows, Field Mapping, CLI, Membrane Agent, Internal API configuration, Credentials store]
session_type: knowledge-transfer
---

## Session Overview

This session is a deep technical discussion between the Apsis engineering team and Membrane's solutions engineer (Vlad) covering the full integration architecture between Apsis One and the Membrane integration platform. Topics span authentication in both directions (Apsis→Membrane and Membrane→Apsis), tenant/workspace modeling, CRM connection management, field and subscription mapping strategies, full sync mechanics with pagination, real-time event flows via webhooks/polling, rate limit handling, retry logic, and custom connector development. The session closes with a discussion of POC scope, CI/CD support, and next steps. The primary goal is evaluating whether Membrane can replace or underpin Apsis's existing CRM integration layer (currently supporting HubSpot, Salesforce, Zoho, Pipedrive, and others).

---

## Workspace and Tenant Architecture

### Workspace as Environment

Membrane **workspaces** are isolated environments. The typical usage pattern is one workspace per environment (dev, staging, production). Each workspace has a **workspace key** and **workspace secret**.

> "Each workspace is like an environment, right? Usually that's how people use them."

Apsis will need **one workspace per environment** (at minimum: staging and production).

### Tenant Model

In Membrane terminology, a **tenant** corresponds to an Apsis customer account. Membrane calls this object "customer" in some places but is in the process of renaming it to "tenant."

- Tenants can represent end users (e.g., employees of a customer) or organizations (e.g., customer accounts), depending on business model.
- **Apsis use case:** each Apsis customer account = one Membrane tenant.
- The tenant ID passed by Apsis is stored by Membrane and used to scope all connections, actions, flows, and logs to that customer.
- Tenant objects are persistent; runtime logs have a **2-week retention window** and are then automatically erased.
- Persistent config stored on tenant: the tenant object itself, field mapping configurations, connection credentials.

**What Apsis needs to store at their end:** only the workspace key and workspace secret. Everything else (connection state, integration status) can be fetched from Membrane as source of truth.

> "You don't need to store anything as well. You can store it on your end, but it's going to be more complex to keep it up to date, so you better rely on us as source of truth for integration here."

---

## Authentication: Apsis → Membrane

### API Authentication

- Apsis authenticates to Membrane using the **workspace key and secret**.
- All API calls are made in the scope of a specific tenant by including the tenant/customer ID.
- Admin-level API requests (workspace-wide) are possible but intended for CI/CD-type operations, not product flows.

### What Apsis Must Store

```
- Workspace Key
- Workspace Secret
```

These are sufficient to make all calls to Membrane on behalf of any tenant.

---

## Authentication: CRM → Membrane (Third-Party Connection Setup)

### Connection Creation Flow

When an Apsis customer installs a CRM integration, they must create a **connection** in Membrane. This requires the end user to authenticate against the third-party CRM (e.g., HubSpot, Salesforce, Zoho, Pipedrive).

- Membrane handles OAuth2 popup flows, API key entry, and other auth methods transparently — the method depends on what the third-party API supports.
- Credentials are stored encrypted by Membrane.
- Membrane provides a **white-label connection UI** option:
  - **Option A:** Membrane-hosted popup (includes Membrane branding unless white-labeled).
  - **Option B:** Fully white-label popup — goes directly to the OAuth screen without any Membrane iframe. Apsis logo can be substituted.
  - **Option C:** Full background API — Apsis builds their own UI entirely and calls Membrane API to create/manage connections.

### Connection IDs

- Each connection gets a unique **connection ID**.
- A **reconnect** operation preserves the same connection ID.
- Alternatively, if only one connection per integration type per user is expected, Apsis can reference connections by **integration key** (e.g., `zoho`, `pipedrive`) rather than by ID.

```
# Example pattern (pseudocode)
membrane.integration.zoho.action.list_deals.run()
# instead of referencing a specific connection ID
```

- Membrane provides an API to **list all current connections** for a tenant, and to delete them.

### Limiting to One CRM Connection Per Customer

Membrane does not enforce a limit of one connection per integration type — this is **business logic that must be enforced in Apsis's own UI**. Membrane provides the API to list and remove connections so Apsis can implement this.

---

## Authentication: Membrane → Apsis (Callbacks/Webhooks)

### Approach 1: Apsis-Generated Secret (HMAC Signing)

Apsis generates a secret per tenant at the time of integration installation. Membrane stores it as **client credentials** on the customer/tenant object and uses it for all outbound calls to Apsis.

- Secret is initialized via a Membrane API call (documented under **Credentials** in Membrane docs).
- Stored per tenant (not per connection — one secret per tenant covers all connections for that tenant).
- The outbound request from Membrane can include HMAC signature of the request body, implemented via a JavaScript snippet that sets up the API client:

> "You can set up API client via JavaScript function and then it's going to return an API client object. You can extend this and have automatic signing for all requests with this token."

- This JavaScript runs **inside Membrane**. Apsis would need to implement the HMAC signing logic in this snippet. Vlad confirmed Membrane can assist with this.

### Approach 2: Reuse Membrane Token (Simpler Option)

Membrane can make outbound requests to Apsis using **the same token Apsis uses to call Membrane**. In this case:

- No need to generate and manage a separate secret.
- Apsis verifies inbound requests by: (a) verifying origin by IP or domain (`api.getmembrane.com`), and (b) validating the token against workspace key and secret.
- Tenant ID is embedded inside the token, so traffic is automatically segregated per tenant.

> "This is usually what people go with when they want the easiest option."

⚠️ **Lukasz noted:** Apsis already has existing code supporting Approach 1, so the choice depends on which requires less rework.

### Outbound Domain / IP

Membrane will always call out from:
```
api.getmembrane.com
```
Static IPs are also available on request.

---

## CRM Connection Lifecycle and Field Mapping

### Overview of Membrane's Mapping Hierarchy

Membrane supports three levels of configuration:

| Level | Scope | Use Case |
|---|---|---|
| **Universal** | All integrations | Same schema, same field names regardless of CRM |
| **Integration-specific** | One CRM type (e.g., all Zoho customers) | Per-CRM field mapping template |
| **Connection-specific** | One tenant's specific connection | Per-customer customization |

Everything ultimately executes at the connection/tenant level — universal and integration-specific configs are **templates** that get instantiated per tenant.

### Universal Actions and Field Mapping

A **universal action** (e.g., `list_contacts`) defines a fixed output schema that Apsis can rely on regardless of which CRM is connected. Membrane maps CRM-specific field names to this schema automatically.

Example mapping configuration (stored as YAML):
```yaml
# Universal level: define desired output schema
fields:
  name: full_name       # maps CRM's "full_name" → Apsis "name"
  email: primary_email  # maps CRM's "primary_email" → Apsis "email"
```

- Membrane auto-generates integration-specific versions of this mapping for each supported CRM.
- Pipedrive, Zoho, Salesforce, HubSpot will each have different underlying field names — Membrane handles the translation.
- **Formulas/transformations** are supported (e.g., converting integer ID to string).
- All configuration is stored as **YAML** and can be managed via Membrane's CLI (not just UI).

### White-Label Field Mapping UI (Apsis-Hosted)

Apsis wants to host their own field mapping UI. The preferred flow:

1. Apsis fetches the **JSON schema** of a specific object in a specific CRM app (the metadata endpoint) from Membrane.
2. Apsis displays field mapping dropdowns to the customer.
3. Customer configures their mapping in Apsis UI.
4. Apsis calls Membrane API to **persist the mapping at the connection level** for that tenant.

This is a supported and common use case ("white-label field mapping UI — very popular with us").

**Trade-off acknowledged:** Apsis will own less mapping richness than Membrane's native UI (which supports text merging, variables, formulas). Membrane offers a **React component** that can be styled to Apsis's design system as a middle ground.

**Impact on architecture:** This approach increases Apsis's implementation effort compared to their original generic connector concept, because each CRM's mapping must be understood and configured in Membrane.

> "Increases our effort, I would say... but at least the mapping is then possible."

### Subscription Mapping

Subscription mapping (mapping CRM list/subscription concepts to Apsis profile/consent lists) is a separate concern from field mapping. It requires **additional Membrane actions** to be configured — either as universal actions or custom actions per CRM. The team agreed to treat this as a follow-on task and not block on it in the POC.

---

## Full Sync Mechanics

### Overview

Full sync is user-triggered in Apsis (there is a "Start Sync" button in the Apsis One UI). When triggered, Apsis calls a Membrane action to fetch all contacts from the connected CRM.

- This is a **synchronous action call** that returns paginated data.
- Apsis manages the sync orchestration (loop, cursor tracking).

### Pagination

Membrane's `list_contacts` (and similar list actions) return a **cursor** for pagination:

```
# Pseudocode
response = membrane.action.list_contacts.run(cursor=None)
while response.cursor:
    process(response.data)
    response = membrane.action.list_contacts.run(cursor=response.cursor)
```

- Apsis is responsible for the pagination loop.
- No delta/changed-only filtering in the initial full sync implementation; full dataset is fetched. Conditions/filters are considered a future extension.

---

## Real-Time Event Flows (Webhook Replacement)

### Current Apsis Architecture

Apsis currently generates a webhook URL per tenant/integration, registers it directly with the CRM, and receives change events (contact created, updated, deleted) directly.

### Membrane Flow-Based Replacement

Membrane replaces this with **Flows** — pre-configured event pipelines:

- **Triggers:** `created`, `updated`, `deleted` per object type (contacts, deals, etc.).
- If the CRM supports webhooks, Membrane subscribes and manages subscription renewal automatically.
- If no webhooks, Membrane does **polling** (or long polling for poor APIs).
- Apsis does not need to worry about subscription management — Membrane handles it.

> "If they have webhooks, we will subscribe to a webhook, we'll refresh subscription before it expires. If there is no webhooks, we do polling. If there is a poor API which doesn't allow efficient polling, we do long polling. You don't care about it."

### Enabling a Flow

Enabling a flow for a tenant is a **single API call**:
```
# Pseudocode
membrane.flow.receive_contact_events.enable(tenant_id="customer_123")
```

From that point, Membrane listens for events and pushes them to Apsis.

### Webhook URL Configuration via Flow Parameters

The destination URL (where Membrane sends events to Apsis) is configured using **flow parameters**:

```yaml
# Flow definition (YAML)
parameters:
  webhook_url: <string>   # set per-flow-instance at enable time

# Last step in flow:
action: api_request_to_your_app
uri: "{{ parameters.webhook_url }}"
```

When Apsis enables a flow for a tenant, it passes the `webhook_url` parameter. This allows per-tenant routing.

### Tenant Identification in Callbacks

Membrane can include the tenant/customer ID in the callback in multiple ways:
- As part of the URI path (current Apsis approach).
- Via the token (Membrane token includes tenant ID).
- Both can be used simultaneously for belt-and-suspenders validation.

### Flow Customization Levels

- Flows are defined at **integration level** (e.g., one flow template for Zoho contact events).
- When a tenant enables a flow, Membrane creates a **copy** scoped to that tenant.
- Per-tenant customization is possible on that copy (e.g., adding filter nodes for enterprise customers with unusual setups).
- Membrane provides a **Filter node** that can be exposed to end users for self-service filter configuration.

---

## Batch Operations and Ordered Processing

### Sending Batches of Records to CRM

Apsis collects high-volume events (e.g., 100,000 email opens in 20 seconds) and wants to send them to CRM in batches rather than individual calls.

**Membrane's approach:**
- **Actions** are synchronous single-step operations — not designed for batch loops.
- **Flows** are asynchronous and support loops (`for each` node):
  - Apsis sends one call to Membrane with a list of records.
  - The flow iterates and makes individual CRM calls internally.
  - Apsis receives a **flow run ID** as the response (async).
  - Some CRMs (e.g., Salesforce) have native batch APIs — a Membrane action can wrap those directly.

⚠️ **Flow runtime limit: 1 hour.** Batches must be sized to complete within this window.

### Processing Order / Event Ordering

For consent and profile update ordering (Apsis requires ordered processing to avoid race conditions):

- Apsis currently queues calls and waits for each to complete before sending the next.
- Membrane has a **flow queue** — flows are eventually executed in order.
- For strict ordering within a batch, flows can be structured sequentially.
- Exact ordering guarantees across concurrent flow instances were not fully resolved in this session. ⚠️ **[Ambiguous — needs follow-up if strict ordering is required across flows.]**

---

## Rate Limits

### Third-Party CRM Rate Limits

- If using **synchronous actions**: rate limit errors from the CRM are passed through to Apsis. Apsis must handle these on their end.
- If using **asynchronous flows**: Membrane handles CRM rate limits internally using **exponential backoff** — the flow is paused and retried automatically.

### Membrane Platform Rate Limits

- Configurable **per workspace** and **per tenant/customer**.
- Per-customer limits can be adjusted by Apsis independently.
- Per-workspace limits are negotiated with Membrane.
- Platform has been tested at **hundreds of requests per second** — not expected to be a bottleneck.

---

## Retry Logic for Membrane → Apsis Callbacks

Any request from Membrane to Apsis that returns a **4xx status code** is automatically retried with **exponential backoff**.

> "Any request returning 4xx inside the flow will be retried with exponential backoff. If your API follows best practices and you give us 4xx, then no worries at all."

This applies to all flow-based outbound requests. No additional configuration needed — it is the default behavior.

---

## Custom Connector Development (Unsupported CRMs)

For integrations Membrane does not yet support (e.g., Meta Lead Ads API was mentioned as a candidate):

**Option 1: Build it yourself**
- Use the **Create Connector** UI in Membrane.
- Everything Membrane can do is exposed — no hidden features.
- All connector config is YAML; use Membrane CLI to export/import.
- Point a coding agent (Claude, Cursor, etc.) at the YAML files + Membrane docs + LLM context files provided by Membrane.

**Option 2: Use Membrane's AI Agent**
- Provide a link to the target API's documentation and credentials.
- Agent builds the connector.
- Quality depends on prompt quality and documentation quality (OpenAPI spec = easy; poor docs = more iteration).

**Option 3: Membrane Expert Service**
- Submit a request to Membrane's solutions team.
- Assigned to a solutions engineer.
- Typical turnaround: **5–10 hours** to ship a connector (ballpark; varies by API complexity).
- Requires a **test account** for the target API — without one, Membrane cannot build it.
- Hourly fee applies.

---

## CI/CD and Configuration-as-Code

Membrane fully supports configuration-as-code workflows:

- All workspace configuration (actions, flows, mappings) is **YAML**.
- **Membrane CLI** allows export from one workspace and import to another.
- Integrates into CI/CD pipelines: dev → staging → production promotion.

```bash
# Example workflow (conceptual)
membrane export --workspace dev > ./membrane-config/
# commit to repo
membrane push --workspace staging ./membrane-config/
```

> "We have a customer who is spinning up a branch for every single ticket they have and creating a separate workspace for every single ticket, making live deployments with this."

---

## POC Scope and Next Steps

### Proposed POC Structure

- Duration: **2 weeks**
- Goal: validate that Membrane technically works for Apsis's use case and establish a skeleton implementation.

**Suggested scope:**
- Set up **bidirectional sync** with one CRM.
- Demonstrate how a second CRM is added (to validate the scaling/reuse story).
- Universal action + field mapping configuration.
- Real-time event flow for contact changes.

**Support from Membrane during POC:**
- Kickoff call to align on requirements and success criteria.
- Live code examples with source code matching Apsis's use case.
- Pair coding sessions / technical calls on request.
- Help configuring initial action and flow prototypes.
- Effectively **enterprise support** for the two-week period.

**Apsis context noted:** Apsis team to spend a few days in Warsaw digesting the session, aligning internally on requirements and confidence level, then reconnect with Membrane (Kyle) end of week or beginning of following week.

**Who owns commercial/pricing negotiation:** Daniel (Apsis).

**Competitor landscape:** Apsis indicated their primary alternative is **in-house development** (not a competing vendor).

---

## Key Takeaways

1. **Authentication summary:** Apsis stores only workspace key + secret. All calls to Membrane include tenant ID for scoping. Membrane→Apsis callbacks can use either a Membrane-generated token (simpler) or an Apsis-generated HMAC secret stored as client credentials on the tenant object. Tenant ID is embedded in the token either way.

2. **Connection management is Apsis's responsibility at the UI layer.** Membrane enforces nothing about how many connections per tenant — Apsis must implement that logic in their product.

3. **Field mapping can be white-labeled.** Apsis can host their own mapping UI, fetch JSON schema from Membrane, and persist customer mapping choices back to Membrane at connection level via API. This is a well-supported pattern but increases implementation effort vs. reusing Membrane's native UI.

4. **Full sync = paginated action calls with cursor.** Simple to implement. Apsis owns the loop and orchestration.

5. **Real-time events = Flows, not direct webhook registration.** Membrane abstracts webhook subscription, renewal, and polling. Apsis enables flows via API and receives events at a configured URL. No webhook registration code needed on the Apsis side.

6. **Retry and rate limit handling is largely automatic in flows.** 4xx → exponential backoff retry. CRM rate limits → Membrane handles internally in flows. Synchronous actions pass rate limit errors through to the caller.

7. **All config is YAML + CLI-exportable.** Full CI/CD integration is possible and documented.

8. **Data retention:** Only tenant object and mapping config are persistent. Runtime log data (including any integration data passing through) is retained for **2 weeks maximum**.

9. **Flow runtime hard limit: 1 hour.** Batch operations must be sized accordingly.

---

## Unresolved Questions and Action Items

| # | Question / Action Item | Owner |
|---|---|---|
| 1 | Decide between Approach 1 (Apsis-generated HMAC secret) vs. Approach 2 (reuse Membrane token) for Membrane→Apsis callback authentication, based on rework cost of existing code. | Apsis engineering |
| 2 | Confirm exact API endpoint/docs reference for storing client credentials on tenant object ("Credentials" section in Membrane docs). | Vlad / Membrane docs |
| 3 | Clarify static IP list for `api.getmembrane.com` — Vlad mentioned these are available on request. | Membrane (Kyle/Vlad) |
| 4 | Subscription mapping (CRM subscription/list concepts → Apsis profile/consent lists) — not fully designed. Needs its own design session. | Apsis + Membrane |
| 5 | Negotiation capability discovery ("what CRM features does this integration support") — partially discussed but deferred. Apsis will preset supported features in Membrane as static config rather than doing runtime negotiation. Needs confirmation. | Apsis |
| 6 | Strict event ordering guarantees across concurrent Membrane flows — not fully resolved. If Apsis requires strict ordering across flow instances (not just within one flow), this needs to be validated. | Apsis + Membrane |
| 7 | Meta Lead Ads API connector — evaluate build-it-yourself vs. Membrane Expert Service. Test account would be required for Membrane to build it. | Apsis (Daniel - commercial) |
| 8 | Kyle to send written recap of this session to Apsis team. | Kyle (Membrane) |
| 9 | Apsis team internal alignment session in Warsaw — then reconnect with Kyle end of week / early following week to align on POC scope. | Lukasz + Kyle |
| 10 | POC commercial terms / pricing negotiation. | Daniel (Apsis) |