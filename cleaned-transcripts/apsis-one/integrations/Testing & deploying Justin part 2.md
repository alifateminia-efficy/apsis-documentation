---
source_file: Testing & deploying Justin part 2.txt
domain: Apsis One Integrations
topics: [Deployment process and CI/CD pipeline, GitHub token authentication and rotation, Consent event handling and timeline tracking, Full sync vs real-time sync verification, Acceptance testing procedures, Efficy Enterprise integration versions]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [AWS CodeBuild, AWS CloudFormation, GitHub token management, Parameter Store, AWS Lambda (implied in streaming/log context), Efficy Enterprise 12.0, Efficy Enterprise 12.1, Generic connectors, Consent management system, Delta sync manager, Web hooks]
session_type: debugging-session
subdomains: [Architecture, Efficy Enterprise 12.0 Integration, Efficy Enterprise 12.1 Integration, Tribe Integration, E-deal Integration]
---

## Session Overview

This knowledge transfer session covers the debugging and deployment of a consent events feature for Apsis One integrations. The primary issue discussed was a GitHub token authentication failure during deployment caused by token rotation not being applied consistently across multiple authentication points in the CI/CD pipeline. The session also covers the acceptance testing strategy for verifying consent event functionality across different Efficy Enterprise versions and generic connectors, with emphasis on the importance of testing both real-time syncs and full syncs.

---

## Deployment Infrastructure and Token Authentication Issues

### The Core Problem: Dual Authentication Points

**Erik Andersson** identified a critical architectural issue in the deployment process: GitHub tokens are authenticated in two separate locations, creating a maintenance burden and a source of deployment failures.

> The issue is that you have two places where you authenticate with the token. In one place, you reference the token inside the deploy spec file and use the GitHub token environment variable by loading this value from the Parameter Store. But there is one step even before this where the token is also utilised—when CodeBuild downloads the whole source code from Git, the credential is configured inside the build as well.

### The Failure Scenario

When **Erik** rotated the GitHub token, he updated it in the deployment file's Parameter Store reference, but the token used by CodeBuild for initial source code checkout was not updated. This created a situation where:

1. CodeBuild attempted to fetch the repository source code with an expired/rotated token
2. The operation failed before reaching the deployment steps that had the updated token
3. The build never progressed to the later steps where the newly-rotated token was configured

**Erik** explained the fix: "I had to reload—or rather I had to reconnect the CodeBuild again after I had rotated the key. I went here into the manage account credential, disconnected, and then re-added the new token."

### AWS Service Configuration Details

- **Service**: AWS CodeBuild
- **Configuration location**: "Manage account credential" section in the build project settings
- **Token source**: Parameter Store (for deployment spec)
- **Credential scope**: Must be updated in both Parameter Store AND CodeBuild's built-in credential management

### Why GitHub Actions Would Be Better

**Erik** noted: "You don't need to do this if you are using GitHub Action because then we have this in one place instead."

This is identified as a technical debt item—consolidating to a single token management location would eliminate this class of deployment failures.

---

## Deployment Pipeline Architecture

### Parallel Service Deployment

The deployment process builds and deploys **three or four services in parallel** using CloudFormation. According to **Erik**: "Everything we do in the deployment steps is like we are deploying the CloudFormation here now. So now it is going through every the building and deploying of every service that we have because we're doing these three or four services in parallel."

### Expected Deployment Duration

After token issues are resolved, the full deployment to beta was expected to take approximately **10 minutes** to complete all services.

---

## Consent Events Feature: Architecture and Testing Strategy

### What Consent Events Are

Consent events appear on user timeline entries in Apsis and represent consent status changes (opt-in or opt-out) for email and SMS channels. When a profile is updated in a full sync or when consent is sent via webhook from the CRM, the system records:

- **Event type**: `FSC Enterprise 12.1 consent opt in` or `FSC Enterprise 12.1 consent opt out`
- **Channel**: Email or SMS
- **Topic**: The specific subscription/preference category
- **Source**: Whether it came from a full sync or real-time sync
- **Full sync indicator**: Which full sync batch this came from

### Data Flow Architecture

Consent data flows through the system via two mechanisms:

1. **Full Sync**: Batch export of all profiles with current consent status
2. **Real-time Sync (Web hooks)**: Delta updates from the CRM instance

**Erik** explained the batching behavior: "FSC enterprise is batching things in the same way as we do. They don't send like 'here's 1 consent, 1 consent, 1 consent.' They send like 'here are 10 consents, 15 consents.'"

### Consent Configuration Mapping

For consent events to appear on timelines, consent fields must be:

1. **Mapped in the integration**: Generic connectors use standardized mappings (e.g., "generic newsletters")
2. **Created as a section**: Sections are configured with specific consent topic mappings
3. **Tracked in Apsis**: The system records which full sync created the entry and which CRM topic it came from

**Erik** demonstrated this with a specific example: Checking a test profile named "Tomax test profile" with ID 253, where the consent mapping for "generic newsletters" was verified.

---

## Acceptance Testing Procedures and Best Practices

### Testing Matrix: Version and Connector Coverage

**Erik** established the acceptance testing standard:

> Anytime you do acceptance testing for an integration, you always need to verify the functionality for FSC Enterprise 12.0 because the majority of customers are still on 12.0. It's a legacy connector with custom functionality. Those customers are very quick to complain when things are not working, and they will escalate directly to the CEO, causing overreactions for days after.

The full acceptance testing matrix requires:

1. **FSC Enterprise 12.0** (mandatory): Legacy connector with large customer base
   - Verify full sync consent events
   - Verify real-time sync consent events (if supported)

2. **One generic connector** (mandatory—any one of):
   - FSC Enterprise 12.1
   - Tribe
   - Microsoft Dynamics
   - E-deal

**Rationale**: Because generic connectors share the same codebase and logic in Apsis, if consent events work for one generic connector, they work for all of them. If something fails, it's a 99.99% probability the CRM system is at fault, not the Apsis integration.

### Why Full Sync Testing Matters

When real-time sync webhooks don't appear to be working (as happened during testing), full sync becomes the verification method. **Erik** noted the challenge:

> You could potentially need to deploy something manually to reject all real-time syncs if you really want to test full sync in isolation, because consent can come from both real-time syncs and full syncs, and it should be visible in both cases.

### Testing Environment Setup

For acceptance testing, **Erik** demonstrated the setup process:

1. Create a new section in the beta environment (e.g., "acceptance_testing_12.0")
2. Select the specific integration version to install
3. Retrieve the API key for the integration
4. Configure the section with the appropriate consent mappings
5. Install the integration
6. Perform a full sync to test consent event creation

**Known issue**: Section creation can time out with a **28-second timeout**, but the background job completes successfully. This is a false negative—the section is actually created and accessible despite the timeout error.

### Debugging Consent Event Failures

When consent events don't appear as expected:

1. **Check delta sync manager logs**: Should show entries for webhook processing
2. **Check CloudWatch logs**: Should show consent events being imported
3. **If logs are empty**: The CRM system likely didn't send the webhook or the data didn't come through
4. **If events appear in full sync but not real-time**: Webhooks from the CRM instance may not be configured or enabled

**Erik's debugging approach**: "If nothing is not working, then that is because the CRM is not doing what they should or they don't support the feature."

---

## Key Differences Between Integration Versions

### FSC Enterprise 12.0 (Legacy)

- Non-generic connector with **custom functionality**
- Used by the majority of Apsis customers
- Requires explicit testing because customer issues escalate rapidly
- Has dedicated test instances maintained by the team

### FSC Enterprise 12.1

- Generic connector implementation
- Newer version of the same base system
- Can be used as a proxy for all generic connectors in testing

### Generic Connectors (Tribe, Microsoft Dynamics, E-deal)

- Share identical codebase in Apsis
- If a feature works for one, it works for all
- Reduces testing burden significantly
- If issues occur, they're typically CRM-side rather than Apsis-side

---

## Operational Context and Technical Debt

### QA Infrastructure Evolution

**Erik** reflected on process degradation: "This was so much easier when we had actual dedicated QA people that did all of this because they had their routine, they knew how they tested it, and they had like all of their sections pre-set up."

This indicates the team now handles acceptance testing without dedicated QA resources, making standardized procedures even more critical.

### Batching Behavior Across Systems

Both Apsis and Efficy Enterprise batch consent updates rather than sending individual records, which affects real-time testing and debugging. Developers need to account for this when verifying webhook payloads.

---

## Key Takeaways

1. **GitHub token authentication requires dual updates**: Token rotation must be applied in both Parameter Store AND CodeBuild's credential management to prevent deployment failures.

2. **Acceptance testing requires version matrix coverage**: Always test FSC Enterprise 12.0 (for customer safety) plus one generic connector (to verify generic connector logic). This covers all customers and code paths.

3. **Full sync and real-time sync must both be verified**: Consent events should appear via both mechanisms. If real-time syncs aren't arriving, it's typically a CRM webhook configuration issue, not an Apsis issue.

4. **Generic connectors reduce testing burden**: Testing one generic connector effectively tests all generic connectors, enabling efficient regression testing.

5. **Section creation timeouts are false negatives**: A 28-second timeout during section creation doesn't indicate failure; the background job completes successfully and the section is accessible.

6. **Customer escalation patterns dictate testing priority**: FSC Enterprise 12.0 is critical to test because large customers on this version escalate issues rapidly and directly to leadership.

7. **Consent mapping is prerequisite for timeline events**: Consent events only appear on timelines if the consent topics are mapped in the integration configuration.

---

## Unresolved Questions and Action Items

### Pending Verification (as of session end)

- **FSC Enterprise 12.0 real-time sync verification**: Erik was setting up a full sync on the 12.0 test section to verify consent events appear, and planned to check real-time sync behavior after charging his computer.

- **Web hook delivery from FSC Enterprise 12.1**: During testing, no real-time consent webhooks were received despite configuration. The cause needs investigation—likely CRM-side configuration issue, but requires verification in staging/beta environments.

- **Generic connector testing in beta**: After deployment completion, someone needs to verify consent functionality for at least one generic connector (Tribe, E-deal, or FSC Enterprise 12.1) in the beta environment.

### Deployment Status at Session End

- Code build deployment to beta environment was in progress (expected to complete in ~10 minutes)
- Integration installations and full sync testing were staged but not yet executed due to Erik's need to leave the session

---

## Technical Details for Future Reference

### AWS CodeBuild Configuration Path
- Credential management: "Manage account credential" → Disconnect old token → Re-add new token

### Parameter Store Usage
- GitHub token stored in Parameter Store
- Referenced in deploy spec file for download operations

### Timeline Event Format
- Format: `[CRM System] [CRM Version] [consent type] [channel]`
- Example: `FSC Enterprise 12.1 consent opt in email`
- Includes metadata about full sync source and consent topic