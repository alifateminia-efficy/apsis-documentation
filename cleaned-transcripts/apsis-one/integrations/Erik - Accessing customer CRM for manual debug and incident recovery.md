---
source_file: Erik - Accessing customer CRM for manual debug and incident recovery.txt
domain: Apsis One Integrations
topics: [Manual debugging of customer CRM instances, Webhook management and removal, Generic Connector API access patterns, Broker service architecture, Incident recovery from erroneous webhook registrations]
speakers: [Erik Andersson, Tomasz Kowalski]
key_components: [Generic Connector, Broker Service, Tribe CRM, Efficy Enterprise, Database connection management, Webhook subscription system, API credential management]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector, Inbound Flow, Tribe]
---

## Session Overview

Erik Andersson walked Tomasz Kowalski through a real incident recovery case involving erroneous webhook registrations on a Tribe customer's CRM instance. The session covered how to manually access and debug customer CRM systems when direct API credentials are unavailable, the difference between legacy connector architectures (Efficy Enterprise) and the modern Generic Connector approach, and detailed operational procedures for identifying and removing problematic webhooks using the Broker Service as a secure proxy.

## The Tribe Webhook Bug Incident

### Background and Root Cause

[Erik Andersson]: In November, a bug was introduced to Tribe customers where webhooks were incorrectly registered for the **lead entity** in their CRM environments. This should never have happened because Tribe will never send Apsis actual lead entities—the integration only expects contact entities.

The problem manifests as follows:
- Every time a contact person is created in Tribe, an empty profile gets created in Apsis
- The system incorrectly uses the Tribe lead ID as the profile identifier
- When the system tries to create or update a profile in the lead keyspace, there is nothing there (because leads should never have been created)
- This results in profiles being created with empty identifying IDs

The core issue: webhooks for the lead entity should not exist, but due to Tribe's behavior and how the system interprets webhook requests, these malformed registrations persist and cause data corruption.

### Impact and Solution Required

The resolution requires two actions:
1. Remove the erroneous webhooks from the customer's CRM instance
2. Remove the corresponding webhook subscription records from the Apsis database

Without removing the database records, reinstalling the integration could fail because the system would expect those subscriptions to already exist in the customer's CRM.

## Database Architecture: Legacy vs. Generic Connector Approaches

### Legacy Efficy Enterprise Architecture

For **Efficy Enterprise** installations, credentials are stored directly in accessible database tables:

```
Table: con_fcc
Columns:
- Account ID
- Section
- API URL
- API Secret (stored in clear text in database, encrypted at rest)
- Webhook Secret (for verification of incoming webhook requests)
```

With this model, an engineer with database access can directly copy API credentials and make direct requests to customer CRM instances for debugging purposes. This is legally permitted under the integration's operational guidelines—anything the connector can do is allowed for debugging and issue resolution.

### Generic Connector Architecture (Current Standard)

For Generic Connector integrations, the architecture is intentionally more restrictive:

```
Table: connections
- No API key field visible
- Account ID
- Section
- API URL (only)
```

The actual API credentials are stored in a **separate table and hashed**, not stored in clear text. This means:
- Engineers cannot directly access the customer's API secret
- This is by design for security reasons
- An alternate access pattern is required

## The Broker Service: Secure Proxy Access Pattern

### Architecture and Purpose

The **Broker Service** acts as a secure proxy between Apsis and customer CRM instances. It decrypts and manages credentials on behalf of the integration without exposing them to engineers.

### How It Works

1. An engineer makes a request to the internal Broker Service endpoint instead of directly to the customer's CRM
2. The request includes:
   - **Account ID**
   - **Section**
   - **Integration ID**
   - **Authorization header** (using an integration management secret key)
   - **X-Forwarded-URL** (the actual endpoint in the customer's CRM system to call)

3. The Broker Service:
   - Validates the authorization secret
   - Uses the account/section/integration ID to locate the stored credentials
   - Decrypts the customer's API secret
   - Attaches it to the authorization header
   - Forwards the request to the customer's CRM
   - Returns the response to the engineer

This pattern ensures the engineer never has direct access to customer credentials while still enabling debugging and incident recovery.

### Example Request Pattern

```
Internal Broker Service Endpoint:
https://[internal-broker-url]

Required Parameters:
- Authorization: [integration-secret-key]
- X-Forwarded-URL: https://[customer-crm-url]/[api-endpoint]
- Query: account=[account-id]&section=[section-id]&integration=[integration-id]
```

For Tribe (a SaaS system), the customer CRM URL is consistent across all instances, making it straightforward to construct requests.

## Identifying Incorrect Webhooks: The Investigation Process

### Step 1: Locate Connection Metadata

Using the connection ID from the database, retrieve all metadata needed:
- Account ID
- Section ID  
- API URL
- Connection ID (used as foreign key to webhook subscriptions)

### Step 2: Query Webhook Subscriptions

Using the connection ID, query all registered webhook subscriptions:

```
SELECT * FROM webhook_subscriptions 
WHERE connection_id = [connection-id]
```

For a correctly configured Tribe integration, you should see:
- **Contact Records webhook** (for profile updates: create/update/delete)
- **Contact Consent webhook** (for consent changes)

For the affected customer, the query revealed:
- Contact Records webhook ✓ (correct)
- Contact Consent webhook ✓ (correct)
- **Lead Records webhook** ✗ (incorrect—should not exist)
- **Lead Consent webhook** ✗ (incorrect—should not exist)

The subscription IDs for the erroneous webhooks are used to delete them.

## Manual Webhook Removal via Broker Service

### Step 1: Verify Webhooks Exist in Customer System

Use the Generic Connector debug endpoint to retrieve all webhooks:

**Request:**
```
GET /v1/webhooks/records?entity=contact
```

**Via Broker Service:**
```
Authorization: [integration-secret]
X-Forwarded-URL: https://[tribe-url]/v1/webhooks/records?entity=contact
Query params: account=[id]&section=[id]&integration=[id]
```

This confirms the erroneous webhooks are actually registered in the customer's CRM instance.

### Step 2: Delete Webhooks from Customer System

For each subscription ID to be removed, use the delete webhook endpoint:

**Delete endpoint:**
```
DELETE /v1/webhooks/[subscription-id]
```

[Erik Andersson] noted discovering a potential bug in Tribe's API: the deletion seemed to work regardless of whether `records` or `consent` was specified in the endpoint path—it appeared to match on ID alone, which is functional but confusing from an API design perspective.

### Step 3: Verify Deletion

Re-run the GET request to confirm the webhooks are gone:

```
GET /v1/webhooks/records?entity=contact
```

After successful deletion, only the contact records and contact consent webhooks should remain.

### Step 4: Remove from Apsis Database

Delete the subscription records from the database to prevent reinstallation from recreating them:

```sql
DELETE FROM webhook_subscriptions 
WHERE id = [subscription-id-1] OR id = [subscription-id-2]
```

[Erik Andersson] initially asked about dry-run capability but proceeded with the deletion. This is a critical operation—verification via the GET request before deletion is strongly recommended.

## Query Parameters in Generic Connector Endpoints

### Entity Filtering

Many Generic Connector endpoints support an optional `entity` query parameter to filter results:

```
GET /v1/webhooks/[type]?entity=[entity-name]
```

### When Entity Parameter is Used vs. URL Path

- **Business logic operations**: Entity is specified in the URL path (e.g., `GET /schema/contacts`, `GET /records/contacts`)
- **Debugging/filtering operations**: Entity is specified as a query parameter (e.g., `GET /webhooks/records?entity=contacts`)

[Erik Andersson] clarified that query parameters are primarily for debugging and response filtering, while path parameters are for core functionality where you must specify which entity to operate on.

### Example Use Case

For Dynamics 365, which may have multiple entities:
- If you only care about contacts, you can filter: `?entity=contacts`
- If you want all entities: omit the parameter

However, configuration specifies which entities Apsis should sync. For most integrations (e.g., Efficy Enterprise 12.1), only one entity (contacts/profiles) is configured, so filtering is not necessary.

## Secret Management and Key Rotation

### Integration Management Secret Key

The authorization key used for Broker Service requests is called the **integration management key** (in secret manager, labeled `delete_integration`).

This is a sensitive credential because it enables:
- All Broker Service access to customer CRM instances
- Force uninstallation of integrations (used when customers cannot delete integrations normally and need complete database cleanup)
- Potential malicious operations if compromised

### Where to Find the Key

```
Secret Manager > [Product Account] > delete_integration
```

### How to Rotate the Key

1. Generate a new random key (using random UID generator or password manager like LastPass)
2. Update the secret in the secret manager with the new value
3. **Restart the Broker Service**

### Important Caveats

- The secret is loaded at startup as an environment variable, not dynamically read from the secret manager
- After updating the secret, the Broker Service must be restarted (takes ~20 seconds)
- Only manual debugging requests via the Broker Service will fail with permission denied during the restart
- Other internal services (the Integration Platform itself) are unaffected by the key rotation
- Users will not experience service interruption for normal integration operations

## Using Direct CRM Access for Debugging

### When You Have Direct API Credentials

In some scenarios, engineers may have direct access to customer API credentials through:
- Professional Services collaboration
- Account manager involvement
- Customer providing credentials directly

In these cases, you can construct direct requests to the CRM without using the Broker Service:

```
Authorization: [customer-api-key]
[Direct request to customer CRM endpoint]
```

This bypasses the Broker Service entirely and is useful for:
- Verifying CRM behavior
- Testing connectivity
- Quick debugging when credentials are available

### When You Don't Have Direct Credentials

Use the Broker Service pattern (as described above). This is the standard approach.

## Generic Connector API Specification

The **Generic Connector specification** documents all available endpoints and their parameters:

**Location:**
```
generic_connectors/lib/[connector-name]/specification
```

This specification should be consulted when:
- Constructing debug requests
- Verifying available query parameters
- Understanding endpoint behavior
- Checking which entities each endpoint supports

[Erik Andersson] noted that some endpoints in the specification are legacy (e.g., e-commerce endpoints for abandoned carts) that never saw production use. These should be cleaned up but remain documented.

## Operational Debugging Patterns

### Common Use Cases for Manual CRM Access

1. **Connectivity verification**: Customer reports integration not working
   - Make a simple GET request to verify the CRM is responding
   - Check for timeout, permission, or service unavailability errors

2. **Instance health checks**: Verify customer's CRM instance is operational
   - If no response, the instance may have crashed
   - Useful for triaging whether the issue is on Apsis or customer side

3. **Schema validation**: Verify the CRM has expected structure
   - Get schema for configured entities
   - Check if custom fields exist

4. **Webhook verification**: Confirm correct webhooks are registered
   - As done in this incident

5. **Record inspection**: Spot-check data in customer's CRM
   - Verify sync is working correctly

### Pattern for Any Request Type

Once you've made one manual request to a customer's system via the Broker Service, the pattern is identical for all future requests:

1. Extract/note the account ID, section ID, integration ID
2. Retrieve the API URL
3. Construct the Broker Service URL with authorization
4. Change only the `X-Forwarded-URL` to point to the desired endpoint
5. Execute the request

This standardization makes it easy to handle multiple customer issues—the core pattern is always the same.

## Postman Configuration for Generic Connector Debugging

[Erik Andersson] referenced having a Postman collection with:
- Direct CRM requests (for when you have API keys)
- Broker Service requests (for standard customer debugging)

This collection should include example requests that new team members can copy and adapt when debugging customer issues.

## Key Takeaways

1. **Webhook registration bugs can have long-tail impacts**: Erroneous webhooks cause continuous data corruption (empty profiles) until manually removed. Both the CRM and database must be cleaned.

2. **Broker Service is the secure standard**: For Generic Connector integrations, always use the Broker Service proxy pattern rather than attempting to obtain customer credentials. This protects credentials while enabling debugging.

3. **Two-step cleanup required**: Remove incorrect registrations from both the customer's CRM instance (via API) and the Apsis database (direct SQL) to prevent reinstallation issues.

4. **Query parameters are for debugging**: In Generic Connector endpoints, entity filtering via query parameters is available but optional—primarily used for narrowing debug output.

5. **Integration management key is sensitive**: The Broker Service authorization secret enables force uninstallation and full customer system access. Handle rotation carefully and plan for ~20 second Broker Service restart.

6. **Patterns are standardizable**: Once you've debugged one customer issue using the Broker Service, subsequent issues follow identical patterns. Only the specific endpoint and credentials context change.

7. **Direct CRM access is convenient when available**: If you have customer API credentials (via PS or account team), direct requests avoid the Broker Service. But the secure path via Broker is standard.

8. **Tribe API has quirks**: The Tribe CRM appears to match webhook deletions by ID alone, not by entity type in the path. Verify actual webhook removal via GET request after deletion.

## Unresolved Questions and Follow-up Items

- [Erik Andersson] planned to continue cleaning additional Tribe instances after the session ended but did not provide details on how many affected customers or timeline
- The Tribe API's webhook deletion behavior (matching by ID vs. by path) may warrant investigation to understand if it's a design choice or undocumented behavior
- The Generic Connector specification has legacy endpoints (e-commerce) that are no longer used—these were flagged for cleanup but no action was taken during the session