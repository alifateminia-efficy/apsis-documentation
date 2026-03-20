---
source_file: Erik - Local development.txt
domain: Apsis One - Integrations
topics: [Local Development Setup, Docker Configuration, Reverse Proxy Architecture, Debugging Managers, Development Workflow, IP Whitelisting Workarounds]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Docker Compose, Postgres, Redis, Kafka, LocalStack, Integration Manager, Mappings Manager, Delta Sync Manager, Outbound Manager, Mock Service, Reverse Proxy, SQS]
session_type: knowledge-transfer
---

## Session Overview

This session covered the complete local development setup for the Apsis One Integrations platform. Erik walked through the architectural necessity of using a reverse proxy to work around IP whitelisting restrictions from external services (Audience, Accounts & Users), explained the Docker Compose infrastructure required for local development, demonstrated how to debug manager services using Visual Studio Code breakpoints, and identified current limitations around worker debugging. The team identified follow-up work needed around proxy configuration details and worker service local execution.

---

## Architectural Problem: IP Whitelisting and Local Development

### The Core Challenge

The integration platform runs entirely in AWS VPC on a private network. All IP addresses used by integration services are whitelisted with external partners like Audience. When developers attempt to call these external services from their local machines, the requests are rejected because local IPs are not whitelisted.

> "If you try to call audience from your local machine, they will reject that."

### The Reverse Proxy Solution

[Erik Andersson]: To solve this, the team implemented a reverse proxy pattern:

1. **Local services stay local**: Requests to the developers' own microservices (running in Docker locally) stay on their machines
2. **External calls route through proxy**: Any request to external services (Audience, Accounts & Users, delegation services) first goes to a reverse proxy residing **within the integration platform environment** in AWS
3. **Proxy forwards requests**: The proxy reads the request URL, pattern-matches it, and forwards to the appropriate endpoint
4. **Credentials flow through**: The original Apsis One credentials are passed along with the request, so authentication still works—Audience still validates the token

[Lukasz Grabowski]: Clarified that the proxy is **publicly accessible** because developers need to reach it from outside the private network. However, this is secure because:
- The proxy has no inherent security; it just forwards requests
- Without valid Apsis One credentials, requests are rejected by the downstream service
- The proxy is just a normal EC2 instance running a reverse proxy service

---

## Docker Compose Infrastructure Setup

### Overview

Before starting the integration platform services themselves, developers need to spin up a complete local infrastructure stack. This is handled via Docker Compose in the repository root.

### Services Started by `docker-compose up`

```
Postgres Database
  - With create schema for all integration platform tables
  - With population script for static data (sync statuses, etc.)

Redis
  - Local caching/session store

Kafka + Zookeeper
  - Message broker for async operations
  - (Erik noted he couldn't remember why Zookeeper is specifically needed)

LocalStack
  - Local AWS services simulation
  - Critical for SQS emulation (though this isn't fully working yet for workers)
```

[Lukasz Grabowski] and [Tomasz Kowalski] confirmed they've successfully run the Docker Compose setup without major issues. [Michal Rosikiewicz] reported some issues with tests but not with Docker itself.

### Key Configuration File

The `docker-compose.yml` file in the root folder orchestrates all services. Running `docker compose up` starts all services simultaneously. Once complete, developers have:
- A working database (Postgres) with full schema
- In-memory caching (Redis)
- Message queue infrastructure (Kafka)
- AWS service mocks (LocalStack)

---

## Starting the Integration Platform Services Locally

### Launch Script and Local Managers Directory

The repository contains a launch script that starts the debug session in Visual Studio Code. Its behavior depends on the folder hierarchy where it's invoked.

**Directory structure:**
```
/app/local/managers/
  - main.go
```

This directory contains a Go launch script that sets up everything needed for local development:

[Erik Andersson] detailed what the launch script does:
- Creates an AVS session
- Retrieves integration config
- Sets up repositories for installation
- Retrieves and sets up all Audience clients needed for calling different services
- Starts each manager service on a specific port

### Starting Individual Manager Services

Developers navigate to the `app/local/managers/` directory and use the VS Code launch configuration in `launch.json`:

1. Select the `main.go` file in the folder
2. Click "Launch Package" with the local development configuration
3. This references the `main.go` for local development specifically
4. Each manager service starts on its configured port:
   - IM (Integration Manager) - specific port
   - Mappings Manager - specific port
   - Delta Sync Manager - specific port
   - (and others as configured)

All these ports are documented in the configuration file, but Erik noted the full enumeration would be tedious to list.

---

## Debugging Manager Services with Breakpoints

### The Capability

Once local services are running, developers can set breakpoints anywhere in the manager code and step through execution with full access to variables and data state.

[Erik Andersson] demonstrated this with the Integration Manager:

1. Navigate to `server/install` handler code
2. Set a breakpoint in the function
3. Trigger the action in the UI (e.g., install WooCommerce integration)
4. Execution halts at the breakpoint
5. Inspect variables: see which account made the request, which integration was requested, the full installation request body
6. Step through to see database queries, return values, etc.

**Example debug session variables shown:**
- Account ID: `AO2WD3`
- Section information
- Integration details
- Installation body contents

### Caveats

- **High-frequency endpoints are tedious**: `GetIntegration` is called very frequently (every time the integration page loads), so setting breakpoints there causes frequent hits
- **Need to set breakpoints strategically**: Avoid frequently-called endpoints for efficient debugging

### Value Proposition

[Lukasz Grabowski]: This provides "a full solution for debugging and development, at least for managers."

---

## What Can and Cannot Run Locally

### Current Limitations

**Cannot run workers locally yet:**

[Erik Andersson]: Workers cannot currently be debugged or run fully locally. The limitation stems from queue connectivity. Workers need to be connected to SQS (Simple Queue Service) in AWS. To run workers locally would require:

1. A local SQS emulation (LocalStack may have this now, but it wasn't fully implemented when discussed)
2. Local managers to send messages to the local SQS queue
3. Local workers to consume from that local SQS queue
4. Full integration of all three components

> "The problem we have had is that every worker needs to be connected to a queue and he never managed to get the local queue system fully working because we use SQS... we would need to enable something locally which acts as SQS."

[Erik] mentioned that a former team member (Adam) attempted to implement this before being let go, and it was never completed.

### What CAN Run Locally

Developers can run and debug all manager services:
- **Integration Manager**: Create/update installations, manage configurations
- **Mappings Manager**: Configure field mappings
- **Delta Sync Manager**: Handle sync operations
- **Outbound Manager**: Register event syncing
- **Broker Service**: As discussed in previous sessions
- **All other manager-type services**

This covers the synchronous request-response path. **Asynchronous work triggered via workers cannot be fully debugged locally.**

---

## Current Workaround for Worker Development

### The Logging-Based Approach

For work on asynchronous workers, developers currently follow this pattern:

[Erik Andersson]:
> "Usually add a metric [load] of logs to the flow and deploy to staging and see what happens essentially."

### Downsides of This Approach

1. **Time cost**: Each deployment takes ~3 minutes because the service must be deregistered from the old deployment, then re-registered
2. **Shared staging environment**: The staging environment is also used by other teams for testing. Developers can potentially break someone else's testing or development work
   - Erik noted this is not a major issue for the integration platform because few teams depend on its services (mainly just the integration manager and outbound manager calls)
   - Other services in the platform are almost exclusively used internally

### Guidance on When to Use This Approach

[Erik Andersson]: The rule of thumb is:
- For manager work: **develop and debug locally first**
- For worker work: **currently, add logs and deploy to staging** (no local debugging option yet)

The staging environment, while expected to be "a bit wonky," usually handles this workload because the integration platform has limited external dependencies.

---

## Integration Setup Process: WooCommerce Example

### How the Installation Flows Through Local Debugging

[Erik Andersson] demonstrated the installation flow by installing WooCommerce:

1. User initiates installation through the frontend
2. Request hits the local Integration Manager
3. Breakpoint stops execution
4. Debug view shows:
   - Account ID: `AO2WD3`
   - Integration type: WooCommerce
   - Installation body with all submitted configuration
   - Client ID and Client Secret from generated Apsis One API key

### Why WooCommerce Looks Different

WooCommerce installations differ from typical integrations because:

1. **Third-party plugin**: WooCommerce integration is **not** developed by the Apsis team; it's developed by another company
2. **Limited integration**: WooCommerce only uses the Apsis One API, not the full integration platform
3. **Installation steps**: 
   - Whitelist keyspaces on the CRM attribute
   - Call Audience to generate an Apsis One API key
   - Display the client ID and client secret to the user in the UI
   - **Stop there** — no field mappings, subscription mappings, or full syncs
4. **No webhook registration or remote injection**: Because WooCommerce doesn't utilize the full integration platform, no remote details injection occurs

This pattern can be fully debugged locally because it only involves the manager service making calls through the reverse proxy to Audience.

---

## Mock Service for CRM System Testing

### Purpose and Functionality

The **mock service** is a development tool that implements the generic connector endpoints. It acts like a CRM system with full generic connector functionality that developers control.

[Erik Andersson]: Use cases:
- **Verify data sent to CRM**: Test whether the integration platform sends correct data to CRM systems
- **Verify response handling**: Test whether the platform correctly handles responses from CRM systems
- **Simulate edge cases**: Configure the mock to send malformed or unusual data to test error handling
- **Full control**: Developers can determine exactly what the mock supports and how it behaves

This allows testing the full request-response cycle with a CRM system locally, using real staging data via the reverse proxy.

---

## Frontend Integration with Local Services

### Redirecting Frontend Requests to Local Development

To test the full stack (frontend + local backend), developers need to redirect frontend requests from staging to their local machine.

### Step-by-Step Configuration

1. **Add a host alias** in `/etc/hosts`:
   ```
   127.0.0.1 dev.appsis
   ```
   [Erik] noted this practice may be common across the platform, as `dev.appsis` appears in other places.

2. **Modify frontend environment configuration**:
   - Location: `Apsis-frontend` repository
   - Navigate to: `source/environments/models/environment.base`
   - This is the central configuration file for all APSIS environments

3. **Update the Integration Base URL**:
   - Change `integration_base_url` to point to `dev.appsis` instead of staging
   - Example: `integration_base_url: http://dev.appsis:9497` (or appropriate port)

4. **Start the frontend**:
   ```
   npm install  # or equivalent (depends on node version)
   npm start
   ```

### Important Caveat

The frontend still calls **staging** for everything except the integration service:
- Other services (user profile, features, etc.) → staging
- Integration service → local machine (dev.appsis)

### Verifying the Connection

[Erik Andersson] demonstrated:
1. Navigate to the integration page in the frontend
2. Check VS Code for integration service logs
3. If logs appear showing requests, the local connection is working
4. Frontend is successfully calling the local Integration Manager

---

## Proxy Configuration Details (Flagged for Follow-up)

### The Gap in Understanding

[Erik Andersson] identified that while the reverse proxy and Squid proxy are discussed, the **routing mechanism** wasn't fully explained:

> "How does the service know if I make a request to the integration manager... how does it know that it should talk to this specific docker container?"

### What Needs Clarification

1. **Proxy routing configuration**: How does the reverse proxy route to the correct Docker container running on the developer's machine?
2. **Squid proxy setup**: What is the role of Squid proxy in this architecture?
3. **Traffic direction**: The specific configuration that directs traffic to the right instances

[Erik] suggested this doesn't "happen by accident"—there's explicit proxy configuration that needs to be understood.

### Recommendation for Follow-up Session

This is flagged as a necessary follow-up session topic to understand the complete picture of how traffic flows through the proxy infrastructure.

---

## Knowledge Transfer and Ongoing Support

### Team Ownership Transition

[Erik Andersson]: He is currently designated as a technical mentor ("tech essentially attached to all of you with like a rope") for the knowledge transfer, specifically supporting [Lukasz Grabowski], [Tomasz Kowalski], and [Michal Rosikiewicz].

He is available unless there's a P1/P2 incident or critical CRM work needed.

### Suggested Support Model

[Lukasz Grabowski] proposed adopting a peer-support model similar to how the Audience team operates:
- Ask questions directly in a team channel
- Paste errors or issues
- The team helps troubleshoot

This would allow knowledge distribution without requiring Erik for every question.

### Homework Assignments

Before the next session, the team should:
1. **Run the full local setup** as Erik demonstrated
2. **Debug a real manager function** by setting breakpoints and stepping through
3. **Start learning Go** (as a prerequisite for understanding the codebase)

---

## Key Takeaways

1. **Local development is possible for managers but not workers**: The synchronous manager services can be fully debugged locally with breakpoints, but async workers currently require staging deployments with added logging.

2. **Reverse proxy is essential and safe**: By routing external calls through a reverse proxy in the AWS environment, developers can work locally despite IP whitelisting restrictions. Credentials flow through the proxy, maintaining security.

3. **Docker Compose provides complete infrastructure**: A single command (`docker compose up`) spins up Postgres, Redis, Kafka, LocalStack, and other services needed for local development.

4. **Breakpoint debugging is the main advantage**: The ability to set breakpoints, inspect variables, and step through manager code in real time is the core value of the local setup versus logging-based debugging.

5. **Frontend can be redirected to local backend**: By modifying the environment configuration file and adding a host alias, the frontend can be pointed to local services while still connecting to staging for other components.

6. **Mock service enables CRM testing locally**: The mock service simulates a CRM system with full control over what it supports and how it responds, allowing testing without external dependencies.

7. **Worker debugging is a known gap**: There's acknowledgment that workers cannot be debugged locally, and the team should explore whether LocalStack now has full SQS support to enable this in the future.

8. **Proxy routing details need clarification**: The exact configuration of how the proxy routes requests to the correct Docker containers is flagged as needing a dedicated follow-up session.

---

## Unresolved Questions and Action Items

### For Follow-up Sessions

1. **Proxy routing mechanism**: How exactly are requests routed through the reverse proxy to the correct Docker container? What is the proxy configuration?

2. **Squid proxy role**: What is the Squid proxy's specific role in this architecture?

3. **Worker local execution**: Can LocalStack now provide a full SQS emulation? Is this viable to implement locally?

### Homework for Team

1. Run the full Docker Compose setup locally
2. Start one of the manager services with VS Code debugging
3. Set a breakpoint and trigger an action to verify debugging works
4. Begin learning Go language fundamentals

### For Erik (Time Permitting)

- Prepare a detailed walkthrough of the proxy configuration in the next session
- Document any recent changes to the local development setup
- Explore feasibility of enabling worker debugging if LocalStack SQS support is available