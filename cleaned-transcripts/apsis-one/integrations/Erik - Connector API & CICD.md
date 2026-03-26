---
source_file: Erik - Connector API & CICD.txt
domain: Apsis One Integrations
topics: [Generic Connector API Specification, Connector Architecture, Legacy vs Generic vs Third-Party Connectors, API Versioning, CI/CD Pipeline, GitHub Actions, AWS Code Pipeline, Deployment Workflow, Webhook Authentication, System Information Endpoints, Field Mappings, Consent Synchronization, Profile Lists, Lead Creation, Database Connectivity]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Generic Connector API, Justin Integration Service, AWS ECS, AWS Code Pipeline, GitHub Actions, Docker, ECR, Kubernetes/ECS Tasks, Mapping Manager, Delta Sync Manager, Full Sync Manager, S3 Bucket (integration-files.apsis.one), Azure Entra/OAuth, LastPass, AWS Bastion Host]
session_type: knowledge-transfer
subdomains: ["Different Types of Connectors", "Lead creation"]
---

## Session Overview

This session covered two major technical domains within Apsis One Integrations: the **Generic Connector API specification** and the **CI/CD pipeline infrastructure**. The team reviewed the OpenAPI specification that defines the contract between Apsis One and external CRM systems implementing the generic connector, including endpoints for system information, data synchronization, consent management, webhook registration, and profile list handling. The second half focused on deployment workflows, comparing the legacy AWS Code Pipeline approach with the newer GitHub Actions infrastructure, and demonstrating practical steps for deploying individual microservices to staging environments.

---

## Generic Connector Architecture & API Specification

### Overview of the Generic Connector Model

The generic connector is Apsis One's standardized interface for third-party CRM systems to integrate. Unlike **legacy connectors** (built in-house by Apsis) and **third-party connectors** (built and maintained by external companies), the generic connector establishes a unified contract that all implementing CRM systems must follow.

[Erik Andersson]: The fundamental principle is that we have one connector inside of Justin with a predefined set of endpoints which we expect the external system to have implemented. If they have generic connector support, we will call any CRM that has implemented it in exactly the same way, expecting the same data format and structure, though features may differ between environments.

This unified approach solves a critical problem that plagued legacy connectors: inconsistent HTTP response handling. In legacy connectors, different CRM systems used the HTTP layer differently — some properly return 404 for missing records, while others (like FCC Enterprise 12.0) return 200 for all requests and communicate errors through response body properties. The generic connector mandates proper HTTP status codes and unified data formats.

### Location and Versioning of the Generic API Spec

**File path**: `/Lib/Connectors/generic-connector/assets/`

Two critical OpenAPI specification files reside here:

1. **generic-api-spec.yaml** — Defines the endpoints the external CRM system must implement and the request/response format Apsis expects from them
2. **apsis-generic-webhook.yaml** — Defines the webhook request format that CRM systems must use when calling back to Apsis

**Auto-updated via code generation**: The specification in the codebase is always kept in sync because a framework in Justin generates Golang interfaces directly from the OpenAPI specification. The workflow is:

1. Design the endpoint in the API specification
2. Generate interfaces from the spec
3. Implement the business logic

This ensures the spec never drifts from implementation.

### Distribution to Partners

The generic API spec is publicly accessible and shared with development partners:

- **S3 bucket**: `integration-files.apsis.one` (public, non-authenticated access)
- **File storage purpose**: The bucket serves as a downloadable file repository, not as a Swagger UI or interactive API documentation
- **Update timing**: The spec is uploaded to the S3 bucket during deployment (confirmed during staging/production merge)
- **Caveat**: If a new feature is added to the specification in the codebase, it may not immediately appear in the S3 bucket until the next deployment. In urgent cases, Erik can send the updated YAML file manually via email or Teams to customers actively developing support for that feature

[Lukasz Grabowski]: I've opened the integration-files URL in the browser and get access denied. Is it protected by IP whitelisting?

[Erik Andersson]: No, IP whitelisting is not the mechanism. The bucket is designed for direct file downloads via a specific URL pattern (though Erik noted needing to verify the exact public URL at the time of discussion).

### API Specification Structure: Grouped by Feature

The OpenAPI spec is logically grouped by feature area. Key groupings include:

#### System Information Endpoint
**Purpose**: CRM systems declare their capabilities and metadata.

**Returns**:
- System ID and instance version
- **Supported Features** (critical for feature gating) — The list of capabilities the CRM supports (e.g., can sync email, can sync events, etc.). This information is forwarded to the Apsis platform to control which sync features are displayed in the UI.
- **Deep Linking URL Pattern** — Format for building direct links from Apsis profiles to the corresponding contact in the CRM system
- **Email Campaign Translation Support** — Which translations are supported when creating email campaign synopses from CRM data

[Erik Andersson]: This endpoint is very frequently called. We invoke it whenever you visit the integration page or list integrations from any tool in Apsis to determine which features can be synced with that specific CRM system.

#### Integration UI Endpoints
These endpoints support the Apsis integration configuration page:

- **Schema Endpoint** — Retrieves the structure of CRM objects (e.g., "Person" entity). Returns field names, data types, and logical field names
- **Consent/Subscription Lists** — Lists available consent bases in the CRM so mappings can be configured
- **Profile Lists Endpoint** — Returns available lists/segments in the CRM (e.g., "Apsis Gold Users", "Technical Newsletter", "VIP Customers")

**Profile Lists Details**: A profile list represents either a static list (manually curated contacts) or a dynamic list (segment based on criteria). When synced to Apsis, each list member receives a tag matching the list name, allowing email campaigns to be sent to those segments.

The distinction is tracked via a `dynamic: true/false` property:
- **Static**: Membership is manually maintained
- **Dynamic**: Membership updates automatically based on criteria (e.g., "first name starts with 'Eric'")

#### E-Commerce Endpoints (Deprecated)
These endpoints supported abandoned cart retrieval for Adobe Commerce/Magento. They are no longer in use and Erik plans to remove them before handover, as no current customers use e-commerce systems.

#### Data Synchronization & Consent Endpoints

**Retrieve Contacts/Records**: 
- `GET /records/{entityType}` — Paginated retrieval of contacts from the CRM
- **Pagination is critical**: With millions of records, you don't fetch all 20+ attributes at once; you'd overwhelm the service
- **Field filtering**: Specify only the fields that have mappings configured in Apsis to avoid downloading unnecessary attributes
- **Single record retrieval**: To support webhook-triggered syncs, you can request a specific CRM ID and download just that contact's latest data

**Retrieve Consents/Subscriptions**:
- `GET /consents` — Paginated list of subscription/consent records in the CRM
- Returns the consent base ID (CRM-specific ID, not Apsis ID) and associated data
- Used during bidirectional sync to map Apsis subscriptions to CRM consent records

**Update Consent (Bidirectional)**:
- `PATCH /consents` — The only profile attribute that is bidirectional
- When a user opts out from an email in Apsis, a PATCH request updates the corresponding CRM consent record
- Apsis checks the consent mapping to determine which CRM subscription to update

**Webhook Registration**:
- `POST /webhooks/records` — Register to receive notifications when records are updated in the CRM
- Apsis specifies the callback URL, which fields to monitor, and a secret for HMAC verification
- `POST /webhooks/consents` — Register for consent change notifications
- **Update capability**: If field mappings are added later, the webhook registration can be updated to include those new fields without re-registering
- `DELETE /webhooks/*` — Remove webhooks when integration is disabled (security requirement)

#### Lead Gathering & Promotion

**Use Case**: When a form submission in Apsis creates a lead/opportunity in the CRM, and that lead is later converted to a real customer:

- Initial sync creates a profile with a **lead ID** (not a real CRM contact ID)
- When the lead is promoted to a customer, the CRM notifies Apsis via the promotion webhook
- Apsis updates the profile with the real **CRM ID**

[Erik Andersson]: This is two separate things. A lead is not a real customer; it might not have all the data. We cover this more closely in the lead gathering flow.

#### Campaign Endpoints

Events from Apsis (email sent, email opened, email clicked, etc.) are batched and sent to the CRM system via the campaign endpoint. Events are grouped under a campaign object, which corresponds to an Apsis activity (e.g., "Eric's Newsletter" email send).

When customers view campaigns in the CRM system, all Apsis-generated events for that campaign are displayed.

---

## Webhook Authentication & Security

### Generic Connector: HMAC-Based Signature Verification

For generic connector webhooks, Apsis uses **HMAC-SHA256 signature verification**:

1. CRM system takes the **request body (payload) + the secret** generated during webhook registration
2. CRM hashes this combination and includes the hash in the `X-Signature` header (or similar)
3. Apsis receives the webhook, performs the same hash calculation, and verifies the signature matches

This ensures:
- Webhook came from the registered CRM system
- Payload was not tampered with in transit

### Legacy Connector: API Key Authentication

Legacy connectors do not use HMAC signatures. Instead, they use a simpler **basic API key** or **UID** that the CRM system includes in the request header. While more straightforward, this is less secure than signature verification.

### Identifying Connector Type in Code

To determine if a connector is generic or legacy, check the **installer file** for each connector:

**Generic connectors** inherit from the generic installer:
```
installer: GenericInstaller
```

Examples: FSC Corporate (EDL), Dynamics, Tribe, WebSerum, MaxO, FSC Enterprise 12.1

**Legacy connectors** define custom installer functions:
```
installer.go (custom function)
```

Examples: FSC Enterprise, Dynamics (some versions), Intermay Loyalty

[Erik Andersson]: The easiest way to know is the list I'll provide you, but you'll learn this by heart after some time which connector works in what way.

---

## API Specification Versioning Strategy

### Adding vs. Removing Properties

**Adding optional properties**: No version bump required
- Example: The `PATCH /consents` endpoint retroactively added `source` and `last_updated` properties
- CRM systems simply ignore unknown optional properties

**Removing existing properties**: Version bump required
- Any removal breaks backward compatibility and requires a new API version

**Rationale**: This strategy allows features to be added without forcing all CRM implementations to update immediately.

---

## CI/CD Pipeline Infrastructure

### Overview: Dual Tech Stack Transition

The integration domain currently uses **two different CI/CD technologies**:

1. **GitHub Actions** (primary for PR checks and Sigrid/SAST scanning) — Actively used
2. **AWS Code Pipeline** (for deployments) — Actively used for builds and ECR pushes

**Reason for dual stack**: When the team migrated to ARM 64 architecture for ECS tasks (Graviton), GitHub Actions runners did not support native ARM 64 builds. Building with emulation increased deployment time from ~20 minutes to 3-4 hours. AWS Code Pipeline offered native ARM 64 runners, so it was adopted for builds. The intent is to eventually consolidate back to GitHub Actions now that GitHub supports ARM 64 runners, but this has not yet been prioritized.

### GitHub Actions Workflow

#### PR Check Workflow

**Trigger**: Every push to a pull request

**Steps**:
1. Spin up local development environment using Docker Compose:
   - PostgreSQL database
   - Redis
   - Kafka
2. Run all unit tests across all services
3. Report code coverage (not a blocker, informational only)
4. Check code quality via Sigrid (SAST scanning)

**Duration**: ~10-20 minutes

**Authentication**: Uses **OIDC** (OpenID Connect) with GitHub's built-in OIDC provider to assume AWS roles. No AWS access keys are stored in GitHub secrets. [Lukasz Grabowski]: Previously the team stored hardcoded AWS access keys in GitHub secrets, which created a rotation problem. The new OIDC approach eliminates this security risk.

**GitHub Token**: A **provisioned GitHub token** for the integration CICD user is stored in GitHub secrets to allow access to private repositories (specifically the email validation package in another repo).

#### Deploy Workflow (Currently Disabled in GitHub Actions)

**Status**: Defined in `.github/workflows/deploy-justin.yml` but disabled in the Actions tab

**Why disabled**: Designed for AWS Code Pipeline instead (see below)

**Intend**: Once GitHub's ARM 64 runners are confirmed stable, this workflow will be re-enabled and AWS Code Pipeline will be deprecated.

**Expected effort to enable**: 2-3 days of work to ensure all script references (especially Swagger file uploads) are correctly configured.

### AWS Code Pipeline Workflow (Current Deployment Path)

**Trigger**: Merge to `develop`, `beta`, or `master` branches

**Branch → Environment Mapping**:
- `develop` → Staging
- `beta` → Beta
- `master` → Prod EU + Prod APAC (two separate deployments)

**Build Steps** (defined in `scripts/build` and `scripts/deploy`):

1. **Boilerplate setup**: Configure AWS credentials, set up profiles, assume IAM roles
2. **Build Docker image**: For each microservice (Mapping Manager, Delta Sync Manager, Full Sync Manager, etc.)
3. **Push to ECR**: Upload image to AWS Elastic Container Registry
4. **Upload OpenAPI specs**: Push swagger files to internal docs repository (`docs-internal`) and upload the generic API spec to the S3 bucket
5. **Deploy CloudFormation**: Create or update ECS task definitions, register tasks in the cluster
6. **Update IAM permissions**: Ensure the ECS task has permissions to decrypt KMS keys, write to CloudFormation, etc.

**Duration**: 30-40 minutes in worst case

**What makes deployment slow**: 
- Docker image is built and pushed for each environment (staging, beta, prod EU, prod APAC) separately
- Image promotion (staging → beta → prod) is not implemented

**Optimization opportunity**: Instead of rebuilding the image for each environment, build once in staging and promote that image to beta and prod. This could significantly reduce deployment time. Felix requested this but it was not prioritized before the team's planned transition.

### AWS Infrastructure Profile Management

Deployment requires AWS profiles configured on your local machine. The scripts expect **specific profile names** that map to configuration files:

**Profile names and their locations** (stored in LastPass under "integration shared folder"):
- `staging` — Configuration for staging environment
- `beta` — Configuration for beta environment  
- `prod` — Configuration for production environment (EU)
- `prod-apac` — Configuration for production environment (APAC)

Each profile points to:
- **ECR registry host** (AWS account-specific)
- **Bastion host** (for SSH tunneling to databases)
- **RDS database host**
- **VPC/security group info**
- **PAM (Privileged Access Management) credentials file location**

[Erik Andersson]: If you change the profile names, you will have issues because the command will fail when it can't find the corresponding configuration file.

---

## Deployment Workflow: Step-by-Step Hands-On

### Scenario: Deploy a Change to a Single Microservice (Staging)

**Example**: Fix a bug in the Mapping Manager and test it on staging.

**Steps**:

1. **Create and commit change**:
   ```bash
   git checkout -b feature/fix-mapping-bug
   # Edit code
   git add .
   git commit -m "Fix: is_trial_account permission issue"
   ```

2. **Verify AWS profile is configured**:
   ```bash
   aws configure --profile staging
   # Enter credentials from LastPass
   ```

3. **Navigate to service directory**:
   ```bash
   cd apps/mappings-manager
   ```

4. **Review the Makefile structure**:
   - Each service's Makefile includes shared functions from `scripts/manager_funcs.mk` (or equivalent)
   - Available targets: `make deploy`, `make deploy-cf` (CloudFormation only)

5. **Deploy the service**:
   ```bash
   AWS_PROFILE=staging make deploy
   ```
   
   This executes:
   - Deploy ECR repository
   - Build Docker image (with service-specific parameters from Makefile variables like `APP_NAME`, `SERVICE_PORT`, `SERVICE_MEMORY`)
   - Push image to ECR
   - Deploy CloudFormation stack (creates ECS task definition, registers with cluster)
   - Apply IAM permissions

6. **Wait for deployment to complete**: 3-4 minutes for ECS to drain old tasks, register new ones, and update routing.

### Optimization: CloudFormation-Only Changes

If you only modify the CloudFormation template (e.g., increase memory allocation) without code changes:

```bash
AWS_PROFILE=staging make deploy-cf
```

This skips the expensive Docker build and ECR push steps.

### Makefile Variables & Parameterization

Each service's Makefile defines variables that are passed to shared deployment scripts. Example from Mapping Manager:

```makefile
SERVICE_NAME = mappings-manager
SERVICE_PORT = 8080
SERVICE_MEMORY = 512
APP_FOLDER = mappings-manager
STACK_NAME = integration-mappings-manager-stack
CLUSTER_NAME = integration-cluster
```

These are injected into the Docker build and CloudFormation templates, eliminating duplication.

---

## Database Connectivity & Development Setup

### Accessing Staging/Prod Databases

Databases in AWS are not directly accessible from outside the VPC. Access requires SSH tunneling through a bastion host.

**Two approaches**:

1. **SSH Tunnel (Shuttle method)**:
   ```bash
   # Terminal 1: Establish tunnel
   shuttle -L 5432:db-host.internal:5432 bastion-user@bastion-host.com
   
   # Terminal 2: Connect via localhost
   psql -h localhost -U dbuser -d integration_db
   ```

2. **AWS RDS Query Editor**: Use the AWS console to run queries directly (no local setup required).

**Credentials**: Stored in LastPass under "integration shared folder" — includes bastion host address, database endpoint, PAM file location, and credentials.

---

## AWS Credential Rotation & Security

### Legacy Approach (Current Issue)

Previously, the integration CICD user had a static AWS access key pair stored in GitHub secrets. This created ongoing maintenance burdens:

- Keys do not auto-rotate
- Manual rotation required updating both AWS IAM and GitHub secrets
- High security risk if compromised

### New Approach: OIDC (OpenID Connect)

Implemented by Shravidya, OIDC allows GitHub to generate short-lived access tokens without storing permanent AWS credentials:

1. GitHub's OIDC provider is whitelisted in the AWS account
2. During GitHub Actions execution, GitHub automatically provides an OIDC token
3. This token can assume a specific IAM role (no access keys needed)
4. Tokens expire after the job completes

**Status in codebase**: 
- PR checks already use OIDC (no AWS secrets visible in the workflow)
- Deploy workflow is not yet updated to use OIDC (still references legacy access key/secret)

**Action item**: Migrate deploy workflow to use OIDC (eliminates the need for the access key setup step).

---

## Key Takeaways

1. **Generic Connector API is the heart of third-party integration**: It enforces a single contract that all CRM systems must implement. The OpenAPI spec in the codebase is always up to date due to code generation from the spec.

2. **API spec distribution via S3**: The generic API spec is publicly available in the `integration-files.apsis.one` S3 bucket for partners to download. New features may not appear in S3 immediately after code commit.

3. **Webhook authentication differs by connector type**: Generic connectors use HMAC-SHA256 signature verification; legacy connectors use basic API keys. Always check the installer file to determine the type.

4. **No version bump for optional properties**: The versioning strategy allows backward-compatible additions without forcing updates. Only removals require a new API version.

5. **Dual CI/CD stack is temporary**: GitHub Actions handles PR checks and SAST scanning. AWS Code Pipeline handles builds and deployments (due to ARM 64 support). Plan is to consolidate back to GitHub Actions.

6. **Deployment is parameterized via Makefiles**: Each service inherits shared deployment logic. Use `make deploy` for full deployment or `make deploy-cf` for CloudFormation-only changes. Always use the correct AWS profile name (staging, beta, prod, prod-apac).

7. **ARM 64 emulation in GitHub was the blocker**: GitHub's lack of native ARM 64 runners forced the adoption of AWS Code Pipeline. Now that GitHub supports ARM 64, the deployment workflow should be migrated to eliminate the dual-stack complexity.

8. **Database access requires bastion tunneling**: Databases are not directly accessible from outside the VPC. Use shuttle for SSH tunneling or AWS RDS Query Editor for queries.

9. **OIDC credentials are more secure than hardcoded keys**: The new OIDC approach eliminates the need for storing permanent AWS access keys in GitHub. Legacy access key rotation is no longer necessary.

10. **Identifying connector types is important**: Check the installer file to determine if a connector is generic (inherits GenericInstaller) or legacy (custom installer function). This affects authentication method and feature support.

---

## Unresolved Questions & Action Items

### Questions Raised During Session

1. **Sigrid/SAST integration**: Is Sigrid still in use, or has it been replaced by SonarCloud? Status unclear; needs verification.

2. **GitHub ARM 64 runner stability**: The team has tested the ARM 64 runner and confirmed Docker image builds work, but hasn't fully transitioned to using it for deployments. Needs decision on migration timing.

3. **Swagger file upload in GitHub Actions**: The current deploy workflow has outdated references to script names for uploading Swagger files. These need to be reviewed and corrected before enabling the GitHub Actions deployment.

### Recommended Action Items

1. **Migrate deployment from AWS Code Pipeline to GitHub Actions**:
   - Enable the `deploy-justin.yml` workflow
   - Update script references for Swagger file uploads
   - Verify OIDC credential flow works in deploy workflow
   - Disable/deprecate AWS Code Pipeline once verified
   - **Estimated effort**: 2-3 days

2. **Implement Docker image promotion**:
   - Build once (in staging)
   - Promote to beta and prod instead of rebuilding
   - Could reduce deployment time from 30-40 minutes to ~10-15 minutes
   - **Not prioritized before Erik's transition**

3. **Database connectivity verification**:
   - Michal and Lukasz should connect to staging database using provided bastion credentials
   - Verify LastPass access to the "integration shared folder"
   - Test shuttle tunneling workflow

4. **Consolidate AWS profile configuration**:
   - Document exact profile names and their locations in developer onboarding guide
   - Consider automating profile setup instead of requiring manual LastPass retrieval

5. **Prepare for Azure Entra/OAuth session**:
   - Review the outdated Azure environment setup (pre-Apsis acquisition)
   - Document dependencies and migration path if Azure secrets need to be rotated
   - This will be covered in a separate follow-up session

---

## Related Topics for Future Sessions

- **Squid Proxy Configuration** (requires preparation/slides)
- **Lead Gathering & Form Submissions** (requires preparation/slides)
- **Azure Entra OAuth Application & Secrets** (ad hoc, Erik owns the test environment)
- **Remediation of Common Integration Issues** (TBD)
- **Dynamic Connector Implementation Details** (CRM-specific customization)