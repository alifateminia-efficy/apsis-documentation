---
source_file: Erik - Squid proxy.txt
domain: Apsis One Integrations
topics: [Proxy Architecture, Squid Proxy Configuration, Security Filtering, Access Control Lists, Credential Management, Whitelisting, Domain Verification, Deployment Process]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Squid Proxy, Broker Service, ACL (Access Control Lists), HTTP Access Rules, External Access Control Helper Script, allowed_staging.txt, squid.conf, CloudFormation, Docker, ECR]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector, Inbound Flow, Outbound Flow]
---

## Session Overview

This session provided a comprehensive knowledge transfer on the Squid proxy service, a critical but rarely-touched infrastructure component in the Apsis One Integrations platform. Participants learned how the Squid proxy acts as a security layer controlling outbound traffic from integration services to external CRM systems and third-party APIs. The session covered the architectural rationale, configuration mechanisms using Access Control Lists (ACLs), the distinction between static and dynamic domain whitelisting, and practical deployment procedures. A hands-on demonstration involved building and running a local Squid proxy instance and testing domain filtering rules.

---

## Understanding the Role of Proxies in Integration Architecture

### The Two-Proxy Model

The integration platform uses two key proxies/middleware services that all outbound traffic passes through:

1. **Broker Service**: A credential decryption and authorization layer
2. **Squid Proxy**: A domain whitelist enforcement and traffic filtering layer

[Erik Andersson]: "Instead of [services] going immediately to the Internet and to the CRM system, they are moving through 2 proxies that handle two different things."

### Traffic Flow Diagram

The typical request path is:

```
Integration Service 
  → Broker Service (credential decryption/authorization header injection)
  → Squid Proxy (destination domain whitelist verification)
  → Internet
  → CRM System
```

Responses from the CRM go directly back to the requesting service without traversing the proxies. Incoming webhooks from CRM systems to Apsis are verified through webhook hash validation and secrets, but are not filtered through Squid.

---

## Broker Service: Credential Management Layer

### Purpose and Operation

The Broker Service is a prerequisite service that handles sensitive credential management before requests leave the integration platform.

### Encryption and Key Management

[Erik Andersson]: "One major difference that we have in the general connector compared to the legacy product is that we don't store the text like in clear text in the database."

**Generic Connector behavior:**
- Credentials (API keys, passwords) are encrypted at rest in the database using AWS KMS
- Developers cannot view credentials in plaintext
- When a service makes a request to a CRM system, the Broker Service decrypts the secret and adds it to the request header

**Legacy Product (Lime, FSC Enterprise):**
- Credentials are stored without KMS encryption
- Developers can read credentials directly from the database if needed

### Broker Service Workflow

1. Receives a request from an integration service
2. Queries the database for credentials associated with the CRM system/section/account combination
3. Retrieves the encrypted credential value
4. Decrypts the credential using KMS
5. Injects the decrypted credential into the `X-API-Key` header (or equivalent)
6. Passes the request forward to Squid Proxy

[Erik Andersson]: "The broker service will retrieve the hashed value from the database. It will decrypt it. Add that result to the header and then pass the request on."

---

## Squid Proxy: Outbound Traffic Filtering and Whitelisting

### Strategic Purpose

Squid Proxy serves as a security boundary that prevents compromised services from exfiltrating data to unauthorized destinations.

[Erik Andersson]: "The use case here would be let's say that someone somehow manages to inject malicious code into our service, be it like SQL injection or whatever. You will not be able to get data out from the system by sending it to some random server."

### Design Rationale

The Apsis team considered using AWS Security Groups for IP-based filtering but rejected this approach because:

- Customer CRM system IP addresses are not static—they may change multiple times per day
- Some customers operate on carrier-grade networks with dynamic IP allocation
- Domain names, by contrast, are stable unless a customer explicitly reinstalls with a new domain

[Erik Andersson]: "AWS cannot handle filtering on HTTP level, only on the IP address on the TCP level. So therefore you need to utilize some kind of third party for this."

### Configuration Structure

Squid proxy configuration uses a Lego brick metaphor: you define reusable building blocks (ACLs) and then explicitly allow or deny traffic based on combinations of these blocks.

#### Key Terminology

**ACL (Access Control List)**: A named rule block that defines a category of traffic.

```
acl <name> <type> <values>
```

**HTTP Access Rule**: Explicit allow/deny directives that apply to ACLs.

```
http_access allow/deny <acl_name>
```

---

## ACL Configuration and Static Whitelisting

### Structure of squid.conf

The Squid proxy configuration file is located at:

```
CloudFormation/squid-proxy/docker/squid.conf
```

### Static Domains List

A file called `allowed_staging.txt` (or `allowed_<environment>.txt`) contains domains that should always be reachable, regardless of customer installation:

```
CloudFormation/squid-proxy/allowed_staging.txt
```

### Domains in Static Whitelist

[Erik Andersson]: "These are service requirements which Justin needs to be able to communicate with."

Common entries include:

```
# AWS Services
*.amazonawS.com
# Apsis internal services
*.apsis.cloud
staging.apsis.cloud
accounts.apsis.cloud
users.apsis.cloud
folder.apsis.cloud
# Microsoft (for Dynamics integration support)
<Microsoft domains>
```

### Wildcard Behavior Warning

⚠️ **Important caveat**: Wildcard matching works only one level deep, similar to certificate wildcard behavior.

- `*.apsis.cloud` matches `staging.apsis.cloud` and `accounts.apsis.cloud`
- `*.apsis.cloud` does NOT match `sub.staging.apsis.cloud`

[Erik Andersson]: "I don't think you could do if you have like... A.B.C and then... it would only go up to like B. It would not go to A. I think it works like a certificate in that aspect that it is only one layer."

### Maintenance Burden

The static list rarely changes. It only requires updates when adding entirely new OAuth integrations that require external service communication.

[Erik Andersson]: "This static list... essentially never changes. But let's say that you are asked to add a new integration... and you need to like make a request to Salesforce to get the data... we have fallen in this trap a lot of times when we added new integrations like we forget to add it here."

---

## Dynamic Whitelisting: Customer-Specific Domains

### The Problem with Static Whitelists

Each customer installs the integration with their own CRM domain (e.g., `customer1.salesforce.com`, `customer2.crm.microsoft.com`). You cannot hardcode every customer's domain into the static list.

### External Access Control Helper Script

Squid supports **external access control** programs—small helper services that perform dynamic lookups.

[Erik Andersson]: "An external access control is essentially like a little helper service that is run alongside Squid... You can find the code there in the squid helper... anytime that the customer installs we will add their... domain that they enter in the API URL during the installation phase to our database."

#### Workflow

1. Squid checks if destination domain is in the static `allowed_staging.txt` list
2. If not found, Squid pipes the request to an external helper script
3. The helper script performs a database lookup in the `allowed_domains` table
4. The script checks: "Does this domain exist for this account/section/installation?"
5. If yes, the request is allowed; if no, it is denied with a 403 Forbidden

### Database Table Reference

[Lukasz Grabowski]: "The table I can see it's allowed domains."

**Table**: `allowed_domains`

Likely schema (inferred from discussion):
```
account_id
section_id
domain
```

### Caching Strategy

[Erik Andersson]: "Squid has caching on this. So let's say that you have a lot of rapid requests to the same domain... it will not do a database look up for every request because that would kill both the Squid proxy and the database so it has this TTL on 5 minutes."

TTL (Time To Live): **5 minutes**

This means:
- First request to a domain triggers a database lookup
- Squid caches the result (allow or deny) for 5 minutes
- Subsequent requests within that window use the cached decision
- No additional database queries during the cache period

---

## Squid Configuration Rules: Explicit Allow, Deny All

### Port Restriction

Only HTTPS traffic is allowed:

```
acl safe_ports port 443 80
http_access deny !safe_ports
```

[Erik Andersson]: "We enforce HTTPS traffic for integration... we will allow requests for domains in this list to the SSL port and the SSL port is 443."

### Static Sites Rule

```
acl static_sites dstdomain "/path/to/allowed_staging.txt"
http_access allow static_sites port SSL
```

### External Database Sites Rule

```
acl external_database_sites external_acl_type_name "check_domain_in_db"
http_access allow external_database_sites port SSL
```

### Default Deny

```
http_access deny all
```

[Erik Andersson]: "We first do the explicit allows and then we do a deny all for everything we haven't given explicit permission for."

---

## Error Handling and Debugging

### 403 Forbidden as a Primary Indicator

When a domain is not whitelisted, the request fails with an HTTP 403 Forbidden response.

[Erik Andersson]: "You can very easily infer it because the error message that will say is that it is a 403 forbidden and anytime you see like forbidden in the error message immediately think squid proxy because there is like almost nothing else in integration that will say this forbidden error."

### Lack of Explicit Error Messages

The platform does not return a custom error message identifying the Squid proxy as the blocker. The 403 is a low-level HTTP response from the proxy.

[Erik Andersson]: "This error is not really handled in the business logic by us, then we we don't have that error because the traffic is routed through the HTTPS proxy parameter which is handled in the HTTP client... it's this that error is not really part of. It's not an error that we handle in the code, it's handled in the transport layer."

### Log Inspection

Squid proxy logs show detailed information about blocked requests:

```
TCP_DENIED/403 <bytes> <method> <url> <user> <hierarchy> <type>
```

Example from the session:
```
TCP_DENIED/403 0 CONNECT aftonbladet.se:443 - HIER_NONE/- text/html
```

---

## Practical Setup: Local Testing

### Prerequisites

- Git access to the integration repository
- Docker installed locally
- AWS credentials configured (to pull images from ECR)

### Building the Squid Proxy Locally

1. **Fetch the latest branch metadata:**
   ```bash
   cd integration-repository
   git fetch
   ```

2. **Check out the Squid proxy demo branch:**
   ```bash
   git checkout <branch-name>
   ```

3. **Navigate to the Squid proxy folder:**
   ```bash
   cd CloudFormation/squid-proxy
   ```

4. **Build the Docker image:**
   ```bash
   AWS_PROFILE=staging make build
   ```

   This command:
   - Reads the `Makefile`
   - Builds a Docker image named `justin-squid-proxy:latest`
   - Uses the configuration from `allowed_staging.txt` (based on `AWS_PROFILE`)

5. **Run the proxy container:**
   ```bash
   docker run <docker-run-options> justin-squid-proxy:latest
   ```

   The container will start listening on port 3128 (default Squid port) for incoming requests.

6. **Verify the container is running:**
   ```bash
   docker ps
   docker exec -it <container-id> /bin/sh
   ```

### Testing with curl

Once the container is running, test domain filtering from within the container:

1. **Enter the container shell:**
   ```bash
   docker exec -it <container-id> /bin/sh
   ```

2. **Test a whitelisted domain:**
   ```bash
   export HTTPS_PROXY=localhost:3128
   curl https://aftonbladet.se
   ```

   **Expected result**: HTTP 200 Connection Established (domain is allowed)

3. **Test a non-whitelisted domain:**
   ```bash
   curl https://example-non-whitelisted.com
   ```

   **Expected result**: HTTP 403 Forbidden (domain is denied)

4. **Check Squid logs:**
   The container logs will show:
   ```
   TCP_DENIED/403 CONNECT example-non-whitelisted.com:443
   TCP_TUNNEL/200 CONNECT aftonbladet.se:443
   ```

### Adding a Domain to the Whitelist (Testing)

To test adding a domain:

1. **Edit `allowed_staging.txt`:**
   ```
   aftonbladet.se
   ```

2. **Rebuild the container:**
   ```bash
   AWS_PROFILE=staging make build
   ```

3. **Stop and restart the container:**
   ```bash
   docker stop <container-id>
   docker run ... justin-squid-proxy:latest
   ```

4. **Re-test with curl:**
   The previously denied domain should now return 200.

### Important Implementation Detail

The domain name must be specified without the protocol scheme:

```bash
# ✓ Correct
aftonbladet.se

# ✗ Incorrect
https://aftonbladet.se
```

[Erik Andersson]: "We are doing this on the domain level, so you never add the HTTPS colon slash slash. It's like always just the pure domain that we that we work on."

---

## Integration with Golang Services

### Environment Variable Configuration

All integration services are configured to use the Squid proxy via a standard Linux environment variable:

```
HTTPS_PROXY=localhost:3128
HTTP_PROXY=localhost:3128
```

[Erik Andersson]: "This is standard protocol used by essentially at least anything in Linux and all of the HTTP transport libraries and services they they should slash have to respect this."

### ECS Task Configuration

For services running in AWS ECS, the environment variables are set in the **container definition**:

```yaml
environment:
  - name: HTTPS_PROXY
    value: "squid-proxy-service:3128"
  - name: HTTP_PROXY
    value: "squid-proxy-service:3128"
```

### Golang HTTP Client Behavior

The Golang standard library `http.Client` automatically respects these environment variables without requiring explicit code changes.

[Erik Andersson]: "This is nothing in Golang. This is just a standard HTTP environment variable that we have configured so the Golang httpclient will see this environment variable and therefore it will send all requests that it does through the HTTPS proxy."

---

## Deployment Process

### Overview

Every service in the integration platform follows a standardized deployment pattern. The Squid proxy deployment is typical of this pattern.

### Deployment Makefile Structure

**Location**: `CloudFormation/squid-proxy/Makefile`

The Makefile defines several targets:

1. **make build**: Build the Docker image locally
2. **make deploy-repo**: Ensure ECR repository exists
3. **make docker-push**: Push the image to ECR
4. **make deploy**: Full deployment (all steps)

### Full Deployment Command

To deploy Squid proxy to the staging environment:

```bash
cd CloudFormation/squid-proxy
AWS_PROFILE=staging make deploy
```

This executes the following steps in sequence:

1. **Build** the Docker image with the `allowed_staging.txt` configuration
   ```bash
   docker build -t justin-squid-proxy:latest \
     --build-arg ALLOWED_SITES=allowed_staging.txt .
   ```

2. **Ensure ECR repository exists** (deploy-repo target)

3. **Authenticate to ECR:**
   ```bash
   aws ecr get-login-password --region <region> | docker login ...
   ```

4. **Push the image to ECR:**
   ```bash
   docker push <ecr-uri>/justin-squid-proxy:latest
   ```

5. **Deploy CloudFormation stack** (updates ECS task definition, load balancer, service)
   ```bash
   aws cloudformation deploy \
     --template-file cloudformation.yaml \
     --stack-name squid-proxy-staging \
     --parameter-overrides ImageUri=<ecr-uri>:latest
   ```

6. **Force restart the ECS service:**
   ```bash
   aws ecs update-service \
     --cluster integration-staging \
     --service squid-proxy \
     --force-new-deployment
   ```

### Why Force Restart?

[Erik Andersson]: "Because sometimes, sometimes for some reason, like I've seen that you deploy a new image, but the image didn't start with the latest version of it if you do like this force new deployment like you manually restart the service which will pick up the latest version."

### Local Testing vs. Deployment

For local development/testing, only the **build** step is needed:

```bash
AWS_PROFILE=staging make build
```

The full `make deploy` is only used when pushing to staging/production.

### Deployment Pattern Consistency

All integration services (managers, workers, sinks) follow this same deployment pattern:

- Same Makefile structure
- Same ECR push process
- Same CloudFormation update
- Same service restart pattern

This consistency means developers can apply knowledge from Squid proxy deployment to any other service.

---

## Configuration for Multiple Environments

### Environment-Specific Whitelists

Separate whitelist files exist for each environment:

```
CloudFormation/squid-proxy/allowed_staging.txt
CloudFormation/squid-proxy/allowed_prod.txt
CloudFormation/squid-proxy/allowed_dr.txt  (Disaster Recovery)
```

### Specifying Environment During Build

The `AWS_PROFILE` environment variable controls which whitelist file is used:

```bash
AWS_PROFILE=staging make build   # Uses allowed_staging.txt
AWS_PROFILE=prod make build      # Uses allowed_prod.txt
AWS_PROFILE=dr make build        # Uses allowed_dr.txt
```

### Disaster Recovery Considerations

When setting up disaster recovery environments, you must:

1. Create a new `allowed_dr.txt` file
2. Include all necessary Apsis internal services with DR-specific domains:
   ```
   accounts-dr.apsis.cloud
   users-dr.apsis.cloud
   folder-dr.apsis.cloud
   ...
   ```

[Erik Andersson]: "If you introduce this for like the disaster recovery, you will need to add a new configuration file that matches the ABS profile you have for it... you would need like a disaster recovery accounts and users disaster recovery folder something."

---

## Security Implications and Limitations

### Attack Vectors Mitigated

The Squid proxy prevents scenarios like:

- **Malicious code injection**: Even if an attacker injects SQL injection or similar code, they cannot exfiltrate data to arbitrary external servers
- **Supply chain attacks**: Compromised dependencies in services cannot phone home to attacker-controlled servers
- **Accidental credential leaks**: Misconfigured services cannot accidentally send data to wrong CRM systems

### Scope Limitations

The proxy only protects **outbound traffic** from integration services:

- **Incoming webhooks**: From CRM → Apsis Delta Sync Manager
  - Not filtered through Squid
  - Validated using webhook hashes and secrets instead
  - No HTTP-level domain verification

[Erik Andersson]: "When the CRM initiates the contact with Apsis, then it will go to the Delta Sync manager service. And we don't have any, we don't have any filtering there in the incoming data."

### Operational Risk

Despite being a security boundary, Squid requires almost no operational maintenance:

- Zero configuration changes in 6+ years for core functionality
- No daily/weekly patching needed
- Only requires updates when adding new OAuth integrations
- Most common mistake: forgetting to add new domains to static list during integration development

---

## Key Takeaways

1. **Two-proxy architecture**: Broker Service (credential decryption) → Squid Proxy (domain whitelist) → Internet
   
2. **Squid serves as a security boundary**: Prevents data exfiltration to unauthorized destinations if integration code is compromised

3. **Static whitelist** (`allowed_staging.txt`): Contains Apsis internal services and public OAuth providers. Rarely changes.

4. **Dynamic whitelist** (`allowed_domains` table + external access control script): Stores per-customer domain allowances set during installation

5. **Domain-based filtering** (not IP-based) accommodates customers with dynamic IP addresses

6. **403 Forbidden = Squid proxy**: This is the primary error indicator for missing domain whitelisting

7. **5-minute caching**: Squid caches domain lookup results to avoid hammering the database

8. **HTTPS_PROXY environment variable**: All services are configured via Linux standard environment variables; no code changes needed

9. **Consistent deployment pattern**: Build → Deploy Repo → Docker Push → CloudFormation Update → Force Restart

10. **Low maintenance overhead**: Squid proxy requires minimal operational attention once deployed

11. **No explicit error messaging**: 403 errors are generated by the HTTP transport layer, not business logic

12. **Incoming traffic is not filtered**: Webhook validation relies on cryptographic signatures, not domain whitelisting

---

## Unresolved Questions and Action Items

1. **Wildcard matching clarification**: Exact documentation on whether `*.*.apsis.cloud` patterns are supported (assumed single-level only based on certificate analogy, but not definitively tested)

2. **Docker image pull issues**: Some participants experienced "access denied" errors pulling from AWS ECR. Confirm whether all team members need specific IAM permissions or if local credential setup issues exist.

3. **curl not available in Alpine image**: The default Squid Docker image (Alpine-based) doesn't include `curl`. Session worked around this, but clarify if `curl` should be added to the Dockerfile for future testing.

4. **External access control script location and details**: Mentioned as "squid helper" but exact file path and implementation language not explicitly shown during session.

---

**Session Duration**: 1 hour 6 minutes 51 seconds  
**Date**: November 20, 2025  
**Facilitator**: Erik Andersson  
**Participants**: Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski