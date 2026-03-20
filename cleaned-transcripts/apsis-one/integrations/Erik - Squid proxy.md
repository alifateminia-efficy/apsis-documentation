---
source_file: Erik - Squid proxy.txt
domain: Apsis One - Integrations
topics: [Squid HTTP proxy configuration, access control and whitelisting, broker service credential management, HTTP proxy environment variables, deployment pipeline, security layers in integration]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Squid Proxy, Broker Service, allowed_staging.txt, allowed_domains table, squid.conf, Docker, CloudFormation, ECS, ECR]
session_type: knowledge-transfer
---

## Session Overview

This knowledge transfer session covers the two main proxy systems in the Apsis One Integrations platform: the **Broker Service** and the **Squid HTTP Proxy**. While these components rarely require direct modification during daily development, they are foundational to how all integration services communicate with external CRM systems. The session explains the architecture, configuration mechanism, and deployment process for these security-critical components, including a hands-on demonstration of Squid proxy behavior.

---

## Architecture Overview: Proxy Chain in Integration Services

The integration platform routes all outbound traffic through a two-stage proxy system before reaching external systems:

1. **Broker Service** — handles credential decryption and header injection
2. **Squid Proxy** — enforces domain whitelisting at the HTTP layer
3. **Internet/CRM System** — final destination

[Erik Andersson]: Every service in the integration platform (Integration Manager, Mappings Manager, Outbound Worker, etc.) must route requests through these proxies rather than communicating directly with the CRM system. This middleware approach provides both credential security and egress filtering.

Incoming traffic from CRM systems takes a separate path: the CRM initiates contact with the **Delta Sync Manager service** directly, bypassing the Squid proxy. No filtering is applied to incoming data; instead, validation occurs via webhook hash and secrets verification.

---

## Broker Service: Credential Decryption and Header Injection

### Purpose and Design

[Erik Andersson]: The Broker Service has a single responsibility: decrypt customer credentials and add them to the authorization header before passing requests along. It is "very, very straight forward" and serves as a trust boundary between plaintext code and encrypted storage.

### Credential Storage Strategy

The integration platform differentiates between product variants:

- **Generic Connector**: Credentials are encrypted in the database using AWS KMS. Even developers cannot see plaintext API keys stored this way.
- **Legacy Product, Lime, and FSC Enterprise**: Credentials are stored without KMS encryption. Developers can read plaintext secrets directly from the database.

[Erik Andersson]: This architectural decision means there is "no need for us to like be able to see and have direct access to the customer's API key."

### Broker Service Workflow

1. Service sends request to Broker Service
2. Broker Service queries the database: "Do I have a credential for this CRM system on this account?"
3. If found, Broker Service decrypts the credential using KMS
4. Broker Service adds the decrypted secret to the `X-API-Key` header
5. Request is passed to the next hop (Squid Proxy)

[Lukasz Grabowski]: "It communicates with database, right? Broker service?"

[Erik Andersson]: "Yes, it does. So the broker service will retrieve the hashed value from the database. It will decrypt it... Add that result to the header and then pass the request on."

[Erik Andersson]: The Broker Service is the **only service with access to plaintext secrets**. Developers should not manually print request bodies to view secrets in flight.

---

## Squid Proxy: HTTP Layer Whitelisting and Egress Control

### Strategic Purpose: Defense in Depth

[Erik Andersson]: "The whole reason is because initially like we never wanted any traffic to leave the VPC and we of course unless until it goes to the Internet, but also we want to regulate to where the traffic goes to like we don't want it to go to any unknown services."

Squid provides a security layer against code injection attacks. If malicious code (SQL injection, etc.) is injected into an integration service, the Squid proxy prevents exfiltration to unauthorized destinations by blocking requests to any domain not on the whitelist.

### Why HTTP-Level Filtering (Not Network-Level)

[Erik Andersson]: AWS Security Groups can only filter on IP address and TCP layer. They cannot perform domain-level filtering. Additionally, customer IP addresses are unstable—they may change multiple times per day (e.g., carrier-grade NAT). Domain names remain stable unless the customer reinstalls the integration, so Squid proxies traffic at the HTTP layer to filter by domain.

### Configuration System: ACLs and HTTP Access Rules

Squid configuration uses a composable rule system:

**ACL (Access Control List)** — Define "Lego bricks" (reusable filter criteria)

```
acl lime_instance dst safetodomain.com
acl safe_ports port 443 80
acl static_sites dstdomain "/path/to/allowed_staging.txt"
```

**HTTP_ACCESS** — Apply rules to allow or deny traffic

```
http_access deny !safe_ports
http_access allow static_sites SSL
http_access deny all
```

[Erik Andersson]: Think of ACLs like "Lego bricks" you compose together. Each ACL defines a type (destination domain, port, etc.) and criteria. HTTP_ACCESS rules then decide whether traffic matching those criteria is allowed or denied. The configuration follows an explicit-allow-then-deny-all pattern.

---

## Squid Configuration Files and Allowlists

### Static Allowlist: `allowed_staging.txt` and `allowed_prod.txt`

Location: `cloud-formation/squid-proxy/`

The static allowlist contains domains that must always be reachable, regardless of customer installation:

```
*.amazonaws.com
accounts.apsis.cloud
users.apsis.cloud
folder-stage.apsis.cloud
api.microsoft.com
```

[Erik Andersson]: These are "service requirements which Justin needs to be able to communicate with." Examples include AWS services for logging/monitoring, Apsis internal services (accounts, users, folder), and third-party OAuth providers (Microsoft Dynamics).

**Key gotcha**: When adding a new integration (e.g., Salesforce), developers commonly forget to add the required domain to this static list. The symptom is a 403 Forbidden error.

[Lukasz Grabowski]: "So if we remove from here for example folder stage Apsis cloud, so then we won't be able able to communicate to the service...we will get 403 in logs?"

[Erik Andersson]: "Yes, because it any service which might read try to read data from folders which is most of the services in integration exactly like it will fail when we try to read the data with a forbidden from the folder folder service."

### Domain Format in Allowlist

Domains are specified without scheme (`https://`), port, or path:

```
# Correct:
login.salesforce.com

# Incorrect:
https://login.salesforce.com:443
```

[Erik Andersson]: "the important thing to remember here is that we are doing this on the domain level, so you never add the HTTPS colon slash slash."

### Wildcard Behavior: Single-Level Matching

[Erik Andersson]: Wildcards (`*.example.com`) only match one subdomain level, similar to certificate matching. `*.apsis.cloud` matches `folder.apsis.cloud` but not `sub.folder.apsis.cloud`.

[Michal Rosikiewicz]: "Why don't we just list dot apsis dot cloud here instead adding multiple?"

[Erik Andersson]: "You can do that... I don't think you could do if you have like... A dot B dot C... I don't think it works like a certificate in that aspect that it is only one layer."

---

## Dynamic Allowlist: External Database Lookups via Helper Script

### Problem: Per-Customer Domain Whitelisting

Static allowlists cannot accommodate customer-specific domains because:
1. Each customer may have a different CRM instance domain
2. Domains are specified during installation
3. Restarting Squid to update the allowlist for every new installation is operationally infeasible

### Solution: External Access Control Helper

An external script (referenced in squid.conf as `external_database_sites`) performs runtime lookups:

```
Location: cloud-formation/squid-proxy/squid_helper
```

For each request not found in the static list:

1. Squid checks static allowlist (e.g., `*.amazonaws.com`, `accounts.apsis.cloud`)
2. If not found, Squid pipes the request to the helper script
3. Helper script queries the `allowed_domains` database table
4. Helper checks: "Does this domain exist for this customer section?"
5. If yes, traffic is allowed; if no, traffic is blocked with 403 Forbidden

[Erik Andersson]: "the only thing this little script does is it does a lookup in the database and checks. Does this domain exist for this installation?"

### Database Table: `allowed_domains`

[Lukasz Grabowski]: "And the the table I can see it's allowed domains."

[Erik Andersson]: "Exactly."

The table structure (inferred from discussion) stores:
- Customer domain
- Section/account identifier
- Installation reference

---

## Caching Layer: TTL and Performance

[Erik Andersson]: "Squid has caching on this. So let's say that you have a lot of rapid requests to the same domain. For example, when we send events, we will not do a database look up for every request because that would kill both the Squid proxy and the database."

**TTL: 5 minutes**

Both cache hits and cache misses are remembered for 5 minutes, preventing database query storms when bulk events are sent to the same customer domain repeatedly.

---

## HTTP Proxy Environment Variable: Standard Linux Mechanism

All integration services respect the standard HTTP proxy environment variable mechanism:

```
HTTPS_PROXY=localhost:3128
HTTP_PROXY=localhost:3128
```

[Erik Andersson]: "This environment variable like this is standard protocol used by essentially at least anything in Linux and all of the HTTP transport libraries and services they they should slash have to respect this."

In the integration platform:

- Each ECS task definition includes `HTTPS_PROXY` and `HTTP_PROXY` environment variables pointing to the Squid proxy
- Go's `net/http` client library (and all standard HTTP clients) automatically respects these variables
- **This is not a Golang-specific feature**; it is a Linux standard that any HTTP transport library must honor

[Erik Andersson]: "This is nothing in Golang. This is pure like Linux environment variables."

---

## Deployment and Configuration Structure

### File Organization

```
cloud-formation/
  squid-proxy/
    Dockerfile
    squid.conf
    allowed_staging.txt
    allowed_prod.txt
    squid_helper/
    Makefile
```

### Build Process: Profile-Based Configuration Selection

When building the Squid proxy image, the `AWS_PROFILE` environment variable determines which allowlist is included:

```bash
AWS_PROFILE=staging make build
# Builds image with allowed_staging.txt

AWS_PROFILE=prod make build
# Builds image with allowed_prod.txt
```

[Erik Andersson]: "So here you have the this is what we do when you run make build like you see we build. We build this specific Docker image and we add these allowed sites as inputs and allowed sites. Here is pointing to this configuration file. So if you add here the ABS profile staging as I had, we try to fetch a file called allowed staging dot TXT."

### Deployment Pipeline: Sequential Steps

All integration services follow the same deployment pattern:

```bash
AWS_PROFILE=staging make deploy
```

This executes in order:

1. **make build** — Compile Dockerfile with appropriate allowlist
2. **deploy_repo** — Ensure ECR repository exists
3. **docker push** — Push built image to ECR
4. **AWS CloudFormation deploy** — Deploy container definition and load balancer
5. **force-new-deployment** — Restart ECS service to ensure latest image is active

[Erik Andersson]: "Then the next step is that we have the deploy squid which actually deploys the ECS task. It starts the. It deploys the container definition and it deploys the service."

[Erik Andersson]: The `force-new-deployment` step is critical: "sometimes, sometimes for some reason, like I've seen that you deploy a new image, but the image didn't start with the latest version of it if you do like this force new deployment like you manually restart the service which will pick up the latest version."

### Make Targets Explained

- **make build** — Build image only (for local testing)
- **make deploy** — Full deployment pipeline (build, push, deploy to AWS)

[Erik Andersson]: "For example, if you want to test your test something locally, like test some manager locally, you only need to do like the make build command. You don't need to do like the whole make deploy because that's going to be unnecessary execution time."

---

## Troubleshooting: 403 Forbidden Error Pattern

**Symptom**: Any service receives `403 Forbidden` when attempting outbound HTTP/HTTPS calls.

**Root cause**: The destination domain is not in either:
1. The static allowlist (`allowed_staging.txt` or `allowed_prod.txt`)
2. The dynamic `allowed_domains` database table for that customer

**Why this error is hard to trace**: 
- The 403 is not generated by application code; it is generated by the HTTP transport layer (Squid proxy)
- Application error handlers do not explicitly reference Squid
- The error manifests as a generic HTTP 403

[Erik Andersson]: "you can very easily infer it because the error message that will say is that it is a 403 forbidden and anytime you see like forbidden in the error message immediately think squid proxy because there is like almost nothing else in integration that will say this forbidden error."

[Michal Rosikiewicz]: "Do we have explicit error saying that Squid proxy doesn't have this domain on allowed list or not really?"

[Erik Andersson]: "We we don't. Uh, but you can... there is like almost nothing else in integration that will say this forbidden error."

**To investigate**: Check the Squid proxy logs for `TCP_DENIED` entries matching the requested domain.

---

## Search Across Codebase: Finding Proxy References

To understand which services use the Squid proxy, search the codebase for the environment variable:

```bash
grep -r "HTTPS_PROXY" .
grep -r "HTTP_PROXY" .
```

[Erik Andersson]: "If you would want to disable the squid proxy, like of course first you would need to delete the service itself, but as you can see every service here is being routed through that service."

All ECS task definitions reference the proxy via environment variables in their container definitions.

---

## Disabling Squid Proxy (Not Recommended)

To disable Squid for a service:

1. Delete the Squid proxy service itself
2. Remove `HTTPS_PROXY` and `HTTP_PROXY` environment variables from all ECS task definitions
3. Services will then make direct outbound connections (losing the security layer)

[Erik Andersson]: "If you simply remove this HTTP and HTTPS proxy environment variable, then the services will stop using. They will stop using that service."

[Erik Andersson]: "In theory you could remove this and the functionality of Justin will be remain completely intact. The good thing is that the squid proxy requires essentially no maintenance because the the code to add the... it's there and it hasn't changed for six years."

---

## Disaster Recovery: Profile-Based Configuration

When setting up a disaster recovery (DR) environment, a new AWS profile and corresponding allowlist file must be created:

```
allowed_dr.txt
allowed_production_dr.txt
```

These files should list the DR endpoints for Apsis services:

```
accounts-dr.apsis.cloud
users-dr.apsis.cloud
folder-dr.apsis.cloud
```

Then deploy with:

```bash
AWS_PROFILE=dr make deploy
```

[Erik Andersson]: "If you introduce this for like the disaster recovery, you will need to add a new configuration file that matches the ABS profile you have for it... Here you would need like a disaster recovery... disaster recovery accounts and users disaster recovery folder something."

---

## Hands-On Demonstration: Local Squid Proxy Testing

### Setup Steps

1. Check out the demo branch:
   ```bash
   git fetch origin
   git checkout bit-proxy-demo
   ```

2. Navigate to Squid proxy directory:
   ```bash
   cd cloud-formation/squid-proxy
   ```

3. Build Docker image with staging configuration:
   ```bash
   AWS_PROFILE=staging make build
   ```

4. Run Squid proxy container:
   ```bash
   docker run -p 3128:3128 justin/squid-proxy:latest
   ```

5. In another terminal, open a shell in the running container:
   ```bash
   docker exec -it <container-id> sh
   ```

### Testing Denied Traffic

Request a domain not on the allowlist (e.g., `aftonbladet.se`):

```bash
curl -x localhost:3128 https://aftonbladet.se
```

**Result**: `HTTP 403 Forbidden`

**Log output**: 
```
TCP_DENIED/403 ... CONNECT aftonbladet.se:443
```

### Testing Allowed Traffic

Add the domain to the `allowed_staging.txt`:

```
aftonbladet.se
```

Rebuild and restart the proxy:

```bash
make build
docker run -p 3128:3128 justin/squid-proxy:latest
```

Retry the request:

```bash
curl -x localhost:3128 https://aftonbladet.se
```

**Result**: `HTTP 200 Connection Established`

**Log output**:
```
TCP_TUNNEL/200 ... CONNECT aftonbladet.se:443
```

---

## Key Takeaways

1. **Squid Proxy is foundational**: Every integration service depends on Squid for outbound connectivity. You rarely modify it, but you must understand it to diagnose 403 errors.

2. **Two-layer whitelisting**: Static allowlist covers service dependencies (AWS, Apsis services, OAuth providers); dynamic database lookups cover per-customer CRM domains.

3. **403 Forbidden is the smoking gun**: If you see this error, immediately suspect the Squid allowlist. There is almost nothing else in the integration platform that returns this status code.

4. **Domain format matters**: Domains are specified without `https://`, port, or path. Wildcards only match one subdomain level.

5. **5-minute cache prevents database storms**: Rapid requests to the same domain are cached; the helper script does not query the database for every single request.

6. **Standard Linux environment variables**: Squid is integrated via `HTTPS_PROXY` and `HTTP_PROXY` environment variables—any HTTP client library must respect these.

7. **Profile-based deployment**: Different environments (staging, prod, DR) require different allowlist files. The `AWS_PROFILE` environment variable selects which one to use.

8. **Deployment is standardized**: All services follow the same `make build` → push → CloudFormation → force-restart pattern.

9. **Credential security**: Broker Service is the only component with access to plaintext secrets. Generic Connector credentials are encrypted; legacy products are not.

10. **Low maintenance burden**: The Squid proxy code and static allowlist have been stable for six years. Configuration changes are rare and only needed when adding new OAuth integrations or setting up new environments.

---

## Unresolved Questions

- **Exact database schema for `allowed_domains`**: The structure was inferred from context but not explicitly detailed. Worth documenting the column names and relationships.
- **Multi-level subdomain wildcard support**: Uncertainty about whether Squid supports deeper wildcard patterns. Could be tested empirically.
- **Explicit error handling in application logs**: Whether adding Squid-specific error codes or messages to application logs would improve debuggability (rejected as out of scope for this session).

---

## Action Items

- **Future session**: Discuss the lead/webhook flow, credential rotation, and outbound worker architecture.
- **Local testing**: Participants can now test Squid configuration changes locally using the demo branch and Docker.