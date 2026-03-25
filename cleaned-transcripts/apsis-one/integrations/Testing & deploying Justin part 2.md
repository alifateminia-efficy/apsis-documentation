---
source_file: Testing & deploying Justin part 2.txt
domain: Apsis One Integrations
topics: [Deployment Pipeline Issues, GitHub Token Management, Consent Events Testing, Generic Connectors vs Legacy Connectors, Acceptance Testing Methodology, Full Sync vs Real-time Sync]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [AWS CodeBuild, AWS CloudFormation, GitHub Token Authentication, Parameter Store, Consent Timeline Events, FSC Enterprise 12.0, FSC Enterprise 12.1, Generic Connectors, Delta Sync Manager, Full Sync, Real-time Webhooks]
session_type: debugging-session
subdomains: [Architecture, Outbound Flow, Efficy Enterprise 12.0, Efficy Enterprise 12.1, Tribe, E-deal (Efficy Corporate), Duplicate profiles in Apsis]
---

## Session Overview

This session focused on debugging and resolving a production deployment failure in the Apsis One Integrations platform, specifically related to GitHub token authentication in the AWS CodeBuild deployment pipeline. After diagnosing and fixing the token issue, the team proceeded to acceptance testing for a new consent events feature that displays consent opt-in/opt-out information on customer timelines. The session covered testing methodology across both legacy (FSC Enterprise 12.0) and generic connectors (FSC Enterprise 12.1, Tribe, E-deal), emphasizing the importance of validating against the legacy connector due to the large customer base still using it.

---

## Deployment Pipeline & GitHub Token Authentication Issue

### Root Cause of Deployment Failure

[Erik Andersson]: The deployment was failing because GitHub token authentication was configured in two separate places within the deployment pipeline, and both needed to be updated when the token was rotated.

The first authentication point occurs when **AWS CodeBuild downloads the source code from GitHub** at the start of the pipeline. This credential is stored within the CodeBuild project configuration itself and must be manually updated in the **account credentials management interface**.

The second authentication point occurs later during the deployment steps in the buildspec file, where the GitHub token is referenced as an environment variable:

```
GitHub token loaded from Parameter Store → used during repository downloads in deployment steps
```

[Erik Andersson]: 
> I updated the token for the step inside the deployment file. However, I had not updated the token where we download the source code after the web trigger in the first place, so it never even got here where it tried to utilize the GitHub token value.

### The Fix

When rotating the GitHub token, both locations must be updated:

1. **In CodeBuild Project Configuration**: Navigate to "Manage account credentials" → Disconnect the old GitHub connection → Re-add with the new token
2. **In the buildspec/deployment file**: Update the environment variable that references the token in Parameter Store

[Erik Andersson]: The reason this is problematic is that you have to remember to update credentials in two places. If you are using GitHub Actions instead, this would be managed in one place, which is why moving away from this dual-authentication approach would be beneficial.

### Parallel Service Deployment

The deployment process builds and deploys three to four services in parallel using AWS CloudFormation. This parallel approach speeds up the overall deployment time but makes the multi-location token issue more critical since all services will fail simultaneously if either authentication point fails.

---

## Consent Events Testing & Feature Validation

### Feature Overview

The newly deployed feature adds **consent event tracking to customer timelines**. When a customer's consent preferences are updated (either through CRM profile modifications or explicit consent changes), the system captures and displays:

- Consent type: `FSC Enterprise 12.1 consent opt in` / `FSC Enterprise 12.1 consent opt out`
- Channel: Email or SMS
- Topic: The specific consent topic/category
- Source: Whether the update came from a full sync or webhook

### Test Case Setup

[Erik Andersson]: The goal is to verify that when a profile receives a consent update in the CRM, it appears on the Apsis timeline with the correct event type and metadata. The test involved:

1. Creating a test profile in the CRM (using CRM ID 253 for example)
2. Adding a consent mapping that Apsis has configured (e.g., "generic newsletters")
3. Triggering either a full sync or real-time webhook to populate the consent event
4. Verifying the event appears on the timeline in Apsis with correct labeling

### Full Sync vs Real-time Sync Testing

[Erik Andersson]: Consent updates can arrive via two mechanisms:

- **Full sync**: Periodic batch sync of all customer data from the CRM. This is reliable and deterministic for testing.
- **Real-time webhooks**: Instant notifications when consent is modified. These are faster but depend on proper webhook configuration in the CRM.

During testing, real-time webhooks did not arrive from the CRM instance. This does not indicate a problem with the Apsis side; rather, it suggests something in the CRM system is not sending webhooks correctly. The feature was validated as working correctly when full sync successfully populated the consent events.

[Erik Andersson]: 
> Because FSC Enterprise is batching things in the same way as we do. They don't send like here's 1 consent, 1 consent, 1 consent. They send like here are like 10 consents, 15 consents.

This batching behavior is normal and expected for full syncs.

### Logging & Troubleshooting

When real-time webhooks were not appearing, the team checked:

- **CloudWatch log streams**: No new log entries appeared, indicating webhooks were not reaching Apsis at all
- **Delta Sync Manager logs**: No entries for the consent changes, further confirming the CRM was not sending webhooks

[Erik Andersson]: 
> That would either mean that the webhooks are not working from FSC Enterprise or something really weird is going on.

This type of diagnostic approach is valuable: if logs show no activity, the problem is upstream (in the CRM), not in Apsis's webhook handling.

---

## Acceptance Testing Methodology & Connector Coverage

### Critical Testing Rule: Always Test FSC Enterprise 12.0

[Erik Andersson]: When performing acceptance testing for any new feature, you **must** include validation against **FSC Enterprise 12.0**, even though it is a legacy connector. This is mandatory because:

1. The majority of Apsis customers are still running FSC Enterprise 12.0
2. Large enterprise customers are particularly sensitive to regressions
3. When issues arise with these major customers, they escalate rapidly and directly to executive leadership

[Erik Andersson]: 
> We have a lot of big customers here and 12.0 is a legacy connector, so it has like custom functionality and those customers historically as soon as something is up, they get very irritated and send emails straight to the CEO and then you need to deal with over reactions for days after.

The platform maintains dedicated test instances for FSC Enterprise 12.0 to support this requirement.

### Generic Connector Testing: Test One, Validate All

[Erik Andersson]: For features implemented using **generic connectors**, you only need to validate the functionality once across one connector type. The generic connector implementations for **FSC Enterprise 12.1, Tribe, Microsoft Dynamics, and E-deal (Efficy Corporate)** all share the same underlying Apsis-side logic and code paths.

[Erik Andersson]: 
> If it works for one of those, then it is working as it should on Apsis side, because the logic for us is the same regardless of which of those integrations it is. If it is not working, then that is because the CRM is not doing what they should or they don't support the feature.

This design principle means that if consent events work correctly for FSC Enterprise 12.1, you have mathematically validated that they work for Tribe, E-deal, and Microsoft Dynamics as well. If they fail for one generic connector, the root cause is either:
- A bug in the CRM system (not in Apsis)
- Missing feature support in that specific CRM product

### Required Acceptance Test Checklist

A complete acceptance test must include:

1. **FSC Enterprise 12.0** (legacy, non-generic): Verify full sync works correctly
2. **At least one generic connector** (recommend FSC Enterprise 12.1, Tribe, E-deal, or Microsoft Dynamics): Verify both full sync and real-time sync

If both pass, the feature is production-ready.

---

## Setting Up Test Environments

### Creating Test Sections

[Erik Andersson]: Test sections are created in the Apsis beta environment to validate features before promoting to production. The process involves:

1. Navigate to the appropriate CRM instance (e.g., FSC Enterprise 12.0)
2. Create a new section (instance) for testing
3. Install the integration version to be tested
4. Retrieve the API key for configuration

### Known Issue: Section Creation Timeout

[Tomasz Kowalski]: There is a known behavior where creating a section in the CRM can return a timeout error after **28 seconds**, but the section is actually created successfully in the background.

[Tomasz Kowalski]: 
> We have a great wait timeout on 28 seconds and audience will finish their job and the section is kind of accessibly created.

This is a false negative: the timeout message does not indicate failure. The section should be checked to confirm it was created despite the timeout message.

### Pre-Configuration & Mappings

Once a section is created and the integration installed, **subscription mappings** must be configured to tell Apsis how to interpret CRM data fields. For the consent events feature, this means mapping CRM consent topics to Apsis consent categories (e.g., mapping a CRM field to "generic newsletters").

[Erik Andersson]: 
> I can't remember where we have the subscriptions...

The mappings configuration lives in a specific location within the test section (the exact location was not fully verbalized in this transcript, but appears to be accessible through the section configuration UI).

---

## Current Deployment Status & Next Steps

### Deployment Completion

At the time the major section of technical discussion concluded, the AWS CodeBuild deployment to the beta environment had completed successfully ("exceeded" status indicates the build succeeded). This means:

- All three to four services are now running the updated code in beta
- The consent events feature is now live in the beta environment
- Acceptance testing can begin immediately

### Remaining Acceptance Testing Tasks

[Erik Andersson]: The following tests must be completed before marking the feature as ready for production:

1. **FSC Enterprise 12.0 full sync**: Install version 12.0 on a beta test section, run full sync, verify consent events appear on timeline
2. **FSC Enterprise 12.0 real-time sync**: Modify a contact's consent in the CRM and verify the event reaches Apsis in real-time (note: this was not working in 12.1, so should be separately validated)
3. **Generic connector validation**: Run the same tests for at least one of: FSC Enterprise 12.1, Tribe, E-deal, or Microsoft Dynamics

[Erik Andersson]: I think I will do that because I have all of those credentials ready. I'm used to working in that environment.

The test plan was to complete at least the FSC Enterprise 12.0 testing before stopping for the day, with the remaining generic connector testing to follow.

---

## Infrastructure & Operational Context

### Why Dedicated QA Teams Were Valuable

[Erik Andersson]: 
> This was so much easier when we had the actual dedicated QA people that did all of this because they had their routine, they knew how they tested it and they had like all of their sections pre-pre set up.

The platform previously had dedicated QA staff who maintained pre-configured test sections, understood the testing routines, and could execute acceptance tests efficiently. The absence of this structure means developers must now set up testing infrastructure on-the-fly, which is time-consuming and error-prone.

### Customer Sensitivity & Escalation Patterns

Large FSC Enterprise 12.0 customers (which represent a significant portion of the customer base) are particularly sensitive to platform changes and regressions. When issues occur, they tend to escalate quickly to executive stakeholders, creating organizational disruption that extends beyond the technical fix. This cultural reality makes testing against 12.0 not just a technical best practice, but an organizational necessity.

---

## Key Takeaways

1. **Token Rotation Requires Two Updates**: When rotating GitHub credentials for deployments using AWS CodeBuild, update both the CodeBuild project's stored credentials AND the token reference in the buildspec file.

2. **The Dual-Authentication Architecture Is a Vulnerability**: Consider migrating to GitHub Actions or other single-point-of-control solutions to eliminate the need to update credentials in two locations.

3. **Consent Events Feature Works in Full Sync**: The feature successfully populates consent events (opt-in, opt-out) on customer timelines when triggered by full sync, though real-time webhooks require investigation on the CRM side.

4. **Generic Connector Code Is Genuinely Shared**: If a feature works for one generic connector (12.1, Tribe, E-deal, Microsoft Dynamics), it mathematically works for all of them in Apsis. Test one; validate all.

5. **FSC Enterprise 12.0 Is Non-Negotiable for Acceptance Testing**: Regardless of generic connector benefits, always test against FSC 12.0 because it's legacy code with custom functionality and your largest customers use it.

6. **Section Creation Timeouts Are False Negatives**: If creating a test section times out after 28 seconds, the section was likely created successfully anyway. Always verify.

7. **Real-time Webhook Absence Suggests CRM-side Issues**: When webhooks don't arrive but full sync works, the problem is in the CRM's webhook configuration, not Apsis.

8. **Losing QA Infrastructure Creates Testing Friction**: Without dedicated QA staff and pre-configured test sections, acceptance testing becomes a developer responsibility that's slower and more error-prone.

---

## Unresolved Questions & Open Items

1. **Why didn't real-time webhooks arrive from FSC Enterprise 12.1?** 
   - The full sync worked correctly, ruling out an Apsis-side issue
   - No log entries appeared in the Delta Sync Manager
   - Likely a CRM-side webhook configuration issue, but not confirmed

2. **Will FSC Enterprise 12.0 consent events work for both full sync and real-time sync?**
   - [Erik Andersson] planned to test this but did not complete it during this session
   - Expected to validate before end of next business day

3. **Is there an alternative to the dual-token authentication in CodeBuild?**
   - [Erik Andersson] mentioned GitHub Actions as a single-point-of-control solution
   - Full migration plan not discussed in this session

---

**Note**: Erik Andersson left the session before completing FSC Enterprise 12.0 acceptance testing due to low battery and personal commitments. Tomasz Kowalski and Michal Rosikiewicz remain available to continue or coordinate the remaining testing work.