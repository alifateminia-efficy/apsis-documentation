---
source_file: Testing & deploying Justin part 2.txt
domain: Apsis One Integrations
topics: [deployment pipeline, GitHub token rotation, consent events on timeline, acceptance testing procedure, FSC Enterprise connectors, full sync vs real-time sync]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [CodeBuild, CloudFormation, Parameter Store, GitHub token, FSC Enterprise 12.0, FSC Enterprise 12.1, delta sync manager, generic connector, legacy connector]
session_type: knowledge-transfer
subdomains: ["Different Types of Connectors", "Lead creation"]
---

## Session Overview

This session covers two main areas. First, Erik explains a deployment failure caused by a GitHub token rotation that was only applied in one of two places where the token is used in the CodeBuild pipeline — a known pain point of the current setup. Second, the team walks through acceptance testing for a new feature: consent events appearing on the Apsis One profile timeline, synced from CRM. Erik demonstrates testing in staging, explains why both FSC Enterprise 12.0 and at least one generic connector must always be tested, and begins setting up the beta environment for the same verification.

---

## GitHub Token Rotation Bug in CodeBuild Deployment Pipeline

### Root Cause

The deployment pipeline uses the GitHub token in **two separate places**:

1. Inside the **deploy spec file** — the token is loaded from **Parameter Store** and set as the `GITHUB_TOKEN` environment variable, then used when downloading repositories via Git.
2. At the **CodeBuild project source level** — CodeBuild downloads the source code from GitHub when the build is first triggered (before the deploy spec runs). This uses a credential configured directly on the CodeBuild project itself, under **Manage Account Credentials**.

When Erik rotated the GitHub token, he updated it in the deploy spec / Parameter Store but did not update the credential at the CodeBuild project source level. As a result, the build failed before it even reached the step that used the Parameter Store value.

> "I updated the token for this step inside the deployment file. However, I had not updated the token where we download the source code after the web trigger in the first place, so it never even got to where it utilized the GitHub token value."

### Fix Applied

Erik went into the CodeBuild project → **Manage Account Credentials**, disconnected the existing GitHub connection, and re-added it using the new token.

### Why This Is a Known Problem

[Erik Andersson]: This is one reason the team wants to move away from the current setup. **GitHub Actions** would consolidate token usage into one place, eliminating this class of failure.

---

## Consent Events on the Profile Timeline — Feature Description

A new feature adds CRM-sourced **consent events** to the Apsis One profile timeline. These events appear as entries like:

- `FSC Enterprise 12.1 consent opt in`
- `FSC Enterprise 12.1 consent opt out`

Each entry also specifies the channel (e-mail or SMS) and the topic/subscription the consent relates to.

### How Consent Reaches Apsis

Consent can arrive via two paths:
1. **Real-time sync (webhooks)** — the CRM sends consent changes as they happen; should appear within 1–3 minutes.
2. **Full sync** — consent is included in the periodic bulk sync.

Both paths should result in the consent appearing on the timeline. The full sync entry also shows which full sync it came from and the source.

### Batching Behavior (FSC Enterprise)

[Erik Andersson]: FSC Enterprise batches consent updates the same way Apsis does — it does not send one consent event at a time. It sends batches of 10–15 consents together.

---

## Debugging: Real-Time Sync Not Arriving

During staging testing, Erik expected webhook-delivered consent events to appear within minutes after modifying consent in the CRM. No log streams appeared in the delta sync manager.

**Conclusion**: There were no log entries at all, meaning the events never reached Apsis. This was not an Apsis-side bug — the CRM (FSC Enterprise instance used for testing) was simply not sending any webhooks at that time. Something was wrong on the CRM side.

> "The reason I don't see any logs is because nothing has actually come to us. But the feature as such is working, as we can see [from the full sync result]."

Erik proceeded with a **manual full sync** to verify the feature works, which confirmed it does. Real-time sync verification for FSC Enterprise 12.0 was deferred.

---

## Acceptance Testing Procedure for Integration Features

### Rule: Always Test FSC Enterprise 12.0

[Erik Andersson]: **FSC Enterprise 12.0** is a **legacy connector** (not generic). It has custom functionality and a large base of important customers. Historically, as soon as anything breaks for those customers, they escalate aggressively — including emailing the CEO directly — triggering days of overreaction handling.

> "Always include FSC Enterprise 12.0 when you test. We have a test instance for it."

### Rule: Test At Least One Generic Connector

You must also verify the feature against at least one **generic connector**: FSC Enterprise 12.1, Tribe, Microsoft Dynamics, Visma Bizsite Shop, or E Deal.

**Rationale**: All generic connectors run on the same Apsis-side code and flow. If the feature works for one generic connector, it works for all of them on Apsis's side.

> "If it works for one of those, then it is working as it should on Apsis's side, because the logic for us is the same regardless of which of those integrations it is."

If something is *not* working for a generic connector, there is a ~99.99% probability the CRM is behaving incorrectly, not Apsis. The real-time sync failure during this session (no log entries in the delta sync manager at all) is a concrete example of this pattern.

### Summary of Required Test Scope for This Feature (Consent Timeline)

| Integration | Type | Environment | Who |
|---|---|---|---|
| FSC Enterprise 12.0 | Legacy | Beta | Erik |
| FSC Enterprise 12.1 | Generic | Beta | Tomasz |

Erik set up a section in beta with FSC Enterprise 12.0 installed and a full sync running before the session ended.

---

## Section / Environment Setup Notes

- When **creating a section** in Apsis One, the request can appear to time out, but the section is actually created successfully in the background (the timeout is a false negative — the gateway times out at 28 seconds but Apsis completes the job).
- [Tomasz Kowalski]: Confirmed this is a known issue. The section will be accessible after a moment.
- Erik noted: testing was previously easier when dedicated QA staff had pre-configured sections, known routines, and stored credentials. That infrastructure no longer exists and the team now sets up test environments ad hoc.

---

## Key Takeaways

1. **Token rotation in CodeBuild requires two updates**: the Parameter Store value used in the deploy spec, AND the credential on the CodeBuild project's source configuration (Manage Account Credentials). Missing either one will cause the build to fail at different stages.
2. **Moving to GitHub Actions** would eliminate this two-location token problem.
3. **Consent events** from CRM now appear on the Apsis One profile timeline, sourced from both real-time webhooks and full syncs.
4. **Acceptance testing rule**: always test FSC Enterprise 12.0 explicitly (legacy, high-stakes customers) AND at least one generic connector. Generic connectors are interchangeable on Apsis's side — one passing test covers all of them.
5. **If no log entries appear in the delta sync manager**, the problem is almost certainly on the CRM side, not Apsis's side.
6. **Section creation timeout** is a known false negative — the section is created successfully even when the UI reports a timeout.

---

## Unresolved Questions / Action Items

- [ ] **Tomasz**: Install FSC Enterprise 12.1 on a beta section, run a full sync, and verify consent events appear on the profile timeline.
- [ ] **Erik**: Verify consent timeline feature on FSC Enterprise 12.0 in beta — both full sync and real-time sync. Session ended before the full sync result was confirmed.
- [ ] **Erik**: Investigate why the FSC Enterprise test instance was not sending real-time webhook events during this session (no entries in delta sync manager).
- [ ] **Open**: Migration away from current CodeBuild token setup toward GitHub Actions — mentioned as desirable but not scheduled.