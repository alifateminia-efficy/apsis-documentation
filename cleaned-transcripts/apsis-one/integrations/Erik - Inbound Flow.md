---
source_file: Erik - Inbound Flow.txt
domain: Apsis One Integrations
topics: [Inbound Data Flow, Integration Manager, Mappings Manager, Delta Sync, Full Sync, Real-time Syncs, Webhook Management, Field Mappings, Consent Mappings, Sync Conditions, ECS Architecture, Database Schema, Eventual Consistency, Sluice Worker]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Integration Manager (IM), Mappings Manager (MM), Delta Sync Manager (DSM), Delta Sync Worker (DSW), Sluice Worker (SLW), Full Sync Manager, Full Sync Process, SQS Queues, ECS Tasks, Application Load Balancer, Aurora RDS, Squid Proxy]
session_type: knowledge-transfer
subdomains: [Architecture, Lead creation]
---

## Session Overview

This session covers the **inbound flow** of the Apsis One Integrations platform—the mechanisms by which data flows from external CRM systems into Apsis One. The discussion explains how integrations are managed, how field and consent mappings work, the two primary data synchronization patterns (real-time delta syncs and on-demand full syncs), the webhook architecture and secret verification methods, and the critical eventual consistency pattern implemented via the Sluice Worker to prevent data race conditions. The session also covers infrastructure components (ECS tasks, load balancing, database structure) and tooling for local development.

---

## Integration Platform Overview and Core Services

### The Integration Manager (IM)

The **Integration Manager** is one of the most central services in the platform. It handles:

- Retrieving the list of available integrations from the database (displayed on the integration page)
- Installing integrations (when clicking "Connect")
- Updating existing installations
- Regenerating API keys for installations
- Uninstalling integrations
- Retrieving metadata from CRM systems (e.g., all fields that exist for a specific entity like "contact")
- Retrieving consent lists or consent bases from CRM systems

As [Erik Andersson] notes: > "Essentially every service in integration is to some extent like a CRUD service. We create something in the integration platform, we update something, we delete something, or list things in the integration platform."

### The Mappings Manager (MM)

The **Mappings Manager** handles field and consent mappings. It:

- Retrieves existing mappings between CRM attributes and Apsis One attributes
- Creates new mappings when users add them in the UI
- Manages mappings between CRM consents and Apsis One subscriptions

### Service Architecture: Microservices vs. Traditional Services

[Lukasz Grabowski] asked: "You mean service? It's not micro service, right?"

[Erik Andersson] clarified: Each service is deployed as a dedicated **ECS task** behind a single Application Load Balancer (ALB). While there's a fine line between "service" and "microservice," compared to other Apsis services, these are considered microservices. Each has a dedicated area of responsibility and runs as a separate ECS task continuously with minimal resource requirements (nothing in integration is resource-heavy except full syncs and real-time syncs).

### API Routing and the Holy Trinity

All API traffic is routed through a single ALB with path-based rules:

```
https://integration.apsis.cloud/mm/mappings/{accountId}/{sectionId}/{integrationId}
https://integration.staging.apsis.cloud/im/integrations/{accountId}/{sectionId}
```

**The "Holy Trinity"** of identifiers is fundamental to the platform:
- **Account ID**
- **Section ID**
- **Integration ID**

These three values together uniquely identify a single installation. [Erik Andersson] emphasizes: > "Anytime you get a support request, you need to ask SOC to provide those three things because it will speed up your debugging considerably."

### Service Abbreviations and Future Refactoring

[Erik Andersson] acknowledges the heavy use of abbreviations in the codebase:
- **IM** = Integration Manager
- **MM** = Mappings Manager
- **DSM** = Delta Sync Manager
- **DSW** = Delta Sync Worker
- **SLW** = Sluice Worker

He plans to rename these in the code repository (not necessarily the API endpoints, to avoid breaking customer configurations), but this is a lower-priority task.

---

## Field Mappings and Configuration

### Setting Up Field Mappings

When configuring an integration, users define **field mappings** between CRM system fields and Apsis One attributes:

```
CRM Field: "key_enterprise"  →  Apsis One Field: "CRM_ID"
CRM Field: "email_address"   →  Apsis One Field: "email"
CRM Field: "birthdate"       →  Apsis One Field: "description"
```

Each mapping stores:
- The CRM system field name
- The Apsis One field name
- The data type (string, integer, float, boolean; recently added: timestamp, datetime)

### Sync Conditions

**Sync conditions** are filters that regulate whether a profile should be added to Apsis One. For example:

```
If "active" field == true, then sync this contact
If "active" field == false, discard the update and don't consume profile quota
```

Sync conditions only support **equality comparisons** (field equals value).

### Consent Mappings

Users also map CRM **consents** to Apsis One **subscriptions** (formerly called "topics"):

```
CRM Consent: "marketing_email"  →  Apsis One Subscription: "email_marketing"
```

---

## Webhook Setup and Secret Management

### Registration During Installation

During the installation process, the Integration Manager:

1. Generates a random UUID as the **webhook secret**
2. Stores the secret in the Apsis One database
3. Makes a request to the CRM system to register a webhook
4. Provides the CRM system with the callback URL and secret, which they store

If webhook registration fails, the entire installation fails because real-time syncs are critical.

### Secret Verification Methods

Different CRM systems use different authentication mechanisms:

#### Microsoft Dynamics (OAuth2 via Apsis One API)
- **Only CRM that uses OAuth2 flow**
- Integration Manager generates a **Apsis One API key** (not a simple secret)
- Apsis One API credentials are sent to Dynamics
- Dynamics stores the credentials encrypted
- When sending webhook updates, Dynamics generates a **JWT access token** using the stored credentials
- The token is sent in the webhook request, and Apsis One verifies it (Apsis One handles the auth)

[Erik Andersson] explains the rationale: > "For Microsoft Dynamics, we paid a CRM consultant to build a plugin and because we own the code for this, we could decide we want to use a proper OAuth flow via Apsis One API."

#### Efficy Enterprise 12.0
- Shared secret approach
- CRM system includes the secret in the request header as an API key
- Apsis One retrieves the secret from the database and compares

#### Generic Connector (Efficy Enterprise 12.1)
- CRM system hashes the request body using the shared secret (SHA-256)
- CRM includes the hash in the request
- Apsis One generates the same hash and verifies the signature matches

#### Lime CRM and Others
- Simple shared secret comparison
- CRM includes the secret in the header

### Rationale for Different Approaches

[Erik Andersson] clarifies: > "The reason Dynamics is the only one with the sophisticated OAuth approach is that we control the plugin code. For most other CRMs, we can't enforce OAuth because their webhook systems don't support it. We had to resort to simpler methods."

### Cleanup on Uninstallation

When uninstalling, Apsis One requests the CRM system to **delete the webhook** to:
1. Prevent unnecessary requests
2. Comply with GDPR (stop sending sensitive personal data without an agreement)

---

## Two Synchronization Patterns

### Delta Syncs (Real-time Syncs)

**Delta syncs** are push-initiated: the CRM system pushes updates to Apsis One whenever a field changes.

**Flow:**
1. CRM detects a change (e.g., contact's first name updated)
2. CRM sends a webhook request to the **Delta Sync Manager (DSM)**
3. DSM verifies the webhook signature/secret
4. DSM puts the update on the **Sluice Queue**

**Entry point:** The Delta Sync Manager is the outer layer that receives all webhook requests from registered CRMs.

### Full Syncs (On-demand Syncs)

**Full syncs** are pull-initiated: Apsis One downloads all data from the CRM system.

**Use cases:**
1. **Initial setup** – After installing an integration and configuring field mappings, you want to download all existing data
2. **Adding new mappings** – If you add a new field mapping after initial setup, a full sync is needed to backfill the data (e.g., adding "birthday" field). Otherwise, you'd wait indefinitely for individual contacts to be updated via delta syncs.
3. **Adding new consent/subscription mappings** – Full sync downloads all consents from the CRM
4. **Incident recovery** – If the CRM had an outage, Apsis One had an incident, webhooks failed, or messages were dropped, a full sync re-syncs all current data to ensure consistency

### Resource Intensity

[Erik Andersson] notes: > "The full sync service is probably the most resource-intense service in all of integration. We've tested this up to 3 million contacts and corresponding consent entries. Typically customers have 50,000 to 70,000 contacts, but 200,000 can happen."

Delta syncs are lightweight because they handle one contact update at a time (or up to ~200 contacts batched together).

---

## Full Sync Architecture

### Three-Component Stack

The full sync consists of:

1. **Full Sync Manager** – Handles the UI interaction (create, list, cancel full syncs)
2. **Producer ECS Task** – Downloads all contacts from the CRM using pagination
3. **Consumer ECS Task** – Consumes messages from the queue and updates Apsis One profiles

### Producer Details

- Uses **5 threads** by default
- Downloads pages in parallel (threads 0-4 request pages 0, 1, 2, 3, 4; if all return data, threads request pages 5-9, etc.)
- Stops when a thread gets an empty page
- Puts each profile update (either attribute or consent type) into a temporary **SQS queue**

### Consumer Details

- Consumes messages from the SQS queue
- Extracts the field mapping configuration
- Makes requests to **Audience** service (HLS endpoint) for attribute updates
- Makes requests to **consent** endpoints for consent updates
- If Audience responds with 429 (rate limit), the message is put back in the queue for retry

### Lifecycle

When the producer finishes downloading and the consumer finishes processing all messages in the queue, the entire temporary stack is **decommissioned**:
- Consumer ECS task stops
- Producer ECS task stops
- Temporary SQS queue is deleted

Multiple full syncs can run simultaneously for different customers without affecting each other (each has its own temporary stack), though they do collectively stress Audience.

---

## Eventual Consistency and the Sluice Worker

### The Problem: Race Condition Between Full Sync and Real-time Updates

[Erik Andersson] describes a critical scenario:

> "You start the full sync. I download, like my name is Eric, on my contacts. But when the full sync has downloaded this message and the message is still in the queue, but there might be 200,000 updates before that message is handled, someone might change my name in the CRM system, so my name is now Frederick instead. That update will go to the Delta Sync Manager. So what can potentially happen now is that the full sync has downloaded my old name so that is ready to be processed. At the same time my new name is handled by the Delta Sync Manager, but because the Delta Sync Manager is usually a lot more Swift with the updates, my new update might be overwritten by my old name when the full sync has been processed."

### The Solution: Sluice Worker

To prevent this, the **Sluice Worker** implements a **gating mechanism**:

```
Delta Sync Manager → Sluice Queue → Sluice Worker → Delta Sync Worker Queue → Delta Sync Worker
```

**Sluice Worker logic:**
1. Consume messages from the Sluice Queue
2. Check: Is there an ongoing full sync for this installation?
3. If **NO full sync ongoing**: Push the message to the Delta Sync Worker Queue
4. If **YES full sync ongoing**: Put the message back in the queue with a visibility timeout (a few minutes), then retry later
5. This ensures full sync messages are processed first, then real-time updates afterward

### Special Case: Efficy Enterprise 12.0

Efficy Enterprise 12.0 has a unique challenge: it only sends the **changed fields** in webhook updates, not the entire contact record.

For example, if a contact's first name changes but the "active" field is not included in the update, Apsis One cannot verify the sync condition.

**Workaround:** For Efficy Enterprise 12.0 only, the Sluice Worker makes a **callback to the CRM** to fetch the complete contact data (including all sync condition fields) before deciding whether to process the update.

[Erik Andersson] explains why this callback is in the Sluice Worker and not the DSM: > "The Delta Sync Manager is synchronous—the CRM makes a request, we need to respond. If we made a callback for each of 5,000+ contacts that Efficy sent at once, that would be impossible. The Sluice Worker is asynchronous, so it doesn't matter if a message takes 5 minutes to handle. That's not a problem."

---

## Delta Sync Worker

The **Delta Sync Worker** is responsible for the final update in Apsis One:

1. Consume messages from the Delta Sync Worker Queue
2. Determine the message type:
   - **Attribute update**: Extract field mappings and make a request to Audience (HLS service) to update profile attributes
   - **Consent update**: Make a request to the consent endpoints
3. Handle errors and retries

---

## Shared Responsibility: Monitoring and Incident Response

[Lukasz Grabowski] asked: "Whose responsibility is it to be notified about webhook failures?"

[Erik Andersson] clarified: > "It depends on where the problem is. If the CRM system has an incident or outage and we never receive a webhook, we have no way of knowing because no real-time sync updates doesn't constitute a bug. The CRM team is responsible for notifying us. But if we experience errors in the Delta Sync Manager or internal server errors when calling the CRM, that's our responsibility."

Historical note: Efficy CRM teams have historically lacked the monitoring sophistication of Apsis, and have sometimes not known about their own outages until Apsis notified them about 503 errors.

---

## Repository Structure and Deployment

### App Folder and Microservice Code

All microservice code is located in the `app/` folder:

```
app/
  integration_manager/      (will rename from "im")
  mappings_manager/         (will rename from "mm")
  delta_sync_manager/       (will rename from "dsm")
  delta_sync_worker/        (will rename from "dsw")
  sluice_worker/            (will rename from "slw")
  full_sync_manager/
  full_sync_process/        (producer/consumer orchestration)
  outbound_manager/
  outbound_worker/
  Dockerfile                (shared across all services)
```

### Single Dockerfile for All Services

Rather than maintaining one Dockerfile per service, the team uses a **shared Dockerfile** that:

1. Compiles Go code
2. Deploys with parameterized service name and image tag

This makes it easier to update the build process centrally.

### Image Storage and ECR

Docker images are stored in **AWS ECR** (Elastic Container Registry):
- Each service has its own repository
- Images are built for staging, beta, and production separately (not promoted across accounts)
- Each image is tagged when pushed

[Lukasz Grabowski] noted: "We don't plan to change this approach for now."

---

## AWS Infrastructure: Load Balancing and Networking

### Application Load Balancer (ALB)

One **ALB** fronts the entire integration platform with **13 routing rules**:

```
Path: /im/*           → Integration Manager ECS task
Path: /mm/*           → Mappings Manager ECS task
Path: /dsm/*          → Delta Sync Manager ECS task
Path: /dsw/*          → Delta Sync Worker ECS task
Path: /slw/*          → Sluice Worker ECS task
... (and others)
```

Traffic routing is handled via **CloudFormation**. [Lukasz Grabowski] observed: > "This setup is very logical and straightforward compared to, for example, the email service where I couldn't find such a clear structure."

### Service Terminology: Managers vs. Workers

**Managers:**
- Customer-facing services
- Triggered by UI interactions
- Handle CRUD operations on configuration
- Examples: Integration Manager, Mappings Manager, Full Sync Manager
- Previously implemented as Lambda (cold-started), now ECS tasks

**Workers:**
- Background services
- Consume from SQS queues
- Process messages asynchronously
- Examples: Delta Sync Worker, Sluice Worker, Full Sync Consumer
- Always running (unlike full sync tasks, which are temporary)

### Database: Aurora RDS (PostgreSQL)

- **Postgres** database
- Runs in a private VPC (not accessible from outside)
- Access methods:
  - Via SSH bastion host tunnel (used by the team)
  - Via AWS RDS Query Editor
  - Via load balancer
- Credentials and connection setup handled via **make files**

[Lukasz Grabowski] noted: "We use the same setup with bastion host and SSH in eventool and webscript survey, so we're familiar with this approach."

---

## Database Schema and Migration Strategy

### Key Tables

#### Installations
- Stores account ID, section ID, integration ID (the Holy Trinity)
- Stores the Apsis One API key for the installation
- Stores the Audience key space ID
- Stores metadata (created, updated timestamps)
- **Constraint:** No duplicate installations for the same account/section/integration

#### Squid Domains
- Account ID, section ID, integration ID
- Domain name where the CRM is hosted (e.g., "mycrmdomain.com")
- Used by the reverse proxy (Squid) to whitelist outgoing traffic

#### Mappings
- Account ID, section ID, integration ID
- Apsis One field ID (target field)
- CRM system field name (source field)
- Data type (string, integer, float, boolean, timestamp, datetime)

#### Consent Mappings
- Account ID, section ID, integration ID
- CRM consent identifier
- Apsis One subscription/topic identifier

#### Sync Conditions
- Account ID, section ID, integration ID
- CRM field name
- Field type
- Comparison value (currently only equality supported)

#### Connections & Credentials
- **Connections**: Generic connector endpoints and metadata
- **Credentials**: API keys and secrets for installations; references a connection

#### Legacy Tables
- Connector-specific metadata (e.g., dynamics_connection with Dynamics tenant ID)
- Mostly unused for new integrations (only used for generic connectors now)

### Schema Management and Migrations

**Current approach:**
- `schema_create.sql` – Defines all tables (run once when setting up a new database)
- `data_static.sql` – Populates static data (sync statuses, etc.)
- `delta.sql` – Stores pending database changes

**Workflow:**
1. New feature or bug fix requires a DB schema change → Add change to `delta.sql`
2. At release time, manually run `delta.sql` against the production database
3. After verification, clear the `delta.sql` file

**Limitation:** No automated migration framework. [Erik Andersson] acknowledges: > "I would have really loved to have an automated migration system with rollback support, but we've not had luck implementing that for Golang. This approach works, but it's not sophisticated."

Historical context: Django (Python) made this easy; Node.js likely has good solutions; Golang frameworks have been less accessible to the team.

### Local Development: Docker Compose

For local development and testing, the `docker-compose.yml` file:

1. Spins up a local Postgres database
2. Automatically runs `schema_create.sql` and `data_static.sql` on startup
3. Each unit test tears down and recreates the database (before and after hooks)

This eliminates manual database setup for developers.

---

## Reverse Proxy (Squid Proxy)

### Purpose

The Squid proxy is an additional security layer that:

1. Filters **outgoing traffic** based on destination domain
2. Prevents data exfiltration if malicious code is injected into a service

**Why needed:** AWS Security Groups can filter by IP address/CIDR, but not by domain. If a customer has malicious code injected, they could attempt to redirect profile data to an arbitrary domain. Squid ensures traffic only goes to whitelisted domains.

### How It Works

1. All outgoing traffic from integration services is routed through Squid
2. Squid checks: Does this destination domain exist in the whitelist for this account/section/integration?
3. If yes: Allow traffic
4. If no: Block traffic

### Whitelist Management

When a customer installs an integration at domain `mycrmdomain.com`, Apsis One:
1. Creates an entry in the `squid_domains` table
2. Squid looks up this table on every outbound request

### Future Deep Dive

[Erik Andersson] and [Lukasz Grabowski] agreed this warrants a **separate 30-60 minute session**, as understanding reverse proxy mechanics requires more time than available in this session.

---

## Architecture Diagram and Visibility

[Lukasz Grabowski] asked: "What is in front of the load balancer? Is there an API Gateway?"

[Erik Andersson] clarified: No API Gateway. A **domain** (e.g., `integration.apsis.cloud` for staging, `integration.apsis.one` for prod) points directly to the **ALB via DNS CNAME**.

```
integration.apsis.cloud (DNS)
           ↓
   Application Load Balancer
           ↓
   ALB Listener Rules (path-based routing)
           ↓
   ECS Tasks (IM, MM, DSM, etc.)
```

---

## Key Takeaways

1. **The "Holy Trinity"** (account ID, section ID, integration ID) uniquely identifies an installation. Always request these three values for debugging.

2. **Two sync patterns exist:**
   - **Delta syncs (real-time)**: CRM pushes changes to Apsis One via webhooks
   - **Full syncs (on-demand)**: Apsis One pulls all data from the CRM

3. **Eventual consistency is enforced** via the Sluice Worker, which gates real-time updates while a full sync is in progress, preventing older full sync data from overwriting newer real-time changes.

4. **Webhook secrets are generated and stored**, with verification methods varying by CRM:
   - Microsoft Dynamics: OAuth2 via Apsis One API (sophisticated)
   - Most others: Simple shared secret or hash verification (basic)

5. **The Integration Manager is the hub** for installation, uninstallation, metadata retrieval, and key regeneration.

6. **Services are organized as:**
   - **Managers**: Customer-facing, configuration-oriented, always running
   - **Workers**: Background, queue-driven, process messages asynchronously

7. **Database migrations are currently manual** (via `delta.sql`). This works but is not automated.

8. **Squid proxy provides an outgoing security layer**, whitelisting CRM domains to prevent data exfiltration.

9. **Repository structure is clear:** All services in `app/` folder with a shared Dockerfile, deployed as ECS tasks behind a single ALB.

10. **Full syncs are resource-intensive** and spawn temporary producer/consumer ECS tasks + SQS queue that are cleaned up afterward. Multiple full syncs can run in parallel without affecting each other.

---

## Unresolved Questions and Action Items

1. **Squid Proxy deep dive** – Scheduled as a separate 30-60 minute session next week to cover whitelist logic, configuration, and edge cases.

2. **Database abbreviation refactoring** – Lower priority; planned to rename folders in the repository (e.g., `im/` → `integration_manager/`) but endpoint abbreviations will remain to avoid breaking customer configurations.

3. **Automated database migrations** – Acknowledged as a limitation but not prioritized. Could be a future refactoring task.

4. **ECR image promotion strategy** – Currently building separate images per environment (staging, beta, prod) rather than promoting a single image. [Lukasz Grabowski] indicated the team doesn't plan to change this for now.