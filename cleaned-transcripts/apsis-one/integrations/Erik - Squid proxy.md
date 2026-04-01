---
source_file: Erik - Squid proxy.txt
domain: Apsis One Integrations
topics: [Squid Proxy configuration, ACL rules, whitelist management, broker service, credential encryption, outbound traffic filtering, deployment process, ECR/ECS deployment, local development and testing]
speakers: ["Erik Andersson (presenter/senior engineer)", "Lukasz Grabowski", "Michal Rosikiewicz", "Tomasz Kowalski"]
key_components: [Squid Proxy, Broker Service, KMS encryption, ECS tasks, ECR, AWS CloudFormation, allowed-domains database table, squid-helper script, HTTPS_PROXY environment variable]
session_type: knowledge-transfer
---

## Session Overview

This session covers the two outbound traffic proxies used across all services in the Apsis One integration platform: the **Broker Service** and the **Squid Proxy**. Erik Andersson explains the architecture, rationale, and configuration of both, with the Squid Proxy as the primary focus. The session includes a hands-on demo of running Squid locally, testing domain whitelisting, and observing allow/deny behavior in the logs. The final portion covers the deployment pipeline for the Squid Proxy and how it generalizes to all integration services.

---

## Architecture Overview: The Two Outbound Proxies

Every service in the integration platform (Integration Manager, Mappings Manager, Outbound Worker, etc.) routes outbound traffic through two proxies in sequence before reaching the CRM system:

```
[Integration Services] → [Broker Service] → [Squid Proxy] → [Internet / CRM]
```

This applies only to **outbound traffic**. Inbound traffic (e.g., CRM-initiated webhook calls) goes directly to the Delta Sync Manager service and is validated via webhook hash/secrets — the Squid Proxy is not involved.

---

## Broker Service: Credential Decryption and Header Injection

The **Broker Service** is the first stop for outbound requests. It is not a pure proxy — it contains business logic — but its sole job is:

1. Receive the outbound request.
2. Look up the encrypted credential for the relevant CRM system (matched by section and account) in the database.
3. Decrypt the secret using **KMS (Key Management Service)**.
4. Inject the decrypted value into the `X-API-Key` header.
5. Pass the request on to the Squid Proxy.

### Why KMS Encryption for Credentials

> "We as developers, there is no need for us to be able to see and have direct access to the customer's API key."

With the **Generic Connector**, credentials are encrypted at rest in the database using KMS. The stored value is garbled/opaque. The Broker Service is the **only service with access to the decrypted secret**.

**Important caveat:** For legacy connectors (Lime, FSC Enterprise), credentials are **not** encrypted with KMS and remain readable by developers. This is a known distinction between legacy and generic connector.

[Erik Andersson]: "Unless you manually go in and print the request, you shouldn't do that."

---

## Squid Proxy: Purpose and Security Rationale

The **Squid Proxy** enforces domain-level outbound traffic whitelisting. It sits after the Broker Service and before the public internet.

### Why Squid Proxy Exists

The primary motivation is **security containment** — specifically preventing data exfiltration in the event of a code injection attack (e.g., SQL injection):

> "You will not be able to get data out from the system by sending it to some random server."

### Why Not Use AWS Security Groups Instead

AWS security groups can only filter at the **TCP/IP layer** (IP addresses). The problem with using IP addresses for customer CRM systems:

- Customer IP addresses can change (e.g., carrier-grade NAT, dynamic IPs).
- There is no guarantee that an IP valid today will be valid tomorrow.

**Domain names** are used instead, as they are stable — if a customer's domain changes, they would need to reinstall anyway.

> "AWS cannot handle filtering on the HTTP level, only on the IP address at the TCP level. So therefore you need to utilize some kind of third party for this."

### Functional Impact

Removing the Squid Proxy would not break any integration functionality under normal conditions. All integration services would continue to work. The proxy is purely a **security layer** — it only matters if malicious code is injected and attempts to exfiltrate data to an unauthorized host.

---

## Squid Proxy Configuration: File Location and Structure

```
cloudformation/
└── squid-proxy/
    ├── docker/
    │   └── squid.conf
    ├── allowed_staging.txt
    └── allowed_prod.txt
```

Everything under `cloudformation/` is infrastructure that is either not tied to a specific service or is global to the whole platform.

### ACL (Access Control Lists) — The "Lego Brick" Model

Configuration is built by defining **ACLs** (access controls) and then writing **HTTP access** rules that reference them.

[Erik Andersson]: "Think of the ACLs as Lego bricks. You define your bricks and then you either allow or deny traffic based on combinations of those bricks."

**ACL syntax:**
```
acl <name> <type> <value(s)>
```

Example — a named domain ACL:
```
acl lime_instance dstdomain .lime.example.com
```

Example — a port ACL:
```
acl safe_ports port 443 80
```

**HTTP access rule syntax:**
```
http_access deny !safe_ports
```
This denies any traffic that is **not** going to port 443 or 80. Integration enforces **HTTPS only** (port 443) for all outbound traffic.

### Important Behavior Note on Wildcard Domains

The leading dot (`.`) in a `dstdomain` ACL entry acts as a wildcard for one subdomain level — similar to how TLS certificates work. It does **not** match multiple levels deep.

[Erik Andersson on why multiple Apsis subdomains are listed separately rather than one wildcard]: "I think this only applies to one level, so if you have `a.b.c`, doing `.b.c` would not match `a.b.c`. It works like a certificate in that aspect."

⚠️ **This behavior was discussed but not fully confirmed during the session** — Erik said "we could try" to use a single wildcard entry. Treat this as a likely-but-unverified limitation.

---

## Static Whitelist: `allowed_staging.txt` / `allowed_prod.txt`

The static file defines domains that must **always** be reachable regardless of what customer installations exist. It is loaded at build time based on the `AWS_PROFILE` environment variable.

Contents include:
- All AWS services: `*.amazonaws.com` (CloudWatch, Secrets Manager, etc.)
- All Apsis services: e.g., `folder.staging.apsis.cloud`, `accounts`, `users`
- Microsoft endpoints (to support Dynamics connections)
- Salesforce endpoints

### Common Gotcha: Forgetting to Add New OAuth Endpoints

> "We have fallen in this trap a lot of times when we added new integrations — we forget to add it here to the static list."

If a new integration requires an OAuth flow against a third-party auth server, that server's domain **must** be added to the static list.

Example that was given:
```
login.salesforce.com
```

This static list **essentially never changes** in normal operation. It only needs updating when:
1. A new integration is added that uses an OAuth flow against a new domain.
2. A new environment is set up (e.g., disaster recovery — see below).

---

## Dynamic Whitelist: Customer Domain Whitelisting via External ACL

Customer CRM domains cannot be in a static file — they are dynamic (set per-installation). Squid handles this via an **External ACL helper**.

### How It Works

Squid defines an external ACL that pipes unrecognized requests to a helper script:

```
acl external_database_sites external <helper-script>
```

The helper script (found in the `squid-helper` folder) is a small standalone program (Python, Bash, or Go). Its sole responsibility:

1. Receive the destination domain from Squid.
2. Query the `allowed_domains` database table.
3. Check: does this domain exist for this specific section/account/installation?
4. Return allow or deny.

### During Installation

When a customer installs an integration, the **API URL / domain they enter** is written to the `allowed_domains` table in the database. That domain is then recognized by the helper script on subsequent requests.

Example: If a customer installs with `ericsdomain.com`, that domain is written to `allowed_domains`. Squid will call the helper, the helper finds the entry, and the request is allowed.

### Caching (TTL = 5 minutes)

Squid caches the helper's response for **5 minutes** to prevent a database lookup on every request (which would be catastrophic during high-frequency event sending).

> "If it gets a hit or a miss, it will remember this for 5 minutes."

---

## Diagnosing Squid Proxy Blocks: The 403 Forbidden Signal

When a request is blocked by the Squid Proxy, the error returned is **`403 Forbidden`**.

[Erik Andersson]: "Anytime you see 'forbidden' in the error message, immediately think Squid Proxy, because there is almost nothing else in integration that will produce a forbidden error."

This error is **not handled in application business logic** — it is returned at the transport layer by the HTTP client. Therefore, there is no explicit "blocked by Squid" error message in the service logs; the 403 is the signal.

The Squid Proxy's own logs will show:
- `TCP_DENIED/403` — request blocked
- `TCP_TUNNEL/200` — request allowed through

---

## HTTPS_PROXY Environment Variable: How Services Route Through Squid

All ECS tasks in integration have an environment variable:

```
HTTPS_PROXY=<squid-proxy-address>
```

This is a **standard Linux/HTTP convention** — it is not custom Golang code. Any HTTP client library on Linux (including the Go `net/http` client) respects this environment variable automatically and routes all outbound requests through the specified proxy.

> "This is nothing in Golang — this is just a standard environment variable. The Golang HTTP client will see this environment variable and therefore send all requests through the HTTPS proxy."

### Disabling the Squid Proxy

To disable the Squid Proxy:
1. Delete the Squid Proxy service itself.
2. Remove the `HTTP_PROXY` and `HTTPS_PROXY` environment variables from all ECS container definitions.

All services will then make direct outbound connections. This can be confirmed by searching for `HTTPS_PROXY` in the codebase — it appears in the container definitions for all integration services.

---

## Local Development and Testing the Squid Proxy

### Prerequisites

- Be in the `cloudformation/squid-proxy/` directory (running `make build` from elsewhere will fail with "no rule to make target").
- Have AWS credentials configured for the target environment (needed to pull the base image from ECR).

### Build and Run Locally

```bash
# Build the image with the staging allowed-sites list
AWS_PROFILE=staging make build

# Run the container
docker run justin-squid-proxy
```

The `AWS_PROFILE` value determines which allowed-sites file is loaded:
- `AWS_PROFILE=staging` → loads `allowed_staging.txt`
- `AWS_PROFILE=prod` → loads `allowed_prod.txt`

If you name your profile `integration-staging` instead of `staging`, you must also rename the file to `allowed_integration-staging.txt`.

### Testing a Request Through the Proxy

Once the container is running, get its ID via `docker ps`, then open a shell inside it:

```bash
docker exec -it <container_id> /bin/sh
```

From inside the container, test with curl using the proxy:

```bash
HTTPS_PROXY=localhost curl https://<domain>
```

**Result if domain is NOT whitelisted:** `403 Forbidden` + `TCP_DENIED/403` in Squid logs.

**Result if domain IS whitelisted:** `HTTP 200 Connection Established` + `TCP_TUNNEL/200` in Squid logs.

### Adding a Domain to the Static Whitelist for Testing

Edit `allowed_staging.txt`, add the domain (no `https://` prefix — domain only):

```
aftonbladet.se
```

Then stop the container, rebuild, and rerun. Make is cached, so unchanged layers rebuild quickly.

> "Always just the pure domain — never add `https://`. We work on the domain level."

---

## Deployment: Squid Proxy and All Integration Services

### Deploying Only the Squid Proxy

From `cloudformation/squid-proxy/`:

```bash
AWS_PROFILE=staging make deploy
```

### What `make deploy` Does (in order)

1. **`make build`** — builds the Docker image with the correct allowed-sites file.
2. **Deploy ECR repo** — ensures the ECR repository exists (idempotent).
3. **`docker push`** — logs into ECR and pushes the newly built image. (Note: `make` deduplicates — if `build` was already run as a dependency, it is only executed once.)
4. **Deploy ECS task + service** — runs the CloudFormation stack for the Squid Proxy, which includes the container definition, service definition, and load balancer.
5. **Force new deployment** — explicitly restarts the ECS service to ensure the latest image version is picked up.

[Erik Andersson on the force restart step]: "I've seen that you deploy a new image but the image didn't start with the latest version. The force new deployment manually restarts the service, which will pick up the latest version."

### Local Build vs. Full Deploy

- `make build` alone: sufficient for local testing. Builds the image locally, does not push anything to AWS.
- `make deploy`: full pipeline — builds, pushes, deploys CloudFormation, force-restarts service. Use this for actual deployments.

> "If you want to test something locally, you only need `make build`. You don't need `make deploy` — that would be unnecessary execution time."

### Pattern Applies to All Integration Services

This same `make deploy` pipeline pattern — build image → deploy ECR repo → push image → deploy CloudFormation → force-restart — is used by **every service in integration**, not just the Squid Proxy. The Makefile structure is consistent across all services.

---

## Disaster Recovery / New Environment Considerations

If a new environment is introduced (e.g., a disaster recovery environment), you must:

1. Create a new allowed-sites file matching the new AWS profile name (e.g., `allowed_disaster-recovery.txt`).
2. Populate it with the Apsis service endpoints for that environment (e.g., DR-specific accounts, users, folder service URLs).

[Erik Andersson]: "You would need something like `disaster-recovery-accounts-and-users` and `disaster-recovery-folder`."

---

## Key Takeaways

1. **All outbound traffic** from integration services flows through the Broker Service (credential decryption) → Squid Proxy (domain whitelisting) → Internet. This is enforced via the standard `HTTPS_PROXY` environment variable on all ECS tasks.

2. **Squid Proxy is a security layer**, not a functional requirement. Removing it would not break integrations, but would eliminate the defense against data exfiltration via code injection.

3. **403 Forbidden = Squid Proxy block.** This is the primary diagnostic signal. Check the Squid logs for `TCP_DENIED/403` to confirm.

4. **Two types of whitelists:** static (file-based, service-level, rarely changes) and dynamic (database-backed via helper script, per-customer-installation).

5. **When adding a new integration with OAuth**, remember to add the OAuth server domain to the static allowed-sites file — this is a historically common oversight.

6. **Domain entries never include the URL scheme** (`https://`). Always use bare domain format in the whitelist files.

7. **Wildcard domain matching is one level only** (behavior analogous to TLS cert wildcards). Verify this if attempting to consolidate Apsis subdomain entries.

8. **KMS-encrypted credentials** (Generic Connector only) mean developers cannot read customer secrets even with database access. The Broker Service is the sole decryption point.

9. **Deployment pattern is universal** across all integration services: build → ECR push → CloudFormation deploy → force restart.

---

## Unresolved Questions / Action Items

- **⚠️ Wildcard domain depth in Squid `dstdomain` ACL:** It was discussed but not confirmed whether `.apsis.cloud` would match `a.b.apsis.cloud`. Erik suggested it works like a TLS certificate (one level only) but said "we could try." This should be verified before attempting to consolidate the static list.

- **Follow-up KT session on "leads"** was referenced but not scheduled. Lukasz to schedule a suitable time with Erik.

- **Tomasz Kowalski** had Docker issues during the demo (Docker update conflict) and may not have completed the hands-on portion. Follow-up if needed.