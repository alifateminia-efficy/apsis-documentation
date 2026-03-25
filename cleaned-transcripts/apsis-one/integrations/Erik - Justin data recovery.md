---
source_file: Erik - Justin data recovery.txt
domain: Apsis One Integrations
topics: [Disaster Recovery Process, Secrets Management, Configuration Management, Infrastructure Deployment, Database Recovery, KMS Encryption, Multi-region Setup]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Secrets Manager, Parameter Store, CloudFormation, RDS, AWS KMS, ECR, Lambda, SNS, Redis, Kafka, SQS, Docker Images, Generic Connector, Legacy Connectors]
session_type: knowledge-transfer
subdomains: [Architecture, Microsoft Dynamics Integration]
---

## Session Overview

This knowledge transfer session covers the **disaster recovery (DR) procedures for the Apsis One Integrations platform**. Erik Andersson walks through the manual and automated steps required to restore the complete integration infrastructure from scratch in a disaster recovery account, using an existing database recovery. The session emphasizes the critical distinction between secrets (which must be manually created) and parameters (which can be automated via configuration files), highlights the deployment strategy using parallel service deployments, and identifies a significant limitation with the current KMS key setup for generic connector credential encryption that affects DR readiness.

---

## Disaster Recovery Architecture Overview

### Two-Tier Configuration Approach

The disaster recovery setup relies on two distinct configuration layers:

1. **Secrets Manager** (manual, per-secret)
   - Stores sensitive credentials that services depend on
   - Cannot be automated via CloudFormation
   - Must be created by hand before deployment

2. **Parameter Store** (automated via configuration files)
   - Contains shared environment variables reused across CloudFormation files
   - Populated by environment-specific configuration files
   - Enables parallel service deployment without interdependencies

[Erik Andersson]: > "Most importantly, we have the dynamics OAuth client ID. This needs to be added manually and the big thing here is knowing where you retrieve this secret from."

### Database Backup Strategy

[Erik Andersson]: The backup process is handled by Cloud Engineering, not the integration team:

- Daily backups are created in the production account at approximately 2:00 AM
- Cloud Engineering maintains a separate backup copy to the backup account
- During disaster recovery exercises, database snapshots are copied to the DR account
- The RDS snapshot identifier is specified via a **Parameter Store variable** in the database CloudFormation template before deployment

[Lukasz Grabowski]: > "So how do you recover database during disaster recovery day?"

[Erik Andersson]: > "Cloud engineering would have copied database snapshots to theirs like it will exist and that would emulate that you are on the disaster recovery accounts."

---

## Step 1: Manual Secrets Manager Setup

### Required Secrets to Create

[Erik Andersson]: You must manually create these secrets before deployment:

- **Microsoft Dynamics OAuth Client ID** and related credentials
- Any other service-specific API credentials or keys
- Database encryption keys
- Integration-specific tokens

### Finding Secret Values

The Dynamics OAuth Client ID must be retrieved from:
1. Navigate to the **Microsoft Dynamics or Azure page for the Apsis International AB account** (not the FSC account)
2. Check the **Registered enterprise apps** section
3. Retrieve the required credentials from there

[Erik Andersson]: > "The problem is that the KMS key that we are using is only set up to be single region. That means that the key is not shared with the backup accounts."

### Deployment Failure as Discovery Tool

[Erik Andersson]: Deployment will explicitly fail and tell you which secret is missing, so you can use trial-and-error approach if needed. However, documenting the exact secrets required upfront would improve the process significantly.

---

## Step 2: Configuration File Setup - Environment Variables

### Purpose and Location

Parameter Store values are configured via environment-specific configuration files that:
- Populate Parameter Store with shared values used across multiple CloudFormation files
- Reduce duplication compared to the legacy approach where each service had its own configuration file
- Enable the `make deploy` command to automatically select the correct configuration based on AWS profile

### File Naming Convention

```
nv-<AWS_PROFILE>.conf
```

If your AWS profile is `prod`, the file is `nv-prod.conf`. For disaster recovery with profile `dr`, create `nv-dr.conf`.

### Required Configuration Values

[Erik Andersson]: The configuration file must specify:

- **NVR domain/URL** (e.g., `nvr.com` for prod, custom URL for DR)
- **API endpoints for external services** (URLs for CRM systems, Audience, E-deal, Tribe, etc.)
- **Service request targets** (which account to request folders from, API endpoint URLs)
- **Integration-specific URLs and identifiers**

[Erik Andersson]: > "This is the most annoying part during the disaster recovery day. Like finding out what is the URL for all of these services you will need to hunt teams down to get them to."

### Historical Context: Why Parameter Store

[Erik Andersson]: Prior to refactoring to use Parameter Store, each service had its own configuration file, requiring value duplication across approximately 15 different folders. This made disaster recovery significantly worse. The current centralized approach in Parameter Store is a substantial improvement.

---

## Step 3: AWS Environment Profile Setup

### Purpose

The `.env` file (named after the AWS profile) is primarily used for:
- Making Docker images during deployment
- Specifying the **ECR (Elastic Container Registry) host** where images will be pushed

### File Naming Convention

```
.<AWS_PROFILE>.env
```

For disaster recovery account with profile `dr`, create `.dr.env`.

### Required Values

**ECR Host** (mandatory for deployment):
```
ECR_HOST=<ACCOUNT_ID>.dkr.<REGION>.amazonaws.com
```

For example, for account `123456789` in EU Central region:
```
ECR_HOST=123456789.dkr.eu-central-1.amazonaws.com
```

### Optional/Legacy Values

- Database connection parameters (only used by `make connect-to-database` command)
- API token generation values (rarely used for local testing)

[Erik Andersson]: > "The ECR host is actually utilized when we push Docker images, so you need to specify like the host for the ECR that you're using."

---

## Deployment Strategy: Parallel Service Deployment

### Main Deployment Command

```bash
make deploy
```

When executed from the root folder with the correct AWS profile set, this triggers a sequential base deployment followed by parallel service deployments.

### Deployment Sequence

#### Phase 1: Base Infrastructure (Sequential)

1. **Base CloudFormation Template**
   - ECR repositories for all services
   - Lambda deployment buckets
   - Reusable IAM policies
   - SNS topics for alarms and notifications
   - All shared infrastructure used across services

2. **Database Template**
   - RDS cluster and instance setup
   - Security groups (rules added later by other services)
   - **This step is the longest-running phase** (approximately 30+ minutes)

3. **Shared Services**
   - Squid proxy deployment
   - Kafka cluster setup (used by outbound worker)
   - Redis cache setup (takes approximately 30 minutes on first deployment)
   - SQS queue creation for all services

#### Phase 2: Service Deployment (Parallel)

[Erik Andersson]: Services are deployed in **4-5 parallel batches** to speed up deployment:

```
[List of all manager services] -> Deploy 5 in parallel
[List of all worker services] -> Deploy 5 in parallel
[List of async sync services] -> Deploy 5 in parallel
```

Complete list of services is maintained in a composite variable that includes all managers, workers, and async services.

### Design Principle: Decoupled Service Architecture

[Erik Andersson]: > "We took great efforts to make sure that every service in integration should be able to be deployed independently on each other. That's why we are deploying all the queues separately in the base deployments."

**Why This Matters:**

If queues were deployed with their consuming services, you'd create circular dependencies:
- Delta Sync Worker queue is used by Sluice Worker
- Could not deploy Sluice Worker before Delta Sync Worker
- Could not deploy Delta Sync Manager before Sluice Worker
- Cannot deploy services in parallel; must deploy sequentially
- Deletion of templates becomes complicated

By decoupling queues into the base deployment, all services can deploy independently and in parallel.

### Expected Deployment Duration

[Erik Andersson]: Assuming all goes smoothly:
- **Database creation**: 30-40 minutes (dominant factor)
- **Redis setup**: ~30 minutes on first deployment
- **All integration services**: ~35-40 minutes
- **Total estimate**: Approximately 2+ hours depending on database size and first-time setup

---

## Critical Infrastructure: Service-to-Database Security

The database is protected by a **security group**. Other services add ingress rules to this security group during their own deployment, allowing them to communicate with the database.

---

## The KMS Key Limitation (Critical Weakness)

### The Problem

[Erik Andersson]: > "There is one weakness in all of this that you need to be aware of."

The platform uses **AWS KMS (Key Management Service)** to encrypt customer API credentials for generic connector integrations:

1. When a customer installs a generic connector integration, they provide API credentials
2. These credentials are encrypted using a KMS key before storage in the database
3. When the broker service needs to make requests to the CRM system, it decrypts the credentials and adds them to request headers

This is a significant security improvement over legacy connectors, where credentials are stored without encryption or hashing.

### Single-Region Limitation

**The critical issue**: The current KMS key is configured for **single-region only**, which means:

- The key does not replicate to backup/disaster recovery accounts
- In a DR scenario, you **cannot decrypt customer API credentials** if restoring to another account
- The generic connector integrations will not function on the DR account
- Legacy connectors (Enterprise 12.0, 12.1) will still function because they don't encrypt credentials

### Why This Happened

[Erik Andersson]: > "The issue is here is that it was created initially with a single region and you cannot change this to a multi region. You need to create a completely new key and set it to multiregion from the start."

Once a KMS key is created as single-region, you cannot change it to multi-region. You must:
1. Create a completely new multi-region KMS key
2. Update all encrypted records in the database to use the new key
3. Replicate the new key to the backup account (similar to how database dumps are currently replicated)

### Impact and Workaround

[Erik Andersson]: > "It would not be an outright data loss situation. The issue is... this is just something that has never really been prioritized."

**Important distinction**: This is NOT a data loss scenario. Customer API credentials are still recoverable through:
- **Reinstalling the integration**: Customers re-authenticate, providing credentials again
- Using legacy connector integrations for verification in DR scenario

However, it prevents end-to-end verification of generic connector integrations during disaster recovery exercises.

### Recommended Fix

[Erik Andersson]: > "This is something which should be considered to be fixed. Like we should create the key, it should be set to multi-region and then it should be replicated to the backup account in the same fashion as the database dump is being done today."

1. Create a new multi-region KMS key from scratch
2. Implement a database migration script to re-encrypt all customer credentials with the new key
3. Replicate the key to the backup/DR account using the same mechanism as database replication
4. Decommission the old single-region key

### Comparison to M8

[Michal Rosikiewicz]: M8 (another platform) does not have this KMS encryption issue because it does not hash customer data in the same way. However, M8 does have database backup and needs similar configuration steps during DR.

---

## Verification Challenges

### Audience Dependency

The greatest challenge during disaster recovery exercises is **full-end-to-end verification of the recovery**, because:

- Almost every integration operation requires **retrieving profile attributes** from Audience
- Profile updates and consent updates depend on Audience being operational
- Many requests fail if Audience is not fully up and running or not properly configured for the DR environment

[Erik Andersson]: > "We tend to have a lot of time or a lot of issues verifying that the whole flow works because we are fully dependent on audience being up and running in order for us to actually be able to have like successful requests."

**In practice**: The integration infrastructure itself can be fully restored and verified as operational. However, meaningful end-to-end testing is blocked until other platform dependencies (particularly Audience) are also restored and configured in the DR account.

### Undiscovered Dependencies

[Erik Andersson]: > "I can also guarantee you that there are some new fun interdependencies that we have overlooked because we have not set up the account from scratch now for I think 2 years."

**Important caveat**: It has been approximately 2 years since the integration infrastructure was deployed from scratch. New services and integrations have been added since then. Expect to discover undocumented service dependencies during a disaster recovery exercise.

### Error Discovery Process

[Erik Andersson]: Missing dependencies and configuration problems are discovered through deployment failures that explicitly state:
- Missing secrets
- Missing service resources
- Failed resource dependencies

The error messages are typically clear, allowing you to iteratively fix configuration issues.

---

## Step-by-Step DR Recovery Checklist

[Erik Andersson]: To summarize the complete recovery process:

### Step 1: Secrets Manager
- Create all required secrets manually in AWS Secrets Manager
- Retrieve Microsoft Dynamics OAuth credentials from the Apsis International AB Azure registered enterprise apps
- Use deployment failures to discover any missing secrets

### Step 2: CloudFormation Environment Configuration
- Create `nv-<PROFILE>.conf` file in the cloud formation configuration directory
- Set `<PROFILE>` to match your AWS profile name (e.g., `dr` for disaster recovery account)
- Specify all required URLs and service endpoints for the DR environment

### Step 3: AWS Environment File
- Create `.<PROFILE>.env` file
- Specify the ECR host for the DR account in the format: `<ACCOUNT_ID>.dkr.<REGION>.amazonaws.com`

### Step 4: Deploy
```bash
make deploy
```

This single command orchestrates:
1. Base infrastructure deployment (sequential)
2. Database recovery from snapshot (if configured)
3. All service deployments (parallel batches of 4-5)

### Expected Timeline
- With database restoration: 2+ hours
- Without database restoration: ~35-40 minutes for services only
- Database creation alone: 30-40 minutes
- Redis setup: ~30 minutes

---

## Known Unknowns and Future Gotchas

[Erik Andersson]: Be prepared for:

1. **Undiscovered service interdependencies** (likely, given 2+ years since last full deployment)
2. **Missing configuration values** (need to "hunt teams down" to get current service URLs)
3. **Audience unavailability during DR testing** (most likely blocker for end-to-end verification)
4. **KMS key limitation** (can't verify generic connector integrations without new key setup)
5. **Database setup time** (can cause significant delays; consider deploying database separately with `make deploy-db`)

---

## Recommendations for Improvement

### Immediate Actions

1. **Document all required secrets** with their retrieval locations before the next DR exercise
2. **Create a DR runbook template** with all required configuration values (URLs, IDs, endpoints)
3. **Pre-generate the multi-region KMS key** now, before it's urgently needed

### Medium-term Improvements

1. **Automate secrets creation** via CloudFormation templates that define secrets without values (currently no secrets are defined in IaC)
2. **Migrate to multi-region KMS key** and re-encrypt all customer credentials
3. **Establish regular DR exercises** (at least annually) to prevent configuration drift and discover new dependencies

### Long-term Improvements

1. **Fully separate database deployments** with option to deploy just `make deploy-db` to avoid waiting 2+ hours for infrastructure when only database restoration is needed
2. **Create service mesh documentation** showing all inter-service API dependencies to enable faster configuration during DR

---

## Unresolved Questions

1. **When will the multi-region KMS key migration be prioritized?** Currently recognized as necessary but not scheduled.
2. **What is the complete list of secrets required?** Needs to be documented from actual deployment experience.
3. **Has the integration platform been tested with a full DR deployment since it was last set up 2+ years ago?** Answer: No.

---

## Key Takeaways

1. **DR Recovery is manual but procedural**: Follow the three configuration steps (secrets, environment variables, AWS environment file), then deploy with a single command.

2. **Secrets and Parameters are distinct**: Secrets must be manually created and cannot be automated. Parameters can be automated via configuration files—this is a major improvement over legacy approaches.

3. **Parallel service deployment is critical**: Services must be decoupled to allow parallel deployment. Queues are deployed upfront in the base infrastructure for this reason.

4. **KMS key single-region limitation is a real gap**: Prevents generic connector verification in DR but is not a data loss scenario. Should be fixed by creating and migrating to a multi-region key.

5. **Audience is the biggest bottleneck**: Infrastructure can be restored in 2 hours, but end-to-end verification requires Audience and other platform services to also be operational.

6. **Expect surprises**: 2+ years of service additions means undocumented dependencies. Use clear error messages to iterate through fixes.

7. **Database restoration is time-consuming**: Plan for 30-40 minutes. Consider deploying database separately to avoid blocking on the longest-running step.