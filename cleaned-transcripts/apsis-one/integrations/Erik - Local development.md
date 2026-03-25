---
source_file: Erik - Local development.txt
domain: Apsis One Integrations
topics: [Local development architecture, Docker-based service setup, Reverse proxy configuration, Debugging managers locally, Development workflow, IP whitelisting workarounds]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Reverse proxy, Docker Compose, Visual Studio Code debugger, Integration Manager, Mappings Manager, Delta Sync Manager, Outbound Manager, Mock Service, AWS services, Audience service, Accounts/Users service, Delegation service, SQS, Redis, PostgreSQL, Kafka, LocalStack]
session_type: knowledge-transfer
subdomains: [Architecture, Different Types of Connectors, Generic Connector, Inbound Flow, Outbound Flow]
---

## Session Overview

This session covers the local development setup for the Apsis One Integrations platform, focusing on how developers can debug and test integration managers on their local machines despite IP whitelisting restrictions imposed by external services. Erik Andersson walks through the architectural solution (using a reverse proxy in the staging environment), the Docker Compose infrastructure setup, how to launch and debug individual services in Visual Studio Code, and the current limitations (workers cannot yet be run locally). The team discusses the workaround for workers (deploying to staging with logs) and decides to conduct follow-up sessions on proxy configuration and worker debugging setup.

---

## Architecture: Local Development with IP Whitelisting Constraints

### The IP Whitelisting Problem

Normally, all integration platform services run in AWS (AVS) on a private network. External services like Audience have their IP addresses whitelisted for calls coming from the integration environment's IP addresses. When developers try to call Audience from a local machine, those calls are rejected due to IP restrictions.

> "If you try to call audience from your local machine, they will reject that. And if you are running everything locally, this is kind of a problem."

[Erik Andersson]

### Solution: Hybrid Local + Staging Architecture

The solution uses a hybrid approach:

- **Local services**: Developers run their own microservices via Docker images on their local machine
- **Local infrastructure**: Database (PostgreSQL), caching (Redis), message queues (Kafka), AWS mocks (LocalStack) all run locally via Docker
- **Staging integration environment**: Acts as a trusted reverse proxy for calls to external services

[Erik Andersson]: "Every request that goes for any of the other apps services like audience your account, accounts and user services or the delegation service... all of these calls, they first go to another proxy here... the proxy is residing within our integration environment, so the proxy that just forwards the request to the correct service."

### Reverse Proxy Security Model

The reverse proxy is **publicly accessible** from the local machine, but security is maintained through credential passing:

- The reverse proxy itself has **no authentication** of its own
- All requests must include **Apsis One credentials** (either a One token or delegation token)
- External services (Audience, etc.) validate these credentials independently
- The proxy is simply a URL-based router that pattern-matches incoming requests and forwards them to designated endpoints

[Erik Andersson]: "The proxy itself doesn't have any actual security on it, but we do pass along the apsis one credential because audience still requires you to have a one token or a delegation token."

**Key point**: The reverse proxy runs on a **standard EC2 instance** in the integration environment, not an API gateway.

---

## Docker Compose Infrastructure Setup

### Full Local Infrastructure Stack

When developing locally, you start all required infrastructure with a single command:

```bash
docker compose up
```

This launches:

1. **PostgreSQL database** - With initialization scripts:
   - `create schema` script - Creates all integration platform tables
   - Population script - Loads static data (sync statuses, etc.)

2. **Redis** - For caching

3. **Kafka + Zookeeper** - For message queuing

4. **LocalStack** - AWS service emulation (S3, SQS, etc.)

5. **Local reverse proxy** - For routing between services (separate from the staging proxy)

[Erik Andersson]: "We start a local Redis instance. We have yet another proxy, which I'm gonna show you in a second. We also start uh local stack like local uh ABS services. We have a local Kafka instance. And then a zookeeper."

The Docker Compose file lives in the **root folder** of the repository.

### Testing Infrastructure

Team members have successfully run Docker Compose locally:

[Michal Rosikiewicz]: "Yeah, Docker Compose sets it up and transits OK locally."

[Tomasz Kowalski]: "I could run mango also in the local folder so."

Michal noted some issues with tests but not with Docker infrastructure itself.

---

## Launching Services Locally: The Launch Script and Configuration

### The Launch Script Architecture

Located at: `app/local/managers/` (relative to the microservices folder)

The main entry point is a **launch.json** configuration in Visual Studio Code that controls which service starts based on the file context.

**How it works**:
1. Navigate to the desired service's `main.go` file
2. Click "Run" or use VS Code's debug launcher
3. The launcher reads `launch.json` which specifies:
   - The program to execute (dynamically set to the selected `main.go`)
   - Environment variables
   - Debug configuration

[Erik Andersson]: "This is just a completely normal EC2 instance that is running the reverse proxy service... Inside of a here where you have the microservices, you have a folder called local. And in local you have managers..."

### Initialization Process

The launch script performs the following initialization steps:

- Creates an AVS session
- Retrieves integration configuration
- Sets up repositories for installation
- Retrieves and sets up Audience clients
- Initializes all client libraries needed to call external services

### Service Port Configuration

Each manager service is started on a **specific, configured port**:

```
IAM service - [port specified]
Mappings Manager service - [port specified]
Delta Sync Manager service - [port specified]
[other manager services...]
```

[Erik Andersson]: "So like we we start the I am service and we specify do it on this port. We start the mappings manager service. We do it on this port, the delta sync manager service."

Once running, all services listen on their designated ports on `localhost`.

---

## Services That Can Run Locally (Current Support)

### Currently Supported: All Manager Services

The following can be debugged locally with breakpoints:

- **Integration Manager** - Manage installations, retrieve integration configurations
- **Mappings Manager** - Set up field and subscription mappings
- **Delta Sync Manager** - Handle incremental synchronization
- **Outbound Manager** - Register and manage event syncing
- **Broker Service** - (mentioned in passing)
- **IAM Service** - Identity and access management
- **Mock Service** - Simulates a CRM system with generic connector endpoints (covered below)

### Current Limitation: Workers

Asynchronous workers **cannot yet be run locally**.

[Erik Andersson]: "If you try to run a full sync locally, you cannot do that right now, but you can run all of the managers... to start this process you then go here to the app local managers of the main.go file..."

**Why workers are limited**: The integration platform uses AWS SQS (Simple Queue Service) for task queues. Workers need to:
1. Consume messages from an SQS queue
2. Process asynchronously

LocalStack provides SQS emulation, but the infrastructure to route local managers to send messages to the local SQS queue and have workers consume from it was never fully completed.

[Erik Andersson]: "The problem we have had is that every worker needs to be connected to a queue and he never managed to get the local queue system fully working because we use SQS... we would need to enable something locally which acts as SQS."

### Future Direction for Workers

In principle, the solution is to:
1. Have local managers send messages to LocalStack's SQS emulation
2. Have worker services consume from the local SQS queue
3. This mirrors the staging/production architecture

[Erik Andersson]: "The principle really is the same. You can in theory essentially copy paste the solution we have for the managers and just run the worker service instead."

### The Mock Service: Local CRM System Simulator

The Mock Service is a development tool that implements all **generic connector endpoints**, allowing it to act as a CRM system under your control.

**Capabilities**:
- Accept requests from the integration platform (as if it were a real CRM)
- Respond with configurable test data
- Simulate various CRM behaviors (malformed data, specific error conditions)
- Verify that the platform sends correct data to the CRM
- Verify that the platform handles CRM responses correctly

[Erik Andersson]: "Essentially the mock service it's is a service we have which has implemented the generic connector endpoints, so it acts like a CRM system that has all of the generic connector functionality... you can simulate the CRM system sending like malformed data to you, or if you want to verify that the functionality you just did is working as it should."

---

## Connecting the Frontend to Local Services

### Setting Up Local Domain Alias

To avoid hardcoding localhost in tests and to create a more production-like environment, add a local domain alias to your system's hosts file:

**Edit**: `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows)

**Add entry**:
```
127.0.0.1 dev.appsis
```

[Erik Andersson]: "One thing we have done is that we have added this dev dot appsis is an alias for a local host... Any request that is directed to dev dot appsis is now pointing to my local local machine and stuff."

This convention is used in multiple places in the codebase (not unique to the integration team).

### Frontend Configuration

**Location**: `Apsis frontend repository → src → environments → environments.model.ts` (or similar config file)

**Change**: Update the integration base URL to point to the local domain:

```
integration.baseURL: 'http://dev.appsis'
```

[Erik Andersson]: "In the Appsys front and repository in source there is a folder called Environments... In models you have an environment base and essentially this is the central configuration file for the different environments in APSIS. So what we would want to do here is that here I have said that the integration base URL. This is now pointing to this dev dot apps instead."

**Behavior**: 
- Requests to `dev.appsis/integration/*` route to your local integration services
- All other requests (e.g., `dev.appsis/audience/*`) still route to the staging environment via the actual staging reverse proxy
- This allows testing the integration UI with real staging data for dependent services while debugging your local code

---

## Debugging Manager Services with Visual Studio Code

### Setting Breakpoints and Stepping Through Code

Once services are running locally, you can debug them like any local application:

1. **Open the target manager** (e.g., `app/local/managers/integration/main.go`)
2. **Set breakpoints** by clicking in the gutter next to line numbers
3. **Trigger the breakpoint** via the frontend or API call
4. **Inspect variables** in the debug panel (left sidebar shows all request context, database values, etc.)
5. **Step through** the code execution to understand control flow and data transformations

### Example: Debugging the Integration Manager

When debugging the `GetIntegration` endpoint:

[Erik Andersson demonstrates setting a breakpoint in the GetIntegration function]:

> "Now for example here you see that we did this request on my account here AO2WD3. And here in the debug variables you can see that this request was for this account for this section."

The debug context shows:
- The account ID from the request
- Request parameters
- Any retrieved data structures
- Return values

### Caveat: High-Traffic Endpoints

Be cautious setting breakpoints on endpoints that are called frequently. `GetIntegration` is hit multiple times when navigating the UI, which can make debugging tedious.

[Lukasz Grabowski]: "Yeah, each time it it gets this point. I guess get integration is the most used endpoint."

---

## Current Workflow for Workers: Deploy to Staging with Logs

### The Logging-Based Debugging Pattern

Until local worker debugging is set up, the workflow for worker development is:

1. Add **extensive logging** throughout the worker code
2. **Deploy to the staging environment**
3. Observe logs from the deployed worker
4. Iterate based on log output

[Erik Andersson]: "Usually add a metric load of logs to the flow and deploy to staging and see what happens essentially."

### Downsides of This Approach

**Time cost**: Each deployment cycle takes **~3 minutes** (service deregistration, redeployment, startup)

[Erik Andersson]: "It takes a lot of time to do the deploy like over and over and over again every time you deploy it because the service needs to be deployed on the old service deregistered and like this can take like 3 minutes every time you do a deployment."

**Staging environment impact**: Staging is shared with other teams for testing and verification. Worker changes could inadvertently break their development work.

[Erik Andersson]: "The staging environment is actually utilised by either other teams to verify functionality. So you may you might very well like break someone else's testing or environment."

However, this is generally acceptable because:
- Few teams depend directly on the integration environment
- Staging is expected to be unstable ("wonky")
- Other teams primarily call the Integration Manager (to list integrations) and Outbound Manager (to register events)

---

## Practical Development Rules of Thumb

[Erik Andersson]: "Like that that that would be like the rule of thumb. Like if you're developing something for a manager, try and do it locally. But of course you can do the same. You can deploy it to staging with logs and see if it is a worker for now. Add add a extra bit of logs and deploy to staging and see see what happens."

**For Manager Development**:
- Always develop and debug locally first
- Use breakpoints and step-through debugging
- Test against local Docker infrastructure
- Use the Mock Service for CRM-side testing

**For Worker Development**:
- Add comprehensive logging
- Deploy to staging
- Monitor logs for behavior
- Plan for 3-minute deployment cycles

---

## Next Steps and Unresolved Topics

### Scheduled Follow-Up Session: Proxy Configuration Deep Dive

[Erik Andersson]: "We should still have a session where I explain the squid proxy and we look at this also."

The team identified that understanding **how the local reverse proxy routes traffic** between Docker containers is important for troubleshooting but not immediately critical.

> "You have now like the squid proxy and you have the reverse proxy. So this we we just need to look at like how does the proxy work."

**Why this matters**: When a local service calls another local service or the staging proxy, the request must be correctly routed to the right Docker container. This routing is currently treated as a black box ("black magic").

### Homework Assignments

Team members (Lukasz Grabowski, Tomasz Kowalski, Michal Rosikiewicz) have been asked to:

1. **Run the full local setup** exactly as Erik demonstrated:
   - Start Docker Compose
   - Launch services via VS Code
   - Connect the frontend
   
2. **Practice debugging**: Set breakpoints and step through at least one manager endpoint

3. **Learn the Go language**: Get familiar with the codebase syntax and patterns

[Lukasz Grabowski]: "Can I ask you, Tomic and me how to do some homework, try to run it like Eric did it?... And try to, you know, debug something and of course go and start learning this this language."

### Support Model

Erik will be available for questions on an ad-hoc basis via Slack/chat:

[Erik Andersson]: "I'm a tech essentially attached to all of you with like a rope right now... I am designated to assist you whatever I can. I fully expect that we might need to look a bit on this."

If blockers arise, team members should:
1. Try to resolve locally first
2. Ask Erik or other team members in a shared channel (following the Audience team's support model)
3. Schedule additional sessions if needed

---

## Key Takeaways

1. **Hybrid Architecture Solves IP Whitelisting**: Local development combines local service execution with a staging-based reverse proxy to external services, allowing full debugging without exposing internal IPs.

2. **Docker Compose is the Foundation**: A single `docker compose up` command provides all needed infrastructure (database, Redis, Kafka, SQS mock, etc.), making setup reproducible and easy.

3. **Managers Can Be Fully Debugged Locally**: All manager services (Integration, Mappings, Delta Sync, Outbound, etc.) support local execution with Visual Studio Code breakpoints, enabling efficient development and troubleshooting.

4. **Mock Service Enables CRM-Side Testing**: The Mock Service acts as a test CRM, allowing developers to verify data handling without depending on real external systems.

5. **Workers Are Currently Limited**: Asynchronous workers cannot be debugged locally yet; developers must deploy to staging and use logging-based debugging. This takes ~3 minutes per cycle but is workable for now.

6. **Frontend Integration is Straightforward**: By setting a local domain alias (`dev.appsis`) and updating the environment config, the frontend can route integration requests to local services while keeping other requests on staging.

7. **Logging-Based Debugging for Workers Is Accepted Practice**: Similar to what other teams do; not ideal but sustainable for worker development until local queue infrastructure is completed.

8. **Reverse Proxy Security Model**: The staging reverse proxy is publicly accessible but secure because external services validate credentials independently.

---

## Unresolved Questions and Action Items

**Action Items:**

- [ ] **Team (Lukasz, Tomasz, Michal)**: Run the complete local development setup as demonstrated
- [ ] **Team (Lukasz, Tomasz, Michal)**: Practice debugging at least one manager endpoint with breakpoints
- [ ] **Team (Lukasz, Tomasz, Michal)**: Begin learning Go language fundamentals
- [ ] **Erik**: Schedule follow-up session on squid proxy and reverse proxy configuration details
- [ ] **Erik** (future): Complete local SQS queue infrastructure for worker debugging (pending)

**Topics for Follow-Up Session:**

- Deep dive into squid proxy configuration and how local services route between Docker containers
- Detailed explanation of how the reverse proxy pattern-matches and routes requests to external services
- Walkthrough of proxy configuration files and request flow tracing