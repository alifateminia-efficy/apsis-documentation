---
source_file: Testing & deploying Justin part 2.txt
domain: Apsis One Integrations
topics: [deployment pipeline, GitHub token rotation, consent timeline events, acceptance testing procedure, FSC Enterprise connectors, generic connectors, CodeBuild configuration]
speakers: [Erik Andersson (senior developer/lead), Michal Rosikiewicz, Tomasz Kowalski]
key_components: [CodeBuild, CloudFormation, Parameter Store, FSC Enterprise 12.0, FSC Enterprise 12.1, generic connectors, delta sync manager, real-time sync, full sync, consent timeline]
session_type: knowledge-transfer
---

## Session Overview

This session covers two main topics. First, Erik explains a deployment pipeline failure caused by GitHub token rotation — specifically that CodeBuild has two separate places where the token is used, and both must be updated. Second, the team walks through acceptance testing for a new feature: consent events appearing on the profile timeline. Erik demonstrates the feature working in staging via full sync and explains the required acceptance testing protocol for the beta environment, including which connectors must always be explicitly tested and why.

---

## GitHub Token Rotation Issue in CodeBuild Pipeline

### Root Cause

The deployment pipeline uses a GitHub token in **two separate locations**:

1. **Inside the deploy spec file** — the token is loaded from **Parameter Store** and set as an environment variable (`GITHUB_TOKEN`), used when downloading repositories via Git during the build steps.
2. **Inside the CodeBuild project source configuration** — used by CodeBuild itself to download the source code when the webhook is first triggered, *before* any build steps run.

When the token was rotated, only the Parameter Store value (location 1) was updated. Location 2 — the CodeBuild project's source credential — was not updated, so the pipeline failed at the very first step before it ever reached the deploy spec.

### Fix

[Erik Andersson]: Go into the CodeBuild project → **Manage account credentials** → disconnect the existing GitHub connection → re-add the new token.

### Why This Wouldn't Happen with GitHub Actions

> "You don't need to do this if you are using GitHub Actions, because then we have this in one place instead."

This is one reason the team is considering moving away from the current CodeBuild-based pipeline.

---

## Deployment Pipeline Structure

- The pipeline deploys via **CloudFormation**.
- **Three or four services are deployed in parallel** during the deployment steps.
- Beta environment deployment is triggered through CodeBuild.
- Deployment status is monitored under **CodeBuild → Deployments** (not CodeCommit).

---

## Consent Events on the Profile Timeline — Feature Overview

### What the Feature Does

When a CRM profile has consent updated (either via real-time webhook or full sync), the consent event should appear on the **profile timeline** in Apsis One. The event entries are labeled as:

- `FSC Enterprise 12.1 consent opt in`
- `FSC Enterprise 12.1 consent opt out`

The entry also specifies the channel: **email** or **SMS**, and includes metadata such as which full sync it came from and which topic it applies to.

### Consent Sources

Consent events can arrive via two paths:
1. **Real-time sync (webhooks)** — should appear within 1–3 minutes of a change in the CRM.
2. **Full sync** — consent is batched and processed as part of the full sync job.

> "Of course you could potentially need to deploy something manually to reject all real-time syncs if you really want to test this from the full sync."

### Batching Behavior

FSC Enterprise batches consent records — they do not send one consent event at a time. They send batches of approximately 10–15 consent records together.

### Staging Verification

Erik verified the feature in **staging** by:
1. Modifying a consent field (generic newsletters mapping) on a test profile in the CRM (account 253, staging environment).
2. Waiting for real-time webhook — **no webhook arrived** (no log entries in the delta sync manager at all; suspected CRM-side issue).
3. Falling back to a **manual full sync** via the producer.
4. Checking the consumer logs and confirming the consent was imported and the timeline entry was created correctly, including the full sync reference and topic.

Result: Feature confirmed working via full sync in staging.

---

## Acceptance Testing Protocol for Beta — Consent Timeline Feature

### Required Test Environments

When the beta deployment completes, the following must be verified:

**1. FSC Enterprise 12.0 (legacy, non-generic connector)**

This must *always* be tested explicitly. Rationale:
- 12.0 is a **legacy connector with custom functionality** — it does not use the generic connector code path.
- The majority of customers are still on FSC Enterprise 12.0.
- These customers are historically very reactive to regressions.

> "Those customers historically, as soon as something is up, they get very irritated and send emails straight to the CEO, and then you need to deal with overreactions for days after. So always include FSC Enterprise 12.0 when you test."

A dedicated test instance exists for FSC Enterprise 12.0. Erik has the credentials and typically handles this verification himself.

**2. At least one generic connector**

Any one of the following is sufficient:
- FSC Enterprise 12.1
- Tribe
- Microsoft Dynamics
- Byside (⚠️ *transcript unclear — "By Site"*)
- E Deal

Rationale:
> "If it works for one of those, then it is working as it should on Apsis side, because the logic for us is the same regardless of which of those integrations it is. If it is not working, then that is because the CRM is not doing what they should."

In other words: all generic connectors share the same code and flow on the Apsis side. Testing one is equivalent to testing all of them from Apsis's perspective.

### Acceptance Test Steps (per environment)

1. Install the relevant integration (e.g., FSC Enterprise 12.1) on a section in the beta account.
2. Configure subscription/consent mapping.
3. Run a full sync.
4. Verify that the consent timeline entries appear on the profile page with correct labels, channel, topic, and sync reference.
5. Optionally verify real-time sync by modifying a consent in the CRM and waiting 1–3 minutes.

### Known Gotcha: Section Creation Timeout

[Tomasz Kowalski]: Creating a section in Apsis One can time out (gateway timeout is approximately 28 seconds), but the section is actually created successfully in the background — it's a **false negative**. Check whether the section exists before retrying.

### Known Gotcha: Installing Integration on Correct Section

[Tomasz Kowalski caught this live]: Make sure the integration is installed on the intended acceptance test section, not the default section.

---

## Debugging: Real-Time Sync Not Arriving

During the staging test, no real-time webhook events arrived from the CRM. Diagnostic approach:

1. Check **CloudWatch log streams** for the relevant service — no streams present indicated nothing had arrived.
2. Check the **delta sync manager** — no log entries at all confirmed the events were never sent by the CRM.
3. Conclusion: The CRM (FSC Enterprise instance) was not sending webhooks, likely a CRM-side issue unrelated to Apsis code.

> "If nothing is not working, then that is because the CRM is not doing what they should... if you have tried it for one integration, you have essentially tried it for everything that uses it."

The feature was confirmed working by falling back to full sync.

---

## Key Takeaways

1. **CodeBuild GitHub token rotation requires two updates**: the Parameter Store value AND the CodeBuild project's source credential (via Manage Account Credentials). Failing to update both will silently break the pipeline at source download time.
2. **Consent timeline events** are now surfaced for both real-time and full sync paths, labeled with connector name, opt-in/out status, channel (email/SMS), and topic.
3. **Always test FSC Enterprise 12.0 explicitly** for any integration-related feature — it is a legacy non-generic connector with a large, vocal customer base.
4. **Testing one generic connector covers all generic connectors** on the Apsis side (FSC Enterprise 12.1, Tribe, Microsoft Dynamics, E Deal, etc.) because they share the same code path.
5. **If real-time sync events don't arrive**, check the delta sync manager first — absence of log entries there means the CRM never sent anything; the issue is external to Apsis.
6. Section creation in Apsis One may return a timeout but succeed in the background — always verify before assuming failure.

---

## Unresolved Questions / Action Items

- [ ] **Tomasz**: Install FSC Enterprise 12.1 on a beta section, run full sync, verify consent timeline entries appear. (Assigned during session.)
- [ ] **Erik**: Verify consent timeline for FSC Enterprise 12.0 on beta (real-time sync + full sync). Was in progress at end of session; result not confirmed on-camera.
- [ ] **Real-time sync gap**: Investigate why no webhooks were received from the FSC Enterprise staging instance — no log entries in delta sync manager. CRM-side issue suspected but not confirmed.
- [ ] **Migration away from CodeBuild**: Mentioned as a reason to move to GitHub Actions (single token location), but no timeline or decision captured in this session.
- [ ] ⚠️ **Ambiguity**: "By Site Shop" mentioned as a generic connector alongside Microsoft Dynamics and E Deal — spelling/exact product name unclear from transcript.