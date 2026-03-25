---
source_file: Erik - Inbound Flow.txt
domain: Apsis One Integrations
topics: [Inbound Flow Architecture, Real-time Syncs, Full Syncs, Integration Manager, Mappings Manager, Delta Sync Flow, Webhook Registration, Service Architecture, Database Schema, AWS Infrastructure]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Integration Manager (IM), Mappings Manager (MM), Delta Sync Manager (DSM), Delta Sync Worker (DSW), Sluice Worker (SLW), Full Sync Service, SQS Queues, ECS Tasks, Aurora RDS, Application Load Balancer, Squid Proxy]
session_type: knowledge-transfer
subdomains: [Architecture, Inbound Flow, Generic Connector, Microsoft Dynamics, Efficy Enterprise 12.0, Efficy Enterprise 12.1]
---

## Session Overview

This session covered the **inbound flow** architecture of the Apsis One Integrations platform—how data from external CRM systems is downloaded and synchronized into Apsis One profiles. Erik walked through the core services involved (Integration Manager, Mappings Manager, Delta Sync Manager, and worker services), the two primary data synchronization methods (real-time syncs via webhooks and full syncs via batch downloads), webhook security mechanisms across different CRM connectors, and the underlying infrastructure (ECS tasks, SQS queues, RDS database, ALB routing). The session included architectural deep dives into the application layer, database schema, and AWS infrastructure.

---

## Integration Platform Overview and Core Services

### What is "Inbound Flow"?

**Inbound flow** refers to the ability for the CRM system to download data into Apsis One and to push data to Apsis One in real time. The complete integration use case is:

1. Retrieve data from the CRM system
2. Add it to profiles in Apsis One according to specific field mapping configuration
3. Enable the CRM to push updates to Apsis One in real time
4. Utilize the synchronized profiles to send emails to specific groups or lists

### Service Abbreviations and Naming

[Erik Andersson]: The abbreviations were used extensively when the integration platform was built. There is a pending task to remove these abbreviations from the code structure before handover.

- **IM** = Integration Manager
- **MM** = Mappings Manager  
- **DSM** = Delta Sync Manager
- **SLW** = Sluice Worker
- **DSW** = Delta Sync Worker

### Integration Manager (IM)

The Integration Manager is one of the most central services in the platform. It handles:

- Retrieving the list of integrations that exist (called when viewing the integration page)
- Installation of integrations (clicking "Connect" on a CRM type)
- Updates to existing installations
- Regenerating API keys
- Uninstallation of integrations
- Retrieving metadata from the CRM system (e.g., all fields for the Contact entity)
- Retrieving consent lists and consent bases from the CRM

[Erik Andersson]: Every service in integration is, to some extent, a "CRUD service"—we create, update, delete, or list something in the integration platform.

### Mappings Manager (MM)

The Mappings Manager handles all operations related to field mappings and consent mappings:

- Retrieving existing mappings between CRM attributes and Apsis One attributes
- Creating new field mappings
- Managing subscription/consent mappings between CRM and Apsis One

**Example mapping structure:**
- CRM system field (e.g., `key` in Enterprise) maps to Apsis One field (e.g., `CRM ID`)
- System field maps to Apsis One field
- Consent in CRM maps to Subscription in Apsis One

---

## Service Architecture: Microservices and Deployment

### Microservice Model

[Lukasz Grabowski]: You mean service, not microservice, right? When it comes to deployment, is it one microservice hosted on ECS or...?

[Erik Andersson]: Yes, essentially everything in integration is fronted by one application load balancer. Behind it you have one ECS task for each microservice. Each service has dedicated responsibilities and runs as an ECS task that runs continuously with minimal resources because nothing in integration is resource-heavy, except full syncs and real-time syncs.

### Request Routing and the Holy Trinity

Traffic routing follows a pattern where the domain is followed by the service name:

```
https://integration.apsis-stage.cloud/{service-name}/{endpoint}
```

**Example:**
```
GET https://integration.apsis-stage.cloud/mm/account/{accountId}/section/{sectionId}/integration/{integrationId}/mappings
```

This retrieves all mappings for a specific account, section, and integration.

[Erik Andersson]: Everything in integration is based on this "holy trinity": the **account ID**, **section ID**, and **integration ID**. As soon as you know these three things, you can pinpoint one exact installation or set of mappings. When you get a support request, ask for these three things—it will speed up debugging considerably.

### Terminology: Managers vs. Workers

**Managers** are services that customers interact with through the UI:
- Integration Manager (retrieving integrations, installing, uninstalling)
- Mappings Manager (managing field and consent mappings)
- Full Sync Manager (starting and managing full syncs)

**Workers** are services that read from SQS queues and act on messages:
- Delta Sync Worker
- Sluice Worker
- Full Sync workers (producer and consumer)

Customers never directly interact with workers; workers operate asynchronously in the background.

### Service Deployment and Docker

All services in integration share a unified Docker deployment approach:

- **Single Dockerfile** for all services (previously there was one per service)
- Docker parameters are injected with the service name and image tag
- Images are built for staging, beta, and production separately and stored in ECR (Elastic Container Registry)
- Each service has its own ECR repository

[Lukasz Grabowski]: What kind of load balancer is this? Is it network or application?

[Erik Andersson]: It's an ALB (Application Load Balancer).

### Application Load Balancer Rules

The ALB has 13 rules that route traffic based on the path:

```
IF path == /im THEN forward to Integration Manager ECS task
IF path == /mm THEN forward to Mappings Manager ECS task
IF path == /dsm THEN forward to Delta Sync Manager ECS task
... (etc. for all services)
```

---

## Configuration: Field Mappings and Sync Conditions

### Field Mappings

When you set up field mappings, you specify which CRM fields map to which Apsis One attributes. The mappings manager stores:

- Source field (from CRM)
- Target attribute (in Apsis One)
- Data type (string, integer, float, boolean, timestamp, datetime)

### Sync Conditions

Sync conditions act as filters that determine whether a profile should be synchronized to Apsis One:

**Example:** Only sync profiles where `active = true`

[Erik Andersson]: In most cases, you do not want to sync all inactive profiles in the CRM system because they are probably not updated, or you don't want to send emails to outdated customers. You can set up a sync condition where for each incoming contact, we verify that if the value in the active attribute is true, we sync the contact to Apsis One. If not, we discard the update and don't use up their profile quota.

---

## Real-time Syncs via Webhooks

### Overview

Real-time syncs (also called delta syncs) are triggered by the CRM system when data changes. The CRM pushes updates to Apsis One whenever a field mapping or sync condition field is modified.

### Webhook Registration During Installation

During the installation process:

1. Generate a unique API key that Apsis One provides to the CRM system
2. Generate a webhook secret (a random UUID)
3. Register the webhook in the CRM system, providing:
   - Callback URL pointing to the **Delta Sync Manager**
   - Account ID, Section ID, Integration ID (included in the URL)
   - Webhook secret (stored by the CRM for later verification)
4. If webhook registration fails, the entire installation fails—real-time syncs are critical

[Lukasz Grabowski]: What is this secret? Is this some API key?

[Erik Andersson]: It's just a random UUID that we generate on the fly. We store it in our database and provide the same secret to the CRM system. When they send a request to us, they include the secret, we retrieve it from our database, and verify that they match.

### Webhook Security Mechanisms

The security mechanism varies by CRM connector:

#### Microsoft Dynamics (OAuth2 Flow via OneAPI)

- Only CRM that uses OAuth2 via OneAPI
- Apsis One provides a OneAPI **key and secret** to Dynamics
- Dynamics stores this encrypted and uses it to generate access tokens (following OneAPI recommended storage time)
- When sending real-time sync updates, Dynamics first generates an access token and sends it in the request
- OneAPI validates the token, then forwards to Delta Sync Manager

[Erik Andersson]: Ironically, Microsoft Dynamics is the only CRM with the more sophisticated authentication method because we built a plugin for it using the OneAPI platform. We own the code for this, so we could decide to use proper OAuth2 flow. For other CRMs, they don't support OAuth2, so we had to resort to simpler methods.

#### Efficy Enterprise 12.0 (Static API Key)

- CRM supports providing a shared secret in the header as an API key
- We compare the API key when they send it

#### Efficy Enterprise 12.1 (Generic Connector with HMAC)

- Uses shared secret with request body hashing
- CRM hashes the request body using the shared secret (SHA256)
- Apsis One generates the same hash and verifies the signature matches

#### Generic Connector (HMAC Signature)

- CRM generates a hash of the request body with the shared secret (SHA256)
- Apsis One does the corresponding hash and verifies the signature

[Erik Andersson]: The reason Microsoft is the only one with the sophisticated OAuth2 way is that we control the plugin for Dynamics. For other CRMs, we can't enforce OAuth2 because their webhook systems don't support it.

### What the CRM System Stores

When Apsis One registers a webhook, the CRM typically stores:

- The webhook endpoint URL
- The webhook secret
- The OneAPI key (if using generic connector)
- The OneAPI URL for the specific environment (staging or production)
- The Section ID in Apsis One
- Metadata about the installation

On uninstallation, Apsis One requests that the CRM delete the webhook to prevent further updates and to comply with GDPR (to stop sending sensitive personal data after the agreement ends).

### Delta Sync Manager Endpoint

The Delta Sync Manager is the outer entry point for real-time sync requests. It:

1. Receives the webhook callback from the CRM
2. Verifies the secret in the appropriate way (OAuth, API key, or HMAC signature)
3. Converts the update format to an internal format
4. Places the update on the **sluice queue**

---

## Full Syncs: Batch Downloads

### When to Use Full Syncs

Full syncs are **on-demand** batch downloads initiated by Apsis One, not by the CRM:

1. **Initial setup** – After installing and configuring field mappings, download all existing data from the CRM
2. **Adding new field mappings** – If you add a new field mapping after initial setup (e.g., add birthday field), run a full sync to populate existing profiles with the new field
3. **Adding new consent/subscription mappings** – Run a full sync to download all existing consents from the CRM
4. **Incident recovery** – If there's been an outage or incident (CRM outage, network outage, Apsis One incident, or a bug), run a full sync to re-sync all current data and ensure consistency

[Erik Andersson]: The delta syncs will only send updates for specific contacts that changed. In contrast, the full sync downloads all contacts and all consents from the CRM. If you go in and add the birthday field and didn't include it in your initial setup, we won't have that data. You need to run another full sync so we can populate the birthday field. Otherwise, you'd have to wait until every contact was randomly updated by a real-time sync, which might never happen.

### Full Sync Architecture: Producer-Consumer Pattern

The full sync is the most resource-intensive service in all of integration. It can handle up to 3 million contacts and corresponding consent entries (though typical customers have 50,000–200,000).

**Three components:**

#### 1. Full Sync Manager

- **Responsibility:** Create, list, and manage full syncs
- **Triggered by:** Customer clicking a "Start Sync" button in the UI
- **Actions:** Creates an ECS task pair (producer and consumer) and temporary SQS queue

#### 2. Producer ECS Task

- **Responsibility:** Download all contacts from the CRM system
- **Method:** Paginated requests with multiple threads (default 5 threads)
- **Flow:**
  - Thread 1 downloads page 0
  - Thread 2 downloads page 1
  - Thread 3 downloads page 2
  - etc.
  - If a thread receives an empty list, it stops
  - If all threads receive data, continue to next page range
  - Stop when no more pages are returned
- **Output:** Each contact/consent downloaded is placed as a message on the temporary SQS queue
- **Message types:** 
  - Attribute updates (e.g., first name, last name, email)
  - Consent updates

#### 3. Consumer ECS Task

- **Responsibility:** Process messages from the queue and update Apsis One
- **Method:** 
  - Extract the mapping previously set up in Mappings Manager
  - Make a profile update request to the **HHLS service** (Apsis One's profile management service)
  - Populate corresponding attributes in Apsis One with CRM data
  - For consent updates, call consent-related endpoints
- **Retry logic:** If Apsis One responds with 429 (rate limit) or other errors, put the message back in the queue for retry

### Full Sync Lifecycle

1. Producer downloads all data and places messages on SQS queue
2. Consumer processes messages and updates Apsis One
3. When producer finishes and consumer has drained the queue, the entire temporary stack is decommissioned:
   - Stop the consumer task
   - Stop the producer task
   - Delete the SQS queue
4. This temporary infrastructure doesn't affect other services

---

## The Real-time Sync Flow: Handling Eventual Consistency

### The Problem: Race Conditions Between Full and Real-time Syncs

Imagine this scenario:

1. Customer starts a full sync
2. Full sync downloads Erik's profile with name "Erik"
3. Before the full sync message for Erik is processed, Erik changes his name to "Frederick" in the CRM
4. The real-time sync update for the name change is processed by Delta Sync Manager
5. But then the full sync message is processed, overwriting the new name "Frederick" with the old name "Erik"

Result: **Stale data in Apsis One**

[Erik Andersson]: We really don't want that to happen. What we do is use a service called the sluice worker to implement eventual consistency.

### The Sluice Worker: Coordination Between Real-time and Full Syncs

**Responsibility:** Check if a full sync is currently running for this installation. If so, hold real-time updates until the full sync completes.

**Process:**

1. Delta Sync Manager receives real-time update and places it on the **sluice queue**
2. Sluice Worker constantly consumes messages from the sluice queue
3. For each message:
   - **Query:** Is there an ongoing full sync for this account/section/integration?
   - **If NO:** Push the message to the **delta sync worker queue** immediately
   - **If YES:** Put the message back with a visibility timeout (a few minutes), then check again later
4. Once full sync completes, subsequent real-time updates proceed immediately

[Lukasz Grabowski]: So Delta Sync cannot work without all three components (DSM, Sluice Worker, DSW), right? They all cooperate all the time.

[Erik Andersson]: Yes, if you want to have always correct data in Apsis One, then yes, all three always work in tandem.

### Exception: Efficy Enterprise 12.0

**Special handling due to limited webhook data:**

Efficy Enterprise 12.0 has a unique limitation: they only send changed fields in webhook updates, not the complete contact record.

**Example:** If a customer changes only the first name, Efficy only includes the first name in the update—not the active field.

**Problem:** The sluice worker can't verify sync conditions (e.g., "only sync if active = true") without the active field.

**Solution:** In the sluice worker, for Efficy 12.0 updates, make a callback to the CRM to fetch the full contact data needed for sync condition verification.

[Erik Andersson]: We can't do this in the Delta Sync Manager because it's synchronous—the CRM needs an immediate response, and Efficy sends up to 6 megabytes of data. Making one callback per contact would be impossible. So we do the sync condition check asynchronously in the sluice worker, which can take a few minutes if needed.

This is the one annoying exception we had to introduce for Efficy Enterprise 12.0.

---

## The Delta Sync Worker: Final Processing

### Responsibility

The Delta Sync Worker performs the actual update in Apsis One:

1. Consume messages from the delta sync worker queue
2. Determine message type: profile attribute update or consent update
3. For **attribute updates:** Call the HHLS service with the mapped attributes
4. For **consent updates:** Call consent endpoints

### Data Flow Summary

```
CRM System
    ↓ (webhook POST to Delta Sync Manager)
Delta Sync Manager (verify signature)
    ↓ (put on sluice queue)
Sluice Queue
    ↓
Sluice Worker (check if full sync running)
    ↓ (yes → retry later; no → forward)
Delta Sync Worker Queue
    ↓
Delta Sync Worker (update attributes or consents)
    ↓
HHLS Service / Consent Endpoints
    ↓
Apsis One Profile Updated
```

---

## Differences: Real-time Syncs vs. Full Syncs

| Aspect | Real-time Sync | Full Sync |
|--------|---|---|
| **Initiator** | CRM system (push) | Apsis One (pull) |
| **Data Scope** | Specific contacts/consents that changed | All contacts and consents |
| **Frequency** | Continuous, as changes happen | On-demand, manually triggered |
| **Resource Usage** | Minimal per update; distributed over time | Intense; concentrated period |
| **Use Case** | Keep Apsis One up-to-date with CRM changes | Initial setup, add fields, recover from incidents |
| **Service Chain** | DSM → Sluice → DSW | Full Sync Manager → Producer → Consumer |

---

## Handling Sync Failures and Responsibility

### Shared Responsibility Model

[Lukasz Grabowski]: Whose responsibility is it when webhooks fail? Is this our responsibility or the external system's?

[Erik Andersson]: It depends. If there's an incident or bug in the CRM system and we've never been notified, we have no way of knowing because no real-time sync updates doesn't constitute an error—it might just mean there were no updates. The CRM team is responsible for notifying us if there's a problem.

It's our responsibility if we notice errors in the Delta Sync Manager endpoint or when we receive internal server errors from the CRM. So it's shared responsibility. The CRM team needs to monitor their systems; Apsis One needs to monitor ours.

### Monitoring and Alerts

[Erik Andersson]: Unfortunately, historically, the CRM people do not have as sophisticated processes as Apsis One. We've sometimes received weird internal server errors or 503 gateway timeouts when contacting them. We then notify them in our internal channel, and sometimes they didn't even know their system had an outage.

We don't have alarms that trigger if there's been no traffic for days. If you want to implement that, you can, but it's not a responsibility we've agreed upon.

---

## Database Schema and Storage

### Core Tables and the Holy Trinity

The database uses three IDs as the primary organizing principle: **account_id**, **section_id**, **integration_id**

#### Installations Table
- Stores all integration installations
- Fields:
  - `account_id`, `section_id`, `integration_id` (unique together)
  - `one_api_key` – API key generated for this installation
  - `audience_id` – ID of the key space in Apsis One
  - Metadata (created_at, updated_at, etc.)

[Erik Andersson]: You don't allow multiple installations of the same integration for the same account and section. This triple is a unique constraint.

#### Field Mappings Table
- Account, section, integration IDs
- CRM field identifier
- Apsis One attribute ID
- Data type (string, integer, float, boolean, timestamp, datetime)

#### Consent Mappings Table
- Account, section, integration IDs
- CRM consent name
- Apsis One subscription ID (historically called "topic")

#### Sync Conditions Table
- Account, section, integration IDs
- CRM field to check
- Data type
- Comparison value (currently only "equals" is supported)

#### Squid Domains Table
- Account, section, integration IDs
- Domain URL (e.g., mycrmdomain.com)
- Stores which domains are whitelisted for this installation (used by reverse proxy)

#### Connections Table
- Stores connection metadata (e.g., API endpoints) for different CRM types
- Used by generic connector

#### Credentials Table
- Stores API keys and secrets for each installation
- Points to existing connections

#### Connector-Specific Tables
- **dynamics_connections** – Stores Microsoft Dynamics tenant IDs
- **lime_connections** – Stores Lime CRM-specific metadata

### Database Infrastructure

- **Type:** Aurora RDS (Postgres)
- **Network:** Private network, not accessible from outside
- **Access methods:**
  - Via load balancer
  - Via AWS Query Editor (inside VPC)
  - SSH tunnel via bastion host

### Schema Management

**Files:**

- `schema.sql` – Creates all tables from scratch
- `data_static.sql` – Populates static data (e.g., sync statuses like "syncing", "done", "cancelled")
- `delta.sql` – Pending database changes to be applied before next release

[Erik Andersson]: We don't have a very sophisticated way of handling database migrations in Golang. Anytime we do a feature or bug fix involving database changes, we add the required change to the delta file. When we do a release, we manually run these changes on the database and then clear the file.

I would have really loved to have an automated migration system like in Django, but we've not had luck implementing that in Golang.

[Lukasz Grabowski]: For example, in EventTool we implemented this at the very beginning. So that was a good decision, because otherwise it would be difficult to add migrations now.

[Erik Andersson]: Yeah, I kind of missed that from Django. I assume Node probably has nice ways of doing that as well.

### Docker Compose and Local Testing

The project includes a `docker-compose.yml` that sets up everything needed to run the integration platform locally:

- Creates and runs a Postgres database
- Automatically applies `schema.sql` and `data_static.sql` on startup
- Enables unit tests to tear down and recreate the database before/after each test

---

## AWS Infrastructure and Deployment

### Networking and Load Balancer

- **ALB (Application Load Balancer)** with 13 routing rules
- **Domain:** Points directly to the ALB
  - Staging: `integration.apsis-stage.cloud`
  - Production: `integration.apsis.one`
- **Database:** Aurora RDS in private network (no public access)
- **ECS Tasks:** Deployed individually behind the ALB, each with minimal resources

### Deployment Process

1. Build Docker image for each service
2. Push to ECR (Elastic Container Registry)
3. Deploy to ECS as a long-running task (except full syncs, which are on-demand)

### SQS Queues

- Permanent queues for continuous workers (sluice queue, delta sync worker queue)
- Temporary queues created during full syncs (deleted when sync completes)

---

## Architecture Code Structure

### Repository Organization

The `app/` folder contains all microservices:

```
app/
├── integration_manager/        (was "IM")
├── mappings_manager/           (was "MM")
├── delta_sync_manager/         (was "DSM")
├── delta_sync_worker/          (was "DSW")
├── sluice_worker/              (was "SLW")
├── full_sync_manager/
├── full_sync_process/          (starts producer, consumer, queues)
├── outbound_manager/
├── outbound_worker/
├── Dockerfile                  (shared for all services)
└── ...
```

**Key point:** All services share a single Dockerfile. Service name and image tag are injected as parameters during build.

### Configuration Files

- Database setup files in dedicated folder
- Environment-specific configurations
- Makefile scripts for local development and deployment

---

## Security: The Squid Proxy

### Purpose

Squid Proxy is an additional security layer that filters outgoing traffic based on destination domain.

[Erik Andersson]: In AWS, security rules can only filter by IP addresses or IP ranges. When a customer installs an integration on domain `mycrmdomain.com`, we want to filter outgoing traffic only to that domain. If someone injects bad code into our services, they can't reroute traffic to an obscure server somewhere because we only whitelist the domains customers have installations for. If they try to send customer data to `myevilsite.com`, that wouldn't work because we haven't whitelisted that domain.

### How It Works

1. All traffic from integration services is routed through Squid Proxy
2. For each request, Squid checks: Does this destination domain have a whitelist entry for this account/section/integration?
3. If yes, allow the request
4. If no, block the request

### Implementation Note

[Erik Andersson]: This requires understanding of reverse proxies, different whitelist logic, handling different connector types, etc. This is something we should cover separately, maybe in a dedicated session.

[Lukasz Grabowski]: I think it would be reasonable to have a separate meeting, maybe half an hour or an hour.

[Erik Andersson]: Half an hour will not be enough, I can tell you that.

---

## Key Takeaways

1. **The holy trinity (account ID, section ID, integration ID)** uniquely identifies every installation and mapping in the system. Always use these to debug issues.

2. **Integration Manager and Mappings Manager** are the customer-facing services. All other services are workers that operate asynchronously in the background.

3. **Real-time syncs** are pushed by the CRM via webhooks; **full syncs** are pulled by Apsis One when needed. Both are essential.

4. **Webhook security** varies by CRM:
   - Microsoft Dynamics uses OAuth2 via OneAPI
   - Efficy Enterprise 12.0/12.1 and generic connector use HMAC signatures
   - Others use simple API key comparison

5. **The sluice worker** prevents race conditions by holding real-time updates until ongoing full syncs complete—critical for data consistency.

6. **Efficy Enterprise 12.0** is a special case: it only sends changed fields in webhooks, requiring a callback to the CRM in the sluice worker to verify sync conditions.

7. **Full sync architecture** uses a producer-consumer pattern with a temporary SQS queue. It's the most resource-intensive service and scales to millions of contacts.

8. **Database migrations** are manual (not automated) but tracked in the delta.sql file.

9. **All services are ECS tasks** behind a single ALB with path-based routing rules.

10. **Squid Proxy** whitelists destination domains per installation to prevent data exfiltration if code is compromised.

---

## Unresolved Questions and Next Steps

1. **Squid Proxy deep dive** – Requires a separate 1+ hour session to cover whitelist logic, domain filtering per installation, and different connector handling.

2. **Abbreviation removal** – Planned task to rename folders and services from abbreviations (IM, MM, DSM, etc.) to full names. Impact on external endpoints to be assessed.

3. **Database migration automation** – No automated migration framework currently in use (unlike Django or Node). Could be a future refactoring project.

4. **Endpoint naming**: Whether to expose full service names in API endpoints or keep abbreviated paths (e.g., `/integration_manager` vs. `/im`). URL redirects could mitigate breaking changes.