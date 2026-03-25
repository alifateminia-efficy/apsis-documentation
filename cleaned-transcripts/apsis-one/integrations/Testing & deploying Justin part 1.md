---
source_file: Testing & deploying Justin part 1.txt
domain: Apsis One Integrations
topics: [Release Process, CI/CD Pipeline, Deployment Architecture, GitHub Actions, AWS CodeBuild, Database Migrations, Docker Image Building]
speakers: [Erik Andersson, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [GitHub Actions, AWS CodeBuild, AWS ECS, Docker, Parameter Store, GitHub Tokens, Makefiles, DDL Folder, E-mail Validator Library]
session_type: knowledge-transfer
subdomains: [Architecture]
---

## Session Overview

This knowledge transfer session covers the end-to-end release and deployment process for the Apsis One Integrations platform. Erik Andersson walks Tomasz Kowalski and Michal Rosikiewicz through the mechanics of opening pull requests, triggering CI/CD pipelines, handling database migrations, and deploying code to beta and production environments across multiple regions. The session includes a live walkthrough of performing an actual release to the beta environment, revealing both the intended workflow and some operational pain points with the current tooling.

---

## Release Workflow Overview

### Pull Request-Based Deployments

**Two primary deployment flows exist:**

1. **Beta Environment**: Pull requests from the `develop` branch to the `beta` branch trigger deployment to the beta environment
2. **Production Environment**: Pull requests from the `beta` branch to the `master` branch trigger deployment to both EU West and APAC production environments

[Erik Andersson]: "In order to do releases in integration both to the beta environment and to the production environments, this is done by opening a pull request to the specific branches."

Every PR automatically triggers a test and validation step via GitHub Actions before deployment can proceed.

### Environment Scope

- **Beta**: Single environment in EU West only
- **Production**: Two concurrent deployments — EU West production and APAC production environments, both triggered simultaneously via separate webhooks listening in each region

---

## CI/CD Architecture: GitHub Actions vs. AWS CodeBuild

### Historical Context and Design Rationale

The deployment pipeline currently uses a **hybrid approach** combining both GitHub Actions and AWS CodeBuild. This was a pragmatic solution to a specific infrastructure constraint.

[Erik Andersson]: "The reason we have this in CodeBuild and not in GitHub Actions is because we didn't want to have a GitLab runner that is ARM 64. Integration consists of approximately 30 ECS tasks which are all ARM 64. If you tried to deploy integration while emulating ARM 64, the deployment took approximately 4.5 hours, which is not feasible."

**Current split:**
- **GitHub Actions**: Unit tests, PR validation (no Docker image building required)
- **AWS CodeBuild**: Docker image building and deployment (native ARM 64 support)

### Deployment Configuration Files

The deployment specification is defined in:
```
deploy_spec.yaml
```

Located in the integration repository with corresponding Makefile-based deployment logic.

### Technical Debt and Future Plans

[Erik Andersson]: "This is a somewhat temporary solution. We have deployed this ourselves — there is no automatic deployment of it. We also have a deploy GitHub action here in the workflows, but it's not active. This is one of the small things we should do — we should update this flow and re-enable it now that we have a GitLab runner based on ARM 64."

The team recognizes that with ARM 64 GitLab runners now available, the reliance on CodeBuild can be reconsidered. A future improvement would be to consolidate everything into GitHub Actions, reducing the number of systems to manage.

### Comparison with Other Platforms (MA)

[Erik Andersson]: "I think you had this for MA as well, and MA fully uses CodeBuild and not GitHub Actions. That is because integration used to have Travis before we migrated to GitHub Actions, but MA has always had CodeBuild and we were not allowed to keep Travis — we migrated that to GitHub Actions. But MA — CodeBuild was considered OK, so we never migrated MA to use GitHub Actions. Also something to consider for the future, maybe."

This shows organizational inconsistency in CI/CD tooling that should be addressed.

---

## Database Migrations: Manual Process

### Critical Release Checklist Item

**Whenever you do a release for integration, you must manually verify database migration requirements.** This is currently not automated.

[Erik Andersson]: "Whenever you do a release for integration, that is the database migrations. Unfortunately we don't have any automatic way of handling changes to the database like there are no automatic migrations executed, so you need to double check any time you do a release — are there any changes in the DDL folder?"

### DDL Folder Structure

Database definitions and change deltas are stored in:
```
DDL/ (Capital letters)
```

This folder contains:
- Database schema definitions
- Any delta/migration files that need to be applied

**Release Step**: Before merging a PR, always check the "Files Changed" tab in GitHub to see if the DDL folder contains any new changes. If it does, those migrations must be executed as part of the deployment process (though the current mechanism is manual/out-of-band).

---

## Docker Image Building and Deployment

### Scale of the Build

The integration platform consists of approximately **28-30 Docker images** that must be built and deployed. These are built in parallel to reduce total deployment time.

[Erik Andersson]: "We are building so many Docker images like I think that we have 28 or 30 of them, but we do build them in parallel."

### Service Architecture

The integration platform comprises multiple service categories:

- **Workers**
- **Managers**
- **Sync Services** (e.g., Full Sync Manager)
- **DSM** (Data Service Manager)

### Deployment via Makefiles

Deployment is orchestrated through Makefile commands in the root folder:

```
make build     # Builds the code
make deploy    # Iterates over each service
```

The root `make deploy` script iterates over every service in the integration platform, calling:
```
make deploy-<service-name>
```

For example:
```
make deploy-full-sync-manager
make deploy-workers
make deploy-managers
```

Each service has its own folder with a local deployment script that handles its specific deployment logic.

[Erik Andersson]: "The make deploy in the root folder essentially iterates over every service we have in integration, so here you see like we have the workers and the managers and the workers on the sync services. So like the make deploy scripts will do like make deploy, full sync manager, make deploy DSM, full sync manager etc. These then regulate the names here in the app. So like when we do make deploy for the full sync manager it will go into the full sync manager folder and do the deployment."

**Parallelization**: The team builds 3-4 services concurrently during deployment.

### Architectural Improvement Proposal

[Erik Andersson]: "This is something me, Greg and Mashek will discuss and look probably also discuss in January. Like we should change the architecture of integration a bit, maybe like combine the services into one. So like we build essentially the integration platform in one go, but that remains to be seen."

The current 28-30 separate images increase build and deployment time significantly. A future improvement would consolidate these into a single platform build.

---

## Live Release Walkthrough (Beta Environment)

### Step 1: Navigate to Pull Requests

In the integration repository's GitHub interface, locate the pull requests tab and find the open PR from `develop` to `beta` branch.

### Step 2: Verify Files Changed

Before merging, review the "Files Changed" tab to check:
- Any DDL folder modifications requiring manual migration (in this case, none were present because migrations had already been deployed)
- Verify that test changes don't introduce breaking changes

### Step 3: Handle Test Failures

[Erik Andersson]: "Now if you go to conversations and down you see here we have one failed test here. That's however nothing to worry about because CodeGrid has now apparently been disabled. I have disabled this step. But you see that the tests are still passing, so we can still go ahead and merge this."

**Important**: Not all test failures block deployment. In this case, a CodeGrid test was failing because the service had been disabled. The rest of the test suite passed, so the release could proceed. However, this requires human judgment — always verify the failure is understood and acceptable.

### Step 4: Merge the Pull Request

Click "Merge pull request" in the GitHub UI.

[Tomasz Kowalski]: "Usually just click on merge, merge pull request."

### Step 5: CodeBuild Triggers Automatically

Once the PR is merged to the `beta` branch, AWS CodeBuild automatically detects the change via webhook and begins the deployment process.

[Erik Andersson]: "Now you have done the release for integration because what will now happen like very, very shortly is that inside AWS... CodeBuild will trigger."

### Step 6: Monitor the Build

The CodeBuild pipeline will:
1. Build all ~28-30 Docker images (in parallel)
2. Deploy all services to the beta environment
3. This process takes "quite a while" due to the scale of images being built

[Erik Andersson]: "It's in progress and this is going to take quite a while because we are building so many Docker images like I think that we have 28 or 30 of them, but we do build them in parallel."

### Production Release Process

The process for releasing to production is identical, except:
1. You create a PR from `beta` to `master` branch instead
2. CodeBuild triggers deployment to both EU West and APAC production environments simultaneously
3. Both regional webhooks fire at the same time, so deployments happen in parallel

---

## GitHub Tokens and CICD User Credentials

### Private Dependency Issue

The integration codebase depends on an external e-mail validator library that is shared but exists in a private repository:

[Erik Andersson]: "We utilise this e-mail validator library that is shared, but of course because this library exists outside of the repository and the repository is private, we need to have a GitHub token for the CICD user so we can retrieve it."

### CICD User Setup

A dedicated **integrations-ma-cicd** GitHub user has been created to own and manage GitHub tokens used by the CI/CD pipeline.

**Token Management Location**: 
- GitHub UI → Settings → Developer Settings → Personal Access Tokens → Tokens (Classic)
- Tokens are stored in **LastPass** for retrieval
- Tokens are configured in **AWS Parameter Store** for use by CodeBuild

### Token Authentication Issue Encountered

During the knowledge transfer session, the team encountered an authentication error:

```
invalid username or token
password authentication is not supported for Git operations
```

**Cause Hypothesis**: The GitHub organization likely has **mandatory MFA (Multi-Factor Authentication)** enabled for all users. This can cause password-based authentication to fail even if the token itself is valid.

[Michal Rosikiewicz]: "Maybe we have MFA right now that is mandatory for all the users."

**Resolution**: The token needed to be regenerated. However, if the token is still technically valid, the issue may require checking the Parameter Store configuration or reconfiguring the GitHub token in the CodeBuild environment.

[Erik Andersson]: "I can regenerate, but if it is still valid, that shouldn't be the problem... I don't know if we need to sit here with everyone looking at this. I'll still regenerate it and let's see what happens."

---

## Operational Pain Points and Known Issues

### CodeBuild Parameter Store Complexity

[Erik Andersson]: "I get so tired when stuff like this happens. This has been running absolutely fine for months and months."

The current setup requires manual configuration of GitHub tokens in AWS Parameter Store, which creates friction when tokens expire or need regeneration. This is precisely why consolidating to GitHub Actions is desirable.

### Token Expiration and Regeneration

Tokens have expiration dates (noted as "one year from now" in the session, with dates like 2026-04-06 being discussed). When tokens approach expiration, they must be manually regenerated and updated in Parameter Store.

### Build Retry Complexity

When a CodeBuild deployment fails, retrying requires care to avoid starting a completely new build from scratch. [Erik Andersson] encountered this issue and noted: "I started it from the wrong place. I need to actually retry the build and not start a completely new one."

---

## Recommended Architecture Changes

### Consolidate to GitHub Actions

[Erik Andersson]: "In one way or another, for simplicity sake, I'm of the opinion that like everything here should be a GitHub action. Like we should do the PR checks in GitHub action. Then the deploy script should also be a GitHub action. So we reduce the tech stacks — not having to do things as we did now, like having to go into AWS to configure the parameter store and set the GitHub token. We will get away from things like that."

[Michal Rosikiewicz]: "I agree."

**Benefits**:
- Single tool for all CI/CD operations
- Eliminate Parameter Store configuration step
- Reduce operational surface area
- Simplify token management

**Blockers**: Removed now that ARM 64 GitLab runners are available.

### Consolidate Service Architecture

[Erik Andersson]: "This is something me, Greg and Mashek will discuss and probably also discuss in January. Like we should change the architecture of integration a bit, maybe like combine the services into one. So like we build essentially the integration platform in one go."

**Current state**: 28-30 separate Docker images built independently
**Proposed state**: Single consolidated platform build
**Impact**: Significant reduction in build and deployment time

This discussion is planned for January planning sessions.

---

## Key Takeaways

1. **Releases are PR-driven**: Beta releases via `develop` → `beta` PR; production releases via `beta` → `master` PR
2. **Manual database migration checks are required**: Always inspect the DDL folder for changes before merging
3. **CI/CD uses a hybrid approach**: GitHub Actions for testing, CodeBuild for building/deploying (because of ARM 64 requirements)
4. **Docker image scale is significant**: ~28-30 images built in parallel; build times are substantial
5. **Deployment is Makefile-orchestrated**: Root Makefile iterates over service-specific Makefiles for each component
6. **CICD tokens require careful management**: Dedicated GitHub user, Parameter Store storage, expiration dates must be tracked
7. **Architecture improvements are planned**: Team wants to consolidate to GitHub Actions and combine service architecture
8. **Test failures don't always block releases**: Human judgment required to determine if a failure is acceptable to proceed
9. **Production deployments are multi-region**: EU West and APAC are deployed simultaneously via webhooks

---

## Unresolved Issues and Action Items

1. **CodeBuild authentication failure** [IMMEDIATE]: Erik to regenerate GitHub token in Parameter Store and retry the build; may require discussion with Prem about CodeBuild configuration
2. **GitHub Actions deployment workflow re-enablement** [PLANNED]: Update and re-enable the deploy GitHub action workflow now that ARM 64 GitLab runners are available
3. **Database migration automation** [PLANNED]: Design and implement automatic database migration handling (currently manual, error-prone)
4. **Consolidate to GitHub Actions exclusively** [PLANNED - January]: Eliminate CodeBuild dependency once ARM 64 runners are confirmed stable
5. **Service architecture consolidation** [PLANNED - January]: Discussion with Greg and Mashek regarding combining 28-30 Docker images into unified platform build

---

## Technical Details to Remember

- **AWS CodeBuild Project**: Listens to `beta` and `master` branch changes via webhooks
- **Makefile locations**: Root `Makefile` in integration repository; service-specific Makefiles in each service folder
- **ARM 64 constraint**: ~30 ECS tasks require ARM 64; emulation would take 4.5+ hours
- **Build parallelization**: 3-4 services built concurrently
- **Token storage**: LastPass for credentials, AWS Parameter Store for CodeBuild access
- **Webhook regions**: Separate webhooks for EU West and APAC production environments