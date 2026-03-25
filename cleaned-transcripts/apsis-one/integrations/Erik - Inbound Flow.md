---
source_file: Erik - Inbound Flow.txt
domain: Apsis One Integrations
topics: [Inbound Flow Architecture, Integration Manager, Mappings Manager, Real-time Synchronization, Full Synchronization, Delta Sync Flow, Webhook Registration, Data Consistency, Database Schema, Deployment Architecture]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Integration Manager (IM), Mappings Manager (MM), Delta Sync Manager (DSM), Sluice Worker (SLW), Delta Sync Worker (DSW), Full Sync Manager, Full Sync Producer/Consumer, SQS Queues, Application Load Balancer (ALB), ECS Tasks, Aurora RDS (PostgreSQL), ECR, Squid Proxy, Reverse Proxy]
session_type: knowledge-transfer
subdomains: [Architecture, Inbound Flow, Generic Connector, Microsoft Dynamics, Efficy Enterprise 12.0]
---

## Session Overview

This knowledge transfer session covers the **Inbound Flow** of the Apsis One Integrations platform—the architecture and mechanisms by which data flows from external CRM systems into Apsis One. The session explains the central services involved (Integration Manager, Mappings Manager, Delta Sync Manager), the two primary sync patterns (real-time delta syncs via webhooks and on-demand full syncs), the critical handling of eventual consistency through the Sluice Worker, and the supporting infrastructure including database schema, deployment architecture via ECS/ALB, and security layers like Squid Proxy. This is a continuation from a previous session on connector types.

---

## Core Concept: What is Inbound Flow?

The **inbound flow** is the mechanism that enables bi-directional data exchange between external CRM systems and Apsis One:

1. **Download data from CRM → Add to Apsis profiles**: Data is retrieved from the external CRM system based on **field mappings** the customer has configured.
2. **Push data to Apsis in real time**: When the CRM system initiates changes, Apsis One receives and processes them immediately.
3. **Enable use cases**: Once profiles exist in Apsis with current data, the platform can utilize them for email campaigns and other marketing operations.

[Erik Andersson]: The integration platform essentially enables downloading data from CRM systems and adding it to profiles in Apsis according to specific field mappings, and also allows the CRM system to push data to Apsis in real time.

---

## Integration Manager (IM): The Central Hub

The **Integration Manager** is one of the most critical services in the entire integration platform. It is responsible for:

- **Retrieving integrations**: When customers access the integration page, IM lists all available and installed integrations.
- **Installation/Updates**: When a customer clicks "Connect" to install an integration, IM handles the entire installation workflow.
- **API key management**: IM regenerates and manages the one-API keys for each installation.
- **Uninstallation**: IM coordinates removal of installations.
- **Metadata retrieval**: IM retrieves fields from the CRM entity (e.g., which fields exist for the "contact" entity in Dynamics).
- **Consent/subscription lists**: IM fetches all consent bases and consent lists from the CRM and makes them available in the UI.

Every service in the integration platform follows the **CRUD pattern**: Create, Read, Update, Delete. Integration Manager is a prime example.

### Naming and Abbreviations

> We were very fond of abbreviating everything, which unfortunately makes the codebase harder to maintain.

Current abbreviations used throughout the platform:
- **IM** = Integration Manager
- **MM** = Mappings Manager
- **DSM** = Delta Sync Manager
- **SLW** = Sluice Worker
- **DSW** = Delta Sync Worker

[Erik Andersson]: There is a task to remove these abbreviations from the code structure so the code will be more maintainable, though this is not yet complete.

---

## The Holy Trinity: Identifying Unique Installations

Every request in the integration platform relies on three core identifiers:

1. **Account ID**
2. **Section ID**
3. **Integration ID**

Together, these three values uniquely identify one exact installation, a set of mappings, or any configuration state. 

[Erik Andersson]: When you get a support request, you need to ask SOC to provide these three things because it will speed up your debugging considerably.

### Example API Request Structure

```
https://integration.abscess.cloud/mm/accounts/{account_id}/sections/{section_id}/integrations/{integration_id}/mappings
```

This retrieves all mappings for a specific installation.

---

## Mappings Manager (MM): Storing Field Relationships

The **Mappings Manager** handles the configuration of how data flows between CRM and Apsis:

### Field Mappings
Maps individual CRM fields to Apsis One fields. For example:
- CRM field `key` → Apsis field `email`
- CRM field `firstName` → Apsis field `first_name`
- CRM field `birthDate` → Apsis field `description` (custom mapping)

### Consent Mappings
Maps CRM consent/subscription types to Apsis subscriptions:
- CRM consent `marketing_email` → Apsis subscription `marketing_communications`

### Type Support
Integration currently supports these field types:
- String
- Integer
- Float
- Boolean
- Timestamp (added recently)
- DateTime (added recently)

---

## Sync Conditions: Filtering Which Profiles to Sync

A **sync condition** is a filter that determines whether a profile from the CRM should be synced to Apsis One.

Example: A customer may not want to sync inactive profiles, so they configure:
- **Field**: `active` (from CRM)
- **Condition**: equals
- **Value**: `true`

Result: Only contacts where `active = true` are synced to Apsis. Others are discarded, preventing quota waste.

[Erik Andersson]: You don't want to sync all inactive profiles in the CRM system because they are probably not updated or you don't want to send emails to outdated customers.

---

## Two Paths for Data Ingestion: Delta Syncs vs. Full Syncs

### Delta Syncs (Real-Time Synchronization)

**When**: Whenever a contact or consent is modified in the CRM system.

**How it works**:
1. During installation, a **webhook** is registered in the CRM system.
2. The webhook points to the Apsis Delta Sync Manager endpoint and includes a secret.
3. When data changes in the CRM, the CRM pushes a notification to the webhook.
4. The update flows through the system: DSM → Sluice Queue → Sluice Worker → Delta Sync Queue → Delta Sync Worker → Apsis.

**Triggered by**: Changes in the CRM (contact update, consent change).

### Full Syncs (On-Demand Synchronization)

**When**: Initiated by the customer from the UI.

**Use cases**:
1. **Initial setup**: Download all existing data from CRM to populate Apsis for the first time.
2. **New field mappings**: After adding new field mappings, run a full sync to backfill historical data.
3. **New consent mappings**: After adding new consent mappings, run a full sync to get all consent data.
4. **Incident recovery**: If webhooks were missed due to CRM outage, network issues, or internal Apsis bugs, a full sync reconciles the data.

[Erik Andersson]: The full sync is the most resource-intense service in integration. We've tested it with up to 3 million contacts and 3 million consent entries. Your typical customer has 50,000 to 70,000 contacts, but 200,000 is also common.

**Triggered by**: Customer action or operational recovery.

---

## Webhook Registration and Secrets

### When Webhooks Are Set Up

During the **installation process**:

1. Customer provides the URL to their CRM system and their API credentials.
2. Integration Manager calls the CRM system to register a webhook.
3. The webhook URL points to Apsis (the Delta Sync Manager endpoint).
4. A **secret** is generated, stored in Apsis database, and sent to the CRM to store.

### Secret Management by CRM Type

#### Microsoft Dynamics (OAuth2 Flow)
- Apsis generates a **one-API key**.
- The one-API key is sent to Dynamics and stored encrypted.
- When Dynamics sends a webhook update, it first generates an access token using the one-API credentials.
- The token follows standard OAuth2 recommended storage time.
- When the token expires, Dynamics generates a new one.
- The request then goes through Apsis's one-API service, which validates and forwards to the Delta Sync Manager.

[Erik Andersson]: Dynamics is the only CRM we control the plugin for, so we could enforce OAuth2. This is more sophisticated than other connectors.

#### Efficy Enterprise 12.0
- CRM stores a simple **shared secret** (random UUID).
- CRM includes the secret in the request header as an API key.
- Apsis compares the received secret with the stored secret.

#### Efficy Enterprise 12.1 (Generic Connector) and Generic Connector
- CRM stores a **shared secret** (random UUID).
- CRM hashes the request body using SHA256 (or similar) with the secret.
- CRM includes the hash signature in the request.
- Apsis regenerates the same hash on its side and verifies the signature matches.

[Erik Andersson]: For the generic connector, they will hash the request body using the secret. We will then do the corresponding hash on our side and verify that the signature matches.

### What Else Is Stored on the CRM System?

Beyond the webhook secret and configuration:
- **One-API key** (if they support custom integrations via one-API).
- **One-API URL** for the environment (staging vs. production).
- **Section ID** (so the CRM can reference the Apsis section).
- **Integration ID** (for reference in custom use cases).

[Erik Andersson]: On uninstallation, we ask the CRM to delete the webhooks so they don't continue sending sensitive personal data—this is also a GDPR requirement.

---

## Real-Time Sync Flow: Delta Sync Manager → Sluice Worker → Delta Sync Worker

### The Problem: Eventual Consistency

A critical issue exists when a **full sync and a delta sync happen simultaneously**:

1. Full sync starts and begins downloading contacts.
2. While the full sync is still processing messages in the queue, a contact is modified in the CRM.
3. The modification (delta sync) is received and processed **faster** than the full sync message.
4. The old data from the full sync overwrites the new data from the delta sync.

**Result**: Profile data becomes stale and inconsistent.

### The Solution: The Sluice Worker

The **Sluice Worker** (SLW) ensures consistency:

1. **Delta Sync Manager (DSM)** receives the webhook update from the CRM.
2. DSM verifies the webhook signature (using the secret).
3. DSM converts the update to the internal format and puts it on the **Sluice Queue**.
4. **Sluice Worker** polls the Sluice Queue and checks:
   - **Is a full sync currently running for this installation?**
   - If **NO**: Push the message to the Delta Sync Queue immediately.
   - If **YES**: Add a visibility timeout (a few minutes) and retry the message later.
5. Once the full sync completes, the Sluice Worker can release queued delta syncs.
6. **Delta Sync Worker (DSW)** consumes from the Delta Sync Queue and performs the actual update in Apsis.

[Lukasz Grabowski]: So once again to sum up this flow, when the change is done on the CRM, they call us this Delta Sync Manager URL or API, right? Then what we do with this request, we just put it to the queue or?

[Erik Andersson]: We verify the secret in whatever way... We take the update and then put it on the sluice queue. So you have like DSM which then goes DSM to sluice queue to sluice worker to Delta sync worker queue to delta sync worker...

### Flow Diagram
```
CRM Webhook → DSM → Sluice Queue → Sluice Worker → Delta Sync Queue → Delta Sync Worker → Apsis
```

---

## Exception: Efficy Enterprise 12.0 Sync Condition Verification

**Problem**: Efficy Enterprise 12.0 only includes **changed fields** in webhook updates, not the full contact record.

Example: If a contact's first name changes but `active = false`, the webhook only contains the first name. Apsis doesn't know if the contact should be synced based on the `active` sync condition.

**Solution**: For FCC Enterprise 12.0 only, the **Sluice Worker** makes a **callback request** to the CRM system to fetch the full contact data (including the `active` field and other sync condition fields).

[Erik Andersson]: We cannot have that check in the Delta Sync Manager because the Delta Sync Manager is synchronous. If we have to make a callback for each contact and they send 5,000 contacts at once, that is not something you can handle in a synchronous call.

**Why not in DSM?**: DSM is synchronous and must respond immediately to the CRM. Making 5,000 callback requests would time out.

**Why in Sluice Worker?**: SLW is asynchronous. It's acceptable for a message to take 5 minutes to process.

> This is the one annoying exception that exists. Otherwise it is just: is there a full sync running? Yes, no.

---

## Full Sync Architecture: Three Components

When a customer initiates a full sync:

### 1. Full Sync Manager (FSM)
- **Responsibility**: Orchestration and lifecycle management.
- **Creates** new full sync jobs (stores state in database).
- **Lists** existing full sync jobs.
- **Cancels** running full sync jobs.
- **Always running**: Customers interact with FSM via the UI.

### 2. Producer ECS Task (On-Demand)
- **Spawned** when a full sync is triggered.
- **Downloads** all contacts from the CRM system using **pagination** with **5 concurrent threads**.
  - Thread 0 downloads pages 0, 5, 10, 15, ...
  - Thread 1 downloads pages 1, 6, 11, 16, ...
  - And so on.
- **Stops** when all threads receive empty responses (no more pages).
- **Produces** messages to a temporary **SQS queue**.
- **Message types**:
  - Attribute updates (e.g., email, first_name, birthday).
  - Consent updates.

### 3. Consumer ECS Task (On-Demand)
- **Consumes** messages from the SQS queue.
- **Extracts** field mappings from Mappings Manager.
- **Calls Apsis APIs**:
  - For attribute updates: calls **HLS service** with profile update requests.
  - For consent updates: calls **consent endpoints**.
- **Handles failures**: If Apsis returns 429 or errors, the message is requeued.
- **Stops** when queue is empty and producer is done.

### Lifecycle
```
Full Sync Triggered
  ↓
FSM creates SQS queue
FSM spawns Producer ECS task
FSM spawns Consumer ECS task
  ↓
Producer downloads pages (5 threads) → SQS queue
Consumer processes messages → Apsis updates
  ↓
Producer finishes, queue empties, Consumer finishes
  ↓
FSM deletes SQS queue and terminates both ECS tasks
```

### Resource Intensity

[Erik Andersson]: The full sync service is probably the most resource-intense service in the whole of integration because we download everything. We've tested this up to 3 million contacts and the corresponding 3 million consent entries.

Each full sync has dedicated, temporary ECS tasks, so multiple full syncs running simultaneously do not interfere with each other. However, they all place load on Apsis (the audience service), which is why full syncs must be coordinated.

---

## Service Responsibilities and Characteristics

### Managers vs. Workers

**Managers**:
- Customer-facing; respond to UI actions.
- Stateless; handle individual requests.
- Always running as persistent ECS tasks.
- Examples: Integration Manager, Mappings Manager, Full Sync Manager.

**Workers**:
- Consume messages from SQS queues.
- Act asynchronously based on message content.
- Always running (except full sync producer/consumer, which are on-demand).
- Examples: Sluice Worker, Delta Sync Worker.

### Resource Profile

[Erik Andersson]: Nothing in integration is resource-heavy apart from the full syncs and the real-time syncs. Each service runs on minimal ECS resources because they are not compute-intensive. The main load comes from network I/O (receiving from CRM, writing to Apsis).

---

## Deployment Architecture

### Application Load Balancer (ALB)

One **ALB** fronts the entire Apsis One Integrations platform. It has **routing rules** (currently 13) that direct traffic to the correct service:

```
Request: https://integration.abscess.cloud/mm/...
  ↓
ALB receives request
  ↓
ALB checks path rule: if path contains "/mm"
  ↓
Route to Mappings Manager ECS task
```

Each service has its own routing rule and target group.

### ECS Tasks

Each microservice (IM, MM, DSM, SLW, DSW, etc.) runs as a dedicated **ECS task** in a private network.

- **Always running**: Managers and persistent workers.
- **On-demand**: Full sync producer/consumer tasks.
- **Minimal resources**: CPU and memory are kept low.
- **Scaling**: If needed, task counts can be increased, but historically not necessary.

### Database: Aurora RDS (PostgreSQL)

- **Type**: Amazon Aurora PostgreSQL.
- **Location**: Private network (no external access).
- **Access methods**:
  - Via load balancer (from ECS tasks).
  - Direct SSH tunnel via bastion host (for debugging).
  - Query Editor (AWS console).

### SQS Queues

- **Permanent queues**: Sluice Queue, Delta Sync Queue.
- **Temporary queues**: One created per full sync job; deleted when sync completes.

### Squid Proxy (Reverse Proxy)

A **reverse proxy** layer filters all outgoing traffic from Apsis integrations:

**Purpose**: Security. Prevent injection attacks that might redirect customer data to malicious domains.

**How it works**:
1. During installation, the customer provides the URL to their CRM system (e.g., `mycrmdomain.com`).
2. This domain is **whitelisted** in Squid Proxy for that installation.
3. All outgoing requests from Apsis integration services are routed through Squid.
4. Squid checks: Does this destination domain exist in the whitelist for this customer?
5. If yes: Request proceeds. If no: Request is blocked.

[Erik Andersson]: We filter the outgoing traffic based on the destination domain. If someone were to be able to inject bad code into our services, they cannot start rerouting traffic to some obscure server somewhere. If they try to send customer data to myevilsite.com, that would not work because we have not added a whitelisting for that domain.

---

## Database Schema and Tables

### Key Tables

#### `installations`
Stores each CRM integration installation.

Columns include:
- `account_id` (part of holy trinity)
- `section_id` (part of holy trinity)
- `integration_id` (part of holy trinity)
- `one_api_key`: The API key generated for this installation.
- `audience_id`: The key space ID in Apsis (for direct API calls).
- `created_at`, `updated_at`: Timestamps.

Constraint: Only one installation per `(account_id, section_id, integration_id)` tuple.

#### `mappings`
Stores field mappings between CRM and Apsis.

Columns:
- `account_id`, `section_id`, `integration_id` (holy trinity).
- `crm_field_id`: The field name in the CRM system (e.g., "email_one").
- `apsis_field_id`: The target attribute in Apsis (e.g., "email").
- `field_type`: One of `string`, `integer`, `float`, `boolean`, `timestamp`, `datetime`.

#### `consent_mappings`
Maps CRM consents to Apsis subscriptions.

Columns:
- `account_id`, `section_id`, `integration_id`.
- `crm_consent_id`: Consent in the CRM.
- `apsis_topic_id`: Subscription in Apsis (historically called "topic").

#### `sync_conditions`
Stores filtering conditions.

Columns:
- `account_id`, `section_id`, `integration_id`.
- `field_id`: The CRM field to check.
- `field_type`: Type of the field.
- `condition_value`: The value to compare against.
- Currently only supports **equality** comparisons (e.g., `active = true`).

#### `squid_domains`
Whitelisted domains for Squid Proxy.

Columns:
- `account_id`, `section_id`, `integration_id`.
- `domain`: The CRM domain (e.g., "mycrm.example.com").

#### `connections`
Generic connector credential management.

Columns:
- `account_id`, `section_id`, `integration_id`.
- API credentials and metadata for generic connector installations.

#### `connector_<name>` (Legacy)
Some older connectors have custom tables (e.g., `connector_dynamics`) for CRM-specific data like tenant IDs. Modern connectors use the generic `connections` table.

#### `sync_statuses` (Static Data)
Enumeration of full sync states:
- `syncing`: Currently in progress.
- `cancelled`: User cancelled it.
- `completed`: Finished successfully.

### Schema Management and Migrations

**Schema Creation**: The `schema_create.sql` file defines all tables from scratch.

```sql
-- Executed when setting up a new AWS account
psql -d integration_db -f schema_create.sql
```

**Static Data**: The `data_static.sql` file populates reference data (e.g., sync statuses).

**Migrations**: The `delta` file accumulates pending database changes.

[Erik Andersson]: We currently do not really have a very sophisticated way of handling database migrations. Anytime we do a new feature or bug fix that modifies the database, we add the required change here in the delta file. Whenever we do a release, we manually run these changes from the delta on the database and then empty the file.

> This is not ideal. Historically this approach worked, but frameworks like Django or Node migrations would be better. This is a known technical debt.

**Docker Compose Integration**: For local development and testing, the Docker Compose file automatically runs migrations on boot:

```yaml
# docker-compose.yml
postgres:
  image: postgres:13
  volumes:
    - ./database/schema_create.sql:/docker-entrypoint-initdb.d/01-schema.sql
    - ./database/data_static.sql:/docker-entrypoint-initdb.d/02-data.sql
```

When the local environment starts, the database is automatically initialized.

---

## Code Repository Structure

### Application Services: `/app`

Each microservice has its own folder:

```
/app
  /delta_sync_manager      → DSM service code
  /delta_sync_worker       → DSW service code
  /full_sync_manager       → FSM service code
  /full_syncs              → Full sync producer/consumer logic
  /integration_manager     → IM service code
  /mappings_manager        → MM service code
  /sluice_worker           → SLW service code
  /outbound_manager        → Outbound flow (covered later)
  /outbound_worker         → Outbound flow (covered later)
  Dockerfile               → Single Dockerfile for all services
```

### Single Dockerfile for All Services

Instead of maintaining separate Dockerfiles, all services use one shared Dockerfile with parameterized injection:

```dockerfile
# Dockerfile (simplified)
FROM golang:1.18
WORKDIR /app
COPY . .
ARG SERVICE_NAME
ARG IMAGE_TAG
RUN go build -o bin/${SERVICE_NAME} ./cmd/${SERVICE_NAME}
ENTRYPOINT ["./bin/${SERVICE_NAME}"]
```

Build command:
```bash
docker build \
  --build-arg SERVICE_NAME=delta_sync_manager \
  --build-arg IMAGE_TAG=v1.2.3 \
  -t {ecr_uri}/delta_sync_manager:v1.2.3 \
  .
```

**Benefit**: Changes to build process (e.g., Go version, base image) apply to all services at once.

### Docker Compose for Local Development

```
/docker-compose.yml
```

Includes:
- PostgreSQL database (initialized with `schema_create.sql` and `data_static.sql`).
- LocalStack for SQS emulation.
- Moto for Apsis one-API mocking.
- All integration services configured to communicate locally.

[Erik Andersson]: We have set up a script to set up the tunnel and then scripts to also connect to the actual database. You can handle this whatever way you want, because I think the setup is identical.

---

## Artifact Storage: ECR

Docker images are stored in **Amazon ECR** (Elastic Container Registry):

- **One repository per service** (e.g., `integration-delta-sync-manager`, `integration-mappings-manager`).
- **Separate images per environment**: Staging, Beta, Production.
- **Tagged with version**: e.g., `v1.2.3`.

[Lukasz Grabowski]: Images storage. Is it root on AWS?

[Erik Andersson]: No, it is unique for each account. So we are building the image for staging, then we're building the image for beta, then building the image for prod.

---

## Addressing Webhook Failure and Responsibility

### Shared Responsibility Model

**CRM Team Responsibility**:
- Monitor their own systems for outages or errors.
- Notify Apsis if webhooks fail to deliver.
- CRM has no way to know if Apsis received the webhook unless we explicitly report an error (5xx response).

**Apsis Integration Team Responsibility**:
- Monitor the Delta Sync Manager for errors (5xx responses, processing failures).
- Proactively alert if messages are dropped due to bugs.
- Monitor for internal server errors when calling back to the CRM.

[Erik Andersson]: The boring answer would be it depends where the problem is. But like of course there it is a shared responsibility. The CRM team need to monitor their issues. Apsis needs to monitor our systems. Unfortunately, historically, the CRM or the FCC CRM people do not have as sophisticated processes as Apsis has had.

### Example: Missed Incidents
Sometimes the CRM team has not had alerts set up and did not realize their system had an outage. Apsis noticed (received 503 errors) and had to notify them via internal channels.

### Full Sync as Recovery
If webhooks were missed or data became inconsistent, the customer can always run a full sync to reconcile.

---

## Efficiency Considerations

### When to Use Full Sync vs. Real-Time Sync

**Full Sync** is designed for:
- Large, one-time downloads (millions of records).
- Dedicated ECS tasks per sync.
- Doesn't interfere with other operations.

**Real-Time Sync** is designed for:
- Individual or small batch updates (1–200 contacts per webhook).
- Shared infrastructure (all integrations use same DSM, workers).
- Low latency, always-on.

[Erik Andersson]: Full sync is the way to go if you need to handle a massive amount of contacts. Full sync is designed to handle a copious amount of data. Whereas the real-time syncs, the idea behind it is someone made one update or they might batch up to 200 contacts at the time. The real-time sync flow is not designed for them to say, hello here is 200,000 contacts that were updated.

---

## Domain Setup: Route 53 and ALB

### Domain Configuration

**Staging**: `integration-stage.abscess.cloud`
**Production**: `integration.abscess.one`

Both domains point to their respective ALBs via **Route 53 DNS** records.

[Lukasz Grabowski]: Can you open it? Which one is this? Probably the last one. See name.

[Erik Andersson]: Here you would route the traffic and then from a list you will pick which load balancer or whatever service you want to direct the traffic to.

Setup is handled via **CloudFormation templates**, which define the ALB rules, target groups, and DNS records.

---

## Key Takeaways

1. **The holy trinity** (account_id, section_id, integration_id) uniquely identifies every installation and is essential for debugging and database queries.

2. **Integration Manager (IM)** is the central hub for all customer-facing operations: installation, metadata retrieval, uninstallation, and configuration management.

3. **Mappings Manager (MM)** stores and retrieves the field mappings and consent mappings that define how data flows between CRM and Apsis.

4. **Delta Sync (real-time)** is push-based: The CRM sends webhooks whenever data changes. It's fast but limited to individual or small-batch updates.

5. **Full Sync (on-demand)** is pull-based: Apsis downloads all data from the CRM. It's resource-intensive but essential for initial setup, backfilling new fields, and disaster recovery.

6. **Webhook secrets** vary by CRM type:
   - Dynamics uses OAuth2 with one-API (most secure).
   - Generic Connector uses HMAC-SHA256 signature verification.
   - Others use simple API key comparison.

7. **Eventual consistency problem**: When a full sync and delta sync overlap, old data can overwrite new data. The **Sluice Worker** prevents this by delaying delta syncs until the full sync completes.

8. **Efficy Enterprise 12.0 exception**: Because it only sends changed fields, the Sluice Worker must call back to verify sync conditions asynchronously. This is the only CRM with this special handling.

9. **Deployment**: One ALB routes traffic to permanent managers and workers (ECS tasks), plus on-demand full sync producer/consumer tasks. All run in private networks accessed via ALB, bastion, or query editor.

10. **Database migrations** are manual (delta file approach), not automated. This is technical debt but has been stable. Each new AWS account requires manual schema + data initialization.

11. **Squid Proxy** adds a security layer by whitelisting domains on a per-installation basis, preventing injection attacks from redirecting customer data.

12. **Service distribution**: Manager services are customer-facing and stateless. Worker services are asynchronous and process queued messages. All run on minimal resources except full sync.

---

## Unresolved Questions and Action Items

### Topics for Separate Sessions

1. **Squid Proxy deep dive**: How the reverse proxy whitelist works, how domains are added/removed, filtering logic. [Scheduled for next week, 30 min to 1 hour session recommended]

2. **Database migration strategy**: Whether to adopt an automated migration framework (e.g., Goose, Migrate) to replace the manual delta file approach.

3. **Abbreviation refactoring**: Decide scope—repository only vs. API endpoints. If API endpoints are changed, implement URL redirects (e.g., `/IM` → `/integration_manager`).

4. **Image promotion across environments**: Evaluate whether to build images once in staging and promote to beta/prod (as Felix suggested) vs. current per-environment builds.

### Clarifications Provided

- [Lukasz]: Are services microservices or just services? → [Erik]: Compared to other apps services, yes, they would be considered microservices with dedicated responsibilities, each running as an ECS task.

- [Lukasz]: How is webhook secret generated and stored? → [Erik]: Random UUID generated on-the-fly, stored in Apsis DB and sent to CRM during installation.

- [Lukasz]: Is webhook authentication sophisticated (JWT, etc.)? → [Erik]: Varies by CRM. Dynamics uses OAuth2 (sophisticated). Others use simple shared secret or HMAC (basic).

- [Michal]: For Efficy 12.0, if they bulk-update profiles and send only changes, won't this cause massive callbacks? → [Erik]: Yes, callbacks are per-message (asynchronous in Sluice Worker), which is acceptable since it's not synchronous.

- [Lukasz]: Is image storage on AWS root? → [Erik]: No, each environment (staging, beta, prod) has its own ECR account/images.

---

## Next Steps

1. **Immediate**: Team to review repository structure (`/app` folders) to locate services for modifications.
2. **Short-term**: Schedule separate 30–60 min session on Squid Proxy architecture.
3. **Review**: Decide on database migration framework adoption.
4. **Documentation**: Add this session's notes to the team's integration platform wiki.