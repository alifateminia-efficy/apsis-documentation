---
source_file: Erik - Accessing customer CRM for manual debug and incident recovery.txt
domain: Apsis One Integrations
topics: [Manual debugging of customer CRM instances, Webhook management and removal, Generic Connector architecture, Broker service proxy access, Incident recovery for Tribe integration, Database cleanup for erroneous webhooks]
speakers: [Erik Andersson, Tomasz Kowalski]
key_components: [Broker service, Generic Connector, Tribe CRM, FCC Enterprise, Microsoft Dynamics, Webhook subscriptions, Connection credentials management, Secret manager]
session_type: knowledge-transfer
subdomains: [Architecture, Tribe Integration, Lead creation]
---

## Session Overview

Erik Andersson walked through an incident recovery case involving erroneous webhooks created for Tribe integration customers, demonstrating how to manually debug and fix customer CRM instances when direct API access is unavailable. The session covered the architectural differences between legacy integrations (FCC Enterprise, Dynamics) and the Generic Connector, explained the broker service proxy mechanism for secure credential handling, and detailed the step-by-step process of identifying and removing incorrect webhook subscriptions using internal endpoints and the Postman collection.

---

## The Tribe Webhook Bug: Root Cause and Impact

### What Happened

In November, a bug was introduced in the Tribe integration that registered webhooks for the **lead entity** in customer CRM environments, despite Tribe never sending lead entities to Apsis One.

[Erik Andersson]: > "We registered web hooks for lead entity in their CRM environments. When we kind of shouldn't because Tribe will never send us any actual lead entities to us and due to unfortunate circumstances, how tribe is also reacting to this like they shouldn't send us updates on these web books, even though we register them."

### The Cascading Problem

The bug creates empty profiles in Apsis with incorrect identifying information:

1. For every contact person created in Tribe, an empty profile is created in Apsis
2. The webhook uses the lead ID instead of the contact ID
3. The system attempts to create/update a profile in the "lead keyspace"
4. Since no matching lead exists, an empty profile is created with the Tribe lead ID as the identifier

[Erik Andersson]: This is incorrect behavior—profiles should only be created when actual lead or contact records exist with proper identifying attributes.

### The Fix Required

Two erroneous webhooks must be removed from the customer's CRM:
- Lead record webhook (for profile updates)
- Lead consent webhook (for consent updates)

Only the **contact entity webhooks** should remain (one for records/profile updates, one for consents).

---

## Architecture: Connection Storage and API Credential Handling

### Legacy Integration Approach (FCC Enterprise, Dynamics)

For legacy integrations like **FCC Enterprise**, connections are stored with direct API access:

```
Table: connections_fec
Columns: account_id, section_id, api_url, api_secret, webhook_secret, custom_config
```

[Erik Andersson]: With this approach, operators can directly copy-paste API keys and make requests to the customer's CRM instance for debugging purposes. This is allowed and has no legal repercussion, as long as it's for legitimate debugging and incident recovery.

The table includes custom configuration fields because FCC Enterprise allows extensive customization, including some problematic configurations that affect consent handling.

### Generic Connector Approach: Encrypted Credentials

The Generic Connector uses a fundamentally different credential storage model:

```
Table: connections (generic)
Columns: account_id, section_id, api_url, [NO api_secret stored here]
```

[Erik Andersson]: "Like here is no API key. We don't have access to this in the general connector and that's because this is stored in a separate table. We don't store it in clear text as we do. Of course the database is encrypted that we can log in and see it here. Hashed. We don't have direct access to this for good reason."

The API credentials are:
- Stored in a **separate encrypted table**
- Hashed and not accessible in clear text
- Never exposed directly to debugging operators

### Why This Matters

This design prevents accidental or malicious credential exposure while still allowing legitimate debugging through the broker service proxy.

---

## The Broker Service: Secure Proxy for Credential Access

### Architecture and Purpose

The broker service acts as an HTTP proxy between Apsis One operators and customer CRM instances. It handles decryption and credential injection without exposing secrets.

**Request Flow:**
1. Operator makes request to the **internal broker endpoint** with authentication
2. Broker decrypts the customer's API credentials (stored in encrypted vault)
3. Broker attaches credentials to the authorization header
4. Broker forwards the request to the customer's CRM instance
5. Response returned to operator without exposing credentials at any point

### Internal Broker Endpoint Structure

```
Base URL: https://internal-broker.apsis.one/debug/proxy

Required URL Parameters:
  - account: Customer account ID
  - section: Section identifier
  - integration: Integration type (e.g., "tribe", "dynamics")
  - X-Justin-URL: Target endpoint in customer's CRM system

Required Header:
  - Authorization: Bearer [integration_secret_key]
```

[Erik Andersson]: "The reason we need all of this now is because they are applied to a connection ID. They use that as what they call foreign keys."

### Example Request Structure

For querying webhooks in the Tribe instance:

```
GET https://internal-broker.apsis.one/debug/proxy?
  account=13938915
  &section=1
  &integration=tribe
  &X-Justin-URL=https://api.tribe.com/v1/webhooks/subscriptions?entity=record

Authorization: Bearer [integration_secret_key]
```

The broker will:
1. Look up connection credentials for account 13938915, section 1
2. Decrypt the Tribe API key
3. Attach it to the request
4. Forward to `https://api.tribe.com/v1/webhooks/subscriptions?entity=record`

---

## Webhook Subscription Management

### Understanding the Data Model

Webhooks are stored with foreign key relationships to connections:

```
Table: webhook_subscriptions
Columns: id, connection_id, entity_type, callback_url, type (record|consent)
```

[Erik Andersson]: "They use that as what they call foreign keys. So now I can say what are the subscription IDs that are connected for this specific connection and as you can we can see here we have one web hook here for contacts as we should for records."

### Expected vs. Actual State

For a properly configured Tribe customer, the webhooks should be:

**Expected (correct state):**
- `entity=contact, type=record` (profile updates)
- `entity=contact, type=consent` (consent updates)

**Actual (buggy state):**
- `entity=contact, type=record` ✓
- `entity=contact, type=consent` ✓
- `entity=lead, type=record` ✗ (should not exist)
- `entity=lead, type=consent` ✗ (should not exist)

### Retrieving Webhooks

Using the Generic Connector debug endpoint:

```
GET /v1/webhooks/subscriptions?entity=record
GET /v1/webhooks/subscriptions?entity=consent
```

The `entity` query parameter filters results. This is specific to debugging—business logic endpoints typically require the entity as a path parameter.

[Erik Andersson]: "For all core functionalities we usually... you simply always have to specify which entity and then you do it in the URL like or as the path. You don't do that as a query parameter. The query parameters are primarily the debugging ones where we want to filter the response."

---

## Step-by-Step Manual Webhook Removal

### Step 1: Identify the Customer Connection

From the database, locate the connection record:

```sql
SELECT id, account_id, section_id, api_url 
FROM connections 
WHERE account_id = 13938915 AND section_id = 1
```

Result: `connection_id = 6789, api_url = https://api.tribe.com`

### Step 2: List All Webhook Subscriptions for This Connection

Query the database to find all webhooks for this connection:

```sql
SELECT id, entity_type, type, callback_url 
FROM webhook_subscriptions 
WHERE connection_id = 6789
```

Results:
- `id=7751, entity=contact, type=record` ✓
- `id=7752, entity=contact, type=consent` ✓
- `id=7753, entity=lead, type=record` ✗
- `id=7754, entity=lead, type=consent` ✗

Subscription IDs 7753 and 7754 must be removed.

### Step 3: Verify Webhooks Exist in the Customer's CRM

Make a broker request to confirm the webhooks exist in the actual CRM instance:

```
GET https://internal-broker.apsis.one/debug/proxy?
  account=13938915
  &section=1
  &integration=tribe
  &X-Justin-URL=https://api.tribe.com/v1/webhooks/subscriptions?entity=record

Authorization: Bearer [integration_secret]
```

Response confirms subscription IDs 7751, 7753 exist in the CRM for record-type webhooks.

### Step 4: Delete Webhooks from the Customer's CRM Instance

For each erroneous webhook, make a DELETE request through the broker:

```
DELETE https://internal-broker.apsis.one/debug/proxy?
  account=13938915
  &section=1
  &integration=tribe
  &X-Justin-URL=https://api.tribe.com/v1/webhooks/subscriptions/7753

Authorization: Bearer [integration_secret]
```

[Erik Andersson]: Repeat for subscription 7754 (the consent webhook for leads).

### Step 5: Verify Deletion in Customer CRM

Confirm the webhooks no longer exist:

```
GET https://internal-broker.apsis.one/debug/proxy?
  account=13938915
  &section=1
  &integration=tribe
  &X-Justin-URL=https://api.tribe.com/v1/webhooks/subscriptions?entity=record

Authorization: Bearer [integration_secret]
```

Result: Only subscription 7751 (contact record) remains.

### Step 6: Clean Up Apsis Database

Remove the erroneous webhook records from the Apsis database:

```sql
DELETE FROM webhook_subscriptions 
WHERE id IN (7753, 7754)
```

[Erik Andersson]: > "If we don't remove them from the database here, there's a risk that the customer will fail on an installation because we then say like, yeah, they're supposed to be like these two."

This prevents the system from automatically re-creating these webhooks during the next installation or sync operation.

### Important Note on API Response Codes

[Erik Andersson]: During the actual remediation, an unexpected behavior was discovered: the Tribe API returns HTTP 201 (Created) even when deleting webhooks, regardless of whether the request was made to the correct entity path. This appears to be a bug in their API implementation where they match subscriptions by ID alone rather than validating the entity-type path parameter. The deletion still works, but the response code is misleading.

---

## Using the Generic Connector for Debugging

### Direct CRM Access vs. Broker Access

**When you have the customer's API credentials directly** (e.g., shared by Professional Services or account manager):

Use the direct Generic Connector endpoints in Postman:
- No broker service needed
- Immediate request to the CRM system
- Useful for quick verification that the CRM behaves as expected

**When you don't have credentials** (standard case):

Use the broker service proxy through the internal endpoint, as described above.

### Query Parameters vs. Path Parameters

The Generic Connector has different conventions:

**Query parameters** (for debugging/filtering):
- `?entity=record` — Filter webhooks by entity type
- Primarily used for manual debugging to reduce response size

**Path parameters** (for business logic):
- `/v1/schemas/{entity}` — Get schema for a specific entity
- `/v1/records/{entity}` — Download records for a specific entity
- Always required for business logic operations; cannot be optional

[Erik Andersson]: "For almost all other options here you have to specify which one it is because it is always like you want to get the schema for leads. You want to get the schema for contacts, so you have to specify which one you are interested in."

### Available Debug Endpoints

All endpoints are documented in the Generic Connector specification:

```
File: libs/generic-connector/spec.json (or equivalent documentation)
Location: In the generic-connector library or API reference
```

Common endpoints:
- `GET /v1/webhooks/subscriptions` — List all webhooks (supports `?entity=` filter)
- `GET /v1/webhooks/subscriptions/{id}` — Get specific webhook
- `DELETE /v1/webhooks/subscriptions/{id}` — Delete webhook
- `GET /v1/schemas/{entity}` — Get entity schema
- `GET /v1/records/{entity}` — Download entity records (with pagination)

---

## Integration Secrets and Key Rotation

### The Integration Management Key

The secret used for broker service requests is called the **integration secret** or **integration_internal_management_key**:

```
Location: Secret Manager (in the product account)
Value: [High-entropy random string, e.g., UUID or generated string]
Used for: Broker service authentication, force uninstallation operations
```

[Erik Andersson]: > "So the that secrets is this, it says now delete it says delete integration because this is the key we also use when we do the force uninstallation... It is the same we use for the broker service. It's kind of a integration internal management key, so be a bit careful with it like because you can in theory do rather evil stuff if you were to get access to it."

### What the Key Allows

With this key, you can:
- Make manual requests to any customer's CRM through the broker
- Perform force uninstallations
- Clean up or modify integration data

This is why it should be treated as a highly sensitive credential.

### How to Rotate the Key

1. **Generate a new secret:**
   - Use `uuidgen`, a random string generator, or password manager (e.g., LastPass)
   - Ensure sufficient entropy (at least 32 characters)

2. **Update the secret in the Secret Manager:**
   ```
   # In the product account's secret store
   integration_internal_management_key = [new_secret_value]
   ```

3. **Restart the broker service:**
   - The secret is loaded as an environment variable on startup, not read dynamically
   - Restart takes approximately 20 seconds
   - Only the broker service is affected; the integration platform itself continues running

[Erik Andersson]: > "Important to know you will need to restart the broker service because the secret is loaded on... it's loaded from the secrets manager and injected as a environment variable, so it's not completely dynamically read."

4. **Expected behavior after rotation:**
   - Old manual requests will fail with permission denied
   - No other internal services are affected
   - The integration platform continues to function normally
   - Only manual debugging requests through the broker will require the new secret

---

## Common Debugging Scenarios

[Erik Andersson]: > "Luckily, it is usually not like this is one of the cases. Typically what I use it for is some PS person comes to say like the integration doesn't work you what is wrong? You need to fix it and then you make like a manual request to their CRM instance and you see like yeah, I'm getting time out or I don't have permission or no the instance has crashed and it's not giving me a response. So you a lot to just like do simple sanity checks for debugging purposes."

### Typical Use Cases

1. **CRM instance connectivity issues:**
   - Make a simple webhook listing request to verify the CRM is responding
   - Identify timeouts, network issues, or authentication problems

2. **Permission errors:**
   - Detect if the API credentials lack required permissions
   - Help Professional Services understand if customer's API account needs escalation

3. **Instance crashes or unavailability:**
   - Quickly determine if the customer's CRM instance is down
   - Differentiate between platform issues and customer infrastructure problems

4. **Webhook verification:**
   - Confirm expected webhooks exist in the customer's system
   - Identify orphaned or erroneous webhooks (as in the Tribe case)

5. **Schema verification:**
   - Query the customer's data structure to understand custom fields or entity configurations
   - Diagnose mapping or synchronization issues related to schema mismatches

---

## Post-Incident Cleanup and Documentation

### Legacy Connector Code Cleanup

[Erik Andersson]: The Generic Connector specification document contains some legacy endpoints that never reached production, such as e-commerce endpoints for abandoned carts. These should be cleaned up to avoid confusion.

[Erik Andersson]: > "For example this here like we we did add some endpoints for e-commerce systems like we could get abandoned cards, etcetera, but this never took flight. And then a deal left and then we essentially craft every plan on making anything with general connector for e-commerce, so there's no pointing you having nothing here."

This is a good reminder to periodically audit API documentation and remove endpoints that no longer serve a purpose.

### Postman Collection Maintenance

The Postman collection for Generic Connector debugging should include:
- At least one example of each major operation (list, get, delete webhooks; retrieve schemas; download records)
- Clear comments on when to use broker service vs. direct access
- Example requests with all required parameters

[Erik Andersson]: Having one verified example in Postman makes subsequent debugging much faster, as all similar operations follow the same pattern—only the target URL and entity type change.

---

## Key Takeaways

1. **The Tribe bug was a design error:** Registering webhooks for entities that are never sent (lead in Tribe) creates orphaned profiles in Apsis with incorrect identifiers.

2. **Generic Connector uses secure credential storage:** Unlike legacy integrations, API secrets are never accessible directly to operators. The broker service proxy must be used for all customer CRM access.

3. **The broker service pattern is essential:** By decrypting credentials server-side and injecting them into requests, secrets are never exposed to human operators or logs, reducing the risk of credential leaks.

4. **Manual remediation requires careful verification:**
   - Identify erroneous subscriptions in the Apsis database
   - Verify they exist in the customer's CRM
   - Delete from both the CRM and the database
   - Confirm deletion to prevent re-creation on next sync

5. **The integration secret is highly sensitive:** It allows full access to any customer CRM through the broker service. Rotation requires a broker service restart but no other downtime.

6. **Debugging follows a consistent pattern:** Once you've done it once (using Postman with the broker endpoint), all subsequent cases follow the exact same structure—only the account ID, section, and target URL change.

7. **Query parameters are for debugging; path parameters are for business logic:** Understanding this distinction helps you use the Generic Connector API correctly and efficiently.

8. **Preventive measures matter:** Removing erroneous webhooks from both the CRM and the database prevents the system from trying to reinstall them during future syncs or installations.

---

## Unresolved Questions / Action Items

- **API Bug in Tribe:** The Tribe API appears to match webhook subscriptions by ID alone, returning 201 (Created) responses even for DELETE operations. This should be documented in the integration notes and potentially escalated to Tribe for clarification.

- **Secret rotation procedure:** After this session, the integration secret should be rotated since it was displayed in the screen share. [Erik Andersson] indicated this would be done shortly after the session.

- **Generic Connector specification cleanup:** Remove or mark as deprecated the e-commerce endpoints that never reached production (abandoned carts, etc.).

- **Postman collection verification:** Ensure the Generic Connector Postman collection has at least one example of each major operation type and is easily discoverable by the team.