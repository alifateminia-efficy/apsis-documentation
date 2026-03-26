---
source_file: Erik - Inbound Flow.txt
domain: Apsis One Integrations
topics: [Inbound Flow Architecture, Integration Manager, Mappings Manager, Real-time Syncs (Delta Syncs), Full Syncs, Webhook Configuration, Data Synchronization, Field Mappings, Sync Conditions, Service Architecture, AWS Infrastructure, Database Schema]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Integration Manager (IM), Mappings Manager (MM), Delta Sync Manager (DSM), Sluice Worker (SLW), Delta Sync Worker (DSW), Full Sync Manager, Full Sync Producer, Full Sync Consumer, SQS Queues, ECS Tasks, Application Load Balancer (ALB), Aurora RDS PostgreSQL, Squid Proxy Reverse Proxy, Docker, ECR]
session_type: knowledge-transfer
subdomains: ["Lead creation", "Different Types of Connectors"]
---

## Session Overview

This session covered the **inbound flow** of the Apsis One Integrations platform—the complete architecture and workflow for syncing data from external CRM systems into Apsis. Erik walked through the core microservices (Integration Manager, Mappings Manager, Delta Sync Manager, and workers), the two primary data ingestion paths (real-time delta syncs via webhooks and full syncs on-demand), webhook security mechanisms, the critical eventual-consistency problem solved by the Sluice Worker, AWS infrastructure (ALB, ECS, Aurora RDS), and the database schema. The session emphasized that account ID, section ID, and integration ID form the "holy trinity" that identifies any unique installation.

---

## Core Concept: What is "Inbound"?

**Inbound** means the ability to download data from a CRM system and add it to profiles in Apsis One according to field mappings you've configured. The complete loop involves:

1. CRM system pushes data to Apsis in real-time
2. Data is stored inside Apsis One profiles
3. Profiles are used to send emails to specific groups

[Erik Andersson]: > "The whole origin of integration was to be able to download data from the CRM and add it to profiles in apps according to like some specific field mapping that you've set up."

---

## The "Holy Trinity": Account ID, Section ID, Integration ID

Every service in the integration platform relies on three identifiers working together:

- **Account ID**: Which Apsis One customer
- **Section ID**: Which section within that account
- **Integration ID**: Which specific CRM integration

[Erik Andersson]: > "Everything in integration is based on this holy Trinity, like the account ID, the section ID and the integration ID as soon as you know these three things. You will always be able to pinpoint one exact installation or like a set of mappings anytime you get the support request like you need to ask SOC to provide those three things. Because it's it will speed up your debugging considerably."

When debugging or providing support, always request these three values.

---

## Microservices Overview

### Service vs Microservice Terminology

All services in the integration platform are technically **microservices**. Each has dedicated responsibility and runs as a separate ECS task, though they are minimal-resource services (apart from full syncs). They are fronted by a single Application Load Balancer (ALB).

[Lukasz Grabowski]: "You mean service? It's not micro service, right? It's servicing Justin and it's when it comes to deployment, is it 1 microservice hosted on ECS?"

[Erik Andersson]: "Yes, exactly... Compared to a lot of the apps services, yes, I think it would be considered a microservice."

### Integration Manager (IM)

**Responsibilities:**
- Retrieve which integrations exist for a given account/section
- Handle installation of new integrations (via "Connect" button)
- Handle updates to existing installations
- Regenerate API keys
- Handle uninstallation
- Retrieve metadata from the CRM system (e.g., field definitions for contact entity)
- Retrieve consent lists and consent bases from the CRM

The Integration Manager is one of the most central services and is customer-facing (accessed via the UI).

### Mappings Manager (MM)

**Responsibilities:**
- List existing field mappings between CRM attributes and Apsis One attributes
- List consent/subscription mappings
- Create new mappings when configured in the UI
- Store and retrieve mapping configurations

All mapping operations go through this service.

### Delta Sync Manager (DSM)

**Responsibilities:**
- Serve as the **outer entry point** for real-time webhook callbacks from CRM systems
- Receive and validate webhook signatures (authentication varies by CRM type)
- Accept the incoming contact/profile update
- Put the update onto the Sluice queue for further processing

This service is synchronous—the CRM system expects an immediate response. It handles high throughput but does minimal processing.

### Sluice Worker (SLW)

**Responsibilities:**
- Consume messages from the Sluice queue
- Check if there is an ongoing full sync for this installation
- If no full sync is running: pass the message through to the Delta Sync Worker queue
- If a full sync is running: add a visibility timeout delay and retry later
- **Special case for FSC Enterprise 12.0**: Call back to the CRM to fetch all required fields for sync condition validation (explained below)

This service solves the **eventual consistency problem** that arises when real-time updates and full syncs run concurrently.

### Delta Sync Worker (DSW)

**Responsibilities:**
- Consume messages from the Delta Sync Worker queue
- Perform the actual update in Apsis One:
  - For profile updates: call HHLS service with mapped attributes
  - For consent updates: call consent endpoints
- Handle retries (put message back on queue if Apsis returns 429 or error)

---

## The "Eventual Consistency" Problem and the Sluice Worker

A critical scenario occurs when a full sync and real-time updates overlap:

**Scenario:**
1. Full sync starts downloading all contacts (including Erik with name "Erik")
2. While the full sync message is queued, Erik's name is changed in the CRM to "Frederick"
3. This new change triggers a real-time delta sync update
4. If not handled correctly, the old name "Erik" from the full sync could overwrite the newer name "Frederick"

**Solution: The Sluice Worker**

The Sluice Worker ensures that during a full sync, any incoming real-time updates are delayed and retried later. Once the full sync completes, real-time updates proceed normally.

[Erik Andersson]: > "What can potentially happen now is that the full sync has downloaded my old name so that is ready to be processed. At the same time my new name is handled by the Delta Sync manager, but because the Delta Sync manager is usually a lot more like Swift with the updates. My new update might be overwritten by my old name when the full sync has been processed and we really don't want that."

The flow: **DSM → Sluice Queue → Sluice Worker → Delta Sync Worker Queue → Delta Sync Worker**

---

## Real-Time Syncs (Delta Syncs)

### What Are Delta Syncs?

Delta syncs occur when the CRM system sends updates to Apsis One in real-time. When a contact's field changes or a consent changes, the CRM pushes that specific update.

### Webhook Registration During Installation

During the installation process:

1. Customer provides the CRM URL and API key
2. Integration Manager generates a **webhook secret** (random UID)
3. Integration Manager stores this secret in its database
4. Integration Manager makes a request to the CRM system to register a webhook callback URL
5. Webhook URL includes the account ID, section ID, and integration ID as query parameters
6. Integration Manager also sends the secret to the CRM; the CRM stores it in their system

[Erik Andersson]: > "So during installation time like you provide us with the URL to your system and you provide us with an API key in this is for the system as part of the installation process like we do a million things like first we generate the one API key, we create a key space for the installation... we start making requests to the CRM system to prepare the CRM system with the required things of which one is we make a request to register a web hook."

If webhook registration fails, the entire installation fails—real-time syncs are too critical to skip.

### Webhook Authentication Mechanisms

The authentication method depends on the CRM type:

#### Microsoft Dynamics (OAuth2 via One API)

- Apsis generates a One API key and secret
- Sends both to Microsoft Dynamics
- When Dynamics sends a webhook callback, it first contacts One API to generate an access token using the key/secret
- Dynamics sends the webhook with the token in the Authorization header
- One API forwards the request to Delta Sync Manager
- This is the **only CRM that uses this sophisticated flow**

[Erik Andersson]: > "For Microsoft Dynamics, which is using a plug-in which CRM Consultana built, they are using a OAuth flow using the one API credentials exchange."

#### FSC Enterprise (Static API Key)

- Apsis generates a shared secret
- Sends to FSC; they store it and include it in the header as an API key
- DSM retrieves the secret from its database and compares

#### Generic Connector (HMAC-SHA256 Signature)

- Apsis generates a shared secret
- CRM hashes the request body using this secret (HMAC-SHA256)
- Includes the signature in the request
- DSM regenerates the hash on its side and verifies the signature matches

[Erik Andersson]: > "For the generic connector version of FCC Enterprise 12.1 there they implemented the generic connector endpoints, so there we could force them to use our like hashing of the request body and us US verifying it."

### Why Microsoft Dynamics is Different

Microsoft Dynamics is the only CRM where Apsis controls the code (a custom plugin). For others, Apsis couldn't mandate OAuth2 because their webhook systems don't support it. The generic connector was built to impose standardized authentication on new integrations.

### What CRMs Store After Installation

CRM systems store:

- **Webhook URL** and secret
- **One API key** (for CRMs that will create custom events or use other One API features)
- **One API environment URL** (staging vs. prod)
- **Section ID and other metadata** about the Apsis installation

All of this is deleted during uninstallation to maintain GDPR compliance—Apsis must not continue receiving sensitive personal data after the integration is removed.

---

## Full Syncs

### Purpose and Use Cases

Full syncs download **all** data from the CRM system, not just changed records. Three main use cases:

1. **Initial Setup**: After configuring field mappings, run a full sync to backfill all existing contacts
2. **Adding New Mappings**: If you add a new field mapping (e.g., birthday field) after initial setup, run a full sync to populate that field for all existing contacts. Real-time updates alone won't help because the CRM only sends changed fields.
3. **Disaster Recovery**: If webhooks failed due to CRM outage, network issue, or Apsis bug, run a full sync to re-sync the current state from the CRM

[Erik Andersson]: > "Second case is if you go in and change your field mapping. Say now like I add like the birthday field here... if you did not include the birthday field in your like initial setup, we will not have added the birthday data to the contact. So if you go and add more field mappings after your initial setup, then you will need to run another full sync."

### Architecture: Three Components

The full sync service consists of three temporary ECS tasks + one SQS queue:

#### 1. Full Sync Manager (FSM)

- Customer-facing service (accessed via UI)
- Creates new full syncs
- Lists existing full syncs
- Allows cancellation of running syncs

#### 2. Full Sync Producer (ECS Task)

- Requests pages of contacts from the CRM system
- Uses **5 threads** (default) to fetch pages in parallel
- Threads fetch pages 0, 1, 2, 3, 4; if all succeed, next round fetches pages 5, 6, 7, 8, 9
- Stops when a thread receives an empty list (no more pages)
- Puts each downloaded profile (attribute and consent updates) onto a temporary SQS queue
- Updates can be: **attribute updates** (field values) or **consent updates**

[Erik Andersson]: > "We have I think it is like 5 threads I think is the default and each thread downloads like page 01234 if any of these threads receive an empty list. Then we do not like try to receive to download any more pages, but if all of the threads receive something in the request, then we try to download the next set of pages, so like page 6789."

#### 3. Full Sync Consumer (ECS Task)

- Consumes messages from the temporary SQS queue
- Extracts field mappings from the Mappings Manager
- For **attribute updates**: calls the HHLS service to populate Apsis One attributes
- For **consent updates**: calls consent endpoints
- Retries on Apsis errors (429, internal server error) by re-queueing the message

#### Temporary Infrastructure

Once the producer has downloaded all updates and the consumer processes the last message:

- Consumer ECS task is stopped
- Producer ECS task is stopped
- Temporary SQS queue is deleted

This entire temporary stack runs without affecting other services. Multiple customers can run full syncs concurrently.

### Scale Testing

Full syncs have been tested with up to **3 million contacts** and 3 million corresponding consent entries. Typical customers have 50,000–200,000 contacts.

---

## Real-Time Flow: The Complete Pipeline

The complete flow for a real-time sync from CRM to Apsis One:

```
CRM System 
    ↓ (webhook callback)
Delta Sync Manager (DSM)
    ↓ (validate signature, convert format)
Sluice Queue
    ↓
Sluice Worker (SLW)
    ↓ (check if full sync running)
Delta Sync Worker Queue
    ↓
Delta Sync Worker (DSW)
    ↓ (call HHLS or consent endpoints)
Apsis One
```

[Lukasz Grabowski]: "So when the change is done on the CRM, they call us this Delta Sync Manager URL or API, right? Then what we do with this request, we just put it to the queue or?"

[Erik Andersson]: "No. We verify the the secret in whatever way, if it is the general connector way or the static API key way or or whatever. We take the update and then put it on the sluice queue."

---

## Special Case: FSC Enterprise 12.0 Incomplete Webhooks

### The Problem

FSC Enterprise 12.0 only includes **changed fields** in webhook updates, not the complete contact record. If a contact's first name changed, FSC only sends the first name—not the active status or other fields.

This breaks sync condition validation. For example, if the sync condition is "active == true", the Sluice Worker cannot verify this condition because the active field wasn't included in the webhook.

### The Solution: Callback in Sluice Worker

For FSC Enterprise 12.0 specifically, the Sluice Worker makes a **callback to the CRM** to fetch the complete contact record (all fields needed for sync condition validation). This is why the Sluice Worker must be asynchronous:

- If this check happened in DSM (synchronous), responding to a 6 MB bulk update of 5,000 contacts would require 5,000 callback requests—impossible in a synchronous call
- Sluice Worker is asynchronous; taking 5 minutes to process a message is fine

[Erik Andersson]: > "For that reason if in in the sluice worker if the update is for FC Enterprise 12.0, we make a call back to the CRM to download all the data we we need so we can actually verify like OK, this update which we received, should we actually proceed with it or not?"

### Why Only FSC 12.0?

- Microsoft Dynamics: sends all data in webhooks
- Generic Connector: Apsis can specify exactly which fields to include
- FSC Enterprise 12.1 (generic connector): same as generic
- FSC Enterprise 12.0 (legacy): limitation of their webhook system

---

## Sync Conditions (Filtering)

### What Are Sync Conditions?

Sync conditions are **filters** that determine whether an incoming contact should be synced to Apsis One. For example:

- Only sync if `active == true`
- Only sync if `status == 'customer'`

[Erik Andersson]: > "If you would want to, you could add these sync condition which is essentially like a filter which regulates like should this profile actually be added to Apsis one or not, for example. In most cases, you probably do not want to sync like all your inactive profiles in the CRM system because they are probably not updated or like you don't want to send emails to like outdated customers, so."

### Current Limitations

- Only **equality comparisons** are supported (`field == value`)
- No range queries, regex, or complex logic

---

## API Routing: Domain and Service Path

All API requests follow this pattern:

```
https://integration.apsis.cloud/[service-name]/v1/...
```

Examples:

```
GET https://integration.staging.apsis.cloud/im/accounts/{accountId}/sections/{sectionId}/integrations
→ Routes to Integration Manager ECS task

GET https://integration.staging.apsis.cloud/mm/accounts/{accountId}/sections/{sectionId}/integrations/{integrationId}/field-mappings
→ Routes to Mappings Manager ECS task
```

The ALB has 13 rules, one for each service path, routing traffic to the appropriate ECS task.

**Note on Abbreviations:** [Erik Andersson]: > "When we built integration, unfortunately we were very fond of abbreviating everything... Right now the services are called like I am for integration manager, MM for mappings manager... I have a task on me and or on us to remove these abbreviations from the Code structure."

The team plans to rename folders in the codebase from `im`, `mm`, `dsm` to full names for clarity. However, renaming API endpoints is lower priority (would require customer migration).

---

## AWS Infrastructure

### Load Balancer

- **Type**: Application Load Balancer (ALB)
- **Domain**: `integration.apsis.cloud` (prod) or `integration.staging.apsis.cloud` (staging)
- **Rules**: 13 rules routing by path to different ECS task groups
- Traffic flows: Domain → ALB → Service Path → ECS Task Target Group

### ECS Tasks

- **Managers**: All running 24/7 (low resource consumption)
  - Integration Manager, Mappings Manager, Delta Sync Manager
- **Workers**: All running 24/7 (low resource consumption)
  - Sluice Worker, Delta Sync Worker
- **Sync Tasks**: Temporary, spun up on demand
  - Full Sync Producer, Full Sync Consumer

Each service is one ECS task with minimal resources.

### Database

- **Type**: Aurora RDS (PostgreSQL)
- **Network**: Private VPC—not accessible from outside
- **Access Methods**:
  - Via load balancer from ECS tasks
  - Via SSM Session Manager inside VPC
  - Via bastion host with SSH port forwarding
  - Via AWS RDS Query Editor (direct in VPC)

### Reverse Proxy: Squid Proxy

[Erik Andersson]: > "What we do in integration is that we added a additional security layer where we we filter the outgoing traffic based on the destination domain."

All outgoing traffic from Apsis integration services flows through a **Squid reverse proxy** that maintains a **whitelist of allowed domains** per installation. This prevents injected code from exfiltrating data to unauthorized domains.

When a customer installs on `mycrmdomain.com`, a whitelist rule is added allowing traffic only to that domain. Any attempt to route data elsewhere is blocked.

**Note**: Squid proxy will be covered in a separate 1-hour session due to complexity.

---

## Database Schema

### Key Tables

#### `installations`

Stores one row per unique integration installation:

- `account_id`, `section_id`, `integration_id` (composite key uniqueness)
- `apsis_one_api_key` (generated during installation)
- `apsis_one_key_space_id` (the Apsis One audience key space ID)
- `installed_at`, `updated_at` (timestamps)

#### `field_mappings`

Maps CRM fields to Apsis One attributes:

- `account_id`, `section_id`, `integration_id` (identifies which installation)
- `apsis_attribute_id` (target attribute in Apsis One)
- `crm_field_key` (source field in CRM system)
- `field_type` (string, integer, float, boolean, timestamp, datetime)

#### `consent_mappings`

Maps CRM consent statuses to Apsis One subscriptions:

- `account_id`, `section_id`, `integration_id`
- `crm_consent_id` (CRM system's consent identifier)
- `apsis_topic_id` (now called "subscription" in Apsis One terminology)

#### `sync_conditions`

Stores filters for which contacts to sync:

- `account_id`, `section_id`, `integration_id`
- `crm_field_key` (field to check)
- `condition_type` (currently only "equal")
- `condition_value` (the value to match)

#### `squid_domains`

Stores whitelisted domains per installation:

- `account_id`, `section_id`, `integration_id`
- `domain` (e.g., `mycrmdomain.com`)

#### `connections` and `credentials` (Generic Connector)

- `connections`: Metadata about a generic connector connection
- `credentials`: Stores API keys and secrets for generic connector installations

#### Connector-Specific Tables

- `connection_dynamics`: Microsoft Dynamics tenant ID and other metadata
- `connection_lime`, `connection_fsc`, etc.: Legacy connectors with custom fields

### Database Initialization

#### Schema Creation

File: `database/schema_create.sql`

- Creates all tables from scratch
- Used when setting up a new AWS account or environment
- Applied manually in production

#### Static Data

File: `database/data_static.sql`

- Populates reference tables (e.g., sync status enum values)
- Examples: `status_syncing`, `status_done`, `status_cancelled`
- Applied once during initialization

#### Migrations (Delta)

File: `database/delta.sql`

- Contains pending schema changes
- Manually applied before each release
- Cleared after release

[Erik Andersson]: > "We currently do not really have a very sophisticated way of handling database migrations. We've had problems setting this up for Golang. So essentially what we do is that anytime we do a new feature or we do a bug fix which includes modifying the database, we add the required change here in the delta file. Whenever we do a release, we manually run these changes from the delta on the database and then when the release is done we empty the file."

**Limitation**: Unlike Django or Node frameworks with automated migration systems, Apsis integration uses a manual process. This is not a priority for refactoring.

#### Docker Compose for Local Development

File: `docker-compose.yml`

- Spins up PostgreSQL container with all services
- Automatically applies `schema_create.sql` and `data_static.sql` on boot
- Enables local unit testing without manual schema setup
- Each test tears down and rebuilds the database

---

## Repository Structure

### Application Code: `/app`

All microservices are in this directory:

```
/app
  /integration_manager/       (formerly: /im/)
  /mappings_manager/          (formerly: /mm/)
  /delta_sync_manager/        (formerly: /dsm/)
  /delta_sync_worker/         (formerly: /dsw/)
  /sluice_worker/             (formerly: /slw/)
  /full_sync_manager/         (formerly: /fsm/)
  /full_sync_process/         (spins up producer + consumer)
  /outbound_manager/          (handled in outbound session)
  /outbound_worker/           (handled in outbound session)
  /Dockerfile                 (single Dockerfile for all services)
```

### Docker Image Building

All services share **one Dockerfile**:

- Compiles Go code
- Accepts parameters for service name and image tag
- Simplifies maintenance vs. per-service Dockerfiles

[Erik Andersson]: > "Previously we had like one Docker file for each of the service which was really annoying to maintain. Then we realized that every service is essentially like compile the Go code and deploy it with a specific name. So now every service is using the same Docker file."

### ECR Image Storage

- Unique ECR repository per service
- Images tagged and stored in each AWS account (staging, beta, prod)
- Current approach: build separate image per account/environment
- Proposed (by Felix): build once in staging, promote to other accounts

---

## Shared Responsibility: Monitoring Webhook Failures

### CRM System Responsibility

The CRM team must monitor their own systems and notify Apsis if:

- Webhook delivery fails
- Their system experiences an outage
- They cannot reach Apsis endpoints

### Apsis Integration Responsibility

Apsis must monitor:

- Delta Sync Manager errors (500 errors when processing webhooks)
- Errors when calling back to the CRM system (500 errors, 429 throttling, 503 gateway timeouts)

[Erik Andersson]: > "If there has been an incident or a bug in the CRM system and we have never been notified, like we we have no way of knowing this because no real time sync updates does not constitute a bug or a problem. It just might be that there has been no updates in the CRM system, so we cannot be responsible."

**Historical Issue**: The CRM team did not always have sophisticated alerting. Apsis has discovered CRM outages (503 errors, internal server errors) that the CRM team didn't know about.

### No Automatic Alarms for Silence

- Apsis does **not** have automatic alarms if no webhook traffic is received
- This could be legitimate (no changes in CRM) or a failure (both could look the same)
- If you want this alarm, you must set it up separately; it's not a shared responsibility agreement

---

## Manager vs. Worker Services

### Managers

**Definition**: Services that customers interact with directly via the UI

**Characteristics:**
- Respond to HTTP requests from the frontend
- **Previously**: Lambda functions (cold-started per request)
- **Now**: ECS tasks (always running)
- Examples: Integration Manager, Mappings Manager, Full Sync Manager

### Workers

**Definition**: Services that process background jobs from SQS queues

**Characteristics:**
- Customers never interact with them directly
- Consume messages from SQS
- Run asynchronously
- Examples: Delta Sync Worker, Sluice Worker

**Why the change from Lambda to ECS:**

Running Lambda functions 24/7 for every webhook callback became expensive. Switching to always-on ECS tasks is more cost-efficient and enables local development.

---

## Real-Time Sync Efficiency Notes

### When to Use Real-Time Syncs

Real-time syncs are designed for **incremental updates** on individual contacts:

- One contact's email changes → one webhook callback
- Batch of up to 200 contacts updated simultaneously → one webhook with up to 200 items

### When to Use Full Syncs

Full syncs are designed for **large-scale backfill**:

- Initial setup with millions of contacts
- Adding new field mappings to all existing contacts
- Recovery from outages or bugs

[Erik Andersson]: > "If you want to have always the correct data in abscess one, then yes, all of these three always will work in in tandem... The real time sync flow is not designed for them to say hello here is like 200,000 contacts that were updated like it's not designed for that flow and there there you should use the full syncs."

---

## Key Takeaways

1. **The Holy Trinity** (`account_id`, `section_id`, `integration_id`) uniquely identifies any installation and mapping. Always request these when debugging.

2. **Real-Time Flow**: CRM webhooks → DSM (verify signature) → Sluice Queue → Sluice Worker (check full sync) → DSW Queue → DSW (apply in Apsis)

3. **Eventual Consistency**: The Sluice Worker prevents real-time updates from overwriting in-flight full sync data by delaying updates until the full sync completes.

4. **Webhook Security**: Each CRM type uses a different authentication method—only Microsoft Dynamics uses sophisticated OAuth2 via One API; others use static API keys or HMAC signatures.

5. **Webhook Registration**: During installation, the Integration Manager registers a webhook with the CRM system and provides a shared secret. On uninstallation, webhooks are deleted for GDPR compliance.

6. **Full Syncs Are Resource-Intensive**: Three-component architecture (Manager, Producer, Consumer) with parallel threaded downloads, temporary SQS queue, and auto-cleanup.

7. **FSC Enterprise 12.0 Is Exceptional**: Its incomplete webhooks require the Sluice Worker to call back to the CRM to fetch complete records for sync condition validation.

8. **Service Architecture**: 
   - Managers = customer-facing, always running ECS tasks
   - Workers = background ECS tasks, always running
   - Sync tasks = temporary ECS tasks spun up on demand
   - Single ALB with 13 path-based routing rules

9. **Database Migrations Are Manual**: No sophisticated ORM migration framework; changes go into `delta.sql` and are manually applied before each release.

10. **Reverse Proxy Security**: Squid proxy whitelists destination domains per installation to prevent exfiltration of customer data if code is compromised.

---

## Unresolved Questions & Action Items

- **Squid Proxy Deep Dive**: Schedule a separate 1-hour session to cover reverse proxy architecture, whitelist logic, and configuration management
- **Abbreviation Cleanup**: Rename service directories in codebase from `im`, `mm`, `dsm` etc. to full names (lower priority; API endpoint rename deferred)
- **Database Migrations Framework**: Consider adopting a Golang migration framework in future refactoring (not a current priority)
- **Endpoint Aliasing**: Consider adding URL redirects from old abbreviations to full names (e.g., `/im` → `/integration-manager`) for backward compatibility