---
source_file: Erik - Local development.txt
domain: Apsis One Integrations
topics: [Local Development Setup, Reverse Proxy Architecture, Docker Compose Infrastructure, Debugging Manager Services, Frontend Integration, Development Workflows]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Reverse Proxy, Docker Compose, Postgres, Redis, Kafka, LocalStack, AWS SQS, Integration Manager, Mappings Manager, Delta Sync Manager, Outbound Manager, Mock Service, Visual Studio Code Debug]
session_type: knowledge-transfer
subdomains: [Architecture]
---

## Session Overview

This session covers the complete local development setup for the Apsis One Integrations platform. Erik Andersson walked through the architectural challenge of IP whitelisting in the staging environment, the reverse proxy solution that enables local development, the Docker Compose infrastructure stack, and practical debugging workflows for manager services. The session included a live demonstration of running the integration platform locally with Visual Studio Code breakpoints and connecting the frontend to locally running services.

---

## Architectural Challenge: IP Whitelisting and Local Development

### The Whitelisting Problem

[Erik Andersson]: In the integration platform, everything normally runs in AWS within the private network. External services like Audience, Accounts & Users, and the Delegation Service have IP whitelists that only allow requests from integration platform IP addresses. When developers try to call these services directly from their local machines, they are rejected because local IPs are not whitelisted.

### The Reverse Proxy Solution

To solve this, the team implemented a **reverse proxy** that acts as an intermediary between local development environments and stage services:

- Local microservices run in Docker on the developer's machine
- Requests destined for external services (Audience, Accounts & Users, Delegation Service) are routed through the reverse proxy instead
- The reverse proxy resides within the integration environment in AWS, so its IP is whitelisted
- The proxy performs simple URL pattern matching and forwards requests to the correct destination endpoints

[Lukasz Grabowski]: The reverse proxy is publicly accessible from outside the network, which is necessary for local machines to reach it. However, security is maintained because:
- The proxy itself doesn't authenticate requests
- Apsis One credentials (tokens) are passed through the proxy and validated by the destination services
- Without valid Apsis One credentials, requests are rejected downstream

> [Erik Andersson]: "The reverse proxy is publicly accessible because we need to be able to reach it from outside. But there is no real security issue because the reverse proxy just forwards the request. If you don't have any Apsis One credential, you will still be rejected in the request."

The reverse proxy is a completely normal **EC2 instance running the reverse proxy service** — not an API gateway.

---

## Docker Compose Infrastructure Stack

### Local Development Environment Setup

When starting local development, developers run `docker compose up` in the root folder, which spins up all required infrastructure:

```
Services started by Docker Compose:
- Postgres (with schema creation and static data population)
- Redis (local instance)
- LocalStack (local AWS services)
- Kafka (local message queue)
- Zookeeper (required dependency)
- Reverse proxy (local routing)
```

[Erik Andersson]: The Docker Compose file creates all necessary infrastructure for the integration platform to function locally. It includes:

1. **Postgres Database**: References two SQL files:
   - `create_schema.sql` — creates all tables in the integration platform
   - A population script — populates static data, including different statuses for the syncing process

2. **Redis**: Local caching layer

3. **LocalStack**: Simulates AWS services locally, including a local version of SQS (though this is noted as incomplete)

4. **Kafka & Zookeeper**: Message queue infrastructure

5. **Reverse Proxy**: Routes requests to staging environment services

### Launch Script and Environment Configuration

[Erik Andersson]: The launch script (located at `app/local/managers/main.go`) is the entry point for local development. It:

- Starts the debug session in Visual Studio Code
- Attaches environment variables
- Creates an AVS session
- Retrieves integration platform configuration
- Sets up repositories for installation
- Configures all required service clients

The exact services and ports started depend on which file is selected when launching the debug session. The configuration specifies exact ports for each service:
- Integration Manager — specific port
- Mappings Manager — specific port
- Delta Sync Manager — specific port

---

## Running Manager Services Locally

### What Can Be Run Locally

**Current Capability**: Only **manager services** can be debugged locally, not asynchronous workers.

Manager services that can run locally:
- Integration Manager
- Mappings Manager
- Delta Sync Manager
- Outbound Manager
- Broker Service

What you can do locally:
- Create installations
- Update installations
- Set consent mappings
- Run broker service
- Run outbound managers with full debugging capability

**Limitation**: Full sync operations cannot be run locally because the asynchronous worker services are not yet available for local debugging.

---

## Debugging with Visual Studio Code

### Setting Breakpoints and Step-Through Debugging

[Erik Andersson]: Once the integration services are running locally, developers can add breakpoints in Visual Studio Code and step through the code:

The launch configuration is defined in `launch.json` and references whichever main.go file was selected. This allows debugging any manager service.

**Example workflow**:
1. Navigate to a manager service file (e.g., `server/install.go` for the Integration Manager)
2. Click the line to add a breakpoint
3. Trigger the action from the frontend (e.g., install WooCommerce)
4. The debugger halts at the breakpoint
5. Use debug variables to inspect request data, database values, and response construction

[Lukasz Grabowski]: The `get_integration` endpoint is heavily used and will hit breakpoints frequently, making it a less ideal target for persistent breakpoints.

[Erik Andersson]: The debug variables panel shows detailed request information, such as:
- Account ID being accessed
- Service configuration details
- Specific CRM platform being installed (e.g., WooCommerce)
- Installation body data

---

## Frontend Integration and Local Routing

### Redirecting Frontend Requests to Local Services

To connect the frontend to locally running services:

1. **Add a local alias** in the system hosts file:
   ```
   dev.appsis -> localhost
   ```
   This maps `dev.appsis` (already used in multiple places in the codebase) to the local machine.

2. **Modify frontend environment configuration**:
   - Repository: `Apsis-frontend`
   - Location: `src/environments/`
   - File: `environment.base.ts` (central configuration)
   
   Change the integration base URL:
   ```
   integration_base_url: "dev.appsis"
   ```

3. **Result**: All frontend requests to integration services now route to the local machine, while other requests (to staging Audience, other services) continue to go through the staging environment.

### Live Frontend-to-Local Workflow

[Erik Andersson]: When the frontend is configured to point to local services and the backend is running locally:
- Frontend requests to integration endpoints hit the locally running managers
- Requests appear in the Visual Studio Code debug session's logs
- This enables end-to-end debugging from UI interaction to backend code

Example: Navigate to the integration page in the frontend → the debug session shows incoming requests → add breakpoints to inspect what's happening.

---

## Special Case: WooCommerce Integration

### Installation Without Full Platform Utilization

[Erik Andersson]: WooCommerce is an example of an integration that uses Apsis One but doesn't fully utilize the integration platform:

- WooCommerce plugin is installed in the CRM by a third-party vendor
- The plugin communicates only via Apsis One API
- Installation process involves:
  - Whitelisting key spaces on the CRM
  - Generating an Apsis One API key via the Audience service (through the reverse proxy)
  - Displaying the client ID and client secret in the UI

What WooCommerce does **not** use:
- Field mappings
- Subscription mappings
- Full sync functionality
- Webhook registration
- Remote configuration injection

This explains why WooCommerce goes through the installation flow but has no additional configuration steps in the integration platform.

---

## Current Limitations and Workarounds

### Worker Service Debugging Not Yet Available

[Erik Andersson]: Worker services (asynchronous processors) cannot yet be debugged locally due to incomplete queue infrastructure setup.

**Reason**: Worker services consume messages from SQS queues. To run workers locally, you would need:
1. A local SQS equivalent (LocalStack provides this, but it's not fully wired)
2. Local managers to send messages to the local SQS queue
3. Local workers to consume from the local queue

One team member (Adam, now departed) started this work but didn't complete it before leaving.

**Current Workaround for Worker Development**:
> [Erik Andersson]: "Usually add a metric load of logs to the flow and deploy to staging and see what happens essentially."

**Downsides of the staging deployment workaround**:
1. Deployment is slow — each cycle takes approximately 3 minutes (service deregistration and reregistration)
2. Risk of disrupting other teams' testing, though in practice this is minimal because:
   - Few other teams depend on the integration environment
   - Only the Integration Manager (to list integrations) and Outbound Manager (for event registration) are used by external teams
   - Everything else is internal to the integration platform team

**Rule of thumb**: Develop and debug managers locally. For workers, add logs and deploy to staging.

---

## Mock Service for CRM System Simulation

[Erik Andersson]: The **mock service** is a development tool that implements generic connector endpoints, allowing developers to simulate a CRM system:

**Capabilities**:
- Acts like a CRM system with generic connector functionality
- Can be configured to accept requests from the integration platform
- Can respond with test data
- Allows verification that the integration platform sends correct data to CRM systems
- Allows testing that the platform handles CRM system responses correctly

**Use cases**:
- Verify data being sent to a CRM is correct
- Test edge cases, like a CRM sending malformed data
- Validate that new functionality works as intended
- Simulate various CRM behaviors without a real system

This enables safer, more controlled testing during development without affecting real integrations.

---

## Next Steps and Follow-Up Topics

### Homework Assignment

[Lukasz Grabowski]: Team members should:
1. Attempt to run the integration platform locally following Erik's setup steps
2. Practice debugging a manager service with breakpoints
3. Familiarize themselves with the Go language used in the platform

### Outstanding Topics for Future Sessions

[Erik Andersson]: The following topics require deeper explanation in future sessions:

1. **Reverse Proxy Configuration**: How does traffic routing actually work? How does the proxy know to forward a request for the Integration Manager to the correct Docker container?
   - This involves understanding the squid proxy configuration
   - How service discovery works for local containers

2. **Squid Proxy**: A separate detailed session on proxy mechanics and configuration

[Erik Andersson]: "Or now you can kind of assume like it works because black magic if that is enough for you right now. But we should still have a session where I explain the squid proxy and we look at this also."

### Support Model

[Erik Andersson]: Erik is designated as the technical mentor for the team:
> "I'm like I said before, I'm a tech essentially attached to all of you with like a rope right now... I am designated to assist you whatever I can."

Team members should:
- Ask questions on the team channel if issues arise
- Paste errors for assistance
- Expect follow-up sessions if problems emerge
- Reach out directly for clarification on the reverse proxy and proxy configuration

---

## Key Takeaways

1. **Local development works through a two-layer approach**: Dockerized local services for the integration platform + reverse proxy routing to staging environment services that have IP whitelisting requirements.

2. **Docker Compose handles all infrastructure**: Single `docker compose up` command starts Postgres, Redis, Kafka, LocalStack, and the reverse proxy — everything needed except the application code.

3. **Managers can be fully debugged locally**: Add breakpoints, step through code, inspect variables, and test with real staging data (via the reverse proxy).

4. **Frontend can point to localhost**: By adding a hosts alias and modifying the environment configuration, the frontend can route integration requests to local services while still accessing staging for other needs.

5. **Workers require staging deployment for now**: The local queue infrastructure isn't complete, so workers must be debugged by adding logs and deploying to staging (slow but unavoidable).

6. **The mock service enables safe testing**: Simulate CRM behavior without a real system for controlled validation.

7. **Reverse proxy is the architectural linchpin**: It solves the IP whitelisting problem but requires deeper understanding of squid proxy configuration — session to follow.

---

## Unresolved Questions and Action Items

**For Team Members**:
- [ ] Run `docker compose up` and verify all services start
- [ ] Launch the integration platform locally from `app/local/managers/main.go`
- [ ] Add a breakpoint in a manager service and trigger it from the frontend
- [ ] Begin learning Go language fundamentals

**For Erik (Follow-Up Sessions)**:
- [ ] Deep dive on reverse proxy and squid proxy configuration
- [ ] How does the proxy pattern match and route requests?
- [ ] Complete local worker service debugging setup (if time permits)
- [ ] Architecture review of service discovery for Dockerized containers

**Channel-Based Support Model**:
- Questions about local setup → ask on team channel
- Errors or blockers → paste error messages and tag Erik for help
- Additional sessions scheduled as needed based on team progress