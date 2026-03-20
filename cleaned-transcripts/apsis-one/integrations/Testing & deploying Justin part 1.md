---
source_file: Testing & deploying Justin part 1.txt
domain: Apsis One - Integrations
topics: [Release Process, CI/CD Pipeline, GitHub Actions, AWS CodeBuild, Docker Deployment, Database Migrations, ARM64 Architecture]
speakers: [Erik Andersson, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [GitHub Actions, AWS CodeBuild, ECS Tasks, Docker Images, Parameter Store, GitHub Personal Access Tokens, Make Deploy Scripts]
session_type: knowledge-transfer
---

## Session Overview

This session covers the end-to-end release and deployment process for the Integrations platform. Erik Andersson walks Tomasz Kowalski and Michal Rosikiewicz through triggering a beta release, including the rationale behind the hybrid CI/CD architecture (GitHub Actions + AWS CodeBuild), critical database migration checks, and the deployment workflow across multiple environments (EU West and APAC production). The session includes a live demonstration of opening a PR, merging it, and observing the CodeBuild deployment pipeline in action, with troubleshooting of GitHub token authentication issues encountered mid-session.

---

## Release Architecture: GitHub Actions + AWS CodeBuild Hybrid Approach

### Why Not Pure GitHub Actions?

The Integrations platform uses a **hybrid CI/CD setup** rather than GitHub Actions alone. Here's the critical constraint:

**The problem:** Integration consists of approximately 28–30 **ECS tasks, all ARM 64 architecture**. When attempted to build Docker images for ARM 64 using a GitLab runner that would have to emulate ARM 64, the deployment took approximately **4.5 hours**—which is not feasible.

**The solution:** AWS CodeBuild was introduced specifically because it **natively supports ARM 64** hardware, eliminating the need for emulation. This made the deployment process practicable.

> [Erik Andersson]: "If you tried to deploy integration while emulating ARM 64, the deployment took approximately 4 hours and 1/2 hours, which is like. I mean that's not feasible. So what we did is we temporarily added the flow to a code build. Which did support ARM 64."

### Current Workflow Division

**GitHub Actions handles:**
- Unit tests and validation on every PR (no Docker image building required, so GitHub Actions runs efficiently)
- This step runs on all PRs to both `develop` → `beta` and `beta` → `master`

**AWS CodeBuild handles:**
- Docker image builds (28–30 images built in parallel)
- Actual deployment to ECS tasks
- Triggered by changes detected on the `beta` and `master` branches via webhooks

### Deploy Specification

AWS CodeBuild uses a **deploy spec file**:
```
deploy spec dot YAML
```

This configuration sits in the repository root and defines the build and deployment steps.

### Future Consolidation Plan

[Erik Andersson] notes there is an **inactive GitHub Actions deploy workflow** in the `workflows` directory that should be re-enabled and used instead of CodeBuild now that a **GitLab runner with ARM 64 support** is available. This is flagged as technical debt to address soon, potentially during a January discussion with Greg and Mashek about a broader architectural change.

> [Erik Andersson]: "This is also one of the small things is we should things we should do like we should update this flow and re enable it. So we use this instead of the code build."

The preference is eventually to consolidate everything into GitHub Actions to reduce tech stack complexity.

### Context: Why Integration ≠ MA (Marketplace)

MA (Marketplace) still uses CodeBuild exclusively and has not migrated to GitHub Actions. Historical context: Integration previously used **Travis CI** before migrating to GitHub Actions, while MA had always used CodeBuild. When Travis was deprecated, the team migrated Integration but kept MA on CodeBuild as it was "considered OK" at the time. This is another potential future consolidation point.

---

## Release Process: Step-by-Step Workflow

### Creating a Release PR

To release code to beta or production, you must:

1. Navigate to the **Integrations GitHub project**
2. Open a **pull request** from the `develop` branch to the `beta` branch (for beta release) or from `beta` to `master` (for production release)
3. Every PR automatically triggers **GitHub Actions test and validation**

### Critical: Database Migration Checks

**Before merging any release PR, you must verify whether any database changes are included.**

Integration does **not have automatic database migrations**. Manual verification is required:

- Check the `files changed` tab in the PR
- Look for any changes in the **`DDL` folder** (capital letters: `DDL`)
  - This folder contains database schema definitions and delta migrations
  - Any files changed here must be carefully reviewed and understood before deployment
  
> [Erik Andersson]: "Whenever you do a release for integration and that is the database migrations unfortunately like we don't have. Any automatic way of handling changes to the database like there are no automatic migrations executed, so you need to double check any time you do a release like are there any changes in the DDL. Folder like we have a folder called DDL with Capital Capital Letters that contains the database definitions."

In the example shown, the `DDL` folder had no changes because migrations had already been deployed.

### Reviewing Test Status

The **Conversations tab** in the PR shows test results:
- Look for failed tests
- In the example session, one **C Grid test failed**, but this was disabled intentionally and could be ignored
- The key is that unit tests still passed overall, so the merge could proceed

### Merging the PR

Once you've verified:
- Tests are passing (or failures are expected/disabled)
- No unexpected DDL changes (or they are understood and intentional)

Simply click **Merge pull request**.

---

## CodeBuild Deployment Pipeline

### Automatic Triggering

Once the PR is merged to `beta` or `master`:
1. CodeBuild detects the branch change via webhook
2. Deployment pipeline starts automatically (no manual trigger needed)
3. For **beta**: one webhook in EU West listens for changes
4. For **production**: webhooks in both **EU West** and **APAC** listen, deploying to both environments simultaneously

### Build and Deploy Sequence

The CodeBuild pipeline executes:

1. **Build step** (from Makefile):
   ```
   make build
   ```
   Compiles the codebase

2. **Deploy step** (from root Makefile):
   ```
   make deploy
   ```
   This is a composite command that iterates over every service in the Integrations platform

### Service Deployment Structure

The root Makefile's `make deploy` command calls service-specific deploy targets:

```
make deploy im           # IM service
make deploy mmm          # MMM service
make deploy dsm          # DSM service
make deploy full sync manager
[and others...]
```

Each service has its own folder with its own Makefile and deployment logic. The root `make deploy` script orchestrates deploying **multiple services concurrently** (approximately 3–4 services in parallel).

### Deployment Duration

Building and deploying all 28–30 Docker images and services takes **a significant amount of time**. The exact duration was not specified in this session, but participants understood it as "quite a while."

---

## GitHub Token and Authentication Management

### CICD User Configuration

The platform uses a dedicated **GitHub CICD user** rather than individual developer accounts:
- User name: `integrations-ma-cicd` (in the GitHub organization)
- This user owns all GitHub personal access tokens used by the CI/CD pipeline
- Credentials are stored in **LastPass**

### Token Storage Location

GitHub tokens are stored in **AWS Parameter Store** (not hardcoded in the repository):
- Access: AWS Console → Parameter Store
- This allows centralized, secure token management
- The token is referenced by the CodeBuild process when needed

### Private Dependency: Email Validator Library

Integration uses an external email validator library:
- The library is **shared but resides outside the repository**
- The repository is **private**
- Therefore, the CodeBuild process (and any CI/CD user) **must have a valid GitHub token** to retrieve this private dependency during builds

### Token Management Procedure

To access and manage tokens:
1. Log in to GitHub as the `integrations-ma-cicd` user
2. Navigate to **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (Classic)**
3. View or regenerate tokens as needed

### Authentication Issue Encountered in Session

During the live demonstration, a token authentication error occurred:
```
Invalid username or token. Password authentication is not supported for Git operations.
```

**Likely causes discussed:**
- Mandatory **Multi-Factor Authentication (MFA)** is now required for all users in the GitHub organization
- MFA prevents password-based authentication
- The token may have expired or been invalidated

**Resolution attempted:**
- [Erik Andersson] regenerated the token in Parameter Store
- The token's expiration date was extended (set to approximately one year from the session date: 2026-04-06 or similar)
- Note: The exact new expiration date in the transcript is unclear, but the procedure is to set it well into the future

---

## Deployment Architecture: Service-Based Parallel Deployment

### Services Deployed

The Integrations platform consists of multiple independent services:
- **IM** (Integration Manager?)
- **MMM** (unclear acronym)
- **DSM** (unclear acronym)
- **Full Sync Manager**
- **Workers** services
- **Sync services**

(Exact service names and purposes were not fully detailed in this session.)

### Parallel Deployment

Services are deployed in **parallel batches** (3–4 at a time) rather than sequentially, reducing overall deployment time.

### Architectural Concern: Monolithic Docker Builds

[Erik Andersson] and team are considering a **significant architectural change**:

> [Erik Andersson]: "This is something me, Greg and Mashek will discuss and look probably also discuss in January. Like we should change the architecture of integration a bit, maybe like combine the services into one. So like we build essentially. We we build the integration platform, uh in one go."

**Current state:** ~28–30 individual Docker images built separately
**Proposed state:** Consolidate into a single Docker image or fewer images built as one cohesive platform
**Benefit:** Significantly faster builds and simpler CI/CD
**Status:** Discussion pending in January with Greg and Mashek

---

## Environment Strategy

### Beta Environment
- **Single region:** EU West only (`EUW`)
- Used for testing releases before production

### Production Environments
- **Two regions (deployed simultaneously):**
  - EU West (`EUW`)
  - APAC (Asia-Pacific)
- Both receive the same code and are updated in parallel via separate webhooks

---

## Future Improvements and Technical Debt

### GitHub Actions Migration

The team plans to **consolidate the entire CI/CD pipeline into GitHub Actions**:
- Move PR validation checks from GitHub Actions (current) to GitHub Actions (no change)
- Move deployment from CodeBuild to a new GitHub Actions workflow
- Benefits: Single tech stack, no need to configure AWS CodeBuild or Parameter Store, simpler maintenance
- Timeline: Likely discussed in January meetings

### Inactive Deploy Workflow

An inactive GitHub Actions deployment workflow already exists in the repository:
```
.github/workflows/deploy.yaml  (or similar)
```
This was not enabled because CodeBuild was considered sufficient. Re-enabling and updating it is flagged as a task.

### Service Architecture Consolidation

Combining 28–30 micro-services into fewer, larger services (or one monolithic image) was noted as a potential January discussion point with Greg and Mashek.

---

## Troubleshooting and Gotchas Encountered

### Issue 1: GitHub Token Invalid After MFA Rollout

**Problem:** Token authentication failed mid-deployment with the message "Password authentication is not supported for Git operations."

**Root cause:** Mandatory MFA enforcement on the GitHub organization made the existing token invalid or inaccessible.

**Workaround:** Regenerate the token in Parameter Store.

**Lesson:** When organization-wide security policies change (e.g., MFA mandatory), CICD tokens may need to be regenerated. This is not always obvious and caused confusion because "this has been running fine for months."

> [Erik Andersson]: "I I get so tired when stuff of this happened like this has been running absolutely fine for months after months."

### Issue 2: CodeBuild Pipeline Failure

During the live demo, the CodeBuild deployment pipeline encountered an unspecified error. [Erik Andersson] indicated:
- The error appeared to be an **Amazon (AWS) issue**, not a configuration problem
- He needed to **retry the build** (not start a completely new one)
- Involvement of Prem was needed to diagnose further

At this point, the session was paused to resolve the infrastructure issue.

---

## Live Demonstration Walkthrough

### Steps Performed (Beta Release)

1. **Opened PR list** in the Integrations GitHub repository
2. **Reviewed an existing open PR** from `develop` → `beta`
3. **Checked files changed** to verify no DDL folder changes
4. **Reviewed conversations tab** to verify tests passed (ignoring the disabled C Grid test)
5. **Clicked merge pull request**
6. Observed CodeBuild pipeline start automatically in AWS console
7. Watched Docker image builds begin (28–30 images in parallel batches)

### Steps NOT Completed (Production Release)

The same process would be used for production, but the PR would be from `beta` → `master` instead of `develop` → `beta`. The example was not executed due to the token issue requiring investigation.

---

## Key Takeaways

1. **Hybrid CI/CD is necessary here:** GitHub Actions alone cannot efficiently build ARM 64 Docker images; AWS CodeBuild provides native ARM 64 support. However, this introduces complexity that should eventually be consolidated into GitHub Actions once a suitable runner is available.

2. **Database migrations are manual:** Always check the `DDL` folder before merging a release PR. There is no automatic migration system; missed DDL changes can break production.

3. **CICD user credentials are critical:** The dedicated `integrations-ma-cicd` GitHub user is the single point of failure for CI/CD. Its token must be kept valid, securely stored in Parameter Store, and regenerated if organization-wide security policies change.

4. **Parallel deployment saves time:** Building ~28–30 Docker images and deploying services in parallel (3–4 concurrently) keeps the pipeline reasonable. Future consolidation should maintain parallelism.

5. **Production deploys to two regions simultaneously:** A single merge to `master` triggers parallel deployments to both EU West and APAC via separate webhooks.

6. **Architecture change pending:** The team intends to discuss consolidating the service architecture and CI/CD pipeline in January. Current setup with 28–30 micro-services and split CodeBuild/GitHub Actions is seen as temporary.

7. **Tech stack reduction is a goal:** Consolidating to GitHub Actions only (removing CodeBuild dependency) is desired to simplify operations and reduce configuration complexity.

---

## Unresolved Issues and Action Items

1. **CodeBuild Pipeline Error (In-Session):** The CodeBuild deployment pipeline encountered an AWS-related error mid-session. Erik Andersson needed to consult with Prem to diagnose. Status: Paused, awaiting follow-up.

2. **GitHub Token Regeneration:** A CICD user token was regenerated due to MFA enforcement. This should be tested in the next deployment to ensure it resolves the authentication issue.

3. **Re-enable GitHub Actions Deploy Workflow:** The inactive deploy workflow in `.github/workflows/` should be updated and re-enabled once the CICD user issue is resolved. This is a prerequisite to the full migration away from CodeBuild.

4. **Architecture Consolidation Discussion:** Greg, Mashek, and Erik plan to discuss consolidating Integrations services into fewer Docker images/services. Scheduled for January.

5. **MA Consolidation:** Consider migrating MA (Marketplace) from CodeBuild to GitHub Actions as well, for consistency. Status: Not yet discussed.