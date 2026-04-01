---
source_file: Erik & Daniel - Membrane IPaaS & SMS alternative routing.txt
domain: Apsis One Integrations
topics: [Membrane iPaaS integration architecture, Generic connector reuse, Authentication flows, SMS alternative routing via Infobip, Tags limit and pricing mechanics, Custom attributes data model]
speakers: ["Daniel Rönnow (Product)", "Erik Andersson (Engineer, last day)", "Lukasz Grabowski (Engineer)", "Ali Fateminia (Engineering Lead)", "Henrik Boye (Unknown role)"]
key_components: [Membrane iPaaS, Apsis One Integrations, Generic Connector, One API, Infobip SMS, Audience/Tags, Back Office, Integration Webhooks]
session_type: knowledge-transfer
---

# Session Overview

This session covers two primary topics. First, a technical architecture discussion about how to integrate Membrane (an iPaaS platform) with Apsis One's existing integrations infrastructure, including two proposed approaches and their trade-offs, with Erik Andersson (on his last day) serving as the primary knowledge holder. Second, a scoping discussion of SMS alternative routing via Infobip sub-accounts as a margin-improvement initiative. The session also touches briefly on the tags limit in Audience and potential pricing mechanics around attributes and tags. Several assumptions about Membrane's capabilities were identified as needing validation in a follow-up meeting.

---

## Membrane iPaaS — Background and Motivation

**Membrane** is an iPaaS (Integration Platform as a Service) solution that supports a large catalog of standard integrations and allows new integrations to be developed rapidly, including with AI tooling assistance.

A prior demo session was held with Membrane (recording not available — it was on their Zoom/Google Meet invite, not recorded). Erik Andersson and Lukasz Grabowski attended and challenged Membrane technically.

**Business motivation (Daniel):** Two goals:
1. Enable a large number of standard integrations (Salesforce, HubSpot, Pipedrive, etc.) without building each one internally — Membrane provides these out of the box.
2. Reduce internal R&D dependency for new integrations — Membrane's AI tooling can build connectors from API documentation. Example: Meta Leads integration could be built against Meta's documentation using Membrane's AI, without Apsis engineering involvement. External support can also be brought in.

> "We can utilize their AI tool to build those integrations... you only need to do that once and we don't need to involve IT in our platform or depend on our platform to do it." — Daniel

---

## Membrane Integration — Two Architectural Approaches

### Approach A: Membrane-Driven (Use Membrane's UI / iframe)

In this approach, Apsis would embed Membrane's UI (iframe or popup) within the Apsis One integrations page. Customers would perform field mapping and authentication inside Membrane's interface. Membrane would then **call Apsis** to fetch data needed to populate its UI.

**Problems identified with Approach A:**
- Apsis would need to expose new API endpoints for Membrane to call (e.g., to fetch available fields, attributes, subscription lists). These do not currently exist in One API.
- One API would need to understand and support Membrane's authentication scheme, potentially impacting other customers — needs to be architecturally segregated.
- Creates an inconsistent UX: customers would have two different UI paradigms for native integrations vs. Membrane integrations.
- Higher overall complexity and cost.
- [Lukasz]: "Suddenly the UI for the customer becomes kind of inconsistent... to me it sounds like a more complex solution. I would go to that if there is a blocker in the more recommended one."

### Approach B: Apsis-Driven via Generic Connector (Recommended)

[Erik Andersson - primary architect of this proposal]

In this approach, **Apsis calls Membrane** rather than Membrane calling Apsis. Membrane acts as a **proxy/middleware layer** between Apsis and the target CRM system.

**Core concept:**
- Membrane exposes what it calls **"actions"** — essentially callable endpoints that perform specific operations against a connected CRM.
- Apsis defines the contract: Membrane actions must return data in a specific JSON schema that the **generic connector** already understands.
- Apsis calls these actions from within existing integration flows (field mapping page, subscription download, full sync, etc.).

**Example action mapping:**

| Apsis Integration Event | Membrane Action Called |
|---|---|
| User opens field mappings tab | `get_metadata` / `get_schema` |
| Download subscription/consent lists | `get_subscriptions` |
| Full sync trigger | `get_contacts` |
| Outbound event (future) | TBD |

**Key advantage — Generic Connector reuse:**

> "You can most likely reuse the generic connector flow for most of this because we can pretend that membrane is a connector implementation and get a lot for free." — Erik

The existing **generic connector** already handles integrations for systems like eDeal, FSA Enterprise 12.1, and the previously-supported Maxo — all using **identical code** because each CRM implements the same endpoint contract. Membrane would do the same: no matter which CRM is behind Membrane, it normalizes the response into the agreed format.

**On "do we need to build per-integration?"** — No, the **logic** is written once. Configuration (display name, integration card metadata, etc.) is set up per-integration but involves no programming — it's database-level configuration.

**Authentication in Approach B:**

For **Apsis → Membrane** calls: Standard API key / token-based auth (similar to existing OAuth2 flows for CRMs, e.g., existing Microsoft Dynamics OAuth dance). This is handled within Integrations, not One API.

For **Membrane → Apsis** (inbound webhooks): Membrane calls Apsis integration webhook endpoints. This is treated as **yet another webhook secret scheme**, which already has infrastructure in Integrations.

Proposed flow for inbound webhook registration:
1. Customer installs a Membrane connector (e.g., Salesforce) in Apsis.
2. Apsis generates a secret, stores it, and registers a webhook in Membrane — specifying the callback URL (which encodes account name, section, integration ID) and the secret.
3. When the customer's CRM notifies Membrane of a change, Membrane calls the Apsis webhook URL and includes the secret.
4. Apsis verifies the incoming secret matches the stored value for that account/section/integration.

> "It's not that membrane will say here is a random request for someone — they will need to specify in the path where this is going." — Erik

**⚠️ ASSUMPTION — needs validation:** Whether Membrane supports registering webhooks with a customer-provided secret that it then passes back on each call. This is the mechanism Apsis uses for all existing CRM webhook integrations, but it has not been confirmed for Membrane.

**Account-level vs. section-level mapping:**

- Lukasz's preference: keep the Apsis ↔ Membrane relationship at **account level** (one Apsis account = one Membrane customer), not section level.
- Rationale: sections are planned for eventual removal; simpler data model; the CRM data on Membrane's side is the same regardless of section.
- Integrations themselves can still remain scoped to sections (as they are today).
- Henrik noted: integrations are currently connected to sections, not accounts — this should not be a blocker, just a design consideration.

**OAuth / CRM authentication by the customer:**

When a customer connects a CRM (e.g., Salesforce) through a Membrane-backed integration, the OAuth flow happens **inside Membrane**, not inside Apsis. Apsis triggers this by directing the user to Membrane's existing UI (popup or equivalent) for that step only.

> "In Apsis you would click on connect to Salesforce — we should in this case actually utilize Membrane's existing UI for this." — Erik

This is the **one place** where Membrane's iframe/popup UI is recommended, even in Approach B.

---

## Membrane Integration — Open Questions and Prerequisites

Erik documented these as prerequisites that must be confirmed before implementation estimates are reliable:

1. **Can Membrane actions be configured to return data in a format Apsis specifies?** — The generic connector reuse only works if Membrane can normalize all CRM responses into Apsis's expected JSON schema, regardless of which CRM is behind it.
2. **Can Membrane register webhooks with a caller-provided secret?** — Required for the inbound authentication approach described above.
3. **Is the data schema setup per-CRM integration or once globally?** — Erik believes it may be partially generic (e.g., "if field type is string, do X; if float, do Y") but this needs confirmation.
4. **How does the Membrane token / customer identity flow work end-to-end?** — Still partially based on assumptions from the demo meeting.

> "I should emphasize that most of these suggestions are based on prerequisites I listed in the document — this is the research that needs to be done." — Erik

**Confidence level on scope:** Erik explicitly stated he is **not confident** the happy-path estimate will hold — surprises from Membrane's limitations are expected. Lukasz noted this risk exists regardless of which approach is taken when integrating with any new external system.

**Terminology note:** Ali proposed calling the initial work **MVP** rather than **POC** — the POC was Membrane's responsibility; what Apsis would build should be production-intent.

---

## Membrane Integration — Recommended Next Steps

1. Erik to document the two critical questions (above) explicitly for whoever leads the Membrane follow-up.
2. Lukasz / Greg to do initial digging using Membrane's documentation and Erik's written materials.
3. Daniel to schedule another meeting with Membrane — come prepared with the specific open questions to validate assumptions.
4. Once assumptions are confirmed, estimate effort based on Approach B (generic connector route).
5. Authentication flows to be contained within the Integrations domain (not One API) for now. Expanding to One API (for customers to use Membrane as a proxy to the full public API) is a future discussion, explicitly deferred.

---

## SMS Alternative Routing via Infobip — Background

**Business context:** Apsis identified high costs and low margins in its SMS business. After renegotiating with vendors (including alternatives to Infobip), direct competitors were not significantly cheaper. Infobip's counter-offer: provide **alternative routing options** via sub-accounts on Apsis's existing Infobip main accounts.

**Three routing tiers proposed:**

| Tier | Deliverability vs. Direct | Cost vs. Direct |
|---|---|---|
| Direct (current) | Baseline | Baseline |
| Alternative 1 | ~5% lower | ~10% cheaper |
| Alternative 2 | ~7–10% lower | ~25% cheaper |

**Goals:**
1. Migrate all customers to Alternative 1 as the new default (improving margins broadly).
2. Create a commercial product structure where routing tier is tied to subscription, allowing customers to self-select (and pay for) higher-quality routing.

**Important note:** Lukasz flagged that migrating everyone to Alternative 1 by default is effectively a **silent deliverability downgrade**. Daniel acknowledged this, noting that many customers already have degraded deliverability due to bounces, and that any rollout must be tested before mass migration.

> "We will need to test before rolling out so that we don't end up having all customers at 15% lower deliverability and everybody will be really upset." — Daniel

---

## SMS Alternative Routing — Technical Design

[Lukasz Grabowski's assessment]

**Scope of changes:**

- **Back Office (back end + front end):** New account-level configuration — a dropdown with 3 routing options (Direct, Alt 1, Alt 2). Sits close to existing SMS subscription settings. This is where routing is set by staff.
- **User Panel (front end, minor):** Display the current routing tier in account settings so customers can see it. Minor front-end work.
- **SMS Domain (back end):** New Infobip sub-account endpoints added to configuration. The back office routing setting is fed into the SMS domain, where it is joined with the existing **currency-to-account mapping** to select the correct Infobip endpoint for sendings.
- **Scope exclusion:** MFA sendings are explicitly **out of scope** for this change.
- **Data migration:** Some migration in the AU domain / Back Office to set default routing values.
- **Audit logging:** Existing audit logging facilities can be reused — minor effort.

**Infobip account structure caveat:**

> "The account structure in Infobip is kind of weird — they send us a plethora of logins to different accounts. They call them sub-accounts but they are just different entities with no relation to one another." — Lukasz

With the alternative routes added, Apsis will need to maintain approximately **~20 Infobip accounts** in their control panel. This is a maintenance overhead but is unavoidable given Infobip's structure.

**Why not switch away from Infobip entirely?** — Lukasz explicitly recommended against it: all existing integration and configuration is geared toward Infobip's APIs. There is significant cross-domain dependency on their APIs. Switching would cost far more in development than the savings would justify.

**Pro platform:** SMS volumes on Pro are currently low. Daniel suggested the same routing change could be a good project for the Pro team (led by Mashek) to run in parallel.

---

## SMS Alternative Routing — Next Steps

- Lukasz to arrange a meeting with the email/SMS team to scope and estimate the work.
- Target: estimates delivered **before end of February** (approximately two weeks, coinciding with a Warsaw trip).
- Daniel to confirm with Mashek whether the Pro team should run a parallel version of this work.

---

## Tags Limit — Current State and Proposed Changes

**Background on the current implementation (important architectural context):**

> Tags in Audience were implemented as a **special type of attribute** — same storage, same data model. A tag is essentially a custom attribute with no value, with a specific display name. This was a deliberate trade-off to accelerate implementation at the time tags were introduced (years ago). It now has consequences. — Lukasz

**Implications of the current tag implementation:**
- More tags = more attributes on a profile = larger profile storage footprint.
- Longer and more complex Athena queries for exports (both in execution time and query string length — there are string length limits in Athena that can be hit).
- Some APIs (e.g., the "360" API referenced) only fetch the first X kilobytes of profile data — the more tags/attributes on a profile, the less event data those APIs will return.
- The **200 tag limit** is a **soft limit** enforced in: (a) the UI, (b) One API, and possibly (c) some Audience APIs. It is **not** enforced at the storage or data layer.

**Current ask from customers:** PS consultants are fielding requests from customers wanting more than 200 tags. Example: one customer had ~250 tags synced from an external system. Some customers are asking to simply "buy more tags."

**Proposed solution — Feature toggle for a second tier:**

Lukasz's recommendation: implement a **feature toggle** that enables a second tier of tags (e.g., 400 total). This is:
- Low risk
- Reuses existing feature toggle infrastructure (already read consistently across the platform)
- Easily extended later to a full attribute/tag limit and pricing tier system

No custom per-customer numeric limits needed — a binary tier toggle is sufficient for the near term.

**Pricing consideration:** Daniel proposed charging for additional tags proportional to profile count, since more tags = more storage per profile. Example framing: "200 additional tags = X EUR per profile per month."

**Lukasz's additional recommendation — Custom attributes as an alternative to tags:**

Many customers hit the tag limit because they're misusing tags for what should be a custom attribute with discrete values (e.g., a "Country" tag duplicated across 50 country-name tags, rather than a single `country` attribute with a value). PS team should recommend this pattern to customers approaching the limit.

**Henrik's related suggestion — "List" attribute type:**

Henrik asked whether it would be possible to add a value to an attribute without overwriting (i.e., a multi-value / list attribute type). Lukasz confirmed this maps to a **complex data type** — a project that was **started but postponed** due to high audience-side cost. This is not a near-term replacement for the simple tag limit increase.

**Tags usage visibility:**

Henrik asked whether it's possible to see if a tag is actually in use (assigned to any profiles, or used in MA flows / email requests) to help customers clean up. Lukasz assessed this as non-trivial:
- Checking by "does any profile have this tag?" requires scanning profile data in Athena — not insignificant effort.
- Checking by "is this tag referenced in an MA flow or email filter?" requires querying different databases.
- Full "is this tag used?" would need to combine both, increasing complexity.

---

## Key Takeaways

1. **Approach B (Apsis calls Membrane) is the consensus-preferred architecture** for Membrane integration. It reuses the generic connector, keeps authentication within the Integrations domain, and avoids the need to build new One API endpoints for Membrane to call. The Membrane iframe/popup is only needed for the customer's CRM OAuth connection step.

2. **Critical assumptions about Membrane's capabilities must be validated before estimates are reliable.** The two most important: (a) can Membrane actions return data in a format Apsis specifies, and (b) can Membrane register webhooks with a caller-provided secret? Erik documented these.

3. **Authentication for Membrane webhooks follows the existing pattern:** per-customer callback URLs (encoding account/section/integration ID) + a secret generated by Apsis and registered with Membrane at installation time. No new One API auth infrastructure needed for the initial scope.

4. **Keep Apsis ↔ Membrane mapping at account level, not section level.** Sections are planned for future removal.

5. **SMS alternative routing is assessed as architecturally straightforward** — an account-level config setting fed into the existing Infobip account/currency mapping in the SMS domain. ~20 Infobip sub-accounts to maintain. MFA sendings excluded from scope. Do not switch away from Infobip — too much platform dependency.

6. **The 200-tag soft limit can be lifted via feature toggle** to a second tier (e.g., 400). This is the safest near-term approach. Tags are stored as attributes — the architectural trade-off from the original implementation has storage, query complexity, and API payload implications at scale.

7. **Erik Andersson was on his last day** — his written documentation and the two explicitly-flagged open questions are the primary knowledge artifacts to preserve from his involvement in the Membrane architecture.

---

## Unresolved Questions and Action Items

| # | Question / Action | Owner | Notes |
|---|---|---|---|
| 1 | Document the two critical Membrane prerequisite questions explicitly | Erik Andersson | Before end of last day |
| 2 | Schedule follow-up meeting with Membrane | Daniel | After Lukasz does initial doc review |
| 3 | Lukasz to review Eric's documentation and Membrane's own docs; prepare questions for Membrane meeting | Lukasz (Greg) | Priority-dependent given current workload (Audience KT, duplicate profiles) |
| 4 | Confirm: does Membrane support per-customer webhook registration with caller-provided secrets? | To be validated with Membrane | Blocker assumption for Approach B inbound auth |
| 5 | Confirm: can Membrane actions be configured to return data in Apsis-specified JSON schema regardless of CRM? | To be validated with Membrane | Blocker assumption for generic connector reuse |
| 6 | Estimate effort for SMS alternative routing | Lukasz + email/SMS team | Target: before end of February |
| 7 | Confirm with Mashek whether Pro team should run parallel SMS routing project | Daniel | Low urgency — Pro SMS volume is currently low |
| 8 | Specify and estimate tags second-tier feature toggle (0→400 tags) | Henrik + team (Daniel to specify) | Not urgent — estimates needed for Q2 planning |
| 9 | Investigate how many customers currently exceed specific custom attribute count thresholds | TBD | To inform potential custom attribute pricing tiers |
| 10 | **⚠️ AMBIGUOUS:** The exact cost/pricing figures for Membrane per-customer were mentioned as "around $10" by Daniel — treat as directional only, not confirmed. | — | Needs commercial confirmation |