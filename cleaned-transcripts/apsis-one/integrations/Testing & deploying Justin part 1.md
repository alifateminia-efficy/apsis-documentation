---
source_file: Testing & deploying Justin part 1.txt
domain: Apsis One Integrations
topics: [Release Process, CI/CD Pipeline, Deployment Architecture, GitHub Actions, AWS CodeBuild, Database Migrations, Docker Image Builds]
speakers: [Erik Andersson, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [AWS CodeBuild, GitHub Actions, GitHub Pull Requests, AWS ECS, Docker Images, AWS Parameter Store, GitHub Tokens, Database DDL Folder, Make Deploy Scripts]
session_type: knowledge-transfer
subdomains: [Architecture]
---

## Session Overview

This session covers the **release and deployment process for the Apsis One Integrations platform**, focusing on the CI/CD pipeline architecture. Erik Andersson walks Tomasz Kowalski and Michal Rosikiewicz through performing an actual beta release, demonstrating how pull requests trigger automated testing and deployment workflows. The session reveals critical infrastructure decisions (why AWS CodeBuild is used instead of GitHub Actions for deployment), deployment architecture concerns (the need to build ~28-30 Docker images in parallel), and operational gotchas (manual database migration checks, GitHub token management via AWS Parameter Store, and ARM 64 architecture constraints).

---

## Release Workflow: PR-Based Deployment Pipeline

### Core Release Process

[Erik Andersson]: There are two key things to consider for releases in integration to both beta and production environments. This is done by opening a pull request to specific branches.

The release workflow operates as follows:

1. **Development to Beta**: Open a PR from the `develop` branch to the `beta` branch
2. **Beta to Production**: Open a PR from the `beta` branch to the `master` branch
3. Upon merge, automated deployment is triggered

Currently, there is **one CodeBuild project in AWS** that handles deployment to all environments.

### Why CodeBuild Instead of GitHub Actions

[Erik Andersson]: The reason we have this in CodeBuild and not in GitHub Actions is because we didn't want to have a GitLab runner that is ARM 64. Integration consists of approximately 30 ECS tasks, which are all ARM 64 architecture. If you tried to deploy integration while emulating ARM 64, the deployment took approximately 4.5 hours, which is not feasible.

**Key decision drivers:**
- Integration's entire infrastructure runs on ARM 64 ECS tasks (~30 tasks)
- Emulating ARM 64 in GitHub Actions/runners would create 4.5+ hour deployments (unacceptable)
- AWS CodeBuild natively supports ARM 64 execution
- This was a **temporary solution** that remains in place

**Contrast with MA (another platform):** MA fully uses CodeBuild and never migrated to GitHub Actions because CodeBuild was considered acceptable at the time. Integration previously used Travis CI before migrating to GitHub Actions for testing, but CodeBuild was retained for deployment.

### Future Direction

[Erik Andersson]: I am of the opinion that, for simplicity, everything here should be a GitHub action. We should do the PR checks in GitHub action, and then the deploy script should also be a GitHub action. So we reduce the tech stacks and not have to do things like having to go into AWS to configure the parameter store and set the GitHub token. We'll get away from things like that.

[Michal Rosikiewicz]: I agree.

---

## Testing and Validation

### GitHub Actions for Unit Tests

Every PR automatically runs **unit tests and validation steps** via GitHub Actions. This is separate from the CodeBuild deployment pipeline because:

> We don't need to build any Docker images here, so that is fine.

All PRs show test status in the GitHub interface. When viewing a PR's conversations/checks, you can see:
- Test pass/fail status
- CI/CD step status

### Known Issue: CGRid Test Failure

During the session, a failed test was observed in the pull request checks related to CGRid. However:

[Erik Andersson]: That's nothing to worry about because CGRid has now apparently been disabled. I have disabled this step. You see that the tests are still passing, so we can still go ahead and merge this.

Despite the failure, the test suite was passing overall and the PR was mergeable.

---

## Critical Pre-Release Checklist: Database Migrations

### Manual DDL Verification Requirement

[Erik Andersson]: Whenever you do a release for integration, one important thing to remember is the database migrations. Unfortunately, we don't have any automatic way of handling changes to the database. There are no automatic migrations executed, so you need to double check any time you do a release: are there any changes in the DDL folder?

**Process:**
1. Check the **DDL folder** (capital letters: `DDL/`) in the files changed section of the PR
2. This folder contains:
   - Database definitions
   - Any potential deltas that need to be included
3. If DDL changes exist, they must be manually verified and coordinated before merge

**Important caveat:** No automatic migrations are executed during deployment. All DDL changes require manual oversight.

---

## Deployment Execution via CodeBuild

### Build and Deployment Specification

Deployments are controlled via `deploy_spec.yaml` in the repository root. When a PR is merged to `beta` or `master` branches, CodeBuild detects the change via webhook and triggers the deployment pipeline.

### Make-Based Deployment Architecture

The deployment process uses Make targets. The root `Makefile` contains a `make deploy` target that iterates over all integration services.

**Service structure (observed):**
- Worker services
- Manager services
- Sync services (full sync manager, etc.)

**Deployment command pattern:**
```
make deploy <service_name>
```

Examples:
```
make deploy manager
make deploy worker
make deploy sync_manager
make deploy full_sync_manager
```

Each service folder contains its own Makefile that executes the actual deployment steps.

### Parallelization

[Erik Andersson]: We are deploying each and everyone of these services and I think we're doing like 3 or 4 concurrently.

The deployment builds and pushes approximately 28-30 Docker images in parallel, which significantly reduces total deployment time compared to sequential builds.

### Deployment Duration and Architecture Concerns

[Erik Andersson]: This is going to take quite a while because we are building so many Docker images like I think we have 28 or 30 of them, but we do build them in parallel. This is something me, Greg and Mashek will discuss and probably also discuss in January. Like we should change the architecture of integration a bit, maybe like combine the services into one. So like we build essentially the integration platform in one go, but that remains to be seen.

**Known issue:** The current microservices architecture requires building and deploying many small Docker images. Future architectural changes under discussion would consolidate this into a single platform build.

---

## Multi-Environment Deployment

### Beta Environment
- **Single environment**: EU West (`eu-w`) only
- One CodeBuild webhook listening for changes to `beta` branch

### Production Environment
- **Two environments deployed simultaneously**:
  - EU West (`eu-w`) production
  - APAC production
- One CodeBuild webhook listening in EU West
- One CodeBuild webhook listening in APAC
- Both webhooks trigger on changes to `master` branch
- **Both environments deployed at the same time** (not sequential)

---

## CI/CD Credentials and Token Management

### GitHub Token Configuration for Private Dependencies

Integration uses an external email validator library that is shared via a private GitHub repository. This requires authentication credentials in the CI/CD pipeline.

**Current credential management approach:**
1. A dedicated **CI/CD user** was created: `integrations-ma-cicd` in the GitHub organization
2. This user owns all GitHub personal access tokens used by the CI/CD system
3. Tokens are stored in **AWS Parameter Store** (not in the repository or GitHub Secrets)
4. CodeBuild retrieves tokens from Parameter Store at build time

**Token retrieval path:** Settings → Developer settings → Personal access tokens → Tokens (Classic)

### Known Operational Issue: MFA and Token Authentication

During the session, a token authentication error occurred:

> invalid username or token password authentication is not supported for Git operations

[Erik Andersson]: This has been running absolutely fine for months after months. Why is this happening now?

**Likely root cause** (identified by Michal Rosikiewicz): MFA may have been made mandatory for all users in the GitHub organization, which affects how tokens are validated during CI/CD operations.

**Resolution attempted:** Regenerating the GitHub token in the CI/CD user account and updating it in AWS Parameter Store.

---

## Performing a Release: Step-by-Step

### Beta Release Steps
1. Navigate to the Integration GitHub repository
2. Go to **Pull requests** tab
3. Locate the open PR from `develop` → `beta` (or create one if not present)
4. Review the **Files changed** tab
   - **Critically**: Check the DDL folder for database migrations
   - Verify all expected changes are present
5. Review the **Conversations** tab
   - Confirm all tests have passed (or note any disabled tests like CGRid)
6. Click **Merge pull request**
7. CodeBuild automatically detects the merge to `beta` branch and initiates deployment
8. Monitor the CodeBuild pipeline in AWS console (builds and deploys Docker images)
9. Deployment will target the single beta environment in EU West

### Production Release Steps
The process is identical, except:
- Create/open PR from `beta` → `master`
- Upon merge, CodeBuild deploys to **both** EU West and APAC production environments simultaneously
- Deployment duration will be longer due to building/deploying 28-30 Docker images across both regions

---

## Operational Gotchas and Limitations

### 1. External Email Validator Library Dependency
The shared private library creates a hard dependency on the CI/CD user's GitHub token being valid. Any changes to GitHub authentication policies (like mandatory MFA) can break the build.

### 2. Manual Database Migration Coordination
No automatic migrations mean:
- Risk of deploying code that expects schema changes that haven't been applied
- Requires manual verification before every release
- Developers must coordinate schema changes separately from code deployment

### 3. CodeBuild Trigger Quirks

[Erik Andersson]: For the love of God, what? That seems to be an Amazon issue.

There are occasional issues with CodeBuild trigger configuration or execution. In the session, a build was accidentally started instead of retried, requiring redeployment of the CodeBuild configuration.

### 4. Disabled GitHub Actions Deployment

A `deploy` GitHub Action exists in `.github/workflows/` but is currently **not active**. 

[Erik Andersson]: We should update this flow and re-enable it. So we use this instead of the code build. Now that we have a GitLab runner based on ARM 64.

This was marked as a future optimization task.

---

## Deployment Architecture Future Improvements

**Under discussion for January planning session** (participants: Erik, Greg, Mashek):

1. **Consolidate microservices**: Combine the ~30 separate ECS services into fewer, larger services or a single platform service
2. **Simplify CI/CD**: Migrate all deployment logic from CodeBuild to GitHub Actions now that ARM 64 runners are available
3. **Eliminate Parameter Store credential management**: Move to GitHub Secrets or native GitHub token handling
4. **Reduce Docker image build time**: Fewer services = fewer Docker images to build in parallel = potentially faster overall builds despite losing parallelization

---

## Key Takeaways

1. **Release process is PR-based**: Merge to beta or master branch triggers CodeBuild deployment automatically
2. **Database migrations are manual**: Must check DDL folder in every release PR and coordinate schema changes separately
3. **ARM 64 architecture drives CodeBuild choice**: 4.5-hour deployment time emulating ARM 64 in GitHub Actions is unacceptable; CodeBuild was chosen for native ARM 64 support
4. **Multi-environment production deployment**: Master branch merges deploy to both EU West and APAC simultaneously
5. **Credential management via Parameter Store**: GitHub tokens for private library dependencies are stored in AWS Parameter Store, not in repository or GitHub Secrets
6. **MFA can break CI/CD**: Recent organizational MFA policies may impact token authentication in the build pipeline
7. **Planned simplification**: Future goal is to consolidate microservices and move all CI/CD to GitHub Actions

---

## Unresolved Questions and Action Items

### Outstanding Issues

1. **CodeBuild trigger configuration error**: A build was accidentally triggered instead of retried. Resolution requires discussion with Prem about potential CodeBuild redeployment.

2. **GitHub token authentication failure**: MFA policy change may be breaking the CI/CD user's token authentication. Token regeneration was attempted, but root cause verification was deferred.

3. **Test runner configuration**: CGRid test was disabled, but unclear if this was a temporary measure or permanent change.

### Planned Future Work

1. **Re-enable GitHub Actions deployment workflow**: Activate the existing `deploy` GitHub Action to replace CodeBuild once architectural constraints are addressed
2. **Migrate to unified GitHub Actions pipeline**: Eliminate CodeBuild dependency for simplified tech stack
3. **Architectural consolidation meeting**: January planning session to discuss combining ~30 ECS services into fewer, larger services
4. **Credential management overhaul**: Move from AWS Parameter Store to native GitHub token handling (once CI/CD is fully in GitHub Actions)

### Follow-up Required

- Erik to verify CodeBuild configuration with Prem before next deployment attempt
- Confirm status of GitHub token regeneration and CI/CD user authentication
- Schedule architectural review meeting for January with Erik, Greg, and Mashek