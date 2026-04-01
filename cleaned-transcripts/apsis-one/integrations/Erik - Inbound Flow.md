---
source_file: Erik - Inbound Flow.txt
domain: Apsis One Integrations
topics: [inbound flow, delta sync, full sync, real-time sync, webhook registration, authentication, service architecture, ECS deployment, SQS queues, sluice worker, database schema, squid proxy, field mappings, consent mappings, sync conditions]
speakers: ["Erik Andersson (outgoing engineer/domain expert)", "Lukasz Grabowski (incoming engineer)", "Michal Rosikiewicz (incoming engineer)", "Tomasz Kowalski (incoming engineer)"]
key_components: [Integration Manager (IM), Mappings Manager (MM), Delta Sync Manager (DSM), Delta Sync Worker (DSW), Sluice Worker (SLW), Full Sync Manager, Full Sync Process, Squid Proxy, Aurora RDS/Postgres, SQS, ECS, ECR, ALB, Apsis One, Audience, HLS service, One API]
session_type: knowledge-transfer
---

# Session Overview

This is the second KT session between Erik Andersson (outgoing) and the incoming engineering team (Lukasz, Michal, Tomasz). Building on the previous session covering connector types, this session dives deep into the **inbound flow** of the Apsis One Integrations platform ("Justin"). Erik walks through all services involved in getting CRM data into Apsis One, covering the Integration Manager, Mappings Manager, Delta Sync Manager, Sluice Worker, Delta Sync Worker, and Full Sync components. The session also covers deployment architecture (ECS, ALB, ECR), database schema, and closes with a brief introduction to the Squid Proxy (to be covered in a dedicated future session).

---

## Service Naming Conventions and Abbreviations

A known pain point: the integration platform services are referred to throughout the codebase by abbreviations. Erik has a task to rename these before handover, but only within the repository folder structure — not the API endpoints.

| Abbreviation | Full Name |
|---|---|
| IM | Integration Manager |
| MM | Mappings Manager |
| DSM | Delta Sync Manager |
| SLW | Sluice Worker |
| DSW | Delta Sync Worker |

> "I have a task on me to remove these abbreviations from the code structure so you will be able to see the full name of the services before we hand this over, otherwise it will be very annoying for you to work with."

**Regarding endpoint URLs:** The API path abbreviations (e.g., `/im/`, `/mm/`) are **not** being changed at this time because every customer installation currently points to those URLs. Changing them would require URL redirects (e.g., `/im` → `/integration-manager`) and is considered a future refactoring project, not a handover priority.

---

## The Holy Trinity: Account ID, Section ID, Integration ID

Every entity in the integration platform is scoped by three identifiers:

- **Account ID**
- **Section ID**
- **Integration ID**

These three together uniquely identify one installation and its associated mappings, sync conditions, credentials, etc.

> "Any time you get a support request, you need to ask SOC to provide those three things because it will speed up your debugging considerably."

This pattern appears in every API request URL and every database table.

---

## Integration Manager (IM)

The Integration Manager is one of the most central services in the platform. It handles:

- **Listing** all integrations for a given account/section (used when a user loads the integrations page)
- **Installation** of a new integration (triggered when a user clicks "Connect" on a connector)
- **Updates** to existing installations (e.g., regenerating a One API key)
- **Uninstallation** — including cleanup of webhooks and CRM-side data
- **Retrieving metadata from the CRM**, including:
  - Available fields for a given entity (e.g., "contact")
  - Available consent lists / consent bases

### URL Routing Example

All traffic is routed through a single **Application Load Balancer (ALB)**. The path structure is:

```
<justin-domain>/<service-abbreviation>/<account-id>/<section-id>/<integration-id>/...
```

Example: a request to retrieve all integrations routes to the Integration Manager service:
```
integration-stage.apsis.cloud/im/<account-id>/<section-id>/
```

---

## Mappings Manager (MM)

The Mappings Manager handles field mappings and consent mappings between CRM fields and Apsis One attributes.

- **Field mappings**: A mapping from a CRM field key (e.g., "enterprise field key") to an Apsis One attribute ID
- **Consent mappings**: A mapping from a CRM consent/topic to an Apsis One subscription
- Operations: create, list, update mappings

When a user visits the "Field Mappings" page in the UI, they are reading from and writing to the Mappings Manager.

---

## Sync Conditions

A **sync condition** acts as a filter to determine whether an incoming contact update should actually be synced to Apsis One.

**Example use case:** You only want to sync active profiles. Configure a condition: `active == true`. For each incoming contact, if the `active` field is not `true`, the update is discarded and the profile is not added to Apsis One (protecting the customer's audience profile quota).

> "You probably do not want to sync all your inactive profiles in the CRM because they are probably not updated or you don't want to send emails to outdated customers."

Only `equal-to` comparison is currently supported.

---

## Inbound Data Flow: Two Mechanisms

### 1. Real-Time Sync (Delta Sync) — CRM-initiated

The CRM **pushes** updates to Apsis in real time via webhooks.

**How it works:**
1. During installation, the Integration Manager **registers a webhook in the CRM system**, providing:
   - A callback URL pointing to the **Delta Sync Manager** endpoint in Justin
   - The Account ID, Section ID, and Integration ID (embedded in the URL so Justin knows which installation the update belongs to)
   - A **secret** for request authentication
2. When data changes in the CRM (e.g., a contact's first name changes), the CRM sends a POST to the DSM callback URL.
3. The DSM verifies the request signature/secret, then puts the message onto the **Sluice Queue** (SQS).
4. The **Sluice Worker** consumes from this queue, checks whether a Full Sync is currently running for this installation, and either:
   - Passes the message forward to the **Delta Sync Worker Queue** (no full sync running), or
   - Delays the message with a visibility timeout and retries later (full sync is running — see race condition section below)
5. The **Delta Sync Worker** consumes messages from its queue and performs the actual updates in Apsis One:
   - Attribute updates → calls the **HLS service**
   - Consent updates → calls the **consent2 endpoint**

**All three real-time components (DSM, Sluice Worker, Delta Sync Worker) are always running** as ECS tasks.

**Webhook failure behavior:**
- If webhook registration fails during installation, **the entire installation fails**. Real-time syncs are considered too critical to allow installation without them.
- On uninstallation, the Integration Manager **deletes the registered webhooks** from the CRM to prevent continued data transmission (both for efficiency and GDPR compliance).

### 2. Full Sync — Apsis-initiated (on-demand)

Apsis **pulls** all data from the CRM. This is a temporary, resource-intensive process spun up on demand.

**When to use a full sync:**
1. **Initial setup** — after configuring field mappings and consent mappings for the first time
2. **After adding new field/consent mappings** — the full sync downloads data for newly mapped fields; without it, those fields remain unpopulated until a coincidental real-time update occurs
3. **Incident recovery** — if the CRM had an outage, Apsis had a bug, or messages were dropped, a full sync re-aligns Apsis data to the current CRM state

**How it works:**
1. User triggers a full sync via the UI → request goes to the **Full Sync Manager**, which creates the sync and starts the temporary ECS stack
2. A **Producer ECS task** starts downloading all contacts from the CRM using **pagination with ~5 concurrent threads**. Each thread fetches pages 0, 1, 2, 3, 4... If any thread receives an empty response, that thread stops; once all threads are exhausted, downloading is complete.
3. Each downloaded record (attribute update or consent update) is placed onto a **temporary SQS queue** created specifically for this sync run.
4. A **Consumer ECS task** reads messages from the queue and makes the corresponding updates in Apsis One Audience (same HLS/consent2 calls as real-time sync, using the field mappings from the Mappings Manager).
5. When the producer finishes and the consumer drains the queue, **the entire temporary stack is decommissioned**: producer stopped, consumer stopped, SQS queue deleted.

The Full Sync Manager also supports listing existing syncs and cancelling in-progress syncs.

**Scale tested:** Up to ~3 million contacts + ~3 million consent entries. Typical customers: 50,000–200,000 contacts.

**Retry on failure:** If Apsis One Audience responds with a `429` (rate limit) or similar error, the message is returned to the queue for retry.

---

## Race Condition Handling: Full Sync vs. Real-Time Sync

This is described by Erik as "golden logic" and "eventual consistency" handling.

**The problem:** If a full sync is in progress and has already downloaded a contact's old name (e.g., "Erik") into the queue, but a real-time sync update arrives with the new name ("Frederick") — the Delta Sync Worker (faster) might write "Frederick" first, and then the full sync consumer overwrites it with the stale "Erik" value.

**The solution — the Sluice Worker:**
- All real-time sync updates pass through the Sluice Worker before reaching the Delta Sync Worker.
- The Sluice Worker checks: **is there an active full sync for this installation?**
  - **No** → pass the message forward immediately
  - **Yes** → apply a visibility timeout (a few minutes) to the message and retry the check later; once the full sync completes, the message proceeds

This ensures real-time updates are never overwritten by stale full sync data.

---

## FSC Enterprise 12.0 Exception in the Sluice Worker

⚠️ **Important caveat — FSC Enterprise 12.0 only:**

Unlike all other CRM systems, FSC Enterprise 12.0 webhook payloads **only include the fields that changed**, not the full contact record. This means the Sluice Worker cannot evaluate sync conditions (e.g., `active == true`) from the webhook payload alone.

**Workaround:** For FSC Enterprise 12.0 updates, the Sluice Worker makes a **callback request to the CRM** to fetch the full set of required fields before evaluating the sync condition.

**Why this is handled in the Sluice Worker and not the Delta Sync Manager:**
- The DSM operates **synchronously** — the CRM expects a timely HTTP response.
- FSC Enterprise 12.0 can send batches of up to **6 MB** (potentially thousands of contacts) in one request.
- Making a callback to the CRM for each contact in a synchronous handler would be unacceptably slow.
- The Sluice Worker is **asynchronous** — it can take as long as needed to process a message without impacting the HTTP response to the CRM.

> "For every other CRM system, either they send us all of the data on the contact... or they have logic where we can say 'when you send us the webhook updates, please include these fields.' FSC Enterprise 12.0 annoyingly only includes the field that has been changed."

---

## Webhook Authentication Mechanisms by CRM System

Authentication between the CRM and the Delta Sync Manager varies by connector type:

| CRM System | Auth Mechanism |
|---|---|
| **Microsoft Dynamics** | OAuth flow via **One API**. Dynamics stores the One API key+secret provided during installation, generates an access token, and uses it when sending updates to Justin. This is the **only** CRM using a proper OAuth flow. |
| **FSC Enterprise 12.0** | Simple **shared secret** included in the request header as an API key; Justin compares it to the stored secret. |
| **Generic Connector (FSC Enterprise 12.1+)** | The CRM hashes the request body using the shared secret (SHA-256), sends the hash as a signature; Justin independently computes the hash and verifies the signature matches. |
| **Lime CRM** | Simple API key comparison (Lime's webhook system does not support OAuth flows). |

**Why Dynamics is the only OAuth one:**
> "For Microsoft Dynamics, this is the first one we built. We paid CRM Consultana to build a plugin and because we own the code for this, we could decide we want to use a proper OAuth flow via One API. However, for most other CRMs we did not control these CRMs — we can't just start saying 'we will totally use One API for Lime CRM' because their webhook systems don't support an OAuth flow."

**Secret generation:** The shared secret is a randomly generated UUID. It is stored in Justin's database, and the same value is sent to the CRM during webhook registration. The CRM stores it and includes it in future requests; Justin retrieves it and compares on each incoming request.

---

## What Justin Provides to CRM Systems During Installation

During the installation process, the following are sent to and stored by the CRM system:

1. **Webhook callback URL** (pointing to the Delta Sync Manager, including account/section/integration IDs)
2. **Webhook secret** (randomly generated UUID, or One API credentials for Dynamics)
3. **One API key** — so the CRM can make custom calls to Apsis One (e.g., creating custom events to trigger MA flows)
4. **One API URL** for the correct environment (staging or production)
5. **Section ID** — so the CRM can use it in One API requests (e.g., to retrieve event definitions or attributes)

All of this is cleaned up (deleted from the CRM) on uninstallation.

---

## Deployment Architecture

### Overall Structure

- **One Application Load Balancer (ALB)** for all of Justin (13 routing rules)
- Each rule forwards based on path prefix to the correct **ECS task** (e.g., `/im/` → Integration Manager task, `/mm/` → Mappings Manager task)
- **No API Gateway** — just the ALB fronted by a DNS record pointing to it:
  - Staging: `integration-stage.apsis.cloud`
  - Production: `integration.apsis.one`
- Infrastructure managed via **CloudFormation**

### ECS Task Model

Every service in Justin is an **ECS task**:
- **Managers** (IM, MM, DSM, Full Sync Manager): customer-facing, handle HTTP requests, always running
- **Workers** (Delta Sync Worker, Sluice Worker): consume SQS queues, always running, never directly customer-facing
- **Full Sync tasks** (Producer + Consumer): temporary, spun up on demand per sync invocation, decommissioned when complete (in the "Syncs" ECS cluster)

> "Previously all the managers used to be Lambda services, which were cold-started whenever we received a request. But as the platform grew, you had Lambda functions running 24/7 and that is not cheap. We migrated every service to ECS tasks instead, which was considerably cheaper and also made it easier to run locally."

The Delta Sync Manager is **more resource-provisioned** than other services because it receives traffic from all integrations across all customers simultaneously.

### Docker Build

All services use a **single shared Dockerfile** located in the `app/` directory:

> "Previously we had one Dockerfile for each service which was really annoying to maintain. Every service is essentially 'compile the Go code and deploy it with a specific name,' so now every service uses the same Dockerfile and we just inject the service name and image tag."

**Current ECR setup:** Images are built and pushed separately per environment (staging → staging ECR, beta → beta ECR, prod → prod ECR). Felix had expressed interest in changing this to build once in staging and promote the image across environments, but this has not been implemented.

### Repository Structure

All microservices live under the `app/` folder:

```
app/
├── dsm/              # Delta Sync Manager
├── dsw/              # Delta Sync Worker
├── full-sync-manager/
├── full-sync-process/  # Starts producer, consumer, SQS queue
├── mm/               # Mappings Manager
├── im/               # Integration Manager
├── ...               # Other services (outbound covered separately)
├── Dockerfile        # Shared by all services
```

These folder names will be renamed to full names before handover (e.g., `dsm/` → `delta-sync-manager/`).

---

## Database (Aurora RDS / PostgreSQL)

The platform uses **Aurora RDS (PostgreSQL)**, residing in a private network (not publicly accessible). Access options:
- AWS Console query editor
- Bastion host + SSH tunnel (same pattern used by other Apsis teams — Makefile scripts exist for this)

### Key Tables

| Table | Contents |
|---|---|
| `installations` | One row per installation. Stores account ID, section ID, integration ID, One API key, Audience key space ID, install/update timestamps. Unique constraint: one installation per account+section combination. |
| `squid_domains` | Whitelisted CRM domains per installation (used by Squid Proxy). |
| `mappings` | Field mappings: account/section/integration ID, CRM field ID, Apsis One attribute ID, data type (string, integer, float, boolean, timestamp, datetime). |
| `consent_mappings` | Consent/subscription mappings: CRM consent ID → Apsis One subscription (topic) ID. |
| `sync_conditions` | Per-installation sync conditions: which CRM field to check, its type, and the value to compare against (equality only). |
| `connections` | Used for Generic Connector installations; stores CRM URL and credentials (API key). |
| `credentials` | Points to a connection; stores the API keys for an installation. |
| Connector-specific tables | e.g., Dynamics-specific table stores the Microsoft Dynamics tenant ID (not needed by other connectors). |

**Data types supported in mappings:** Originally only `string`, `integer`, `float`, `boolean`. `timestamp` and `datetime` were recently added.

### Schema Management

Three SQL files govern the database:

```
schema_create.sql   # Creates all tables from scratch
data_static.sql     # Inserts static/seed data (e.g., sync status enum values)
delta.sql           # Pending migration changes for the next release
```

**Migration process (manual and unsophisticated — acknowledged as a known gap):**
1. Accumulate schema changes in `delta.sql` during development
2. At release time, **manually run** `delta.sql` against the database
3. After release, **clear** `delta.sql`

> "We currently do not really have a very sophisticated way of handling database migrations. We've had problems setting this up for Golang. I would have really loved to have an automated migration system with rollback support, but this has not been prioritised because it kind of has worked even though it is not sophisticated."

**Local testing:** The `docker-compose.yml` file starts a local Postgres instance and automatically applies `schema_create.sql` + `data_static.sql` on boot. Each unit test tears down and recreates the database.

**`schema_create.sql` is used manually** when setting up a new AWS environment. In disaster recovery, a **snapshot restoration** would be used instead (which includes all data).

---

## Squid Proxy (Outbound Traffic Filtering)

⚠️ **This topic will be covered in a dedicated future session.** The below is a brief introduction only.

The **Squid Proxy** is a reverse proxy that filters **all outgoing traffic from Justin** based on destination domain.

**Why it exists:** Standard AWS security rules only support IP address/range filtering. CRM systems are identified by domain name, not static IP. The Squid Proxy maintains a whitelist of allowed domains (stored in the `squid_domains` database table per installation). When Justin makes an outgoing request to a CRM system, the proxy checks whether the destination domain is whitelisted for that installation.

**Use case it protects against:** If malicious code were somehow injected into a Justin service, it could not exfiltrate customer data to an arbitrary domain — the proxy would block any domain not explicitly whitelisted.

**When domains are added/removed:** During installation, the CRM domain is added to the whitelist. On uninstallation, it is removed.

> "This is an additional security layer we added before release. It could potentially be revisited — it was added out of caution, but every service already runs in a private network."

**Action item: Schedule a dedicated ~1 hour session on the Squid Proxy.**

---

## Monitoring and Incident Responsibility

There are no automated alarms for "no real-time sync traffic received in X days" (absence of messages is not inherently a problem — it could mean no CRM updates occurred).

**Shared responsibility model:**
- **CRM team's responsibility**: Monitor their own webhook delivery; notify Justin team if there is an outage on their side
- **Justin team's responsibility**: Monitor for errors in DSM (e.g., `500` responses when processing inbound webhooks, `503` / gateway timeouts when calling back to CRM systems)

**Communication channel:** There is one internal Slack channel per CRM system. Justin team has historically noticed CRM outages before the CRM teams themselves:

> "We've sometimes noticed that we are getting weird internal server errors when we call, or the CRM system responds with internal server error when we make requests, or we've received 503 errors when we try to contact them. We notify them in our internal channel — and they didn't know their system had an outage."

---

## Key Takeaways

1. **The "holy trinity" of account ID + section ID + integration ID** is the universal key for all data in the platform. Always obtain these three when debugging support cases.

2. **The inbound flow has two paths**: real-time (CRM pushes via webhook → DSM → Sluice Queue → Sluice Worker → Delta Sync Worker Queue → Delta Sync Worker → Audience) and full sync (on-demand pull, temporary ECS stack per sync).

3. **The Sluice Worker exists specifically to prevent race conditions** between full syncs and real-time updates — without it, a full sync can overwrite more recent real-time data.

4. **FSC Enterprise 12.0 is a special case** in the Sluice Worker: it requires a callback to the CRM to retrieve full contact data before sync conditions can be evaluated, because its webhook payloads only contain changed fields.

5. **Microsoft Dynamics is the only CRM using a proper OAuth flow** (via One API); all others use shared-secret or hash-signature approaches due to limitations of those CRMs' webhook systems.

6. **Database migrations are manual**: changes accumulate in `delta.sql`, are applied manually at release time, then the file is cleared. This is a known gap with no current automated solution.

7. **Abbreviations in folder names will be replaced** before handover (repo only; API endpoint paths will not change for now).

8. **The full sync has been tested at 3 million contacts** but typical customers have 50K–200K. It is the most resource-intensive component and runs in isolated temporary ECS stacks per invocation so it cannot destabilize other services (except potentially putting load on Apsis One Audience).

---

## Unresolved Questions and Action Items

- [ ] **Schedule dedicated Squid Proxy session** (~1 hour minimum; Lukasz to organize for next week)
- [ ] **Erik to rename service folders** in the repository from abbreviations to full names before handover
- [ ] **Database migration automation**: No current solution for Golang; could be explored as a future improvement (lower priority)
- [ ] **ECR image promotion strategy**: Felix had proposed building once in staging and promoting across environments — not implemented; left for the incoming team to decide
- [ ] **Outbound flow** will be covered in a subsequent KT session (referenced but not covered in this session)
- [ ] **Prem to cover MA (Marketing Automation) database migrations** — Erik mentioned MA handles this automatically