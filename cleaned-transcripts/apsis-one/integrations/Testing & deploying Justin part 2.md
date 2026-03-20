---
source_file: Testing & deploying Justin part 2.txt
domain: Apsis One - Integrations
topics: [Token Management and Credential Rotation, CodeBuild Deployment Pipeline, GitHub Authentication, Consent Event Testing, CRM Integration Testing, FSC Enterprise Connector Versions, Full Sync vs Real-time Sync, Acceptance Testing Procedures]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [CodeBuild, CloudFormation, AWS Parameter Store, GitHub Token, CRM System, FSC Enterprise 12.0, FSC Enterprise 12.1, Generic Connectors, Consent Events, Delta Sync Manager, Webhook Integration]
session_type: knowledge-transfer
---

## Session Overview

This session covers the deployment and testing of an Apsis One integration feature, specifically focusing on consent event handling across CRM connectors. Erik Andersson walks through a critical issue discovered during deployment related to token credential management in CodeBuild, then guides the team through acceptance testing procedures for verifying that consent events are properly captured and displayed on customer timelines. The discussion emphasizes the testing strategy for both legacy (FSC Enterprise 12.0) and modern (FSC Enterprise 12.1 and generic) connectors, and highlights important caveats about webhook vs. full sync behavior.

---

## Token Management and Credential Rotation in CodeBuild

### The Dual-Token Problem

**[Erik Andersson]**: The deployment pipeline was failing due to GitHub token authentication happening in two different places within the CodeBuild configuration:

1. **Source stage credential** — When CodeBuild initially downloads the source code from GitHub (happens at the web trigger stage)
2. **Buildspec file reference** — Later in the deployment steps, when the build process explicitly references the GitHub token to download additional repositories

When the GitHub token was rotated, only the second location was updated. The token in the CodeBuild project's source provider credential was not refreshed, causing the initial source code download to fail before the build process even reached the steps that use the token from the Parameter Store.

### The Solution and Why It Matters

**[Erik Andersson]**: To fix this issue, I had to manually reconnect the CodeBuild project:
- Navigate to **Manage Account Credentials** in the CodeBuild project settings
- Disconnect the old GitHub token
- Re-add the new token

This is a critical gotcha because the failure manifests as the build never even starting properly, so developers may not immediately realize the problem is with initial credential authentication rather than something within the build process itself.

### Why GitHub Actions Is Better

**[Erik Andersson]**: The advantage of using GitHub Actions for deployment is that the token is configured in only one place (repository secrets or environment variables), eliminating this dual-configuration problem entirely. This is one of several reasons the team is considering moving away from the current CodeBuild approach.

---

## Deployment Pipeline and Service Parallelization

### Current Architecture

The deploy project is structured such that all deployment steps use CloudFormation to deploy services. Multiple services (three or four) are deployed in parallel, all triggered through the CodeBuild pipeline. Once the deployment completes (approximately 10 minutes), all services in the beta environment are ready for acceptance testing.

---

## Consent Event Testing Strategy

### Understanding Consent Events

Consent events need to be captured and displayed on customer timelines in two scenarios:

1. **Full sync** — When a customer's full data is synchronized from the CRM system
2. **Real-time sync** — When consent changes are pushed to Apsis via webhooks

The consent event display format should follow this pattern:
```
[Connector Name] Consent [Action] [Channel]
Examples:
- FSC Enterprise 12.1 Consent Opt In (Email)
- FSC Enterprise 12.1 Consent Opt Out (SMS)
```

### CRM Profile Mapping Requirements

**[Erik Andersson]**: For testing consent events, you cannot simply add consent through the Apsis form. The consent must originate from the CRM system and either:
- Be pulled during a full sync from the CRM, OR
- Be pushed via webhook when consent is modified in the CRM

The team needs to verify that the mapped consent fields from the CRM appear correctly on the timeline with proper labels and channel information.

### Testing Against Staging Environment

The initial verification was performed in staging using a profile with CRM ID 253. The consent that appears must match a mapping that exists in the configuration — for example, "generic newsletters" as the topic. The test confirmed that when a full sync is executed, the consent events are imported and displayed with:
- The consent action (opt in/opt out)
- The channel information (email or SMS)
- The source sync information (which full sync)
- The mapped topic name

---

## FSC Enterprise Connector Versions and Testing Requirements

### Legacy vs. Modern Connectors

**FSC Enterprise 12.0** is the legacy version with custom functionality specific to enterprise customers. This is a non-generic connector with its own implementation.

**FSC Enterprise 12.1** and other modern connectors (Tribe, Microsoft Dynamics, SAP, E.Deal) use the **generic connector framework**, meaning they share the same underlying code and logic in Apsis.

### The Generic Connector Testing Principle

**[Erik Andersson]**: This is critical for acceptance testing:

> "If you have tried it for one integration, you have essentially tried it for everything that uses it... If it works for one of those, then it is working as it should on Apsis side, because the logic for us is the same regardless of which of those integration it is. If nothing is not working, then that is because the CRM is not doing what they should or they don't support the feature."

This means:
- If functionality works for FSC Enterprise 12.1, it works for all generic connectors
- If it fails for a generic connector, the problem is almost certainly in the CRM, not in Apsis
- The generic connectors all execute the same code path

### Mandatory Testing Checklist

**[Erik Andersson]**: Every acceptance test for integrations must include:

1. **FSC Enterprise 12.0** (mandatory)
   - Reason: Large installed base of legacy customers who are highly vocal if issues arise
   - These customers escalate directly when problems occur
   - Has custom functionality that differs from generic connectors
   
2. **At least one generic connector** (mandatory)
   - Choose from: Tribe, E.Deal, SAP Site Shop, or FSC Enterprise 12.1
   - If it works for one, it works for all (same code path)

This testing strategy ensures coverage of both the legacy customer base and the modern connector ecosystem without redundantly testing every generic connector variant.

---

## Full Sync vs. Real-Time Sync and Webhook Behavior

### Expected Behavior

Consent can be received from the CRM system in two ways:

1. **Real-time syncs** — Triggered by webhooks when consent changes in the CRM
2. **Full syncs** — Periodic batch synchronization of all data

Both should result in consent events appearing on the timeline.

### Batching Behavior with FSC Enterprise

**[Erik Andersson]**: FSC Enterprise does not send individual consent events one-by-one. Instead, like Apsis's own batching approach, it sends multiple events in a single batch:
```
Instead of: Consent, Consent, Consent
It sends: [10 consents], [15 consents]
```

This is important for testing because the events will arrive in batches and should be processed as a batch by the import logic.

### Troubleshooting Real-Time Sync Issues

During testing, no webhook events were received from the CRM instance even though full sync worked correctly. **[Erik Andersson]**: This indicates the problem is not in Apsis's receipt and processing logic (since full sync worked), but rather in the CRM system's webhook configuration or delivery mechanism.

Evidence that this is a CRM-side issue:
- No log entries appeared in the Delta Sync Manager at all
- Full sync successfully imported the consent events
- The feature implementation itself is working correctly

If you need to test full sync behavior in isolation without interference from webhooks, you may need to manually deploy configurations to reject real-time syncs, though this is complex.

---

## Acceptance Testing Workflow

### Creating Test Sections

**[Erik Andersson]**: The acceptance testing process involves:

1. Create a section (test environment instance) in beta with a descriptive name, e.g., "Acceptance Testing 12.0"
2. Install the specific connector version on that section (12.0, 12.1, or generic variant)
3. Obtain the API key from the installed connector
4. Create a mapping (subscription mapping) between CRM fields and Apsis consent topics
5. Execute a full sync
6. Verify that consent events appear correctly on the profile timeline

### Section Creation Timeout Behavior (Important Gotcha)

**[Tomasz Kowalski]**: The section creation endpoint has a **28-second timeout**, but this does not always indicate failure:

> "We have a grace wait time out on 28 seconds and the backend will finish their job and the section is kind of accessibly created."

This is a **false negative** — the API call times out but the section is actually created successfully in the background. Always check whether the section was created before retrying.

### Pre-Deployment Advantage

**[Erik Andersson]**: When QA was a dedicated function, this acceptance testing was much smoother because QA had:
- Established test routines and checklists
- Pre-configured test sections already set up
- Institutional knowledge of edge cases

Without dedicated QA, developers must now perform these manual steps, which is more time-consuming and error-prone.

---

## Testing Procedures for Consent Events

### Verification Steps in Beta

Once the deployment to beta is complete, the acceptance testing includes:

1. **FSC Enterprise 12.1 Testing** (someone)
   - Install FSC Enterprise 12.1 on a beta section
   - Run a full sync
   - Verify that consent events appear on the contact's timeline

2. **FSC Enterprise 12.0 Testing** (Erik planned to do this)
   - Install FSC Enterprise 12.0 (legacy version) on a beta section
   - Run a full sync
   - Verify consent events appear
   - ALSO verify real-time sync behavior by modifying consent in the CRM and checking for immediate updates in Apsis

3. **Webhook Verification**
   - Real-time syncs are expected to work automatically when consent changes in the CRM
   - Events should appear within 1-2 minutes

### Why Both Sync Types Matter

**[Erik Andersson]**: When doing acceptance testing, you should verify both:
- Full sync can deliver consent events
- Real-time sync can deliver consent events (if webhooks are working in the CRM)

However, if real-time syncs are not appearing despite full sync working, the issue is almost certainly in the CRM's webhook configuration, not in Apsis's implementation.

---

## Key Takeaways

1. **Token credential rotation requires updates in multiple places** — When rotating GitHub tokens for CodeBuild deployments, update both the source provider credentials AND any references in the buildspec/deployment files. Consider migrating to GitHub Actions to consolidate this into one location.

2. **Generic connectors share code** — If a feature works for one generic connector (FSC Enterprise 12.1, Tribe, E.Deal, SAP Site Shop), it works for all of them in Apsis. Testing one proves the implementation; if it fails, the CRM is likely the culprit.

3. **FSC Enterprise 12.0 is mandatory for acceptance testing** — Because of a large installed customer base with high escalation tendencies, always explicitly test with the legacy 12.0 connector in addition to modern connectors.

4. **Consent events require CRM-originated data** — Consent cannot be added through Apsis forms for testing purposes; it must come from the CRM system via full sync or webhook.

5. **Section creation timeouts are false negatives** — If section creation hits a 28-second timeout, check the backend to verify the section was actually created before retrying.

6. **Webhook failures often indicate CRM-side issues** — If full sync works but real-time sync (webhooks) doesn't, check the CRM system's webhook delivery configuration; Apsis's receipt and processing logic is working.

7. **Batching behavior matters** — FSC Enterprise sends multiple consent events in batches rather than individually, which is expected and correct behavior.

---

## Unresolved Questions and Action Items

- **Pending**: Verify that FSC Enterprise 12.0 consent events appear correctly on timelines in beta (Erik was about to test this before leaving the session)
- **Pending**: Confirm real-time webhook delivery for FSC Enterprise 12.0 consent changes (Erik will check if needed)
- **Pending**: Investigate why no webhooks were received from the CRM instance despite successful full sync (likely a CRM-side configuration issue, not Apsis)
- **Pending**: Complete acceptance testing for at least one generic connector variant in beta to confirm generic connector code path is working