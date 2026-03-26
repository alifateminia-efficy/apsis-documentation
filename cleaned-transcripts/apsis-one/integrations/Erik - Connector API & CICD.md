---
source_file: Erik - Connector API & CICD.txt
domain: Apsis One Integrations
topics: [Generic Connector API specification, OpenAPI spec file locations, S3 bucket for spec distribution, Connector endpoint groups, Webhook authentication, CI/CD pipeline, GitHub Actions, AWS CodePipeline, ARM64/Graviton migration, Local and staging deployment workflow, AWS profile configuration, Bastion host and database access, Azure environment for Dynamics, OIDC authentication for AWS]
speakers: ["Erik Andersson (outgoing developer/domain expert)", "Lukasz Grabowski (incoming team)", "Michal Rosikiewicz (incoming team)", "Tomasz Kowalski (incoming team)"]
key_components: [Justin (integration service), Generic Connector, Delta Sync Manager, Mappings Manager, ECS tasks, AWS CodePipeline, GitHub Actions, S3 bucket (integration-files.apsis.one), ECR, CloudFormation, Bastion host, SSHuttle, LastPass, Sigrid, SonarCloud, Kafka, Redis]
session_type: knowledge-transfer
subdomains: ["Different Types of Connectors", "Lead creation"]
---

# Session Overview

Erik Andersson led a knowledge transfer session covering two main areas: the Generic Connector API specification (its structure, endpoint groups, file locations, and how it is shared with external CRM development partners) and the CI/CD pipeline for the integration service (Justin). The session explains why the deployment pipeline migrated from GitHub Actions to AWS CodePipeline (ARM64/Graviton build time issues), the current state of a partially-completed migration back to GitHub Actions, and the practical steps for deploying individual microservices to staging. The session also briefly touched on the Azure environment used for Dynamics OAuth and flagged it as a fragile, legacy setup requiring a dedicated future session.

---

## Generic Connector API Specification: Purpose and Design Philosophy

The **Generic Connector** (inside the service called **Justin**) works by defining a fixed, predefined set of endpoints that any external CRM system must implement if it wants to support the generic connector. Apsis calls every compliant CRM in exactly the same way and expects the same data format; what differs between environments is which features each CRM declares it supports.

### Unified HTTP Response Handling

A key motivation for the generic connector design was to fix a longstanding problem with **legacy connectors**: they were very hard to work with because CRM systems like FSC Enterprise 12.0 used HTTP strictly as a transport layer. If a request was sent and the requested record did not exist, they would still return `200 OK` and embed an error inside the response body. The generic connector removes this entirely — CRMs are now required to return proper HTTP status codes (e.g., `404` for a missing record). Data format and HTTP responses are fully unified.

### API Spec File Location

The specification lives inside the codebase under:

```
lib/connectors/generic/assets/
```

Two files exist here:

- **Generic API spec** — defines the endpoints Apsis *will call* on the CRM, the request bodies it will send, the expected response format, and the valid HTTP status codes.
- **Justin Generic Webhook spec** — defines the endpoints and request body format the CRM must use when *calling back* to Apsis (webhooks/notifications), and the HTTP codes it can expect Apsis to return.

### Why the Spec Is Always Up to Date in the Codebase

Justin uses a framework that **generates Go interfaces directly from the OpenAPI specification**. The workflow is:

1. Design the endpoint in the API specification YAML first.
2. Generate the Go interfaces from it.
3. Implement the business logic against those interfaces.

This means the spec cannot drift out of sync with the implementation — it is the source of truth for interfaces.

### Sharing the Spec with External CRM Partners

The spec files are published to a **publicly accessible S3 bucket**:

```
s3://integration-files.apsis.one
```

This bucket is publicly exposed (no secret — the generic connector API is not confidential). Files can be downloaded directly via HTTPS from that bucket URL.

> "This bucket contains the generic connector API — this is nothing secret, so it is open for everyone."

**Caveat:** When a new feature is added to the spec in the code repository, it may not yet be uploaded to the S3 bucket. Erik occasionally sends the latest spec manually via email or Teams to CRM development partners who are actively building support for a new feature. The upload to S3 is triggered by a script that runs as part of the deployment pipeline.

---

## Generic Connector API Endpoint Groups

The API specification contains many endpoints, organized into logical groups. The following are the most important ones to understand:

### System Information

- Returns metadata: instance ID, version, etc.
- Most important sub-endpoint: **supported features** — the CRM lists which capabilities it has. Apsis forwards this to tools like Apsis One to control UI toggles (e.g., "can sync email", "can sync events").
- Also returns: the **URL format for deep-linking** to a CRM contact from an Apsis profile page (used in the Unified Data / Maxo project).
- Called frequently: on every page load of the integration page, and whenever any Apsis tool checks whether a feature can be synced.

### Integration UI

Endpoints used on the integration configuration page:

- **Schema endpoint** — retrieves all available fields (field name, type, logical name) from the CRM for a given entity (e.g., contacts). Used to populate the field mapping UI.
- **Consent basis / subscriptions** — lists the consent bases in the CRM system so they can be mapped to Apsis subscriptions in the integration UI.

### Profile Lists

- Two separate endpoint groups for profile lists (e.g., "VIP Customers", "Technical Newsletter"):
  - **List of lists** — called from the UI; returns the available profile lists, their IDs, entity types, and whether they are **static** (manually managed) or **dynamic** (criteria-based, like Apsis segments).
  - **List members** — called asynchronously from an ECS background task; retrieves the actual contact members of a list.
- When synced into Apsis, each profile list membership is stored as a **tag** on the Apsis profile. E.g., "Apsis Gold Users" list → tag labelled "Apsis Gold Users" added to each member's profile.

### Records and Consents (Full Sync and Webhooks)

- **Get records for entities** (e.g., contacts) — paginated, supports field filtering. If a mapping covers only 5 of 50 fields, only those 5 are requested. Also used when a webhook notification arrives: Apsis calls back to the CRM with the specific CRM ID from the notification to fetch the latest data.
- **Get consents** — paginated list of all consent states from the CRM, including the CRM-side consent base ID and the consent value.
- **PATCH consent** — used when a contact opts out inside Apsis; Apsis calls back to the CRM to update the consent there. Consent is the **only bidirectional data** — there is no PATCH endpoint for profile data.
- **Authentication header** — all generic connector calls use an `X-API-Key` header.

### Webhook Registration and Management

- On **installation**, Apsis registers webhooks with the CRM specifying:
  - The callback URL (generated at installation time, including account ID, section, and integration ID).
  - The fields of interest (so CRM only sends notifications when those fields change).
  - A **secret** for HMAC signature verification.
- Separate webhooks for record updates and consent changes.
- Endpoints also exist to **update** webhooks (e.g., after adding more field mappings) and to **delete** webhooks on uninstallation (for information security — the CRM must stop sending data once the customer stops using Apsis).

### Webhook Authentication (Generic vs. Legacy)

- **Generic connector**: Apsis generates a secret at webhook registration time. The CRM takes the request payload + secret, hashes them (HMAC), and sends the hash. Apsis verifies by recomputing the hash with the stored secret.
- **Legacy connectors**: Simple API key / UID sent in the request. More straightforward but less secure.

### Campaigns (Event Sync)

- Used to send batches of email events (opens, clicks, etc.) from Apsis to the CRM.
- Each Apsis activity instance (e.g., "Eric's Newsletter" email send) maps to a **campaign** in the CRM. All events for that activity are linked to the campaign, so CRM users can view all Apsis events from within the CRM's campaign view.

### Promotions (Lead Flow)

- Relevant only when the CRM has implemented the **lead gathering flow**.
- When a form submission creates a lead/opportunity in the CRM (not a full contact), the CRM eventually notifies Apsis when that lead is **promoted** to a real contact, providing the real CRM ID.
- Apsis then updates the profile to replace the lead ID with the real CRM ID.
- Covered in more detail in the lead gathering flow session.

### Consent-Based Subscription Sync (Maxo-only, currently unused)

- Allowed the CRM to **create Apsis subscriptions** directly via webhook: CRM creates a consent base → Apsis creates a corresponding subscription and sets up the mapping automatically.
- No current customers use this (Maxo was discontinued).

### Attribute Management (currently unused)

- Allowed CRM systems to create new attributes in Apsis from within the CRM, with automatic field mapping setup.
- No current customers use this (was Maxo-only).

### E-commerce Endpoints (to be removed)

- Built for a generic connector implementation for Magento/Adobe Commerce (not a CRM).
- Use case: abandoned cart emails and purchase confirmation emails.
- No current customers use this. Erik stated he intends to **remove these endpoints before handover**.

### Debugging Endpoints

- Not used in business logic, but useful for support:
  - List existing webhooks on the CRM instance.
  - Retrieve webhook configuration details.
  - Check which API domain the installation is pointing to.
  - Various metadata useful for diagnosing webhook issues.

---

## How to Identify Whether a Connector is Generic or Legacy

To check whether a specific connector uses the generic installer or has a custom legacy installer, look at the installer file for each connector:

```
lib/connectors/<connector_name>/installer.go
```

- **Generic connector** users (e.g., FSC Corporate / EDL, Dynamics, Side Shop, Maxo, FSC 12.1, Tribe, Web CRM 2): their installer simply **inherits/overloads from the generic installer** — no custom functions defined.
- **Legacy connectors** (e.g., FSC Enterprise, Dynamics [older], Intermay Loyalty): each defines its own custom installer functions.

> "Easiest will be to check the list I'm going to provide you, but you will learn this by heart after some time."

[⚠️ The promised list of which connectors are generic vs. legacy was not provided in this session — it is an action item.]

---

## API Versioning Rules

- Adding **new optional properties** to existing endpoints: no version bump required. CRM systems will discard unknown optional properties if they don't support them.
- **Removing existing properties**: a new version is required.

---

## CI/CD Pipeline Architecture

### Current State: Two-Tool Setup

The integration service (Justin) currently uses **two separate CI/CD tools**:

1. **GitHub Actions** — used for:
   - PR checks (unit tests, code coverage reporting, Sigrid/SonarCloud publication).
   - ⚠️ Note: Sigrid may be discontinued; SonarCloud is already used for MA and Ecom. Unclear if SonarCloud is fully active on integration yet.

2. **AWS CodePipeline** — used for:
   - All actual deployments (staging, beta, prod EU, prod APAC).

### Why CodePipeline Instead of GitHub Actions for Deployment

When the service migrated to the **Graviton (ARM64) architecture** for ECS tasks, Docker images had to be built for `linux/arm64`. At the time of migration, no GitHub Actions runner supported ARM64 natively, so builds required **emulation**. This increased deployment time from ~20 minutes to **3–4 hours** — unacceptable.

The solution: keep PR checks in GitHub Actions (no Docker build needed), and move deployments to AWS CodePipeline, which had ARM64 runners available and was already used for MA.

### Current Status of the GitHub Actions Deploy Workflow

A `deploy.yml` GitHub Actions workflow file **exists in the repository** and has been tested with the new ARM64 GitHub runner. However, it is **currently disabled** in GitHub Actions. CodePipeline remains in use for deployments.

To verify:

> In GitHub Actions → the "Deploy Justin" workflow shows as disabled.

### Path to Migrating Back to GitHub Actions (Improvement Item)

Erik estimates this would take **2–3 days maximum**:

1. The deploy YAML already has the correct ARM64 configuration.
2. One step referencing a Swagger upload script name needs updating.
3. The AWS credential approach needs updating (see OIDC section below).
4. CodePipeline would then be disabled.

> "I honestly don't think it will be a very big project to do it."

---

## AWS Authentication in GitHub Actions (OIDC vs. Access Keys)

### Legacy Approach (problematic)

Previously used a dedicated AWS IAM user with a static access key/secret stored in GitHub secrets. Key rotation was manual and painful — required updating both AWS and GitHub secrets.

### Current Approach: OIDC (implemented by Shravidya)

The PR checks workflow now uses **OIDC (OpenID Connect)**:

- The GitHub Actions workflow's webhook ID is **whitelisted** in the AWS account.
- The workflow assumes an IAM role using that webhook ID.
- AWS generates a **short-lived access token** — no static secrets needed.
- No rotation required.

In the workflow YAML this appears as a step with `role-to-assume` referencing the IAM role ARN.

The legacy deploy workflow still references static access key/secret steps. When migrating to GitHub Actions for deployment, **those steps should be replaced** with the same OIDC role-assumption approach.

---

## GitHub Token for Private Package Access

A **CICD user** exists in the GitHub organization with a GitHub token attached. This token is needed because Justin depends on a **private repository** (the email validation package). The token allows the GitHub Actions runner to access that package during `go build`.

This token is stored as a GitHub Actions secret called `PROVISIONED_GITHUB_TOKEN` (or similar).

---

## PR Check Pipeline Details

- Runs on every PR creation and every push to a PR branch.
- Steps: spins up a local database, Redis, and Kafka via `docker-compose up`, then runs all unit tests.
- Code coverage is reported but does **not** fail the build.
- Runtime: approximately **10 minutes**.

---

## Deployment Pipeline Details (CodePipeline)

### What Gets Deployed

Deployments are per-branch:
- `develop` branch → staging
- `beta` branch → beta
- `master` branch → prod EU and prod APAC

### Structure: Per-Service Makefiles

Each microservice has its own `Makefile` that includes shared logic from a central `manager_funcs.mk` (or similar). The root `Makefile` iterates over all services (managers, workers, async features) and calls each one.

Deploy steps per service:
1. Deploy ECR repository (if not already existing).
2. Build Docker image.
3. Push Docker image.
4. Deploy CloudFormation stack (ECS task definition, IAM permissions, etc.).

Deployment time: **30–40 minutes** end-to-end for all services.

**Improvement opportunity noted by Felix:** Promote images between environments (staging → beta → prod) instead of rebuilding from source each time. Not yet implemented due to scope.

### Deploying a Single Service to Staging (Step-by-Step)

To deploy only one service (e.g., Mappings Manager) to staging:

1. Ensure your AWS credentials file has a profile named `staging` with valid credentials.
2. Navigate to the service directory:
   ```
   apps/mappings_manager/
   ```
3. Run:
   ```bash
   AWS_PROFILE=staging make deploy
   ```

This triggers: ECR repo deploy → Docker build → Docker push → CloudFormation deploy → ECS task update.

**Shortcut for infrastructure-only changes** (no code change, e.g., changing memory allocation in CloudFormation):

```bash
AWS_PROFILE=staging make deploy_manager
```

This skips the Docker build/push and only updates the CloudFormation stack.

ECS deployment takes **3–4 minutes** after push (image pull, registration, routing update, drain of old task).

> "Local development is better for managers [for iteration speed], but deploying to staging will still work."

### AWS Profile Naming Convention

⚠️ **Critical:** The deploy scripts are written expecting specific AWS profile names. Changing them will break the scripts because they look up configuration files by profile name.

The scripts use the AWS profile name to look up:
- ECR host URL
- Bastion host
- DB host
- Parameter Store (SSM) paths (PEM files, etc.)

**Use the profile names as defined.** These are documented in the integration shared folder in **LastPass**.

---

## Database Access (Staging and Production)

Credentials and connection details are stored in the **LastPass integration shared folder**.

Connection method uses **SSHuttle** (SSH-based VPN via bastion host):

1. In one terminal: run the SSHuttle command to set up the tunnel via the bastion host (acts as a VPN).
2. In a second terminal: connect to PostgreSQL — traffic is redirected through the tunnel to the RDS instance inside the VPC.

Alternative: use the **AWS RDS Query Editor** in the AWS console.

---

## Azure Environment for Dynamics (Flagged as Fragile — Separate Session Planned)

The Azure OAuth application used by all Dynamics customers exists in an **old Azure environment** under the "Apsis International AB" domain — a tenant that predates Apsis being acquired by FSC and that the IT department has wanted to decommission for years.

**Why it can't be migrated easily:**

> "You can't move a secret to another environment. We would need to create a new secret, and that means every Dynamics customer would need to completely undo everything they have done, reinstall using the new secret — and that is not really an option."

**Current situation:** Erik owns access to this environment. This is a known risk (access control, maintenance responsibility, IT wanting to shut it down). A dedicated KT session is planned to cover this, including access handover and subscription details.

---

## Key Takeaways

1. **The Generic Connector API spec** is the single source of truth for all generic connector integrations. It lives at `lib/connectors/generic/assets/` and is also published to `s3://integration-files.apsis.one` for external CRM partners. It is always up to date because Go interfaces are generated from it.
2. **Endpoint groups** to be aware of: system info/supported features, integration UI (schema, consent bases), records/consents (full sync + webhooks), webhook registration/management, profile lists, campaigns (event sync), and promotions (lead flow).
3. **Consent is the only bidirectional data** — there is a PATCH endpoint for consent, but not for profile data.
4. **Legacy vs. generic connector identification**: check whether the connector's `installer.go` uses the generic installer base or defines its own functions.
5. **CI/CD uses two tools**: GitHub Actions for PR checks, AWS CodePipeline for deployments. This split was forced by ARM64 build time issues. Migration back to unified GitHub Actions is planned, estimated at 2–3 days effort, and is largely already set up.
6. **AWS authentication** in GitHub Actions should use OIDC (already implemented for PR checks) rather than static access key/secret rotation.
7. **Deploying a single service to staging** is straightforward: set the correct `AWS_PROFILE`, navigate to the service folder, run `make deploy`.
8. **AWS profile names must match** what the scripts expect — see LastPass integration shared folder for the correct names and credential files.
9. **The Azure environment for Dynamics** is a known fragile dependency under an old Apsis tenant. Do not attempt changes without a dedicated handover session.
10. **E-commerce endpoints** in the generic connector spec are unused and planned for removal.

---

## Unresolved Questions and Action Items

- [ ] **Erik to provide a list** of which connectors are generic vs. legacy (promised but not delivered in session).
- [ ] **Improvement task**: Migrate deployment from AWS CodePipeline to GitHub Actions `deploy.yml` (~2–3 days effort). Key steps: update Swagger upload script reference, apply OIDC authentication, disable CodePipeline.
- [ ] **Improvement task**: Implement image promotion between environments (staging → beta → prod) instead of rebuilding, to reduce 30–40 minute deploy times (flagged by Felix, not yet scoped).
- [ ] **Michal** to set up staging database connection (SSHuttle via bastion host) and verify access.
- [ ] **Dedicated session needed**: Azure environment for Dynamics — access handover, subscription details, OAuth app secrets, IT decommission risk.
- [ ] **Next sessions planned**: Squid proxy (requires Erik preparation/slides), Lead creation (requires slides), Dynamics / Azure Test Environment / Entra (ad hoc), remediation of common issues.
- [ ] Confirm whether **SonarCloud is fully active** on the integration service (Erik was uncertain).
- [ ] Confirm whether **Sigrid is being discontinued** for the team and what replaces it.