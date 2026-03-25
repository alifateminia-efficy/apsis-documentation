---
source_file: Findings of Enterprise issue.txt
domain: Apsis One Integrations
topics: [HTTP Status Code Misuse in CRM Systems, Content-Length Header Bug, Generic Connector Debugging, Enterprise CRM Integration Issues, Request Body Truncation]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Generic Connector, Broker Service, Squid Proxy, Resty HTTP Framework, Efficy Enterprise 12.1, CloudWatch Logging]
session_type: debugging-session
subdomains: [Architecture, Generic Connector, Outbound Flow, Efficy Enterprise 12.1]
---

## Session Overview

Erik Andersson presents findings from a week-long investigation into a critical integration bug between Apsis and Efficy Enterprise (FCC) 12.1. A customer syncing email campaign data discovered that only ~100 out of ~280 expected contacts appeared in the Enterprise campaign, despite all events being successfully logged in Apsis. The investigation revealed that Efficy Enterprise uses HTTP 200 status codes even when rejecting requests with invalid payloads, combined with a data truncation bug triggered by missing Content-Length headers. The fix involved enabling Content-Length headers in the Broker Service's outbound request handler.

---

## Initial Problem Statement

[Erik Andersson]: The customer was syncing email campaigns from Apsis to Efficy Enterprise (FCC) 12.1. They had a relatively small dataset—approximately 300 profiles resulting in 600 events (compared to larger customers who send 200,000+ events). When checking the campaign in Enterprise, only about 100 out of approximately 280 expected contacts were visible, despite the Apsis report showing all expected contacts present.

### Initial Hypotheses

The investigation started with two main questions:
- Did Apsis fail to retrieve all events from the source?
- Did Apsis fail to send the events to Enterprise?

[Erik Andersson]: I verified the source data by checking example profiles the customer identified as missing. All data sent to Enterprise was confirmed to have been transmitted in the batch request. The initial assumption was that nothing could have been missed from the Apsis side.

---

## The 200 Response Code Deception

### Initial Investigation: Logs Showed Success

[Erik Andersson]: When posting campaign events through the generic connector, the HTTP response code received was 200, which indicates successful handling. The logs showed that Apsis had sent the batch and received confirmation from the CRM system.

> The response code is very important. We expect a 200, which means we have successfully handled this. I could see this in our logs, and the contacts that were matching were logged as successful.

However, the customer's Enterprise logs showed no signs of the batch arriving at all, creating a contradiction that took a full day to resolve.

### The Critical Discovery: Enterprise Returns 200 with Error in Body

[Erik Andersson]: Rachel discovered that Enterprise had a separate error log showing errors registered as soon as the campaign was created. The error message indicated invalid JSON being rejected. This conflicted with the 200 status code Apsis received.

The root cause: **Efficy Enterprise does not use HTTP status codes for business logic validation.** The system uses HTTP codes strictly for transport layer concerns (whether the request was successfully received), while placing actual error messages in the response body. Enterprise had responded with:
- HTTP Status: 200 (transport successful)
- Response Body: Error message stating "Invalid JSON, rejecting"

This is a fundamental architectural incompatibility with how Apsis and most modern systems expect CRM APIs to behave.

### Efficy Enterprise Quirks

[Erik Andersson]: Enterprise has several notable idiosyncrasies:
- If no birth date is provided in a date field, it defaults to 1899 as the year
- It does not utilize HTTP codes for business logic—only for transport layer status

---

## The Root Cause: Missing Content-Length Header

### Debugging Process

[Erik Andersson]: To prove the issue wasn't data truncation on the Apsis side, I:
1. Enabled full debug logging in the Broker Service (normally disabled due to CloudWatch costs)
2. Tested with local development server and confirmed the request payload was complete
3. Set up Efficy Enterprise locally on Rachel's development environment to reproduce the error
4. Tested the same payload in Postman—it worked perfectly with no JSON errors

The breakthrough came when comparing the successful Postman request with the failing Apsis request.

### The Header Difference

[Erik Andersson]: The only significant difference discovered was the **Content-Length header**:
- Apsis requests: Content-Length header was not included by default
- Postman requests: Content-Length header was automatically set

When the Content-Length header was missing, a library inside Efficy Enterprise would truncate the request body to half its size and attempt to parse the resulting (incomplete) JSON, which obviously failed validation.

> When you removed the content length header, then a library inside of FCC Enterprise started doing truncating of the data. When we sent a big JSON BLOB without the content length header, that library essentially cut the body in half and then tried to parse it as JSON, which obviously will fail because that's not valid.

### The Fix

[Erik Andersson]: The solution was implemented in the Broker Service, which is the central point where all outbound requests to CRM systems are created. The Broker Service is located between the Outbound Worker and the CRM system and is responsible for:
- Adding customer API credentials from the database
- Adding proper authentication headers
- Forwarding requests to the destination

The Resty HTTP framework (used by Apsis) supports Content-Length header inclusion out-of-the-box. By enabling it in one central location in the Broker Service's request creation function, all outbound requests (events, consents, etc.) now automatically include the Content-Length header.

> Now in the broker service we have one central function where we essentially create each outgoing request. I have enabled that Resty should include the content length in all of the requests, and because I do this here, this is now done for all of the requests going to the CRM system, whether it's events, consents, or not.

---

## Architecture Context: Proxy Layers

[Erik Andersson]: Apsis uses two separate proxy layers in the integration architecture:

1. **Squid Proxy**: Handles access control—determines if a customer is allowed to make requests to specific destination domains
2. **Broker Service**: Handles authentication and credential injection—decrypts the customer's API credentials from the database and adds them to the request headers. Developers do not have direct access to these credentials; the Broker Service acts as a secure intermediary.

The Outbound Worker → Broker Service → CRM System flow ensures credentials are never exposed to developer access.

---

## Debugging Approach: CloudWatch Logging

[Erik Andersson]: Full request/response debugging was enabled specifically for troubleshooting. The Broker Service's debug logs include:
- Request URL
- Request method
- Request body
- Response status code
- Response body

These logs are normally disabled in production due to CloudWatch cost considerations. For this investigation, debug logging was enabled selectively for the specific customer using a conditional check, allowing full visibility into the request/response cycle without incurring costs for other customers.

---

## Key Insights and Caveats

### The Danger of HTTP 200 with Error Bodies

> The main takeaway from this is be aware that FCC Enterprise in particular—the legacy one, but also the 12.1—is not directly compatible with the Apsis way of handling HTTP codes. The problem here is that the error does not come from a component built for the generic connector. It's a standard part of Enterprise giving this error. Normally in 12.1, they have adapted to our HTTP errors, but in this case it doesn't come from the generic connector and thus it is this annoying 200.

This creates a critical logging and monitoring blind spot: **Apsis logs will report success while the integration has actually failed.** Any monitoring systems that rely solely on HTTP status codes will miss these failures.

### Postman Can Be Misleading

[Erik Andersson]: Postman can add to confusion rather than help during debugging because:
- Postman sets default headers (like Content-Length) automatically
- The integration code may not set the same defaults
- This can mask environmental differences between testing and production

> Be aware of headers in Postman because they can easily differ from the ones that we are utilizing inside of Apsis.

### Compromise Between Specification and Pragmatism

[Erik Andersson]: Typically, bugs in the generic connector are Apsis's responsibility to fix, and quirks in the CRM system are the vendor's responsibility. However, in this case:

> This is one of the times where like there were some compromise needed. Typically if there are bugs in the general connector, we have to fix them. If there are weird stuff happening in the CRM system they need to fix it, but in this case it was so much easier for me to fix this on Apsis side. So the customers are not suffering from this as long as it comes from Apsis.

The fix was implemented in Apsis rather than waiting for Efficy Enterprise to resolve the root cause (updating their truncation library or changing configuration) because:
- The Apsis fix took 2 minutes to implement
- Efficy Enterprise's change would require modifying core library code
- Rolling out changes to all Enterprise customers would take significantly longer

---

## Implementation Details

### Broker Service Code Change

The change was made in the central request creation function of the Broker Service:

```
REQUEST CREATION FUNCTION:
- method: from outbound worker
- body: from outbound worker  
- headers: cleaned/normalized
- content-length: NOW ENABLED (Resty automatic calculation before sending)
```

This ensures every request (whether for events, consents, or other operations) includes the Content-Length header before being sent to the CRM system.

### Production Status

[Erik Andersson]: This fix is already deployed to production and is now active for all outbound requests to generic connector CRM systems.

---

## Outstanding Work

[Erik Andersson]: There is additional technical debt to be addressed:
- Removal of old/deprecated integrations
- CloudWatch log group cleanup
- General system maintenance to avoid accumulating technical debt

These tasks are pending and may limit availability for some meetings.

---

## Key Takeaways

1. **Efficy Enterprise uses HTTP 200 for transport success regardless of business logic errors** — Always check response bodies, not just status codes, when integrating with Enterprise CRM systems.

2. **Missing Content-Length headers trigger data truncation in Enterprise** — A legacy library in Enterprise truncates request bodies when Content-Length is absent, causing JSON parsing failures that are masked by HTTP 200 responses.

3. **The Broker Service is the centralized point for request handling** — By enabling Content-Length header calculation at this single point, the fix applies to all outbound requests (events, consents, etc.) without duplication.

4. **CloudWatch debugging can be selectively enabled** — Full request/response logging can be conditionally enabled for specific customers to avoid astronomical costs while maintaining debugging capability.

5. **Postman behavior can mask integration issues** — Default headers in Postman may not match production code behavior; always verify actual integration code, not just Postman tests.

6. **Sometimes pragmatism trumps specification** — When a vendor's system has a fundamental quirk that's expensive to fix on their side, implementing a workaround in the integration layer may be justified.

---

## Unresolved Questions / Action Items

- Technical debt cleanup planned: removal of old integrations and CloudWatch log group cleanup
- Monitor for similar issues with other CRM systems that may not properly use HTTP status codes
- Consider whether other generic connector integrations might have similar header-related issues