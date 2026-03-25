---
source_file: Erik - Justin data recovery.txt
domain: Apsis One Integrations
topics: [Disaster Recovery Procedures, Secrets Manager Configuration, Database Restoration, KMS Encryption, Parameter Store Setup, CloudFormation Deployment, Infrastructure as Code, Multi-Region Backup Strategy]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Secrets Manager, Parameter Store, CloudFormation, RDS, Lambda, ECR, KMS, SNS, Redis, Kafka, SQS, Generic Connector, Database Snapshots, AWS accounts]
session_type: knowledge-transfer
subdomains: []
---

## Session Overview

This knowledge transfer session covers the complete disaster recovery procedure for the Apsis One Integrations domain. Erik Andersson walks through the manual and automated configuration steps required to restore the integration infrastructure from scratch using an existing database backup. The discussion covers Secrets Manager setup, parameter store configuration, CloudFormation deployment orchestration, and critically identifies a significant weakness in the current KMS key architecture that prevents credential decryption in backup accounts. The session emphasizes that while infrastructure restoration typically takes 35-40 minutes (excluding database setup time), the biggest bottleneck is waiting for dependent services like Audience to become operational.

---

## Disaster Recovery Architecture Overview

### Two-Tier Configuration System

The disaster recovery process relies on two distinct configuration layers:

1. **Secrets Manager** - Contains sensitive credentials that must be manually created
2. **Parameter Store** - Contains shared, non-sensitive environment variables reused across CloudFormation files

[Erik Andersson]: The reason for this separation is that we have CloudFormation files for most infrastructure, but certain secrets cannot be defined in code because their values are sensitive. The Parameter Store approach was implemented as a refactoring improvement over the legacy system where each service had its own configuration file, requiring values to be duplicated across 15+ different folders.

---

## Manual Secrets Manager Configuration

### Required Secrets for Integration Deployment

The following secrets must be manually created in Secrets Manager before deployment:

- **Microsoft Dynamics OAuth Client ID** - This is the most critical secret
- **Database key** - Required for database operations
- Additional service-specific credentials

### Locating the Dynamics OAuth Client ID

[Erik Andersson]: The Dynamics OAuth client ID must be retrieved from the **Microsoft Dynamics or Azure page for the Apsis International AB account** (not the FSC account). Navigate to the registered enterprise applications section to find this value.

The challenge is knowing where to retrieve each secret from. The deployment will fail and explicitly indicate which secret is missing, but the burden is on the person performing recovery to locate the source of each credential.

### Improvement Opportunity

[Erik Andersson]: While we cannot store secret values in CloudFormation files, we could define the secrets structure in code without values. This would eliminate the trial-and-error deployment cycle where each failed deployment reveals the next missing secret. Currently, the process looks like: deploy → encounter error → deploy → encounter second error, etc.

---

## Parameter Store and Environment Variables Configuration

### Purpose and Scope

The Parameter Store contains configuration values reused across nearly all CloudFormation templates and services. Examples include:

- Audience account URLs and endpoints
- CRM system API URLs
- Shared service endpoints
- Request routing configuration

[Erik Andersson]: Almost every worker and manager service depends on these shared values. Rather than hardcoding them in each service's CloudFormation file (the legacy approach), we centralize them in Parameter Store.

### File Naming Convention

Environment variable files must be named according to the AWS profile used for deployment:

```
nv-<AVS_PROFILE>.conf
```

For example:
- If deploying to the disaster recovery account with profile `dr`, create: `nv-dr.conf`
- If deploying to `prod` account, use: `nv-prod.conf`

The file is located in the cloud formation directory under `environment variables`.

### Disaster Recovery Challenge

[Erik Andersson]: The most annoying part of disaster recovery is hunting down the correct URLs for all these services from different teams. In the previous configuration system, this was "20 times worse" because duplicated values had to be entered manually in 15+ different locations.

### Required Configuration Values in nv-*.conf

Each environment variable file must specify:
- Audience account URLs
- CRM system endpoints
- API gateway URLs
- Service discovery endpoints
- Any other shared infrastructure addresses

---

## AVS Environment Configuration

### Purpose

The AVS (?) environment files are utilized by:
- Make file commands during deployment
- Docker image push operations to ECR

### ECR Host Configuration - Critical for Docker Operations

The **ECR host** parameter is mandatory for Docker image operations, even if other values in the file are optional.

Example ECR host format for disaster recovery account (account ID: 123, region: eu-central):
```
123.dkr.ecr.eu-central-1.amazonaws.com
```

### Optional Configuration Values

The following values are optional for deployment but may be used for specific operations:
- Database connection parameters (only used by `make connect-database` command)
- API token generation settings (legacy, rarely used now)

---

## CloudFormation Deployment Process and Orchestration

### Overall Deployment Strategy

Deployment follows a sequential layering approach triggered by a single command in the root folder:

```bash
make deploy AVS_PROFILE=dr
```

### Deployment Sequence

#### Phase 1: Base Infrastructure Template
Deploys shared resources used across all services:
- ECR repositories for all services
- Lambda deployment buckets
- IAM policies reused by multiple services
- SNS topics for alarms
- Other shared infrastructure

#### Phase 2: Database Template
Sets up RDS infrastructure:
- Creates RDS cluster and instances
- Establishes security groups
- **This phase takes the longest** - typically 30+ minutes

[Erik Andersson]: When restoring from a snapshot during disaster recovery, the database template CloudFormation file uses an RDS parameter that points to the snapshot in the backup account. Before deployment, populate this parameter with the snapshot identifier.

#### Phase 3: Supporting Infrastructure
Deployed as part of the main CloudFormation stack:
- Squid proxy
- Kafka (utilized by outbound worker)
- Redis cache
- All SQS queues used by services

#### Phase 4: Microservice Deployment
All integration services deployed in parallel batches (4-5 services per batch):

Services are split into three categories:
- **Manager services** - Coordination and state management
- **Worker services** - Processing and integration logic
- **Asynchronous sync services** - Async batch operations

### Service Independence Architecture

[Erik Andersson]: Significant effort was taken to ensure each service can be deployed independently. All queues are deployed separately in the base phase, not alongside their consumer services. This prevents circular dependencies.

Example of circular dependency avoided:
- The Sluice Worker service adds messages to the Delta Sync Worker queue
- Without decoupling, you couldn't deploy Sluice Worker before Delta Sync Worker, and couldn't deploy Delta Sync Manager before Sluice Worker
- This would make parallel deployment and template deletion impossible

### Parallel Deployment Mechanism

The make file executes parallel deployments using bash logic:
```
Deploy 5 services in parallel && await batch completion && deploy next 5
```

[Erik Andersson]: While the bash command syntax may appear intimidating, it simply means: deploy five services simultaneously and wait for all five to complete before proceeding to the next batch.

### Typical Deployment Timeline

- Database creation: ~30+ minutes (longest phase)
- Redis setup: ~30 minutes (first-time setup only)
- Service deployments: ~5 minutes per batch with parallel execution
- **Total integration deployment (excluding database):** 35-40 minutes

---

## Disaster Recovery Workflow Summary

### Three-Step Setup Process

**Step 1: Recreate Secrets Manually**
- Manually create required secrets in Secrets Manager
- Consult the Dynamics OAuth retrieval instructions
- Expect deployment failures to identify additional missing secrets

**Step 2: Create Environment Variables Configuration File**
- Create `nv-<ENVIRONMENT_NAME>.conf` in the CloudFormation directory
- Populate with all shared service URLs and endpoints
- File name must match the AWS profile name exactly

**Step 3: Create AVS Environment File**
- Create `.env.<PROFILE_NAME>` file
- Add ECR host parameter (mandatory): `ECR_HOST=<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com`
- Add optional database connection parameters if needed

**Step 4: Deploy**
```bash
make deploy AVS_PROFILE=<PROFILE_NAME>
```

---

## Critical KMS Encryption Limitation - Multi-Region Key Issue

### The Problem

The integration uses KMS encryption for customer API credentials in the generic connector:

[Erik Andersson]: When a customer installs a generic connector integration, they provide API credentials. These credentials are encrypted using KMS before storage in the database. Developers never see the plaintext credentials, unlike legacy connectors where credentials are stored without encryption.

When the broker service makes requests to customer CRM systems, it decrypts the credentials and adds them to the request headers.

### Current Implementation Flaw

**The KMS key is configured for single-region only.**

This means:
- The key exists only in the primary region
- The key is **not shared with backup accounts**
- Encrypted credentials cannot be decrypted in the disaster recovery account
- Data restoration to the backup account will succeed, but generic connector integrations cannot be verified

### Impact Assessment

[Erik Andersson]: This is not an outright data loss situation. The issue can be solved through reinstallation of integrations (customers re-entering credentials), but it is "very, very suboptimal."

Legacy connectors (Efficy Enterprise 12.0) do not suffer from this limitation because they store credentials without hashing/encryption, so they can be verified in backup accounts.

### Root Cause

The KMS key was created with single-region configuration. **AWS does not allow changing an existing key from single-region to multi-region.** A completely new key must be created with multi-region enabled from inception.

### Migration Path (Not Yet Implemented)

To fix this:
1. Create a new multi-region KMS key
2. Replicate it to the backup account
3. **Modify all encrypted credentials in the database to use the new key** (this step has not been prioritized)
4. Deprecate the old single-region key

[Erik Andersson]: This has not been prioritized because while recognized as a weakness, it was not considered a complete data loss scenario. However, it should be addressed before the next disaster recovery exercise.

### Verification Limitation on Backup Account

During disaster recovery:
- ✅ Generic connector integrations can be restored (data)
- ❌ Generic connector integrations **cannot be verified** (credentials can't be decrypted)
- ✅ Legacy connectors (Enterprise 12.0) **can be verified** without issues

---

## Comparison with M8 Disaster Recovery

[Michal Rosikiewicz]: M8 has a similar setup requiring manual Secrets Manager and Parameter Store configuration.

Key difference: M8 does not have the KMS key limitation because it does not encrypt customer credentials using the same mechanism.

[Erik Andersson]: M8 also has database backup and snapshot configuration in the exact same fashion as integration. However, M8 does not hash the data in the same way, so the KMS issue does not apply.

---

## Known Challenges and Unknowns

### Primary Bottleneck: Dependent Service Availability

[Erik Andersson]: The biggest challenge in disaster recovery is not infrastructure restoration—that reliably succeeds. The challenge is verifying that the entire workflow functions correctly.

Integration is heavily dependent on **Audience** service:
- Needs to retrieve profile attributes for nearly every operation
- Needs to update profiles for nearly every operation
- Needs to update consent settings for nearly every operation

Multiple fragile steps in the workflow require Audience to be operational. Even if integration infrastructure is fully restored, service verification cannot complete until Audience is running.

### Undiscovered Interdependencies

[Erik Andersson]: There are almost certainly undiscovered interdependencies that have been overlooked because the disaster recovery account has not been set up from scratch in approximately 2 years.

These will surface during the recovery process, typically as error messages like:
- "Cannot find this specific secret"
- "Dependent on this resource from this service"

[Erik Andersson]: This requires trial and error, but the errors are usually clear. The recommendation is to get the database deployed first (potentially using `make deploy-db` in isolation) and complete that phase before attempting service deployments, to avoid sitting with database deployment as a hold-up point after 40+ minutes.

### Historical Precedent

[Tomasz Kowalski/Others]: In previous disaster recovery exercises, the team waited for multiple dependent services including Audience. Without credentials/delegation from dependent teams, integration services cannot function regardless of local infrastructure status.

---

## Recommendations and Action Items

### Immediate (Before Next Disaster Recovery Exercise)

1. Document the exact secrets required and their source locations
2. Create CloudFormation secret definitions without values (reduces trial-and-error)
3. Establish a standard procedure document for parameter store URL discovery

### Medium-term (Should Be Addressed)

1. **Migrate to multi-region KMS key** for generic connector credential encryption
   - Create new multi-region key
   - Update all database records to reference new key
   - Replicate key to backup accounts
2. Conduct a full disaster recovery simulation to discover undocumented interdependencies before they're needed in actual crisis

### Long-term (Infrastructure Improvement)

[Erik Andersson]: Consider automation of parameter store configuration discovery, potentially by querying dependent services for their current endpoint URLs rather than manual entry.

---

## Key Takeaways

1. **Three critical setup steps required:** (1) Secrets Manager secrets, (2) Environment variables file, (3) AVS environment file with ECR host
2. **Secrets must be manually created** - No CloudFormation exists for secrets, but error messages will identify missing ones
3. **File naming matters** - Environment file names must exactly match AWS profile names
4. **Service independence was intentional** - Parallel deployment of 4-5 services is safe due to deliberate architectural decoupling
5. **Database is the time bottleneck** - Plan for 30+ minutes; can deploy in isolation first to avoid late-stage blocking
6. **KMS key is a known weakness** - Cannot decrypt generic connector credentials in backup account; single-region key cannot be converted to multi-region
7. **Audience dependency is the verification bottleneck** - Infrastructure may be ready in 35-40 minutes, but cannot verify functionality until Audience is operational
8. **Refactoring to Parameter Store was significant improvement** - Eliminated duplicating values across 15+ configuration files (legacy approach was "20 times worse")
9. **Undiscovered dependencies likely exist** - 2-year gap since last account setup; expect surprise interdependencies to surface as error messages
10. **Legacy connectors bypass KMS limitation** - Enterprise 12.0 integrations can be verified in backup account even if generic connectors cannot be

---

## Unresolved Questions / Follow-ups

1. **When will the multi-region KMS key migration be scheduled?** - Currently deprioritized despite known risk
2. **What is the complete definitive list of all Secrets Manager secrets required?** - Currently discovered through trial-and-error deployment failures
3. **How are dependent services (Audience, etc.) prioritized for disaster recovery?** - Historical precedent suggests integration must wait for them
4. **What specific undocumented interdependencies exist?** - Unknown until next full recovery simulation; likely to emerge as cryptic error messages
5. **Is there a documented procedure for discovering current parameter store values?** - Currently requires "hunting down teams" to obtain URLs