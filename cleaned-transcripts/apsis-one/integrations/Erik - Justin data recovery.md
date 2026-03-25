---
source_file: Erik - Justin data recovery.txt
domain: Apsis One Integrations
topics: [Disaster Recovery Procedures, Secrets Management, Database Restoration, Infrastructure Deployment, Configuration Management, KMS Encryption, Multi-Region Architecture]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Secrets Manager, Parameter Store, CloudFormation, RDS, Lambda, ECR, KMS Keys, SNS Topics, Redis, Kafka, SQuid Proxy, Generic Connector, Database Snapshots]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector]
---

## Session Overview

This session covers the complete disaster recovery (DR) and data recovery procedures for the Apsis One Integrations platform. The discussion focuses on the manual and automated steps required to restore the entire integration infrastructure from scratch using existing database backups, including secrets configuration, parameter store setup, CloudFormation deployment orchestration, and critical infrastructure dependencies. A significant weakness around KMS key multi-region configuration for generic connector credentials is identified as a known limitation that should be prioritized for future fixes.

---

## Disaster Recovery Overview and High-Level Process

The disaster recovery process for integrations is described as relatively straightforward but requires significant manual work, particularly around secrets management. The entire procedure can be summarized into three main steps:

1. Recreate secrets manually in Secrets Manager
2. Create environment variable configuration files for parameter store population
3. Create AWS environment files for build tooling, then execute the full deployment

[Erik Andersson]: > "To set up integration completely from scratch with an existing database recovery, there's some preparational work that you need to do. But otherwise it should just be one command that you run."

The entire deployment process, from initial infrastructure to all microservices running, typically takes approximately 35-40 minutes (excluding database creation time, which takes roughly 30 minutes, and Redis setup, which adds another 30 minutes).

---

## Secrets Manager Configuration

### Manual Secrets Recreation

The first critical step in disaster recovery is manually recreating secrets in AWS Secrets Manager. These secrets cannot be deployed via CloudFormation templates and must be created by hand before deployment begins.

**Key secrets that require manual creation:**

- **Dynamics OAuth Client ID** — The most important secret to identify and configure correctly. This credential must be retrieved from the Microsoft Dynamics or Azure portal for the **Apsis International AB account** (not the FSC account). Specifically, you must navigate to registered enterprise apps within the Azure portal.

[Erik Andersson]: The deployment process will fail and explicitly tell you which secrets are missing, so you can use deployment failures as a checklist for missing credentials.

### Improvement Opportunity

[Erik Andersson]: > "What we could do as an improvement is that you define the secrets in a CloudFormation file, but obviously you don't enter the values. Doing that you would still know exactly which secrets you need to fill in, instead of having to do trial and error by deploying, encountering error, deploying, encountering second error, etcetera."

This would eliminate discovery of missing secrets through repeated deployment failures.

---

## Parameter Store and Environment Variable Configuration

### Two-Tier Configuration Architecture

The platform uses two separate configuration sources:

1. **Secrets Manager** — For sensitive credentials (API keys, OAuth tokens)
2. **Parameter Store** — For non-sensitive configuration values shared across services

### Parameter Store Values

Parameter store contains values that are reused across nearly all CloudFormation files and services, such as:

- Target audience account identifiers
- URLs for CRM system APIs
- Service endpoint addresses
- Other environment-specific configuration

[Erik Andersson]: > "A lot of our cloud formation files reuse the same environment variables. For example, almost every worker and manager we have are using values like which audience account should I make requests to when I request folders or stuff, or what is the URL to the one API that we send over to the CRM systems."

### Creating Environment Variable Configuration Files

Before deployment, you must create a configuration file under the environment variables directory following a specific naming convention:

```
env/nv-{AVS_PROFILE}.conf
```

For example, if deploying to a disaster recovery account named `dr`, create:

```
env/nv-dr.conf
```

The `AVS_PROFILE` environment variable (used in the make system) must match the configuration filename exactly.

**Configuration file contents:**

The file specifies all URLs and endpoints for dependent services. Example structure:

```
# Required parameter store values:
AUDIENCE_API_ENDPOINT=...
CRM_SYSTEM_URL=...
[other service endpoints]
```

[Erik Andersson]: > "This is the most annoying part during the disaster recovery day — finding out what is the URL for all of these services. You will need to hunt teams down to get them to provide these URLs."

### Historical Configuration Complexity

[Erik Andersson]: > "With the previous way of us handling the configuration in integration, this was 20 times worse because each service had its own configuration file. You had to duplicate the values in like 15 different folders. It was an absolute nightmare, so I refactored all of that. We are reusing the values from the parameter store now instead."

This refactoring significantly reduced the disaster recovery burden by centralizing configuration values.

---

## AWS Environment Files for Build Tooling

### Purpose and Naming Convention

AWS environment files are used by make commands and, critically, by the Docker image push process to ECR (Elastic Container Registry). These files follow the same naming convention as parameter store configuration:

```
.env.{AVS_PROFILE}
```

For a disaster recovery profile named `dr`:

```
.env.dr
```

### Required Values

The most critical value in the environment file is:

```
ECR_HOST=<account_id>.dkr.ecr.<region>.amazonaws.com
```

For example, if the disaster recovery account ID is `123456789012` and the region is `eu-central-1`:

```
ECR_HOST=123456789012.dkr.ecr.eu-central-1.amazonaws.com
```

This value is **essential** for Docker image deployment to work correctly.

### Optional Values

Other values in the environment file (like database connection details or API token generation settings) are only used by specific make commands:

- Database connection values are only used by `make connect-to-database`
- API token generation settings are only used for local testing scenarios

These are not required for deployment to function.

---

## Database Restoration and RDS Configuration

### Daily Backups and Snapshot Strategy

Database backups are handled by cloud engineering, not the integration team:

- **Daily backups** occur in the production account (typically at 2:00 AM)
- **Cloud engineering** maintains additional backups in a dedicated backup account
- Backups are automated; integration team focuses on restoration procedures

### Disaster Recovery Account Database Restoration

During a disaster recovery exercise, cloud engineering copies database snapshots to the disaster recovery account. To restore from a specific snapshot:

1. Identify the snapshot name in the disaster recovery account
2. Add the snapshot identifier to a **parameter store variable** before deploying the database CloudFormation template
3. Specify this parameter in the database deployment configuration file

[Erik Andersson]: > "If you want to restore from a database or from a snapshot, then you fill in this parameter store variable. The snapshot would exist in the account, and before you deploy the database template, you add this in the configuration file."

The database template reads this variable during deployment and restores from the specified snapshot instead of creating a new empty database.

### RDS Configuration Details

The integration platform uses **RDS** (AWS Relational Database Service) for the main database. The choice between databases, regions, and other RDS parameters is specified in the CloudFormation template itself.

---

## CloudFormation Deployment Orchestration

### Deployment Command

The entire integration infrastructure is deployed with a single command executed from the root directory:

```bash
make deploy AVS_PROFILE=dr
```

Where `dr` is the target environment (or whatever name you've chosen for the disaster recovery account).

### Deployment Execution Order and Parallelization

The make system orchestrates deployment in a specific sequence:

#### Phase 1: Base Infrastructure
Deploys foundational resources needed by all services:

```
base-template:
  - ECR repositories
  - Lambda deployment buckets
  - Shared IAM policies
  - SNS topics for alarms
```

All base infrastructure is shared across services.

#### Phase 2: Shared Services and Infrastructure
Deploys infrastructure required by microservices:

```
database-template:
  - RDS cluster and instances
  - Security groups
  - (Takes longest: ~30 minutes)

kafka-squid-redis-queues:
  - Kafka clusters (for outbound worker)
  - Squid proxy
  - Redis cache
  - All SQS queues used by services
```

#### Phase 3: Microservices (Parallel Deployment)

All microservices are deployed in **parallel batches of 4-5 at a time**:

```
manager-services:
  - Delta Sync Manager
  - [other managers]

worker-services:
  - Delta Sync Worker
  - Outbound Worker
  - [other workers]

async-sync-services:
  - [async services]
```

### Design Rationale: Service Decoupling

[Erik Andersson]: > "We took great efforts to make sure that every service in integration should be able to be deployed independently of each other. That's why we deploy all the queues separately in the base deployments."

**Why this matters:** If the Delta Sync Worker service and its queue were deployed together, you couldn't deploy the Sluice Worker (which depends on the Delta Sync Worker queue) before deploying the Delta Sync Worker itself. This creates circular dependencies. By decoupling queue creation from service deployment, all services can be deployed in parallel.

### Deployment Time Estimates

Based on actual deployment experience:

- **Base templates:** Relatively quick
- **Database creation:** ~30 minutes (longest single component)
- **Redis setup:** ~30 minutes (first time)
- **All microservices (parallel):** ~5-10 minutes
- **Total deployment time (excluding database/Redis):** 35-40 minutes

---

## Critical Infrastructure Dependencies and Testing Challenges

### Dependency Chain for Integration Verification

While infrastructure deployment typically succeeds without major issues, verifying that the complete integration flow works is significantly more complex due to external dependencies.

[Erik Andersson]: > "We tend to have a lot of time or a lot of issues verifying that the whole flow works because we are fully dependent on Audience being up and running in order for us to actually be able to have successful requests."

**Critical external dependencies:**

1. **Audience service** — Required for:
   - Retrieving customer profile attributes
   - Updating customer profiles
   - Retrieving and updating consent data

2. **Profile data retrieval** — Required before any data transformation or synchronization
3. **CRM system availability** — Required to complete end-to-end flows

[Erik Andersson]: > "We need to retrieve the attributes in essentially everything we do. We need to retrieve profile data for anything we do. We need to be able to update profiles for anything we do. We need to be able to update consents in everything we do. So there are a lot of fragile steps there."

**Deployment Success vs. Flow Verification:** The integration infrastructure itself typically deploys successfully. However, end-to-end testing of actual data flows often fails due to these external dependencies not being available or properly configured during disaster recovery exercises. Infrastructure verification (services running, queues created, database accessible) is straightforward; business logic verification is difficult.

### Audience Service Deployment Time

A particular challenge in disaster recovery exercises is that **Audience service takes an extremely long time to deploy**. This creates a significant bottleneck:

- Integration infrastructure may be ready in 1-2 hours
- Audience service may take many additional hours
- Cannot fully test integration flows until Audience is operational
- Team members may give up and leave if sitting idle waiting for dependencies

---

## Critical KMS Key Limitation for Generic Connector Credentials

### The Problem

A significant architectural weakness has been identified in how generic connector credentials are encrypted and backed up:

**Single-Region KMS Key Configuration:**

The platform uses a **KMS (Key Management Service) key to encrypt customer API credentials** when they install a generic connector integration. This key was configured as **single-region only** at creation time.

[Erik Andersson]: > "The KMS key that we are using is only set up to be single region. That means that the key is not shared with the backup accounts. Meaning that we would not be able to decrypt the customer's API keys if you were to restore this to another account."

### Why This Matters

When a customer installs a generic connector:

1. Customer provides their API credentials for their CRM system
2. These credentials are **encrypted using the KMS key** and stored in the database
3. When making requests to the customer's CRM, the broker service **decrypts the credentials** using the KMS key and adds them to the request header

The encryption protects sensitive credentials from being readable by developers or logs.

### The Disaster Recovery Impact

If you restore the integration database to a disaster recovery account:

- ✅ Database is restored with all customer data
- ✅ Encrypted credential values are present in the database
- ❌ **KMS key is not available in the DR account** (single-region)
- ❌ Cannot decrypt customer credentials
- ❌ Generic connector integrations cannot be tested or verified
- ❌ Broker service cannot retrieve credentials for API calls

### Workaround and Non-Impact Assessment

[Erik Andersson]: > "This can be solved with a reinstallation. We absolutely do not want to end up in this situation, but it would not be an outright data loss situation."

**Workaround:** If a full restoration occurs, customers would need to reinstall their generic connector integrations (re-enter credentials). This is annoying but not catastrophic.

**Non-Impact on Legacy Connectors:** This issue only affects the generic connector. Legacy connectors (Enterprise 12.0, Enterprise 12.1, etc.) store credentials without KMS encryption, so they can be verified on the DR account without issues.

### The Fix (Not Yet Implemented)

To solve this permanently:

1. **Create a new KMS key** configured as **multi-region** from the start
2. **KMS keys cannot be converted** from single-region to multi-region — a completely new key must be created
3. **Re-encrypt all existing encrypted credentials** in the database to use the new multi-region key
4. **Replicate the key** to the backup account (automatic with multi-region keys)
5. **Ensure database snapshots** replicate the new key's availability

[Erik Andersson]: > "You need to create a completely new key and set it to multiregion from the start. But that means that all of the keys in our database needs to be modified to utilize the new key instead of the old one."

### Current Status

[Erik Andersson]: > "This is just something that has never really been prioritized because while we know this is a weakness, it has not been considered a full data loss in case of us having to do a complete restoration."

**Current state:**

- ⚠️ **Known weakness** — team is aware of the issue
- ❌ **Not yet fixed** — requires re-encryption effort
- ❌ **Not prioritized** — viewed as annoying but not critical data loss
- ✅ **Should be fixed** — long-term recommendation

---

## Secrets Manager and Parameter Store Comparison with M8 Platform

[Michal Rosikiewicz]: The M8 platform handles disaster recovery similarly, requiring manual secrets and parameter store setup before deployment.

**M8 Platform Differences:**

- M8 does have daily database backups and disaster recovery snapshot configurations similar to integrations
- M8 **does not use generic connector credentials** and therefore **does not suffer from the KMS key issue**
- M8 has database encryption, but this is different from the KMS hashing of generic connector credentials

[Erik Andersson]: > "MA also has database backup and needs to be configured in the exact same fashion. However, MA does not suffer from the same KMS issue because we don't hash the data like we do for the generic connector credentials."

The KMS encryption issue is **unique to integrations** due to the generic connector architecture.

---

## Known Unknowns and Future Challenges

### Undiscovered Dependencies

[Erik Andersson]: > "I can also guarantee you that there are some new fun interdependencies that we have overlooked because we have not set up the account from scratch now for I think 2 years."

The platform has not undergone a complete rebuild from scratch in approximately 2 years. This guarantees that:

- Some dependency linkages between services may be missing from documentation
- Some manual configuration steps may have been forgotten
- New features added over time may have dependencies not reflected in disaster recovery procedures

**How to identify:** Missing dependencies surface as clear error messages during deployment: `"I cannot find this specific secret"` or `"I am dependent on this resource from this service."`

### Database Deployment Caution

[Erik Andersson]: > "You don't want to happen is to sit and have the database deployment being held back after 40 minutes because then you just close the computer and go home and rethink your life."

**Recommendation:** Consider deploying the database first in isolation using `make deploy DB` to verify database creation succeeds before attempting full infrastructure deployment. This prevents waiting 40+ minutes only to discover the database deployment fails.

---

## Step-by-Step Disaster Recovery Checklist

### Step 1: Recreate Secrets in Secrets Manager

Manually create all required secrets in AWS Secrets Manager. Critical secret:

```
dynamicso-f-client-id: <retrieved from Apsis International AB Azure account>
```

Deployment will fail and indicate which secrets are missing if any are forgotten.

### Step 2: Create Environment Variable Configuration File

Create configuration file:

```
env/nv-{ENVIRONMENT_NAME}.conf
```

Example for disaster recovery account named `dr`:

```
env/nv-dr.conf
```

Populate with all required service endpoints and configuration values. The environment name in the filename must match the `AVS_PROFILE` value used in make commands.

### Step 3: Create AWS Environment File for Build Tooling

Create file:

```
.env.{ENVIRONMENT_NAME}
```

Example for `dr` environment:

```
.env.dr
```

Populate with ECR host information:

```
ECR_HOST=<account_id>.dkr.ecr.<region>.amazonaws.com
```

### Step 4: Execute Deployment

From the root directory:

```bash
make deploy AVS_PROFILE=dr
```

This will:

1. Deploy base infrastructure (ECR, buckets, policies, SNS topics)
2. Deploy shared services (RDS, Kafka, Redis, Squid proxy)
3. Deploy all microservices in parallel batches
4. Complete in approximately 35-40 minutes (plus database/Redis setup time)

---

## Key Takeaways

1. **Disaster recovery is mostly automated** but requires manual setup of three configuration areas: Secrets Manager secrets, environment variable configuration files, and AWS environment files for build tooling.

2. **KMS key single-region limitation is a known architectural weakness** for generic connector credentials that should be addressed by creating a new multi-region KMS key and re-encrypting existing data. Until fixed, generic connector integrations cannot be verified on disaster recovery accounts, though this is not a data loss situation.

3. **Service decoupling by queue separation** allows parallel deployment of microservices (4-5 at a time), dramatically reducing deployment time from what would otherwise require sequential dependencies.

4. **External service dependencies (particularly Audience) are the critical bottleneck** for verifying complete integration flows during disaster recovery exercises, not infrastructure deployment itself.

5. **Expect undocumented dependencies** when performing the first complete account rebuild in several years. These surface as explicit error messages during deployment and can be resolved by gathering information from dependent teams.

6. **Parameter store configuration consolidation** (refactored from per-service files to centralized values) significantly reduced disaster recovery complexity compared to legacy approaches.

---

## Unresolved Questions and Action Items

### For Future Disaster Recovery Exercise:

1. **Validate all current service dependencies** — No complete rebuild has occurred in ~2 years; undocumented dependencies likely exist and should be discovered during actual exercise.

2. **KMS key multi-region migration** — Should be prioritized for implementation:
   - Create new multi-region KMS key
   - Re-encrypt all existing generic connector credentials in database
   - Replicate key to backup account
   - Update disaster recovery procedures to account for new key

3. **Secrets documentation** — Create comprehensive documentation of all required Secrets Manager secrets, their purposes, and where to retrieve them (currently only Dynamics OAuth Client ID is documented).

4. **Endpoint discovery process** — Formalize the process for gathering all required service endpoint URLs from dependent teams before a disaster recovery exercise begins.

5. **Timeline expectations** — Set realistic expectations that full disaster recovery exercises will likely take 2-3 days or more if waiting for external services (particularly Audience) to complete deployment.