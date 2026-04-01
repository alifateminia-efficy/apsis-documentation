---
source_file: Erik - 20.11.2025.txt
domain: Apsis One Integrations
topics: [forced uninstallation, clear integration endpoint, Postman collections, Lime integration, Dynamics 365, FSC Enterprise, webhook cleanup, ignore flags, delete integration secret key, Altasync manager, integration debugging]
speakers: ["Erik Andersson (Senior/Lead Developer)", "Michal Rosikiewicz (Developer)", "Tomasz Kowalski (Developer)"]
key_components: [Integration Manager, Altasync Manager, Secrets Manager, Postman Collections, FSC Enterprise Connector, Lime Connector, Generic Connector, Dynamics 365 Connector, RDS Database, Squid Proxy]
session_type: knowledge-transfer
---

# Apsis One Integrations — Forced Uninstallation Flow & Postman Collections KT

## Session Overview

This session covers the Postman collection landscape for Apsis One Integrations, with a deep focus on the **Clear Integration endpoint** used for forced/manual uninstallation of customer integrations. The team walks through a live forced-deletion for a real customer (Meltx Plastics) whose Lime integration could not be cleanly uninstalled due to CRM-side errors. The session also covers the rationale behind the `ignore external system errors` and `ignore audience` flags, a known squid proxy edge case during deletion, and the GDPR/data-privacy implications of leaving webhook configurations active after a forced delete.

---

## Postman Collection Overview

Erik maintains several Postman collections covering different connectors. Each collection is scoped to the functionality actually used in production integrations.

### FSC Enterprise Collection
- **FSC Enterprise** is a **legacy connector** — it is not the generic connector.
- It uses internal APIs of the FSC system.
- The collection contains the relevant endpoints utilized in the integration (e.g., get all contacts).

### Generic Connector Collection
- Contains the full set of endpoints available in the generic connector.
- You can configure which domain you are calling.
- Useful for isolating bugs: "Is the issue because I am getting bad data, or because we are receiving correct data but handling it incorrectly?"

### Lime Collection
- **Lime** is also a legacy connector — not the generic connector.
- The collection does not have a full suite, only representative examples.

### Dynamics 365 Collection
- Contains all endpoints that Apsis utilizes when communicating with Dynamics 365.
- [Erik Andersson]: Very useful for debugging Dynamics customer issues — checking data shape, language (Swedish vs. English), permissions, empty results, etc.

---

## The Clear Integration Endpoint

### Purpose and Context
- The **Clear Integration endpoint** triggers the uninstallation flow with **`ignore_external_system_errors`** set to true.
- Use case: A customer wants to uninstall, but something has changed — API credentials, configuration, or permissions — causing the uninstallation request to their CRM to fail. They still need Apsis to delete everything on its side.
- What it deletes on the Apsis side:
  - Database entries
  - Credentials
  - All internal installation records
- What it does **not** guarantee: removal of webhook configuration in the customer's CRM.

### How It Differs from Normal Uninstallation
- The normal uninstallation endpoint will abort if the CRM returns an error (by design, for data-privacy reasons — see below).
- The Clear Integration endpoint appends the `ignore_external_system_errors` query parameter, causing the flow to proceed regardless of CRM-side errors.
- [Erik Andersson]: "There is absolutely nothing preventing you from adding a force-delete button which calls the same endpoint but appends the query parameter set to `ignore_external_system_errors`."

### Authentication: Delete Integration Secret Key
- The Clear Integration endpoint uses an **internal API key**, not the customer's credentials.
- The customer's credentials are still loaded internally during the uninstallation flow to clean up their CRM — but the *trigger* call is authenticated with this internal key.
- The key is stored in **Secrets Manager** under the name:

```
Delete Integration Secret Key
```

- It is added as an **Authorization header** on the request.
- **⚠️ WARNING:** This key works for every integration (Dynamics, Lime, EDL, etc.) — it can trigger deletion of any integration. Do not leak this key. [Erik Andersson]: "In theory, if a customer had this, you can delete every integration, which is right now a bit suboptimal."
- [Erik Andersson, on future improvement]: "Maybe this should be a delegation key, but this is something we/you can change in the future to optimize this."

### Required Parameters — "The Holy Trinity"
To pinpoint one unique installation, every request requires three parameters:

1. **Account ID** — e.g., `meltxplastics` (note: this customer predates UID account IDs, so the account name is used literally)
2. **Section ID** — the specific section within the account
3. **Integration ID** — must match an existing integration ID (e.g., `lime`); if you pass `fsc-enterprise-12.0` for a Lime installation, the request will fail with a validation error — the system validates that the integration ID exists

---

## Known Flags: `ignore_external_system_errors` and `ignore_audience`

The uninstallation flow has (at least) two ignore flags:

| Flag | When to use |
|---|---|
| `ignore_external_system_errors` | CRM has misconfiguration or credentials have changed; CRM-side deletion will fail but Apsis-side cleanup should proceed |
| `ignore_audience` | No users exist in the account so Apsis cannot delete event listeners from Audience, **or** the account has been terminated so delegation key exchange is impossible |

[Erik Andersson]: "That's why we have the ignore audience — if the account has been terminated so we can't exchange delegation keys. And the ignore external system errors — if the CRM has some misconfiguration."

---

## Data Privacy / Legal Implications of Forced Deletion

> "The reason we are not exposing this today is because it does have a drawback — because we are failing to remove things in their CRM, they will still be trying to send webhook updates to us. And this is very bad from a legal perspective."

- After a forced delete, Apsis will **reject** incoming webhook calls (no installation exists), but will still **receive** the HTTP requests.
- The raw incoming data from those requests is not logged or displayed in Altasync Manager (deliberately).
- [Erik Andersson]: "In theory you could have access to the data if you wanted to." He confirmed this is a deliberate non-logging decision.
- **Standard procedure after a forced delete:** Notify the customer (via SOC/support) to manually remove any lingering webhook configuration from their CRM.

---

## Live Debugging: Meltx Plastics Lime Integration Forced Deletion

This section documents the actual forced deletion performed during the session and the issues encountered.

### Initial 401 Error
- Michal made the first request and received a 401 ("provided credentials are not valid").
- The 401 was determined to have originated from **Lime** (not from the Apsis endpoint itself) and was unexpectedly propagated up to the caller.
- No logs were found in Integration Manager for this request.
- [Erik Andersson]: "The reason you don't get any logs is because we don't log this — you get in here and then we just return the error. We don't log it."

### Root Cause: Wrong Account ID
- The initial request used an incorrect account ID.
- The correct account ID is **`meltxplastics`** (no space; note this account predates UID-based account IDs so it uses a literal name string).
- [Erik Andersson]: "This customer predated the migration to UID account IDs. So the account name literally is Meltx Plastics."
- Section ID confirmed as `16134`.
- Once the correct account ID was used, the request proceeded and logs appeared in Integration Manager.

### The Squid Proxy Edge Case
- During deletion, the system encountered an error — HTML returned from a gateway. This is a known edge case.
- [Erik Andersson]: "You remember the squid proxy that we have for some reason? Occasionally, very rarely, when you try to uninstall, the database refuses to delete that squid proxy entry — it just hangs on that request for whatever reason."
- Because `ignore_external_system_errors` was set, the flow proceeded past this error and completed cleanup on the Apsis side.
- ⚠️ **This edge case is known and intermittent.** It has caused failed uninstallation attempts in the past.

### Outcome
- All Apsis-side records for the Meltx Plastics Lime integration were successfully deleted.
- Webhook configuration in the customer's Lime CRM was **not** deleted (due to the CRM-side error).
- Follow-up action: SOC to inform the customer to remove lingering webhook configuration from their Lime instance.
- [Erik Andersson]: "Now we just need to ask SOC to ask the customer to do the webhook deletion."

### Observation: Active Incoming Webhooks
- Before deletion, Altasync Manager showed the customer had been continuously sending data since approximately the 13th of the month, with updates getting progressively closer to the current date.
- This confirmed the customer's Lime instance was still actively pushing data to Apsis.

---

## Integration Storage: RDS, Not DynamoDB

- Integration data is stored in **RDS**, not DynamoDB.
- [Erik Andersson]: "It's in RDS. We don't use DynamoDB in integration."

---

## Feature Gap: Force-Delete Button in UI

[Discussed as a potential improvement — not yet implemented]

- Currently there is no UI button for force-deletion; it requires manual Postman usage with the internal key.
- [Michal Rosikiewicz]: "I don't see any value in doing it ourselves manually. If something is failing and they know that they definitely want to remove it, why not leave that possibility for them to force the deletion?"
- [Erik Andersson]: "There is absolutely nothing that stops us from doing it. This button would call the same endpoint but you just append the query parameter `ignore_external_system_errors`."
- Proposed implementation: add a "Force Delete" button in the app UI with a disclaimer instructing the user to manually delete webhook configurations in their CRM.

---

## Updating API Keys Without Reinstalling

[Erik Andersson]: If a customer has simply rotated their API key (not deleted the integration), they do **not** need to uninstall and reinstall.

- For FSM integrations (at least `fsm-press-12.0`) there is an **Update API Key** endpoint/action in the integration.
- In theory, if a key has been removed: update the key first, then proceed with normal uninstallation.

---

## Pending / Unrelated Issue: Dynamics Customer Domains Request

- A Dynamics customer has asked for the list of domains that Apsis can be reached on, in order to configure a CORS policy.
- An endpoint was visible in the collection:

```
prod-dynamics-contact.apsis.ap[...] (publish)
```

⚠️ **[AMBIGUOUS]** The full URL was not fully captured in the transcript. Erik was uncertain whether this is an endpoint inside the CRM plugin or an Apsis-side endpoint.

- Action: Erik to get more information and loop in Michal/Tomasz if investigation is needed.

---

## Key Takeaways

1. **The Clear Integration Postman collection** is the primary tool for performing forced/manual integration deletions. It uses the `Delete Integration Secret Key` from Secrets Manager as the Authorization header.

2. **The holy trinity** for any installation operation is: Account ID + Section ID + Integration ID. Getting any one of these wrong will silently fail or return a misleading error.

3. **Forced deletion leaves webhooks active in the customer's CRM.** Always follow up by instructing the customer (via SOC) to remove webhook configuration manually. This is both a functional and a data-privacy concern.

4. **`ignore_external_system_errors`** bypasses CRM-side failures. **`ignore_audience`** bypasses Audience-side failures (e.g., terminated accounts, no users). These flags exist specifically for scenarios where a clean uninstall is impossible.

5. **A known squid proxy edge case** can cause the database to hang on deleting the proxy entry during uninstallation. It is intermittent and the `ignore_external_system_errors` flag will bypass it.

6. **Integration data is in RDS**, not DynamoDB.

7. **The Delete Integration Secret Key must be treated as highly sensitive** — it can trigger deletion of any integration across all accounts.

8. **A self-service force-delete UI button** has been identified as a desirable and technically straightforward improvement.

---

## Unresolved Questions & Action Items

- [ ] **Meltx Plastics follow-up:** SOC to contact the customer asking them to remove lingering Lime webhook configuration.
- [ ] **Dynamics domains request:** Erik to gather more information on what the customer is actually asking for (CORS policy domains). Involve Michal/Tomasz if investigation is required. Erik noted a URL visible in collection: `prod-dynamics-contact.apsis.ap[...]` — full URL not confirmed.
- [ ] **Force-delete UI button:** No owner assigned, but technically feasible. Needs design for the disclaimer/warning.
- [ ] **Delete Integration Secret Key scope:** Consider converting to a delegation key for better security scoping (noted by Erik as a future optimization).
- [ ] **New incoming issue (CRM implementation partner):** A person named Denis (CRM implementation partner) has reported issues — unclear if Integrations or Audience. Erik to triage and involve team if needed.