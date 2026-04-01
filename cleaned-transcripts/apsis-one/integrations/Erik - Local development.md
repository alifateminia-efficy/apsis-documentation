---
source_file: Erik - Local development.txt
domain: Apsis One Integrations
topics: [local development setup, reverse proxy architecture, Docker Compose infrastructure, debugging with breakpoints, local service orchestration, worker service limitations, mock service, SQS local emulation]
speakers: ["Erik Andersson (Integration Platform Engineer)", "Lukasz Grabowski", "Michal Rosikiewicz", "Tomasz Kowalski"]
key_components: [integration platform, reverse proxy, squid proxy, Docker Compose, LocalStack, Kafka, Redis, Postgres, SQS, mock service, Visual Studio Code debugger, launch.json, main.go, audience service, accounts/users service, delegation service]
session_type: knowledge-transfer
---

# Local Development Setup — Apsis One Integrations

## Session Overview

Erik Andersson walked the team through the local development setup for the Apsis One integration platform. The session covered the architectural challenge of IP whitelisting with external services (particularly the Audience service), how a reverse proxy inside the integration environment is used to solve this, how to spin up local infrastructure via Docker Compose, how to launch and debug integration manager services locally in VS Code, and the current limitation that async worker services cannot yet be run locally. A mock service for simulating CRM systems was also introduced. The session concluded with an action item to cover the squid proxy and internal traffic routing in a follow-up session.

---

## IP Whitelisting Problem and Reverse Proxy Solution

### The Core Problem

All integration platform services normally run in AVS (AWS) within a private network. When services in the integration platform communicate with external/partner services — specifically the **Audience service**, **Accounts and Users service**, and the **Delegation service** — those calls originate from the integration environment's IP addresses, which are **whitelisted** by those services (particularly Audience).

> "If you try to call Audience from your local machine, they will reject that."

This creates a problem for local development: requests originating from a developer's laptop will be rejected by Audience and other services.

### The Solution: A Reverse Proxy in the Integration Environment

To solve this, a **reverse proxy** has been deployed inside the integration environment (an EC2 instance). When running locally:

- Requests to **our own microservices** stay on the developer's local machine (we control those Docker containers).
- Requests destined for **external/partner services** (Audience, Accounts/Users, Delegation) are routed through this reverse proxy.
- Because the proxy resides inside the integration environment, **its IP is whitelisted**, so those calls succeed.

The proxy performs simple URL pattern matching: it reads the destination URL and redirects the request to the correct endpoint based on that pattern.

### Security Considerations

[Lukasz Grabowski]: Asked how the proxy is secured, given it is publicly accessible.

[Erik Andersson]: The reverse proxy itself has **no authentication layer on it**, but this is acceptable because:
- The proxy only forwards requests.
- All forwarded requests must still carry valid **Apsis One credentials** (a token or delegation token) to be accepted by the downstream services (e.g., Audience).
- Without valid credentials, requests will still be rejected by the target service.

> "There is no real security issue in itself because the reverse proxy just forwards the request. But if you don't have any Apsis One credential, you will still be rejected."

The proxy is a plain EC2 instance — there is no API gateway in front of it.

---

## Local Infrastructure: Docker Compose Setup

A `docker-compose.yml` file in the **root folder** of the integration platform repository spins up all required local infrastructure. Run with:

```bash
docker compose up
```

### Services Started by Docker Compose

| Service | Purpose |
|---|---|
| **PostgreSQL** | Main relational database |
| **Redis** | Local Redis instance |
| **LocalStack** | Local emulation of AWS services |
| **Kafka** | Local message broker |
| **Zookeeper** | Required by Kafka (exact reason not recalled by Erik) |
| **Proxy** | A local proxy (details to be covered in a follow-up session — see Unresolved Questions) |

### Database Initialization

The Postgres service references two initialization scripts:

1. **`create_schema`** — Creates all tables used by the integration platform.
2. A second script — Populates **static data** required by the services, particularly different status values used in the syncing process.

> ⚠️ **Note:** Docker Compose only starts the required infrastructure — the integration platform services themselves are **not** started at this point.

Michal and Tomasz confirmed Docker Compose runs successfully on their local machines. Michal noted some issues with tests, but not with Docker itself.

---

## Starting Integration Manager Services Locally

### Launch Script and `main.go`

The integration services are started via VS Code's debug launcher. The entry point is:

```
app/local/managers/main.go
```

Navigating to this file in VS Code and launching with the **"Launch Package"** configuration (defined in `launch.json`) starts the debug session. The `launch.json` configuration:
- Attaches environment variables to the session.
- Determines which program to run based on the file selected when the debug session starts — selecting `main.go` in `app/local/managers/` means the local development launcher is used.

### What `main.go` Does

This is described as a **"go launch container"** — a setup script that:
- Creates an AVS session.
- Retrieves the integration config.
- Sets up repositories for the installation.
- Retrieves and sets up Audience clients.
- Initializes all other clients used to call external services.
- Starts each **manager microservice** on a specific, designated port (e.g., ports in the `4979`–`4999` range were shown).

### Which Services Can Be Run Locally (Current State)

**Can run locally:**
- Integration Manager
- Mappings Manager
- Delta Sync Manager
- Outbound Manager
- All other manager services (request-handling, synchronous)
- Mock Service (see below)

**Cannot yet run locally:**
- Async worker services

---

## Connecting the Frontend to Local Services

### Redirecting Frontend Traffic

To connect the Apsis frontend to local manager services instead of the staging integration environment:

1. Add a hosts file entry mapping `dev.apsis` to `localhost`:
   - File: `/etc/hosts` (environment-dependent)
   - `dev.apsis` appears to be a convention used in other parts of the Apsis platform as well.

2. In the **Apsis frontend repository**, edit the environment base configuration:
   ```
   src/Environments/models/environment.base
   ```
   This is described as the **central configuration file for all environments in Apsis**.

3. Change the `integrationBaseUrl` to point to `dev.apsis` (i.e., `localhost`).

With this in place, starting the frontend will direct all integration service requests to the local machine, while all other requests (accounts, audience, etc.) continue to use the staging environment via the reverse proxy.

### Result

All API calls from the frontend to the integration manager will hit the locally running services. VS Code will show live request logs and honor any breakpoints set in the Go code.

---

## Debugging with Breakpoints

Once the local services are running and the frontend is pointed at `dev.apsis`, full breakpoint debugging is available for any manager service.

### Example Demonstrated: Install WooCommerce

Erik demonstrated setting a breakpoint in the `install` function of the Integration Manager. When an installation was triggered from the frontend:
- Execution paused at the breakpoint.
- The VS Code debug variables panel showed the full request context: account ID, section, integration type, JWT token, installation body, etc.
- Stepping through revealed exactly what data was retrieved from the database and what was returned.

### WooCommerce Installation — Notable Behavior

WooCommerce is a special case: the WooCommerce plugin is **not developed by Apsis**. It works by the third-party plugin calling the **Apsis One API** directly. Therefore, during installation:
- A **One API key** (client ID + client secret) is generated and displayed clearly in the UI.
- A **key space** is set up on the CRM attribute.
- This One API key generation is done by **calling the Accounts/Users service via the reverse proxy**.
- There are **no field mappings, subscription mappings, full syncs, webhook registrations, or remote detail injection** — WooCommerce does not utilize the integration platform beyond the initial key/keyspace setup.

---

## Mock Service

The **mock service** is a local development tool that implements the **generic connector endpoints**, effectively acting as a CRM system.

**Use cases:**
- Verify that the integration platform sends the correct data to a CRM system.
- Verify that the integration platform correctly handles responses from a CRM system.
- **Simulate bad/malformed data** from a CRM system to test error handling.
- Simulate partial or full generic connector support to test conditional logic.

The mock service is fully configurable — you control what it accepts and what it responds with.

---

## Worker Services: Current Limitation and Workaround

### Why Workers Can't Run Locally Yet

The work to enable local worker debugging was started by a former team member (Adam, no longer at the company) and was **never completed**. The blocking issue was getting a **local SQS queue** working:

- Workers need to be connected to an SQS queue (AWS).
- To run locally, you would need a local SQS emulator (Erik notes that LocalStack may now have a fully working SQS emulation — worth investigating).
- Local manager services would need to **publish messages to the local SQS queue**.
- Local worker services would need to **consume from that local queue**.

> "The principle really is the same. You can in theory essentially copy-paste the solution we have for the managers and just run the worker service instead. The problem is that every worker needs to be connected to a queue and he never managed to get the local queue system fully working."

### Current Workaround for Worker Development/Debugging

> "Usually add a metric load of logs to the flow and deploy to staging and see what happens."

**Downsides of this approach:**
1. **Slow iteration:** Each deployment requires deregistering the old service, deploying the new one, etc. — approximately **3 minutes per deployment cycle**.
2. **Staging pollution:** Staging is shared with other teams for testing. Breaking staging can impact others. However, Erik notes that very few other teams depend on the integration environment directly — mainly calls to the Integration Manager (list integrations) and Outbound Manager (event registration), so in practice this is rarely a serious problem.

### Rule of Thumb

> "If you're developing something for a manager, try and do it locally. If it's a worker for now, add extra logs and deploy to staging."

---

## Key Takeaways

1. **IP whitelisting** by Audience and other services means local machines cannot call those services directly. A **reverse proxy on an EC2 instance inside the integration environment** bridges this gap.
2. The reverse proxy is **publicly accessible but relies on downstream service authentication** (Apsis One tokens) for security.
3. **Docker Compose** in the repo root starts all local infrastructure (Postgres, Redis, Kafka, LocalStack, local proxy). Run `docker compose up` before starting any services.
4. **Integration manager services** are started via VS Code debug launcher targeting `app/local/managers/main.go` with the `launch.json` "Launch Package" config.
5. The frontend can be pointed at local services by editing `src/Environments/models/environment.base` and adding a `dev.apsis` → `localhost` hosts file entry.
6. **Full breakpoint debugging** is available for all manager services; inspect live request data, DB queries, and responses.
7. **Worker services cannot yet run locally** due to incomplete SQS local queue integration. Workaround is log-heavy staging deploys (~3 min/cycle).
8. **Mock service** simulates a CRM system for testing integration platform behavior end-to-end without a real CRM.
9. The local proxy routing configuration (squid proxy) was **not yet covered** in this session and needs a follow-up.

---

## Unresolved Questions and Action Items

| # | Item | Owner | Notes |
|---|---|---|---|
| 1 | **Squid proxy deep-dive** — How is internal traffic routed to the correct Docker container? How does the squid proxy config work? | Erik (to present), full team | Erik explicitly flagged this as needing its own session. Currently "black magic" for the new team members. |
| 2 | **Worker local development** — Investigate whether LocalStack's current SQS emulation is sufficient to complete Adam's unfinished work and enable full local debugging of worker services. | TBD | Erik believes LocalStack may now support this fully. Would require: local SQS emulator + managers publishing to local queue + workers consuming from local queue. |
| 3 | **Homework for Michal and Tomasz** — Run the local environment as demonstrated: Docker Compose up → start `app/local/managers/main.go` → point frontend at `dev.apsis` → set a breakpoint and step through a request. | Michal Rosikiewicz, Tomasz Kowalski | |
| 4 | **Ongoing support channel** — Team agreed to use a shared channel (similar to how the Audience team operates) for questions, errors, and paste-and-ask debugging. Erik is currently dedicated to supporting this team. | All | Erik noted he is available unless a PP123 incident or CRM team request takes priority. |