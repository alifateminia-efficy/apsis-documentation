---
source_file: Erik - Justin data recovery.txt
domain: Apsis One Integrations
topics: [disaster recovery, secrets management, AWS Secrets Manager, AWS Parameter Store, CloudFormation deployment, KMS encryption, database snapshot restoration, RDS, ECR, infrastructure deployment order, generic connector credentials]
speakers: ["Erik Andersson (departing engineer, domain expert)", "Lukasz Grabowski (Engineering Lead)", "Michal Rosikiewicz (Engineer)", "Tomasz Kowalski (Engineer)"]
key_components: [Secrets Manager, Parameter Store, CloudFormation, RDS, KMS, ECR, Redis, Kafka, SNS, SQS, Squid Proxy, Delta Sync Worker, Sluice Worker, Generic Connector, Broker Service, Audience (external dependency)]
session_type: knowledge-transfer
---

# Disaster Recovery: Apsis One Integrations

## Session Overview

Erik Andersson walks the team through the full disaster recovery (DR) procedure for the Integrations domain, covering it from a cold-start perspective on a DR account. The session covers the three manual pre-deployment steps (Secrets Manager, Parameter Store environment file, AWS environments file), the automated `make deploy` pipeline, and the deployment order rationale. A significant known weakness is identified: the KMS key used to encrypt generic connector customer credentials is single-region and cannot be shared with the backup account, preventing verification of generic connector integrations during DR. The session also briefly compares the situation to M8 (MA), confirming MA does not share the KMS issue.

---

## Overview of DR Approach

The disaster recovery process for Integrations is described as "straightforward, albeit a bit manual for the secrets." The end goal is that once pre-deployment configuration is in place, the entire integration platform can be restored with **a single command**: `make deploy` from the root folder.

There are two categories of configuration that must be prepared before deployment:
1. **Secrets** in AWS Secrets Manager — must be created manually
2. **Parameter Store values** — populated via an environment variables configuration file

> "It should just be one command that you run." — Erik Andersson

---

## Step 1: Recreating Secrets in AWS Secrets Manager

### What Must Be Done

Secrets in Secrets Manager must be **created by hand** before deployment. There are no CloudFormation files that create these secrets. If they are missing, the deployment will fail with an error identifying which specific secret is absent — so the process can involve some trial and error.

> "Of course the deployment will fail and it will tell you exactly which secret is missing." — Erik Andersson

### Key Example: Dynamics OAuth Client ID

A critical example is the **Dynamics OAuth client ID**. This must be retrieved manually from:

- **Azure Portal** → **Apsis International AB** account (not the FSC account)
- Navigate to: **Registered Enterprise Apps**

> ⚠️ **Warning:** Use the **Apsis International AB** Azure account, not the FSC account.

### Improvement Opportunity (Not Yet Implemented)

Erik noted that a future improvement would be to define the secrets *structure* (without values) in a CloudFormation file. This would make the list of required secrets explicit and discoverable, avoiding the current trial-and-error deployment cycle.

---

## Step 2: Creating the Parameter Store Environment Variables File

### Why Parameter Store Is Used

Many CloudFormation files across Integration services share the same environment variables — for example:
- Which Audience account to send requests to when requesting folders
- The URL of the One API used by CRM systems

Rather than duplicating these values across 15+ service configuration files (the previous approach, which Erik refactored away), they are now loaded from Parameter Store at deploy time.

### How to Create the File

The file lives under the `environment variables` configuration directory in the repo. The filename convention is:

```
nv-<AWS_PROFILE>.conf
```

For example, if deploying to a DR account with AWS profile `dr`:

```
nv-dr.conf
```

The file contains all the shared URLs and configuration values that services depend on. During a real DR event, this is the most time-consuming manual step:

> "This is the most annoying part during the disaster recovery day — finding out what the URL is for all of these services. You will need to hunt teams down to get them to respond." — Erik Andersson

---

## Step 3: Creating the AWS Environments File

An AWS environments file must also be created. The naming convention mirrors the AWS profile name. This file is used by Makefile commands and — critically — **when pushing Docker images to ECR**.

The key value required here is the **ECR host**, which is account-specific:

```
<account_id>.dkr.ecr.<region>.amazonaws.com
```

For example, if the DR account ID is `123` in `eu-central-1`:
```
123.dkr.ecr.eu-central-1.amazonaws.com
```

Other values in this file (e.g., database connection helpers, One API token generation) are used only for local development convenience and are not required for deployment to function.

---

## Automated Deployment: `make deploy`

### Running the Deployment

Once the three configuration steps above are complete, the entire integration stack is deployed from the root folder:

```bash
make deploy
```

The AWS profile used determines which environment file is loaded (matching the `nv-<profile>.conf` and AWS environments file created above).

### Deployment Order and Rationale

The Makefile orchestrates deployment in a deliberate sequence:

#### Phase 1: Base Infrastructure Templates

The base template is deployed first. It creates all shared infrastructure:
- **ECR repositories**
- **Lambda S3 buckets**
- **Shared IAM policies**
- **SNS topics** for alarms
- **Squid proxy**
- **Kafka** (used by the outbound worker)
- **Redis**
- **All SQS queues** for every service

**Why queues are deployed here (critical design decision):** All queues are deployed in the base template rather than alongside their respective services. This ensures every service can be deployed independently and in parallel without inter-service dependencies at deploy time.

> "If you were to deploy the Delta Sync Worker queue together with the Delta Sync Worker, then you would not be able to deploy the Sluice Worker before you had deployed the Delta Sync Worker, and you would not have been able to deploy the Delta Sync Manager before the Sluice Worker... It would be a pain to both deploy things in parallel or even more so to delete templates." — Erik Andersson

#### Phase 2: Database Template

Deploys:
- RDS cluster and instance
- Security group (rules added later by individual services)

This phase takes the longest. **Count on approximately 1 hour for the database.**

#### Phase 3: All Integration Services (Parallel)

All services — managers, workers, async sync services — are deployed **5 at a time in parallel**. The Makefile defines:
- A list of all manager services
- A list of all worker services
- A list of all async sync services
- A combined complete list

The deploy runs batches of 5 from the combined list concurrently.

**Estimated total deployment time (excluding database):**
- Database: ~1 hour
- Redis first-time setup: ~30 minutes
- All other services: ~35–40 minutes

### Database Snapshot Restoration

To restore from a snapshot (as would be the case on a DR account), set the snapshot identifier in the **database CloudFormation template's Parameter Store variable** before running `make deploy`. The snapshot from the production account will have been copied to the DR account by cloud engineering.

> "Because we are using RDS, you would specify that in the database templates when you deploy it." — Erik Andersson

### Practical Tip: Deploy Database First

Erik recommends running the database deployment in isolation first to avoid a scenario where the database creation holds up the entire pipeline for 40+ minutes and blocks everything else:

```bash
make deploy-db
```

---

## Database Backup Architecture

- **Daily automated backups** exist in the production accounts at approximately 2:00 (time zone unspecified in session).
- **Cloud Engineering** manages a secondary backup to a dedicated backup account — this is not managed by the Integrations team.
- On a DR day, cloud engineering would have pre-copied a database snapshot to the DR account, which is what is used for restoration.

---

## Known Critical Weakness: KMS Key Is Single-Region

### The Problem

When a customer installs a **Generic Connector** integration, they provide API credentials. These credentials are **encrypted using a KMS key** before storage in the database. The Broker Service decrypts them at request time to add the API key to outbound headers.

The KMS key used for this encryption was configured as **single-region** at creation time.

**Consequences:**
- The key is not replicated to the backup/DR account.
- On a DR account, the Broker Service **cannot decrypt the stored customer API credentials**.
- Generic Connector integrations **cannot be verified** during a DR exercise on the backup account.

### Why It Cannot Be Fixed In-Place

> "You cannot change a single-region key to multi-region. You need to create a completely new key and set it to multi-region from the start." — Erik Andersson

Fixing this would require:
1. Creating a new KMS key with multi-region enabled
2. Re-encrypting all stored customer credentials in the database to use the new key
3. Replicating the new key to the backup account

This has not been prioritized because it is not an outright data loss scenario — customers could reinstall their integrations. However, it is considered a significant suboptimal situation.

### Workaround During DR Exercises

Verification during DR can still be performed using **legacy connectors (e.g., Enterprise 12.0)**, because those do not use the KMS-encrypted credential flow (API keys are stored without this manual encryption step, though the database itself is encrypted at rest).

> ⚠️ **Action Item / Known Debt:** The KMS key should be recreated as multi-region and replicated to the backup account in the same manner as the database snapshot. This has never been prioritized but should be.

---

## Comparison with M8 (MA) Platform

- M8 has the same requirement: secrets and Parameter Store values must be manually configured before deployment, after which it is a single-command deploy.
- **M8 does NOT have the KMS issue** — M8 does not perform a manual KMS encryption step on customer credentials in code.
- The database in M8 is also encrypted at rest (RDS encryption), but that is standard infrastructure-level encryption, not the same as the application-level KMS hashing used in the Generic Connector.

---

## Warning: Potential Undiscovered Dependencies

Erik explicitly flagged that the integration stack has not been deployed from scratch for approximately **2 years**. There is a real possibility that new inter-service dependencies or missing secret references have been introduced that are not yet known.

> "I can guarantee you there are some new fun interdependencies that we have overlooked because we have not set up the account from scratch for about 2 years." — Erik Andersson

These will surface as deployment errors identifying the missing resource or secret. The approach is iterative — errors will be explicit.

---

## DR Dependency on Audience Platform

A recurring challenge during DR exercises is that Integration is **fully dependent on the Audience platform** being operational:

- Retrieving profile attributes
- Reading profile data
- Updating profiles
- Updating consents

Audience is a complex service and historically takes a very long time to come up during DR exercises. The Integration infrastructure itself tends to restore without issues, but **end-to-end flow verification is blocked until Audience is operational**.

---

## Step-by-Step DR Summary

1. **Recreate all secrets manually** in AWS Secrets Manager on the target account (check for the exact list — deployment errors will identify missing ones)
2. **Create `nv-<aws_profile>.conf`** in the environment variables directory with all required shared service URLs and configuration values
3. **Create the AWS environments file** with the same name as the AWS profile, specifying the correct **ECR host** for the target account
4. Run from the root folder:
   ```bash
   make deploy
   ```
   This deploys base infrastructure first, then the database, then all services in parallel batches of 5.

---

## Key Takeaways

- DR for Integrations is fundamentally a **three-step manual prep + one automated deploy** process.
- The **Secrets Manager secrets must be manually recreated** — no CloudFormation covers this. The deployment will fail with specific error messages identifying what's missing.
- The **Parameter Store / environment config file** is critical — without it, all shared configuration across services will be missing.
- The **ECR host in the AWS environments file** is required for Docker image pushes to work.
- **Deployment order is enforced by the Makefile** and is intentional — queues first in base, then database, then services in parallel.
- The **KMS single-region key is a known, unresolved weakness** that will prevent verification of Generic Connector integrations on the DR account. Legacy connector integrations can still be verified.
- **Audience platform availability** is the most likely external bottleneck to full DR verification.
- The last full from-scratch deployment was ~2 years ago — **expect undiscovered issues**.

---

## Unresolved Questions / Action Items

1. **KMS Key (High Priority Debt):** Create a new multi-region KMS key, re-encrypt all generic connector customer credentials in the database to use the new key, and configure replication to the backup account. Currently unprioritized but acknowledged as a real gap.
2. **Secrets Documentation:** Erik mentioned he would try to write down exactly which secrets need to be created manually and where to retrieve them. Confirm this documentation was produced before his departure.
3. **CloudFormation Improvement (Low Priority):** Define secret *names* (without values) in CloudFormation so the full required secrets list is explicit and discoverable without trial-and-error deployment.
4. **DR Exercise:** The team has not done a full DR exercise recently. One should be scheduled to surface any new dependency issues that have accumulated over the past ~2 years.