---
source_file: Erik - Connector API & CICD.txt
domain: Apsis One Integrations
topics: [Generic Connector API Specification, API Contract and Endpoints, Webhook Specifications, Authentication Mechanisms, CI/CD Pipeline Architecture, Deployment Process, Local Development and Testing]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Generic Connector, Justin (Integration Service), Delta Sync Manager, Mappings Manager, Full Sync Manager, GitHub Actions, AWS CodePipeline, ECS Tasks, Docker, CloudFormation]
session_type: knowledge-transfer
subdomains: [Architecture, Efficy Enterprise 12.0 Integration, Efficy Enterprise 12.1 Integration, E-deal Integration, Lead creation]
---

## Session Overview

This session covered two major areas of the Apsis One Integrations domain: (1) the **Generic Connector API specification**, which defines the contract between Apsis and external CRM systems, including all endpoints, request/response formats, and HTTP status codes; and (2) the **CI/CD pipeline architecture**, including GitHub Actions for PR checks and deployment, AWS CodePipeline for building and pushing Docker images, and the deployment process for individual microservices to staging and production environments. The session included a practical walkthrough of how to deploy a single service change to a staging environment.

---

## Generic Connector API Specification

### Overview and Design Philosophy

The **generic connector** is a unified integration framework that allows Apsis to work with any CRM system that implements the connector. Rather than building native connectors for each system, Apsis defines a standard set of endpoints and data formats that external systems must implement.

[Erik Andersson]: > "We have one connector inside of Justin and in that connector we have a like a predefined set of endpoints which we expect the external system to have implemented. If they have the generic connector support, that means that we will call any CRM that has implemented connector, we will call them in the exact same way we exact. We expect the same data and then of course like what features they have that will differ from environment to environment."

This approach is **API-first**: the specification is designed first, interfaces are generated from it, and business logic is then implemented—ensuring the specification never becomes out of date.

### Location and Distribution

The **Generic API Specification** is located in the codebase at:
```
/lib/connectors/generic/assets/
```

Two key files exist here:
1. **generic-api-spec.yaml** - Defines the endpoints and formats that external systems must implement when calling Apsis
2. **justin-generic-webhook.yaml** - Defines the webhook request bodies that Apsis expects to receive from CRM systems

The generic API spec is published to a public S3 bucket for development partners:
```
https://integration-files.appsis.one/
```

When new features are added to the specification in the code repository, they may not immediately be available in the S3 bucket. During active development of features, the spec may be sent manually via email or Teams to development partners.

### API Specification as Code-First Design

[Erik Andersson]: > "Instead of adding like instead of starting by building the business logic and adding the API points manually and then having an out to out of date API specification, we start by adding the by designing the end point in the API specification. Then we have the rate the business logic interfaces based on it to define it in the API specification, you generate the interfaces and then you actually implement the the code in the in the services and in the connectors."

The integration platform uses a framework that **generates Golang interfaces directly from the OpenAPI specification**. This ensures:
- The API spec in the codebase is always up to date (by necessity)
- Interface contracts are automatically maintained
- Implementation follows the contract

### HTTP Status Code Standardization

A critical difference from legacy connectors: the generic connector requires **proper HTTP status codes** to indicate business logic results.

[Erik Andersson]: > "For the native connectors it has been very or the legacy connectors. It has been very, very hard to handle HTTP responses because apps is using the HTTP layer to handle like the results of the business logic like if I managed to if I if I want to say retrieve a specific record using the API, but it doesn't exist. We we expect there to be a 404, whereas for example FCC Enterprise 12.0 they strictly use the HTTP layer as a transp. So if we manage to send the request to them, they will respond with a 200. It doesn't matter that like the the profile that we wanted or the content we want to retrieve might not exist. But the HTTP response is still at 200, like everything was OK."

Legacy systems like **Efficy Enterprise 12.0** used HTTP 200 for all responses and encoded business logic results in the response body (often in an error property). The generic connector eliminates this by requiring proper HTTP semantics: 404 for not found, 200 for success, etc.

### Version Management

**Adding optional properties** to endpoints does not require a version bump—CRM systems will discard unknown optional fields. **Removing existing properties requires a new version** to maintain backward compatibility.

---

## Generic Connector API Endpoints

### System Information

The **system information endpoint** is frequently called across Apsis to determine:
- CRM system metadata (ID, version)
- **Supported features** (determines which sync capabilities are displayed in the UI)
- **Deep link URL format** (for linking from Apsis profiles to corresponding CRM contacts)
- **Supported translations** (for email campaign handling)

[Erik Andersson]: > "And the URL to build that link we also retrieve from this system information. Here they say like yeah you can find contacts on this URL formats."

This endpoint is called whenever:
- User navigates to the integration page
- Any tool in Apsis lists integrations
- Apsis needs to determine what features can be synced for that integration

### Integration UI Endpoints

This group handles field mapping and subscription configuration on the integration setup page.

**Schema Endpoint**: Retrieves the list of available fields from the CRM system, including field name, type, and logical name.

**Consent/Subscription Endpoints**: Lists available consent bases or subscriptions in the CRM that can be mapped to Apsis subscriptions for synchronization.

### Profile Lists

A **profile list** is a CRM concept representing a grouping of contacts (e.g., "Newsletter Subscribers" or "VIP Customers"). The generic connector supports two types:

- **Static lists**: User-managed, explicit membership
- **Dynamic lists**: Criteria-based (e.g., "all contacts where first_name starts with 'Eric'")

When synced to Apsis, profile lists are implemented as **tags** on profiles. For each contact in the profile list, Apsis adds a tag with the list name.

[Erik Andersson]: > "So that's what this integration UI uh is like everything we need to do here in the um on in the integration page."

Two endpoints exist:
1. Retrieve available profile lists (called from UI)
2. Retrieve members of a specific list (asynchronous, called from background task)

### Record and Consent Synchronization

This group handles the core data sync operations.

**Get Records Endpoint**: Fetches contacts/records from the CRM in paginated chunks. Supports:
- Filtering by specific fields (to avoid downloading unused attributes)
- Filtering by specific record IDs (used for webhook-triggered updates)

**Get Consents Endpoint**: Retrieves consent/subscription status for contacts, paginated.

**Patch Consent Endpoint**: Updates consent status bidirectionally. When a profile opts out in Apsis, this endpoint is called to sync the change back to the CRM.

[Erik Andersson]: > "So we don't have a patch end point here for the profile for the actual profile data... we have a patch end point for consent, but we don't have a patch end point here for the profile."

**Important**: Only consent is bidirectional. Profile data is one-way (CRM → Apsis only).

### Webhook Registration and Management

Upon installation, Apsis registers webhooks in the CRM for two purposes:

**Record Update Webhooks**: Notifies Apsis when a contact is created, updated, or deleted. Parameters:
- Callback URL
- List of fields to monitor (e.g., only notify if first_name or last_name changed)
- Secret for HMAC verification

**Consent Change Webhooks**: Notifies Apsis when consent status changes in the CRM.

Endpoints also exist to **update** (e.g., when new field mappings are added) and **delete** webhooks on uninstallation.

### Campaign Endpoints

Events from Apsis activities are batched and sent to the CRM via the campaign endpoint. The CRM groups these events under a **campaign**, which is the CRM's equivalent of an Apsis activity.

[Erik Andersson]: > "If I create a e-mail send out like Eric's newsletter in APSIS, we will first create a campaign in the CR system called Eric's Newsletter. And then all of the events that happen for that activity will be like linked to this campaign inside the CRM system."

This enables bidirectional visibility: customers can view Apsis events in the CRM's campaign dashboard.

### Lead Promotion Flow

When a CRM system supports lead gathering from Apsis forms, it may convert a lead to a customer. The **promotion endpoint** handles notification that a lead ID should be updated to a real customer ID.

### E-commerce Endpoints (Deprecated)

Originally implemented for Magento/Adobe Commerce to support abandoned cart flows. **No customers currently use these endpoints**, and they will be removed from the specification.

### Debugging Endpoints

Endpoints that provide metadata for troubleshooting:
- List existing webhooks
- Retrieve webhook configuration
- Get installation configuration details
- Query current CRM domain configuration

### Attribute Management (Not Currently Implemented)

The specification includes support for CRM systems to create new attributes/fields within Apsis and automatically set up field mappings. This feature was only implemented for Maxo, which has been discontinued.

---

## Webhook Authentication

### Generic Connector: HMAC Signature

CRM systems send webhooks with **HMAC-SHA256 signatures**:

1. CRM takes the JSON request body
2. CRM concatenates with the secret (generated by Apsis during webhook registration)
3. CRM generates SHA256 hash
4. CRM sends hash in a header

Apsis performs identical verification on receipt. This approach is more secure than basic API keys.

### Legacy Connectors: API Key

Legacy integrations use simple **API key or UID** authentication, which is less secure but simpler to implement.

---

## Integration Identifier in Code

### Generic vs. Legacy Connectors

To determine if a connector uses the generic connector framework, check the **installer file** for each connector:

**Generic Connector Integrations** (inherit from generic installer):
- Efficy Enterprise 12.1 (Edc)
- Dynamics (CRM) (DynamicsCRM)
- SideShop
- Maxo
- Tribe
- WebSerum2

**Legacy Connectors** (custom installer implementations):
- Efficy Enterprise 12.0 (FSC_Enterprise)
- Dynamics 365 (DynamicsInstaller)
- Intermay Loyalty
- SAP Commerce

Code location:
```
/lib/connectors/<connector-name>/installer.go
```

Generic connectors will show:
```go
// inherits from generic installer
```

Legacy connectors will have custom functions implemented in their own installer file.

---

## Apsis Inbound Webhook Specification

[Erik Andersson]: > "Here we have the specification for the request 2 appsis for the web hooks and you will see that almost all of them here go to the Delta Sync manager."

### Endpoints (All use Delta Sync Manager)

**Record Updates**: Account, Section, Integration ID + entity type + CRM ID + status (created/updated/deleted)

**Consent Changes**: Similar structure for consent/subscription updates

**Promotions**: Convert lead ID to customer ID

**Merge**: Handles contact deduplication events from the CRM

**Subscription Creation**: CRM can trigger creation of new subscriptions in Apsis (Maxo feature)

**Attribute Creation**: CRM can create attributes in Apsis (Maxo feature, not currently used)

### Authentication

Webhooks are verified using the same HMAC signature mechanism described above.

---

## CI/CD Pipeline Architecture

### Overview

Integration uses a **hybrid CI/CD approach**:
- **GitHub Actions**: PR checks, code quality analysis (Sigrid/SonarCloud)
- **AWS CodePipeline**: Actual deployment (builds Docker images, pushes to ECR, deploys CloudFormation)

The rationale for this split is historical: GitHub Actions lacked ARM64 runners when CodePipeline support was needed.

### PR Check Flow (GitHub Actions)

Triggered on every push to a PR branch.

**Steps**:
1. Spin up local services using Docker Compose:
   - PostgreSQL database
   - Redis
   - Kafka
2. Run all unit tests across all services
3. Collect code coverage metrics (informational, does not fail build)
4. Publish results

**Duration**: ~10 minutes

**Why Docker Compose locally**: Matches production architecture, tests services in isolation, no external dependencies.

### Code Quality Analysis

**Sigrid**: Legacy system, may be deprecated in favor of SonarCloud.

**SonarCloud**: Now used by M&A team; integration team transitioning to it.

### Deployment via GitHub Actions (Planned Migration)

Currently **disabled** but ready to enable. Located at:
```
.github/workflows/deploy.yml
```

Status in repository: **Present but disabled in GitHub Actions UI**.

[Erik Andersson]: > "However, this deployment pipeline in GitHub Action is not utilized as of today. The reason for this is that when we transitioned to Um. When we transition to using the what is it called the Graviton architecture for the ECS tasks, that meant that we had to build ARM ARM 64 images. And when this project was implemented, we had no runner inside of a GitHub that was running on the ARM 64 architecture."

**Historical context**: The deploy action was disabled because ARM64 emulation in GitHub runners made builds take 3-4 hours (vs. 20 minutes on native runners). AWS CodePipeline had native ARM64 support, so it was used instead.

**Current status**: GitHub Actions now has ARM64 runners. The deploy workflow is already updated to use proper ARM64 architecture and has been tested successfully, but the migration to re-enable it in GitHub Actions has not been completed.

### AWS CodePipeline (Current Deployment Method)

Used for actual service deployments due to faster ARM64 image builds.

**Pipeline steps** (executed for each merge to develop/beta/master branches):

1. **Build service**:
   ```bash
   make build integration
   ```
   Compiles all services.

2. **Push to ECR**:
   - Pushes Docker images to AWS Elastic Container Registry
   - Pushes to staging, beta, and prod ECR repositories sequentially
   - Each push is a separate build for each environment

3. **Deploy CloudFormation**:
   - Updates ECS task definitions
   - Updates IAM roles and permissions
   - Registers new tasks with the ECS cluster

4. **Upload artifacts**:
   - Swagger API documentation
   - Generic API specification (to S3)

**Duration**: 30-40 minutes (due to sequential builds for each environment)

**Optimization opportunity**: Image promotion (build once, promote through environments) could reduce time significantly, but this was deprioritized before Erik's departure.

### Authentication & Credentials

#### AWS Access via OIDC (Recommended)

Modern approach using **OpenID Connect (OIDC)**:

[Erik Andersson]: > "Shravidya implemented a new way of dealing with this where I can't remember what the framework was called now, but essentially AVS has a flow where you can take the ID of the web hook inside the GitHub action and then you can add it to a like whitelist for the AVS account and that will enable that specific webhook to generate a short like a short lived access token."

**How it works**:
1. GitHub webhook ID is whitelisted in AWS IAM
2. GitHub Actions uses the webhook ID to request temporary credentials
3. No persistent AWS secrets are stored or rotated
4. Token is short-lived

**Implementation**: Already used in PR checks workflow:
```yaml
- name: configure-aws-credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::...
```

**Status**: Fully functional in PR checks; deploy pipeline has not yet been migrated.

#### GitHub Token (Private Repository Access)

[Erik Andersson]: > "We need this GitHub token to be able to access that package when we build the service because it's a private repository."

Each team has a **CICD user** in GitHub organization with an associated token. This token is required to access the private **email validation package** during builds.

#### AWS Access Keys (Legacy - Being Phased Out)

Some deploy steps still reference AWS access keys directly. These should be eliminated by switching to OIDC-based authentication in the deploy workflow.

---

## Local Service Deployment

### Prerequisites

**AWS Profile Setup**:
Each developer must configure AWS profiles in their credentials file with access keys for each environment:
```
~/.aws/credentials
  [staging]
  aws_access_key_id = ...
  aws_secret_access_key = ...
  
  [beta]
  aws_access_key_id = ...
  aws_secret_access_key = ...
  
  [prod]
  aws_access_key_id = ...
  aws_secret_access_key = ...
```

**Bastion Host Configuration**:
The bastion hosts and database credentials for each environment are stored in LastPass:
```
Integration Shared / Bastion Staging / Integration Bastion VM file
```

Used for SSH tunneling to access staging/prod databases from local machine.

### Deploying a Single Service

**Scenario**: Fix a bug in the Mappings Manager and deploy to staging.

**Steps**:

1. **Create feature branch**:
   ```bash
   git checkout -b fix/mappings-manager-bug
   ```

2. **Make code changes**:
   Edit the Mappings Manager source code.

3. **Navigate to service folder**:
   ```bash
   cd app/manager/mappings-manager/
   ```

4. **Deploy to staging**:
   ```bash
   AWS_PROFILE=staging make deploy
   ```

**What this does**:
- Reads AWS profile configuration (ECR endpoint, CloudFormation stack name, IAM roles)
- Builds the Docker image (using Dockerfile in the service folder)
- Pushes image to the staging ECR repository
- Invokes CloudFormation to update the ECS task definition
- ECS drains old container instances and starts new ones with the updated image

**Duration**: ~3-4 minutes for the ECS rollout (additional time from the `make deploy` command itself)

### Conditional Deployment: CloudFormation-Only Changes

If only **CloudFormation configuration** is changed (e.g., memory allocation, environment variables) without code changes:

```bash
AWS_PROFILE=staging make deploy-manager
```

This skips:
- Building Docker image
- Pushing to ECR

And deploys only the CloudFormation stack update.

### Makefile Structure

Each service inherits a common makefile pattern:

**Location**: `/lib/connectors/manager/Makefile`

**Variables** (injected from service-specific makefiles):
```makefile
SERVICE_NAME := mappings-manager
SERVICE_PORT := 8080
SERVICE_DISPLAY_NAME := Mappings Manager
TEMPLATE_PATH := templates/
```

**Shared targets** (reused across all manager services):
- `make deploy` - Full deployment
- `make deploy-manager` - CloudFormation only
- `make build` - Docker build only
- `make push` - Push to ECR only

**Why parameterized**: Eliminates duplication across 5+ manager services; all follow the same deployment pattern.

### Database Access from Local Machine

**Two options**:

**Option 1: SSH Tunnel via Bastion**

```bash
# Terminal 1: Set up tunnel
ssh -i <bastion-key> -L 5432:db-host:5432 bastion@bastion-host

# Terminal 2: Connect via psql
psql -h localhost -U postgres
```

Apsis uses a tool called **shuttle** to automate this for convenience.

**Option 2: AWS Query Editor**

Use the AWS RDS Query Editor from the console (no CLI setup required).

### Environment Variable Configurations

Service-specific environment variables are stored in:
```
/lib/connectors/<service-name>/cloudformation/config.yaml
```

These are injected into ECS task definitions during deployment.

---

## Key Takeaways

### Generic Connector API
1. **Unified contract**: One set of endpoints that all CRM integrations must implement
2. **API-first design**: Specification generated before code; interfaces auto-generated from spec
3. **Always up-to-date**: Generated directly from OpenAPI YAML in codebase
4. **Proper HTTP semantics**: Unlike legacy connectors, HTTP status codes convey business logic results
5. **Endpoint groups**: ~50+ endpoints organized by feature (schema, consent, records, webhooks, campaigns, etc.)
6. **Bidirectional consent only**: Records flow one-way (CRM → Apsis); consent is two-way

### Authentication
- **Generic connectors**: HMAC-SHA256 signatures on webhook payloads
- **Legacy connectors**: Simple API key/UID
- **GitHub Actions to AWS**: OIDC-based short-lived tokens (recommended); legacy access keys being phased out

### CI/CD Architecture
1. **Hybrid approach**: GitHub Actions for PR checks + code quality; AWS CodePipeline for deployment
2. **Rationale**: GitHub Actions lacked ARM64 support when Graviton ECS tasks were introduced; CodePipeline had native support
3. **Planned migration**: Deprecate CodePipeline and move everything to GitHub Actions (doable, just deprioritized)
4. **Deployment time**: 30-40 minutes due to sequential builds for each environment
5. **Optimization opportunity**: Image promotion pattern not yet implemented

### Local Development
1. **Make deployment simple**: Use `AWS_PROFILE=<env> make deploy` from service folder
2. **Parametrized makefiles**: All services follow identical deployment pattern
3. **Database access**: Via bastion host (SSH tunnel) or AWS Query Editor
4. **Configuration**: AWS profiles and bastion credentials stored in LastPass

---

## Unresolved Questions & Action Items

1. **S3 Bucket Access**: Lukasz reported access denied when trying to download generic API spec from integration-files.appsis.one S3 bucket. Erik clarified the correct URL, but access issue was not fully resolved in the session.

2. **GitHub Actions ARM64 Migration**: While the deploy workflow is ready, the decision to migrate from CodePipeline has not been made. Erik estimates 2-3 days of work to complete the transition and update artifact upload scripts.

3. **Sigrid vs. SonarCloud**: Unclear if Sigrid will be deprecated or if integration team will transition to SonarCloud. Status should be clarified with leadership.

4. **Azure Oauth Environment**: Legacy Azure environment (pre-APSIS acquisition by FSC) still hosts Dynamics OAuth secrets. No good solution for migration without breaking all Dynamics customers. IT wants to discontinue it, but dependency is too critical. Long-term strategy needed.

### Suggested Follow-up Sessions

1. **Azure/Dynamics OAuth Setup** (scheduled next) - Erik owns the test environment and can provide access
2. **Squid Proxy** (requires preparation) - Erik needs to prepare slides
3. **Lead Creation Flow** (requires preparation) - Erik needs to prepare slides
4. **Hands-on Exercise**: Connect to staging database via bastion (recommended for new team members Lukasz and Michal)