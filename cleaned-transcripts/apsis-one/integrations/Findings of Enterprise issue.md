---
source_file: Findings of Enterprise issue.txt
domain: Apsis One - Integrations
topics: [Enterprise CRM Integration, HTTP Status Code Handling, Request Header Management, Generic Connector, Debugging Methodology, Content-Length Header, JSON Payload Truncation]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Broker Service, Outbound Worker, FCC Enterprise CRM, Generic Connector, Squid Proxy, CloudWatch Logging, REST Framework]
session_type: debugging-session
---

## Session Overview

Erik Andersson presents a detailed post-mortem of a critical enterprise integration issue discovered during customer email campaign synchronization from Apsis to FCC Enterprise. A customer syncing 600 events found only ~100 of ~280 expected contacts visible in Enterprise, despite all contacts appearing correctly in Apsis reports. Through systematic investigation involving log analysis, local environment testing, and collaboration with Rachel, the root cause was identified as a missing `Content-Length` HTTP header, which caused FCC Enterprise's legacy request body parsing library to truncate JSON payloads—a problem masked by FCC Enterprise's non-standard practice of returning HTTP 200 status codes for business-logic errors.

---

## The Initial Problem Report

The issue originated with a customer syncing email campaigns from Apsis to FCC Enterprise (version 12.1). The customer had approximately 300 profiles and was sending around 600 events—relatively small compared to larger customers capable of sending 200,000+ events.

The discrepancy: approximately 280 contacts should have appeared under the campaign in Enterprise, but only ~100 were visible. However, when checking the report in Apsis itself, all expected contacts were present. This immediately raised questions about data loss between systems.

---

## Initial Investigation and False Leads

### Ruling Out Audience-Side Issues

Erik's first step was to verify that Apsis was actually sending the data. He took example profiles that the customer reported as missing and confirmed:
- All profiles the customer reported were actually transmitted to FCC Enterprise
- Nothing was lost or filtered out on the Apsis/Audience side
- The batch itself was quite large, but Erik had seen considerably larger batches to other CRM systems with no issues

### Confirming Successful HTTP Transmission

Erik checked whether the batch post succeeded by examining the HTTP response:
- FCC Enterprise returned an HTTP 200 status code
- This response code indicated successful handling from the HTTP transport layer perspective
- Initial conclusion: "The problem is not in Apsis because we're responding with 200, so the problem is on your side"

### The Customer's Counterargument

The customer then provided logs from FCC Enterprise showing no signs of the batch arriving at all. This contradiction forced deeper investigation.

---

## FCC Enterprise's Non-Standard HTTP Behavior

### Discovery of Separate Error Logging

After approximately one day of back-and-forth investigation, Rachel discovered that FCC Enterprise has a separate error log. For the campaign in question, errors appeared as soon as the campaign was registered:

> "This is not valid JSON, so we are rejecting it."

### The Core Problem: Business Logic in Response Body, Not HTTP Status

FCC Enterprise has an unusual architectural pattern:
- **Standard practice**: Use HTTP status codes (4xx, 5xx) to indicate business-logic errors
- **FCC Enterprise practice**: Use HTTP 200 for all transport-layer successes, but place error messages in the response body

This means:
- HTTP 200 was returned (transport succeeded)
- The actual error message was buried in the response body JSON
- Standard HTTP client libraries checking only status codes would incorrectly report success

> "Enterprise is a very, very special CRM system in lots of ways... they do not utilize HTTP codes for their business logic. They strictly use this for their transport layers."

### Example of Enterprise's Quirks

Erik notes that FCC Enterprise sends `1899` as the default year in date fields if no date is selected—an unusual default value indicative of legacy system behavior.

---

## Root Cause Analysis: The Missing Content-Length Header

### Enabling Debug Logging

To diagnose the actual error, Erik enabled full debug logging in the **broker service** (the proxy layer that decrypts customer API credentials from the database and adds them to outgoing requests). This logging included:
- Request URL
- Request method
- Request body
- Response status code
- Response body

CloudWatch costs normally prevent such verbose logging in production, so this was enabled selectively for this customer investigation.

### Systematic Testing Approach

Erik's debugging progression:

1. **Local server testing**: Set up a test instance and confirmed larger batches sent correctly with no truncation
2. **Rachel's local environment**: Rachel installed FCC Enterprise locally and reproduced the JSON error
3. **Postman comparison**: Tested the same payload in Postman—it worked with no errors
4. **Payload inspection**: Rachel could see the full request body in his local environment and confirmed the JSON was perfectly valid

### The Critical Discovery: Missing Content-Length Header

The key difference between working (Postman) and failing (Apsis) requests:

- **Apsis requests**: No `Content-Length` header set by default
- **Postman requests**: `Content-Length` header set by default

When requests lacked the `Content-Length` header:
- FCC Enterprise's internal request parsing library (not part of the generic connector) would truncate the request body
- The library would cut the body approximately in half
- It would then attempt to parse the truncated content as JSON
- This resulted in invalid JSON and the cryptic "not valid JSON" error

Once the `Content-Length` header was added to the request, everything worked correctly.

---

## The Fix: Adding Content-Length Header Support

### Implementation Location

Rather than requiring every customer or every integration to handle this, Erik implemented the fix centrally in the **broker service**, specifically in the one function where all outgoing requests are created:

```
Broker Service Request Creation Function:
- Method: taken from outbound worker
- Body: taken from outbound worker
- Headers: cleaned up and standardized
- Content-Length: NOW CALCULATED AND INCLUDED
```

By enabling `Content-Length` calculation at this single point:
- All requests to FCC Enterprise (events, consents, etc.) automatically include the header
- No need to add this in multiple places
- Applies to all generic connector requests, not just FCC Enterprise

### Why This Approach Was Chosen

Normally, if bugs exist in the generic connector specification, FCC Enterprise should fix them. However, Erik made a pragmatic decision:

> "In this case it was so much easier for us to fix this on Apsis side. So the customers are not suffering from this as long as it comes from Apsis."

Two minutes to implement the fix in Apsis versus an unknown timeline for FCC Enterprise to modify core system libraries and deploy to all customers.

### Technical Safety

The `Content-Length` header is:
- A standard HTTP header
- Harmless when included
- Supported out-of-the-box by the REST framework being used
- Now calculated automatically by the REST library before sending

---

## Key Lessons and Gotchas

### FCC Enterprise Compatibility Issues

1. **HTTP Status Code Unreliability**: FCC Enterprise (both legacy and 12.1 versions) does not follow standard HTTP conventions. Always check response bodies for error details, not just status codes.

2. **Version-Specific Behavior**: In FCC 12.1, the generic connector has adapted to handle HTTP errors in most cases, but errors from non-connector components (like core request handling) may still bypass this.

3. **Header Differences Between Tools**: Postman adds headers by default (like `Content-Length`) that Apsis wasn't adding. This masked the real issue during testing.

> "Be aware of headers in Postman because they can easily differ from the ones that we are utilizing inside of Apsis."

### Logging and Debugging Strategy

- **Broker Service Role**: Acts as a proxy that decrypts customer API credentials (hidden from developers) and adds them to requests before forwarding
- **CloudWatch Costs**: Full request/response logging is expensive; use selective debugging for specific customers when needed
- **Wrapping with If Clauses**: When enabling debug logging, wrap with conditions to only log for specific customers or scenarios

### The Danger of False Positives

The most critical finding: **Our logs showed success, but the integration was actually failing completely.**

> "Our logs will say success, but in reality it has failed horribly behind the scenes."

This is particularly dangerous because:
- Standard monitoring would pass
- Customer data appears to sync without errors
- But no actual contacts reach the destination system
- Data loss is silent and difficult to detect

---

## Technical Architecture Notes

### Request Flow for Generic Connector

1. **Outbound Worker**: Prepares the request (method, body, etc.)
2. **Broker Service**: 
   - Retrieves encrypted API credentials from database
   - Adds authentication headers
   - Adds standard headers (now including `Content-Length`)
   - Forwards to destination
3. **Squid Proxy** (separate from broker): Enforces domain whitelist—allows/denies based on whether customer is permitted to contact that domain

Erik clarifies these are distinct components:
- Squid proxy: Domain-level access control
- Broker service: Credential injection and request forwarding

---

## Unresolved Questions & Follow-Up Items

- Erik noted he has accumulated unwrapped technical debt and will be doing cleanup work the following day
- Cleanup includes removing old integrations and CloudWatch log groups
- Limited availability for meetings during cleanup work

---

## Key Takeaways

1. **FCC Enterprise's non-standard HTTP error handling is a persistent gotcha**: Always inspect response bodies, not just status codes, when integrating with this system.

2. **The `Content-Length` header is now mandatory in all broker service requests** to prevent silent data truncation in FCC Enterprise's request parsing library.

3. **Postman can mask real integration issues** by automatically including headers that the actual client doesn't. Always test against the real client code, not just HTTP tools.

4. **False-positive success responses are dangerous**: HTTP 200 status codes from FCC Enterprise tell you nothing about whether the request was actually processed. This requires careful logging at the broker service level.

5. **Pragmatic fixes trump perfect architecture**: It was faster and more customer-friendly to fix this on the Apsis side rather than waiting for FCC Enterprise to modify core system libraries.

6. **Selective CloudWatch debug logging is essential for production diagnostics** but must be wrapped conditionally to avoid cost explosion. The broker service is the right place to enable this.

---

**Session completed by**: Lukasz Grabowski  
**Date**: February 11, 2026