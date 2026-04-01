---
source_file: Erik - Connector API & CICD.txt
domain: Apsis One Integrations
topics: [Generic Connector API Specification, OpenAPI Spec Structure, Connector Endpoint Groups, Webhook Authentication, CI/CD Pipeline, GitHub Actions, AWS CodePipeline, ARM64/Graviton Migration, Local Deployment Workflow, AWS Profile Setup, Database Access, Azure Environment for Dynamics]
speakers: ["Erik Andersson (outgoing engineer, subject matter expert)", "Lukasz Grabowski (incoming engineer)", "Michal Rosikiewicz (incoming engineer)", "Tomasz Kowalski (incoming engineer)"]
key_components: [Generic Connector, Justin (integration service), Mappings Manager, Delta Sync Manager, Full Sync Manager, ECS Tasks, AWS CodePipeline, GitHub Actions, S3 (integration-files.apsis.one), ECR, CloudFormation, Sigrid, SonarCloud, Audience, APSIS One, Dynamics CRM, FSC Enterprise, EDL (E-deal), Maxo]
session_type: knowledge-transfer
---

# Session Overview

This knowledge transfer session covers two main areas: (1) the Generic Connector API specification — its structure, endpoint groupings, how it is versioned and distributed to CRM development partners, and the webhook authentication model; and (2) the CI/CD pipeline for the integration service, including why AWS CodePipeline is currently used instead of GitHub Actions for deployments (ARM64/Graviton build time issue), the current state of a partially completed migration back to GitHub Actions, and a live walkthrough of how to manually deploy a single microservice to staging. The session also briefly touches on AWS profile setup, database access via bastion host, and the Azure environment used for Dynamics OAuth credentials.

---

## Generic Connector API Specification: Purpose and Design Philosophy

### Overview of the Generic Connector Model

The **Generic Connector** (referred to internally as part of "Justin", the integration service) works by defining a single connector inside Justin with a **predefined set of endpoints** that any external CRM system must implement in order to support the generic connector.

> "We will call any CRM that has implemented [the generic] connector in the exact same way. We expect the same data."

The features a given CRM supports will differ per environment, but this can be verified via the generic connector endpoints themselves. The contract is strictly regulated by the **Generic API Specification**.

### What the Spec Defines

The Generic API Spec defines:
- Every endpoint that exists in the generic connector
- Exact request body formats Justin will send to the CRM
- Expected response formats and allowable HTTP status codes

This is in deliberate contrast to legacy/native connectors where HTTP response handling was very inconsistent. Example of the legacy problem:

> "FSC Enterprise 12.0 strictly uses the HTTP layer as a transport. If we managed to send the request to them, they respond with a 200 — it doesn't matter that the profile we wanted doesn't exist. The HTTP response is still a 200. The error is inside a property in the response body."

In the generic connector, the CRM is required to return proper HTTP status codes (e.g., 404 for not found). All of this — data format, HTTP responses, etc. — has been unified.

### Location in the Codebase

The spec files live at:

```
lib/connectors/generic/assets/
```

Two files are present:
1. **`generic_api_spec` (YAML)** — Defines what Justin will send to the CRM and what it expects back. This is the "outbound" contract.
2. **`justin_generic_webhook` (YAML)** — Defines what the CRM must send *to* Justin (webhook/notification requests), the expected data formats, and the HTTP codes Justin will respond with.

### Why the Codebase Version Is Always Up to Date

The integration service uses a framework that **generates Go interfaces from the OpenAPI specification**. The workflow is:

1. Design/add the endpoint in the YAML spec file.
2. Generate Go interfaces from the spec.
3. Implement the business logic against those interfaces.

This means you cannot have code that diverges from the spec — the spec is always the source of truth. You cannot add code first and update the spec later.

### Distribution to External Partners

The spec files are also published to a **publicly accessible S3 bucket**:

```
s3://integration-files.apsis.one
```

This bucket is publicly exposed (no authentication required for download). When a deployment runs, a script uploads the latest generic API spec to this bucket so development partners can always download the current version.

**Important caveat:** If you are actively developing a new feature in the generic API, the updated spec may already be in the codebase repository but **not yet uploaded to the S3 bucket** (i.e., not yet deployed). In that case, Erik sometimes manually sends the spec via email or Teams to CRM partners who are keen to start implementing support immediately.

---

## Generic API Spec: Endpoint Groups and Their Purpose

The spec has a large number of endpoints organized into logical groupings. The following are the key groups:

### System Information

- Returns metadata: CRM instance ID, version, etc.
- Not used in business logic directly, but useful for **debugging**.
- More importantly: contains the **supported features** list — this is what Justin uses to determine which APSIS tool features (e.g., can sync email, can sync events) are available for a given integration.
- Also contains the **deep linking URL template** for building direct links from an APSIS profile to the corresponding CRM contact.
- Contains supported translation locales for email campaign sync.

> "This endpoint is very frequently used. We call it in a lot of different places — whenever you go to the integration page, whenever you list the integration from us, meaning whenever you go to any tool in APSIS. This will be called to see if we can sync that specific feature."

### Integration UI

Endpoints used on the integration configuration/setup page:

- **Schema endpoint** (`get records schema for persons`): Returns field names, field types, and logical names for CRM fields. Used in the E-deal (EDL) field mapping UI.
- **Consent basis / subscriptions endpoint**: Lists what consent bases exist in the CRM and can be mapped. Used to set up subscription mappings in the integration UI.

*Note from session:* These calls go to Justin first, which then proxies the call to the CRM instance. Requests are not visible in the browser console as direct calls to the CRM.

### Profile Lists

- **List endpoint** (called from the UI, synchronously): Returns the available profile lists (email lists / contact groups) in the CRM, including their IDs, record types, and whether they are **static** or **dynamic**.
  - **Static list**: A manually curated list of contacts.
  - **Dynamic list**: Filter-based, equivalent to an APSIS segment (e.g., "all contacts whose first name is Erik"). Contacts can appear/disappear based on data changes.
- **Members endpoint** (called from ECS background task, asynchronously): Retrieves the actual members of a specific profile list.

When profile lists are synced into APSIS, each CRM contact ID on the list gets a **tag** added in APSIS with the label of the list name (e.g., "APSIS Gold Users"). This tag can then be used to target email sends.

### Records / Full Sync / Consent Sync

The core data sync endpoints:

- **Get records for entity (e.g., contacts)** — Paginated. Supports field filtering so Justin only downloads fields that have a configured mapping (avoids downloading all 50 attributes when only 5 are mapped).
- **Get records by specific IDs** — Used by the webhook flow: when the CRM notifies Justin that contact `123` was updated, Justin calls back to fetch only the latest data for that specific ID.
- **Get consents (paginated)** — Returns consent base IDs (CRM-side IDs, not APSIS subscription IDs) and consent state. Justin looks up the consent mapping to find the corresponding APSIS subscription and updates it.
- **Patch consent** — Called when someone opts out in APSIS; Justin calls back to the CRM to update the consent state there. This is the **only bidirectional data flow** — there is no patch endpoint for profile data itself.

### Webhook Registration / Management

- **Register webhook (on install)** — Justin tells the CRM: "Notify me at this URL when records or consent are updated, for these specific fields, using this secret."
  - Fields of interest are specified so the CRM only needs to notify Justin when relevant fields (e.g., first name, last name) change.
  - A **secret** is generated and shared; the CRM must include an HMAC hash of the payload + secret with every webhook call.
  - Separate webhook registration endpoints exist for record updates and consent changes.
- **Update webhook** — Used when new field mappings are added; Justin can update the CRM on which fields it now wants to be notified about.
- **Delete webhook (on uninstall)** — Required for information security: once a customer stops using APSIS, the CRM must not continue sending data. Delete endpoints exist for this cleanup.

The webhook callback URL is generated at installation time and includes the account ID, section, and integration ID.

### Consent-Based Subscription Sync (Metadata)

*Currently only supported by Maxo; not in active use as Maxo is discontinued.*

Allows a CRM to create/delete subscriptions inside APSIS by calling Justin. Justin acts as a proxy to the Audience service. This eliminates the need for manual steps: create consent list in CRM → create subscription in APSIS → go to integration page and map them. With this feature, creating a consent base in the CRM automatically creates and maps the corresponding APSIS subscription.

### Promotions (Lead Promotion)

Used when the CRM has implemented the **lead gathering flow** (retrieving leads from APSIS forms). A lead/opportunity in the CRM has a "lead ID" that differs from a real customer CRM ID. When the lead is promoted to an actual contact in the CRM, the CRM calls Justin with a promotion notification. Justin then updates the APSIS profile to replace the lead ID with the real CRM ID.

> "We will cover this more closely in the lead gathering flow [future KT session]."

### Campaign / Event Batches

- Every APSIS activity instance (e.g., an email send-out called "Erik's Newsletter") is represented as a **campaign** in the CRM system.
- Justin creates the campaign in the CRM first, then sends batches of events (opens, clicks, bounces, etc.) linked to that campaign.
- In the CRM's campaigns view, all APSIS events for that activity will be visible.

### E-Commerce Endpoints

*No longer in active use. Was built for a Magento/Adobe Commerce generic connector implementation.*

Handled abandoned cart notifications and purchase confirmation emails. Erik noted intent to **remove these endpoints** before handover since no customer currently uses them and current integrations are all CRM systems, not e-commerce platforms.

### Attribute Creation (Proxy to Audience)

*Supported by Maxo; not in active use.*

Allows CRM systems to create attributes in APSIS directly. CRM creates a new field (e.g., "Success Rating") → calls Justin → Justin creates the same attribute in Audience and adds it to field mappings for the integration. Today, attributes must be created from within APSIS manually.

### Debugging Endpoints

Not used in production business logic. Available for support/debugging scenarios:
- Retrieve what webhooks currently exist in the CRM.
- Retrieve current configuration (e.g., which API domain a specific installation is pointing to).
- Lots of metadata useful for diagnosing issues like "why are my webhooks not working?"

---

## Webhook Authentication: Generic vs. Legacy Connectors

### Generic Connector (HMAC Signature)

For all webhook calls *from* the CRM *to* Justin, the CRM must:
1. Take the full request payload body.
2. Combine it with the **secret** that Justin generated and shared at webhook registration time.
3. Hash this combination and send the hash to Justin.

Justin performs the same verification on its side: takes the incoming body, adds the stored secret, generates the hash, and verifies it matches.

### Legacy Connectors (API Key / UID)

For legacy integrations, authentication is simpler: a plain API key or UID is required in the request. More straightforward but less secure.

### How to Tell If a Connector Is Generic or Legacy

**Option 1 (by heart):** Erik will provide a list. You learn this quickly in practice.

**Option 2 (in code):** Check the installer file for each connector under `lib/connectors/`. If the connector's `Installer` struct embeds/inherits from the **generic installer**, it is a generic connector. If it has its own custom installer functions, it is a legacy connector.

Examples from the session:
- **Generic connectors** (use generic installer): FSC Corporate (EDL), Dynamics Side Shop, Maxo, FSC Enterprise 12.1, Tribe, Web CRM 2
- **Legacy connectors** (have custom installer): FSC Enterprise (legacy), Dynamics (custom), Intermay Loyalty (custom)

---

## API Spec Versioning Rules

- If you add **new optional properties** to existing endpoints: **no version bump required**. CRM systems will discard/ignore unknown optional fields.
- If you **remove existing properties**: a new version is required.

---

## Webhook Spec: Justin-Side Endpoints

The `justin_generic_webhook` spec defines the endpoints on Justin's side that CRMs call. Almost all of them route to the **Delta Sync Manager** (the service that receives real-time external notifications).

Endpoints:
- **Record updates**: CRM sends CRM ID + action (`created`, `updated`, or `deleted`). Note: `created` and `updated` are handled identically today, but the distinction was preserved for potential future use. `deleted` is handled differently.
- **Consent changes**: CRM notifies Justin of consent state changes.
- **Promotions**: Lead-to-contact promotion notification.
- **Merge**: CRM notifies Justin that two duplicate contacts have been merged. Justin supports acting on merge requests.
- **Subscription creation/deletion** (proxy to Audience's metadata sync service).
- **Attribute creation/deletion** (proxy to Audience).

The webhook spec is also available on the same S3 bucket (`integration-files.apsis.one`).

---

## CI/CD Pipeline Architecture

### Two-Stack Reality

The integration service currently uses **two separate CI/CD tools** for different purposes:

| Purpose | Tool |
|---|---|
| PR checks (unit tests, coverage) | GitHub Actions (active) |
| Code quality (Sigrid / SonarCloud) | GitHub Actions (active) |
| Build & Deploy to environments | AWS CodePipeline (active) |
| Build & Deploy (alternative) | GitHub Actions `deploy.yml` (exists but **disabled**) |

### Why AWS CodePipeline Is Used for Deployment

When the team migrated to **Graviton (ARM64) architecture** for ECS tasks, they needed to build **ARM64 Docker images**. At the time of the migration, GitHub Actions had no ARM64 runners available. Emulating ARM64 on x86 runners caused build times to balloon:

> "Our deploy time went from around 20 minutes up to around 3 to 4 hours."

Since AWS CodePipeline already had ARM64 runners, and CodePipeline was already being used for the MA service, the team copied that flow to the integration environment. GitHub Actions was retained only for PR checks and Sigrid.

### Current State of GitHub Actions Deploy Workflow

A `deploy.yml` file **does exist** in the repository and has been tested with the new ARM64 GitHub runner. It is confirmed to work. The workflow is already configured for ARM64. However, it has **never been switched on** (it is disabled in GitHub Actions). The remaining work to fully migrate away from CodePipeline:

- Verify/update the step that uploads Swagger files to `docs.internal` (the script name reference may be slightly out of date — estimated ~30 minutes to fix).
- Remove the AWS Access Key / Secret Key setup steps (replace with OIDC — see below).
- Disable CodePipeline.
- Enable the `deploy.yml` GitHub Action.

[Erik Andersson]: "From the top of my head this would be like 2 days, 3 days maximum."

> **Action item / improvement:** Migrate deployment from AWS CodePipeline to GitHub Actions `deploy.yml`. This is a low-risk, bounded project.

### SonarCloud / Sigrid

- MA and E-com use **SonarCloud**.
- Integration also has SonarCloud configured (Erik was uncertain at time of session).
- **Sigrid** may be discontinued — Erik had just learned this.

---

## GitHub Actions: Authentication with AWS (OIDC vs. Access Keys)

### Legacy Problem

The team had historically used a manually created AWS IAM user with static access keys for the CodePipeline / GitHub Actions flows. These keys were **not auto-rotated** due to the complexity of propagating a new key to GitHub Actions secrets. Rotation required manual intervention.

### Current Solution: OIDC (OpenID Connect)

Implemented by **Shravidya**. AWS supports a flow where:
1. The GitHub Action workflow (identified by its webhook ID) is **whitelisted** in the AWS account as a trusted identity provider.
2. The workflow **assumes an IAM role** using its webhook ID, generating a short-lived access token.
3. No static AWS access key or secret key is stored or referenced.

This is already implemented in the **PR check** GitHub Actions workflow — that workflow has no AWS secret references at all, it just assumes a role via OIDC.

In the `deploy.yml`, the equivalent OIDC setup step would replace the current static key setup block entirely.

The OIDC mechanism is referred to as **OIDC** in the workflow YAML.

### GitHub Personal Access Token (for Private Packages)

The deploy workflow references a `PROVISIONED_GITHUB_TOKEN`. This is a GitHub token attached to a **team CICD user** (a service account within the GitHub organization). It is required because the integration service depends on the **email validation package**, which lives in a private repository. This token grants the build process access to that private package during `go build`.

---

## Manual Deployment Workflow: Deploying a Single Microservice to Staging

### Prerequisites

- AWS credentials for the target environment configured locally as an **AWS profile** (named according to the convention expected by the make scripts — do not rename these profiles or scripts will fail to find the corresponding config files).
- The `aws-environments/` folder (found in the integration shared folder in **LastPass**) contains the PEM files, DB host, bastion host configurations referenced by the scripts.

### Profile Name Convention

The deploy scripts look up configuration from files named after the AWS profile. Specifically, when you run `make deploy` with `AWS_PROFILE=staging`, the script reads from `aws-environments/<staging>/` for things like:
- ECR host URL
- Bastion host address
- DB host
- PEM file location

If you deviate from the expected profile names, the commands will fail with "could not find a file with this name."

### Deployment Steps for a Single Service

Example: deploying a change to the **Mappings Manager** to staging.

```bash
# From the repo root, navigate to the specific service folder
cd apps/mappings-manager/

# Set AWS profile and run deploy
AWS_PROFILE=staging make deploy
```

This `make deploy` command (defined in the service's `Makefile`, which includes the shared `manager-funcs.mk`) will:
1. Deploy/update the ECR repository.
2. Build the Docker image (ARM64).
3. Push the Docker image to ECR.
4. Deploy the CloudFormation stack for the service (creates or updates the ECS task definition, IAM permissions, routing, etc.).

### Makefile Structure

- Each service has its own `Makefile` in its folder under `apps/`.
- Service-specific `Makefile`s include a shared `manager-funcs.mk` (or equivalent) from the root, which contains the common deploy/build/push logic.
- Service-specific parameters (port, service name, display name, stack name, target cluster, memory allocation, etc.) are defined as variables in the service's `Makefile` and passed as build parameters to Docker and CloudFormation scripts.
- This avoids duplicating logic — services only declare their parameters; shared logic handles the rest.

### Partial Deployment (CloudFormation Only)

If you only change infrastructure configuration (e.g., memory allocation in the CloudFormation template) and do **not** change any Go code:

```bash
AWS_PROFILE=staging make deploy-manager
```

This skips the Docker build/push and only redeploys the CloudFormation stack.

### Deployment Time

- Full service deployment (build + push + ECS re-registration): **~3–4 minutes** per service (ECS needs to start the new image, re-register routing, drain old resources).
- Full environment deployment (all services): **30–40 minutes** in worst case (currently rebuilds images separately for each environment: staging, beta, prod).

**Known improvement:** Image promotion (promote staging image to beta, beta image to prod) rather than rebuilding from source for each environment. Felix wanted this but it was not implemented due to time constraints.

### Database Access

Access to the RDS database goes through the **bastion host** using `sshutle` (a VPN-style SSH tunnel tool):
1. In one terminal: run the `sshuttle` command to set up the tunnel via the bastion host.
2. In another terminal: connect to the database via `psql` — traffic is redirected through the tunnel.

Alternatively, use the **AWS RDS Query Editor** in the AWS console. PEM files and connection details are in the **LastPass integration shared folder**.

---

## Which Connectors Are Generic vs. Legacy

Erik committed to providing a definitive list. In the meantime, the code-inspection method (checking installer inheritance) is the reliable way to determine this. Known examples from the session:

**Generic Connector (use generic installer):**
- FSC Corporate / EDL (E-deal)
- Dynamics Side Shop
- Maxo (discontinued)
- FSC Enterprise 12.1
- Tribe
- Web CRM 2

**Legacy Connectors (custom installer):**
- FSC Enterprise (legacy / 12.0)
- Dynamics (base)
- Intermay Loyalty

---

## Azure Environment for Dynamics OAuth Credentials

⚠️ **Warning — Unusual/Fragile Setup**

The Azure environment used for Dynamics CRM OAuth (the Azure AD application and its secrets) is an **old APSIS International AB Azure tenant** — predating the FSC acquisition of APSIS. This environment:

- Is **not owned or managed by FSC IT** (IT has wanted to discontinue it for a long time).
- Is under the "old APSIS international AB domain."
- Is currently **owned/controlled by Erik** (and the integration team by extension).
- Requires manual maintenance (removing access when people leave, etc.).

**Why it cannot simply be migrated:**

> "You can't move a secret to another environment. We would need to create a new secret, and that means every Dynamics customer would need to completely undo everything they have done and reinstall using the new secret — and that is not really an option."

Therefore this old Azure tenant is essentially kept alive solely to preserve the existing OAuth app secrets for all Dynamics customers. A dedicated KT session on this topic is planned (see next steps).

---

## Key Takeaways

1. **The Generic API Spec is the source of truth** for all generic connector integrations. It lives at `lib/connectors/generic/assets/` and is always in sync with the code because Go interfaces are generated from it. A public copy is hosted on `s3://integration-files.apsis.one` for partner access.

2. **Two spec files exist**: the outbound API spec (what Justin sends to CRMs) and the webhook spec (what CRMs send to Justin). Both are on the S3 bucket.

3. **HTTP semantics matter**: the generic connector standardized proper HTTP status codes. Legacy connectors (e.g., FSC Enterprise 12.0) used 200 for everything and buried errors in the response body — this is explicitly not supported in generic connector implementations.

4. **Consent is the only bidirectional data** in the sync model. Profile/contact attribute data is one-directional (CRM → APSIS). There is a `PATCH /consent` endpoint but no `PATCH /profile`.

5. **CI/CD uses two tools**: GitHub Actions for PR checks and code quality; AWS CodePipeline for deployments. A nearly complete migration back to GitHub Actions for deployments exists (`deploy.yml`) but is disabled. Estimated 2–3 days to complete the migration.

6. **The ARM64/Graviton migration** is the historical reason for the CodePipeline detour. GitHub Actions now has ARM64 runners and has been validated.

7. **AWS authentication uses OIDC** (not static keys) — implemented by Shravidya. The PR check workflow already uses this. The deploy workflow still needs to be updated to use OIDC before it can go live.

8. **Deploying a single service** is straightforward: `cd apps/<service-name> && AWS_PROFILE=<env> make deploy`. AWS profile names must match the convention expected by the scripts.

9. **The Azure tenant for Dynamics OAuth is fragile** and not owned by IT. It cannot be migrated without forcing all Dynamics customers to reinstall. The team must maintain it manually.

10. **Several features are unused** (e-commerce endpoints, consent-based subscription sync, attribute proxy) — primarily because they were built for Maxo which was discontinued. The e-commerce endpoints are candidates for removal.

---

## Unresolved Questions and Action Items

| # | Item | Owner | Notes |
|---|---|---|---|
| 1 | Migrate deployment from AWS CodePipeline to GitHub Actions `deploy.yml` | Incoming team | ~2–3 days effort. Update Swagger upload script reference, replace key setup with OIDC, disable CodePipeline, enable GitHub Action. |
| 2 | Implement image promotion in deploy pipeline (staging → beta → prod) | Incoming team (future) | Currently rebuilds from source for each env; promotion would cut deploy times significantly. |
| 3 | Erik to provide definitive list of which connectors are generic vs. legacy | Erik Andersson | Promised during session. |
| 4 | Remove e-commerce endpoints from generic connector spec | Erik Andersson (before handover) | No customers use them; all current integrations are CRM systems. |
| 5 | Michal and Lukasz to set up staging database connection via bastion/sshuttle | Lukasz Grabowski, Michal Rosikiewicz | As homework; Tomasz already has this working. |
| 6 | Dedicated KT session on Dynamics Azure environment / Entra setup | Erik Andersson (to prepare) | Covers the old APSIS International AB Azure tenant, subscription details, and access handover. |
| 7 | KT session on Squid Proxy | Erik Andersson (needs slides) | Scheduled for next week. |
| 8 | KT session on Lead Creation flow | Erik Andersson (needs slides) | Scheduled for next week. |
| 9 | Confirm whether SonarCloud is already configured on the integration repo | Erik Andersson | Was uncertain during session. |
| 10 | ⚠️ AMBIGUOUS: The exact S3 URL / direct link to the spec files was demonstrated live but not captured in the transcript text. | — | The bucket is `integration-files.apsis.one`; exact object paths for `generic_api_spec.yaml` and `justin_generic_webhook.yaml` were shown in screen share but not verbalized clearly. |