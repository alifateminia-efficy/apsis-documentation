---
source_file: Erik - Dynamics setup part 2 & code review of event listeners for campaigns.txt
domain: Apsis One - Integrations
topics: [Dynamics 365 Setup, User Access Control, Azure AD Configuration, OAuth Applications, Event Listener Architecture, Campaign Management, CRM Connector Flow]
speakers: [Erik Andersson, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [Dynamics 365, Power Platform, Azure Entra ID, OAuth Applications, Event Listeners, Outbound Manager, Campaign Activities, Apsis One Platform]
session_type: knowledge-transfer|debugging-session|setup-troubleshooting
---

## Session Overview

This session covered two main areas: (1) a planned refactor of the outbound manager and campaign patching flow due to CRM system requirements changes, and (2) hands-on troubleshooting of Dynamics 365 access issues for team members. The team discovered a critical permission model bug where users with "administrative" access could manage the Dynamics environment but could not read data due to missing Microsoft 365 licenses. The session also included Azure AD management, OAuth application configuration review, and cleanup of test users and applications.

---

## Part 1: Outbound Manager Refactor and Event Listener Architecture

### Current Campaign Creation Flow

[Erik Andersson]: The current flow works as follows:
1. Email campaign tool makes a request to the outbound manager
2. Outbound manager makes a request to the CRM system saying: "Someone has created an email campaign here with this Activity ID and name"
3. CRM system responds confirming the campaign creation

### Child Activities Concept

[Erik Andersson]: For event tool and marketing automation activities, there is a concept of **child activities** — these are sub-activities. For example:
- An event tool activity connects to email send-outs
- Form activities for registration might exist under the same parent activity
- Today, all of these are sent to the CRM system

> "Currently, we register all of these listeners in Apsis first, so we create an event listener for this email send-out before we do a request to the CRM system, and the same thing for all of the event tool activities."

### The Problem: E-deal/FCC Corporate Requirements

[Erik Andersson]: E-deal (formerly called FCC Corporate) has stated they are **not interested in child activities at all** — they only want the event tool activity itself. They don't want email send-outs, form submissions, or other sub-activities sent to them.

**The Issue**: The current flow registers listeners in Apsis One **before** requesting from the CRM system. When E-deal creates an activity:
- They only create an activity for the event tool
- But Apsis One has already registered listeners for form submissions, emails, and other events
- These events still get sent to the CRM system despite E-deal never creating activities to connect them to
- Result: orphaned events being sent to a system with no corresponding parent activity

### Proposed Solution: Reverse the Order

[Erik Andersson]: The refactor should:
1. First make the request to the CRM system
2. Have the CRM system tell us which activities they're actually interested in receiving
3. **Only then** register the listeners for those activities

> "Instead of us taking like the whole list for registering the listeners, we should only take the response from the CRM system as that input instead."

### Code Review Notes

[Erik Andersson]: The pull request implementing this change goes far beyond what's necessary:

> "In essence, we should only need to move like one code block before and then take the output of this request as the input for what when we register the listener... But this pull request it goes like I said way and beyond and does some more refactoring."

**Status**: The implementation is over-engineered. Erik estimates it would be easier to start from scratch. This has been moved to the integration to-do board for a fresh implementation rather than trying to salvage the existing PR.

---

## Part 2: Dynamics 365 Access Troubleshooting and User Permission Configuration

### Initial Problem: "Security Group Not Set" Error

[Tomasz Kowalski] was unable to access the Dynamics 365 instance in Integration Development environment, receiving a "Security Group Not Set" error and HTTP 404 responses.

### Root Cause Discovery: Administrative vs. Read-Write Access Model

After extensive troubleshooting, the team discovered the **critical permission bug**:

> [Erik Andersson]: "It feels like you are running into the same issue here now that you seem to have like administrative rights, but you cannot access the data which kinda sounds like it."

**The Issue**: Users with "**administrative**" access level in Dynamics could manage the environment but could not read the actual data. The system appeared to require both:
1. Administrative capability (for management operations)
2. Read-Write access level (for data access)
3. Appropriate Microsoft 365 licenses assigned

### Solution: License Assignment and Access Level Change

The fix required:
1. Assigning appropriate Microsoft 365 licenses to the user in Entra ID (Azure AD)
2. Changing the user's access level from "administrative" to "read/write"
3. Refreshing/re-enabling the user after changes

[Erik Andersson]: "Now I understand — the main culprit is the read write, but in order to set the read write you have to add the licences otherwise it will go back to administrative."

**Key licenses required for Dynamics access**:
- Microsoft Dynamics 365 Sales Enterprise Edition (limited availability — only 3 licenses)
- Power Automate license
- Microsoft Fabric license (optional for most use cases)

[Erik Andersson]: "If you have sales, you can see leads here and you can see marketing lists, but that's the only difference. The sales should do. You should still be able to see contact contacts etcetera."

### Why This Wasn't Obvious

[Michal Rosikiewicz]: "I had this security group issue error and so on."

The error messages were misleading — they appeared to be permission or security group configuration issues when the actual problem was that the user had administrative rights but no read-write capability to see data.

---

## User Setup and Access Configuration

### Creating New User: Lucas Grabowski

The team created a new dedicated user (Lucas Lab) in the Integration Development environment with the following steps:

1. Create user account in Entra ID
2. Set as "Member" type (not Guest — guests cannot make changes)
3. Assign roles:
   - Global Administrator
   - Platform Administrator
   - Dynamics Administrator (for context)
4. Assign Microsoft 365 licenses:
   - Power Automate Free
   - Microsoft Fabric (trial)
5. Change access level to "read/write" (after licenses are assigned)

[Erik Andersson]: "If you are a guest, you can't change anything. But if you are a Member, then you can do essentially whatever."

### External User vs. Dedicated User Trade-off

[Erik Andersson] explained the setup strategy:

**External FCC User** approach:
- Can theoretically work after initial setup is complete
- Problem: Cannot perform initial OAuth application permission setup because external users cannot add permissions to the FCC organization
- Requires global admin involvement for this step

**Dedicated Lab User** approach:
- Works for all operations after creation
- Setup requires initial creation with proper licenses
- Easier to manage permissions independently

> [Erik Andersson]: "You cannot do the initial setup with the external FCC users because that's when we felt like when we tried... it is trying to add permission to FEC for the application and that of course we are not allowed to do because we are not global administrators."

---

## User Cleanup and Security

### Test and Abandoned Users Deleted

The team removed several users that were no longer needed:
- **Benjamin**: Global administrator test account (disabled)
- **Janita**: Was already disabled by Erik previously
- **Shraddha/Sravidya**: Parental leave — kept enabled as still technically employed
- **Felix**: Former team member (deleted)
- **Admin Tool**: Test account created for experimental admin setup (deleted)
- **External Wukash Grabowski**: FCC external user (deleted — not needed with dedicated users)

### Users Retained with Notes

- **Test Not Admin**: Disabled test user created to explore whether installation could work without global admin (determined not possible — global admin required for OAuth app permissions)

- **Najeeb**: CRM consultant and developer — disabled but not deleted. Kept for potential future access. Marked with company name and role for context.

- **Gustav Estalund** (Gustav Lab): Dynamics consultant at CRM Consultants. Disabled but retained. Primary architect behind the Dynamics solution design. Handles solution design while Najeeb handles implementation.

- **Apsis Lead Integration User** (created 2015): Predates the integration platform by ~4 years. Likely connected to legacy Pro or Lead system. Kept because deleting it could break unknown dependencies.

[Erik Andersson]: "This user is created for the Pro or the legacy legacy lead system... It could very well be like the main user for the Pro access, in which case bad stuff would happen."

---

## Azure Entra ID and OAuth Application Management

### OAuth Application Architecture

The team reviewed OAuth applications registered in Azure for Apsis One integrations:

**Current Production Applications**:
- `Apsis One Sandbox` — for local development on `localhost`
- `Apsis One Staging 2` — current staging environment
- `Apsis One Beta` — beta environment
- `Apsis One APEC Beta` — APEC-specific beta (environment no longer exists)
- `Apsis One EU Prod` — production environment

Each application has configured redirect URIs for callbacks to specific environments.

### Redirect URI Configuration

[Erik Andersson]: "Here you configure like the redirect URLs like which which URLs are allowed to like or which URL or like callbacks allowed."

Example redirect URIs per environment:
- Staging: `apps-one-stage.cloud/integration/callback`
- APEC Beta: (environment deleted, application can be disabled)
- Production: `apps-one-prod.cloud/integration/callback`

### Disaster Recovery and Secret Management

[Erik Andersson]: "If for some reason you were to lose access to the secret, let's say that AVS, Ireland is naked. And the backup of the secret is also has also failed. Then you would need to generate a new secret for the for the authentication."

**Process for secret rotation without customer re-installation**:
1. Generate new client secret for the same application ID
2. Update the secret value in the secrets manager (Azure Key Vault/AVS)
3. Customers do NOT need to reinstall if the Application ID remains the same
4. Only the internal secret value changes, not the public-facing app ID

> "If you do that, then I don't think the customers would, they would not need to reinstall because then you are using the same like you're using the application with the same IED as they have already."

**Historical incident**: During a previous staging environment recreation, all test environments using that environment's OAuth app stopped working because a new application ID was generated.

### Applications Marked for Deletion

- **Sandbox applications** (Sandbox, Sandbox 2): Removed during cost reduction — environments deleted
- **Dev**: No certificate, likely inactive
- **APEC Beta**: Environment no longer exists
- **Test Client**: No credentials, not used
- **Permission Test Client**: No credentials, not used
- **Temp Beta**: No credentials, not used
- **Web API Test**: Has expired credentials but still contains secrets from former developer "Hayes" (from ~2019)

### Sensitive: Applications Requiring Owner Access

**Issue**: The application owner "Nicolas" has a **disabled account** in Entra ID. This user created/owns:
- Apsis One Sandbox
- Apsis One Staging 1 (expired)
- Apsis One Staging 2
- Apsis One Beta
- Apsis One EU Prod
- Several others

[Erik Andersson]: "And Nicholas is the owner of all almost all apps... Uh. ****. OK, this is a then it was very, very good that we. Very, very good that we did this."

[Erik Andersson]: "I have the credentials for his account saved for this specific reason... I am extremely paranoid by nature or things like this and so far I have only been proven right."

**Why this paranoia was justified**: During a 2-week MA (Marketing Automation?) outage, Erik was able to recreate the system from logs because he had saved access to Nicolas' account. Without this, 300,000 messages would have been permanently lost.

> "Luckily, because of that, we didn't lose 300,000 messages in MA because we could recreate it from the logs. Otherwise, we would have been leaving deep crap because we would have lost two weeks worth of MA traffic, which is like, oops."

---

## Generic Connector vs. Legacy Dynamics Flow

### Why Global Admin is Required for Legacy Dynamics

[Erik Andersson]: "Like it is a hard requirement on dynamics that whenever you add this like oaf user to the organisation, it has to be a global admin because it is such a sensitive operation and the whole flow is dependent on a oaf user being added to the organisation."

> "We can't remove that requirement for the legacy dynamics. This is however not the case for the generic for the generic connector, because then we we connect to an intermediate service. So person like apps is will never touch dynamics in that flow, we only communicate with the intermediate service and the intermediate service handles everything with dynamics."

**Key architectural difference**:
- **Legacy Dynamics connector**: Apsis One adds itself as an OAuth user directly to customer's Dynamics instance → requires global admin approval
- **Generic connector**: Apsis One connects to an intermediate service → intermediate service handles Dynamics interaction → Apsis One never touches Dynamics → global admin not required

This distinction is important for future customer onboarding and migration strategies.

---

## Integration Development Environment (Staging) vs. Production

The team worked in the **Integration Development** (staging) environment, not production, specifically because:

> [Erik Andersson]: "This environment that we are doing in right now, this is still only the test environment. So nothing here is destructive for customers. If we were doing this in the apps is one environment, we might want to be a bit more careful."

---

## Challenges and Workarounds

### Browser Session/Token Issues

During troubleshooting, the team encountered issues with browser caches holding old authentication tokens:

[Michal Rosikiewicz]: "It's always, it's our platform... I didn't delete the cookie sometimes still logged in."

Solution: Close all browser tabs and completely clear session state before retrying authentication.

### User Enable/Disable Reset Bug

[Erik Andersson]: "When I set the read and write administrative thing... It was me, it it was... Previously I want to press press refresh and then return to administrative."

After changing user access levels, the system would sometimes revert changes or require multiple attempts. Full browser refresh or closing all tabs often resolved this.

### External User Invitation Complications

During initial setup, an FCC external user was invited to the environment. This caused permission issues:

[Tomasz Kowalski]: "Yeah, remove the external user from from my PC that we were invited... From FCC."

Removing this external user resolved cascade permission failures related to organization-level permissions.

---

## Key Learnings and Handover Notes

### Administrative Access Limitation Discovery

This session revealed a **previously unknown limitation** in how Dynamics 365 access is controlled:

> [Erik Andersson]: "I am going to show you here. What? What? I am actually looking at so because there's one one thing which is down here here you have this like administrative... I then looked here on the mirror lab and you are administrative. And as soon as I change you from administrative to read, write as I am..."

This finding should be documented in onboarding procedures for future team members.

### Handover Progress and KPI Tracking

[Erik Andersson]: "I actually explicit explicitly in my sign off agreement. I have like four or five. What is it called? Like KKPI or whatever it's called. That's like the handover quality is gonna be measured about."

Erik is tracking handover completion through KPI checkboxes. The team should evaluate:
- Theoretical knowledge transfer completion
- Whether written guides need expansion
- Whether additional sessions are needed in specific areas

[Erik Andersson]: "I don't want to sit there in February and then hear someone say like no the handover is not the handover quality is not good enough."

---

## Unresolved Questions and Future Work

1. **Hassan Action Adapter**: Unknown purpose. Has configuration with `localhost:mock` endpoints. Marked for investigation but not deleted yet.

2. **Integration Tests Application**: Purpose unclear. Needs documentation.

3. **Web API Test Application**: Has legacy secrets from former developer "Hayes" (2019). Should be cleaned up but marked for later action.

4. **Shraddha's Access**: Employee on parental leave — policy unclear on whether access should be maintained or revoked. Currently kept active.

5. **Apsis Lead Integration User**: Unknown dependencies. User created in 2015 predates current platform. Safest to keep but should be documented.

6. **Future Secret Rotation Process**: Never tested in practice. Team should document and test the secret rotation workflow for disaster recovery scenarios.

---

## Action Items and Next Steps

- [Erik Andersson] to finalize credentials cleanup for Nicolas' disabled account (beginning of next week/Monday)
- Delete expired OAuth applications: APEC Beta, Test Client, Permission Test Client, Temp Beta (low priority)
- Document purpose of mysterious applications (Hassan Action Adapter, Integration Tests) or delete
- Consider Lucas Grabowski setup as template for future team member onboarding
- Evaluate Dynamics access configuration documentation for clarity
- Schedule follow-up sessions on:
  - Hands-on coding/development environment setup
  - Event listener implementation details
  - Campaign management workflow

---

## Session Duration
1 hour 51 minutes 4 seconds