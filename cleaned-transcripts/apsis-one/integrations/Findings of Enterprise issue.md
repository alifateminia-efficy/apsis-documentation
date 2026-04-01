---
source_file: "Findings of Enterprise issue.txt"
domain: Apsis One Integrations
topics: [FCC Enterprise integration bug, HTTP response code misuse, content-length header missing, broker service debug logging, generic connector architecture, JSON truncation root cause]
speakers: ["Erik Andersson (investigator/developer)", "Lukasz Grabowski (session facilitator)"]
key_components: [FCC Enterprise 12.1, Broker Service, Generic Connector, Outbound Worker, Squid Proxy, CloudWatch, Resty HTTP framework, Postman]
session_type: knowledge-transfer
---

## Session Overview

Erik Andersson presents a post-mortem investigation into a data sync issue affecting a customer using Apsis One to sync email campaign events to FCC Enterprise 12.1. The customer observed that only ~100 of ~280 expected contacts appeared under the campaign in Enterprise, despite Apsis logs showing successful delivery. The root cause was a combination of FCC Enterprise's non-standard HTTP status code behavior and a missing `Content-Length` header causing a third-party library inside Enterprise to silently truncate request bodies. The fix — enabling `Content-Length` header generation globally in the broker service — has been deployed to production.

---

## Incident Summary and Initial Symptoms

The customer was syncing email campaigns from Apsis to **FCC Enterprise 12.1**. They were a relatively small customer with ~300 profiles, resulting in ~600 events sent. Out of approximately 280 contacts expected to appear under the campaign in Enterprise, only ~100 were visible. However, the Apsis-side reports showed all expected contacts present.

Initial investigation questions:
- Did Audience fail to emit all events?
- Did Apsis fail to send them to Enterprise?

Erik confirmed that no events were missed from Audience, and that the batch send appeared successful because Apsis received a **200 HTTP response** from the CRM system.

---

## FCC Enterprise's Non-Standard HTTP Status Code Behavior

This is the most critical finding from the investigation.

**FCC Enterprise does not use HTTP status codes for business logic.** It uses HTTP codes strictly for transport-layer acknowledgment. As a result:

- Enterprise responded with **HTTP 200** even when it encountered an error
- The error detail ("`invalid JSON`") was embedded in the **response body**, not surfaced via a 4xx/5xx status code
- Apsis logs therefore recorded the request as a success when it had in fact silently failed

> "Enterprise is a very, very special CRM system in lots of ways... they do not utilize HTTP codes for their business logic. They strictly use this for their transport layer. So they had responded with an error in the response body saying 'this is invalid JSON, you're doing something wrong'... but they responded with status 200 because we managed to transport the package to them."

This was compounded by the fact that Enterprise's normal request processing path (built for the generic connector) **has been adapted** to Apsis's HTTP error conventions. However, the error in this case originated from a **separate, lower-level component** — a standard part of Enterprise not specific to the generic connector — and that component uses the raw, non-standard behavior.

⚠️ **Warning:** As things currently stand, Apsis logs will report **success** for requests that have actually failed inside Enterprise if the failure occurs in this lower-level component. There is no HTTP-level signal to catch it.

---

## Root Cause: Missing `Content-Length` Header Causing JSON Truncation

### How the truncation was discovered

The error log discovery came from Rachel finding a **separate error log** in Enterprise (distinct from the main logs the Enterprise team had been checking). That log showed errors appearing as soon as the campaign was registered, with the message: **"this is not valid JSON"**.

### Why the JSON was invalid

When Apsis sends requests, the **`Content-Length` header is not added by default**. When Rachel set up FCC Enterprise in a local development environment, he was able to inspect the full request body and confirmed the JSON sent by Apsis was **perfectly valid**.

However, a **library inside FCC Enterprise** behaves incorrectly when the `Content-Length` header is absent: it truncates the request body partway through, then attempts to parse the truncated fragment as JSON — which fails.

- With `Content-Length` present (as Postman adds it by default): ✅ Works correctly
- Without `Content-Length` (as Apsis sent by default): ❌ Body is truncated, JSON parse fails, silent 200 returned

> "When you removed the content header, then a library inside of FCC Enterprise... started truncating the data. So when we sent a big JSON blob without the content-length header, that library essentially cut the body in half and then tried to parse it as JSON, which obviously will fail."

### Why Postman obscured the issue

Postman **adds `Content-Length` by default**, so testing the same payload in Postman succeeded. This initially increased confusion because it appeared to rule out a payload problem. The difference in default header behavior between Postman and the Apsis HTTP client was the key diagnostic gap.

> "This was a very, very weird thing to debug, especially because it looked correct from Postman but it was incorrect from Apsis. That's one of those cases where Postman added to the confusion more than it helped."

---

## The Fix: Enabling `Content-Length` in the Broker Service

### Why it was fixed on the Apsis side

While the correct long-term fix would be for Enterprise to update or reconfigure the library that truncates bodies — the affected code is in a "super duper mega core" part of the Enterprise system, making that change slow and complex to roll out to all customers. The Apsis-side fix took approximately **two minutes** to implement.

### What was changed

In the **broker service**, there is a single central function responsible for constructing all outgoing requests. The **Resty HTTP framework** (which Apsis uses) supports automatic `Content-Length` calculation and header attachment natively — it just needs to be enabled.

Erik enabled this in the one central request-building location in the broker service, meaning:

- It now applies to **all outgoing CRM requests** — campaign events, consents, and any other request types
- It does not need to be enabled in multiple places
- It is already **deployed to production**

> "I do this here, this is now done for all of the requests going to the CRM system, whether it's events or consents or not. So we don't need to do this in multiple places."

`Content-Length` is a standard HTTP header and there is no risk introduced by always including it.

---

## Broker Service Architecture and Debug Logging

### Broker service role (clarification for new developers)

The **broker service** is the outermost proxy layer in the outbound request path:

```
Outbound Worker → Broker Service → CRM System (e.g., FCC Enterprise)
```

- The **Outbound Worker** constructs the request payload and method
- The **Broker Service** retrieves and decrypts the customer's API credentials from the database, attaches them to the request as headers, and forwards the request to the destination
- Apsis developers do **not** have access to customer credentials directly in the generic connector — the broker service handles this separation

This is distinct from the **Squid Proxy**, which handles a different concern: checking whether the destination domain is an allowed target for the customer.

### Debug logging in the broker service

Normally, the broker service logs **minimal information** due to CloudWatch cost concerns. Full debug logging — which includes URL, request body, response code, and response body — would be "astronomical" in cost if always enabled.

When debugging this issue, Erik:
1. Changed the debug log level to `info` so it would always print
2. Wrapped it in an `if` clause to scope it to **only the specific customer** being investigated

This scoped debug logging approach is the recommended pattern for production debugging in the broker service.

---

## Generic Connector Campaign Events — Expected Response Format

Under the **generic connector specification**, campaign events are sent as batches via POST. The expected successful response is an **HTTP 200**, which under normal (standards-compliant) CRM behavior indicates the batch was accepted and the matching contacts processed. FCC Enterprise's non-standard use of 200 for error responses is an exception to this contract.

---

## Key Takeaways

1. **FCC Enterprise (legacy and 12.1) does not use HTTP status codes for business logic.** Errors may be returned as HTTP 200 with an error payload in the response body. Apsis logs will show success in these cases. Always check Enterprise's **separate error log** when investigating suspected delivery failures.

2. **The `Content-Length` header fix is now live in production.** The broker service now includes `Content-Length` on all outgoing CRM requests via Resty. This prevents the truncation bug in Enterprise's internal library.

3. **Postman's default headers differ from Apsis's HTTP client defaults.** Specifically, Postman adds `Content-Length` automatically; Apsis did not. When using Postman to reproduce integration bugs, verify that headers match exactly what Apsis sends. Differences can mask or introduce bugs.

4. **The broker service has scoped debug logging capability.** Full response body logging can be temporarily enabled for a specific customer without flooding CloudWatch. Use the `if`-clause pattern Erik implemented.

5. **The correct fix belongs in Enterprise**, but the timeline for patching a core Enterprise library is long. The Apsis-side fix is pragmatic and safe.

---

## Unresolved Questions / Notes

- ⚠️ The underlying Enterprise library that truncates bodies on missing `Content-Length` remains unfixed on the Enterprise side. Other customers integrating directly with Enterprise APIs (not via Apsis) may encounter the same silent failure.
- Erik mentioned needing to clean up some old integrations and CloudWatch log groups before handover — this is a pending housekeeping action item.
- No questions were raised by other attendees (Tomic, Michal) before the session ended.