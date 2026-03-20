---
source_file: Erik - Connector API & CICD.txt
domain: Apsis One - Integrations
topics: [Generic Connector API Specification, Connector Architecture, API Endpoints by Feature Group, Webhook Integration, Authentication Mechanisms, CI/CD Pipeline, GitHub Actions, AWS Code Pipeline, Docker Image Deployment, Local Development Setup, AWS Profile Configuration]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Generic Connector, Delta Sync Manager, Mappings Manager, Full Sync Manager, ECS Tasks, ECR Repository, Cloud Formation, S3 Integration Files Bucket, AWS Code Pipeline, GitHub Actions]
session_type: knowledge-transfer
---

## Session Overview

This knowledge transfer session covered two major areas of the Apsis One Integrations domain: the **Generic Connector API specification** and the **CI/CD pipeline architecture**. Erik Andersson walked through the standardized API contract that external CRM systems must implement to integrate with the platform, organized into logical feature groups with detailed endpoint specifications. The session then shifted to deployment mechanisms, explaining how the system transitioned from GitHub Actions to AWS Code Pipeline for building ARM64 Docker images, and demonstrating the practical workflow for deploying individual microservices to staging and production environments.

---

## Generic Connector Architecture and Design Philosophy

### Core Principle: Unified API Contract

The generic connector operates on a fundamental principle: **all CRM systems that support the generic connector must implement a predefined set of endpoints and follow a standardized data format**. This unified approach solves a critical problem that plagued legacy connectors.

[Erik Andersson]: The key issue with native/legacy connectors was handling HTTP responses. Apps uses the HTTP layer to convey business logic results. For example, when retrieving a record that doesn't exist, we expect a 404. However, some CRM systems like Fcc Enterprise 12.0 treat HTTP as purely a transport layer—they always respond with 200 even when the requested content doesn't exist, and return error details in response body properties instead. The generic connector eliminates this ambiguity by requiring all CRM systems to return proper HTTP status codes.

### API-First Development Workflow

Rather than writing business logic first and then documenting it, the integration system uses an **API-first approach**:

1. Define endpoints in the OpenAPI specification
2. Generate Golang interfaces based on the specification
3. Implement business logic against those interfaces
4. Push specification changes through CI/CD

[Erik Andersson]: This ensures the API spec in the codebase is always up-to-date by necessity, since the code is generated from it. We don't have the traditional problem of API documentation drifting from implementation.

---

## Generic API Specification Structure and Location

### File Locations

The Generic API specifications are located in the codebase at:

```
lib/connectors/generic/assets/
```

Within this directory you will find two critical files:

1. **generic-api-spec.yaml** - Defines the endpoints that external CRM systems must implement and the expected request/response format
2. **justin-generic-webhook.yaml** - Defines the request bodies that Apsis expects CRM systems to send when notifying us of changes

[Erik Andersson]: The generic-api-spec is what we send to external systems and what we expect them to respond with. The webhook document is what we expect them to call us with.

### Distribution to Development Partners

The specification is shared with development partners via an **S3 bucket that is publicly exposed**:

```
https://integration-files.appsis.one/
```

This bucket stores OpenAPI specifications, demo videos, and other integration documentation. However, there is an important caveat: the bucket contains the **latest deployed version**, not necessarily the latest development version.

[Lukasz Grabowski]: There is a timing gap—if a new feature is added to the spec in the code repository, it may not yet appear in the publicly exposed S3 bucket until the next deployment. In cases where partners need the very latest specification for an in-progress feature, specification updates are sent manually via email or Teams.

### Version Management for Specification Changes

[Erik Andersson]: If you add new optional properties to endpoints, you do not need to increase the API version. CRM systems will discard properties they don't recognize. However, if you remove existing required properties, you must create a new version of the specification. This allows backward compatibility during transitions.

---

## Generic API Specification: Endpoint Groupings by Feature

The specification contains numerous endpoints organized into logical feature groups. Each group corresponds to a specific feature or workflow.

### System Information Endpoint

This endpoint is frequently called throughout the platform to retrieve metadata and supported features from the CRM system.

**Response includes:**
- CRM instance ID and version
- **Supported features list** - indicates which sync capabilities are available (email sync, event sync, etc.)
- Deep linking URL format - allows Apsis to generate direct links to contacts in the CRM
- Supported translations for email campaigns

[Erik Andersson]: The supported features array is critical because it controls what we display in the Apsis UI. We check this endpoint whenever users visit the integration page or list integrations from a CRM, to determine which features can be synced.

### Integration UI Endpoint Group

These endpoints support the integration configuration page where users map fields and set up subscriptions.

**Key endpoints:**

- **Schema endpoint** - Returns the fields available on contact/person records in the CRM, including field name, data type, and logical name
- **Consent/Subscription list endpoint** - Lists subscription types in the CRM that can be mapped to Apsis subscriptions for consent management
- **Profile lists endpoint** - Returns the lists/segments available in the CRM system

### Profile Lists: A Critical Workflow

Profile lists represent a key integration feature that required a workaround in Apsis.

[Erik Andersson]: When you sync contacts from a CRM to Apsis and want to send emails to specific groups, you need to send to an email list. But Apsis email tool didn't support sending to CRM lists directly. So we created profile lists as a mapping mechanism.

A profile list in the CRM (e.g., "Technical Newsletters" list with specific contacts) gets represented in Apsis as a tag applied to matching profiles. When syncing profile lists:

1. Each list from the CRM is retrieved via the profile lists endpoint
2. For each contact ID in that list, we add a tag in Apsis with the list name
3. Users can then target profiles by those tags when sending campaigns

Profile lists can be **static** (manually maintained) or **dynamic** (criteria-based, like segments in Apsis). The endpoint returns a boolean flag indicating list type.

### Records and Consent Management Endpoints

This group contains the most heavily used endpoints for core sync functionality.

**Get Records endpoints:**
- Retrieve contacts/records from CRM with pagination support (critical for syncing millions of records)
- Support field filtering—if mapping specifies only 5 fields, download only those instead of all 50+ available
- Support specific record retrieval by ID (used when processing webhook notifications)

**Consent endpoints:**
- Retrieve all consents/subscriptions from CRM
- Patch/update consent status (this is the **only bidirectional sync** in the connector—all other data flows from CRM to Apsis only)
- When a user opts out in Apsis, we call back to the CRM to update their consent status

[Erik Andersson]: Consent is unique because changes flow both directions. We sync consent from the CRM to Apsis initially, but when users manage subscriptions in Apsis, we need to push those changes back to the CRM.

### Webhook Registration and Management

Upon installation, the connector registers webhooks to be notified of changes:

**Webhook registration includes:**
- Callback URL where CRM will send notifications
- Fields of interest—CRM only sends updates if specified fields changed
- Secret for HMAC signature verification

**Webhook types:**
- Record update webhooks
- Consent change webhooks
- Delete webhook endpoints (used during uninstallation to prevent further notifications)
- Update webhook endpoints (to add/remove fields being monitored if mappings change)

### Campaign and Activity Synchronization

The campaign endpoint is where all marketing activity events are sent to the CRM.

[Erik Andersson]: Events from Apsis activities are grouped under a campaign in the CRM. When you create "Eric's Newsletter" email send in Apsis, we first create a campaign in the CRM with that name, then synchronize all events (sends, opens, clicks) against that campaign. CRM users can then see Apsis activity data within their CRM campaigns view.

### Promotions Endpoint Group (Lead Gathering)

These endpoints support a specific workflow where leads created in Apsis through forms are promoted to actual contacts in the CRM.

[Erik Andersson]: When a form in Apsis creates a lead and that lead is later converted to a customer in the CRM, the CRM notifies us and provides the real contact ID. We then update the Apsis profile to link it to the actual CRM ID instead of the temporary lead ID.

### E-Commerce Endpoints (Deprecated)

These endpoints were implemented for a Magento/Adobe Commerce integration to handle abandoned carts and customer purchase information. They are being removed as the system now supports only true CRM systems.

### Debugging Endpoints

Non-business-logic endpoints available for troubleshooting:

- Retrieve registered webhooks configuration
- Get current system configuration (API domain, environment details)
- Fetch webhook metadata and status

### Attribute Creation Endpoint

This endpoint (implemented for Maxo but no longer active) allowed CRM systems to create new attributes/fields and have them automatically created in Apsis and added to field mappings.

---

## Webhook Specifications and Request/Response Contracts

The `justin-generic-webhook.yaml` specification defines what Apsis expects from CRM webhook notifications. All webhook endpoints are handled by the **Delta Sync Manager** service.

### Webhook Endpoint Types

**Record Updates:**
```
POST /delta-sync/records
Body: { entity_type, crm_id, action: "created|updated|deleted", fields }
```

**Consent Changes:**
```
POST /delta-sync/consents
Body: { consent_base_id, crm_id, action }
```

**Promotions (Lead to Contact):**
```
POST /delta-sync/promotions
Body: { lead_id, contact_id, conversion_details }
```

**Merge Events (Duplicate Handling):**
```
POST /delta-sync/merge
Body: { entity_type, primary_id, secondary_id }
```

**Subscription Creation (Metadata Sync):**
```
POST /metadata-sync/subscriptions
Body: { subscription_name, subscription_id, config }
```

The exact callback URLs are generated at installation time and include account ID, section ID, and integration ID in the path.

### Webhook Authentication

There are two authentication mechanisms depending on connector type:

**Generic Connector Webhook Authentication:**
- Uses HMAC-SHA256 signature verification
- CRM sends: `X-Signature: HMAC-SHA256(body + secret)`
- Apsis verifies the signature using the secret registered during webhook creation

[Erik Andersson]: When we register a webhook, we provide a secret. The CRM must hash the request body concatenated with this secret and send the result in the signature header. We do the same calculation on our side and verify they match.

**Legacy Connector Authentication:**
- Uses simple API key/UID header
- Less sophisticated but simpler for legacy integrations

### Determining Connector Type

To determine if an integration uses the generic connector, check the installer file:

```go
// Generic connector integration inherits from generic installer
type MyConnectorInstaller struct {
    generic.GenericInstaller
}

// Legacy connector has custom implementation
type LegacyConnectorInstaller struct {
    // Custom install logic
}
```

[Erik Andersson]: The easiest way is to check the integration list I'll provide, but you'll learn by heart which connectors are generic vs. legacy after working with them. Currently, generic connectors include: Dynamics, EDL (FSC Corporate), Magento, FSC Enterprise 12.1, Tribe, and Webserum 2. Legacy connectors include FSC Enterprise and FSC Intermay Loyalty.

---

## CI/CD Pipeline Architecture

The integration platform uses a hybrid CI/CD approach with different tools for different stages.

### GitHub Actions Usage

**Currently Active in GitHub Actions:**

1. **PR Check workflow** - Runs on every PR and branch push
   - Executes all unit tests across services
   - Runs code coverage analysis (informational only, not blocking)
   - Spins up local Docker Compose environment with database, Redis, Kafka
   - Duration: ~10 minutes
   - Uses newly available ARM64 GitHub runners

2. **Sigrid/Code Quality publish** - Publishes code quality metrics
   - Currently uses Sigrid; monitoring for potential transition to SonarCloud

**Currently Disabled in GitHub Actions:**

The `deploy.yaml` workflow exists but is disabled. It would handle building Docker images and deploying to environments but is not currently used due to historical performance issues.

### AWS Code Pipeline (Currently Active for Deployments)

The deployment process uses AWS Code Pipeline instead of GitHub Actions, which was necessitated by architecture changes.

[Erik Andersson]: When we transitioned to Graviton (ARM64) architecture for ECS tasks, we needed to build ARM64 Docker images. GitHub Actions didn't have ARM64 runners at that time, so we had to emulate ARM64, which increased build time from 20 minutes to 3-4 hours. AWS Code Pipeline had native ARM64 runners, so we switched deployment to Code Pipeline while keeping PR checks in GitHub Actions.

**Code Pipeline workflow includes:**
- ECR repository deployment
- Docker image build for the service
- Push image to ECR repository
- Deploy CloudFormation stack
- Deploy IAM permissions/policies
- Upload Swagger/API documentation to docs-internal
- Upload generic API specification to S3 bucket

**Deployment targets (by branch):**
- `develop` branch → staging
- `beta` branch → beta environment  
- `master` branch → prod-EU and prod-APAC

**Current deployment time:** 30-40 minutes (includes building and pushing images for all three environments)

### Future Optimization: GitHub Actions Consolidation

[Erik Andersson & Lukasz Grabowski]: There is a plan to migrate deployment back to GitHub Actions now that ARM64 runners are available. This would consolidate the tech stack and eliminate the need to maintain two different CI/CD systems.

**Effort estimate:** 2-3 days maximum
- Most steps are already implemented in `deploy.yaml`
- Primary change needed: update references to deployment script names for Swagger uploads
- Benefits: single tech stack, simpler maintenance, unified workflow definition

---

## Authentication and Secrets Management

### AWS Access Key Management (Legacy Approach)

Historically, the system used static AWS access keys for CI/CD:

- Created a dedicated "pipeline" IAM user in AWS
- Generated long-lived access key and secret
- Stored in GitHub Actions secrets
- Manual rotation required (cumbersome, often overdue)

[Erik Andersson]: This was problematic because rotating the key meant manually updating it in AWS, then manually propagating the new secret to GitHub, which was error-prone and frequently delayed.

### OIDC-Based Authentication (Current Best Practice)

Shravidya implemented a modern approach using **AWS OpenID Connect (OIDC)**:

**How it works:**
1. GitHub Action gets a short-lived OIDC token with the workflow's identity
2. This token is whitelisted in AWS to assume a specific IAM role
3. The role has necessary permissions (ECR push, CloudFormation deploy, etc.)
4. No static credentials needed—token generated per workflow run

**Implementation in workflow:**
```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::ACCOUNT:role/github-actions-role
    aws-region: eu-west-1
```

No AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY needed—AWS grants access based on the OIDC token.

**Advantages:**
- No secrets to rotate or store
- Fine-grained permission control via IAM roles
- Audit trail of which workflow runs accessed what
- Automatic token expiration

### GitHub Token for Package Access

The system uses a dedicated GitHub CICD user account with a personal access token for accessing private packages (email validation package).

[Erik Andersson]: When we build the integration service, we pull in a private email validation package from another repository. We need a GitHub token to authenticate to that private repo. This token is created for the CICD user within the organization.

---

## Local Development and Manual Deployment

### Microservice Deployment Workflow

The integration platform uses a make-based deployment system for individual microservice management.

**Directory structure:**
```
integration/
├── apps/
│   ├── make.functions        # Shared make functions (build, deploy, etc.)
│   ├── mappings-manager/
│   │   ├── Makefile          # Inherits from make.functions
│   │   └── cloudformation.yaml
│   ├── delta-sync-manager/
│   ├── full-sync-manager/
│   └── [other services]/
└── Makefile                  # Root makefile iterates all services
```

### Prerequisites for Deployment

Before deploying to any environment, you must have AWS credentials configured locally:

```bash
# Setup AWS profile (e.g., for staging)
# Credentials stored in ~/.aws/credentials
[staging]
aws_access_key_id = XXXXX
aws_secret_access_key = XXXXX

# Or use role assumption if configured
```

**Important:** Profile names must match expected names because deployment scripts reference specific profile configurations:
- `staging` profile
- `beta` profile
- `production` profile

Misconfiguring profile names will cause deployment to fail when scripts try to load environment-specific config files (bastion host, database host, etc.).

### Step-by-Step Deployment Example

[Erik Andersson demonstrates live deployment of a change to mappings-manager]:

**Scenario:** Bug fix in mappings-manager, deploy to staging

1. Create feature branch:
   ```bash
   git checkout -b fix/trial-account-check
   ```

2. Make code changes (e.g., in mappings-manager service)

3. Navigate to service directory:
   ```bash
   cd apps/mappings-manager
   ```

4. Deploy with AWS profile:
   ```bash
   AWS_PROFILE=staging make deploy
   ```

   This executes in order:
   - Deploy ECR repository (if needed)
   - Build Docker image (using Dockerfile in service directory)
   - Push image to staging ECR repository
   - Apply CloudFormation template to update ECS task definition
   - Register new task in ECS cluster
   - Perform service update with rolling deployment

5. **Service startup time:** 3-4 minutes (ECS pulls image, starts container, re-registers with load balancer, drains old container)

### Deployment Options

**Full deployment** (code + infrastructure changes):
```bash
AWS_PROFILE=staging make deploy
```
- Rebuilds Docker image
- Pushes to ECR
- Updates CloudFormation
- Time: ~10 minutes + ECS startup

**Infrastructure-only deployment** (CloudFormation changes only):
```bash
AWS_PROFILE=staging make deploy-cf
```
Use this if you only changed CloudFormation (memory, environment variables, etc.) without changing code.

### Make Variables and Parameterization

Make files use variables for service-specific configuration:

```makefile
APP_NAME := mappings-manager
APP_PORT := 8001
STACK_NAME := integration-$(APP_NAME)
CLUSTER_NAME := integration-cluster
MEMORY := 512
```

These variables are:
- Passed as build arguments to the Docker build process
- Used in CloudFormation templates
- Substituted into deployment scripts

This approach eliminates code duplication across services—each service inherits shared logic and provides parameters.

### Database Access from Local Machine

Staging and production databases are not directly accessible from your local machine. Access requires routing through a bastion host.

**Method 1: SSH Port Forwarding with Shuttle**

Bastion host credentials are stored in LastPass:
```
Integration shared folder → bastion-staging.pem (or similar)
```

Use SSH shuttle to set up a local tunnel:
```bash
# Terminal 1: Create SSH tunnel via bastion
shuttle -L 5432:db-host.internal:5432 bastion-user@bastion-host

# Terminal 2: Connect to local port (forwarded to DB)
psql -h localhost -U postgres -d integration
```

**Method 2: AWS RDS Query Editor**

Alternatively, use the RDS Query Editor in the AWS Console:
- Navigate to RDS → Databases → Select environment's database
- Open Query Editor
- No SSH setup required

---

## Service Architecture and Make Functions

The root Makefile orchestrates deployments across all services:

```makefile
MANAGERS := mappings-manager delta-sync-manager full-sync-manager ...
WORKERS := [list of worker services]
ASYNC := [asynchronous feature services]

deploy: $(addprefix deploy-, $(MANAGERS) $(WORKERS) $(ASYNC))
```

When you run `make deploy` from root, it deploys **every** service in sequence.

[Erik Andersson]: This is why for development, you deploy the specific service directory instead. If you deployed from root, you'd rebuild and push all services, which would take much longer. The individual service Makefiles inherit from shared functions but apply them to just that service.

---

## Key Takeaways

1. **Generic Connector = Unified Standard:** The generic connector solves the legacy connector problem by enforcing a single API contract. All CRM systems must implement the same endpoints and HTTP semantics. This greatly simplifies integration development and maintenance.

2. **API-First Development Is Built-In:** The codebase generates Golang interfaces from the OpenAPI spec, ensuring the spec never drifts from implementation. Design the endpoint in YAML, generate interfaces, implement code—never the reverse.

3. **Spec Distribution Has Timing Gaps:** The public S3 bucket has the latest deployed spec, not dev-branch spec. If partners need in-progress features, specs are sent manually via email/Teams.

4. **Webhooks Are Bidirectional for Consent Only:** Nearly all data flows CRM→Apsis. Only consent/subscriptions flow both directions, which is why that's the only endpoint with a PATCH method.

5. **Hybrid CI/CD: GitHub for PR Checks, AWS Code Pipeline for Deployment:** This arose from the Graviton (ARM64) transition. GitHub Actions now has ARM64 runners, so consolidation back to GitHub Actions is planned (2-3 day effort).

6. **OIDC-Based Auth Eliminates Secret Rotation:** Modern workflows use AWS OIDC tokens instead of static access keys. No secrets to manage, automatic expiration, better audit trail.

7. **Deploy Individual Services, Not Root:** Always deploy from the service directory (`apps/mappings-manager`) with `make deploy`, not from root. Root deployment rebuilds everything, which is slow and unnecessary for single-service changes.

8. **AWS Profile Names Matter:** Deployment scripts expect specific profile names (`staging`, `beta`, `production`). Misconfiguring profile names breaks deployment because scripts can't find environment configuration files.

9. **ECS Service Startup Takes 3-4 Minutes:** After pushing an image, account for several minutes for ECS to pull the image, start the container, re-register with load balancer, and drain the old container. This is expected behavior, not a failure.

10. **Database Access Requires Bastion Host:** Staging and prod databases are not publicly accessible. Use SSH shuttle port forwarding or AWS RDS Query Editor. Credentials/host info are in LastPass.

---

## Unresolved Questions and Action Items

### Potential Action Items

1. **Migrate CI/CD from Code Pipeline to GitHub Actions**
   - Status: Not yet started, but feasible
   - Effort: 2-3 days
   - Prerequisite: Verify GitHub Actions ARM64 runners are stable for this workload
   - Note: Most logic already exists in `deploy.yaml`, just needs to be enabled

2. **Consolidate AWS Secrets Management**
   - Current approach: Code Pipeline uses manually-managed AWS credentials
   - Proposed: Migrate Code Pipeline to also use OIDC (if AWS Code Pipeline supports it)
   - Benefit: Eliminate static credentials entirely across CI/CD

3. **Optimize Deployment Times**
   - Current: 30-40 minutes per deployment (includes staging, beta, prod)
   - Proposed: Implement image promotion strategy (build once, promote to environments)
   - Note: Felix Staub requested this but was not completed due to time constraints

### Open Questions

1. **Health of Sigrid Integration:** Erik mentioned Sigrid might be discontinued. Need to clarify and plan migration path if necessary.

2. **Azure Test Environment Maintenance:** The Azure Oauth environment is owned by Apsis International AB (old domain) and is a pain point for IT. Plan a dedicated session to document the setup, access controls, and contingency if the environment becomes unavailable.

---

## Related Topics for Future Sessions

- **Squid Proxy Configuration** (requires Erik preparation)
- **Lead Creation Workflow** (requires slides/preparation)
- **Azure Test Environment & Entra Setup** (can be ad-hoc, Erik owns environment)
- **Common Integration Issues and Remediation**
- **Dynamics-Specific Integration Patterns**