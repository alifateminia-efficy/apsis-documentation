---
source_file: Testing & deploying Justin part 2.txt
domain: Apsis One Integrations
topics: [Deployment Process, GitHub Token Management, Consent Event Testing, Full Sync vs Real-time Sync, Acceptance Testing Strategy, Legacy vs Generic Connectors]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [AWS CodeBuild, AWS CloudFormation, GitHub Token Authentication, Parameter Store, Efficy Enterprise 12.0, Efficy Enterprise 12.1, Generic Connectors, Consent Events, Full Sync, Real-time Webhooks, Delta Sync Manager, Beta Environment]
session_type: debugging-session
subdomains: [Architecture, Generic Connector, Outbound Flow, Efficy Enterprise 12.0, Efficy Enterprise 12.1, Tribe, E-deal (Efficy Corporate)]
---

## Session Overview

This session captures a live debugging and deployment verification session focused on testing consent event functionality across different Efficy connector versions. Erik Andersson walks through a multi-service deployment to the beta environment via AWS CodeBuild and CloudFormation, then performs acceptance testing to verify that consent events (opt-in/opt-out) are properly captured and displayed on contact timelines. Key issues discussed include GitHub token rotation failures, webhook delivery problems from Efficy instances, and the importance of testing both legacy (12.0) and generic connector implementations before release.

---

## Deployment Issues and Token Management

### GitHub Token Authentication Problem

[Erik Andersson]: The core issue encountered during deployment was related to GitHub token authentication having two distinct locations in the deployment process:

1. **Parameter Store reference** in the deploy spec file, where the GitHub token environment variable is set
2. **CodeBuild project source provider credentials**, where the token is also configured

The problem arose because [Erik Andersson] rotated the GitHub token but only updated it in one location:

> I updated the token for this step inside the deployment file. However, I had not updated the token where we download the source code after the web trigger in the first place, so it never even got here where it tried to utilize the GitHub token value.

When CodeBuild downloads the entire source code from Git initially (before any deployment steps execute), it uses the credentials configured in the build project's source provider settings. This step failed silently because the token was outdated, preventing the pipeline from ever reaching the later deployment steps where the updated token in Parameter Store would have been used.

### Resolution

[Erik Andersson]: To fix this, the following steps were necessary:
- Navigate to "Manage Account Credentials" in CodeBuild
- Disconnect the old GitHub token credential
- Re-add the new token credential

### Why This Architecture Is Problematic

[Erik Andersson]: This dual-location token requirement is one of the reasons the team should consider moving away from this CodeBuild-based deployment approach to GitHub Actions, where credentials can be managed in a single location.

---

## Deployment Architecture and Parallelization

### Multi-Service Deployment Flow

The deployment project in CodeBuild is structured to build and deploy every service in parallel (three to four services at a time). The process:

1. CloudFormation manages the infrastructure-as-code deployment
2. All services are built and deployed concurrently
3. The entire deployment is expected to take approximately 10 minutes to complete
4. Upon successful completion, all services in the beta environment are updated

---

## Consent Events Testing and Validation

### Testing Objectives

After the deployment completes, the primary acceptance test is to verify that **consent events appear correctly on contact timelines**. Consent events include:

- `FSC Enterprise 12.1 consent opt in`
- `FSC Enterprise 12.1 consent opt out`
- Channel specification (email or SMS)

### Consent Event Sources

Consent updates can arrive in two ways:

1. **Full sync**: Periodic bulk synchronization of all data from the CRM
2. **Real-time webhooks**: Immediate notifications when consent changes in the CRM system

Both pathways should result in consent events appearing on the contact timeline in Apsis One.

### Testing Approach for Different Connector Types

[Erik Andersson] emphasizes a critical testing strategy:

> You need to check the functionality for one of the generic connectors, so like FC Enterprise 12.1, Tribe, Microsoft Dynamics by Site Shop or E Deal. Because if it works for one of those, then it is working as it should on Apsis side, because the logic for us is the same regardless of which of those integration it is.

**Enterprise 12.0 Testing (Legacy Connector)**

- Must always be explicitly tested because it is a legacy connector with custom functionality
- Most major customers still use FSC 12.0
- These customers have historically had low tolerance for issues: they escalate directly to leadership when problems occur
- Testing on 12.0 is mandatory because the codebase is different from generic connectors

**Generic Connector Testing**

The following are all built on the generic connector framework:
- Efficy Enterprise 12.1
- Tribe
- Microsoft Dynamics
- E-deal (Efficy Corporate)

[Erik Andersson]: 

> If it works for one of them in Apsis then it works for all of them in Apsis. If it is not working then it is like a 99.99% probability that the CRM is doing something weird.

The key insight: all generic connectors run on the same code path and flow, so testing one generic connector variant is sufficient to validate the feature works across all generic connector variants.

---

## Live Testing: Staging Environment Validation

### Full Sync Verification

The team sets up a test profile in the staging environment and verifies consent event behavior:

1. Create a contact in the CRM (Efficy Enterprise 12.1 in this case)
2. Add consent mappings in the CRM (example: "generic newsletters" subscription mapped to a topic in Apsis)
3. Trigger a full sync
4. Check the Apsis timeline for consent event entries

### Observed Behavior in Staging

After running a full sync, the following information appeared on the timeline:

- **Consent status**: FSC Enterprise 12.1 e-mail consent opt-in
- **Full sync source identifier**: which full sync batch it came from
- **Topic**: which topic the consent applies to

This validated that the consent event processing logic was functioning correctly in the staging environment.

### Real-time Webhook Issue

[Erik Andersson]: When testing real-time consent updates via webhooks (by modifying consent in the CRM system):

> It didn't come anything from the instance like at all. So something is weird there.

Despite modifying consent in the Efficy instance, no webhook events were received by Apsis One. Investigation showed:

- No log streams in the delta sync manager
- No entries in the webhook logs
- The CRM system appeared to not be sending webhooks at all

This indicates the issue is likely on the CRM side, not in the Apsis One code, since the full sync path worked correctly. Possible causes could include:
- CRM webhook configuration issue
- CRM system not configured to send real-time events
- Network/connectivity issue between CRM and Apsis webhook receiver

### Batch Processing in Efficy Enterprise

[Erik Andersson]: An important characteristic of Efficy Enterprise's behavior:

> FSA enterprise is batching things in the same way as we do. They don't send like here's 1 consent, 1 consent, 1 consent. They send like here are like 10 consents, 15 consents.

Both Efficy Enterprise and Apsis One batch consent events for efficiency, so testing should account for batch payloads rather than single event handling.

---

## Acceptance Testing Setup and Process

### Environment and Credentials

Beta environment testing requires:
- Access to a beta section/instance
- API keys for integrating with different Efficy Enterprise versions
- Separate test instances for Enterprise 12.0 and 12.1 (and generic connectors)

### Section Creation Timeout Caveat

[Tomasz Kowalski]: There is a known issue with section creation:

> We have great wait time out on 28 seconds and audience well will finish their job and the section is kind of accessibly created.

When creating a new section in beta, the UI may show a timeout error after 28 seconds, but the creation typically succeeds in the background. This is a false negative that can be confusing during testing setup.

### Test Case Execution Plan

For acceptance testing after deployment:

1. **Install Efficy Enterprise 12.1** on a test section in beta
2. **Run a full sync** and verify consent events appear on the timeline
3. **Check in a different section** with Enterprise 12.0 to ensure the legacy connector still works
4. **For real-time webhook verification**, modify consent in the CRM system and check that events arrive within 1-2 minutes (allowing for batch processing delays)

[Erik Andersson] notes that having dedicated QA resources would streamline this process:

> This was so much easier when we had the actual dedicated QA people that did all of this because they had their routine, they knew how how they tested it and they had like all of their sections pre pre set up.

---

## Deployment Status and Next Steps

### Deployment Completion

The AWS CodeBuild deployment succeeded and reached the beta environment. The CodeCommit/CodeBuild pipeline shows "succeeded" status, indicating all services are now deployed to beta.

### Outstanding Testing Tasks

[Erik Andersson] identifies the remaining acceptance test work:

1. **Someone** should install Efficy Enterprise 12.1 on a test section in beta, run a full sync, and verify consent events appear on the timeline
2. **Enterprise 12.0 verification** (legacy connector) must also be tested to ensure no regression
3. **Real-time sync testing** needs to be performed on at least one generic connector variant

[Erik Andersson] volunteers to perform the Enterprise 12.0 testing because:
- He has all necessary credentials ready
- He is familiar with the legacy environment
- He can verify both full sync and real-time webhook paths

### Timing Considerations

Due to Erik's computer battery status and personal commitments (social plans), the remaining acceptance testing may extend into the next day. The priority is ensuring all testing is complete before any production release.

---

## Key Technical Insights

### Generic Connector Value Proposition

The use of generic connectors dramatically reduces testing and debugging burden. Once a feature is validated to work with one generic connector variant (e.g., Tribe or E-deal), it is effectively validated for all variants because they share the same code path. This principle is foundational to the acceptance testing strategy.

### Legacy Connector Special Handling

Enterprise 12.0 is a legacy connector with custom functionality and historically sensitive customer base. It cannot be treated the same as generic connectors and must always be tested separately, despite increased effort, because:
- Different code path than generic connectors
- Major customers depend on it
- Escalation risk is high if issues occur

### Token Rotation Friction

The current deployment architecture creates friction during credential rotation because authentication is configured in multiple places (CodeBuild source provider AND Parameter Store). This increases the likelihood of partial updates and subtle deployment failures.

---

## Unresolved Questions and Action Items

### Open Issues

1. **Webhook delivery from Efficy Enterprise to Apsis One**: Why are real-time webhooks not being received during testing? Investigation needed on the CRM side.
2. **Computer performance**: Erik's system was struggling during the session (CPU/memory issues). Root cause not definitively identified.

### Action Items

- [ ] Erik Andersson to complete acceptance testing on Enterprise 12.0 connector (consent opt-in/opt-out on timeline)
- [ ] Someone else to verify Enterprise 12.1 generic connector consent events in beta
- [ ] Debug real-time webhook delivery issue with Efficy Enterprise (likely CRM-side configuration)
- [ ] Document the full sync batch format and behavior for consent events
- [ ] Consider migrating from CodeBuild to GitHub Actions to consolidate credential management to a single location
- [ ] Evaluate pre-staging test sections for acceptance testing to reduce setup friction