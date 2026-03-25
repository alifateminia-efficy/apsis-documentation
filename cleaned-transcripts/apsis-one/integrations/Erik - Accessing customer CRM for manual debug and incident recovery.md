---
source_file: Erik - Accessing customer CRM for manual debug and incident recovery.txt
domain: Apsis One Integrations
topics: [Manual CRM Debugging, Webhook Management, Generic Connector Architecture, Broker Service Mechanics, Incident Recovery, API Credential Handling]
speakers: [Erik Andersson, Tomasz Kowalski]
key_components: [Broker Service, Generic Connector, Tribe CRM, Efficy Enterprise, Connections Database, Webhook Subscriptions, Secret Manager]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector, Inbound Flow, Different Types of Connectors, Zapier Integration]
---

## Session Overview

Erik walked Tomasz through the process of manually debugging and recovering customer CRM instances when integration issues occur. The session centered on a real incident where webhooks were incorrectly registered on Tribe CRM instances for lead entities, causing empty profiles to be created in Apsis. Erik demonstrated how to use the broker service as a proxy to securely access customer systems without direct exposure of API credentials, how to query and delete webhooks via the Generic Connector API, and the workflow for cleaning up integration database records. The session included live debugging examples and emphasized the secure credential handling architecture that differs between legacy (Efficy Enterprise) and modern (Generic Connector) integrations.

---

## The Tribe Webhook Bug: Root Cause and Impact

### Background: The November Bug Introduction

[Erik Andersson]: In November, a bug was introduced for Tribe customers where webhooks were registered for the lead entity in their CRM environments. This should never have happened because Tribe will never send us actual lead entities.

### How the Bug Manifests

The issue creates a problematic flow:

1. When a contact person is created inside Tribe, an empty profile gets created in Apsis
2. The system uses the contact's ID but tags it with the lead ID (from the lead webhook)
3. The system attempts to create or update a profile in the "lead key space" using the Tribe lead ID
4. Since there's no matching data in that keyspace, an empty profile is created with this identifier
5. This breaks the expected behavior where only contact entities should generate profiles

### Why It Happened

Tribe's API behavior combined with a design mistake: the system was registering webhooks it shouldn't, and Tribe's API responses treat webhook deletion inconsistently (a bug in their implementation that Erik discovered during the debugging session).

---

## Architecture: Credential Storage Across Connector Types

### Legacy Approach: Efficy Enterprise (FCE)

For legacy Efficy Enterprise installations, credentials are stored directly in accessible database tables:

**Table: `connections_fce`**

Contains:
- Account ID
- Section
- API URL
- API secret (clear text in database)
- Webhook secret (for verifying incoming webhook requests)

[Erik Andersson]: If we look at Efficy Enterprise, we store all connections—which is the API URL and credentials we need to utilize the instance. Here in this table we store the credentials for the legacy FCC enterprise installations. This means I have direct access to the API key. I could copy-paste it and make a direct request to the customer's CRM instance. We are allowed to do this for debugging purposes—anything the connector can do, we are allowed to do to correct issues or debug situations if the customer requires it.

This straightforward approach works because Efficy Enterprise is self-hosted and customizable, requiring direct access for troubleshooting.

### Modern Approach: Generic Connector

For Generic Connector instances, the architecture is deliberately different:

**Table: `connections` (generic)**

Contains:
- Account ID
- Section  
- API URL
- Connection ID
- **NO API secret stored directly**

[Erik Andersson]: For the generic connector, there is no API key. We don't have access to this in the general connector because it is stored in a separate table. When the customer enters it in the UI, we don't store it in clear text. The database is encrypted, and we don't have direct access to this for good reason.

The API secret is hashed and stored separately, preventing operator access to customer credentials.

### Why This Matters for Debugging

The consequence: operators cannot simply copy credentials and make direct API calls. Instead, they must use the broker service as an intermediary.

---

## The Broker Service: Secure Proxy Architecture

### What the Broker Service Does

[Erik Andersson]: The broker service sits as a proxy between us and the customer. We forward the request to the proxy and specify which customer CRM instance we want to contact. The proxy then decrypts the secret for that specific customer and attaches it to the authorization header, then forwards the request. We never need to have direct access to these secrets.

### Making Requests Through the Broker Service

#### The Internal Broker Endpoint

```
https://[internal-broker-url]/proxy
```

This is an internal-only endpoint (IP-whitelisted systems).

#### Required Authorization

```
Authorization: Bearer [INTEGRATION_SECRET]
```

The `INTEGRATION_SECRET` is a single shared key used for:
- Internal broker service requests
- Force uninstallation operations
- Any integration platform internal management task

[Erik Andersson]: This secret exists in the secret manager inside of Apsis. Be a bit careful with it because you can in theory do rather evil stuff if you were to get access to it. It's a kind of integration internal management key.

#### Request Structure: Three Components

The request URL requires three pieces of information:

```
https://[broker-url]/proxy?account=[ACCOUNT_ID]&section=[SECTION]&integration=[INTEGRATION_ID]
```

Followed by a request header:

```
X-Forward-Url: [CUSTOMER_CRM_API_ENDPOINT]
Authorization: Bearer [INTEGRATION_SECRET]
```

**Component Breakdown:**

1. **Account ID + Section**: Tells the broker service which customer's credentials to decrypt
   - One CRM system per section (you cannot have multiple CRM systems on one section)
   - These values come from the `connections` table query

2. **Integration ID**: Specifies which type of integration (e.g., "tribe", "dynamics")
   - Used by broker to select the correct credential decryption logic

3. **X-Forward-Url**: The actual endpoint in the customer's CRM system you want to call
   - This is where the decrypted credentials get applied

[Erik Andersson]: The reason we need all of this is because the webhooks are applied to a connection ID using that as a foreign key. So we can say "what are the subscription IDs connected to this specific connection" and then find the webhooks we need to manage.

---

## Querying Customer Webhooks: The Generic Connector API

### Step 1: Gather Connection Information

Query the database to get the connection details:

```sql
SELECT id, account_id, section_id, api_url 
FROM connections 
WHERE [customer-identifying-criteria]
```

From this query, extract:
- `connection_id` (e.g., 139815)
- `account_id` (e.g., 13938915)
- `section_id`
- `api_url` (e.g., `https://tribe.example.com`)

### Step 2: Query Webhook Subscriptions

```sql
SELECT * 
FROM webhook_subscriptions 
WHERE connection_id = [CONNECTION_ID]
```

This shows all registered webhooks for the customer.

**Example output shows:**
- Webhook subscriptions with their unique `id`
- Entity types they're monitoring (e.g., "contacts", "records", "leads")
- Webhook type (e.g., "record", "consent")

### Step 3: Identify Problematic Webhooks

For the Tribe bug, the problematic webhooks are:
- Lead entity record webhook (should not exist)
- Lead entity consent webhook (should not exist)

Only the contact entity webhooks should remain.

### Step 4: Retrieve Webhooks via API

Use the Generic Connector API endpoint via the broker:

**Request:**
```
GET https://[broker-url]/proxy?account=[ACCOUNT_ID]&section=[SECTION]&integration=tribe
Header: X-Forward-Url: /v1/webbooks/record
Header: Authorization: Bearer [INTEGRATION_SECRET]
```

**Tribe API endpoint (from Generic Connector spec):**
```
GET /v1/webbooks/record
```

Optional query parameter to filter by entity:
```
?entity=contacts
```

[Erik Andersson]: The query parameter `entity` comes from the generic connector contract. You can specify which entity you're interested in. For most core functionalities in business logic, you have to specify which entity in the URL path itself. But query parameters are primarily for debugging to filter the response.

### Example Response

The API returns subscription IDs and callback URLs:

```
{
  "subscriptions": [
    {
      "id": "7751",
      "callbackUrl": "https://apsis-webhook-endpoint.com/...",
      "entity": "contacts"
    },
    {
      "id": "7752",
      "callbackUrl": "https://apsis-webhook-endpoint.com/...",
      "entity": "leads"  // <- SHOULD NOT EXIST
    }
  ]
}
```

---

## Deleting Webhooks: API and Database Cleanup

### Step 1: Delete via API

Use the delete webhook endpoint for each problematic subscription:

**For contacts record webhook:**
```
DELETE https://[broker-url]/proxy?account=[ACCOUNT_ID]&section=[SECTION]&integration=tribe
Header: X-Forward-Url: /v1/webbooks/record/[SUBSCRIPTION_ID]
Header: Authorization: Bearer [INTEGRATION_SECRET]
```

**For contacts consent webhook:**
```
DELETE https://[broker-url]/proxy?account=[ACCOUNT_ID]&section=[SECTION]&integration=tribe
Header: X-Forward-Url: /v1/webbooks/consent/[SUBSCRIPTION_ID]
Header: Authorization: Bearer [INTEGRATION_SECRET]
```

[Erik Andersson]: We make the request to delete the consent webhook, but I actually discovered a bug in their API. They just based the deletion on the subscription ID, which is fine because they're unique, but the endpoint behavior was a bit confusing.

### Step 2: Verify Deletion in CRM

After deleting via API, confirm the webhooks are gone:

```
GET https://[broker-url]/proxy?account=[ACCOUNT_ID]&section=[SECTION]&integration=tribe
Header: X-Forward-Url: /v1/webbooks/record
Header: Authorization: Bearer [INTEGRATION_SECRET]
```

Expected result: Only the contact entity webhooks remain.

### Step 3: Clean Up Database Records

This is **critical**—if database records aren't removed, the customer may fail on reinstallation:

```sql
DELETE FROM webhook_subscriptions 
WHERE id = [SUBSCRIPTION_ID]
```

[Erik Andersson]: On installation we iterate over each subscription connected to this connection. If we don't remove them from the database, there's a risk that the customer will fail on an installation because we'll say "yeah, these webhooks are supposed to exist."

Execute one DELETE per problematic webhook ID from the earlier query.

### Why Both API and Database Cleanup Are Required

- **API deletion**: Removes the webhook from the customer's actual CRM system
- **Database deletion**: Removes the record from Apsis's internal tracking
- **Both needed**: During reinstallation, the system checks the database to see which webhooks should exist. If the database record remains, reinstallation will try to re-register the webhook.

---

## Debugging Workflow: General Patterns and Use Cases

### Common Debugging Scenarios

[Erik Andersson]: Typically what I use the broker service for is when a PS person says "the integration doesn't work, what is wrong, you need to fix it." Then you make a manual request to their CRM instance and see:

- **Timeout errors**: The customer's instance is slow or unreachable
- **Permission errors**: The credentials don't have sufficient scope
- **Instance crashes**: The customer's CRM has crashed and isn't responding
- **Unexpected responses**: The instance is behaving differently than expected

### Direct vs. Proxied Requests

**Direct requests (if you have customer credentials):**
- Used when collaborating with professional services or account managers who have shared their API key
- Make requests directly to the customer's CRM
- Useful for quick verification of CRM behavior

**Proxied requests through broker (normal case):**
- Required when you don't have customer credentials
- Used for any production debugging of customer instances
- Maintains security by not exposing credentials to operators

[Erik Andersson]: Sometimes you actually have access to the credentials because you sit and collaborate with professional services or some account manager and they have the customer's API key. Then it's straightforward—you can just run the relevant request. But usually you will need to take this detour around the forward URL using the broker service.

### Making the Same Request Pattern for Multiple Customers

Once you've made one successful proxy request, the pattern is identical for any other customer:

1. Query the database for the new customer's `account_id`, `section`, and `integration_id`
2. Swap those values into the proxy URL
3. Swap the `X-Forward-Url` value if calling a different endpoint
4. Keep the `Authorization` header (same secret) and integration type

[Erik Andersson]: If you have done this once, then in any other customer scenario, you follow the exact same pattern. All requests follow the exact same structure. The only thing you need to make sure is that you're using the correct account, section, and integration ID, and you need to switch out what is the URL that you want to call for in the customer.

---

## Managing Integration Secrets and Rotation

### Where the Integration Secret Is Stored

The secret used in all broker requests is stored in the secret manager (AWS Secrets Manager or equivalent) inside the Apsis platform:

```
Secret name: [something like "integration-platform-internal-key"]
```

### Uses of the Integration Secret

1. **Broker service requests**: All manual debugging requests
2. **Force uninstallation**: Completely erasing a customer's integration from the database if they can't delete it normally
3. **Integration internal management**: Any operation that needs to modify or inspect integrations globally

[Erik Andersson]: If there's some issue with any customer's integration and they can't delete it, we have the tools to completely erase it from our database so they can reinstall if they want. Of course, they need to do the cleanup themselves. We use the same secret for this as for the broker service.

### How to Rotate the Secret

1. **Generate a new secret** (use a secure random string generator, LastPass, etc.)

2. **Update in secret manager**:
   ```
   Update the secret with the new value
   ```

3. **Restart the broker service**:
   - The secret is loaded on startup as an environment variable
   - It's not dynamically reloaded during runtime
   - Broker service restarts in ~20 seconds

[Erik Andersson]: Important to know: you will need to restart the broker service because the secret is loaded from the secrets manager and injected as an environment variable. It's not completely dynamically read. The broker service restarts in about 20 seconds, so that's not a big deal.

### Impact of Rotation

- **Broker service**: Will be unavailable for ~20 seconds during restart
- **Integration platform**: No impact—will continue working normally
- **Other internal services**: No impact
- **Manual debugging requests**: Will fail with "permission denied" until broker service restarts

[Erik Andersson]: You will notice it because you'll get a permission denied error. But no other internal service will be affected if you change it. The integration platform will not stop working. It's only your manual requests that will fail.

---

## The Generic Connector API Specification

### Where to Find the Spec

The complete API specification is in:
```
[Repository location]: /generic-connectors/lib/[api-spec-file]
```

### What It Defines

- All available endpoints for the generic connector
- Query parameters and their meanings
- Entity types supported
- Entity-specific fields and behaviors

### Key Distinction: Path Parameters vs. Query Parameters

[Erik Andersson]: For all core functionalities and business logic, you have to specify which entity in the URL path itself. You don't do that as a query parameter. Query parameters are primarily for debugging ones where we want to filter the response.

**Example—business logic requires entity in path:**
```
GET /v1/schema/contacts
GET /v1/schema/leads
```

You must specify which entity. You can't do `/v1/schema?entity=contacts`.

**Example—debugging can use query parameters:**
```
GET /v1/webbooks/record?entity=contacts
```

You can filter which entity's webhooks you want to see.

### Entity Specification in Different Contexts

When downloading records during a full sync:
```
GET /v1/records/contacts  // Always download contacts
GET /v1/records/leads     // Only if explicitly configured per integration
```

[Erik Andersson]: In our case we always want to download only contacts. We are not interested in the leads. So there is no point in the CRM system giving us the leads unless we specify in certain cases. For example, dynamics by side shop has not actually implemented it yet, but they have explicitly said "we want apsis to download the leads as well." We can specify for each integration: should you do this or not?

### Note on Orphaned Endpoints

The specification may contain endpoints for functionality that was never completed:

[Erik Andersson]: We did add some endpoints for e-commerce systems like we could get abandoned carts, etcetera, but this never took flight. And then a deal left and then we essentially scrapped every plan on making anything with general connector for e-commerce. So there's no point in you having those endpoints.

These are technical debt from exploration that was abandoned and can be ignored.

---

## Incident Recovery: Summary of the Complete Process

### For the Tribe Webhook Bug

**What went wrong:**
- Webhooks were registered for lead entities
- Tribe never sends lead data, so leads remain empty
- System tried to create profiles with empty lead IDs
- Result: junk profiles in Apsis

**How it was fixed:**

1. **Identified the customer** (Master Planning): Query database to find account/section IDs

2. **Confirmed the problem**: Used broker service to call `/v1/webbooks/record` and saw lead webhooks present

3. **Deleted from CRM**: Made DELETE requests via broker to remove both record and consent webhooks for leads

4. **Verified deletion**: Called `/v1/webbooks/record` again and confirmed only contact webhooks remained

5. **Cleaned database**: Executed SQL DELETE statements to remove the webhook subscription records

6. **Result**: Clean state—reinstallation will work correctly and only contact webhooks will be registered

---

## Key Takeaways

1. **Credential Architecture**: Generic Connector uses a fundamentally different (more secure) credential model than legacy Efficy Enterprise. Operators never see customer API keys directly.

2. **Broker Service Is Essential**: All manual debugging of customer CRM instances goes through the broker service, which decrypts credentials server-side and injects them into outgoing requests.

3. **Webhook Management Requires Two-Step Cleanup**: Both API deletion (removes from customer's CRM) and database deletion (removes Apsis internal tracking) are required. Missing database cleanup can cause reinstallation failures.

4. **API vs Database State Can Diverge**: CRM state and Apsis database state must be kept in sync manually during incident recovery.

5. **Query Parameters Are for Debugging**: The generic connector API uses path parameters for required business logic operations and query parameters for optional filtering (primarily debugging scenarios).

6. **Secret Rotation Has Time Costs**: Rotating the integration secret requires restarting the broker service (~20 seconds of manual debugging downtime), but doesn't affect other services.

7. **Process Is Repeatable**: Once you've debugged one customer using the broker service, the exact same pattern applies to any other customer. Only the account ID, section, and integration type values change.

8. **Sanity Checks Are Primary Use Case**: Most broker service usage is for basic sanity checking (does the CRM respond? Do I have permission? Is the instance up?) rather than complex operations.

---

## Unresolved Questions and Observations

1. **Tribe API Bug**: The endpoint returns consistent results, but deletion behavior based on subscription ID (rather than entity/type) was confusing. This appears to be intentional but underdocumented in Tribe's API.

2. **E-Commerce Endpoints**: Orphaned endpoints in the Generic Connector spec for e-commerce (abandoned carts, etc.) should be removed or clearly marked as deprecated.

3. **Database Transactions**: The session used direct SQL DELETEs without explicit transaction handling—unclear if this is safe or if there's a standard pattern for bulk webhook deletion.