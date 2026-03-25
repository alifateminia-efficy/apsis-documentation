---
source_file: Erik - Squid proxy.txt
domain: Apsis One Integrations
topics: [Squid Proxy Architecture, Access Control Lists (ACLs), HTTP Proxy Configuration, Whitelist Management, Broker Service, Credential Handling, Security Filtering, Docker Deployment, Environment Configuration]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Squid Proxy, Broker Service, ACL (Access Control Lists), External Access Control Helper, Allowed Domains Database, ECS Tasks, CloudFormation, Docker, HTTPS Proxy Environment Variable]
session_type: knowledge-transfer
subdomains: [Architecture]
---

## Session Overview

This session covers the Squid HTTP proxy infrastructure that sits between all integration services and external CRM systems. Erik Andersson explains why the Squid proxy exists as a security layer, how it uses ACLs (Access Control Lists) to whitelist allowed domains, and how both static domain lists and dynamic per-customer domain lookups work. The team then performs a hands-on demonstration of running Squid locally, testing blocked requests, and adding domains to the allowed list. The session concludes with an overview of the deployment pipeline and how configuration changes are applied across environments.

---

## Architecture Overview: The Request Flow Through Proxies

### Two-Proxy Pattern in Integration

All integration services communicate with external CRM systems through exactly two proxies, which are transparent to daily development work but critical to understand for system maintenance and troubleshooting.

[Erik Andersson]: The architecture looks like this: different services (integration manager, mappings manager, outbound worker, etc.) do not go directly to the Internet or CRM systems. Instead, requests flow through two sequential proxies.

```
Service → Broker Service → Squid Proxy → Internet → CRM System
```

The request originates from an integration service, passes through the broker service, then through the Squid proxy, and finally exits to the Internet and the customer's CRM system.

### Role of the Broker Service

The broker service is the first proxy layer and handles credential injection. It is not a pure network proxy but rather business logic that performs a single critical task.

[Erik Andersson]: The broker service receives a request, checks the database for credentials associated with the CRM system for the given section and account, decrypts that credential using KMS, and adds it to the `X-API-Key` header before passing the request to the Squid proxy.

**Key Security Model**: 
- In the generic connector, credentials are encrypted at rest in the database using AWS KMS
- Only the broker service has access to decryption keys
- Developers cannot view customer API keys in plaintext (unless they manually inspect request logs, which should not be done)
- Legacy products (Lime, FSC Enterprise) do not use KMS encryption, so developers can still read those secrets

[Lukasz Grabowski]: Does the broker service communicate with the database?

[Erik Andersson]: Yes. The broker service retrieves the hashed/encrypted credential value from the database, decrypts it, and adds it to the header.

---

## Squid Proxy: Purpose and Security Model

### Why Squid Proxy Exists

The Squid proxy serves as an outbound traffic filter that enforces domain-level whitelisting. This adds a security boundary against code injection attacks.

[Erik Andersson]: The core motivation: if someone manages to inject malicious code into our service (SQL injection, arbitrary code execution, etc.), they cannot exfiltrate data by sending it to arbitrary external servers. The Squid proxy blocks all outbound requests to domains that are not explicitly whitelisted.

**Secondary Motivation**: AWS security groups can only filter on IP addresses at the network layer, but customer CRM instances are identified by domain names. IPs are unstable (customer networks may change IPs multiple times per day or use carrier-grade NAT), but domains are stable. Since AWS VPC security groups cannot filter on HTTP-level domain names, a separate HTTP-aware proxy (Squid) is needed.

### Incoming vs. Outgoing Traffic

[Lukasz Grabowski]: Does the CRM communicate back through the Squid proxy?

[Erik Andersson]: No. Outgoing traffic from integration services goes through Squid. When the CRM initiates contact with Apsis (e.g., webhooks), the traffic is directed to the Delta Sync manager service, which does not use Squid filtering. For incoming data, we verify authenticity using webhook hashes and pre-shared secrets rather than domain whitelisting.

---

## Squid Proxy Configuration: ACLs and HTTP Access Rules

### Configuration Location and Structure

```
CloudFormation/
└── squid-proxy/
    ├── Dockerfile
    ├── squid.conf
    ├── allowed-staging.txt
    ├── allowed-prod.txt
    └── squid-helper/
```

The Squid proxy code and configuration reside in the `CloudFormation/squid-proxy` folder, which contains global infrastructure not tied to any specific service.

### ACL (Access Control List) Concept

[Erik Andersson]: Think of ACL configuration like defining Lego bricks. Each ACL is a named set of rules, and HTTP access rules combine these bricks to allow or deny traffic.

An ACL has two key components:
1. **Name**: What you call this rule set
2. **Type**: What kind of rule it is (destination domain, port, source IP, etc.)

### Example ACL Definitions

```
acl lime_instance dstdomain example-crm.lime.com
acl safe_ports port 443 80
acl static_sites dstdomain "/allowed-staging.txt"
```

- `lime_instance`: An ACL that matches traffic to the domain `example-crm.lime.com`
- `safe_ports`: An ACL that matches traffic to ports 443 (HTTPS) and 80 (HTTP)
- `static_sites`: An ACL that matches traffic to any domain listed in the `allowed-staging.txt` file

### HTTP Access Rules

HTTP access rules determine what traffic is allowed or denied based on matching ACLs.

```
http_access deny !safe_ports
http_access allow static_sites safe_ports
http_access deny all
```

This means:
1. Deny any traffic that is not on safe ports (443 or 80)
2. Allow traffic to domains in `static_sites` ACL on safe ports
3. Deny everything else (default deny)

[Erik Andersson]: We always explicitly allow specific traffic, then deny everything else. This is a whitelist model, not a blacklist model.

---

## Static Whitelisting: The Allowed Domains Lists

### Role of allowed-staging.txt and allowed-prod.txt

These files contain a static list of domains that should always be reachable by integration services, regardless of customer configuration. These lists are per-environment (staging, production, disaster recovery, etc.).

**Example entries from allowed-staging.txt**:

```
*.amazonaws.com
*.apsis.cloud
accounts.apsis.cloud
users.apsis.cloud
folder.apsis.cloud
*.microsoft.com
```

- `*.amazonaws.com`: Allows all AWS services (CloudWatch, Secrets Manager, etc.)
- `*.apsis.cloud`: Allows connections to internal Apsis services
- `*.microsoft.com`: Allows connections to Microsoft services (for Dynamics integration)

### When Does This List Change?

[Erik Andersson]: This static list almost never changes. It only requires updates when:

1. **Adding a new OAuth/OIDC integration** that requires connecting to a third-party OAuth provider
2. **Changing Apsis infrastructure** (adding new Apsis-owned domains)
3. Adding support for a new CRM vendor with cloud endpoints

[Lukasz Grabowski]: If we remove `folder.apsis.cloud` from this list, then services that try to read from Folder won't be able to, correct?

[Erik Andersson]: Exactly. Most integration services need to read from Folder at some point. If the domain is not whitelisted, requests will fail with a 403 Forbidden from the Squid proxy.

### Common Mistake: Forgetting to Add New Integration Domains

[Erik Andersson]: This list has been a pain point historically. When we add a new integration (e.g., Salesforce), developers sometimes forget to add the Salesforce domain (`login.salesforce.com`) to this list. The integration will fail with 403 Forbidden errors.

---

## Dynamic Whitelisting: Per-Customer Domain Lookup

### The Problem with Static Lists

Static domain lists work for internal Apsis services and third-party OAuth providers. However, customers have unique CRM instances at different domains (e.g., `customer1.crm.com`, `customer2.crm.com`). We cannot maintain a static file with every customer's domain.

### Solution: External Access Control Helper

For any domain not found in the static list, Squid pipes the request to an external helper script (the **external access control**), which is small program (Python, Bash, Go) that performs a dynamic lookup.

**Flow**:

1. Squid receives a request to `customer-crm.com`
2. Check: Is `customer-crm.com` in `static_sites` list? No.
3. Pipe the request to the external helper script
4. Helper script queries the database table `allowed_domains`
5. Check: Is `customer-crm.com` registered for this installation/section/account? 
   - If yes: allow the request
   - If no: block the request (403)

[Erik Andersson]: When a customer installs an integration, they provide their CRM API URL (e.g., `https://ericsdomain.com/api`). We extract the domain (`ericsdomain.com`) and store it in the `allowed_domains` database table along with the section ID and account ID.

**Key field**: `allowed_domains` table (as referenced by [Lukasz Grabowski])

### Caching in Squid

[Erik Andersson]: Squid caches the result of external access control lookups with a TTL (time-to-live) of 5 minutes. This is critical for performance: if every request to the customer's CRM triggered a fresh database lookup, we would overload both Squid and the database.

When a domain lookup result is cached, rapid requests to the same domain use the cached result (either hit or miss) for the next 5 minutes, avoiding redundant database queries.

---

## Diagnosing Squid Proxy Issues

### The 403 Forbidden Error

[Erik Andersson]: Anytime you see a 403 Forbidden error in integration, immediately think Squid proxy. There is almost nothing else in the system that returns this error code.

However, the error is returned at the HTTP transport layer, not in our application logic. So you may not see a specific "Squid denied this" error message in application logs. Instead, you see a generic HTTP 403.

[Michal Rosikiewicz]: Is there a way to identify that Squid specifically blocked the request, separate from other 403 sources?

[Erik Andersson]: Not really in the business logic. The HTTP client sees the 403 response from the proxy. But you can infer it's Squid:
- The error occurred when trying to reach an external service
- The error is 403 Forbidden
- Check the Squid proxy logs directly (if you have access) to see the blocked domain

### Debugging: Check the Squid Logs

If you can access the Squid proxy container logs, you'll see entries like:

```
TCP_DENIED/403 ... CONNECT aftonbladet.se:443
```

This tells you:
- The request type: `TCP_DENIED` (blocked)
- HTTP status: `403`
- Action: `CONNECT` (tunnel to)
- Target domain: `aftonbladet.se`
- Target port: `443`

---

## Hands-On Demo: Running Squid Locally and Testing Whitelisting

### Setting Up Local Squid

The team performed a local demonstration of Squid behavior. Prerequisites:

1. Check out the `bit-proxy-demo` branch from the integration repository
2. Navigate to `CloudFormation/squid-proxy`
3. Ensure AWS credentials are configured locally (for pulling ECR images)

### Build and Run Commands

```bash
cd CloudFormation/squid-proxy

# Build the Docker image with the staging environment configuration
AWS_PROFILE=staging make build

# Run the container
docker run -d -p 3128:3128 --name squid-proxy justin/squid-proxy:latest

# Get the container ID
docker ps
```

[Erik Andersson]: The important detail: when building with `AWS_PROFILE=staging`, the build process loads the `allowed-staging.txt` configuration file. If you use `AWS_PROFILE=prod`, it would load `allowed-prod.txt`. The profile name must match the configuration file name.

### Testing Blocked Requests

Inside a container with Squid running, you can test proxy connectivity:

```bash
# Set the HTTPS_PROXY environment variable (standard Linux convention)
export HTTPS_PROXY=localhost:3128

# Try to reach a domain NOT on the whitelist
curl -v https://aftonbladet.se

# Result: 403 Forbidden
# TCP_DENIED/403 ... CONNECT aftonbladet.se:443
```

[Lukasz Grabowski]: Why did that fail with 403?

[Erik Andersson]: Because `aftonbladet.se` is not in the `allowed-staging.txt` static list, and it's not in the `allowed_domains` database (since we're testing locally with no customer data).

### Adding a Domain and Retesting

To allow a domain, add it to the `allowed-staging.txt` file:

```
aftonbladet.se
```

Then rebuild and redeploy the Squid proxy:

```bash
# Stop the running container
docker stop <container-id>
docker rm <container-id>

# Rebuild
AWS_PROFILE=staging make build

# Run again
docker run -d -p 3128:3128 --name squid-proxy justin/squid-proxy:latest

# Test again inside the container
export HTTPS_PROXY=localhost:3128
curl -v https://aftonbladet.se

# Result: 200 OK (or actual HTTP response from aftonbladet.se)
# TCP_TUNNEL/200 ... CONNECT aftonbladet.se:443
```

[Erik Andersson]: Notice the difference:
- **Blocked**: `TCP_DENIED/403`
- **Allowed**: `TCP_TUNNEL/200` followed by the actual HTTP response

### Important Caveat: Domain-Only Whitelisting

When adding domains to `allowed-staging.txt`, use only the domain name, not the full URL:

```
✓ Correct:   aftonbladet.se
✗ Incorrect: https://aftonbladet.se
✗ Incorrect: https://aftonbladet.se/
```

Squid works at the HTTP layer and filters on domain names and ports, not the full URL path.

### The HTTPS_PROXY Environment Variable

[Erik Andersson]: The `HTTPS_PROXY` (and `HTTP_PROXY`) environment variables are standard Linux conventions. Any HTTP client library that respects these variables will automatically route traffic through the proxy.

In integration, every ECS task has `HTTPS_PROXY` and `HTTP_PROXY` environment variables pointing to the Squid proxy. The Golang `net/http` package automatically respects these variables—there is no special code required. This is purely an OS-level configuration.

```
HTTPS_PROXY=squid-proxy.internal:3128
HTTP_PROXY=squid-proxy.internal:3128
```

---

## Disabling or Removing Squid Proxy

### Can We Remove Squid?

[Lukasz Grabowski]: What happens if we remove the Squid proxy entirely?

[Erik Andersson]: In theory, the integration functionality would remain completely intact. The systems would still work. However, you would lose the security boundary against code injection attacks attempting to exfiltrate data to unauthorized domains.

The trade-off:
- **With Squid**: Security enforcement; zero maintenance (the code hasn't changed in 6 years)
- **Without Squid**: Slightly simpler architecture, but loss of a critical security control

### How Squid Is Configured in ECS

Every integration service (manager, worker, sink, etc.) is deployed as an ECS task with environment variables:

```
HTTPS_PROXY=squid-proxy.internal:3128
HTTP_PROXY=squid-proxy.internal:3128
```

To disable Squid, you would:

1. Delete the `HTTPS_PROXY` and `HTTP_PROXY` environment variables from all ECS task definitions
2. Services would stop routing traffic through Squid
3. Delete the Squid proxy ECS service itself

This is straightforward because the proxy integration is entirely through environment variables, not hardcoded service discovery or DNS.

---

## Deployment: Building, Pushing, and Deploying Squid

### The Makefile Pattern

Every service in integration, including Squid, follows the same deployment pattern using Makefiles.

```makefile
make build    # Build the Docker image
make deploy   # Full deployment: build → push to ECR → deploy CloudFormation → restart service
```

### Deploying Squid Proxy to Staging

```bash
cd CloudFormation/squid-proxy

AWS_PROFILE=staging make deploy
```

This command performs the following steps in order:

1. **Build**: Compile the Docker image with the `allowed-staging.txt` configuration
   ```bash
   docker build -t justin/squid-proxy:latest \
     --build-arg ALLOWED_SITES=allowed-staging.txt .
   ```

2. **Create ECR Repository** (if it doesn't exist)
   ```bash
   aws ecr describe-repositories ... (via CloudFormation)
   ```

3. **Push to ECR**: Upload the built image to AWS Elastic Container Registry
   ```bash
   aws ecr get-login-password | docker login ...
   docker push <ecr-url>/justin/squid-proxy:latest
   ```

4. **Deploy CloudFormation Stack**: Create or update the ECS task definition, load balancer, and service
   ```bash
   aws cloudformation deploy --template-file squid-proxy.cf.yaml ...
   ```

5. **Force Restart Service**: Ensure the ECS service picks up the latest image
   ```bash
   aws ecs update-service --force-new-deployment ...
   ```

[Erik Andersson]: The force new deployment step is important. Occasionally, ECS does not automatically start the new image version when we push. By forcing a restart, we guarantee the latest version is running.

### Deployment File Structure

```
CloudFormation/squid-proxy/
├── Makefile                  # Deployment automation
├── Dockerfile                # Container definition
├── squid.conf                # Squid configuration (loaded from Docker)
├── allowed-staging.txt       # Staging whitelist
├── allowed-prod.txt          # Production whitelist
├── allowed-dr.txt            # Disaster recovery whitelist
└── squid-proxy.cf.yaml       # CloudFormation template (ECS task, service, load balancer)
```

When you run `AWS_PROFILE=staging make deploy`:
- The build process loads `allowed-staging.txt` as a build argument
- The Dockerfile embeds this file into the image
- Squid inside the container reads the whitelisted domains at startup

### Local Testing vs. Full Deployment

If you're making changes and only need to test locally (not push to production):

```bash
cd CloudFormation/squid-proxy

# Just build the image locally
AWS_PROFILE=staging make build

# Run the container and test
docker run -d -p 3128:3128 --name squid-proxy justin/squid-proxy:latest

# ... test your changes ...

# Do NOT run make deploy unless you want to push to production
```

Avoid `make deploy` during development because it triggers the full pipeline, which is unnecessary and time-consuming.

### Deploying to Different Environments

The same pattern works for all environments:

```bash
# Staging
AWS_PROFILE=staging make deploy

# Production
AWS_PROFILE=prod make deploy

# Disaster Recovery
AWS_PROFILE=dr make deploy
```

The `AWS_PROFILE` parameter determines which whitelist file is used and which AWS account receives the deployment.

### Adding Configuration for a New Environment

If you introduce a new environment (e.g., disaster recovery), you must:

1. Create a new allowed list file: `allowed-dr.txt`
2. Define all required endpoints for that environment:
   ```
   accounts.apsis-dr.cloud
   users.apsis-dr.cloud
   folder.apsis-dr.cloud
   ```
3. Update any CI/CD scripts to include the new profile
4. Deploy with: `AWS_PROFILE=dr make deploy`

[Erik Andersson]: This is especially important for disaster recovery setups. The disaster recovery environment has its own Apsis domain names (e.g., `apsis-dr.cloud` instead of `apsis.cloud`), and all of these must be whitelisted.

---

## Key Takeaways

1. **Squid is a mandatory security layer**: All outbound traffic from integration services passes through Squid proxy to enforce domain whitelisting. This prevents exfiltration attacks from injected code.

2. **Two-tier whitelisting model**:
   - **Static list** (`allowed-staging.txt`): Domains that are always needed (AWS services, Apsis internal services, OAuth providers)
   - **Dynamic lookup** (`allowed_domains` table): Per-customer CRM domains stored during installation

3. **Static list rarely changes**: Only updates needed when adding new OAuth integrations or changing Apsis infrastructure. This hasn't changed much in 6 years.

4. **Diagnosis via 403 Forbidden**: When you see 403 Forbidden errors related to external service calls, the first suspect is Squid proxy blocking an unlisted domain.

5. **The HTTPS_PROXY environment variable**: This is a standard Linux convention. Integration services use it automatically; no special code required.

6. **Deployment is consistent across services**: Build → ECR push → CloudFormation deploy → force service restart. The same pattern works for Squid and all other services.

7. **Domain format matters**: Always use bare domain names in whitelists (e.g., `example.com`), never full URLs (e.g., `https://example.com`). Squid works at the HTTP layer.

8. **Caching prevents database overload**: Squid caches domain lookup results for 5 minutes to avoid hammering the database on rapid requests to the same domain.

9. **Removing Squid is technically possible but not advisable**: Functionality would remain intact, but you lose an important security boundary. The maintenance burden is minimal (zero changes in 6 years).

10. **New integrations require whitelist updates**: If you add a new integration that requires reaching a third-party service (e.g., `login.salesforce.com`), remember to add it to the static whitelist. This is the most common source of 403 Forbidden errors when onboarding new integrations.

---

## Unresolved Questions and Action Items

- **Wildcard domain matching**: The team discussed whether `*.apsis.cloud` syntax works and questioned if `*.*.apsis.cloud` (matching multiple subdomain levels) is supported. [Erik Andersson] suggested it works like certificate matching (one level only), but suggested "we could try" to confirm the exact behavior.

- **Explicit error messaging**: [Michal Rosikiewicz] asked whether there is explicit error messaging that identifies Squid proxy as the blocker. Currently, there is none—developers must infer it from the 403 Forbidden response and context.

- **Follow-up session on lead creation**: The team ran out of time and deferred detailed discussion of lead creation and duplicate profile handling to a future session.