---
source_file: Erik - Justin data recovery.txt
domain: Apsis One - Integrations
topics: [disaster recovery procedures, secrets management, KMS encryption, database restoration, CloudFormation deployment, configuration management, generic connector credentials, multi-region replication]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Secrets Manager, Parameter Store, KMS keys, RDS, Lambda, ECR, SNS, Redis, Kafka, generic connector, legacy connectors, database snapshots, CloudFormation templates]
session_type: knowledge-transfer
---

## Session Overview

This session covers the end-to-end **disaster recovery (DR) procedure for the Apsis One Integrations domain**. Erik Andersson walks through the manual and automated steps required to restore the integration system from scratch using an existing database backup. The discussion covers secrets management, environment configuration, infrastructure deployment, and importantly, identifies a critical KMS key limitation that would prevent verification of generic connector integrations on a DR account.

---

## Disaster Recovery Process Overview

### High-Level Steps

The complete disaster recovery process consists of three main configuration steps followed by automated infrastructure deployment:

1. **Secrets Manager recreation** (manual)
2. **Environment variables configuration** (parameter store population)
3. **AWS environment files setup** (ECR host specification)
4. **Infrastructure deployment** (automated via `make deploy`)

[Erik Andersson]: > The disaster recovery in integration is rather straight forward, albeit a bit manual for the secrets.

To restore integration completely from scratch with an existing database recovery, there is preparatory work required, but otherwise it should be a single `make deploy` command that handles most of the deployment.

---

## Secrets Manager Configuration

### Manual Secret Creation

The integrations system depends on two sets of configurations:

1. **Secrets Manager** - contains sensitive values that must be created **by hand**
2. **Parameter Store** - contains non-sensitive configuration values that can be automated

[Erik Andersson]: The secrets you need to create by hand. Most importantly, um. Where do we have it? The dynamics O of client ID. This needs to be added manually and the big thing here is knowing where you retrieve this secret from.

### Dynamics O Client ID Retrieval

**Critical source location**: Microsoft Azure account under **Apsis International AB** (not the FSC account)

Steps to retrieve:
- Navigate to the Apsis International AB Azure account (not FSC)
- Check **registered enterprise apps** section
- Extract the Dynamics O client ID
- Add to AWS Secrets Manager manually

[Erik Andersson]: If you remember like we have been in on the Microsoft Dynamics or in the Azure page for the Apsis International AB account, so not the FSC account but the Apsis International AB. And then you check on the registered, uh, registered enterprise apps.

### Deployment Error Handling for Secrets

The deployment process will fail and explicitly tell you which secret is missing, making troubleshooting easier than trial-and-error. However, this requires knowing upfront which secrets to recreate.

[Erik Andersson]: Of course the deployment otherwise will fail and it will tell you exactly which secret is missing.

### Improvement Opportunity for Secrets Setup

[Erik Andersson]: What we could do as an improvement is that you define the secrets in a cloud formation file, but obviously you don't enter the values. ... Doing that you would still know exactly which secrets you need to fill in in opposite to like having to manually do at trial and error by deploying encounter error, deploying encounter second error, etcetera, etcetera.

---

## Parameter Store and Environment Configuration

### Shared Configuration Values

The integrations domain reuses many configuration values across CloudFormation files. Examples include:

- **Audience account** - which account to query for profile data
- **API endpoint URLs** - URLs for CRM systems and other services
- **Common service URLs** - shared across workers and managers

[Erik Andersson]: A lot of our cloud formation files, they reutilize the same environment variables, for example like almost every worker and manager we have are like using values from like where can I find or which audience account should I make request to when I request folders or stuff or like what is the URL to the one API that we send over to the um. To the CRM systems.

### Environment Variables File Structure

Create a configuration file in the root with naming pattern:

```
env.<ENVIRONMENT_NAME>.conf
```

Where `<ENVIRONMENT_NAME>` matches your AWS profile name.

**Example**: If deploying to DR account named `dr`, create:
```
env.dr.conf
```

### Configuration File Variables

The environment configuration file specifies all shared parameter store values including:

- Service URLs
- API endpoints
- Account identifiers
- CRM system URLs
- Integration system endpoints

[Erik Andersson]: For example like if you are deploying to the Um. Disaster recovery like you would have here like nvr.com and in integration like this is automated so like if you do make deploy and your ABS profile is prod. Then it will look for a file called Nv dash and then the AVS profile. So if you're deploying to the disaster recovered day account, let's call it Dr. you would create a Nv. Dashdr.conf confile and you can see here like all the things which which you need to which you need to specify.

### Configuration Discovery Challenge

[Erik Andersson]: This this is the most annoying part during the disaster recovery day. Like finding out what is the URL for all of these services you will need to hunt teams down to get them to.

This is a known pain point — teams must be contacted to provide the correct service URLs and endpoints for the DR environment.

### Historical Context: Configuration Refactoring

Previously, each service had its own configuration file, requiring duplication of values across 15+ different folders. This made disaster recovery significantly more complex.

[Erik Andersson]: I can tell you with the previous way of us handling the configuration integration, this was 20 times worse because each service had its own configuration file, so you had to duplicate the values in like 15 in different folders, it was an absolute nightmare, so I refactored all of that. So we are reusing the values from the parameter store now instead.

---

## AWS Environment Files Setup

### AVS Environment Files Purpose

AWS environment files are used for:
1. **Make file commands** - provide build context
2. **Docker image pushing** - ECR repository specification

Create files matching the AWS profile name:

```
<PROFILE_NAME>.env
```

### Required ECR Host Configuration

The **ECR host** is the critical required value. Other variables may be optional but ECR is essential for pushing Docker images.

[Erik Andersson]: The ECR host is actually utilized when we push Docker images, so you need to specify like the the host for the. ECR that you're using. So let's say that you are have like a the disaster recovery account is like 123 then you would have like the ECR host here 123 dot DKR and then if it is EU central or whatever it might be.

### Example ECR Host Format

```
123.dkr.ecr.eu-central-1.amazonaws.com
```

Where:
- `123` = AWS account ID for DR account
- `ecr.eu-central-1` = ECR region

### Optional Variables

Variables like database connection details and API token generation are optional and only used if:
- Running `make connect-to-database` commands
- Generating local API tokens for testing

---

## Infrastructure Deployment Process

### Deployment Command

Once all configurations are in place:

```bash
make deploy
```

Run this from the root folder with your AWS profile set appropriately (e.g., `--profile dr`).

### Deployment Stages

The deployment follows a specific order of operations:

#### Stage 1: Base Infrastructure

The **base template** deploys foundational shared resources:
- ECR repositories
- Lambda buckets
- Reusable IAM policies
- SNS topics for alarms
- All shared infrastructure components

#### Stage 2: Database and Core Services

The **database template** establishes:
- RDS cluster
- RDS instances
- Security groups (rules added later by services)
- **Duration**: longest stage, takes significant time

Also deployed in this stage:
- Squid proxy
- Kafka (utilized by outbound worker)
- Redis
- All queues utilized by services

#### Stage 3: Microservices Deployment

After shared infrastructure is ready, individual services deploy in **parallel batches of 4-5**.

[Erik Andersson]: We go and deploy like every micro service or maybe not so micro anymore, but every service in integration and we do this like 4 in four I think it's. Four or five in parallel because we we took great efforts to make sure that every service in integration should be able to be deployed independently on on each other.

### Service Dependency Decoupling

Services are architecturally decoupled to enable parallel deployment:

- **All queues** are deployed separately in base infrastructure
- Services do **not** depend on each other being deployed first
- This enables 4-5 services to deploy simultaneously

[Erik Andersson]: Let's say that you were to deploy the Delta Sync Worker queue together with the Delta Sync Worker because the Sluice Worker service is adding things to the Delta Sync Worker queue. Then you would not be able to deploy the sluice worker before you had deployed the Delta Sync worker and you would not have been able to deploy the Delta Sync manager before you had deployed the sluice worker like it would be a pain in the back to both deploy things in parallel or even more so to delete templates if you would need to do that. So we we made sure to decouple each and every service.

### Service Lists in Makefile

Services are organized into categories:
- **Manager services** list
- **Worker services** list
- **Asynchronous sync services** list
- **Combined list** of all services (used for deployment)

The makefile then runs `make deploy` on all services with parallel batch execution.

### Expected Deployment Timeline

- **Database setup**: ~30 minutes
- **Redis setup**: ~30 minutes
- **All services deployment**: ~35-40 minutes (if database excluded)
- **Total with database restoration**: 1.5-2 hours approximate

[Erik Andersson]: The database creation that takes like a long time, but if you disregard and also the yeah the red disk takes quite a while to set up the first time. But otherwise, if everything goes as it should, then it takes approximately 3540 minutes for all of the integration services to be deployed. Count on an. Power for the database and. Think like 30 minutes or so for Redis to be set up.

---

## Database Restoration from Snapshots

### Backup Strategy

Daily backups are handled by **cloud engineering**, not the integration team:

- **Daily backup time**: 2:00 AM
- **Backup location**: kept in own accounts
- **Cloud engineering backup**: copies database snapshots to backup account
- **Backup account**: not managed by integration team

### RDS Snapshot Restoration

When restoring to DR account during DR exercise:

1. Cloud engineering copies database snapshots to DR account
2. In RDS database template, specify snapshot ID via **parameter store variable**
3. Deployment uses this variable to restore from snapshot instead of creating new database

[Erik Andersson]: If you want to restore it from a database or from a snapshot, then you feel this. Parameter store variable. ... the snapshot would exist in the account and before you deploy the database template you you add this in the configuration file

### Disaster Recovery Account Concept

> A DR account is a separate AWS account that simulates the production environment during disaster recovery exercises. Cloud engineering pre-populates it with database snapshots to test restoration procedures.

---

## KMS Key Limitation: Critical Issue for Generic Connector

### The Problem

The integrations domain uses **KMS encryption** to secure customer API credentials for the **generic connector**. This has a critical limitation that affects disaster recovery:

[Erik Andersson]: We have a KMS key here. ... The KMS key that we are using is only set up to be single region. That means that the key is not shared with the backup accounts. Meaning that we would not be able to decrypt the customer's API keys if you were to restore this to another account.

### How Generic Connector Credentials Are Encrypted

When a customer installs a generic connector integration:

1. Customer provides API credentials
2. Credentials are **encrypted using KMS** and stored in database
3. During requests, broker service **decrypts credential** and adds to request header
4. Developers never see plaintext credentials (unlike legacy connectors)

[Erik Andersson]: Whenever a customer installs an integration, of course they provide us with the their like API credentials. And when we store this in the database, then we encrypt this value using KMS. So we as developers, we will never fully see the credentials that they have entered.

### Why Single-Region KMS is a Problem

A single-region KMS key:
- Only exists in one AWS region
- Cannot be shared to backup/DR accounts
- Means encrypted credentials cannot be decrypted in DR account
- Makes it impossible to verify generic connector integrations after restore

[Erik Andersson]: This is of course very, very suboptimal. It's still not the end of the world because this can be solved with a reinstallation. We absolutely do not want to end up in this situation, but it would. It would not be an outright. Data loss situation.

### The Workaround

If DR restore is needed:
- Generic connector integrations **cannot be verified** until KMS key issue is fixed
- **Legacy connectors** (Enterprise 1.2.0) can still be verified since they don't use KMS
- Credentials could be reinstalled by customers post-recovery

[Erik Andersson]: You will not be able to verify a generic connector integration. On the backup account, but you can verify it with any of the. Any of the legacy connectors like enterprise 12.0 because there we don't, we don't hash the API key.

### The Fix (Not Yet Implemented)

To properly solve this:

1. **Create new KMS key** with **multi-region enabled from creation**
2. Replicate key to backup account (same as database dump process)
3. **Migrate all database records** to use new key instead of old key
4. Old single-region key becomes obsolete

[Erik Andersson]: You need to. Change. You need to create a completely new key and set it to multiregion from the start. But that means that all of the keys in our database needs to be modified to utilize the new key instead of the old one.

### Why This Hasn't Been Fixed

[Erik Andersson]: This is just something that has never really been. Prioritized because while we know this is a weakness, yeah, it has not been considered a full data loss in case of us having to do a. Complete restoration, but this is something which should be considered to to be fixed.

This has been deprioritized because:
- Not classified as outright data loss
- Workaround exists (reinstall integrations)
- Requires migration of all database records
- Significant effort for non-critical recovery scenario

### Critical Caveat

⚠️ **MUST BE AWARE**: When performing DR restoration, you will **not be able to verify generic connector integrations** until this KMS multi-region setup is implemented. Plan accordingly and communicate this limitation to stakeholders.

---

## Comparison with M8 Domain

### Similarities

M8 disaster recovery follows nearly identical patterns:

- Same manual secrets creation process
- Same parameter store configuration approach
- Same CloudFormation deployment strategy
- Database backup and restoration configured identically

[Erik Andersson]: MA kind of has the same issue that you need to set up all of these secrets and some parameter stores manually before before it is deployed. But when you have set up those configurations then it is also quite straightforward that you just like. Do the. You just deploy it in one go, essentially.

[Erik Andersson]: MA also has database backup and Uniad. It needs to be configured in the exact same fashion.

### Difference: No KMS Issue in M8

M8 does **not** suffer from the KMS encryption limitation because:

[Erik Andersson]: MA does not suffer from the same KMS issue because we don't hash the data like. ... This hashing issue is only for the generic connector credentials for the customer because we have a manual. Step in the code where we load the secret and do a manual hash of the data.

M8 uses encrypted database protection but does not have the custom manual KMS hashing layer that generic connector does.

---

## Common Challenges and Gotchas

### Audience Dependency

[Erik Andersson]: The biggest challenge we've had while doing the recovery is of course, I mean audience takes a crazy amount of time for it to be for it to be deployed. So typically. We can succeed in restoring integration like all the infrastructure. We verified that the data exists in the database. We tend to have a lot of time or a lot of issues verifying. That the whole flow works because we are fully dependent on audience being up and running in order for us to actually be able to have like successful requests because we need to retrieve the attributes in essentially everything we do. We need to retrieve profile data for anything we do. We need to be able to update profiles for anything we do. We need to be able to update consents in everything we do.

The integration domain cannot fully validate restored infrastructure until the Audience service is operational and available in the DR environment.

### Overlooked Dependencies

[Erik Andersson]: I can also guarantee you that there are some new fun interdependencies that we have overlooked because we have not set up the account from scratch now for I think 2 years, but. Typically it is. You will see this quite clearly because it will say like hello, I cannot find this specific secret or I am dependent on this resource from this service.

**Expectations**: Hidden dependencies will likely surface during DR exercises. CloudFormation error messages are usually clear about what's missing.

### Trial and Error Nature

While deployment is mostly automated, the configuration discovery phase requires manual effort:

- Hunting down service teams for URLs
- Determining correct environment values
- Managing inter-team dependencies (Audience, others waiting for Integration)

[Erik Andersson]: So like yes, it can take some trial and error, but. As long as you get the database up and running, hell, you can even like manually go in and do the make deploy db just to have that out of the way. Because what you don't want to happen is to sit and have like the database deployment being. Hold back after like 40 minutes because then you were like you just close the computer and go home and rethink your life.

### Recommendation: Deploy Database First

Deploy the database separately first (`make deploy db`) rather than as part of full deployment to avoid:
- Wasting 40+ minutes waiting for full deployment if database fails
- Prolonged total recovery time
- Frustration and context-switching

---

## Practical Step-by-Step Summary

### Step 1: Secrets Manager

Recreate all required secrets manually in AWS Secrets Manager. At minimum, ensure these are present:
- Dynamics O client ID (retrieved from Apsis International AB Azure account, registered enterprise apps)
- All other service-specific secrets (deployment will tell you which ones are missing)

### Step 2: Environment Configuration

Create environment variables configuration file:

```
env.<PROFILE_NAME>.conf
```

This file must include all parameter store values:
- Service URLs
- API endpoints
- Account references
- Integration endpoints

Hunt down relevant teams to obtain current values for the DR environment.

### Step 3: AWS Environment File

Create AWS environment file:

```
<PROFILE_NAME>.env
```

**Minimum required**:
```
ECR_HOST=<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com
```

Example:
```
ECR_HOST=123456789.dkr.ecr.eu-central-1.amazonaws.com
```

### Step 4: Deploy

```bash
make deploy
```

Expected timeline:
- ~30 min database setup
- ~30 min Redis setup  
- ~35-40 min all services
- **Total: ~1.5-2 hours**

⚠️ **Critical caveat**: Validation of generic connector integrations **will not be possible** until KMS multi-region key issue is resolved.

---

## Backup Account vs. DR Account

### Backup Account
- Maintained by cloud engineering
- Contains database snapshots
- Not used for active deployments
- Not managed by integration team

### DR Account
- Separate AWS account (e.g., account ID ending in different number than production)
- Simulates production environment
- Receives database snapshots during DR exercises
- Used for testing full recovery procedures

---

## Key Takeaways

1. **Three-step configuration requirement**: Secrets Manager (manual) → Environment config (parameter store) → AWS environment file (ECR host), then single `make deploy` command handles rest

2. **KMS multi-region key is critical gap**: Generic connector integrations cannot be verified post-DR until this is fixed. This is a known limitation that should be addressed before critical recovery scenario.

3. **Services are decoupled for parallelization**: All queues deployed separately in base infrastructure enables 4-5 services to deploy simultaneously, significantly reducing deployment time

4. **Database is longest stage**: Plan for ~30 minutes of setup time. Consider deploying separately first to avoid losing 40+ minutes if full deployment fails partway through

5. **Audience dependency is validation blocker**: Infrastructure can be restored, but full end-to-end testing requires Audience service to be operational

6. **Configuration discovery is manual pain point**: Must contact multiple teams to obtain correct service URLs and endpoints for DR environment

7. **Expect surprises**: System hasn't been deployed from scratch in 2+ years. Overlooked dependencies will surface during exercise, but CloudFormation errors are usually clear

8. **M8 follows same pattern with one exception**: Identical process but without KMS encryption limitation since M8 doesn't use manual hashing

---

## Unresolved Questions & Action Items

⚠️ **Critical Outstanding Item: KMS Multi-Region Key Migration**
- **What**: Create multi-region KMS key and migrate all database records from single-region key
- **Why**: To enable generic connector integration verification in DR account
- **Owner**: TBD
- **Status**: Deprioritized but should be prioritized before next critical incident

⚠️ **Likely Discovery During Next DR Exercise**: Unknown interdependencies that haven't surfaced in 2+ years
- **Mitigation**: Run full DR exercise to identify and document

📝 **Documentation Needed**: Exact list of which secrets must be created manually in Secrets Manager (currently learned through deployment errors)