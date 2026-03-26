---
source_file: Testing & deploying Justin part 2.txt
domain: Apsis One Integrations
topics: [GitHub token authentication in CodeBuild, Deployment pipeline troubleshooting, Consent events and timeline functionality, Integration testing strategy, Full sync vs real-time sync verification, Generic connector testing methodology]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [AWS CodeBuild, GitHub authentication, Parameter Store, FSC Enterprise 12.0, FSC Enterprise 12.1, Generic connectors (Tribe, Microsoft Dynamics, E-Deal), Consent events, Delta sync manager, Web hooks, Full sync]
session_type: debugging-session
subdomains: ["Lead creation", "Different Types of Connectors"]
---

## Session Overview

This KT session documents a deployment and testing workflow for Apsis One integrations, specifically focusing on resolving GitHub token authentication issues in the CodeBuild pipeline and establishing a comprehensive acceptance testing strategy. The team deployed consent event functionality to the beta environment and validated the feature across both legacy (FSC Enterprise 12.0) and generic connector types (FSC Enterprise 12.1, Tribe, E-Deal). A critical lesson emerged about the difference between legacy connectors (custom, version-specific) and generic connectors (standardized interface, functionally equivalent across implementations).

---

## GitHub Token Authentication Issue in CodeBuild Deployment

### Root Cause: Multiple Authentication Points

[Erik Andersson] identified that the deployment pipeline references the GitHub token in two distinct locations, creating a synchronization problem:

1. **Source Code Download Phase**: When CodeBuild initially downloads the source repository from Git after receiving a webhook trigger, it uses credentials configured in the CodeBuild project itself
2. **Deployment Spec Phase**: Later in the deployment steps, the code references the token again within the deploy specification file, loading it from Parameter Store as an environment variable

### The Problem

When Erik rotated the GitHub token, he updated it in Parameter Store (which feeds into the deploy spec file) but failed to reconnect the CodeBuild project with the new credentials. This meant:

> The deployment never reached the point where it tried to utilize the GitHub token value from Parameter Store, because it failed earlier during the initial source code download phase.

### Resolution

[Erik Andersson] had to manually reconnect CodeBuild to GitHub by going to "manage account credential," disconnecting the old token, and re-adding the new one.

### Why GitHub Actions Is Better

[Erik Andersson] noted that using GitHub Actions instead of CodeBuild would eliminate this problem:

> You don't need to do this if you are using GitHub action because then we have this in one place instead.

This is a key architectural advantage: GitHub Actions centralizes token management in a single location, whereas the current CodeBuild setup distributes authentication across the source provider configuration and the runtime environment variables.

---

## Deployment Pipeline and Service Parallelization

The deployment process builds and deploys three to four services in parallel through CloudFormation. After the token issue was resolved, the full deployment ran to completion, releasing all beta services. The team uses AWS CodeCommit/CodeBuild for orchestration and monitors deployment status through the AWS console (Deployments tab in CodePipeline).

---

## Consent Events Feature: Testing Strategy and Validation

### What Consent Events Are

Consent events track changes to email and SMS subscription preferences. When a profile is updated in Apsis One (either via full sync from the CRM or via real-time webhook), the system records consent state changes with the following information:

- **Event format**: `[Connector Name] consent [opt in/opt out] [channel: email or SMS]`
- **Example**: `FSC Enterprise 12.1 consent opt in email`
- **Metadata**: The system records which full sync the event came from and which topic/list the consent change applies to

### Testing Across Two Sync Pathways

The feature must work in both scenarios:

1. **Full Sync Path**: CRM sends a batch of consent changes during a scheduled full sync
2. **Real-Time Path**: CRM sends individual consent changes via webhook in near-real-time

[Erik Andersson] explained that consent can arrive through both channels and the system must handle both correctly. During testing, the team observed that FSC Enterprise was not sending real-time webhooks (no log entries in the delta sync manager), but the full sync path worked correctly.

### Why FSC Enterprise Batches Differently

[Erik Andersson] noted that FSC Enterprise batches consent changes the same way Apsis does:

> They don't send like here's 1 consent, 1 consent, 1 consent. They send like here are 10 consents, 15 consents.

This is important for understanding throughput and debugging logs—consent events should appear in batches, not individually.

---

## Acceptance Testing Methodology: Legacy vs. Generic Connectors

### The Two Types of Connectors and Testing Implications

**Legacy Connector (FSC Enterprise 12.0)**:
- Built in-house by Apsis with custom functionality
- Version-specific behavior
- **Must be tested** because many major customers still use 12.0
- Historically, these customers are quick to escalate issues to executives when something breaks

[Erik Andersson] explained the business risk:

> We have a lot of big customers on FSC Enterprise 12.0 and those customers historically as soon as something is broken, they get very irritated and send emails straight to the CEO. Then you need to deal with overreactions for days after.

**Generic Connector (FSC Enterprise 12.1, Tribe, Microsoft Dynamics, E-Deal)**:
- Standardized interface implemented by CRM vendors
- Functionally equivalent across all implementations in Apsis code
- **Only one needs to be tested** to validate the entire category

### Testing Strategy: The "One Test Proves All" Principle

[Erik Andersson] articulated the key insight:

> If it works for one generic connector in Apsis, then it works for all of them in Apsis. If it is not working, then it is a 99.99% probability that the CRM is doing something weird, because they run on the same code and the same flow.

This means acceptance testing requires:

1. **Always test FSC Enterprise 12.0** (legacy) explicitly—no shortcuts
2. **Test at least one generic connector** (e.g., Tribe, E-Deal, or FSC Enterprise 12.1)—if it passes, all generic connectors pass
3. **Skip testing the other generic connectors**—they use identical code paths

### Why This Matters for Debugging

When a feature works in the generic connector but fails in the legacy connector (or vice versa), the root cause is immediately clear:

- **Generic works, Legacy fails**: The custom logic in the legacy connector is the issue (CRM-side problem or custom Apsis code)
- **Generic fails, Legacy works**: The shared code path has a regression (Apsis-side problem in core logic)

---

## Real-Time Sync vs. Full Sync: Debugging Observation

During testing, [Erik Andersson] observed no log entries in the delta sync manager when expecting real-time webhook consent events, despite the full sync working correctly. This indicates:

> When nothing is coming through real-time syncs but full sync works, something is up in the CRM system sending the webhooks. It's not an Apsis problem.

The team uses log visibility in the delta sync manager as a diagnostic tool: if there are no logs at all, it means the CRM system is not sending webhooks to Apsis.

---

## Environment Setup for Acceptance Testing

### Section/Tenant Creation

Teams create isolated test environments by setting up "sections" (Apsis terminology for test instances within beta). There is a known timeout behavior:

> Creating a section tends to time out with a 28-second wait timeout, but the request completes in the background and the section is actually created successfully.

[Tomasz Kowalski] confirmed this is expected behavior—the UI shows a timeout error (false negative) while the backend successfully provisions the section.

### Installation Steps

1. Create a section with a descriptive name (e.g., "acceptance_testing_12.0")
2. Install the desired connector version (12.0 for legacy, 12.1 for generic)
3. Retrieve the API key for the installed connector
4. Configure subscription/consent mapping on the test section
5. Run either a full sync or wait for real-time webhook events
6. Verify the consent events appear on the profile timeline

---

## Operational Considerations

### Pre-Deployment Setup vs. Current State

[Erik Andersson] noted a significant operational shift:

> This was so much easier when we had actual dedicated QA people who did all of this. They had their routine, they knew how they tested, and they had all their sections pre-set up.

Without dedicated QA, developers are now responsible for setting up test environments, which adds friction to the acceptance testing process.

### Testing Environment State

Consent events were successfully verified in the staging environment for FSC Enterprise 12.1, showing the feature is working in Apsis code. The next step is to replicate the test in the beta environment for both 12.0 and 12.1 after the deployment completes.

---

## Key Takeaways

1. **Token Authentication Risk**: CodeBuild's dual token references (source provider + runtime environment) create synchronization problems. Migration to GitHub Actions would consolidate authentication to a single point.

2. **Generic Connector Equivalence**: Testing a feature on one generic connector (Tribe, E-Deal, FSC 12.1) is equivalent to testing it on all generic connectors. This significantly reduces acceptance testing scope while maintaining confidence.

3. **Legacy Connector Criticality**: FSC Enterprise 12.0 must always be tested separately because it uses custom code paths and serves large, sensitive customers who escalate issues aggressively.

4. **Real-Time vs. Full Sync Debugging**: Absence of delta sync manager logs indicates the CRM is not sending webhooks, not an Apsis code problem. Always verify both sync paths work when testing new features.

5. **Timeout Behavior Is Expected**: Section creation timeouts are false negatives—the backend completes the operation while the UI reports failure. Verify creation succeeded in the background.

6. **Consent Events Format**: Consent changes are recorded with connector name, opt-in/out state, channel (email/SMS), topic, and sync source. This metadata is critical for debugging user-facing consent issues.

---

## Unresolved Questions & Action Items

- **Erik's Test**: Verify consent event functionality works on FSC Enterprise 12.0 (legacy connector) in beta environment via full sync
- **Real-Time Webhook Issue**: Investigate why FSC Enterprise is not sending real-time consent webhooks to Apsis (likely a CRM-side configuration issue, but needs confirmation)
- **Architecture Discussion**: Evaluate migration path from CodeBuild to GitHub Actions for simplified token management
- **QA Process**: Consider re-establishing dedicated QA resources or formalizing the developer-led acceptance testing workflow to reduce friction