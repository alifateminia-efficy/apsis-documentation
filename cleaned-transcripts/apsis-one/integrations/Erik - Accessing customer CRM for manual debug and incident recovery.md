---
source_file: "Erik - Accessing customer CRM for manual debug and incident recovery.txt"
domain: Apsis One Integrations
topics: [webhook incident recovery, broker service proxy, generic connector debugging, CRM credential management, webhook subscription cleanup, secret key rotation]
speakers: ["Erik Andersson (Senior/Lead Developer)", "Tomasz Kowalski (Developer, KT recipient)"]
key_components: [broker service, generic connector, webhook subscriptions, connections table, secret manager, Postman collection, Tribe CRM, FCC Enterprise]
session_type: knowledge-transfer
---

## Session Overview

Erik walks Tomasz through a live incident recovery procedure for a Tribe CRM bug introduced in November where lead entity webhooks were incorrectly registered in customer CRM instances. The session covers how to locate connection credentials in the database, why direct API access is unavailable for generic connector integrations (by design), how to use the broker service as an authenticated proxy to make manual requests to customer CRM instances, and how to clean up both the CRM-side webhook subscriptions and the corresponding database records. Erik also covers the integration management secret key — its uses, location, and rotation procedure.

---

## Background: The Tribe Webhook Bug

A bug was introduced in November that caused **lead entity webhooks** to be registered in Tribe customer CRM environments when they should not have been. The root cause and consequence:

- Tribe will never send lead entities to Apsis, but the webhooks were registered anyway.
- Tribe, for its part, does send updates on these webhooks despite the registration being incorrect.
- The result: for every contact person created inside Tribe, an **empty profile is created in Apsis** using the lead ID as the identifier — because the incoming webhook is interpreted as a lead event, and the contact's ID is taken as the lead ID. A create/update is attempted in the lead key space, finds nothing, and creates an empty profile with the Tribe lead ID as the identifying field.

The remediation requires:
1. Removing the incorrect lead webhooks from each affected customer's CRM instance.
2. Removing the corresponding webhook subscription records from the Apsis database.

---

## Database Structure: Connection Credentials by CRM Type

### Legacy / Per-System Connection Tables

For older CRM integrations, credentials are stored in **dedicated per-system tables**:

- `connections_fcc` — legacy FCC Enterprise
- `connection_dynamics` — Microsoft Dynamics
- `con_lime` — Lime CRM

These tables store credentials in clear text (the database itself is encrypted) and include fields such as: account ID, section, API URL, API secret, and webhook secret (used to verify incoming webhook requests by checking a shared secret included by the CRM sender).

> "You can customise your FCC enterprise instance to doomsday come. There are some annoying configurations that some customers enable, despite us telling them not to, because it completely breaks the whole way we handle consents in enterprise. We ultimately had to come up with a solution to handle that configuration."

Because credentials are directly readable in these tables, it is technically possible to copy-paste an API key and make a direct request to a legacy FCC customer's CRM instance for debugging purposes.

### Generic Connector: `connections` Table

From the generic connector onward, all integrations use a single unified `connections` table. **Critically, API secrets are not stored in clear text here.** When a customer enters their API key in the UI, it is stored hashed/encrypted in a separate table. Developers do not have direct access to the plaintext secret.

> "We don't store it in clear text... We don't have direct access to this for good reason."

This means a different approach is required to make authenticated requests to generic connector customer instances — the **broker service**.

---

## The Broker Service: Authenticated Proxy for Customer CRM Access

The **broker service** acts as an authenticated proxy between Apsis internal tooling and customer CRM instances. Its role:

1. Receives a forwarded request from an internal caller.
2. Looks up the encrypted credential for the specified customer.
3. Decrypts the secret and attaches it to the `Authorization` header.
4. Forwards the request to the customer's CRM instance.

This means developers never need plaintext access to customer API secrets for operational/debugging work.

### Making a Manual Request via the Broker Service

Requests to the broker service use an **internal endpoint URL**. The required parameters are:

- **Account**, **Section**, and **Integration ID** — so the broker knows which credential to decrypt and attach.
- **Authorization header** — authenticates the caller to the broker service itself, using the integration management secret key (see [Secret Key Management](#secret-key-management-and-rotation) below).
- **`X-Forward-URL` header** (referred to as the "forward forward URL") — specifies the actual endpoint in the customer's CRM instance to be called.

Example construction:
```
Broker internal endpoint URL:
  /internal/... (account, section, integration specified as path or query params)

X-Forward-URL:
  <customer API URL from connections table> + <CRM endpoint path>
  e.g., <tribe_api_url>/v1/webhooks/record
```

> "The broker service could in reality extract the URL from data you have in the database, but we have it such that you specify the URL for the customer's instance first, then you specify what is the actual endpoint in the CRM system that you want to reach."

For **Tribe specifically**, the API URL is the same for every customer (it is a SaaS product), which simplifies this step.

For other CRM systems where you happen to have the API key available (e.g., when collaborating with Professional Services or an Account Manager who holds the customer's key), you can make requests directly using the generic connector Postman collection without going through the broker service.

---

## Generic Connector API Specification and Postman Collection

The generic connector contract/specification lives at:

```
connectors/lib/generic/  (generic API spec)
```

All available endpoints are listed there. The Postman collection has two relevant configurations:
- **Direct** (generic connector collection): requests go straight to the CRM system. Requires API key + URL.
- **Via broker service**: uses the `X-Forward-URL` pattern described above.

### The `entity` Query Parameter

[Tomasz asked about the `entity` query parameter visible in a forwarded request URL.]

The `entity` query parameter comes from the generic connector contract. It is used on **debug/listing endpoints** to filter results to a specific entity type (e.g., `contact`, `lead`).

For core business logic endpoints, the entity is always specified **in the URL path** (e.g., get schema for leads vs. contacts), not as a query parameter.

The query parameter form is primarily for debugging endpoints where you want to filter the response.

**Multi-entity context:** Most CRM systems only have one relevant entity (e.g., FCC Enterprise → contacts only). However, some systems like E-Deal or Dynamics have both contacts and leads. For each integration, there is a configuration flag for whether Apsis should also download leads. 

> "The idle implementation for the love of God don't download the leads to Apsis... we can specify for each integration, should you do that or not."

Dynamics by side shop has explicitly requested lead download but this had not been implemented yet at the time of this session.

---

## Live Incident Recovery: Removing Incorrect Tribe Webhooks

### Step 1: Identify the Connection

Look up the customer in the `connections` table. Retrieve:
- Connection ID
- Section
- API URL

The customer being fixed in this session was identified from a report by Anita (account: "him master planning"). Connection ID: `13938915` ⚠️ *(exact value — verify in DB)*.

### Step 2: Retrieve Webhook Subscriptions from Database

Webhook subscriptions are stored in the `web_hook_subscriptions` table (or similar), using the `connection_id` as a foreign key. Query to retrieve all subscriptions for the connection.

Expected correct state for a Tribe contact integration:
- ✅ One webhook for `contact` records (profile create/update/delete)
- ✅ One webhook for `contact` consents

Incorrect state found:
- ❌ One webhook for `lead` records
- ❌ One webhook for `lead` consents

### Step 3: Confirm Webhooks Exist in the Customer's CRM

Use the broker service to call the generic connector debug endpoint:

```
GET /v1/webhooks/record?entity=<entity_type>
```

via `X-Forward-URL` pointing at the customer's Tribe API URL. Confirmed: a callback URL for leads existed in the customer's instance alongside the correct contact callback URL.

### Step 4: Delete Webhooks from the Customer's CRM

Use the delete webhook endpoint via the broker service, specifying the `subscription_id` for each incorrect webhook. 

**Note — discovered API quirk:** During live execution, Erik observed that the Tribe webhook delete API appears to match only on subscription ID, regardless of which entity path is used in the URL (e.g., deleting via the `consent` path using a `record` subscription ID still returned `201`). This is likely a Tribe API implementation detail — IDs are globally unique so the entity path may be ignored.

> "I think I actually discovered a bug in their API because they just based it on the ID, which I guess is fine because they are unique. Just a bit confusing."

Delete both:
1. Lead record webhook
2. Lead consent webhook

Verify via GET after deletion that only contact record and contact consent webhooks remain.

### Step 5: Delete Webhook Subscription Records from Apsis Database

After removing from the CRM, delete the corresponding rows from the webhook subscriptions table:

```sql
DELETE FROM web_hook_subscriptions WHERE id = 7752;
-- (repeat for the second subscription)
```

Erik verified the row count before committing.

**Why this step is critical:** On installation/reinstallation, the system iterates over all subscriptions linked to a connection. If stale/incorrect subscriptions remain in the database, the customer's reinstallation may fail because the system expects those webhooks to exist but they don't.

---

## Secret Key Management and Rotation

### What the Key Is Used For

The integration management secret key (referred to in the secret manager as the **"delete integration" key**) serves two purposes:

1. **Broker service authentication** — authorizes manual requests routed through the broker service (as described above).
2. **Force uninstallation** — used when a customer's integration is stuck and cannot be deleted normally. The key allows complete erasure of the integration from the database so the customer can reinstall. (Note: the customer must handle CRM-side cleanup themselves after a force uninstall.)

> "It's kind of an integration internal management key — be a bit careful with it because you can in theory do rather evil stuff if you were to get access to it."

### Where to Find It

The key is stored in **AWS Secrets Manager** in the product account.

### How to Rotate It

1. Go to the secret in AWS Secrets Manager.
2. Update the value with a new randomly generated string (e.g., via LastPass or a random UUID generator).
3. **Restart the broker service.** The secret is loaded at startup and injected as an environment variable — it is not dynamically re-read at runtime.

Broker service restart takes approximately 20 seconds.

**Impact of rotation:**
- ✅ No impact on the integration platform or any other internal service.
- ❌ Any in-flight or saved manual debug requests using the old key will receive `403 Permission Denied` until updated with the new key.

---

## Typical Use Cases for Manual CRM Access

Erik noted the lead webhook cleanup is an unusual use case. The more common scenario:

> "Typically what I use it for is some PS person comes to say the integration doesn't work, what is wrong? You need to fix it. And then you make a manual request to their CRM instance and you see like, yeah, I'm getting a timeout, or I don't have permission, or the instance has crashed and it's not giving me a response. So you use it a lot to just do simple sanity checks for debugging purposes."

---

## Permissions and Data Access Policy

Developers are permitted to make requests to customer CRM instances through the broker service for:
- Debugging integration issues
- Correcting problems (as in this incident)

They are **not** permitted to download customer data arbitrarily. Any access should be justified by a support/debugging need or customer request.

---

## Key Takeaways

1. **Generic connector integrations do not expose API secrets to developers** — by design. All manual access to customer CRM instances must go through the broker service.
2. **The broker service requires three pieces of identifying information** in every request: account, section, integration ID — used to look up and decrypt the correct credential.
3. **The `X-Forward-URL` (forward forward URL)** specifies the actual CRM endpoint to call. For generic connector endpoints, refer to the API spec in `connectors/lib/generic/`.
4. **Webhook cleanup is a two-step process**: remove from the CRM first (via broker service), then delete from the Apsis database — order matters to avoid leaving the system in an inconsistent state.
5. **The integration management secret** is the same key for both broker service access and force uninstallation. It lives in AWS Secrets Manager. Rotation requires a broker service restart (~20s) but has no broader platform impact.
6. **Tribe's webhook delete API** ignores the entity type in the URL and matches only on subscription ID — noted as a potential API quirk/bug.
7. **Stale webhook subscription DB records can break reinstallation** — the installer iterates over DB subscriptions and will fail if CRM-side webhooks are missing.

---

## Unresolved Questions / Action Items

- [ ] **Erik to rotate the integration management secret key** (mentioned he would do this immediately after the session).
- [ ] **Erik to continue cleaning remaining affected Tribe customer instances** (him master planning was the first; more remain).
- [ ] **Erik to clean up the generic connector API spec** — abandoned e-commerce endpoints (abandoned cart, etc.) from an E-Deal/general connector e-commerce initiative that was cancelled should be removed before Erik's departure.
- [ ] **⚠️ Ambiguity**: The exact internal broker service endpoint URL pattern was not fully captured in transcript (screen was shared but URL details were partially obscured/not read aloud). Tomasz should verify the full URL structure from the Postman collection or ask Erik to document it explicitly.
- [ ] **Tomasz to verify he has access to the Postman collection** for generic connector (Erik asked him to confirm during the session).