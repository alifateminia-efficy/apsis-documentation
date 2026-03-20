---
source_file: Erik - Inbound Flow.txt
domain: Apsis One - Integrations
topics: [Integration Manager, Mappings Manager, Delta Sync Flow, Full Sync Process, Webhook Configuration, Field Mappings, Sync Conditions, Real-time Syncs, Architecture & Deployment, Database Schema, Reverse Proxy, Authentication & Secrets]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Integration Manager (IM), Mappings Manager (MM), Delta Sync Manager (DSM), Delta Sync Worker (DSW), Sluice Worker (SLW), Full Sync Manager, Full Sync Process, ECS Tasks, Aurora RDS, Application Load Balancer, SQS Queues, Squid Proxy, Webhooks]
session_type: knowledge-transfer
---

## Session Overview

This session covers the inbound data flow architecture for the Apsis One Integrations platform. The discussion centers on how data flows from external CRM systems into Apsis One, including the core services involved (Integration Manager, Mappings Manager, Delta Sync components), the configuration process (field mappings, sync conditions, consent mappings), and the two primary synchronization mechanisms: real-time syncs via webhooks and on-demand full syncs. The conversation also covers deployment architecture, database schema, and infrastructure components including load balancing and the reverse proxy security layer.

---

## Core Integration Services and Their Responsibilities

### Integration Manager (IM)

The **Integration Manager** is one of the most central services in the integration platform. It handles:

- Retrieving which integrations exist for a given account/section
- Installation of new integrations (when a user clicks "Connect")
- Updating existing installations
- Regenerating API keys
- Uninstalling integrations
- Retrieving metadata from CRM systems (such as available fields for entities like contacts)
- Retrieving consent lists and consent bases from the CRM system

As [Erik Andersson] explains:

> Every service in integration is to some extent a CRUD service. We create something in the integration platform, we update something in the integration platform, we delete something in the integration platform or list things in the integration platform.

### Mappings Manager (MM)

The **Mappings Manager** handles all field mapping operations:

- Retrieving existing field mappings between CRM attributes and Apsis One attributes
- Creating new mappings when users configure them in the UI
- Managing consent-to-subscription mappings
- Storing and retrieving the associations between external system fields and Apsis One fields

### Request Routing and the "Holy Trinity"

Every request in integration follows a specific structure that leverages the core identifying information:

```
https://integration.[environment].apsis.cloud/[service]/v1/accounts/{accountId}/sections/{sectionId}/integrations/{integrationId}/[resource]
```

The **account ID, section ID, and integration ID** form what [Erik Andersson] calls the "holy trinity":

> Everything in integration is based on this holy trinity, like the account ID, the section ID and the integration ID as soon as you know these three things. You will always be able to pinpoint one exact installation or like a set of mappings.

[Lukasz Grabowski] notes the practical importance:

> When you get a support request like you need to ask SOC to provide those three things because it will speed up your debugging considerably.

---

## Field Mappings and Configuration

### What Are Field Mappings?

Field mappings are the connections between:
- A field from the external CRM system
- An attribute in Apsis One

When data is synced, the Integration platform uses these mappings to know which CRM field value should populate which Apsis One attribute.

### Supported Data Types

Integration currently supports four primary data types for field mappings:
- String
- Integer
- Float
- Boolean

Recently added support (mentioned as newer):
- Timestamp
- Date time

### Sync Conditions

Sync conditions act as filters to regulate which profiles should actually be synced to Apsis One. For example:

> If you set up a sync condition where we for each of the incoming contact data verify that if the value in this field, in this active attribute on the incoming contact data is true, then we will sync the contact to Apsis One. If it is not true, we will just discard that update and we won't fill up their quota of profiles in audience.

This prevents unnecessary syncing of outdated or inactive contacts.

### Consent Mappings

Similar to field mappings, consent mappings connect:
- A consent/topic in the external CRM
- A subscription in Apsis One

---

## Real-Time Synchronization (Delta Sync) Flow

### Webhook Registration and Secrets

During the integration installation process, the Integration Manager registers a webhook in the external CRM system. The CRM is instructed to notify Apsis whenever specific contact data changes.

[Erik Andersson] describes the process:

> When we install an integration, essentially any integration that we have in one way or another, we register a webhook in the CRM system where we say please notify us about any changes to the say contact data like whenever you change someone's first name, e-mail, mobile number, any like if you change any of the fields that you have set up a mapping for here, then we want that to be updated in Apsis One.

The webhook registration process:

1. During installation, the Integration Manager receives the CRM system URL and API key from the customer
2. A unique webhook secret (random UID) is generated and stored in the Apsis database
3. This secret is sent to the CRM system, which stores it in their database
4. When registering the webhook, the callback URL includes the account ID, section ID, and integration ID:

```
https://integration.[environment].apsis.cloud/dsm/v1/accounts/{accountId}/sections/{sectionId}/integrations/{integrationId}/contact-updates
```

### Authentication Methods by CRM Type

#### Microsoft Dynamics (OAuth 2.0 Flow)

Microsoft Dynamics is unique in using a proper OAuth 2.0 flow via the Apsis One API:

1. An Apsis One API key is generated during installation
2. This key is sent to Microsoft Dynamics, which stores it encrypted
3. When Dynamics needs to send a webhook update, they first generate an access token using the stored credentials
4. They follow the recommended storage time and regenerate tokens as needed
5. The token is sent with the webhook request to Apsis
6. The request routes through Apsis One API, which validates the token before forwarding to the Delta Sync Manager

[Erik Andersson] explains why Dynamics is different:

> The reason for this is that like for the Microsoft Dynamics one like this is one of those CRMS which it is the first one we built. So we paid the CRM consultant to build a plug in and because we own the code for this, we could decide that we want to use a proper OAuth flow for this via one API.

#### FCC Enterprise 12.0 (Simple API Key)

For FCC Enterprise 12.0:
- The CRM system is provided a shared secret via API key
- The secret is included in the request header
- Apsis verifies the secret matches what's stored in the database

#### FCC Enterprise 12.1 Generic Connector (HMAC-SHA256)

For the generic connector:
- The CRM generates a hash of the request body using the shared secret (SHA-256)
- Apsis generates the same hash on its side
- Both verify that the signatures match

#### Other Systems (Generic Connector)

> For the generic connector, like I said, like they will send us the request body, but they also generate a hash based on the request body with their secrets. We will try to generate this same uh signature on ours and when we receive their request and verify uh verify the hashes.

### The Three-Component Real-Time Sync Pipeline

The real-time sync flow is handled by three distinct services working in sequence:

#### 1. Delta Sync Manager (DSM)

- Receives incoming webhook requests from the CRM system
- Verifies the webhook secret/signature (method depends on CRM type)
- Converts the webhook payload to Apsis internal format
- Places the message onto the **Sluice Queue**
- Returns a response to the CRM immediately (synchronous operation)

#### 2. Sluice Worker (SLW)

- Consumes messages from the Sluice Queue asynchronously
- Checks: **Is there currently a full sync running for this installation?**
- **If NO full sync is running:** Forwards the message to the Delta Sync Worker Queue
- **If YES full sync is running:** Delays the message with a visibility timeout (a few minutes) and retries the check later
- Handles an important exception for FCC Enterprise 12.0

#### 3. Delta Sync Worker (DSW)

- Consumes messages from the Delta Sync Worker Queue
- For profile/attribute updates: Makes requests to the Apsis One HLS service to update profile attributes
- For consent updates: Makes requests to the Apsis One consent endpoint to update subscriptions
- Performs the actual data updates in Apsis One (asynchronous operation)

[Lukasz Grabowski] clarifies the flow:

> So when the change is done on the CRM, they call us this Delta Sync Manager URL or API, right? Then what we do with this request, we just put it to the queue or?

[Erik Andersson] confirms:

> We verify the secret in whatever way, if it is the general connector way or the static API key way or or whatever. We take the update and then put it on the sluice queue.

### Eventual Consistency and the Sluice Worker Exception

The Sluice Worker exists to solve a critical race condition when a full sync is running simultaneously with real-time updates:

**Problem Scenario:**
1. Full sync starts downloading all contacts from the CRM
2. Erik's contact data is downloaded to the queue (name: "Erik")
3. Meanwhile, Erik changes his name in the CRM to "Frederick"
4. The real-time update arrives at Delta Sync Manager
5. The real-time update is processed quickly and updates Apsis to "Frederick"
6. Later, the full sync message is processed, overwriting the name back to "Erik"

**Solution:**
The Sluice Worker checks if a full sync is active. If it is, it delays real-time updates until the full sync completes, ensuring the full sync doesn't overwrite newer changes.

### FCC Enterprise 12.0 Special Case

FCC Enterprise 12.0 presents an unusual challenge. Unlike other CRM systems:

> For every other CRM system, either they send us all of the data on the contact, for example Dynamics, or they have logic where we can say when you send us the webhook updates, please include these fields and that enables us to verify like all of the sync conditions. FSC Enterprise 12.0 annoyingly only includes the field that have been changed.

This means if only the first name changes, Apsis doesn't receive the "active" field needed to evaluate sync conditions.

**Solution:**
In the Sluice Worker, for FCC Enterprise 12.0 only:
1. When a real-time update arrives, the Sluice Worker makes a callback to the CRM system
2. Requests the full contact data (including all fields needed for sync condition evaluation)
3. Verifies sync conditions before allowing the message to proceed

[Erik Andersson] explains why this can't happen in the Delta Sync Manager:

> We cannot have that check here in the Delta Sync Manager because the Delta Sync Managers is synchronous like the CRM system makes a request to us. We need to make a response to them if it's succeeded or not. And enterprise can send us like 6 megabytes of data at the same time. So imagine having to make one call back for each and every one of those contacts that they included like 5000 contacts like that is not something you can handle in a synchronous call.

---

## Full Sync (Batch Synchronization)

### When to Use Full Sync

Full sync is used in three main scenarios:

1. **Initial Setup:** After installing an integration and configuring field mappings, download all existing data from the CRM into Apsis One

2. **Configuration Changes:** When adding new field mappings or consent mappings after initial setup
   
   > If you go and add more field mappings after your initial setup, then you will need to run another full sync so we can populate the field that you have mapped into APSIS. Otherwise you would need to sit and wait until every contact randomly had been updated by a real-time sync in the CRM.

3. **Data Recovery:** After incidents or data inconsistencies
   
   > If there has been an incident in whatever, for whatever reason, it might be that the CRM system has had an outage and their webhooks didn't work. So there might have been a lot of updates in the CRM, but they were never sent to Apsis... in that case the full sync is also very useful because then once again we will download all of the data from the current data from the CRM system and make sure that it the data in APSIS is as it should be.

### Full Sync Architecture: Three Components

The Full Sync process creates a temporary stack with three components:

#### 1. Full Sync Manager (FSM)

- Handles user requests to start/list/cancel full syncs
- Customer-facing service (users interact with it via the UI)
- Creates new full sync job entries
- Manages the lifecycle of full sync operations

#### 2. Producer ECS Task (Downloader)

- Runs temporarily during a full sync
- Makes paginated requests to the CRM to download all contacts
- Default concurrency: **5 threads**
- Each thread downloads one page; if all threads receive data, continues to next batch
- Stops when any thread receives an empty response

**Pagination Logic:**
```
Thread 1: downloads page 0
Thread 2: downloads page 1
Thread 3: downloads page 2
Thread 4: downloads page 3
Thread 5: downloads page 4

If all return data → request next batch (pages 5-9)
If any returns empty → stop pagination
```

- Puts all downloaded data onto a **temporary SQS queue**

#### 3. Consumer ECS Task (Updater)

- Runs concurrently with the producer
- Consumes messages from the SQS queue
- For each message:
  - Retrieves the field mappings configured for this installation
  - Makes requests to the Apsis One HLS service to update profile attributes
  - Makes requests to the Apsis One consent endpoint for consent updates
- Implements retry logic: if Apsis responds with a 429 (rate limit) or other error, the message is returned to the queue

### Full Sync Resource Requirements

> This is probably the most resource intense service in whole of integration and that is because like we download everything that can be downloaded essentially like we've tested this up to I think 3 million contacts and then the corresponding like 3 million consent entries. This never happens. Luckily your typical customer will have anything between like 50,000 to 70,000 maybe, but like 200,000 can also happen.

**Important Note on Data Downloads:**

The full sync only downloads the data corresponding to the field mappings that have been configured. If a field isn't mapped, its data isn't downloaded.

### Full Sync Cleanup

When the full sync completes:
1. Consumer detects an empty queue (no more messages)
2. Producer ECS task is stopped
3. Consumer ECS task is stopped
4. Temporary SQS queue is deleted
5. The entire temporary stack is decommissioned

This ensures the resource-intensive sync doesn't permanently consume resources.

---

## Webhook Management and Uninstallation

### CRM Data Storage During Installation

When an integration is installed, the external CRM system stores:

1. **The webhook configuration** itself (the callback URL pointing back to Apsis)
2. **The webhook secret** generated by Apsis
3. **The Apsis One API key** (for generic connectors, in case they want to make custom API calls)
4. **The Apsis One API URL** for the specific environment (staging or production)
5. **Installation metadata:** Section ID and other identifiers (in case the CRM wants to query Apsis One API for event definitions, attribute definitions, etc.)

### Uninstallation Cleanup

During uninstallation, the Integration Manager:

> We ask CRM to delete the webhooks so they don't continue sending things to us because like one, we don't want to have unnecessary request. Second, that would be a GDPR breach for them to actually continue sending us sensitive personal data, even though we don't have an agreement with them anymore.

---

## Shared Responsibility: Monitoring and Incident Response

[Lukasz Grabowski] asks about responsibility for webhook failures:

> How are we notified about this? Is this our responsibility or is this external system responsibility to be notified about it?

[Erik Andersson] explains the shared responsibility model:

> If there has been an incident or a bug in the CRM system and we have never been notified, like we have no way of knowing this because no real time sync updates does not constitute a bug or a problem. It just might be that there has been no updates in the CRM system, so we cannot be responsible like we we don't have any alarms or anything if we don't receive anything.
>
> The CRM team is responsible for like notifying us if there has been any problem or like if they fail to deliver it to us for some reason. Of course it is our responsibilities if we notice that we have errors in the Delta Sync Manager endpoint where we like fail to put the message further into Apsis or we receive a like internal server error when we try to make a call back to the CRM system.

**Historical Incident:**

> Unfortunately, historically, the CRM or the FCCRM people do not have as sophisticated processes as Apsis has had like for example we've sometimes noticed that like we are getting in like weird internal server errors when we call or rather the CRM system responds with internal server error when we make requests... They didn't know that their system had an outage.

---

## Deployment Architecture

### Microservices and ECS Tasks

All services in integration run as **ECS tasks** behind a single **Application Load Balancer (ALB)**:

[Erik Andersson] clarifies terminology:

> Compared to a lot of the apps services, yes, I think it would be considered a microservice. So each of these like services have their like dedicated area of responsibilities and each of them is a ECS task that is just like running forever, but with very minimal resources because nothing in integration is resource heavy.

Each service is a dedicated **ECS task** running continuously (except full sync tasks, which are temporary):
- Integration Manager → ECS task
- Mappings Manager → ECS task
- Delta Sync Manager → ECS task
- Delta Sync Worker → ECS task
- Sluice Worker → ECS task
- And others...

### Load Balancer Rules

The ALB has **13 routing rules**, one for each service. Requests are routed based on the path component:

```
Domain: integration.[environment].apsis.cloud/[service]/...
          ↓
        ALB
          ↓
    Route to specific ECS task based on [service]
```

For example:
- `/im/` → Integration Manager ECS task
- `/mm/` → Mappings Manager ECS task
- `/dsm/` → Delta Sync Manager ECS task

### Service Abbreviations (Technical Debt)

[Erik Andersson] notes:

> When we built integration, unfortunately we were very fond of abbreviating everything, so right now the services are called like I am for integration manager, MM for mappings manager. I have a task on me and or on us to remove these abbreviations from the code structure.

[Lukasz Grabowski] asks about the scope:

> You mentioned earlier that you are going to change this abbreviations. You mean here in in the repository folders or?

[Erik Andersson] confirms:

> Yes, here it will say like actually full syncs. Here it will say integration manager. Here it will say mappings manager, outbound manager, outbound worker, sluice worker.

However, changing the API endpoint paths would break existing customer configurations. [Lukasz Grabowski] suggests:

> It's very difficult and time consuming to do it. So I guess we won't doing this at all.

[Erik Andersson] proposes:

> A simple URL redirect. So like it redirects from slash IM to slash integration manager.

But notes that for development purposes, the repository cleanup is the priority.

### Docker Image Building

A single **Dockerfile** is used for all services, avoiding maintenance of multiple Dockerfiles:

> Every service is essentially like compile the Go code and deploy it with a specific name. So now every service is using the same Docker file. And we just inject it with like whatever parameters we name like we inject like what is the name of the service and what should the docker file or the image be tagged with?

### Docker Image Storage

Images are stored in **AWS ECR (Elastic Container Registry)**, with a unique repository for each service:

[Lukasz Grabowski] asks:

> Images storage. Is it root on AWS?

[Erik Andersson] clarifies:

> It's unique for each account. So we are building the image for staging, then we're building the image for beta, then we're building the image for prod.

### Database: Aurora RDS

The integration platform uses **Aurora RDS (PostgreSQL)**:

- Resides in a **private VPC** (not accessible from the internet)
- Access methods:
  - Via the load balancer from applications
  - Via AWS Query Editor
  - Via SSH tunnel (bastion host approach, similar to other Apsis services)

[Lukasz Grabowski] notes familiarity:

> We usually also use Aurora in event tool and in web script for survey... and we use bastion host and SSH so to get it from the local.

---

## Database Schema and Tables

### Core Tables

#### Installations
Stores integration installation records with:
- Account ID, Section ID, Integration ID (the "holy trinity")
- The generated Apsis One API key
- Audience key space ID
- Metadata: created/updated timestamps
- **Unique constraint:** Only one installation per account/section/integration combination

#### Field Mappings
Stores mappings between CRM fields and Apsis One attributes:
- Account ID, Section ID, Integration ID
- CRM field ID/name
- Target Apsis One attribute ID
- Data type (string, integer, float, boolean, timestamp, datetime)

#### Consent Mappings
Stores mappings between CRM consents and Apsis One subscriptions:
- Account ID, Section ID, Integration ID
- CRM consent/topic ID
- Target Apsis One subscription/topic ID

#### Sync Conditions
Stores filter conditions for determining which profiles to sync:
- Account ID, Section ID, Integration ID
- CRM field to evaluate
- Field type
- Condition value
- **Current limitation:** Only supports equality (==) comparison

#### Squid Domains
Stores the whitelisted domains for each installation:
- Account ID, Section ID, Integration ID
- Domain name (e.g., "mycrmdomain.com")

#### Connections
Used for generic connector installations:
- Stores connection metadata for new integrations
- Referenced by credentials entries

#### Credentials
Stores API keys and secrets:
- Pointers to connections
- Account/section/integration metadata

#### Sync Statuses (Static Reference Data)
Pre-populated statuses for full sync jobs:
- Syncing
- Cancelled
- Done

### Database Migration Strategy

[Erik Andersson] explains:

> We currently do not really have a very sophisticated way of handling database migrations. We've had problems setting this up for Golang. So essentially what we do is that anytime we're anytime we do a new feature or we do a bug fix which includes modifying the database, we add the required change here in the delta file.

**Process:**
1. New schema changes are added to a `delta.sql` file
2. During a release, changes are manually applied to the database
3. After verification, the `delta.sql` file is cleared
4. This is **not automated** and relies on manual execution

[Lukasz Grabowski] notes:

> That's you know how it is. For example in event tool we implemented this at the beginning, at the very beginning it was the first step when we designed it. So that was good decision because otherwise currently I don't think we would have chance to add this migration to the database.

### Schema Initialization

#### Schema Creation
The `schema_create.sql` file:
- Defines all tables and their structure
- Used when setting up a completely new database
- Automatically applied in Docker Compose for local development and testing

#### Static Data
The `data_static.sql` file:
- Populates reference data (e.g., sync statuses)
- Also used in Docker Compose and local setup

#### Local Development
Both files are referenced in the **Docker Compose configuration**, which automatically applies them during database initialization:

> When you start the local environment, the database will be automatically populated with this, but this does not apply if you create a new AVS account or anything like then you need to do it manually.

---

## Network Architecture and Security

### Application Load Balancer (ALB) Architecture

```
Domain (integration.[env].apsis.cloud)
        ↓
    ALB (13 rules)
        ↓
    Route based on path → Appropriate ECS task
```

### Traffic Routing

Requests follow this URL structure:

```
https://integration.[environment].apsis.cloud/[service]/v1/accounts/{accountId}/sections/{sectionId}/integrations/{integrationId}/[resource]
```

### Squid Reverse Proxy (Security Layer)

The integration platform adds a **reverse proxy (Squid)** as an additional security layer to filter outgoing traffic:

[Erik Andersson] explains the motivation:

> In AWS you can of course have the security rules, but the security rules can only be used for IP addresses or IP ranges when a customer installs and say hello, my CRM system exists on mycrmdomain.com We want to filter the traffic only to domains that customer have an installation for.
>
> If someone were to be able to like inject some bad code into our services, like they cannot start rerouting traffic to some obscure server somewhere. So like if they try to send customer data to like myevilsite.com that would not work because we have not added a white listing for that domain.

**How It Works:**
1. All outgoing traffic from Justin services is routed through Squid
2. Squid checks a whitelist maintained in the database
3. Only traffic to whitelisted domains is allowed
4. All other traffic is blocked

**Whitelist Management:**
- When an installation is created for a domain (e.g., "mycrmdomain.com"), that domain is added to the whitelist for that installation
- The lookup is based on: Account ID, Section ID, Integration ID, plus the destination domain

[Lukasz Grabowski] suggests:

> I think it it will be reasonable to have separate meeting half hour or 11 an hour maybe something like.

[Erik Andersson] agrees:

> 1/2 an hour will not be enough, I can tell you that.

---

## Managers vs. Workers Terminology

### Managers (Customer-Facing Services)

Managers are services that customers interact with through the UI:

- **Integration Manager**: Users click "Connect" to install integrations
- **Full Sync Manager**: Users start/stop/list full syncs

Characteristics:
- Respond to user requests
- Initiate actions
- Typically run continuously as ECS tasks

### Workers (Event-Driven Services)

Workers are services that consume messages from queues and act based on their contents:

- **Delta Sync Worker**: Processes real-time updates from the sluice queue
- **Sluice Worker**: Filters real-time updates based on full sync status
- **Full Sync Producer/Consumer**: Handles bulk data downloads and updates

Characteristics:
- Customers never interact with them directly
- Driven by messages on SQS queues
- Perform the actual work (updates, syncs, etc.)

**Historical Context:**

[Erik Andersson] explains:

> Previously, like all the managers, used to be a Lambda service, which of course like it was just cold started whenever we received a request. But like the bigger we grew with the platform, like you had essentially Lambda functions running 24/7. And that is not cheap. So we migrated every service to be a ECS task instead, and that was considerably easier and also made it easier to like run it locally if we would want to.

---

## Code Repository Structure

### Service Implementation Location

All microservices are implemented in the `/app` folder:

```
/app/
  ├── delta_sync_manager/       # DSM - receives webhook requests
  ├── delta_sync_worker/         # DSW - updates Apsis One
  ├── full_sync_manager/         # FSM - manages full sync jobs
  ├── full_sync_process/         # Full sync producer & consumer
  ├── mappings_manager/          # MM - manages field/consent mappings
  ├── integration_manager/       # IM - manages installations
  ├── [other services]/          # Outbound and other services
  └── Dockerfile                 # Single Dockerfile for all services
```

### Database Initialization Files

```
/database/
  ├── schema_create.sql          # Table definitions
  ├── data_static.sql            # Reference data (e.g., sync statuses)
  └── delta.sql                  # Pending schema changes (cleared after release)
```

### Docker Compose for Local Development

The `docker-compose.yml` file includes:
- All required services
- Database setup using `schema_create.sql` and `data_static.sql`
- Full local environment for development and testing

---

## Key Takeaways

1. **The "Holy Trinity"** (Account ID, Section ID, Integration ID) uniquely identifies an installation and is the foundation of the entire system. Always ask for these three values when debugging support issues.

2. **Real-time syncs require three cooperating services:**
   - Delta Sync Manager receives and validates webhooks
   - Sluice Worker ensures consistency with full syncs
   - Delta Sync Worker performs the actual Apsis updates

3. **The Sluice Worker is critical for data consistency** — it prevents real-time updates from overwriting full sync data by delaying updates when a full sync is in progress.

4. **FCC Enterprise 12.0 is an exception:** It requires callback requests in the Sluice Worker to fetch complete contact data, since it only sends changed fields in webhooks.

5. **Full sync is resource-intensive** and should be used for:
   - Initial data loads
   - Adding new field/consent mappings
   - Data recovery after incidents

6. **Webhook secrets are handled differently by CRM type:**
   - Microsoft Dynamics: OAuth 2.0 via Apsis One API (most secure)
   - FCC 12.0: Simple API key comparison
   - Generic connector: HMAC-SHA256 hash verification

7. **All services run as ECS tasks** behind a single ALB with 13 routing rules. Each service is stateless and resource-light (except full sync tasks).

8. **Database migrations are manual** (not automated) — changes are added to `delta.sql` and applied before each release.

9. **Squid proxy provides domain-level egress filtering** to prevent code injection from rerouting customer data to unauthorized destinations.

10. **Service abbreviations (IM, MM, DSM, DSW, SLW)** exist in the codebase and endpoints but are planned for renaming to improve maintainability.

---

## Unresolved Questions & Action Items

1. **Squid Proxy Deep Dive** — Scheduled for a separate 1+ hour session to cover reverse proxy architecture, whitelist management, and domain filtering logic

2. **Abbreviation Cleanup** — Planned as a refactoring project; scope decision needed on whether to include endpoint renaming (with URL redirects) or just code repository changes

3. **Database Migration Framework** — Identified as technical debt but deprioritized; Golang migration solutions should be revisited if the team can allocate resources

4. **API Gateway vs. ALB** — Decision has been made to use ALB only (no API Gateway); understand implications for rate limiting, authentication, and future enhancements