---
source_file: Erik - Squid proxy.txt
domain: Apsis One Integrations
topics: [Squid Proxy Architecture, Security Whitelisting, Broker Service, HTTP Proxy Configuration, Access Control Lists, External Access Control, Deployment Patterns, Credential Management]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Squid Proxy, Broker Service, ACL (Access Control List), HTTP Access Rules, External Database Sites, Allowed Domains, ECS Tasks, CloudFormation, Docker, ECR]
session_type: knowledge-transfer
subdomains: [Architecture, Outbound Flow]
---

## Session Overview

This knowledge-transfer session covers the **Squid proxy** and **Broker service**, two critical infrastructure components in the Apsis One Integrations platform that handle all outbound traffic from integration services. While developers rarely interact directly with these proxies in daily work, understanding their purpose and configuration is essential for troubleshooting connectivity issues, deploying new integrations, and disaster recovery scenarios. The session includes hands-on demonstration of the Squid proxy configuration, ACL (Access Control List) rules, and the whitelisting mechanism for customer domains.

---

## Architecture: Traffic Flow and Security Layers

### Overview of Service Architecture

All services within the integration platform route their outbound traffic through two sequential middleware components before reaching the internet or CRM systems:

1. **Broker Service** - Handles credential decryption and header injection
2. **Squid Proxy** - Enforces whitelist-based destination domain filtering

[Erik Andersson]: The flow works like this: different services in integration (integration manager, mappings manager, outbound worker, etc.) do not communicate directly with CRM systems. Instead, if they need to talk with a CRM system, they first go to the Broker service, then through the Squid proxy, and only then out to the internet.

### Why These Security Layers Exist

[Erik Andersson]: The whole reason we do this is because initially we never wanted any traffic to leave the VPC unless it goes to the internet. We also want to regulate where the traffic goes to—we don't want it to go to any unknown services. One approach could be using customer IP addresses in security groups, but that has a fundamental problem: customer IPs change. A customer might be installed at 1.1.1.1 today and 2.2.2.2 tomorrow. They could be on carrier-grade networks where their IP changes multiple times per day. You have no way of controlling this. What you can do instead is utilize the domain, because that's not going to change most likely. If it does, the customer would have to reinstall anyway.

[Erik Andersson]: The problem is that AWS cannot handle filtering on the HTTP level—only on the IP address at the TCP level. Therefore, we need to utilize a third party, which is the Squid proxy, to do this domain-level filtering.

---

## Broker Service: Credential Decryption and Authorization

### Core Responsibility

The Broker service is a simple, single-purpose middleware that sits between all integration services and external systems. Its only task is to decrypt customer credentials and inject them into request headers.

[Erik Andersson]: The Broker service receives a request, checks if there's a credential for this CRM system for this section in this account. If it has one, it will decrypt that secret and add it to the `X-API-Key` header, then pass the request along. It's a very simple service.

### Credential Storage and Access Control

A major architectural difference between the generic connector and legacy products is credential encryption:

- **Generic Connector**: Credentials are encrypted in the database using **KMS (Key Management Service)**. The encrypted value appears garbled in the database.
- **Legacy Products** (Lime, FSC Enterprise): Credentials are stored unencrypted, allowing developers to read them directly.

[Erik Andersson]: One major difference between the generic connector and the legacy product is that we don't store API keys in clear text in the database. Of course the database itself is encrypted, but as a developer, there's no need for us to be able to see and have direct access to the customer's API key. When they store it with the generic connector, it is encrypted in the database with KMS. When a service makes a request to the CRM system, the Broker service will decrypt the secret and add it to the header.

### Security Boundary

The Broker service is the **only service with access to the clear-text version of secrets**. Developers should not manually inspect decrypted values in logs or debugging sessions.

[Erik Andersson]: The Broker service is the only service which has access to this key. As developers, we don't have access to the clear text version of the secret, unless you manually go in and print the request. But you shouldn't do that.

### Database Interaction

[Lukasz Grabowski]: Does the Broker service communicate with the database?

[Erik Andersson]: Yes. The Broker service retrieves the hashed value from the database, decrypts it, adds that result to the header, and then passes the request on.

---

## Squid Proxy: Configuration and Access Control

### Infrastructure Location

The Squid proxy code and configuration reside in the integration repository under a specific path:

```
cloudformation/
└── squid-proxy/
```

This folder contains all infrastructure code that is either global for the platform or not tied to any specific service. The Squid proxy is a prerequisite for every other service to function fully.

### Configuration Paradigm: Lego Bricks

[Erik Andersson]: The configuration for the HTTP proxy works by defining your own set of Lego bricks and then you either allow or deny traffic based on combinations of these bricks.

The configuration uses two key terminologies:

#### Access Control Lists (ACL)

An ACL is a named rule that defines what traffic is eligible for filtering. Think of it as a "Lego brick."

**Example ACL definitions:**

```
acl lime_instance dst aftonbladet.se
acl safe_ports port 443 80
acl static_sites dstdomain "/etc/squid/allowed_staging.txt"
acl external_database_sites external "/usr/local/bin/squid_helper.py"
```

Breaking down the first example:
- **Name**: `lime_instance`
- **Type**: `dst` (destination domain)
- **Value**: `aftonbladet.se`

A more practical example:
- **Name**: `safe_ports`
- **Type**: `port`
- **Values**: `443` and `80`

#### HTTP Access Rules

HTTP access rules are the actual allow/deny statements that apply the ACLs:

```
http_access deny !safe_ports
http_access allow static_sites
http_access deny all
```

These rules are evaluated in order: explicit allows first, then explicit denies, then a final deny-all for anything not explicitly permitted.

### Static Whitelist: Pre-approved Domains

The static whitelist is maintained in a text file that is baked into the Docker image during build. This list contains domains that must always be reachable regardless of customer-specific installations:

**File location:**
```
allowed_staging.txt  (for staging environment)
allowed_prod.txt     (for production environment)
```

**Example entries from the static whitelist:**

```
*.amazonaws.com
*.apsis.cloud
accounts.apsis.cloud
users.apsis.cloud
folders.apsis.cloud
microsoft.com
```

[Erik Andersson]: This static list contains domains we always want to be able to reach—Amazon services, Apsis services, and Microsoft for Dynamics support. For example, `*.amazonaws.com` means anything like cloudwatch.amazonaws.com or secrets-manager.amazonaws.com is allowed.

### Critical Gotcha: Adding New Integrations

When a new external service is added to the integration platform, its domain must be added to the static whitelist, or all outbound requests to that service will fail with a 403 Forbidden error.

[Erik Andersson]: We have fallen into this trap a lot of times when we added new integrations—we forget to add the domain to the static list. Let's say you're asked to add Salesforce integration and need to make requests to `login.salesforce.com`. That request will fail with a 403. But once you add `login.salesforce.com` to the static list and redeploy, it works.

[Michal Rosikiewicz]: Do we have an explicit error saying that Squid proxy doesn't have this domain on the allowed list?

[Erik Andersson]: We don't. But you can very easily infer it because the error message is a 403 forbidden. Anytime you see "forbidden" in an error message, immediately think Squid proxy, because there's almost nothing else in integration that will say this forbidden error. The error is not handled in business logic—it comes from the HTTP transport layer and the HTTP client, so it's not an error we explicitly handle in code.

### Domain-Level Filtering Only

Squid operates at the domain level, not the protocol or path level:

[Erik Andersson]: The important thing to remember is that we're doing this on the domain level, so you never add the HTTPS, colon, slash slash. It's always just the pure domain that we work on.

**Correct format:**
```
aftonbladet.se
login.salesforce.com
```

**Incorrect format:**
```
https://aftonbladet.se  ❌
https://login.salesforce.com/  ❌
```

### Wildcard Behavior and Limitations

Wildcards in domain configuration have specific scoping rules:

[Michal Rosikiewicz]: Can we use `*.apsis.cloud` instead of adding multiple subdomains?

[Erik Andersson]: You can do that. It depends on how many levels you're going. I don't think you can do more than one level with the wildcard. If you have like `a.b.c.apsis.cloud`, then `*.apsis.cloud` would only match up to the `b` level, not the `a` level. It works similar to a certificate in that respect—only one level.

### Dynamic Whitelisting: Customer Installation Domains

Since it's impossible to maintain a static list of all customer CRM domains (they vary per installation), Squid uses an **external access control** mechanism to check customer domains dynamically.

#### How Dynamic Whitelisting Works

When a request arrives for a domain not in the static list, Squid pipes it to an external helper script:

[Erik Andersson]: First, Squid checks if the destination domain exists in the static list (Microsoft, AWS, Apsis one). If not, we have what's called an external access control—essentially a little Python or Bash script that runs alongside Squid. Anytime Squid gets a request that it doesn't find in the static list, it runs the request through this helper script.

The helper script performs a database lookup:

```
Table: allowed_domains
Lookup: Does this domain exist for this installation/section/account?
Return: Allow or Deny
```

[Erik Andersson]: When a customer installs, we add the URL or domain they enter in the API URL during installation to our database. The helper script does a lookup and checks: does this domain exist for this installation? If yes, the request is allowed. If no, it's blocked.

#### Database Caching for Performance

To avoid database queries on every request, Squid caches both hits and misses:

[Erik Andersson]: Squid has caching on this. If you have a lot of rapid requests to the same domain—for example, when we send events—we won't do a database lookup for every request because that would kill both the Squid proxy and the database. It has a TTL of 5 minutes. So either a hit or a miss, it will remember this for 5 minutes.

This caching is critical for performance when services send high volumes of events to customer CRM systems.

---

## Configuration File Details

### Allowed Staging Configuration

The configuration file is located at:
```
cloudformation/squid-proxy/allowed_staging.txt
```

Key principles for configuration:
- **Never changes** under normal circumstances
- Only modified when adding new integrations that use external OAuth flows
- Generic connector integrations don't require static whitelist additions because they only communicate with whitelisted CRM systems as part of the installation process

[Erik Andersson]: This static list normally never changes. You only modify it if you ever add a new integration which uses some kind of OAuth flow. But with the generic connector, no, because you will be interacting with CRM systems which are part of the installation process. We'll come to that very soon.

### Inbound vs. Outbound Traffic

The Squid proxy **only filters outgoing traffic** from integration services to external systems. It does not filter incoming traffic from CRM systems:

[Lukasz Grabowski]: Does the CRM communicate through Squid proxy back to us?

[Erik Andersson]: No. The CRM responds immediately to the request we initiated. When the CRM initiates contact with Apsis, it goes to the Delta Sync manager service. We don't have any filtering there for incoming data. For incoming data, we verify it using webhook hashes and secrets. Squid filtering is only for outgoing traffic.

---

## Practical Demonstration: Testing Squid Configuration

### Setup: Running Squid Locally

To test Squid proxy locally, you need to:

1. **Check out the branch** from the integration repository
2. **Navigate to the squid-proxy folder:**
   ```
   cd cloudformation/squid-proxy
   ```

3. **Build the image for a specific environment:**
   ```
   AWS_PROFILE=staging make build
   ```
   - The `AWS_PROFILE` environment variable determines which allowed list file to use (`allowed_staging.txt` or `allowed_prod.txt`)

4. **Run the container:**
   ```
   docker run -p 3128:3128 justin-squid-proxy:latest
   ```
   - The container listens on port 3128 for proxy requests

### Testing from Inside the Container

The test involves:
1. Starting the Squid proxy container
2. Opening a shell into the running container
3. Using curl to test traffic

**Prerequisites:**
- The container must have `curl` installed (it must be included in the Dockerfile)
- The proxy listens on `localhost:3128` inside the container

**Testing syntax:**

```bash
# Set up the proxy environment variable
export HTTPS_PROXY=localhost:3128

# Test a domain that's allowed in the static list (should return 200)
curl -I https://aftonbladet.se

# Test a domain that's not whitelisted (should return 403)
curl -I https://some-random-domain.com
```

[Lukasz Grabowski]: When I run the curl command, I get 403.

[Erik Andersson]: Correct. And if you look in the Squid proxy logs, you'll see:
```
TCP_DENIED/403 CONNECT aftonbladet.se:443
```

[Erik Andersson]: The request was denied because the domain isn't in the allowed list. To allow it, you need to add it to the `allowed_staging.txt` file, rebuild the image, and restart the container.

### Live Demonstration: Adding a Domain

**Step 1: Add domain to allowed list**

Edit `allowed_staging.txt` and add:
```
aftonbladet.se
```

**Step 2: Stop the running container**

```bash
docker stop <container-id>
```

**Step 3: Rebuild the image**

```bash
make build
```

**Step 4: Run the new container**

```bash
docker run -p 3128:3128 justin-squid-proxy:latest
```

**Step 5: Test again from inside the container**

```bash
curl -I https://aftonbladet.se
```

Expected result:
```
HTTP/1.1 200 Connection established
```

The Squid logs now show:
```
TCP_TUNNEL/200 CONNECT aftonbladet.se:443
```

### Understanding Log Entries

Squid logs show the request outcome and destination:

- **`TCP_DENIED/403`**: Request was blocked (domain not whitelisted)
- **`TCP_TUNNEL/200`**: Request was allowed (domain whitelisted)
- The log includes the method (CONNECT for HTTPS), destination domain, and port

---

## HTTP Proxy Environment Variable: The Standard Mechanism

All integration services use a standard Linux/Unix mechanism to route traffic through Squid:

```bash
HTTPS_PROXY=localhost:3128
HTTP_PROXY=localhost:3128
```

[Erik Andersson]: This is a standard protocol used by everything in Linux. All HTTP transport libraries and services have to respect this. In integration, all ECS tasks have an environment variable `HTTPS_PROXY` that points to our Squid proxy. This is nothing Golang-specific—it's pure Linux environment variables. The Golang HTTP client will see this environment variable and route all requests through the HTTPS proxy.

This mechanism is language-agnostic and works with any HTTP client that respects the standard proxy variables.

---

## Deployment and Configuration Management

### Configuration Strategy by Environment

The Squid proxy deployment follows a pattern where environment-specific configuration is selected at build time:

```
AWS_PROFILE=staging make build  # Uses allowed_staging.txt
AWS_PROFILE=prod make build     # Uses allowed_prod.txt
```

If you need to deploy for a disaster recovery environment, you would need to create a new configuration file:

```
allowed_disaster_recovery.txt
```

Then deploy with:
```
AWS_PROFILE=disaster-recovery make build
```

[Erik Andersson]: If you introduce disaster recovery, you will need to add a new configuration file that matches the AWS profile you have for it. This specifies the corresponding endpoints—for example, `disaster_recovery_accounts`, `disaster_recovery_users`, `disaster_recovery_folders`, etc.

### Build and Deploy Process

The deployment process uses Make targets and follows this sequence:

```makefile
make build       # Build the Docker image with AWS_PROFILE selection
docker push      # Push image to ECR
deploy           # Deploy CloudFormation stack containing:
                 #  - ECS service definition
                 #  - Load balancer configuration
                 #  - Task definition
docker force restart  # Force new deployment to pick up latest image
```

**To deploy only Squid proxy to staging:**

```bash
cd cloudformation/squid-proxy
AWS_PROFILE=staging make deploy
```

[Erik Andersson]: When you run `make deploy`, it first runs the build, then creates the ECR repo (ensuring it exists), logs into ECR, pushes the image, deploys the CloudFormation stack (which contains the ECS task and service), and finally forces a new deployment to ensure the service picks up the latest image.

### Why Force Restart is Necessary

[Erik Andersson]: Sometimes when you deploy a new image, the service doesn't start with the latest version. We have the force restart step just to really make sure that it is the latest version that is running.

### Build Optimization: Make Dependency Tracking

The Makefile uses Make's built-in dependency tracking to avoid redundant builds:

```makefile
docker_push: build
    # Push step depends on build
    # If build has already run in this Make invocation, it won't run again
```

[Erik Andersson]: Make only executes each target once. So if you have `docker_push` depends on `build`, and `build` has already been executed, the `build` step won't run again even if you reference it multiple times.

**This is important for local testing:**

```bash
# For local testing, just build—don't do full deploy
make build

# Then run locally for testing
docker run -p 3128:3128 justin-squid-proxy:latest
```

---

## Integration Across All Services

Every service in the integration platform follows the same deployment and proxy pattern. This consistency means:

1. **All services route through Squid** - Every outbound request goes through the proxy via the `HTTPS_PROXY` environment variable
2. **All services follow the same deploy pattern** - Build, push to ECR, deploy CloudFormation, restart
3. **All services require Squid to function** - Without it, they cannot reach CRM systems

[Erik Andersson]: Every service in integration follows this pattern where you have the functionality to build the image, push the repo, push the CloudFormation file, and then force restart the service.

---

## Maintenance and Operational Considerations

### Zero Maintenance for Six Years

[Erik Andersson]: The good thing is that the Squid proxy requires essentially no maintenance. The code for the check hasn't changed for six years, nor has the static list. If anything, you might need to add things to the static list if you add a new OAuth server that you need to utilize for any new integration.

### When Configuration Changes Occur

Configuration updates happen only in these scenarios:

1. **Adding a new integration** that requires a new external OAuth provider
   - Add domain to `allowed_staging.txt` and/or `allowed_prod.txt`
   - Redeploy with `AWS_PROFILE=staging make deploy`

2. **Adding a disaster recovery environment**
   - Create `allowed_disaster_recovery.txt`
   - Add corresponding Apsis service endpoints (accounts, users, folders, etc.)

3. **Updating internal Apsis service endpoints**
   - Modify the static list if an internal service domain changes (very rare)

### Removing Squid Proxy (Hypothetical)

If Squid were to be removed:

1. **Delete the infrastructure** - Remove the CloudFormation stack and ECS service
2. **Remove environment variables** - Delete `HTTPS_PROXY` and `HTTP_PROXY` from all service task definitions
3. **Functionality remains intact** - All integration features would continue to work, just without domain-level filtering

[Erik Andersson]: In theory, you could remove this and the functionality of integration would remain completely intact. Nothing would happen until someone manages to inject something in your code and manages to transport data out.

However, this would remove a critical security layer and is not recommended.

---

## Key Takeaways

1. **Squid Proxy is a Security Layer**: It prevents exfiltration of data if malicious code is injected into services by restricting outbound traffic to only whitelisted domains.

2. **Two Types of Whitelisting**:
   - **Static list** (`allowed_staging.txt`): Pre-approved domains like AWS, Apsis services, and Microsoft—never changes under normal circumstances
   - **Dynamic list** (database lookup): Customer-specific CRM domains verified at installation time

3. **ACLs are Lego Bricks**: Squid configuration works by defining named access control rules (ACLs) and then allowing/denying traffic based on combinations of these rules.

4. **External Access Control**: A helper script performs database lookups for customer domains, with 5-minute caching to avoid performance impact.

5. **403 Forbidden = Squid Denial**: When you see a 403 forbidden error, the first thing to check is whether the destination domain is in the Squid whitelist.

6. **Domain-Only Filtering**: Squid filters at the domain level (e.g., `aftonbladet.se`), not at the protocol or path level. Never include `https://` in domain entries.

7. **Outbound Only**: Squid filters only outgoing traffic from integration services. Incoming CRM webhooks use different security mechanisms (webhook hashes and secrets).

8. **No Maintenance Required**: Squid requires essentially no maintenance. Configuration changes are rare and only occur when adding new integrations with new external OAuth providers.

9. **Broker Service Handles Credentials**: The Broker service decrypts customer credentials (encrypted with KMS in the database) and injects them into request headers. Only the Broker service has access to clear-text secrets.

10. **Standard Deployment Pattern**: All services in integration follow the same pattern: `AWS_PROFILE=<env> make build`, push to ECR, deploy CloudFormation, force restart.

---

## Unresolved Questions and Action Items

1. **Wildcard behavior confirmation**: The behavior of wildcard domain matching (e.g., whether `*.apsis.cloud` matches multi-level subdomains) was discussed but not fully tested in this session.

2. **Docker image caching**: During the demonstration, some participants experienced Docker image caching issues on first pull from ECR—this appears to be environment-specific and may require AWS credential configuration.

3. **Curl in Alpine image**: The alpine-based Squid image may not include `curl` by default; this should be verified or documented if curl is needed for testing.

4. **Next session**: Lead creation process was deferred to a future session. This should be scheduled when participants have capacity.