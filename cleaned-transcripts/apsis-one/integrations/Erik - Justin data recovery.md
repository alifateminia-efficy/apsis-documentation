---
source_file: Erik - Justin data recovery.txt
domain: Apsis One - Integrations
topics: 
  - Disaster Recovery Procedures
  - Database Recovery and Restoration
  - Secrets Management Configuration
  - Environment Configuration and Parameter Store
  - KMS Encryption and Multi-Region Key Management
  - Infrastructure Deployment via CloudFormation
  - Service Interdependencies
session_type: knowledge-transfer
speakers: 
  - Erik Andersson
  - Lukasz Grabowski
  - Michal Rosikiewicz
  - Tomasz Kowalski
key_components:
  - Secrets Manager
  - Parameter Store
  - AWS CloudFormation
  - RDS Database
  - KMS (Key Management Service)
  - ECR (Elastic Container Registry)
  - Lambda
  - SNS
  - Redis
  - Kafka
  - Squid Proxy
  - Generic Connector
  - Enterprise Connectors
  - Broker Service
---

## Session Overview

This session covers the complete disaster recovery procedure for the Apsis One Integrations platform, focusing on how to restore the integration infrastructure from scratch while recovering an existing database. The key discussion centers on the manual configuration of secrets and environment variables, the CloudFormation deployment pipeline that orchestrates 40+ microservices in parallel, and a critical limitation with KMS key encryption that prevents credential decryption in backup accounts for generic connectors. Erik Andersson walks through the three main preparatory steps and the subsequent automated deployment process, while also highlighting the unresolved issue of single-region KMS keys and the lengthy wait times for dependent services like Audience to come online.

---

## Disaster Recovery Setup: Three Critical Preparatory Steps

### Step 1: Manual Secrets Recreation in AWS Secrets Manager

[Erik Andersson]: The disaster recovery process in integrations is relatively straightforward, albeit somewhat manual for secrets management.

To set up the integration infrastructure completely from scratch with an existing database recovery, there are preparational steps required. The first and most critical step is recreating secrets in **AWS Secrets Manager** manually, as there are no CloudFormation files that define them.

**Secrets that must be created manually:**
- **Microsoft Dynamics OAuth Client ID** — This is the most important one
- Database encryption key
- Any other service-specific credentials

[Erik Andersson]: The big thing here is knowing where you retrieve these secrets from. For the Dynamics OAuth Client ID, you need to access the Microsoft Dynamics or Azure page for the **Apsis International AB account** (not the FSC account), then check the **Registered Enterprise Apps** section to obtain the client ID.

**Important caveat**: The deployment will fail and explicitly tell you which secrets are missing. However, this creates a trial-and-error process if you're not aware upfront of what needs to be added. As an improvement, you could define secrets in CloudFormation files without entering their values—this would clarify exactly which secrets need to be filled in, rather than discovering them one at a time through deployment failures.

### Step 2: Environment Variable Configuration File

[Erik Andersson]: The second configuration set is more automated and uses **AWS Parameter Store** to store shared configuration values. Many CloudFormation files reuse the same environment variables across services—for example, the Audience account URL, CRM system URLs, and folder service endpoints.

**Configuration file structure:**
- Create a file under the environment variables directory named following the pattern: `nv-<ABS_PROFILE>.conf`
- For example, if deploying to a disaster recovery account named "dr", create: `nv-dr.conf`
- If deploying to integration with ABS profile "prod", create: `nv-prod.conf`

**Example configuration values you'll need to specify:**
- `nvr.com` (Audience/CRM account details)
- URLs for all dependent services (Folder service, CRM systems, APIs)
- Any shared configuration values referenced across multiple services

[Erik Andersson]: This is the most annoying part during a disaster recovery exercise—finding out what the URLs are for all these services. You will need to reach out to teams to gather this information.

**Historical context**: In the previous approach to configuration management, each service had its own configuration file, requiring duplication of values across 15+ different folders. This was refactored to centralize values in Parameter Store, drastically reducing the manual work during recovery.

### Step 3: AVS Environment-Specific Configuration

[Erik Andersson]: In the AVS environments, you also need to create a configuration file with the same name as your ABS profile.

**Key parameter: ECR Host**
- This is the only parameter strictly required for deployment to work
- Format: `<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com`
- Example: If the disaster recovery account is `123456789`, the ECR host would be `123456789.dkr.ecr.eu-central-1.amazonaws.com`

**Other parameters** (used for optional make file commands):
- Pam file (used only for specific `make connect-to-database` commands)
- Database connection details (used only for local testing)
- API token generation parameters (legacy, no longer regularly used)

The ECR host is actually utilized when pushing Docker images to the ECR registry during deployment, so it must be correctly specified.

---

## CloudFormation Deployment Pipeline and Architecture

### Deployment Process Overview

[Erik Andersson]: Once secrets and configuration files are in place, deployment is straightforward: run `make deploy` in the root folder with your ABS profile (e.g., `make deploy ABS_PROFILE=dr`).

This triggers a long orchestrated process that deploys all necessary infrastructure in phases:

### Phase 1: Base Infrastructure Deployment

The **base template** is deployed first, which sets up:
- All ECR repositories (one per service)
- Lambda deployment buckets for code uploads
- Shared IAM policies (reused across services)
- SNS topics for alarms and alerts
- Foundation for all shared infrastructure

### Phase 2: Database and Persistent Services

The **database template** is deployed next, establishing:
- RDS cluster configuration
- RDS instance creation
- Security groups for the database (rules added later by dependent services)

**Additional infrastructure deployed in this phase:**
- **Squid Proxy** — for outbound network traffic
- **Kafka** — utilized by the outbound worker
- **Redis** — for caching and session management
- All **SQS queues** used by integration services

[Erik Andersson]: Database creation takes the longest time. Redis also takes considerable time to set up on first deployment. This phase is resource-intensive and unavoidable.

### Phase 3: Microservice Deployment in Parallel

[Erik Andersson]: Once base infrastructure is ready, all services in the integration domain are deployed in parallel batches of 4-5 at a time.

**Service categories:**
- Manager services (orchestration and coordination)
- Worker services (processing and async operations)
- Asynchronous sync services
- Supporting services

The deployment strategy uses a **Makefile pattern** that:
1. Defines lists of all managers, workers, and async services
2. Combines them into a complete service list
3. Executes `make deploy` on all services simultaneously in batches
4. Awaits batch completion before proceeding to the next batch

**Why parallel deployment was critical**: Each service was designed to be independently deployable with no hard dependencies on other services. This was achieved by deploying all queues separately in the base phase. If queues were coupled with their consuming services, deployment order would become a nightmare:
- Delta Sync Worker queue must exist before deploying Delta Sync Worker service
- Delta Sync Worker service must exist before deploying Delta Sync Manager
- This would prevent parallel deployment and make template deletion extremely fragile

[Erik Andersson]: The Bash code that orchestrates this looks intimidating, but it essentially means: deploy five services in parallel, wait for the batch to finish, then deploy the next five.

### Deployment Timing Expectations

[Erik Andersson]:
- **Database creation**: 30+ minutes
- **Redis setup**: 30+ minutes
- **All integration services**: 35-40 minutes (if database and Redis are excluded)
- **Total estimated time**: 1.5-2+ hours for complete restoration

**Database takes the longest and is unavoidable, so deploy it early** — even manually running `make deploy-db` separately allows you to kick it off and work on other tasks.

---

## Critical Limitation: KMS Key Encryption and Backup Account Recovery

### The Problem: Single-Region KMS Key

[Erik Andersson]: There is one significant weakness in the current setup that you absolutely need to be aware of.

The integration infrastructure uses an **AWS KMS key** for encrypting customer API credentials when they are stored in the database. This encryption only applies to the **generic connector** architecture.

**How it works:**
1. When a customer installs a generic connector integration, they provide their API credentials
2. These credentials are encrypted using KMS before being stored in the database
3. Developers never see the full credentials in plaintext
4. When the broker service needs to make a request to a customer's system, it decrypts the credentials using KMS and adds them to the request header

**The limitation**: The KMS key was created with **single-region configuration only**. This means:
- The key exists only in the primary region
- The key is not replicated to backup or disaster recovery accounts
- **If you restore the database to a backup account, you cannot decrypt the customer API credentials**

[Erik Andersson]: This is very suboptimal but not an outright data loss situation. It can be solved through customer reinstatement—customers would need to reinstall their integrations, providing their credentials again. However, this is something to absolutely avoid.

### Why This Happened and How to Fix It

[Erik Andersson]: The KMS key was created initially with single-region configuration, and unfortunately, **you cannot change an existing single-region key to multi-region**. You must:

1. Create a completely new KMS key with **multi-region enabled from the start**
2. Modify all credential entries in the database to reference the new key instead of the old one
3. Replicate the new key to the backup account (similar to how database snapshots are currently replicated by Cloud Engineering)

**Why it hasn't been fixed yet**: While this weakness is known, it hasn't been prioritized because:
- Credential loss isn't considered a full data loss scenario (customers can reinstall)
- The effort to migrate all encrypted values is non-trivial
- No recent disaster recovery exercises have forced the issue

[Erik Andersson]: This should be considered for fixing, but understand this is a known technical debt item.

### Workaround: Legacy Connector Verification

[Erik Andersson]: During a backup account verification exercise, you will not be able to verify generic connector integrations due to the KMS limitation. However, you **can** verify integrations using legacy connectors like **Enterprise 12.0** because:
- Legacy connectors do not encrypt API keys
- Keys are stored in a different format that doesn't depend on the multi-region KMS setup

This allows partial verification of the restored environment without waiting for the KMS key migration.

---

## Comparison: M8 vs. Integrations Disaster Recovery

[Michal Rosikiewicz]: During the M8 disaster recovery discussion (held the day before), similar steps were required for secrets and parameter store configuration.

**Key differences between M8 and Integrations:**

| Aspect | M8 | Integrations |
|--------|--|----|
| Manual secrets | Yes | Yes |
| Parameter store config | Yes | Yes |
| KMS encryption issue | **No** | **Yes** |
| Data encryption method | Database-level only | Database + credential-specific KMS encryption |

[Erik Andersson]: M8 does not suffer from the same KMS credential encryption issue because M8 does not implement the manual hashing step for customer credentials. While the database itself is encrypted at rest, this is different from the manual credential encryption used in the generic connector.

---

## Deployment Interdependencies and Known Challenges

### Audience Service Dependency

[Erik Andersson]: The biggest challenge faced during recovery testing is that **Audience takes an extraordinarily long time to be fully deployed and operational**. 

The integrations infrastructure has significant dependencies on Audience:
- **Profile retrieval**: Essential for almost every operation—needed to fetch customer attributes
- **Profile updates**: Required for any data modification operations
- **Consent updates**: Every consent management operation depends on Audience

[Erik Andersson]: We typically succeed in restoring all integration infrastructure and can verify that data exists in the database, but validation of the complete workflow often stalls waiting for Audience to become operational. The infrastructure comes up successfully, but functional testing is blocked.

### Undiscovered Interdependencies

[Erik Andersson]: I can guarantee there are new interdependencies that have been overlooked because the infrastructure hasn't been set up from scratch in approximately 2 years.

These typically manifest as:
- Missing secrets during deployment
- Resource references to services not yet deployed
- Configuration values pointing to unavailable endpoints

**The good news**: These errors surface clearly in deployment logs:
- "Cannot find this specific secret"
- "Dependent on resource X from service Y that doesn't exist"

**Mitigation strategy**: Trial and error is acceptable. As long as you get the database up and running first, you can handle other issues incrementally. Don't wait for all errors to surface—deploy the database separately to avoid a situation where, after 40 minutes of waiting, database deployment fails and blocks everything else.

---

## Summary: Step-by-Step Disaster Recovery Checklist

[Erik Andersson]: To summarize the complete procedure:

**Step 1: Secrets Manager Setup**
- Recreate all required secrets manually in AWS Secrets Manager
- Specifically retrieve and add:
  - Microsoft Dynamics OAuth Client ID (from Apsis International AB's registered enterprise apps in Azure)
  - Database encryption key
  - Any other service-specific credentials

**Step 2: Environment Configuration File**
- Create a file: `nv-<ENVIRONMENT_NAME>.conf` (where environment name matches your ABS profile)
- Populate with all shared configuration values:
  - Audience account URLs
  - CRM system endpoints
  - Folder service URLs
  - All other dependent service URLs
- Add these values to Parameter Store during deployment

**Step 3: AVS Environment Configuration**
- Create a file with the same name as your ABS profile
- Specify the **ECR host** (required): `<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com`
- Optionally specify other parameters if running local commands

**Step 4: Deployment**
- Run: `make deploy` in the root folder (with appropriate ABS_PROFILE setting)
- This orchestrates:
  1. Base infrastructure (ECR, Lambda buckets, policies, SNS)
  2. Database and persistent services (RDS, Redis, Kafka, Squid Proxy)
  3. All microservices in parallel batches (4-5 at a time)

**Expected timeline**: 1.5-2+ hours depending on database and Redis setup times

**Known limitation to plan for**: Generic connector credentials cannot be decrypted in backup accounts due to single-region KMS key—plan for customer reinstallation or accept this as a known gap.

---

## Unresolved Issues and Future Considerations

### Priority: KMS Key Migration

The single-region KMS key limitation should be addressed before the next full disaster recovery exercise:
- Create new multi-region KMS key
- Migrate all encrypted credentials in the database to use the new key
- Replicate key to backup account
- Document the migration process

### Documentation Gap

An improvement would be to create explicit documentation listing:
- Exactly which secrets must be manually created and where to retrieve them
- Complete URL mappings for all dependent services
- Step-by-step screenshots for obtaining Dynamics credentials

This would eliminate the current requirement to "hunt teams down" for configuration details during recovery.

### Testing Cadence

[Erik Andersson]: A complete disaster recovery exercise should ideally be performed at least annually (or every 2 years maximum) to:
- Uncover new interdependencies before they become critical
- Validate the procedure remains accurate
- Train team members on the recovery process

The 2-year gap since the last from-scratch setup is concerning for maintaining operational readiness.

---

## Key Takeaways

1. **Disaster recovery for integrations is achievable but manual in parts**: Secrets and configuration must be manually recreated, but the CloudFormation deployment pipeline automates the rest.

2. **Three preparatory steps are essential**: (1) Secrets Manager recreation, (2) Environment configuration file with all URLs, (3) ECR host specification in AVS environment file.

3. **Parallel deployment architecture is critical**: Services were deliberately decoupled (queues deployed separately) to enable 4-5 services to deploy simultaneously, avoiding dependency chain nightmares.

4. **Database deployment is the bottleneck**: Expect 30+ minutes for RDS and plan accordingly. Deploy it first separately if needed to avoid blocking the entire recovery.

5. **Generic connector credentials have a critical vulnerability**: Single-region KMS keys prevent credential decryption in backup accounts. This is a known issue requiring future architectural work but is solvable through customer reinstallation.

6. **Audience service is a major dependency**: Plan for extended wait times for the Audience service to come online during disaster recovery testing—this often blocks functional validation despite successful infrastructure deployment.

7. **Expect undiscovered issues**: It's been 2 years since a complete from-scratch setup. New interdependencies certainly exist and will surface during deployment, but they're usually self-documenting in error messages.

8. **Trial-and-error is acceptable**: As long as you understand the basic three-step setup and deployment pipeline, incremental problem-solving is reasonable and expected during actual recovery.