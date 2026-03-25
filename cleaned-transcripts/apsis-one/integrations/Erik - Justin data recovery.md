---
source_file: Erik - Justin data recovery.txt
domain: Apsis One Integrations
topics: [Disaster Recovery Procedures, Infrastructure Deployment, Secrets Management, Database Recovery, KMS Encryption, Configuration Management, Cloud Formation]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Secrets Manager, Parameter Store, RDS, Kafka, Redis, Lambda, ECR, KMS, Database Snapshots, Squid Proxy, Generic Connector, Legacy Connectors]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector]
---

## Session Overview

This session covers the complete disaster recovery (DR) procedure for the Apsis One Integrations platform. Erik Andersson walks through the multi-step process of restoring the entire integration infrastructure from scratch using an existing database backup. The session covers three main configuration phases (Secrets Manager, environment variables, and AWS environment files), explains the deployment orchestration via Makefile, identifies a critical KMS encryption limitation with generic connector credentials, and discusses timing dependencies on upstream services like Audience.

---

## Disaster Recovery Overview and High-Level Process

**Disaster recovery for integrations is mostly straightforward but requires manual setup for secrets.** The recovery process involves two main configuration phases, followed by automated infrastructure deployment.

[Erik Andersson]: To set up integration completely from scratch with an existing database recovery, there's some preparational work needed, but otherwise it should just be one command that you run.

The process is broken into three distinct steps:
1. Manually recreate secrets in Secrets Manager
2. Create environment variable configuration files to populate the Parameter Store
3. Create AWS environment files with ECR host information
4. Execute `make deploy` from the root folder to orchestrate all infrastructure

---

## Phase 1: Secrets Manager Configuration

### Manual Secret Creation

The Secrets Manager contains configurations that **do not have CloudFormation templates** and must be created manually before deployment. The system will fail deployment and indicate which secrets are missing, but this manual creation is unavoidable.

[Erik Andersson]: Some of these secrets you need to create manually because we don't have any cloud formation files for that. I'll try to write down exactly which ones you need to add and where you can find them.

### Microsoft Dynamics OAuth Credentials

The most critical secret is the **Dynamics OAuth Client ID**. This must be retrieved from the Microsoft Azure portal under the Apsis International AB account (not the FSC account).

[Erik Andersson]: The dynamics OAuth client ID needs to be added manually. The big thing here is knowing where you retrieve this secret from. We've been in the Microsoft Dynamics or in the Azure page for the Apsis International AB account, and then you check on registered enterprise apps.

### Future Improvement: CloudFormation-Defined Secrets

Rather than discovering missing secrets through failed deployments, there's a potential improvement: define all secrets in CloudFormation files without providing values. This would provide a complete list upfront instead of trial-and-error deployment cycles.

[Erik Andersson]: What we could do as an improvement is that you define the secrets in a cloud formation file, but obviously you don't enter the values. By doing that you would still know exactly which secrets you need to fill in instead of having to manually do that through trial and error by deploying, encountering error, deploying, encountering a second error, etcetera.

---

## Phase 2: Parameter Store Configuration via Environment Variables

### Shared Configuration Rationalization

The integration platform uses a **centralized Parameter Store approach** where common configuration values (service URLs, API endpoints, audience accounts) are reused across all services rather than duplicated in individual service configurations.

[Erik Andersson]: We also have values here in the parameter store because a lot of our cloud formation files reuse the same environment variables. Almost every worker and manager we have uses values like which audience account should I make requests to, or what is the URL to the API that we send to CRM systems. Configuration like that is shared between almost all cloud configuration files, and therefore we load those values from parameter store when we deploy.

### Creating Environment-Specific Configuration Files

For each deployment environment (DR, staging, production), create a configuration file named `nv-{ENVIRONMENT_NAME}.conf` containing all necessary parameter values.

**File naming convention:**
```
nv.com (example for production)
nv-dr.conf (example for disaster recovery)
nv-staging.conf (example for staging)
```

The environment name used in the configuration file **must match the AWS profile name** used for deployment. If deploying to a disaster recovery account with profile `dr`, the file must be named `nv-dr.conf`.

### Configuration Parameters Required

The configuration file must specify URLs and endpoints for services including:
- CRM system endpoints
- API service URLs
- Integration service endpoints
- Audience account references
- Other shared infrastructure URLs

[Erik Andersson]: This is the most annoying part during the disaster recovery day—finding out what is the URL for all of these services. You will need to hunt teams down to get them to provide these values.

### Historical Context: Configuration Refactoring

Previously, the integration platform had **individual configuration files for each service**, requiring the same values to be duplicated across 15+ different folders. Erik Andersson refactored this to use centralized Parameter Store values.

[Erik Andersson]: With the previous way of handling configuration in integration, this was 20 times worse because each service had its own configuration file, so you had to duplicate the values in different folders. It was an absolute nightmare, so I refactored all of that to reuse the values from the parameter store now instead.

---

## Phase 3: AWS Environment Files Configuration

### ECR Host Registration

In the AWS environments directory, create a file matching the deployment profile name (e.g., `.env-dr`) that specifies the **ECR (Elastic Container Registry) host** for pushing Docker images.

**Critical requirement:** The ECR host must be specified because it's used when pushing Docker images to the registry during deployment.

```
ECR_HOST=123456789012.dkr.ecr.eu-central-1.amazonaws.com
```

Where:
- `123456789012` is the AWS account ID of the disaster recovery account
- `dkr.ecr` is the standard ECR endpoint
- `eu-central-1` (or equivalent) is the regional endpoint

### Optional Parameters

Other parameters in the environment file (database connection, API token generation) are included for local testing scenarios but are **not required for deployment to work**.

[Erik Andersson]: Not all of this needs to be specified for the actual deployment to work, but the ECR host you need to [specify] because this parameter file here, for example, or the database—these values are only utilized if you make like the 'connect to database' makefile command that we have. But the ECR host is actually utilized when we push Docker images.

---

## Database Recovery and Snapshots

### Backup Strategy

Daily database backups are performed and stored in the AWS account. Additionally, Cloud Engineering maintains a secondary backup in the backup account (not the DR account). The backup occurs at **2:00 AM daily**.

[Erik Andersson]: We do have daily backups in our own accounts. The daily backups occur at 2:00, and cloud engineering has set up a backup to the backup account, but that is not handled by us.

### RDS Snapshot Restoration

When recovering the database during a DR exercise, Cloud Engineering copies the RDS snapshots to the disaster recovery account. During the database template deployment, the **snapshot identifier must be specified in the configuration**.

[Erik Andersson]: If you want to restore from a database or from a snapshot, then you fill this parameter store variable. The snapshot would exist in the account, and before you deploy the database template you add this in the configuration file.

### Procedure

1. Cloud Engineering copies database snapshots to the DR account
2. In the database CloudFormation template configuration, specify the snapshot identifier
3. Deploy the database template, which will restore the RDS cluster and instance from the snapshot

---

## Infrastructure Deployment Orchestration

### Main Deployment Process

From the root folder, execute:
```
make deploy ABS_PROFILE=dr
```

This command triggers a coordinated deployment sequence that takes approximately **90-120 minutes** total (accounting for database and Redis setup time).

### Deployment Sequence

**1. Base Templates (first)**
- Creates all ECR repositories
- Creates Lambda deployment buckets
- Creates IAM policies reused across services
- Creates SNS topics for alarms and monitoring
- Sets up shared infrastructure

**2. Database Templates (longest phase)**
- Sets up RDS cluster and instances
- Creates security groups
- Estimated time: **30 minutes**

**3. Infrastructure Templates**
- Deploys Squid Proxy
- Deploys Kafka (used by outbound worker)
- Creates Redis cluster
- Creates all message queues for integration services
- Estimated time: **30 minutes for Redis**

**4. Microservices Deployment (parallel)**
- Deploys all integration services in parallel (4-5 at a time)
- Estimated time: **35-40 minutes**

[Erik Andersson]: The database creation takes a long time, and Redis also takes quite a while to set up the first time. But otherwise, if everything goes as it should, then it takes approximately 35-40 minutes for all of the integration services to be deployed. Count on time for the database and think like 30 minutes or so for Redis to be set up.

### Parallel Service Deployment Strategy

Services are deployed in parallel batches (4-5 at a time) by design. This is possible because **every service in integration is decoupled and can be deployed independently**.

[Erik Andersson]: We took great efforts to make sure that every service in integration should be able to be deployed independently of each other. That's why we deploy all the queues separately in base deployments—because if we deployed the Delta Sync Worker queue together with the Delta Sync Worker service, then you wouldn't be able to deploy the worker before the queue, and you wouldn't be able to deploy the manager before the worker. So we made sure to decouple each and every service so you can deploy them five at a time.

### Service Lists and Deployment

The Makefile defines:
- List of all manager services
- List of all worker services
- List of all asynchronous sync services
- Combined list of all services

The deployment executes `make deploy` on each service in batches:
```bash
# Pseudo-code representation
Deploy 5 services in parallel
Wait for batch to complete
Deploy next 5 services
[repeat until all services deployed]
```

---

## Critical Limitation: KMS Encryption and Generic Connector Credentials

### The Problem

**This is a significant weakness in the current disaster recovery capability.** The KMS key used to encrypt generic connector API credentials is configured as **single-region only**. This means:

1. The KMS key exists only in the primary account
2. The key is **not replicated to backup or disaster recovery accounts**
3. If infrastructure is restored to a DR account, **customer API credentials cannot be decrypted**

[Erik Andersson]: The KMS key that we are using is only set up to be single region. That means that the key is not shared with the backup accounts. Meaning that we would not be able to decrypt the customer's API keys if you were to restore this to another account.

### Why This Happens

When customers install a generic connector integration, they provide API credentials. The system encrypts these credentials using KMS before storing them in the database. This prevents developers from viewing plaintext credentials (unlike legacy connectors).

[Erik Andersson]: Whenever a customer installs an integration, they provide us with their API credentials. When we store this in the database, we encrypt this value using KMS so we as developers will never fully see the credentials that they have entered, in contrast to the legacy connectors where everything is visible.

### Impact on Disaster Recovery

During a DR exercise, you **can restore the database** and **can verify the infrastructure**, but you **cannot verify generic connector integrations** because the encrypted credentials cannot be decrypted without the KMS key.

However, **this is not a complete data loss situation** because:
- Customer integrations can be reinstalled (customers would need to re-enter credentials)
- Legacy connectors (Enterprise 12.0) can still be verified because they don't hash/encrypt credentials

[Erik Andersson]: It's still not the end of the world because this can be solved with a reinstallation. We absolutely do not want to end up in this situation, but it would not be an outright data loss situation. You will not be able to verify a generic connector integration on the backup account, but you can verify it with any of the legacy connectors like Enterprise 12.0 because there we don't hash the API key.

### The Fix (Not Yet Implemented)

To properly solve this:

1. **Create a new KMS key configured for multi-region from the start** (cannot convert existing key to multi-region)
2. **Replicate the new key to the backup/DR account**
3. **Re-encrypt all existing customer API credentials** using the new multi-region key
4. **Replicate the KMS key alongside database backups** (similar to how database dumps are replicated to the backup account)

[Erik Andersson]: The issue is that it was created initially with a single region and you cannot change this to multi region. You need to create a completely new key and set it to multiregion from the start. But that means that all of the keys in our database need to be modified to utilize the new key instead of the old one.

### Priority Status

This has **not been prioritized** because while it's a known weakness, the impact is not total data loss—only the loss of generic connector credentials (which can be recovered through customer reinstatement).

[Erik Andersson]: This is just something that has never really been prioritized because while we know this is a weakness, it has not been considered a full data loss in case of us having to do a complete restoration. But this is something which should be considered to be fixed.

---

## Dependency on Upstream Services: Audience

### Critical Dependency

**The biggest challenge during DR recovery is Audience startup time.** The integration platform is **fully dependent on Audience being operational** for end-to-end verification.

[Erik Andersson]: The biggest challenge we've had while doing the recovery is of course Audience takes a crazy amount of time for it to be deployed. We can succeed in restoring integration—all the infrastructure, we verified that the data exists in the database—but we tend to have a lot of issues verifying that the whole flow works because we are fully dependent on Audience being up and running in order for us to actually be able to have successful requests.

### Why Audience is Critical

Nearly every operation in integration requires interaction with Audience:
- **Retrieve attributes** for profiles
- **Retrieve profile data** for processing
- **Update profiles** 
- **Update consents**

[Erik Andersson]: We need to retrieve the attributes in essentially everything we do. We need to retrieve profile data for anything we do. We need to be able to update profiles for anything we do. We need to be able to update consents in everything we do.

### Implications

- Infrastructure can be restored and verified independently within 90-120 minutes
- End-to-end flow validation is blocked until Audience is operational
- This creates a **fragile dependency chain** that extends DR timeline significantly

---

## Deployment Troubleshooting Strategy

### Expect Hidden Dependencies

[Erik Andersson]: I can also guarantee you that there are some new fun interdependencies that we have overlooked because we have not set up the account from scratch now for I think 2 years.

Unknown dependencies will be discovered during deployment through clear error messages:
- `"Cannot find secret X"`
- `"Dependent on resource Y from service Z"`

### Mitigation Approach

Deploy the database independently and early to avoid holding up the entire process:

[Erik Andersson]: You don't want to sit and have the database deployment being held back after like 40 minutes because then you'd just close the computer and go home and rethink your life. You can even like manually go in and do the `make deploy db` just to have that out of the way.

### Trial and Error is Expected

While there will be surprises, the system is resilient and errors are generally self-explanatory.

[Erik Andersson]: Typically it is quite clear because it will say like "hello, I cannot find this specific secret" or "I am dependent on this resource from this service." So yes, it can take some trial and error, but as long as you get the database up and running, you're in good shape.

---

## Comparison with M8 Disaster Recovery

### Similarities

M8 has the same disaster recovery structure: secrets must be set up manually, parameter stores must be populated, and deployment is orchestrated through CloudFormation.

[Erik Andersson]: M8 also has the same issue—you need to set up all of these secrets and some parameter stores manually before it is deployed. But when you have set up those configurations, then it is also quite straightforward—you just deploy it in one go, essentially.

### Key Difference: No KMS Issue in M8

M8 does **not suffer from the KMS encryption limitation** because it doesn't use KMS hashing for customer credentials. The database itself is encrypted, but this is different from the application-level KMS hashing used in generic connector.

[Erik Andersson]: M8 does not suffer from the same KMS issue because we don't hash the data like [in generic connector]. The database is encrypted, but that's not what this is about. This hashing issue is only for the generic connector credentials for the customer because we have a manual step in the code where we load the secret and do a manual hash of the data.

---

## Summary of Disaster Recovery Steps

[Erik Andersson]:

**Step 1:** Recreate the secrets manually in Secrets Manager

**Step 2:** In the CloudFormation files under environment variables, create a new configuration file. The environment name here should be the same as you use for your AWS profile. If you name it `dr`, then it should be `nv-dr.conf`

**Step 3:** In the AWS environments, again create a file with the same name as the profile and add the ECR host for the new account (format: `{ACCOUNT_ID}.dkr.ecr.{REGION}.amazonaws.com`)

**Step 4:** Execute `make deploy` from the root folder to deploy everything—base templates first, then each of the services

---

## Key Takeaways

1. **Disaster recovery is procedurally straightforward** but requires careful manual setup of three configuration layers before orchestrated deployment can proceed.

2. **Secrets are the primary manual bottleneck**—define all required secrets upfront via CloudFormation files to eliminate trial-and-error deployment cycles.

3. **Parameter Store centralization significantly improved the DR process**—moving from duplicated per-service configs to shared values was essential for practical recovery.

4. **Service decoupling enables parallel deployment**—deliberate architectural choices (separate queue deployments, independent service templates) allow 4-5 services to deploy simultaneously.

5. **The KMS encryption limitation is a known weakness that should be addressed**—single-region KMS keys prevent credential recovery on DR accounts and require either re-encryption with multi-region keys or customer re-installation.

6. **Audience dependency is the critical path blocker**—while infrastructure restores in 90-120 minutes, end-to-end validation is blocked until Audience is operational.

7. **Expect undocumented interdependencies**—the system hasn't been deployed from scratch in 2+ years, so new dependency chains will likely emerge and require troubleshooting.

8. **Deploy the database early and independently**—don't let the multi-hour database creation delay validation of other services.

---

## Unresolved Questions and Action Items

### Open Action Items

- **KMS Multi-Region Key Migration**: Create new multi-region KMS key and re-encrypt all generic connector credentials (not yet scheduled)
- **Secrets Definition in CloudFormation**: Add CloudFormation definitions for all required secrets (without values) to eliminate discovery-by-error (proposed improvement, not implemented)
- **Disaster Recovery Exercise**: A full DR exercise has not been conducted in 2+ years and would likely reveal additional undocumented dependencies (timeline unclear)

### Known Unknowns

- Specific list of all secrets requiring manual creation (Erik mentioned he would document this separately)
- Exact URLs and endpoints for all services needed in `nv-{ENV}.conf` files (must be gathered from individual teams)
- New dependencies that will emerge when attempting a fresh account setup