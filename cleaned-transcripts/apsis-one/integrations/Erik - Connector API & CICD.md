---
source_file: Erik - Connector API & CICD.txt
domain: Apsis One Integrations
topics: [Generic Connector API Specification, API Contract Design, Webhook Authentication, CI/CD Pipeline Architecture, Deployment Workflows, GitHub Actions, AWS Code Pipeline, Docker Image Building, ECS Task Deployment, AWS IAM/OIDC Authentication, Local Development Workflow]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Generic Connector, Generic API Spec, Justin Generic Webhook Spec, Delta Sync Manager, Mappings Manager, Integration Manager, Full Sync Manager, GitHub Actions, AWS Code Pipeline, AWS ECR, AWS ECS, AWS S3, Bastion Host, PostgreSQL]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector, Inbound Flow, Outbound Flow, Microsoft Dynamics]
---

## Session Overview

This session covered two major areas of the Apsis One Integrations domain: (1) the **Generic Connector API Specification** and how it standardizes communication between Apsis and external CRM systems, and (2) the **CI/CD pipeline architecture** used to build, test, and deploy integration services. Erik provided a comprehensive walkthrough of the API contract design philosophy, the structure of the API specification files, and the endpoint categories. The second half focused on deployment workflows, the transition from pure GitHub Actions to a hybrid approach using AWS Code Pipeline, and practical steps for deploying changes to staging/production environments.

---

## Generic Connector API Specification

### Overview and Design Philosophy

The **generic connector** is Apsis's unified approach to integrating with external CRM systems. Instead of building a custom connector for each CRM, Apsis defines a standardized set of endpoints and data formats that any CRM can implement.

[Erik Andersson]: The key principle is that we have one connector inside Justin that calls external systems in the exact same way. We expect the same data structure, and what features they support will differ from environment to environment, but we can verify that using generic connector endpoints.

This approach solves a critical problem from the legacy connector era: **HTTP response handling inconsistency**. Legacy connectors struggled because apps like Efficy Enterprise 12.0 use HTTP as pure transport—they return 200 OK even when the requested record doesn't exist, placing error information in the response body instead. The generic connector enforces proper HTTP semantics.

[Erik Andersson]: For legacy connectors, it was very hard to handle HTTP responses because apps use the HTTP layer to handle results of business logic. If a record doesn't exist, we expect 404. But FCC Enterprise 12.0 strictly uses HTTP as transport—they respond with 200 if they reach the instance, regardless of whether the data exists. The error lives in a response property. We've removed all this messiness in the generic connector. Now we expect proper HTTP status codes.

### Generic API Specification Files Location and Structure

The specification files are located in the codebase at:

```
lib/connectors/generic/assets/
```

Two key files exist here:

1. **`generic-api-spec.yaml`** — Defines the endpoints that external CRM systems must implement and the request/response formats Apsis will use when calling them.

2. **`justin-generic-webhook.yaml`** — Defines the request bodies and formats that Apsis expects from webhook calls initiated by the CRM system.

[Erik Andersson]: The generic API spec defines "this is what we will send to you and what we expect you to respond with." The webhook document defines "this is what we expect you to call us with" and the data format and HTTP codes you can expect from us.

### API Specification Generation and Version Control

A critical design decision ensures the specification is always in sync with implementation:

[Erik Andersson]: We have a framework in Golang that generates interfaces based on the API specifications. Instead of building business logic first and adding API points manually—leaving the spec out of date—we design the endpoint in the API specification first. We generate the interfaces from the spec, then implement the code in the services. This ensures the code base version is always up to date.

**Versioning caveat:** Adding new optional properties to endpoints does not require a version bump because CRM systems will discard properties they don't support. However, **removing existing properties requires a new version**.

### Sharing the Specification with Partners

The generic API spec is not secret and is shared publicly with development partners and CRM teams.

**Current process:**
- The specification lives in the code repository (always up to date).
- A production S3 bucket at `integration-files.apsis.one` hosts a public copy.
- During deployment pipelines, the spec is uploaded to this bucket.
- **Important timing caveat**: If you add a new feature to the spec in the code repo, it may not yet be in the S3 bucket available to external partners until the next deployment. Occasionally, team members send the latest spec manually via email or Teams if a partner is keen to develop support for a new feature.

[Lukasz Grabowski]: I tried to access the bucket and got access denied.

[Erik Andersson]: The bucket should be publicly accessible. You can download the YAML file directly from the S3 link—it's just file storage, not an internal API docs page exposed in Swagger.

---

## Generic API Specification Endpoint Categories

### System Information Endpoint

This endpoint exposes metadata about the CRM system:

- **CRM ID and instance version** — For debugging purposes.
- **Supported features** — A critical list that tells Apsis which sync features are enabled (sync email, sync event, etc.). This information is forwarded to Apsis tools and determines what UI elements and features are available to the user.
- **Deep linking support** — The URL format for building direct links to contacts in the CRM. If this integration supports deep linking in the unified data project, the URL pattern is retrieved here.
- **Translation support** — Which languages are supported when handling email campaign creation from the CRM.

[Erik Andersson]: This endpoint is very frequently used. We call it whenever you go to the integration page or list integrations in any Apsis tool to see if we can sync that specific feature.

### Integration UI Endpoints

These endpoints support the integration configuration page where users map fields and set up subscriptions.

#### Schema Endpoint
Retrieves the schema (field definitions) for entities like contacts:
- Field name
- Field type
- Logical name in the CRM system

Example: When setting up field mappings in an E-deal integration, the schema endpoint returns what fields exist in E-deal and their types.

#### Consent/Subscription Endpoints
Used to retrieve consent bases or subscriptions in the CRM so users can map them to Apsis subscriptions during integration setup.

#### Profile List Endpoints
Retrieves lists (or segments) from the CRM system.

[Erik Andersson]: Imagine you have an email list in the CRM called "Technical Newsletters" containing every contact who wants technical newsletter emails. The profile list endpoint returns the list of lists—e.g., "Apsis Gold Users" with ID 123, the record types it contains, and whether it's static (user-managed) or dynamic (criteria-based, like a segment).

When synced to Apsis, a tag is added to each profile with the label of the list name, which can then be used for email targeting.

**Static vs. Dynamic lists distinction:**
- **Static**: Manually managed list of contacts.
- **Dynamic**: Criteria-based list (e.g., "contacts whose first name is Eric"). Membership changes as data changes.

### E-Commerce Endpoints (Deprecated)

These endpoints handled abandoned carts and other e-commerce use cases. Erik notes that this functionality is no longer used and will be removed before handing the code over since Apsis now works only with actual CRM systems, not e-commerce platforms.

### Record Synchronization Endpoints

These endpoints are core to the full sync and real-time update workflows:

#### Get Records for Entities (Paginated)
Fetches all records (e.g., contacts) from the CRM system with pagination.

**Key parameters:**
- `fields` — Specify which fields to download. If the integration only maps 5 fields, don't download all 50 available attributes; download only what's mapped in Apsis.
- CRM ID filtering — Can specify a single CRM ID to fetch data for just that record.
- `X-API-Key` header — Authentication token.

[Erik Andersson]: Pagination is essential because you might have 2 million contacts. You don't want to download 20 attributes for 2 million contacts in one go—you'll kill any service. If you've only mapped 5 fields, we only download those 5, not all 50 attributes that might exist.

#### Get Consents (Paginated)
Retrieves all consent records for contacts, including the CRM consent base ID. Apsis maps these to internal subscriptions.

#### Patch Consent Endpoint
Updates a specific consent/subscription for a contact (e.g., when someone opts out in Apsis, this endpoint is called to reflect that change in the CRM).

**Important asymmetry**: Consent is bidirectional—there's a patch endpoint for consent updates but not for general profile data updates.

### Webhook Management Endpoints

#### Register Webhooks
On installation, Apsis registers webhooks with the CRM to receive notifications when records are updated.

**Parameters:**
- Notification trigger type (e.g., "record updated")
- Callback URL where the CRM should POST notifications
- List of fields of interest (CRM only sends updates if these fields change)
- Webhook secret — Used for request signature verification

#### Delete and Update Webhooks
Support removal and modification of registered webhooks (e.g., if additional field mappings are added, the webhook can be updated to monitor those fields too).

### Campaign Endpoint

Events from Apsis activities are sent to this endpoint, grouped under a campaign (the CRM's equivalent of an Apsis activity instance).

[Erik Andersson]: If I create an email send called "Eric's Newsletter" in Apsis, we create a campaign in the CRM called "Eric's Newsletter," and all events for that activity are linked to it. If the user navigates to campaigns in the CRM and clicks on "Eric's Newsletter," they see all the Apsis events synchronized there.

### Promotions Endpoint (Lead Gathering)

Used when a lead created via Apsis forms is converted to a customer in the CRM. The CRM notifies Apsis, which then adds the real CRM ID to the profile (replacing the temporary lead ID).

[Erik Andersson]: A lead is not a real customer—it might not have all the data. When a lead is promoted to an actual contact, they notify us and we add the real CRM ID to that profile.

### Debugging Endpoints

Endpoints not used in business logic but helpful for troubleshooting:
- List all webhooks registered in the CRM
- Retrieve webhook configurations and details
- Get current configuration for specific integration sections
- Retrieve metadata like which API domain the installation points to

---

## Justin Generic Webhook Specification

The webhook specification defines the request bodies Apsis expects when the CRM sends notifications. Almost all endpoints route to the **Delta Sync Manager** service.

### Record Update Webhook

Notification when a record is created, updated, or deleted:
- CRM ID
- Change type: `created`, `updated`, or `deleted`

[Erik Andersson]: There's no real difference between created and updated because we perform the same actions, but we include it for future differentiation. Deleted is handled differently.

### Consent Change Webhook

Notifies Apsis when consent/subscriptions are modified in the CRM.

### Merge Webhook

When duplicate contacts are identified and merged, the CRM notifies Apsis to consolidate profiles.

### Subscription/Attribute Creation Webhooks

**Subscription Creation** (via Metadata Sync Manager): The CRM can create subscriptions in Apsis by sending webhook notifications. This eliminates manual mapping—if the CRM creates a consent base, Apsis automatically creates the corresponding subscription and sets up the mapping.

[Erik Andersson]: For Maxo, this was supported. The CRM could create a consent base like "Consent for SMS Sendings," and we'd receive a webhook, create a corresponding subscription in Apsis, and set up the mapping automatically. No need to create in the CRM, then in Apsis, then manually map.

**Attribute Creation** (via Metadata Sync Manager): Similar to subscriptions—the CRM creates a field, and Apsis acts as a proxy to create the attribute in its system and set up field mappings.

---

## Webhook Authentication

### Generic Connector: HMAC-SHA256 Signature Verification

The CRM signs webhook requests using the secret provided during webhook registration:

1. Take the request body
2. Concatenate with the webhook secret
3. Generate HMAC-SHA256 hash
4. Include the hash in the request

Apsis verifies by:
1. Taking the incoming request body and the stored secret
2. Generating the same hash
3. Comparing hashes

[Erik Andersson]: The secret is generated on installation time when we create the webhook. The CRM needs to use the exact same secret to sign requests.

### Legacy Connectors: Basic API Key

Legacy integrations use a simpler approach: a UID or basic API key without signature verification.

---

## Connector Type Identification

**How to determine if a connector uses the generic API or is a legacy connector:**

Check the installer file for each connector:
- If the installer inherits from the generic installer (e.g., `EfFicy E-deal` or `Dynamics`), it's a generic connector.
- If it has a custom installer with tailored functions, it's a legacy connector.

[Erik Andersson]: The easiest way is to check a list I'll provide, but you'll learn by heart which connectors work which way after some time.

**Generic Connector Integrations:**
- Efficy E-deal (formerly FSC Corporate)
- Microsoft Dynamics
- Efficy Enterprise 12.1
- Maxo
- Tribe
- Web Serum 2

**Legacy Connectors (Custom Installers):**
- Efficy Enterprise 12.0 (FSC)
- Intermay Loyalty
- (Possibly others)

---

## CI/CD Pipeline Architecture

### Overview: Hybrid Approach

Apsis Integration uses **two different tech stacks** for CI/CD:

1. **GitHub Actions** — For PR checks and publishing (e.g., to Sigrid/SonarCloud)
2. **AWS Code Pipeline** — For actual deployment (currently active)

[Erik Andersson]: We have a hybrid approach because when we transitioned to Graviton ARM 64 architecture for ECS tasks, GitHub Actions runners didn't support ARM 64 natively. Building ARM 64 images with emulation took 3-4 hours instead of 20 minutes, so we moved deployment to AWS Code Pipeline.

**Status**: The GitHub Actions deploy workflow exists in the repository but is currently **disabled**. The intention is to migrate back to GitHub Actions now that they have ARM 64 runners.

### PR Check Flow (GitHub Actions)

**Trigger**: Every push to a PR or changes to a PR branch.

**Steps**:
1. Spin up local infrastructure using `docker-compose up`:
   - Local PostgreSQL database
   - Local Redis
   - Local Kafka
2. Run all unit tests across all services
3. Check code coverage (informational, doesn't fail the build)

**Duration**: ~10 minutes

**Rationale**: Mimics local development workflow—tests happen against local infrastructure without requiring AWS credentials or external services.

### Code Quality Publishing (GitHub Actions)

A separate GitHub Action publishes code metrics to **Sigrid** (or potentially **SonarCloud** for some projects; status uncertain).

[Erik Andersson]: We may transition to SonarCloud instead of Sigrid. I actually don't remember if Sigrid is still active for the integration service.

### AWS Code Pipeline Deployment (Current Production)

**Trigger**: Merge to develop, beta, or master branch.

**Workflow**:
1. Build Docker images for each service
2. Push images to Amazon ECR (Elastic Container Registry)
3. Deploy CloudFormation templates to create/update ECS tasks
4. Manage IAM permissions for each service

**Deployment Targets**:
- `develop` branch → Staging environment
- `beta` branch → Beta environment
- `master` branch → Production EU and Production APAC

**Duration**: 30-40 minutes in worst case (includes multi-region deployment)

**Performance Issue**: Currently, the pipeline builds the same image and pushes it to ECR for staging, beta, and prod. An optimization (discussed but not yet implemented) would be to build once and promote the image through environments, reducing deployment time significantly.

[Erik Andersson]: We build and push the image for staging, then build and push for beta, then build and push for prod. Felix wanted us to promote images from staging to beta to prod instead, but that was a bigger project than we had time for.

### Manual Deployment to Staging

For testing feature branches before merging, developers can deploy individual services manually using **make commands**.

---

## Local Development and Manual Deployment

### Prerequisites

1. **AWS CLI configured** with profiles for `staging`, `beta`, and `prod`
2. **Access credentials** stored in `~/.aws/credentials`
3. **Bastion host credentials** for database access (stored in LastPass under `integration shared folder`)

[Lukasz Grabowski]: Where can we find the bastion staging VM file?

[Erik Andersson]: In the integration shared folder in LastPass.

### Basic Deployment Process

**Example: Deploy a change to mappings-manager to staging**

1. Create a feature branch:
   ```bash
   git checkout -b test-feature
   ```

2. Make code changes (e.g., add debugging output, fix a bug)

3. Navigate to the service folder:
   ```bash
   cd apps/mappings-manager
   ```

4. Deploy using make with the staging AWS profile:
   ```bash
   AWS_PROFILE=staging make deploy
   ```

**What `make deploy` does**:
- Builds the Dockerfile
- Pushes the Docker image to the staging ECR repository
- Deploys the CloudFormation template for the service
- Configures IAM permissions
- Updates the ECS task definition
- The ECS service drains old tasks and starts new ones (takes ~3-4 minutes)

### Make File Structure

Each service has a `Makefile` that inherits shared logic from `manager-funcs.mk` (for manager services) or similar shared files.

**Key variables defined**:
- `SERVICE_PORT` — Which port the service listens on
- `CONTAINER_NAME` — Docker image name
- `SERVICE_DISPLAY_NAME` — Human-readable name for CloudFormation/ECS
- `MEMORY_ALLOCATION` — ECS task memory
- `STACK_NAME` — CloudFormation stack identifier
- `TARGET_CLUSTER` — Which ECS cluster to deploy to

[Erik Andersson]: We have lots of variables so you don't repeat logic. You just give inputs to shared scripts—what is the port, the name, the display name in Apsis—and we inject that into the build and deploy process.

**Build parameters** passed to Docker:
- Application folder name
- Service name and port
- Template locations
- All derived from make file variables

### CloudFormation-Only Deployments

If you only modify CloudFormation configuration (e.g., increase memory, change environment variables) without changing application code:

```bash
AWS_PROFILE=staging make deploy-cf
```

This skips Docker image building and pushing, saving time.

### AWS Profile Naming Convention

**Important**: Make scripts are designed with specific AWS profile names in mind:
- `staging`
- `beta`
- `prod`

Scripts automatically look up configuration files and populate them with:
- ECR host
- RDS/PostgreSQL host
- Bastion host
- Database credentials

[Erik Andersson]: If you change the AWS profile names, the command will fail because it can't find the configuration file with that name. I really recommend following these profile names.

### Database Access

Two approaches:

1. **SSH Tunnel via Bastion Host** (recommended):
   - Set up an SSH tunnel using the bastion host credentials from LastPass
   - Use `shuttle` (SSH tunneling tool) to proxy database connections
   - Connect to PSQL through the tunnel
   ```bash
   shuttle psql staging
   ```

2. **AWS Query Editor**:
   - Use the RDS Query Editor inside the AWS Console
   - No credentials needed, relies on IAM permissions

---

## Authentication and Credential Management

### GitHub Token for Private Package Access

Each integration service needs to access a private email validation package from another repository. A **GitHub CICD user token** is provisioned for this.

[Erik Andersson]: With the new GitHub organization structure, each team has a CICD user. This user can have one or more GitHub tokens, each dedicated to a separate service. We need the token because we use a private package from another repository, and without the token, we can't access it during the Docker build.

### AWS Credentials Management: OIDC (OpenID Connect)

**Historical problem**: AWS access keys and secret keys were stored in GitHub Secrets and manually rotated—a cumbersome process.

**Current solution (implemented by Shrividia)**: Use **OIDC** (OpenID Connect) with GitHub webhooks.

1. GitHub webhook ID is whitelisted in AWS
2. The webhook can assume an AWS role without needing an access key or secret key
3. Short-lived access tokens are generated based on the webhook ID
4. No credential rotation needed

[Erik Andersson]: You take the ID of the webhook inside the GitHub action and add it to a whitelist in the AWS account. This enables that specific webhook to generate short-lived access tokens. You don't need to store or rotate AWS secrets.

**Status**: Already implemented for PR checks and publishing steps. The deploy workflow has not yet been updated to use OIDC (still uses access key/secret).

[Lukasz Grabowski]: So you have `assume-role` in the PR checks workflow but the deploy pipeline still uses access keys?

[Erik Andersson]: Correct. The PR checks use OIDC and assume a role. The deploy workflow in Code Pipeline still uses the old access key/secret method. When we migrate deployment back to GitHub Actions, we can use the same OIDC approach there.

---

## Deployment Workflow Summary for New Team Members

### Local Testing (Before PR)

1. Use local development environment (Docker Compose)
2. Run unit tests and manual testing locally
3. No need for AWS credentials

### Creating a PR

1. Push feature branch to GitHub
2. GitHub Actions automatically runs PR checks (~10 minutes)
3. Code coverage and other metrics are published

### Testing on Staging

1. Ensure AWS profile `staging` is configured with correct credentials
2. Make code changes and commit
3. Navigate to the service folder
4. Run `AWS_PROFILE=staging make deploy`
5. Wait 3-4 minutes for ECS to drain and start new tasks
6. Test the changes in the staging environment

### Merging to Production

1. Create PR against `develop`, `beta`, or `master` branch
2. PR check passes automatically
3. Code review and approval
4. Merge to branch
5. AWS Code Pipeline automatically triggers and deploys to appropriate environment (currently ~30-40 minutes)

---

## Known Issues and Technical Debt

### 1. Deployment Pipeline Efficiency

**Issue**: Images are rebuilt and pushed to ECR for each environment (staging → beta → prod) instead of being promoted.

**Impact**: Deployment takes 30-40 minutes. Could be reduced to ~10-15 minutes with image promotion strategy.

**Status**: Identified but not yet implemented.

### 2. CloudFormation Update Scripts

**Issue**: Some deploy scripts reference outdated shell script names and locations. These work but need cleanup when migrating deployment from Code Pipeline to GitHub Actions.

**Time to fix**: ~30 minutes to change script references.

### 3. Legacy Azure OAuth Setup for Dynamics

**Issue**: Microsoft Dynamics connector uses OAuth secrets stored in an old Azure environment (pre-APSIS acquisition by FSC). This environment is not owned by IT and is slated for discontinuation, but secrets cannot be moved to a new environment without breaking all existing Dynamics customer integrations.

[Erik Andersson]: The upside is we have full control. The downside is we're responsible for maintenance. If someone leaves the company, we must ensure access is revoked. We can't move the secret to a new environment because secrets can't be migrated—we'd have to create a new secret, and every Dynamics customer would need to reinstall completely with the new secret. That's not an option.

**Status**: Ongoing maintenance required. A dedicated session on this will be scheduled.

---

## Next Steps and Action Items

### High Priority

1. **Database Connectivity Exercise** (Homework):
   - Michal and Lukasz: Set up AWS profile for staging
   - Configure SSH tunnel via bastion host using LastPass credentials
   - Connect to PostgreSQL database and verify access
   - Alternatively, use AWS Query Editor in the console

2. **Migrate GitHub Actions Deployment Workflow** (Future Project):
   - Replace AWS Code Pipeline with GitHub Actions for deployment
   - Update workflow to use ARM 64 support (now available in GitHub runners)
   - Update CloudFormation script references
   - Enable OIDC for AWS credential management in deploy step
   - **Estimated effort**: 2-3 days

### Medium Priority

3. **Prepare for Squid Proxy Session** — Erik to prepare slides

4. **Prepare for Lead Creation Session** — Erik to prepare slides

5. **Microsoft Dynamics / Azure Setup Session** — Ad hoc, covers Azure OAuth environment, Entra, test environment configuration

### Lower Priority

6. Investigate image promotion strategy for faster deployments

7. Clean up CloudFormation script references in deploy pipeline

---

## Key Takeaways

1. **Generic Connector Design**: The generic API spec is the contract between Apsis and external CRM systems. It's code-generated from the spec, ensuring the code base version is always up to date. Adding optional properties doesn't require a version bump; removing properties does.

2. **Endpoint Categories**: The spec is organized logically—system info, integration UI, records, consents, webhooks, campaigns, promotions, and debugging endpoints. Understanding the categories helps navigate what's available.

3. **Webhook Security**: Generic connectors use HMAC-SHA256 signature verification with secrets. Legacy connectors use basic API keys. Verification happens on both sides (CRM signs, Apsis verifies).

4. **Connector Types**: Identify whether a connector is generic or legacy by checking the installer file. Generic connectors inherit from `generic_installer`. Legacy connectors have custom installers.

5. **Hybrid CI/CD Stack**: PR checks and code publishing run in GitHub Actions (~10 minutes). Deployment currently uses AWS Code Pipeline (~30-40 minutes). The GitHub Actions deploy workflow exists but is disabled; migrating back is a 2-3 day project.

6. **Manual Deployment**: Use `AWS_PROFILE=staging make deploy` from the service folder. Requires AWS credentials and proper profile setup. Takes 3-4 minutes for ECS to drain and start new tasks.

7. **OIDC and Credentials**: PR checks use OIDC for AWS auth (no secrets to rotate). Deploy workflow still uses access keys but can be updated to OIDC. GitHub token for private package access is handled via CICD user in GitHub organization.

8. **AWS Profile Naming**: Use standard profile names (`staging`, `beta`, `prod`). Scripts look up configuration files by name. Changing names breaks the deployment flow.

9. **Bastion Host Access**: All database access requires tunneling through a bastion host. Credentials are in LastPass. Use `shuttle` for SSH tunneling or AWS Query Editor.

10. **Future Session Topics**: Squid proxy, lead creation flow, Microsoft Dynamics/Azure OAuth environment setup, and common debugging/remediation issues will be covered in subsequent sessions.

---

## Unresolved Questions

1. **S3 Bucket Access**: Lukasz encountered an "access denied" error when trying to access `integration-files.apsis.one`. The bucket should be publicly accessible; this may be an IP allowlist issue or a temporary access problem. **Action**: Erik to investigate.

2. **Sigrid vs. SonarCloud**: Erik was uncertain whether Sigrid is still active for the integration service or if it has been replaced with SonarCloud. **Action**: Verify the current code quality publishing setup.

3. **GitHub Actions ARM 64 Runner Timing**: Lukasz asked if GitHub Actions now supports ARM 64 runners. Erik confirmed they do but didn't provide details on when this was added or specific runner specs. **Action**: Document the new runner specs and estimated build time.

4. **Azure OAuth Migration Path**: A long-term question—how can the legacy Azure OAuth environment be migrated to a new, supported environment without breaking all existing Dynamics customers? This was identified as a problem but no solution is currently in place. **Action**: Schedule dedicated discussion with IT and Dynamics stakeholders.