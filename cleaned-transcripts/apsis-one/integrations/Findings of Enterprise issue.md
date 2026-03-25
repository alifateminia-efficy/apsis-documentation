---
source_file: Findings of Enterprise issue.txt
domain: Apsis One Integrations
topics: [HTTP Status Code Handling, Content-Length Header Issue, Generic Connector Debugging, FCC Enterprise Integration, Request/Response Validation]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Generic Connector, Broker Service, FCC Enterprise 12.1, Squid Proxy, Resty HTTP Framework, CloudWatch Logs]
session_type: debugging-session
subdomains: [Architecture, Generic Connector, Outbound Flow, Efficy Enterprise 12.1]
---

## Session Overview

Erik Andersson presents findings from a week-long investigation into a critical issue where email campaigns synced from Apsis to FCC Enterprise 12.1 appeared incomplete in the destination system despite successful HTTP responses. The root cause was traced to FCC Enterprise's non-standard HTTP handling: the system returns HTTP 200 status codes even when rejecting requests due to invalid JSON in the response body. The underlying problem was that Apsis was not including the `Content-Length` header in outbound requests, causing a library inside FCC Enterprise to truncate the request body during parsing, resulting in malformed JSON that was then rejected by the system.

---

## Initial Problem Statement

**Customer Scenario:**
- Customer was syncing email campaigns from Apsis to FCC Enterprise 12.1
- Small customer by Apsis standards: 300 profiles, approximately 600 events sent
- **Expected behavior:** ~280 contacts should appear in the campaign within Enterprise
- **Actual behavior:** Only ~100 contacts visible in Enterprise
- **Critical observation:** The Apsis report showed all expected contacts present, indicating the problem was not in Apsis data collection

[Erik Andersson]: This discrepancy immediately indicated the issue was somewhere in the integration pipeline between systems, not in Apsis itself.

---

## Investigation Process and Methodology

### Phase 1: Ruling Out Apsis Data Loss

[Erik Andersson] began by verifying that all customer profile data was actually being sent from Apsis:

- Retrieved example profiles from the customer that should have appeared in the campaign
- Verified in logs that all expected data was sent to FCC Enterprise as a batch
- **Result:** No events were missed from the Apsis side; the outbound transmission was complete

### Phase 2: HTTP Response Analysis

Initial log analysis showed successful delivery:
- Checked responses from the CRM system when the batch was sent
- Received HTTP 200 status code from FCC Enterprise
- Concluded: "The problem is not in Apsis because I can see that you are responding with 200 so the problem is on your side"

**The customer countered with their own logs:**
- FCC Enterprise error logs showed no sign of the batch being received
- This created apparent contradiction: Apsis logs showed successful transmission (HTTP 200), but Enterprise logs showed no incoming batch

### Phase 3: Discovery of Dual Error Reporting in FCC Enterprise

After approximately one day of investigation, Rachel discovered that FCC Enterprise was logging errors in a separate error log that wasn't visible in the main logs.

[Erik Andersson]: The error message revealed the core issue: **"This is not valid JSON, so we are rejecting it."**

This is where the investigation breakthrough occurred.

---

## Root Cause Analysis: FCC Enterprise's Non-Standard HTTP Behavior

### The Core Problem: HTTP Code vs. Business Logic Separation

FCC Enterprise handles HTTP status codes in an unconventional way:

> "Enterprise is a very, very special CRM system in lots of ways... they do not utilize HTTP codes for their business logic. They strictly use this for their transport layers."

**Specific behavior observed:**
- FCC Enterprise responds with HTTP 200 when the **transport** is successful (the request reached the system)
- However, the **actual business logic result** (validation, processing) is communicated **only in the response body**, not via HTTP status codes
- In this case: HTTP 200 status code + error message in response body stating "This is not valid JSON"

[Erik Andersson]: This is a dangerous pattern because it violates HTTP semantics where 200 typically indicates complete success, not just transport success.

### Historical Context: Date Field Quirks

As a side note on FCC Enterprise's unusual behaviors, [Erik Andersson] mentioned:
- If no birth date or date is selected in date fields, FCC Enterprise defaults to the year 1899
- This indicates the system carries legacy assumptions and non-standard handling throughout

---

## The Missing Content-Length Header: Actual Root Cause

### Discovery Through Multi-Environment Testing

[Erik Andersson] followed a systematic debugging approach:

1. **Enabled full debug logging in the broker service:**
   - Logged URL, request body, response code, and response body
   - Normally this is not done due to astronomical CloudWatch costs
   - Only enabled for this specific customer
   - Confirmed the CRM system was returning "invalid JSON" error

2. **Verified data wasn't being truncated by Apsis:**
   - Set up local testing environment
   - Tested with even larger batches
   - Confirmed full request was being sent correctly from Apsis

3. **Installed FCC Enterprise locally for debugging:**
   - Rachel installed FCC Enterprise on his local development machine
   - Could reproduce the "invalid JSON" error from Apsis requests
   - Same request worked fine in Postman with identical payload

### The Critical Difference: Content-Length Header

[Erik Andersson]: When testing in Postman, the request succeeded. When investigating why, Rachel noticed:

- **Postman behavior:** Sets `Content-Length` header by default
- **Apsis behavior:** Does NOT set `Content-Length` header by default
- **Rachel's observation:** He could see the full request body in Postman, confirming the JSON was valid

When the `Content-Length` header was **removed from Postman requests**, the same "invalid JSON" error appeared.

### How FCC Enterprise Processes Requests Without Content-Length

A library inside FCC Enterprise relies on the `Content-Length` header to determine the request body size:

- **Without Content-Length:** The library appears to have a default truncation mechanism
- **Observed behavior:** Library cuts the request body **in half**
- **Consequence:** Attempts to parse truncated JSON as complete JSON
- **Result:** JSON parser fails with validation error

[Erik Andersson]: As soon as `Content-Length` header was included in requests, everything worked correctly.

---

## The Solution Implemented

### Why Fix It in Apsis Rather Than Ask FCC Enterprise to Change

[Erik Andersson] made a deliberate trade-off decision:

> "Under normal circumstances, like this is something which would like things like this should be changed inside of enterprise... the time it would take to like replace that library or change the configuration of it and have that out to all customers which would be a very long time."

**The calculation:**
- Asking FCC Enterprise to fix their library: Weeks or months for deployment to all customers
- Fixing it in Apsis: 2 minutes to enable an existing feature

[Erik Andersson]: "Whereas in this case it took me two minutes to enable this because the rest the HTTP framework which we are utilizing it supports this out-of-the-box if you enable it."

### Implementation Details

**Location:** Broker service — centralized request creation function

The broker service is the proxy layer where:
- Requests come from the outbound worker
- Apsis decrypts customer API credentials from the database (developers don't have direct access)
- Adds authentication headers
- Forwards requests to the destination CRM

**The fix applied:**
```
In the broker service's central request creation function:
- HTTP framework used: Resty
- Change: Enabled Resty to calculate and include Content-Length header
- Scope: All outgoing requests to CRM systems (events, consents, notifications, etc.)
- No need to implement in multiple places
```

[Erik Andersson]: By making this single change in the centralized request function, the fix applies to **all** generic connector requests, whether going to FCC Enterprise or other CRM systems.

**Deployment status:** This change is already live in production.

---

## Broader Implications and Lessons Learned

### FCC Enterprise Compatibility Issues

[Erik Andersson]: The main take-away: "Be aware that FCC Enterprise in particular the legacy one, but also the 12.1 is not directly compatible with the Apsis way of handling HTTP codes."

**Key distinction:**
- Error messages should come from the generic connector component (which has been adapted to Apsis HTTP error handling in version 12.1)
- But in this case, the error came from a **standard core part of Enterprise**, bypassing the adapted connector layer
- Result: The problematic 200 status code with error in body

### Postman as a Double-Edged Debugging Tool

[Erik Andersson]: "Be aware of headers in Postman because they can easily differ from the ones that we are utilizing inside of Apsis."

**Issue:** Postman's default behavior (automatically setting `Content-Length`) masked the problem initially. The request succeeded in Postman but failed in Apsis, making it harder to identify that a missing header was the culprit. This added to confusion rather than helping debugging efforts.

### The Danger of Silent Failures in Logs

[Erik Andersson]: "The main takes from this is... our logs will say success, but in reality it has failed horribly behind the scenes."

This issue created a situation where:
- Apsis logs showed HTTP 200 and successful transmission
- FCC Enterprise silently rejected the request (only visible in their error logs)
- No automated alert would catch this discrepancy
- Manual investigation required days to discover

---

## Debugging Infrastructure Used

### Broker Service Debug Logging

The broker service is the key debugging point because it:
- Sits between the outbound worker and the destination CRM
- Has access to both customer API credentials and request/response details
- Is responsible for adding authentication

**Debug logging approach:**
```
Normal production configuration:
- Limited logging to control CloudWatch costs

Debug configuration (when needed):
- Log URL
- Log HTTP method
- Log request body
- Log response status code
- Log response body
- Can be enabled per-customer to avoid excessive costs
```

**Implementation for this issue:**
- Changed logging level to INFO (instead of only on errors)
- Wrapped with conditional (if clause) to target specific customer only
- This targeted approach allowed full debugging without astronomical CloudWatch costs

### Comparison of Proxy Layers

[Erik Andersson] clarified the two proxy layers in the integration architecture:

1. **Squid Proxy:**
   - First layer of filtering
   - Determines if customer is allowed to make requests to the destination domain
   - Basic access control

2. **Broker Service:**
   - Second layer
   - Decrypts API credentials from database
   - Adds authentication headers
   - Forwards requests to actual destination
   - Primary debugging point for request/response issues

---

## Key Takeaways

1. **HTTP Status Codes Are Not Universal:** FCC Enterprise uses HTTP codes only for transport layer, not business logic. Always check response bodies for actual error details, especially with enterprise systems.

2. **Content-Length Header Matters:** Even headers that seem optional can be critical for downstream systems. FCC Enterprise requires `Content-Length` to properly parse request bodies.

3. **Centralized Fixes Scale Better:** By fixing the issue in one place (broker service request function) rather than multiple connectors, the fix applies to all CRM systems automatically.

4. **Tool Behavior Can Hide Problems:** Postman's default headers differ from Apsis framework defaults. Testing tools may inadvertently mask issues by providing features that aren't enabled by default in production code.

5. **Logging Strategy Matters:** Full debug logging is necessary but expensive. Use conditional, per-customer logging to debug issues without breaking production costs.

6. **Multi-Layer Verification is Essential:** This issue required verification at multiple layers (Apsis logs, FCC Enterprise logs, local development setup, Postman testing) to isolate the real problem.

7. **Legacy System Quirks Require Defensive Coding:** When integrating with systems like FCC Enterprise that have non-standard behavior, implement defensive measures (like adding headers) even when not theoretically necessary.

---

## Action Items and Cleanup

[Erik Andersson] noted additional work needed:
- Some old integrations need to be removed
- CloudWatch log groups require cleanup
- Minor follow-up work scheduled for the following day
- May not be available for meetings during cleanup period

---

## Unresolved Considerations

- The fix (adding `Content-Length` header) was implemented as a pragmatic workaround. Ideally, FCC Enterprise should update their internal library to not depend on this header for request parsing, but this would require their significant development effort and multi-week deployment cycle.
- Future integration work with other legacy enterprise systems may reveal similar non-standard HTTP handling patterns.