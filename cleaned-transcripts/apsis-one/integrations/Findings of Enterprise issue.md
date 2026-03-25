---
source_file: Findings of Enterprise issue.txt
domain: Apsis One Integrations
topics: [HTTP Status Code Handling, Generic Connector Issues, Enterprise CRM Integration, Content-Length Header, Request Body Truncation, Debugging Methodology]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Broker Service, Squid Proxy, FCC Enterprise 12.1, Generic Connector, Resty HTTP Framework, CloudWatch Logs, Outbound Worker]
session_type: debugging-session
subdomains: [Architecture, Efficy Enterprise 12.1 Integration]
---

## Session Overview

Erik Andersson and Lukasz Grabowski investigated a critical integration issue where email campaign syncing from Apsis to FCC Enterprise 12.1 appeared to succeed at the HTTP level (200 status code) but actually failed silently. The customer sent 600 events covering approximately 280 contacts, but only 100 appeared in the enterprise system despite all events reaching the broker service. The root cause was traced to a missing `Content-Length` header in outbound requests, which triggered undocumented truncation behavior in Enterprise's legacy HTTP library. The fix was implemented in the broker service's central request-building function, enabling `Content-Length` calculation by default for all generic connector requests.

---

## The Initial Issue: Missing Contacts in Campaign

[Erik Andersson]: The customer was syncing email campaigns from Apsis to FCC Enterprise 12.1. They sent approximately 600 events for roughly 280 contacts, but only about 100 of those contacts appeared in the enterprise system despite the events being present in the Apsis reporting.

This discrepancy was significant because:
- The customer was small (only 300 profiles)
- Apsis can handle much larger volumes (200,000+ events for bigger customers)
- All expected contacts showed up correctly in Apsis reports
- No contacts were missing from the Apsis side

[Erik Andersson]: The customer's initial concern was whether Apsis had failed to send events or whether the data was lost in transit.

---

## Initial Investigation: Ruling Out Apsis as the Source

[Erik Andersson]: I verified that all profiles the customer claimed should be in the campaign were actually sent to FCC. We batched them and got a **200 HTTP response** from the enterprise CRM system, indicating successful delivery.

The investigation process:
1. Checked that example profiles existed in the batch being sent
2. Verified the batch was sent completely to enterprise
3. Confirmed receipt with a 200 response code from the CRM

[Erik Andersson]: I sat going back and forth for almost a day, and I initially concluded the problem was not in Apsis because I could see the 200 response. But the customer countered with enterprise logs showing no sign of the batch arriving, which increased confusion significantly.

---

## Enterprise's Unusual HTTP Error Handling

[Erik Andersson]: Enterprise is a very special CRM system in how it handles HTTP codes. Unlike standard REST APIs, **Enterprise does not utilize HTTP status codes for business logic—only for transport layer status**. They instead place error messages in the response body.

### The Critical Finding

Rachel discovered that Enterprise had a separate error log for the campaign showing errors immediately upon registration: "this is not valid JSON, so we are rejecting it." However, Enterprise returned a **200 status code** because the HTTP transport succeeded—the package was delivered to the server.

> "Enterprise had responded with an error in the response body saying like this is invalid JSON, you're doing something wrong... but they responded with the status 200 because we we managed to transport the package to them. And this is of course quite a big problem."

[Erik Andersson]: This is a dangerous pattern because our logs show success (200) but the data actually failed to process.

### Other Enterprise Quirks

[Erik Andersson]: Enterprise has additional oddities, such as defaulting to the year 1899 for any date fields where no birth date is selected—"that's of course what you do."

---

## Debugging: Enabling Full Request Logging

[Erik Andersson]: I enabled full debug logging in the **broker service**, which is the outer proxy layer. The broker service decrypts customer API credentials from the database and adds authentication headers before forwarding requests to the destination.

### Broker Service Architecture

The integration flow includes two proxy layers:
1. **Squid Proxy**: Determines if a customer is allowed to make requests to specific destinations
2. **Broker Service**: Decrypts and adds API credentials (developers do not have direct access to these credentials in the generic connector)

[Erik Andersson]: We don't normally log response bodies in production because CloudWatch costs would be astronomical. For this debug session, I wrapped the logging with conditional logic to only log for this specific customer, converting debug logs to info level to ensure they always print.

The debug logs captured:
- Request URL
- Request method
- Response status code
- Response body

This revealed the invalid JSON error from the CRM system.

---

## Root Cause: Missing Content-Length Header

[Erik Andersson]: I suspected HTTP libraries might be truncating the request if it became too large, but I ruled this out because:
- I've seen considerably larger batches sent to other CRM systems without issues
- I tested with an even bigger batch on my local server and the whole request was sent correctly

### Testing in Postman vs. Apsis

[Erik Andersson]: We tried the same payload in Postman and it worked perfectly—no JSON errors. The critical difference was that **Postman includes the `Content-Length` header by default**, while our Apsis requests do not.

Rachel set up FCC Enterprise locally on his development machine and was able to see the full request body. This revealed that **the JSON we received in Enterprise was truncated**, even though it was valid when sent from Postman with the same payload.

> "When you removed the content header, then a library inside of FCC Enterprise, that's where it started doing truncating of the data. So when we sent a big JSON BLOB without the content length header, that library essentially cut the body in half and then try to parse it as JSON, which obviously will fail."

[Erik Andersson]: Once I deployed the fix to add the `Content-Length` header, everything started working again.

---

## The Fix: Content-Length Header in Broker Service

[Erik Andersson]: The solution was implemented in the **broker service** at the central point where all outgoing requests are constructed. The broker service has one central function that builds each request to the CRM system:

- Takes the method from the outbound worker
- Takes the body from the outbound worker  
- Cleans up headers
- **Now includes Content-Length calculation via Resty before sending**

By enabling this at the broker service level, the fix applies to all generic connector requests—events, consents, and any other payload type—without needing changes in multiple places.

[Erik Andersson]: This is standard HTTP practice, so there's nothing dangerous about including this header. The Resty HTTP framework supports this out-of-the-box when enabled.

### Why We Fixed It On Apsis Side

[Erik Andersson]: Typically, if there are bugs in the generic connector, we fix them in Apsis. If there are weird behaviors in the CRM system, the customer should request that the CRM vendor fix it. However, in this case:

> "In this case it was so much easier for me us to fix this on Apsis side. So the customers are not suffering from this as long as it comes from Apsis at least."

The fix was a simple two-minute change to the broker service, whereas asking Enterprise to modify or replace the legacy HTTP library in their core system would take an extremely long time to deploy across all customer instances.

---

## Confusion Factor: Postman vs. Actual Implementation

[Erik Andersson]: This was one of those cases where Postman added more confusion than it helped initially. Postman automatically includes headers that Apsis does not include by default, masking the root issue when testing manually. The payload looked correct in Postman, but was incorrect when sent from Apsis because of the missing header.

---

## Key Takeaways

1. **Enterprise's HTTP Code Handling is Non-Standard**: Be aware that FCC Enterprise (particularly the legacy version, but also 12.1) does not use HTTP status codes for business logic. Errors come back as 200 with error details in the response body. This means success logs can mask actual failures.

2. **Error Location Matters**: In this case, the error did not come from Enterprise's generic connector component but from a standard, core part of the system. The 12.1 version has adapted to Apsis's HTTP error handling in the generic connector, but not in legacy core libraries.

3. **Watch for Header Differences Between Tools and Implementation**: Postman applies default headers that may differ from your actual HTTP client. The `Content-Length` header is standard and safe to include, but was the difference between success and truncation here.

4. **Content-Length Header is Now Standard for All Generic Connector Requests**: The fix has been deployed to production. Resty now calculates the body size and attaches it to the `Content-Length` header for all outbound requests to any CRM system through the broker service.

5. **Debugging at the Broker Service Level**: The broker service is the right place to instrument logging because it's the central point where all requests are constructed. Conditional logging per customer can help keep CloudWatch costs manageable while still capturing necessary debug data.

---

## Action Items

- [Erik Andersson]: Has additional cleanup work to complete, including removal of old integrations and CloudWatch log groups. Will be doing some work the day after this session and may not be available for meetings.
- Monitor for any recurrence of this issue with other customers integrating with FCC Enterprise 12.1.