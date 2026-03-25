---
source_file: Erik - Connector API & CICD.txt
domain: Apsis One Integrations
topics: [Generic Connector API Specification, REST API Endpoints, Webhook Integration, Authentication & Security, CI/CD Pipeline, GitHub Actions, AWS Code Pipeline, Docker Deployment, ECS Tasks, Local Development & Deployment]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Generic Connector, Delta Sync Manager, Mappings Manager, Full Sync Manager, ECR Repository, ECS Tasks, GitHub Actions, AWS Code Pipeline, CloudFormation, Bastion Host, S3 Bucket (integration-files.apsis.one), Azure Environment]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector, Inbound Flow, Outbound Flow]
---

## Session Overview

This session covered two major areas of the Apsis One Integrations domain: (1) the **Generic Connector API Specification** — the standardized contract that external CRM systems must implement when integrating with Apsis One, including detailed endpoint groupings, authentication mechanisms, and webhook handling; and (2) the **CI/CD Pipeline Architecture** — how code is tested, built, and deployed through GitHub Actions and AWS Code Pipeline, including the practical workflow for deploying individual microservices to staging/beta/production environments. Erik provided both conceptual overview and hands-on walkthrough of deployment procedures.

---

## Generic Connector API Specification Overview

### What is the Generic Connector

The **generic connector** is Apsis One's unified approach to integrating with multiple CRM systems. Rather than building connector-specific logic for each CRM, Apsis One defines a single, standardized set of **predefined endpoints** that all external CRM systems must implement if they claim to support the generic connector.

[Erik Andersson]: > "We have one connector inside of Justin and in that connector we have a like a predefined set of endpoints which we expect the external system to have implemented. If they have the generic connector support, that means that we will call any CRM that has implemented connector, we will call them in the exact same way we expect the same data and then of course like what features they have that will differ from environment to environment."

### Key Advantage Over Legacy Connectors

The generic connector solved a critical problem with legacy connectors: **HTTP status code handling**. In legacy connectors (e.g., Efficy Enterprise 12.0), CRM systems would use the HTTP layer as a **transport mechanism only**, not to communicate business logic results. For example:

- A request for a non-existent record would still return HTTP 200
- The error was buried in a response property, not in the HTTP status code
- This made error handling extremely complex for Apsis One

[Erik Andersson]: > "For the native connectors it has been very, or the legacy connectors. It has been very, very hard to handle HTTP responses because apps is using the HTTP layer to handle like the results of the business logic... Efficy Enterprise 12.0 they strictly use the HTTP layer as a transport. So if we manage to send the request to them, they will respond with a 200. It doesn't matter that like the profile that we wanted or the content we want to retrieve might not exist. But the HTTP response is still at 200."

In the generic connector, all CRM systems must respond with **proper HTTP status codes** (404 for not found, etc.) and standardized data formats.

### Generic API Specification Location & Version Control

**File Location in Codebase:**
```
/lib/connectors/generic/assets/
```

Two key files exist:
1. **generic-api-spec.yaml** — Endpoints that Apsis One will call on the external CRM system; request/response formats and HTTP codes
2. **apsis-generic-webhook.yaml** — Request bodies that Apsis One *expects* to receive from the CRM system when webhooks are triggered

[Erik Andersson]: > "This one in the code base, this is kind of by necessity always up to date because whenever we have a framework which generates interfaces like in Golang based on the API specifications that we have set up."

The API specification is **always kept up to date** because Apsis One uses a **code-first approach**: developers define endpoints in the OpenAPI YAML spec first, then auto-generate Golang interfaces from it, then implement the business logic. This prevents specification drift.

### API Specification Distribution

The specification is shared with development partners and CRM teams via:

1. **Production S3 Bucket**: `integration-files.apsis.one` (public, no IP whitelisting)
   - Contains the latest released version of the generic API spec
   - Updated during release pipeline deployments

2. **Code Repository**: Always contains the latest development version (may include unreleased features)

3. **Manual Distribution**: If a feature is in active development and a partner wants early access, the spec is sent via email or Teams

**Caveat**: If you're developing a new feature in the generic API, it will be in the code repo before it reaches the S3 bucket. Coordination with partners is required to avoid confusion about what version they're working with.

---

## Generic Connector API Endpoint Structure

The specification is logically grouped by feature. Understanding these groups is essential for navigating what endpoints exist and when they're called.

### System Information Endpoint

**Purpose**: CRM systems declare their identity and capabilities.

**Key Data Returned:**
- System ID and instance version (metadata, used for debugging)
- **Supported Features** — A list of features the CRM supports (e.g., "can_sync_email", "can_sync_events")
- Deep linking URL format — If the CRM supports it, the URL pattern for building direct links to contacts
- Supported translations for email campaigns (limited use case; primarily used by Maxo)

[Erik Andersson]: > "The CRM can tell us like what features it supports. That's very important because that will regulate like do we display the can sync e-mail, can sync event tool, etc etc."

**Called Frequently**: This endpoint is called whenever you navigate to the integration page, add a new integration, or list integrations in any Apsis One tool to determine what sync options to show.

### Integration UI Category

These endpoints support the **Integration Configuration Page** where users map fields and set up consent subscriptions.

#### Schema Endpoint
Retrieves the **field schema** from the CRM (field names, types, logical names). Used when displaying the field mapping UI.

#### Consent/Subscription Mappings
Retrieves the list of **consent bases** or **subscriptions** available in the CRM system, so users can map them to Apsis One subscriptions on the integration page.

### Profile Lists Endpoint

**Conceptual Overview**: This is used to sync **email lists or static/dynamic contact groups** from the CRM into Apsis One.

[Erik Andersson]: > "A profile list in integration terms. It's essentially like a list of or contacts in the CRM system. So imagine that you have like a e-mail list called technical newsletters... or you might have one list like VIP customers..."

**How It Works in Apsis One**:
1. The CRM reports a list (e.g., "Technical Newsletters" with 5000 contacts)
2. Apsis One creates a **tag** in its system with that list name
3. Each contact in that list gets the tag applied
4. Users can then segment by tag when sending emails

**Static vs. Dynamic Lists**:
- **Static**: User-managed (contacts are manually added/removed)
- **Dynamic**: Criteria-based (like Apsis segments; contacts are added/removed automatically based on attributes)

The profile list endpoint returns:
```json
{
  "id": "list-id-123",
  "name": "Apsis Gold Users",
  "types": ["contact"],
  "dynamic": false
}
```

**Note**: There are two endpoints:
1. List all available lists (called from UI)
2. Retrieve members of a specific list (called asynchronously in background task)

### E-Commerce Endpoints (Deprecated)

These endpoints were designed to retrieve abandoned cart data from e-commerce systems (originally Magento/Adobe Commerce). [Erik Andersson notes this is being removed as no customers currently use it, and Apsis One now focuses on CRM systems only.]

### Core Sync Operations: Records, Consent, and Webhooks

#### Get Records Endpoint
Retrieves contacts/profiles from the CRM with pagination support.

**Features:**
- Paginated (critical for millions of records)
- Optionally filter by specific field IDs (only download fields that are mapped)
- Optionally retrieve data for specific contact IDs (used in webhook callbacks)
- Always includes X-API-Key authentication header

**Example Response**:
```json
{
  "contacts": [
    {
      "id": "contact-123",
      "email": "user@example.com",
      "firstName": "Eric"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 100
  }
}
```

[Erik Andersson]: > "This can be like easily 2 million records. You don't want to download like 20 attributes for 2 million contacts in one go like you will kill any service that you have. We can specify like which fields on the contacts that we want to download."

#### Consent Synchronization Endpoints

**Get Consent** — Retrieve all consent/subscription data for contacts:
- Returns consent base IDs and which contacts have opted in/out
- Paginated

**Patch Consent** — Update consent status in the CRM:
- **This is the only bidirectional sync** (consent can be updated in Apsis One and pushed back to CRM)
- When a user opts out from an email subscription in Apsis One, this endpoint is called to update the CRM
- Parameters include CRM contact ID and consent base ID

[Erik Andersson]: > "The consent is the only thing which is bidirectional. So we don't have. We have a patch end point for consent, but we don't have a patch end point here for the profile for the actual profile data."

**Why Only Consent Is Bidirectional**: Contact attributes (name, email, etc.) flow in one direction (CRM → Apsis One). Consent is special because it originates in Apsis One (user preferences) and must sync back to the CRM.

#### Webhook Registration & Management

**Register Webhooks Endpoint** (called at installation time):
```
POST /webhooks
{
  "event_type": "record_updated",
  "callback_url": "https://apsis-integration.example.com/webhooks/callback",
  "fields": ["firstName", "lastName"],
  "secret": "<generated-secret>"
}
```

**Authentication**: CRM systems must sign webhook requests using the secret. Apsis One verifies the signature using:
```
hash = HMAC-SHA256(request_body + secret)
```

**Update Webhooks Endpoint**: If field mappings are added/changed, webhooks can be updated without re-registering (e.g., "also notify us if birthday changes").

**Delete Webhooks Endpoint**: Removes webhooks during uninstallation (for information security — stop data flow when relationship ends).

### Webhook Inbound Category (How CRM Calls Apsis One)

All webhooks inbound to Apsis One are routed to the **Delta Sync Manager** service.

**Webhook Types**:

1. **Record Updates** (`POST /delta/records`)
   - Payload: `{ "id": "crm-id-123", "action": "created|updated|deleted", "attributes": {...} }`
   - No difference between created/updated in implementation (both trigger same logic)
   - Deleted records are handled separately

2. **Consent Changes** (`POST /delta/consent`)
   - Notifies Apsis One that consent status changed in the CRM

3. **Merge Notifications** (`POST /delta/merge`)
   - CRM alerts Apsis One that two contacts were identified as duplicates and merged
   - Parameters include entity type (contact/lead/etc.) and which IDs were merged

4. **Promotion Notifications** (`POST /delta/promotions`)
   - Related to lead generation flow (covered separately)
   - When a lead is promoted to a real customer, CRM notifies Apsis One

5. **Subscription/Attribute Metadata Sync** 
   - CRM can create subscriptions or attributes and notify Apsis One to create corresponding records
   - **Maxo-only feature** currently; no active users

**Authentication for Inbound Webhooks**:
- CRM must include signature in header (HMAC-SHA256 of request body + secret)
- Apsis One verifies before processing

### Campaign/Activity Synchronization

Every activity in Apsis One (email send, SMS, etc.) is synchronized as a **campaign** in the CRM.

[Erik Andersson]: > "Every type of event in the CRM system is grouped under a specific campaign and a campaign is essentially the CRM corresponding of these activity instance in APSIS. So if I create a e-mail send out like Eric's newsletter in APSIS, we will first create a campaign in the CR system called Eric's Newsletter. And then all of the events that happen for that activity will be like linked to this campaign inside the CRM system."

**Flow**:
1. User creates email campaign "Eric's Newsletter" in Apsis One
2. Apsis One calls CRM's campaign creation endpoint with campaign metadata
3. Apsis One then sends event batches (opens, clicks, bounces, etc.) linked to that campaign
4. In CRM's UI, user can view all Apsis One events under the campaign

### Profile List Member Retrieval (Asynchronous)

The endpoint to retrieve actual members of a profile list is in a **separate category** because it's:
- Called asynchronously (via ECS background task)
- Not invoked from the UI (which only lists available lists)
- Potentially long-running (thousands of members)

---

## Versioning Policy for API Specifications

**Adding Optional Properties**: No version bump required
- CRM systems will discard unknown optional properties
- [Erik Andersson]: > "If you add new optional properties in function on endpoints like the patch consent here like we added the source and the last updated retroactively, but we did not need to increase the version because the CRM systems will like if they get new optional properties, they will just discard it or disregard it"

**Removing Properties**: Version must be incremented
- Breaking change; CRM systems expecting those properties will fail

---

## Identifying Generic vs. Legacy Connectors

To determine if a connector uses the generic connector or a legacy/custom implementation, check the **installer file**:

### Generic Connector Implementation Pattern

In `/lib/connectors/<connector-name>/installer.go`:
```go
// Generic connector pattern - inherits everything from generic installer
func (i *Installer) GetSystemInfo(ctx context.Context) {
    // Calls inherited method from generic installer
}
```

The file references the **generic installer** and reuses all functions.

### Legacy Connector Implementation Pattern

In `/lib/connectors/<legacy-connector>/installer.go`:
```go
// Custom implementation - defines own functions
func (i *Installer) GetSystemInfo(ctx context.Context) {
    // Custom implementation specific to this CRM
}
```

Defines custom implementations for each function.

### Quick Reference - Connector Types

**Generic Connector Implementations**:
- Efficy Enterprise 12.1
- Efficy Corporate (E-deal)
- Microsoft Dynamics
- Tribe
- WebServe 2

**Legacy/Custom Implementations**:
- Efficy Enterprise 12.0 (custom installer)
- Intermay Loyalty (custom installer)
- Maxo (custom installer, now discontinued)

[Erik Andersson]: > "You will like learn this by heart after some time like which connector works in what way?"

---

## CI/CD Pipeline Architecture

### Overview: Two-Part Tech Stack

Apsis One Integrations uses **two separate CI/CD tools** for different stages:

1. **GitHub Actions** — PR checks, linting, code coverage, security scanning
2. **AWS Code Pipeline** — Production deployments (builds Docker images, pushes to ECR, deploys CloudFormation)

### GitHub Actions: PR Checks & Testing

**Trigger**: Every time a PR is created or a commit is pushed to the PR branch

**What Runs**:
1. **Unit Tests** — All unit tests for all services in the integration domain
2. **Code Coverage Analysis** — Highlights services below coverage threshold
3. **Local Infrastructure** — Spins up Docker containers:
   - PostgreSQL database
   - Redis
   - Kafka
   - All listening on localhost

[Erik Andersson]: > "We spin up like we do the docker compose up so inside the inside the GitHub action runner like we start a like local database, we start a local Redis, we start a local Kafka."

**Duration**: ~10-20 minutes

**Status**: Coverage failures don't block merge; they're warnings only

**Important**: This GitHub Actions deployment flow (`.github/workflows/deploy.yaml`) is **currently disabled**. Deployments use AWS Code Pipeline instead (see below).

### Historical Context: Why Not GitHub Actions for Deployment

Prior to Graviton architecture migration, builds were attempted in GitHub Actions. The issue:

**Problem**: When Apsis One migrated ECS tasks to **ARM 64 (Graviton) architecture**, GitHub Actions runners didn't have native ARM 64 support. Building required **emulating ARM 64**, which caused:
- Build time: 20 minutes → 3-4 hours
- Unacceptable for production deployments

**Solution**: Moved deployment pipeline to **AWS Code Pipeline**, which has native ARM 64 runners.

**Current Status**: GitHub Actions now supports ARM 64 runners natively, so theoretically we could migrate back, but the migration work hasn't been completed.

[Erik Andersson]: > "When we transitioned to using the what is it called the Graviton architecture for the ECS tasks, that meant that we had to build ARM ARM 64 images. And when this project was implemented, we had no runner inside of a GitHub that was running on the ARM 64 architecture."

[Lukasz Grabowski]: > "Is it something we can put on the list as improvement?"

[Erik Andersson]: > "Yeah, yeah, for sure... I don't think it will be that much work to do it because I mean like the gist of it, like this, these steps are still relevant. It's just that we have not, we have not enabled it... I would really estimate like 2 days, three days maximum to deprecate code pipeline and utilise the deploy action in GitHub action."

### AWS Code Pipeline: Deployment Flow

**Trigger**: Merge to `develop` (staging), `beta` (beta), or `master` (production)

**Pipeline Steps**:
1. Check out code from appropriate branch
2. Build Docker images for **all services** (Managers and Workers)
3. Push images to **ECR repositories** (separate repos for staging, beta, prod-eu, prod-apac)
4. Deploy CloudFormation stacks for each service
5. Upload Swagger/OpenAPI specs to docs server
6. Upload generic API spec to S3 bucket

**Environments Deployed To**:
- `develop` branch → Staging
- `beta` branch → Beta
- `master` branch → Prod EU & Prod APAC (simultaneous)

**Duration**: 30-40 minutes in worst case

[Erik Andersson]: > "The deployments take considerably longer because like we build the docker images... you can probably considerably speed this up if you modify modify the flow to instead promote images from like from staging to beta from beta to products Felix wanted us to do, but like that is that was a bigger project than we had time for at the time."

### Authentication & Credentials in CI/CD

#### Original Approach: AWS Access Keys

The pipeline originally used hardcoded AWS access key/secret pair stored in GitHub Secrets. **Problem**: Keys must be manually rotated and re-added to GitHub (extremely cumbersome).

#### Modern Approach: OIDC (OpenID Connect) with Role Assumption

Shravidya implemented a **keyless authentication** system:

**Mechanism**:
1. GitHub Actions runner has a unique webhook ID
2. AWS account has a whitelist of allowed webhook IDs
3. When the workflow runs, GitHub provides a short-lived token linked to the webhook ID
4. The token is exchanged for temporary AWS credentials via `sts:AssumeRole`
5. **No stored secrets needed**; tokens expire automatically

**Visible in Code**:
```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    role-to-assume: arn:aws:iam::ACCOUNT_ID:role/GitHubActionsRole
    aws-region: eu-west-1
```

**Current Implementation**: PR checks already use OIDC (no secrets referenced). Deployment pipeline still uses legacy approach but can be migrated.

#### GitHub Token for Private Packages

[Erik Andersson]: > "We need this because when we install or when you utilize integration, we also utilize a package from another repository, namely the e-mail validation package. And we need we need this GitHub token to be able to access that package when we build the service because it's a private repository."

A **GitHub token** is stored for a CI/CD user account created in the GitHub organization. This token is used to fetch the `email-validation` package (from a private repo) during builds.

**How to Provision**: 
- Create a CI/CD user in the GitHub organization
- Generate a GitHub token for that user
- Add it to the repository secrets as `PROVISIONED_GITHUB_TOKEN`

---

## Deploying Services Locally & to Staging

### Prerequisites

1. **AWS Credentials**: Set up AWS profiles in `~/.aws/credentials` with the naming convention:
   - `staging`
   - `beta`
   - `prod-eu`
   - `prod-apac`

2. **Bastion Host & DB Credentials**: Stored in LastPass folder `integration_shared`
   - Pem files for SSH access
   - Database connection strings
   - RDS endpoints for each environment

### Deployment Workflow: Practical Example

[Erik Andersson walked through a real-time example of deploying a small change to the Mappings Manager]

#### Step 1: Make Code Changes

```bash
git checkout -b test-feature
# Edit code...
# For example, change behavior in is_trial_account function
```

#### Step 2: Navigate to Service Directory

Each service is under `/apps/<service-name>/` with its own Makefile.

```bash
cd apps/mappings-manager
```

#### Step 3: Check Service Configuration

Service Makefiles inherit common deployment commands from `manager-funcs.mk`:

```makefile
# Located in apps/mappings-manager/Makefile
include ../manager-funcs.mk

# Makefile defines:
# - SERVICE_NAME = mappings-manager
# - DOCKER_PORT = 8080
# - STACK_NAME = integration-mappings-manager
# - CLUSTER_NAME = integration
```

#### Step 4: Deploy to Staging

```bash
AWS_PROFILE=staging make deploy
```

**What This Does**:
1. Extracts AWS configuration from `staging` profile in credentials
2. Deploys ECR repository
3. Builds Docker image for ARM 64 architecture
4. Pushes image to staging ECR
5. Deploys/updates CloudFormation stack
6. Configures IAM permissions for the service
7. Registers the service with the ECS cluster
8. Updates routing to the new task definition

**Duration**: 3-4 minutes for ECS registration & routing updates

#### Step 5: (Optional) Deploy Only CloudFormation

If you only changed CloudFormation configuration (e.g., memory allocation) and not code:

```bash
AWS_PROFILE=staging make deploy-cf
```

This skips Docker build/push and only updates the CloudFormation stack.

### Database Access from Local Development

To query databases, use the **Bastion Host** as a proxy:

```bash
# Terminal 1: Set up SSH tunnel (Shuttle)
shuttle -i /path/to/pem -J ec2-user@bastion.staging -L 5432:rds.staging:5432 &

# Terminal 2: Connect via psql
psql -h localhost -U postgres -d integration_db
```

Alternatively, use the **AWS RDS Query Editor** in the AWS Console (no local setup needed).

### Make Target Commands

Common make targets available in all service Makefiles:

```bash
make build          # Build Docker image locally
make push           # Push to ECR
make deploy         # Full deployment (build + push + CF + IAM)
make deploy-cf      # Deploy CloudFormation only
make deploy-iam     # Deploy IAM policies only
```

### Important Notes on AWS Profiles

[Erik Andersson]: > "When you run make deploy with AWS profile staging, our script will go into the AWS environments and then look at the configuration file for the corresponding AWS profile and then populate it with like these pam files, the DB host, Bastion host, etc."

**Profiles Must Match Script Expectations**: The scripts look for specific AWS profile names (`staging`, `beta`, `prod-eu`, `prod-apac`). If you rename them, the deployment will fail because the scripts cannot find the corresponding configuration.

### Deployment Topology

```
Local Code Push
     ↓
git push to branch
     ↓
(Optional) Deploy manually to staging via make deploy
     ↓
Code Pipeline (triggered on merge to develop/beta/master)
     ↓
Build + Push to all environments
     ↓
ECS task registration
     ↓
Routing updates (3-4 min)
```

---

## Service Architecture: Managers, Workers, and Async Features

All integration services are **ECS tasks** (containerized Golang services). They're organized into categories:

### Manager Services
Stateless services that handle synchronous requests and orchestration.

**Examples**:
- `integration-manager` — Main API and orchestration
- `mappings-manager` — Field mapping logic
- `delta-sync-manager` — Webhook inbound processing
- `full-sync-manager` — Bulk data sync orchestration

### Worker Services
Long-running processes that poll for work and process asynchronously.

### Asynchronous Features
Separate services for specific async workflows (e.g., lead promotion, consent sync).

### Shared Infrastructure

All services use the same:
- **Docker image build process** (Dockerfile in each service)
- **CloudFormation template** structure
- **IAM permission model**
- **Environment variables** and secrets injection
- **Monitoring & logging** (CloudWatch)

This is why deployment can be orchestrated from the root Makefile:

```bash
make deploy # Deploys all managers + workers
```

---

## Key Takeaways

1. **Generic Connector API Specification** is the contract between Apsis One and external CRM systems. It's always up-to-date in the code repo because it's code-generated.

2. **HTTP Status Codes Matter**: Unlike legacy connectors, the generic connector uses proper HTTP semantics (404 for not found, etc.), not application-level error codes.

3. **Consent is Bidirectional**: It's the only data that syncs both ways (CRM → Apsis and Apsis → CRM). Everything else flows one direction.

4. **Webhooks are Central**: Real-time updates from CRM to Apsis flow through webhook notifications, validated with HMAC-SHA256 signatures.

5. **CI/CD is Split**: PR checks use GitHub Actions; deployments use AWS Code Pipeline (due to ARM 64 architecture). Migration to unified GitHub Actions approach is feasible but not yet completed.

6. **Deployment is Service-Oriented**: Each service can be deployed independently via `make deploy` after navigating to its directory and setting the correct AWS profile.

7. **Credentials & Authentication**: Modern approach uses OIDC tokens instead of stored secrets. Bastion hosts required for database access.

8. **Avoid Profile Name Changes**: AWS profiles must match expected names (`staging`, `beta`, `prod-eu`, `prod-apac`) or deployment scripts fail.

---

## Unresolved Questions & Action Items

1. **GitHub Actions ARM 64 Migration** (Improvement)
   - Estimated effort: 2-3 days
   - Status: Technically feasible; tests already done; not yet implemented
   - Owner: Integration team
   - Benefit: Unified CI/CD stack (single tool instead of two)

2. **AWS Secrets Rotation for Code Pipeline** (Security)
   - Legacy access key/secret pair needs rotation procedure
   - Could be fully replaced with OIDC (already implemented in PR checks)
   - Status: Not yet applied to deployment pipeline

3. **Azure Environment Legacy Status** (Follow-up Session)
   - Old Apsis International AB Azure tenant hosts Dynamics Oauth credentials
   - Cannot be migrated without breaking all Dynamics customers (would require reinstall)
   - Session scheduled to cover Azure environment setup, subscriptions, and credentials
   - Owner: Erik Andersson (personally maintains environment)

4. **Hands-On Lab: Database Access**
   - Setup: Lukasz Grabowski and Michal Rosikiewicz to connect to staging database
   - Objective: Verify bastion host access and query database
   - Method: Use shuttle SSH tunnel or AWS RDS Query Editor

5. **Upcoming Sessions**
   - Squid Proxy (Erik needs slides)
   - Lead Creation (requires preparation)
   - Azure/Entra/Dynamics Test Environment (ad-hoc, can be scheduled next)
   - Remediation of Common Issues