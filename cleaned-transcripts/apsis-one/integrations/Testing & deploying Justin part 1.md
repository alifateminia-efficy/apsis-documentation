---
source_file: Testing & deploying Justin part 1.txt
domain: Apsis One Integrations
topics: [Release process, CI/CD pipeline, Deployment to beta and production, GitHub Actions vs AWS CodeBuild, Docker image builds, Database migrations, GitHub token management, Architecture discussion]
speakers: [Erik Andersson, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [GitHub Actions, AWS CodeBuild, AWS ECS (ARM 64), GitHub repositories, Docker images, Parameter Store, Database DDL folder, Make files, E-mail validator library]
session_type: knowledge-transfer
subdomains: [Architecture]
---

## Session Overview

This session covers the complete release and deployment process for the Apsis One Integrations platform, including how pull requests trigger testing and deployment pipelines. Erik Andersson walks through the current CI/CD architecture, which uses GitHub Actions for testing and AWS CodeBuild for deployments. The session demonstrates a live release to the beta environment, highlighting infrastructure decisions, current pain points, and planned future improvements. A GitHub token authentication issue is encountered during the session, which is left for follow-up with another team member (Prem).

---

## Release Process Overview

### Two-Environment Release Flow

Releases to both **beta** and **production** environments are triggered by opening pull requests to specific branches:

- **Beta deployment**: PR from `develop` branch → `beta` branch
- **Production deployment**: PR from `beta` branch → `master` branch

Once a PR is merged to either the `beta` or `master` branch, the deployment process is automatically triggered by AWS CodeBuild webhooks listening to those branches.

### Current Testing and Deployment Architecture

**GitHub Actions** handle the test and validation steps that run on every PR:
- No Docker image building needed for unit tests
- This makes GitHub Actions appropriate for the testing phase
- All PRs automatically execute unit tests before they can be merged

**AWS CodeBuild** handles actual code deployment:
- Listens for changes on `beta` and `master` branches
- Builds and deploys all Docker images
- Configured via `buildspec.yaml` in the repository

---

## Why CodeBuild Was Chosen Over GitHub Actions for Deployment

### ARM 64 Architecture Requirement

The integration platform consists of approximately 28-30 ECS tasks, all requiring **ARM 64** architecture. This architectural choice necessitated CodeBuild rather than GitHub Actions.

> "The reason we have this in code build I'm not in GitHub actions is because we didn't use to have a GitLab runner that is arm 64 and integration consists of like, I think we're up to say 30 or so ECS tasks which are all ARM 64. If you tried to deploy integration while emulating ARM 64, the deployment took approximately 4 hours and 1/2 hours, which is like. I mean that's not feasible."

**GitLab runner ARM 64 emulation was prohibitively slow** (4.5+ hours vs. native execution with CodeBuild), making the current approach necessary. However, this is noted as a temporary solution that should be reconsidered now that native ARM 64 GitHub runners are available.

### Historical Context: Travis CI Migration

- Integration originally used **Travis CI** before migrating to GitHub Actions
- The **MA** (another service/module) has always used CodeBuild and was never migrated to GitHub Actions
- This architectural inconsistency across services is something to reconsider in the future

---

## Database Migrations: Critical Manual Step

### No Automatic Database Migration System

A critical aspect of any integration release: **there is no automatic way of handling database changes**. This requires manual verification before every release.

**Process**:
1. Before merging any release PR, check the **DDL folder** (capital letters) in the repository
2. The DDL folder contains database definitions and any delta changes that need to be applied
3. If DDL changes exist, they must be manually verified and coordinated before deployment
4. In the example PR shown during this session, no DDL changes were present (already deployed)

> "Whenever you do a release for integration and that is the database migrations unfortunately like we don't have any automatic way of handling changes to the database like there are no automatic migrations executed, so you need to double check any time you do a release like are there any changes in the DDL folder like we have a folder called DDL with Capital Capital Letters that contains the database definitions and any potential any potential deltas that we need to that we need to include"

---

## Deployment Pipeline Details

### Make Files and Service Iteration

The actual deployment is orchestrated using **Makefiles**:

```
Root folder: make deploy
└── Iterates over all services:
    - make deploy-workers
    - make deploy-managers  
    - make deploy-sync-services
    - make deploy-full-sync-manager
    - (etc. for each service)
```

Each service has its own folder containing its Makefile, and the root-level `make deploy` script iterates through them:

> "The make deploy in the root folder essentially iterates over every server service we have in integration... the make deploy scripts will do like make deploy I am make deploy, MMM, make deploy DSM full sync manager etc etc."

**Concurrent deployment**: Services are deployed 3-4 concurrently to reduce total deployment time.

### Docker Image Build Scale

- Building approximately 28-30 Docker images per release
- Images are built in parallel to reduce total time
- This scale is noted as a potential architectural concern

---

## Multi-Region Production Deployment

For production releases, both regional environments are deployed simultaneously:

- **EU West (EU-W) environment**: Has one webhook listening in AWS CodeBuild
- **APAC environment**: Has a separate webhook listening in AWS CodeBuild
- When a PR is merged to `master`, **both webhooks trigger automatically and deploy in parallel**

Beta environment currently only exists in EU-W.

---

## GitHub Token and Parameter Store Configuration

### Current Setup Issues

The integration platform uses a shared **e-mail validator library** that exists in an external private repository. This requires authentication to retrieve during the build process.

**Current configuration**:
- A dedicated **`integrations-ma-cicd` user** has been created in GitHub to own the GitHub tokens used by CodeBuild
- The token is stored in **AWS Parameter Store** (not in the repository)
- CodeBuild retrieves the token from Parameter Store during the build process

**Location in GitHub**: Settings → Developer settings → Personal access tokens → Tokens (Classic)

### Token Authentication Problem Encountered

During the session, an authentication failure occurred:
```
Error: invalid username or token
Error: password authentication is not support for Git operations
```

**Probable cause**: Mandatory Multi-Factor Authentication (MFA) was recently enabled for all users in the organization, which may have invalidated or changed the token requirements.

**Troubleshooting steps taken**:
- Verified the token existed in Parameter Store
- Confirmed the `integrations-ma-cicd` user has access to the e-mail validator library
- Attempted to regenerate the token

**Status**: This issue was not fully resolved during the session. Erik flagged this for follow-up with Prem (another team member with CodeBuild expertise). The session was paused at this point, with the first step of the release (PR merge to beta) successfully completed, but the CodeBuild deployment step blocked by the token issue.

---

## Future Improvements and Architecture Discussion

### Consolidate CI/CD to GitHub Actions Only

Erik's stated preference is to migrate the entire pipeline to GitHub Actions:

> "So in one way or another, for simplicity sake, I'm of the opinion that, like everything here should be a GitHub action like we should do the PR cheques in GitHub action. Then the deploy script should also be a GitHub action. So we reduce the tech stacks like not having to do things as we did now, like having to go into ABS to configure the parameter store and set the GitHub token like we will get get away from things like that."

**Michal agreed** with this approach. Benefits include:
- Reduced operational complexity
- No need to manage separate Parameter Store credentials
- Simpler troubleshooting
- Single tool for the entire CI/CD pipeline

**Blocker**: Only feasible now that GitHub offers native ARM 64 runners (previously impossible due to emulation performance)

### Potential Service Architecture Consolidation

A separate architectural discussion is planned with Greg and Mashek (possibly in January) regarding consolidating the 28-30 microservices:

> "This is something me, Greg and Mashek will discuss and look probably also discuss in January. Like we should change the architecture of integration a bit, maybe like combine the services into one. So like we build essentially we build the integration platform uh in one go"

Current situation: 28-30 separate Docker images built per release = long build times even with parallelization. Potential future: Single consolidated build artifact for the entire platform.

---

## Deployment Workflow Walkthrough

### Step 1: Navigate to Pull Requests
In the GitHub integration repository, locate the open PR from `develop` to `beta` branch.

### Step 2: Verify Changes
- Review the **Files Changed** tab
- **Critical**: Check for any changes in the `DDL` folder (database migrations)
- Verify tests are passing (note: C-Grid tests may be disabled, but other tests must pass)

### Step 3: Merge the PR
- Click "Merge pull request" when ready
- This triggers the CodeBuild webhook automatically

### Step 4: Monitor Deployment in CodeBuild Console
- CodeBuild will start building Docker images (~28-30 of them)
- Images build in parallel, but the overall process takes significant time
- Both EU-W and APAC environments are deployed if releasing to production (master branch)

### Step 5: For Production Release
- Once beta release is complete and verified, create a new PR from `beta` → `master`
- Follow the same process (verify, merge, wait for CodeBuild)
- Production deployment will auto-trigger to both EU-W and APAC environments via webhooks

---

## Known Gotchas and Warnings

### Test Failures to Ignore
During the session, a C-Grid test failed. Erik clarified:

> "we have one failed test here. That's however nothing to worry about because C grid has now apparently been disabled. So I have disabled this step."

The test step has been disabled in the GitHub Actions configuration, so this failure doesn't block merging, but other tests must pass.

### CodeBuild Console UI Issues
The AWS CodeBuild console has UX issues that can be confusing:
- Erik accidentally triggered a new build instead of retrying the failed one
- When redeploying CodeBuild, be careful to "retry" the existing build rather than starting a new one

### External Dependencies
The shared **e-mail validator library** in a private external repository creates fragility:
- Requires GitHub token in Parameter Store
- Token management is error-prone (as demonstrated by the MFA issue)
- **Future direction**: Consider whether this library should be maintained internally or licensed differently

---

## Key Takeaways

1. **Release process is straightforward once working**: Create PR to beta/master branch → merge → CodeBuild automatically handles deployment
2. **Manual DDL verification is essential**: Always check the DDL folder before releasing to catch database schema changes
3. **Current architecture is pragmatic but temporary**: Using CodeBuild for ARM 64 deployment was necessary but should migrate to GitHub Actions now that native ARM 64 runners exist
4. **GitHub token management is a pain point**: Parameter Store configuration works but is fragile, especially with mandatory MFA. Moving to GitHub Actions would simplify this
5. **Service architecture needs review**: 28-30 microservices creating 28-30 Docker images per release is inefficient. Consolidation is being considered
6. **Token regeneration may be needed**: If you see "invalid username or token" errors, the GitHub token in Parameter Store may need regeneration, especially if MFA was recently enabled organization-wide

---

## Unresolved Issues and Action Items

1. **[BLOCKING]** GitHub token authentication failure in CodeBuild
   - Error: "invalid username or token" / "password authentication is not supported for Git operations"
   - Probable cause: Organization-wide mandatory MFA
   - **Action**: Erik to follow up with Prem on CodeBuild configuration and token regeneration
   - **Status**: Session paused pending this investigation

2. **[FUTURE]** Re-enable GitHub Actions deployment workflow
   - Current `deploy` GitHub Action in workflows directory is disabled
   - Should be re-enabled once ARM 64 runner support is confirmed stable
   - Would eliminate need for CodeBuild and Parameter Store token management
   - **Owners**: Erik, Michal (agreed this is the right direction)

3. **[FUTURE - January]** Architecture review with Greg and Mashek
   - Evaluate consolidating 28-30 microservices into fewer, larger services or a monolith
   - Current 28-30 Docker image builds per release is inefficient
   - Would significantly reduce build times

4. **[FUTURE]** Audit MA (other service) CI/CD approach
   - MA uses CodeBuild exclusively, not GitHub Actions
   - Reason unclear - may be historical. Should be documented and potentially aligned with integration approach