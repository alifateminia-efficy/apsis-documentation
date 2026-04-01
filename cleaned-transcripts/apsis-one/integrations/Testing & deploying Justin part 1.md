---
source_file: "Testing & deploying Justin part 1.txt"
domain: Apsis One Integrations
topics: [deployment pipeline, CI/CD, AWS CodeBuild, GitHub Actions, Docker builds, ECS, database migrations, environment management, GitHub token management, ARM64 builds]
speakers: ["Erik Andersson (senior engineer/lead)", "Michal Rosikiewicz", "Tomasz Kowalski"]
key_components: [AWS CodeBuild, GitHub Actions, ECS tasks, AWS Parameter Store, LastPass, DDL migrations, email-validator library, integration services (workers/managers/sync services)]
session_type: knowledge-transfer
---

# Session Overview

Erik Andersson walks Michal Rosikiewicz and Tomasz Kowalski through the release and deployment process for the Apsis One Integrations platform. The session covers the two-phase CI/CD setup (GitHub Actions for testing, AWS CodeBuild for deployment), the reasons behind this architecture, the manual database migration check process, and the multi-environment production deployment flow. The session was interrupted mid-way by a live CodeBuild failure caused by an expired or invalid GitHub token stored in AWS Parameter Store, which remained unresolved at the end of the recording.

---

## CI/CD Architecture: Why CodeBuild Instead of GitHub Actions

The integration platform uses a **split CI/CD approach**:

- **GitHub Actions** handles unit tests and PR validation (no Docker image builds required here, so no ARM64 constraint).
- **AWS CodeBuild** handles the actual deployment of code to environments.

### Reason for CodeBuild (Historical Context)

> "Integration consists of approximately 30 ECS tasks which are all ARM64. If you tried to deploy integration while emulating ARM64, the deployment took approximately 4 to 4.5 hours, which is not feasible."

CodeBuild supported ARM64 natively at the time this decision was made, so the deployment was moved there. GitHub Actions did not have an ARM64 runner available to the team at that point.

### Current State and Known Tech Debt

A deploy GitHub Action file **does exist** in the workflows directory but is **currently inactive**. Now that a GitHub-hosted ARM64 runner is available, the team should:

1. Update the deploy workflow in GitHub Actions.
2. Re-enable it to replace CodeBuild.
3. This would eliminate the need to manage things like GitHub tokens in AWS Parameter Store separately.

[Erik Andersson]:
> "For simplicity's sake, I'm of the opinion that everything here should be a GitHub Action — we should do the PR checks in GitHub Actions, then the deploy script should also be a GitHub Action. So we reduce the tech stack."

### Comparison with MA (Marketing Automation)

- **MA** has always used CodeBuild exclusively and never used Travis or GitHub Actions.
- **Integration** previously used Travis CI, then migrated to GitHub Actions for tests — but the deployment side was moved to CodeBuild during that migration and never completed the move to GitHub Actions.
- Migrating MA to GitHub Actions is also a future consideration.

---

## Release Process: Step-by-Step

### Branch Strategy

Releases are triggered by opening and merging Pull Requests to specific branches:

- **Beta release**: merge `develop` → `beta`
- **Production release**: merge `beta` → `master`

CodeBuild has **webhooks** listening for changes on the `beta` and `master` branches. When a merge occurs, CodeBuild triggers automatically.

### Step 1: Open a Pull Request

In the integration GitHub repository, open a PR from the source branch to the target branch. Every PR automatically runs the **test and validation** GitHub Action (unit tests only — no Docker build).

### Step 2: Check for Database Migrations (CRITICAL MANUAL STEP)

> "We don't have any automatic way of handling changes to the database — there are no automatic migrations executed."

Before merging any release PR, **manually review the `DDL` folder** (all caps) in the repository:

```
DDL/
```

This folder contains:
- Database schema definitions
- Delta scripts that need to be applied manually

**You must check whether any DDL changes are present in the diff and apply them manually before or alongside the deployment.** There is no automated migration runner.

### Step 3: Check PR Status Checks

One currently failing check (`C Grid` — a now-disabled action) can be safely ignored. Verify that the unit tests pass, then proceed to merge.

### Step 4: Merge the PR

Click **Merge Pull Request**. This triggers CodeBuild via webhook.

### Step 5: Monitor CodeBuild

Navigate to AWS CodeBuild to monitor the deployment pipeline progress. The deployment:

- Builds all Docker images **in parallel** (~28–30 images)
- Deploys all services (3–4 concurrently)
- Uses **Makefiles** to orchestrate everything

---

## Deployment Internals: Makefiles and Service Structure

The CodeBuild deploy spec is defined in:

```
deployspec.yaml
```

(located in the repository root — described as a "somewhat temporary solution" that was manually deployed into CodeBuild rather than being auto-configured)

### Make Deploy Flow

The root-level `make deploy` command iterates over every service in integration. Services are organized into three categories:

- **Workers**
- **Managers**
- **Sync Services**

Example make targets:

```
make deploy IAM
make deploy MMM
make deploy DSM
make deploy full-sync-manager
# etc. for all ~28-30 services
```

Each `make deploy <service>` descends into that service's folder and executes the deployment. The service folder names correspond directly to the names used in the application.

### Architecture Discussion (Future)

[Erik Andersson]: The team (Erik, Greg, and Mashek) plans to discuss in January whether to consolidate the ~30 separate services into fewer combined services to reduce build time and complexity. This remains undecided.

---

## Production vs. Beta Environment Differences

| Aspect | Beta | Production |
|---|---|---|
| Number of environments | 1 (EU-West only) | 2 (EU-West + APAC) |
| CodeBuild webhooks | 1 | 2 (one per region) |
| Deployment trigger | Merge to `beta` branch | Merge to `master` branch |

For production, both the EU-West and APAC environments are deployed **simultaneously** via their respective webhooks. No manual per-region action is required.

---

## GitHub Token for Private Dependency: Email Validator Library

### The Dependency

Integration uses a shared **email validator library** that:

- Lives in a **private** external GitHub repository (outside the integration repo)
- Must be fetched during the CodeBuild build process

### How Authentication Works

A dedicated CI/CD GitHub user has been created:

```
integrations-ma-cicd  (user in the GitHub organisation)
```

This user's **Personal Access Token (Classic)** is stored in:

```
AWS Parameter Store
```

CodeBuild retrieves this token at build time to authenticate against the private repository.

The credentials for this user are stored in **LastPass**.

### Live Incident During This Session

The CodeBuild deployment triggered during this session failed with:

```
invalid username or token — password authentication is not supported for Git operations
```

**Suspected causes discussed:**
- The token may have expired
- MFA may have been made mandatory for all GitHub organisation users, invalidating token-based auth

**Attempted remediation:**
- Erik regenerated the Personal Access Token (Classic) with an expiry of approximately 2026-04-[date]
- The new token needed to be updated in **AWS Parameter Store**
- The build then needed to be **retried** (not started fresh — use the retry function on the existing build)

**Resolution status**: Unresolved at end of session. Erik planned to consult with Prem before reconnecting with the group.

**[⚠️ Warning]**: This token management overhead is a known pain point. Erik explicitly called out that moving to GitHub Actions would eliminate this class of problem.

---

## Key Takeaways

1. **Two-tool CI/CD**: GitHub Actions for tests (all PRs), CodeBuild for deployment (merges to `beta`/`master`). This split exists because of ARM64 build time constraints.
2. **Database migrations are fully manual**: Always check the `DDL/` folder before merging a release PR. There is no automated migration runner.
3. **~28–30 Docker images** are built per deployment, in parallel. Deployments take significant time.
4. **Production deploys to two regions simultaneously** (EU-West + APAC) via two separate CodeBuild webhooks.
5. **The `deployspec.yaml` and inactive GitHub Actions deploy workflow** represent known tech debt — the intent is to consolidate everything into GitHub Actions.
6. **A private email-validator library dependency** requires a GitHub Personal Access Token (Classic) stored in AWS Parameter Store under the `integrations-ma-cicd` user. Credentials in LastPass. This is fragile and a known operational risk.
7. **Architectural consolidation** of the ~30 services is under discussion for early 2026.

---

## Unresolved Questions and Action Items

- [ ] **[Erik/Prem]** Resolve the CodeBuild GitHub token failure — update regenerated token in AWS Parameter Store and verify build succeeds.
- [ ] **[Team]** Migrate deployment pipeline from CodeBuild to GitHub Actions (ARM64 runner now available). Re-enable the existing but inactive deploy GitHub Action.
- [ ] **[Team]** Evaluate migrating MA's CodeBuild pipeline to GitHub Actions as well.
- [ ] **[Erik/Greg/Mashek — January]** Architectural discussion: consolidate integration's ~30 services to reduce build complexity and time.
- [ ] **[Tomasz/Michal]** Complete the hands-on production release walkthrough (was set up but not completed due to the CodeBuild incident).