---
source_file: Erik - Inbound Flow.txt
domain: Apsis One Integrations
topics: [inbound flow, real-time sync, delta sync, full sync, service architecture, ECS deployment, SQS queues, webhook registration, authentication/secrets, field mappings, sync conditions, sluice worker, eventual consistency, squid proxy, database schema, ALB routing]
speakers: [Erik Andersson (outgoing engineer/domain expert), Lukasz Grabowski (incoming engineer), Michal Rosikiewicz (incoming engineer), Tomasz Kowalski (incoming engineer)]
key_components: [Integration Manager (IM), Mappings Manager (MM), Delta Sync Manager (DSM), Delta Sync Worker (DSW), Sluice Worker (SLW), Full Sync Manager, Full Sync Process, Aurora RDS/Postgres, SQS, ECS, ECR, ALB, Squid Proxy, One API, Audience/HHLS service]
session_type: knowledge-transfer
subdomains: ["Different Types of Connectors", "Lead creation"]
---

# Apsis One Integrations — Inbound Flow

## Session Overview

This session is the second in a series of knowledge-transfer meetings from Erik Andersson to the incoming team (Lukasz, Michal, Tomasz). The previous session covered types of connectors; this session focuses on the **inbound flow** — how data moves from CRM systems into Apsis One. The session covers all services involved in the inbound pipeline (Integration Manager, Mappings Manager, Delta Sync Manager, Sluice Worker, Delta Sync Worker, Full Sync), webhook registration and authentication, the eventual consistency problem and how the Sluice Worker solves it, a notable FSC Enterprise 12.0 exception, and the AWS infrastructure and database structure underpinning the platform.

---

## What "Inbound" Means

The **inbound flow** is the core integration use case: downloading data from a CRM system and adding it to profiles in Apsis One according to configured field mappings. Two directions exist:

1. **Full sync**: Apsis initiates and downloads all data from the CRM (batch, on-demand).
2. **Delta sync / real-time sync**: The CRM pushes updates to Apsis in real time via webhooks.

The end goal is to have CRM contact data available as Apsis One profiles so they can be used for email campaigns and audience segmentation.

---

## Service Architecture Overview

### Naming Conventions and Abbreviations

> "Unfortunately we were very fond of abbreviating everything."

[Erik] has a task to rename abbreviations in the repository folder structure before handover. The abbreviations currently in use:

| Abbreviation | Full Name |
|---|---|
| `IM` | Integration Manager |
| `MM` | Mappings Manager |
| `DSM` | Delta Sync Manager |
| `SLW` | Sluice Worker |
| `DSW` | Delta Sync Worker |

The endpoint paths currently use these abbreviations (e.g., `/im/`, `/mm/`). The plan is to rename only the **repository folder names** for now. Renaming the API endpoints would risk breaking existing customer installations and would require URL redirects — this is considered low priority and left as a potential future refactoring task.

### Deployment Model

Every service in the integration platform ("Justin") is an **ECS task**:
- All services are fronted by a **single Application Load Balancer (ALB)**.
- The ALB has routing rules (13 rules in prod) that forward traffic to the correct ECS task based on the URL path.
- URL structure: `<justin-domain>/<service-abbreviation>/<account-id>/<section-id>/<integration-id>`
- Example domains: `integration-stage.apsis.cloud` (staging), `integration.apsis.one` (prod).
- There is **no API Gateway** — only the ALB.

All services except full sync tasks run **continuously** with minimal resources. Full sync tasks are **spun up on demand** as temporary stacks.

Previously, manager services were Lambda functions, but as usage grew they were running 24/7, making ECS more cost-effective and also easier to run locally.

### Docker Images and ECR

- Every service uses a **single shared Dockerfile**, parameterized by service name and image tag.
- Images are built separately per environment (staging, beta, prod) and stored in **ECR**, with one repository per service.
- [Erik note]: Felix wanted to build once in staging and promote the image — this has not been implemented. The incoming team is free to change this approach.

### Manager vs. Worker Distinction

| Type | Description |
|---|---|
| **Manager** | Customer-facing; responds to UI requests (CRUD operations, metadata retrieval). Examples: Integration Manager, Mappings Manager, Full Sync Manager, Delta Sync Manager. |
| **Worker** | Internal; consumes SQS queues and performs actions. Customers never interact with workers directly. Examples: Delta Sync Worker, Sluice Worker. |

---

## The Holy Trinity: Account ID + Section ID + Integration ID

> "Everything in integration is based on this holy Trinity — the account ID, the section ID, and the integration ID. As soon as you know these three things, you will always be able to pinpoint one exact installation."

These three identifiers appear on virtually every database table and every API request. When handling support tickets, always ask SOC to provide all three — it will speed up debugging considerably.

---

## Integration Manager (IM)

The Integration Manager is one of the most central services. Responsibilities:

- Listing existing integrations for an account/section
- **Installing** an integration (generating One API keys, creating key spaces, whitelisting CRM IDs, registering webhooks in the CRM)
- **Updating** installations (e.g., regenerating API keys)
- **Uninstalling** integrations (including cleaning up webhooks in the CRM and removing stored data — GDPR requirement)
- Retrieving **metadata** from the CRM: available fields for an entity (e.g., "contact"), available consent lists/consent bases

On **uninstallation**, the Integration Manager instructs the CRM to delete the registered webhooks. Allowing a CRM to continue sending personal data after the agreement ends would be a GDPR breach.

---

## Mappings Manager (MM)

Handles all field mapping configuration. Responsibilities:

- Listing existing mappings between CRM attributes and Apsis One attributes
- Creating new mappings
- Listing and creating consent/subscription mappings (CRM consent ↔ Apsis One subscription)

API path example: `<justin-domain>/mm/<account-id>/<section-id>/<integration-id>/mappings`

---

## Sync Conditions

Sync conditions act as a **filter** on incoming contact data. Example: only sync contacts where the `active` field equals `true`. If a contact does not meet the condition, the update is discarded and the customer's Apsis One profile quota is not consumed.

If the `active` field changes to `false` on an existing contact, the profile should be **deleted** from Apsis One.

---

## Webhook Registration and Authentication

### Registration During Installation

As part of the installation process, the Integration Manager:
1. Generates a One API key and creates a key space.
2. Makes requests to the CRM system to register a webhook, providing:
   - The **callback URL** pointing to the Delta Sync Manager (including account ID, section ID, integration ID).
   - A **shared secret** (a randomly generated UID) that is stored in Justin's database. The CRM stores this and includes it on every webhook call.

> "If we fail to register the webhook, the whole installation will fail. We won't allow you to install without this working because the real-time syncs are such a crucial part of the integration."

### Authentication Methods by Connector Type

| CRM / Connector | Auth Method |
|---|---|
| **Microsoft Dynamics** | OAuth flow via **One API** — Dynamics generates an access token using the One API key+secret we provided; the request is forwarded through One API to our DSM. This is the most sophisticated method. |
| **FSC Enterprise 12.0** (legacy) | Simple **static API key** comparison — we provide the secret, they include it in the header, we compare. |
| **Generic Connector (FSC Enterprise 12.1+)** | **HMAC/SHA-256 signature** — the CRM hashes the request body with the shared secret; we hash it on our side and verify the signatures match. |
| **Lime CRM and others** | Simple shared secret / API key comparison (webhook system does not support OAuth flows). |

> "Microsoft Dynamics is ironically the only one with the more sophisticated way of authenticating to us — because it was the first one we built and we paid CRM Consultant to build a plugin, so we could dictate the auth flow. For most other CRMs, we don't control their webhook systems."

---

## What the CRM System Stores After Installation

When an integration is installed, the CRM is given (and stores):
- The **webhook URL and secret**
- The **One API key** (so the CRM can make custom calls — e.g., creating custom events on profiles to trigger MA flows)
- The **One API URL** for the correct environment (staging or prod)
- The **section ID** in Apsis One

All of this is cleaned up on uninstallation.

---

## Full Sync

### When to Use a Full Sync

1. **Initial setup**: After configuring field mappings, run a full sync to download all existing CRM data into Apsis One.
2. **Adding new field or consent mappings**: Only data with configured mappings is downloaded. Adding a new mapping (e.g., birthday field) requires a full sync to back-fill that data — otherwise you'd have to wait until every contact is touched by a real-time update, which may never happen.
3. **Incident recovery**: CRM outage, network outage, or a bug in Justin that caused messages to be dropped — run a full sync to restore data consistency.

### Architecture of the Full Sync

The Full Sync consists of three components:

1. **Full Sync Manager**: Handles the API (create, list, cancel syncs). When a sync is triggered, it spins up the producer and consumer ECS tasks.

2. **Producer ECS Task**: Downloads all contact/consent data from the CRM using **pagination and parallel threads** (default: ~5 threads). Each thread fetches consecutive pages. If any thread receives an empty page, it stops; if all threads receive data, the next set of pages is requested. Every downloaded update (attribute update or consent update) is placed on a **temporary SQS queue**.

3. **Consumer ECS Task**: Consumes messages from the SQS queue and performs the corresponding updates in Apsis One:
   - Attribute updates → calls the **HHLS service** in Audience, applying the field mappings configured in the Mappings Manager.
   - Consent updates → calls the **consent endpoints** in Apsis One.

On completion (producer done, queue empty), the entire temporary stack is **decommissioned**: producer stopped, consumer stopped, SQS queue deleted. Each customer's full sync runs in an isolated stack so customers don't affect each other (except for load on Audience).

**Scale tested**: Up to ~3 million contacts + 3 million consent entries. Typical customers: 50,000–200,000 contacts.

---

## Real-Time (Delta) Sync Flow

### Components

```
CRM System
    ↓ HTTP POST (webhook)
Delta Sync Manager (DSM)
    ↓ verified + converted → SQS (sluice queue)
Sluice Worker (SLW)
    ↓ (if no full sync running) → SQS (delta sync worker queue)
Delta Sync Worker (DSW)
    ↓
Audience (HHLS / consent endpoints)
```

All three components run **continuously** as ECS tasks.

### Delta Sync Manager (DSM)

- Receives the inbound webhook HTTP request from the CRM synchronously.
- Verifies the signature/secret.
- Converts the update to Justin's internal format.
- Puts the message on the **sluice queue**.
- Must respond quickly — cannot do expensive back-calls to the CRM here (see FSC 12.0 exception below).

### Sluice Worker (SLW) — Eventual Consistency Guard

**Problem**: A full sync and a real-time sync can race. Example:
- Full sync downloads a contact with name "Erik".
- Before the full sync processes that message, someone renames the contact to "Fredrik" in the CRM — the real-time sync arrives at DSM.
- The real-time update (Fredrik) gets processed first.
- Then the full sync processes its queued message and overwrites with the old name (Erik).
- Result: stale data in Apsis One.

**Solution**: The Sluice Worker checks whether a full sync is currently running for the specific installation. 
- **No full sync running**: pass the message through immediately to the Delta Sync Worker queue.
- **Full sync running**: apply a visibility timeout delay (a few minutes) and retry later until the full sync is complete, then let it through.

### Delta Sync Worker (DSW)

- Consumes messages from the delta sync worker queue.
- Attribute updates → calls HHLS in Audience.
- Consent updates → calls consent endpoints.
- On Audience errors (e.g., HTTP 429): puts the message back in the queue for retry.

---

## FSC Enterprise 12.0 Exception in the Sluice Worker

> "This is the one annoying exception that exists."

**Problem**: FSC Enterprise 12.0 webhook payloads only include **the field(s) that changed**, not the full contact record. This means Justin cannot evaluate sync conditions (e.g., check if `active == true`) from the webhook payload alone.

**Why it can't be solved in DSM**: FSC 12.0 can batch up to **6 MB** of data in a single webhook call, potentially representing thousands of contacts. Making a synchronous call back to the CRM for each contact would be impossible within a synchronous HTTP request-response cycle.

**Solution**: In the **Sluice Worker** (which is asynchronous), for FSC Enterprise 12.0 updates, before deciding whether to pass the message through, the Sluice Worker makes a **callback to the CRM** to download the full contact record — fetching all the fields needed to evaluate sync conditions.

All other CRM systems either send the full contact record in the webhook or support configuration to specify which fields to include.

---

## Load Balancer and Routing

- Single **ALB** for the entire Justin platform.
- 13 listener rules in prod mapping URL path prefixes to ECS target groups.
- Example: path `/im/*` → Integration Manager ECS task; `/mm/*` → Mappings Manager ECS task.
- DNS: `integration-stage.apsis.cloud` → ALB (staging); `integration.apsis.one` → ALB (prod).
- Traffic routing is set up via **CloudFormation**.

---

## Database (Aurora RDS / PostgreSQL)

Hosted in a **private network** — not publicly accessible. Access via:
- AWS Console query editor
- Bastion host + SSH tunnel (Makefiles provided for this)

### Key Tables

| Table | Contents |
|---|---|
| `installations` | One row per installation. Stores account ID, section ID, integration ID, One API key, Audience key space ID, install/update timestamps. Unique constraint: one installation per account+section. |
| `squid_domains` | Whitelisted CRM domains per installation (used by Squid Proxy). |
| `mappings` | Field mappings: account/section/integration ID, CRM field ID, Apsis One attribute ID, data type (string, integer, float, boolean; timestamp/datetime added recently). |
| `consent_mappings` | CRM consent ↔ Apsis One subscription (topic) mappings. |
| `sync_conditions` | Per installation: which CRM field to evaluate, its type, and the value to compare against (only `equals` comparison currently supported). |
| `connections` | Generic connector installations — stores API credentials. |
| `credentials` | Points to a connection; stores API keys for a given installation. |
| Connector-specific tables | e.g., Dynamics-specific table storing Microsoft tenant ID. Lime CRM has its own table. Only the generic connector table is reused for new integrations. |

### Schema and Migration Management

- **`schema_create`**: Full DDL to create all tables from scratch. Run manually when provisioning a new AWS account/environment.
- **`data_static`**: Populates static reference data (e.g., full sync status values: `syncing`, `done`, `cancelled`).
- **`delta` file**: Ad-hoc migration file. When a release includes a DB change, the SQL is added here. On release day, it is **manually applied** to the database, then the file is **cleared**.

> "We currently do not have a very sophisticated way of handling database migrations. We've had problems setting up a migration framework for Golang."

⚠️ **Warning**: There is no automated migration system. The delta file approach is manual and error-prone. This is a known gap. Automated migration frameworks were attempted but not successfully implemented. This is a candidate for improvement if the incoming team wishes to tackle it.

The `schema_create` and `data_static` files **are** used in automated testing: the Docker Compose local environment spins up a Postgres container and applies them on boot. Unit tests tear down and recreate the database before/after each test run.

---

## Squid Proxy (Outbound Traffic Filtering)

⚠️ **Covered only briefly — a dedicated session is planned.**

The **Squid Proxy** is a reverse proxy that filters all **outgoing traffic** from Justin based on destination domain.

**Rationale**: AWS security groups only support IP/IP-range rules, not domain-based filtering. When a customer installs a CRM at `mycrmdomain.com`, Justin whitelists that domain in the `squid_domains` table. All outgoing HTTP traffic from Justin is routed through the Squid Proxy, which checks if the destination domain is in the whitelist for that installation. If not, the request is blocked.

This prevents potential code injection scenarios where traffic could be rerouted to malicious domains, ensuring customer data can only be sent to their registered CRM endpoint.

> "This is an additional security layer we added before release. We can cover it in a separate session."

**Action item**: Schedule a dedicated ~1-hour session on Squid Proxy.

---

## Repository Structure

The `app/` folder contains all microservices:

```
app/
├── dsm/           # Delta Sync Manager          (to be renamed: delta-sync-manager)
├── dsw/           # Delta Sync Worker            (to be renamed: delta-sync-worker)
├── full-sync-manager/
├── full-sync-process/  # Producer + consumer + SQS setup
├── mm/            # Mappings Manager             (to be renamed: mappings-manager)
├── im/            # Integration Manager          (to be renamed: integration-manager)
├── slw/           # Sluice Worker                (to be renamed: sluice-worker)
├── ...            # Outbound services (covered in next session)
└── Dockerfile     # Single shared Dockerfile for all services
```

Also present:
- `docker-compose` file for running the full platform locally (includes local Postgres that auto-applies `schema_create` and `data_static` on boot).
- `Makefile` with scripts for DB tunnel setup, connecting to the database, and other common operations.

---

## Key Takeaways

1. **The inbound flow has two paths**: full sync (Apsis-initiated, batch) and delta sync (CRM-initiated, real-time via webhooks). Both always run through the same eventual write to Audience.
2. **The Sluice Worker is essential for data consistency** — without it, full sync and real-time sync can race and produce stale data in Apsis One.
3. **The holy trinity** — account ID + section ID + integration ID — is the key to everything. Always collect these three from SOC for any support request.
4. **Authentication varies by connector**: only Microsoft Dynamics uses a proper OAuth flow (via One API); all others use simpler shared-secret or HMAC approaches due to limitations in the CRM vendors' webhook systems.
5. **FSC Enterprise 12.0 is a special case**: its partial webhook payloads require a synchronous back-call to the CRM inside the Sluice Worker to fetch the full contact record for sync condition evaluation.
6. **No automated DB migrations exist** — the delta file is applied manually on each release and then cleared. This is a known gap.
7. **All services are ECS tasks behind a single ALB** — managers respond to HTTP requests, workers consume SQS queues. Full sync tasks are temporary/on-demand; all others run continuously.
8. **Squid Proxy adds domain-based outbound traffic filtering** — a dedicated session is needed to fully understand it.

---

## Unresolved Questions / Action Items

- [ ] **Schedule dedicated Squid Proxy session** (~1 hour minimum, per Erik's estimate) — Lukasz to organize for next week.
- [ ] Erik to complete renaming of service abbreviations in repository folder names before handover.
- [ ] **Database migration system**: Current delta-file approach is manual and unsophisticated. Evaluate whether a proper migration framework for Go (or equivalent) should be introduced by the incoming team.
- [ ] Clarify whether Docker images should be built once in staging and promoted (as Felix wanted) vs. the current approach of building separately per environment.
- [ ] Outbound flow not yet covered — scheduled for a future session.