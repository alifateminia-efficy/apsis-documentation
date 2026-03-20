---
source_file: Erik - Accessing customer CRM for manual debug and incident recovery.txt
domain: Apsis One - Integrations
topics: [Debugging customer CRM instances, Webhook management and remediation, Generic connector architecture, Broker service proxy pattern, Secrets management and rotation, API credential handling, Incident recovery procedures]
speakers: [Erik Andersson, Tomasz Kowalski]
key_components: [Broker service, Generic connector, Tribe CRM, FCC Enterprise, Dynamics, Lime CRM, Webhook subscriptions, API credentials management, Secrets manager]
session_type: knowledge-transfer
---

## Session Overview

Erik walks Tomasz through a real incident scenario where buggy webhooks were accidentally registered for Tribe customers in November, causing empty profiles to be created in Apsis for every contact. The session demonstrates how to diagnose and manually remediate this issue when you don't have direct access to customer API credentials. Erik covers the architecture differences between legacy FCC Enterprise (where credentials are stored in plaintext) and the Generic Connector (where credentials are hashed), explains the broker service proxy pattern used to safely access customer systems, and walks through the actual debugging workflow using the broker service's internal endpoints.

---

## The Tribe Webhook Bug: Root Cause and Impact

### What Went Wrong

[Erik Andersson]: Back in November, we introduced a bug for Tribe customers where we registered webhooks for the lead entity in their CRM environments when we shouldn't have. Tribe will never send us actual lead entities, and due to how Tribe reacts to these webhooks, they're still sending us updates even though they shouldn't.

The core problem: For every contact person created inside Tribe, an empty profile gets created in Apsis with the lead ID. Here's why this happens:

1. We registered webhooks on the lead entity that we shouldn't have
2. When a contact is created in Tribe, Tribe sends us a lead webhook notification
3. The contact's ID is interpreted as a lead ID
4. We try to create or update the profile in the lead keyspace using the Tribe lead ID as the identifier
5. Since there's no actual lead data, we create an empty profile

### The Solution Required

We need to remove these incorrect webhooks from the customer's CRM instances and clean them from our database. If we don't remove them from the database, there's a risk the customer will fail on reinstallation because we'll try to re-register webhooks that are supposed to be there but aren't.

---

## Credential Storage Architecture: Enterprise vs. Generic Connector

### FCC Enterprise (Legacy Approach)

In FCC Enterprise, credentials are stored directly in database tables and are accessible in plaintext:

- Table: `connections_fsc` (for legacy FCC)
- Stored fields: API URL, API secret, webhook secret
- Direct access available for debugging purposes

When requesting connections for FCC Enterprise:
```
SELECT account_id, section, api_url, api_secret, webhook_secret FROM connections_fsc
```

[Erik Andersson]: We have direct access to the API key. If I want to, I could just copy-paste this and make a direct request to the customer's CRM instance. We are allowed to do this for debugging purposes—anything the connector can do, we're allowed to do to correct issues or debug situations if the customer requires it.

### Generic Connector (Modern Approach)

The Generic Connector uses a fundamentally different approach:

- Single table: `connections` (streamlined across all connector types)
- API credentials are **hashed**, not stored in plaintext
- Credentials are stored in a separate, encrypted location
- Developers cannot directly access customer API secrets from the database

When querying a generic connector connection:
```
SELECT account_id, section, api_url FROM connections WHERE customer='Master Planning'
```

Notice: No API secret field present.

**Why this matters**: This design prevents direct database access to customer credentials, which is a security best practice. However, it complicates manual debugging scenarios.

---

## The Broker Service: Safe Proxy Access to Customer Systems

### How It Works

The broker service acts as a secure proxy between our internal systems and customer CRM instances:

1. We make a request to the broker service's internal endpoint (IP-whitelisted)
2. We provide the account ID, section, and integration ID
3. The broker service decrypts the customer's API secret from the secrets manager
4. The broker service attaches the secret to the authorization header
5. The broker service forwards the request to the customer's CRM instance
6. We never directly handle or see the customer's actual credentials

### Internal Broker Endpoint URL Structure

```
https://broker-internal.apsis.io/forward?
  X-Account-ID={account_id}
  &X-Section={section}
  &X-Integration={integration_id}
  &X-Justin-Url={customer_crm_endpoint_url}
  &Authorization={internal_management_key}
```

**Required Parameters:**

- **X-Account-ID**: The Apsis account ID (e.g., `139389815`)
- **X-Section**: The section within the account (only one CRM system per section allowed)
- **X-Integration**: The integration ID, used as foreign key to webhooks subscriptions table
- **X-Justin-Url**: The actual endpoint URL in the customer's CRM system that you want to reach
- **Authorization**: Internal management secret (same key used for force uninstallation)

**Example**: To retrieve webhooks for a customer:
```
https://broker-internal.apsis.io/forward?
  X-Account-ID=139389815
  &X-Section=default
  &X-Integration=gen_connector_master_planning
  &X-Justin-Url=https://api.tribe.com/v1/webhooks/record
  &Authorization={integration_internal_management_key}
```

---

## Finding Required Information for Broker Requests

### Step 1: Identify the Connection ID

Query the connections table to find the customer:
```sql
SELECT id, account_id, section, api_url FROM connections
WHERE customer_name LIKE '%Master Planning%'
```

Example result:
- Connection ID: `conn_12345`
- Account ID: `139389815`
- Section: `default`
- API URL: `https://api.tribe.com`

### Step 2: Find Webhook Subscriptions

Use the connection ID to find all registered webhooks:
```sql
SELECT id, subscription_id, entity_type, webhook_url FROM webhook_subscriptions
WHERE connection_id = 'conn_12345'
```

This reveals what webhooks are currently registered. In the Tribe bug scenario, we see:

- ✅ Contact record webhook (correct)
- ✅ Contact consent webhook (correct)
- ❌ Lead record webhook (incorrect—should not exist)
- ❌ Lead consent webhook (incorrect—should not exist)

### Step 3: Retrieve Webhooks from Customer System

Use the broker service to list all webhooks in the customer's CRM:

```
GET https://broker-internal.apsis.io/forward?
  X-Account-ID=139389815
  &X-Section=default
  &X-Integration=gen_connector_master_planning
  &X-Justin-Url=https://api.tribe.com/v1/webhooks/record
  &Authorization={secret}
```

This confirms which webhooks actually exist in the customer's system.

---

## Deleting Incorrect Webhooks: The Workflow

### Step 1: Verify You Have the Right Webhook

From the GET request above, you get a subscription ID for the webhook you want to delete:
```
subscription_id: 77520  (lead record webhook)
```

### Step 2: Delete from Customer's CRM

Use the DELETE endpoint through the broker service:

```
DELETE https://broker-internal.apsis.io/forward?
  X-Account-ID=139389815
  &X-Section=default
  &X-Integration=gen_connector_master_planning
  &X-Justin-Url=https://api.tribe.com/v1/webhooks/record/77520
  &Authorization={secret}
```

Expected response: `HTTP 201` (subscription deleted)

### Step 3: Verify Deletion in Customer System

Repeat the GET request to confirm the webhook is gone:

```
GET https://broker-internal.apsis.io/forward?
  X-Account-ID=139389815
  &X-Section=default
  &X-Integration=gen_connector_master_planning
  &X-Justin-Url=https://api.tribe.com/v1/webhooks/record
  &Authorization={secret}
```

The deleted webhook should no longer appear in the response.

### Step 4: Clean Up Local Database

Once confirmed deleted from the customer's system, remove from our webhook_subscriptions table:

```sql
DELETE FROM webhook_subscriptions
WHERE id IN (77520, 77521)  -- the lead record and consent webhooks
```

> **Critical**: Do NOT skip this step. If webhooks exist in our database but not in the customer's CRM, a customer reinstallation will fail because we'll try to re-register webhooks we think should be there.

---

## Gotchas and Lessons Learned from This Incident

### Tribe API Quirk

[Erik Andersson]: Tribe has an interesting behavior with their webhook API. When you delete a webhook by subscription ID, they actually delete by ID alone—it doesn't matter if you specify `record` vs `consent` in the URL. The subscription ID is unique across all types, so they just match on that.

This is a bit confusing from a design standpoint, but it works. Just be aware: the entity type in the URL path doesn't filter the deletion.

### The Entity Type Query Parameter

The generic connector specification allows an optional `?entity=` query parameter on debug endpoints:

```
GET /v1/webhooks/record?entity=lead
```

This filters results to a specific entity type. However:

- **For business logic operations**: Entity type is specified in the URL path (required)
- **For debugging operations**: Entity type can be a query parameter (optional, filters response)

Most operations require the entity in the path because you always need to know which entity you're operating on. Query parameters are primarily for debugging to filter down responses when you have multiple entities registered.

---

## The Generic Connector API Specification

### Where to Find It

The complete API specification for the Generic Connector is documented in the codebase:

```
/apsis/integrations/lib/generic_connector/api_spec.json
```

[Erik Andersson]: All existing endpoints are listed here. You can always check the specification if anything looks weird. For example, we added endpoints for e-commerce systems (abandoned carts, etc.) but that never took flight and the developer left the company, so those are now stale endpoints we should clean up. Don't waste time on those.

### Common Debug Endpoints

- `GET /v1/webhooks/record` — List all webhooks for profile updates (contacts)
- `GET /v1/webhooks/record?entity={entity_type}` — List webhooks for a specific entity
- `GET /v1/webhooks/consent` — List consent webhooks
- `DELETE /v1/webhooks/record/{subscription_id}` — Delete a webhook by subscription ID
- `GET /v1/schema` — Get the data schema for entities
- `GET /v1/records` — Download all records (full sync)
- `GET /v1/records?entity={entity_type}` — Download records for a specific entity

---

## Entity Type Configuration: When to Download Leads

### Default Behavior

By default, Apsis only downloads contacts from CRM systems. We don't download leads unless explicitly configured.

### Per-Integration Configuration

Some customers may want leads synced in addition to contacts:

- **Dynamics by Side-Shop**: Have explicitly requested lead downloads (not yet implemented)
- **Generic configuration**: We can specify per integration whether to include leads in full sync or not

> **Important**: For most integrations, do NOT download leads to Apsis. This is a deliberate design decision. Only enable this if the customer explicitly requests it and you've configured it in their integration settings.

---

## Secrets Management and Rotation

### Where the Broker Service Key Lives

The internal management key used for broker requests is stored in the secrets manager:

```
Secrets Manager → integration.internal_management_key
```

This is the same key used for:
- Manual broker requests (webhook debugging, CRM access)
- Force uninstallation (completely erasing a customer integration)

[Erik Andersson]: This is a management key, so be careful with it. In theory, if someone gets access to it, they could do rather evil stuff. It's not a customer key, but it's still a privileged credential.

### Rotating the Key

1. Go to the secrets manager and update the value
2. Generate a new random UID or use a password manager to generate a random string
3. **Important**: Restart the broker service after rotation

The secret is loaded from the secrets manager and injected as an environment variable on startup, not read dynamically.

### Impact of Rotation

- ✅ The broker service will be unavailable for ~20 seconds during restart
- ✅ No other internal services are affected (platform will keep running)
- ❌ Any manual requests using the old key will fail with `Permission Denied`
- ✓ Once broker service is back up, all new requests with the new key will work

---

## Use Cases for Manual Broker Requests

### Typical Debugging Scenarios

[Erik Andersson]: The main use case is when a Professional Services person comes to you saying "the integration doesn't work, what's wrong?" You make a manual request to their CRM instance and can check:

- Is the instance responding at all? (timeout?)
- Do we have permission to access the endpoint? (403?)
- Has the instance crashed? (5xx errors?)
- What is the actual state of webhooks in their system?

These manual sanity checks are much faster than digging through logs.

### Direct Access Alternative

If you're collaborating with Professional Services or an account manager and they can provide the customer's API key directly, you can use the direct request pattern in Postman without going through the broker:

```
GET /v1/webhooks/record
Host: {customer_api_url}
Authorization: Bearer {customer_api_key}
```

This bypasses the broker and goes straight to the customer's CRM. However, you need the actual API key, which is usually only available when collaborating directly with the customer.

---

## Key Takeaways

1. **Credential Architecture Matters**: Legacy systems store credentials plaintext (simpler but less secure), modern systems hash them (requires proxy pattern but more secure).

2. **The Broker Service is Your Friend**: When you need to access a customer's CRM without direct API keys, use the broker service's internal endpoints. It's designed for exactly this scenario.

3. **Always Know Your Parameters**: Account ID, section, integration ID, and the target CRM endpoint URL are required for every broker request. Mistakes here will fail silently or mysteriously.

4. **Database Cleanup is Non-Optional**: If you delete webhooks from a customer's CRM, you MUST delete the corresponding records from `webhook_subscriptions`, or reinstalls will fail.

5. **Manual Debugging Saves Time**: Rather than diving into logs, a quick GET request through the broker can answer "is this endpoint even responding?" in seconds.

6. **Check the API Spec**: The Generic Connector spec is the source of truth. Don't guess at endpoints or parameters—look it up.

7. **Secrets Rotation Requires Downtime**: The broker service needs to restart to pick up a new management key. Plan for ~20 seconds of downtime for manual requests.

8. **Query Parameters Are for Debugging**: Entity type in the path is required for business logic. Query parameters (like `?entity=`) are for filtering debug responses only.

---

## Unresolved Questions and Follow-Up Items

- [Erik Andersson notes]: Need to clean up stale e-commerce endpoints from the Generic Connector API spec before leaving the company
- [Erik Andersson]: Going to continue cleaning Tribe instances after this session; planned to rotate the integration management key "very swiftly, very shortly" but will execute separately to not consume Tomasz's time