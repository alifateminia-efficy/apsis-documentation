---
source_file: Erik - Local development.txt
domain: Apsis One Integrations
topics: [Local Development Setup, Docker Compose Configuration, Reverse Proxy Architecture, Debugging Managers, Integration Manager, Mock Service, Worker Limitations, Environment Configuration]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Docker, Postgres, Redis, Kafka, LocalStack, Reverse Proxy, EC2, Visual Studio Code, Mock Service, Integration Manager, Mappings Manager, Delta Sync Manager, Outbound Manager, SQS]
session_type: knowledge-transfer
subdomains: [Architecture, Inbound Flow, Outbound Flow]
---

## Session Overview

Erik Andersson led a hands-on knowledge transfer session on local development practices for the Apsis One Integrations platform. The session covered the architectural setup that enables developers to run integration services locally despite IP whitelisting constraints in production, the Docker Compose infrastructure stack, debugging workflows using Visual Studio Code breakpoints, and current limitations around worker services. The team discussed practical debugging techniques and limitations when developing asynchronous worker functionality locally.

---

## Local Development Architecture Overview

### The IP Whitelisting Problem and Reverse Proxy Solution

In production, all integration platform services run in AWS VPC on a private network. When the integration platform communicates with external services like Audience, the calls originate from whitelisted IP addresses. This creates a fundamental problem for local development: direct calls from a local machine to Audience will be rejected because the local IP is not whitelisted.

[Erik Andersson]: To solve this, we created a reverse proxy that lives within the integration environment (a normal EC2 instance). When developers run services locally, here's the traffic flow:

- **Local requests to services controlled by the integration team** (Integration Manager, Mappings Manager, etc.) stay on the local machine
- **Local requests to external services** (Audience, Accounts, Users, Delegation service, feature toggles) are first routed to the reverse proxy
- The reverse proxy, residing in the integration environment with whitelisted IPs, forwards the request to the actual destination service

### Reverse Proxy Security and Credentials

[Lukasz Grabowski asked about security]: "If you call reverse proxy from public network... How is it secured?"

[Erik Andersson]: The reverse proxy itself has no built-in security layer—it's essentially pattern-matching on URLs. However, security is maintained through credential passing:

- Every request forwarded through the proxy includes Apsis One credentials (either a standard token or delegation token)
- Even though the proxy is publicly accessible (required so local machines can reach it), **downstream services reject any request without valid Apsis One credentials**
- The proxy is just a completely normal EC2 instance running the reverse proxy service—not an API Gateway

---

## Docker Compose Infrastructure Stack

### Services Started by Docker Compose

When you run `docker compose up` from the root folder, the following services start in Docker containers:

**Database & Cache:**
- **PostgreSQL** instance with two SQL scripts:
  - `create_schema.sql`: Creates all tables needed by the integration platform
  - `populate_static_data.sql`: Populates static data like sync process statuses

- **Redis**: Local instance for caching

**Message Queue & Infrastructure:**
- **Kafka**: Message broker for the integration platform
- **Zookeeper**: Required by Kafka (Erik noted he couldn't remember the exact reason, but it's needed)
- **LocalStack**: Mocks AWS services locally

**Proxy:**
- A second proxy instance (different from the reverse proxy in the integration environment)

[Michal Rosikiewicz reported]: "Docker Compose sets it up and transits OK locally"

[Tomasz Kowalski added]: "I could run mango also in the local folder so" [indicating successful local setup]

This infrastructure is **prerequisite only**—the integration platform itself has not started at this point.

---

## Starting Integration Services Locally

### Launch Script and Configuration

The development setup uses a launch script located in the codebase:

```
app/local/managers/main.go
```

This script performs critical initialization:
- Creates an AVS session
- Retrieves the integration platform configuration
- Sets up repositories for the installation
- Retrieves and sets up Audience clients
- Configures all clients needed to call different services

### Starting Individual Manager Services

[Erik Andersson demonstrated]: Developers use Visual Studio Code's debug session launcher with the `launch.json` configuration. When you select a file like `app/local/managers/main.go` and launch the debug session, the program path automatically references that file.

The launch process starts each manager service on a specific port:
- **IAM Service**: specific port
- **Mappings Manager Service**: specific port  
- **Delta Sync Manager Service**: specific port
- [And all other manager services similarly]

Each service is configured with environment variables needed for operation. All services now listen on localhost at their designated ports, ready to accept API requests.

---

## Debugging Manager Services with Visual Studio Code

### Setting Breakpoints and Stepping Through Code

One of the primary advantages of local development is the ability to debug with breakpoints.

[Erik Andersson demonstrated a concrete example]:

1. Navigate to a manager function (e.g., `server/install` in the Integration Manager)
2. Place a breakpoint in the code
3. Trigger the action from the frontend (e.g., install WooCommerce)
4. Execution pauses at the breakpoint
5. In the debug variables panel, you can inspect the request details:
   - Account ID (e.g., `AO2WD3`)
   - Installation details
   - Mapping data

[Lukasz Grabowski noted a gotcha]: "Each time it [gets] this point. I guess get integration is the most used endpoint."

The `GetIntegration` endpoint is called frequently (when listing integrations, etc.), so placing breakpoints there means the debugger stops many times. Choose breakpoint locations carefully to avoid excessive interruptions.

### Observable Data Points During Debugging

When stepping through a manager function, developers can observe:
- What data is retrieved from the database
- The exact request parameters
- The response being returned
- The flow through different code paths

This capability is described as "extremely useful" for debugging and identifying issues like null pointer exceptions.

---

## The Mock Service: Simulating CRM Systems

### Purpose and Architecture

The mock service is a development tool that implements the **generic connector endpoints**. It acts like a CRM system that has all generic connector functionality but is under developer control.

[Erik Andersson explained its use cases]:

The mock service can be configured to:
- **Accept requests** from the integration platform
- **Respond with test data** that you control
- **Simulate invalid or malformed data** to test error handling
- **Verify that functionality works as intended** before connecting to real CRM systems

This is invaluable for testing:
- Whether the integration platform sends correct data to a CRM system
- Whether the platform correctly handles responses from a CRM system
- Error scenarios and edge cases

---

## Current Limitations: Manager vs. Worker Services

### What Can Be Run Locally

[Erik Andersson stated explicitly]: "There exists 1 limitation right now with the local development and that is what you can run locally so far are all of them Managers meaning each of the service that you interact with with requests, not the asynchronous workers."

**Can be debugged locally:**
- Integration Manager
- Mappings Manager
- Delta Sync Manager
- Outbound Manager
- Broker service
- All manager services

**Cannot yet be debugged locally:**
- Worker services (asynchronous background processes)

If you attempt a full sync locally, you cannot do so because full syncs rely on workers. However, you can:
- Create an installation
- Update an installation
- Set consent mappings
- Use the broker service
- Use outbound managers

### Why Workers Cannot Run Locally Yet

[Erik Andersson explained the technical blocker]:

> "The problem we have had is that every worker needs to be connected to a queue and he never managed to get the local queue system fully working because we use SQS."

**The implementation gap:**
1. Workers require connection to a message queue (SQS in production)
2. Managers would need to send messages to a local SQS queue
3. Workers would consume from the local SQS queue
4. A previous team member (Adam) was working on this but was reassigned before completion
5. LocalStack now has a fully localized version of SQS, but the integration hasn't been completed

[Erik noted]: "If you manage to pull this off then you have a complete local functionality for whole of integration platform."

---

## Frontend Integration with Local Services

### Redirecting Frontend Requests to Localhost

To test the frontend against local services, developers modify the frontend configuration:

**Step 1: Create a localhost alias**
```
/etc/hosts
dev.appsis  127.0.0.1
```

(This may vary by OS/environment, but `dev.appsis` as a localhost alias is a pattern used in multiple repositories)

**Step 2: Modify frontend configuration**
```
In Apsis frontend repository:
src/environments/environment.base.ts (or similar)
```

Set the integration base URL:
```
integration_base_url: 'https://dev.appsis'  // (or appropriate dev URL)
```

**What this accomplishes:**
- Frontend requests to integration services now go to localhost instead of staging
- Frontend requests to **other services** (not integration) still go to the staging environment
- This allows isolated testing of integration functionality with local backend code

[Erik demonstrated this in action]: When navigating to the integration page in the frontend, Visual Studio Code logs showed requests arriving at the local integration service, confirming the routing worked correctly.

---

## Current Development Practice for Workers

### Logging and Staging Deployment

Since workers cannot be debugged locally yet, the current workflow is:

[Erik Andersson]: "Usually add a metric **** load of logs to the flow and deploy to staging and see what happens essentially."

**The downsides of this approach:**

1. **Deployment time**: Every deployment takes ~3 minutes because the service must be deregistered from the old deployment before the new one starts
2. **Shared staging environment**: The staging environment is used by other teams to verify functionality, so changes can potentially break others' testing (though in practice, few teams depend on the integration environment—mostly for getting integration lists and registering event subscriptions)

[Lukasz Grabowski responded]: "That's fine. I frankly speaking, I often do the same with some services in our domain."

**Current guidance**: 
- For manager functionality: develop and debug locally
- For worker functionality: add thorough logging and deploy to staging to verify behavior
- The rule of thumb is to prefer local development for managers when possible, to avoid repeated staging deployments

---

## WooCommerce Installation Example: When Services Don't Use the Full Platform

### A Concrete Integration Example

[Erik Andersson walked through WooCommerce as a case study]:

WooCommerce integration demonstrates an important pattern:
- A plugin is installed in the CRM system (but **not developed by Apsis**, by a third party)
- The plugin calls the **Apsis One API** (not the integration platform directly)
- During installation, the integration platform:
  - Whitelists key spaces on the CRM attribute
  - Generates an Apsis One API key via the reverse proxy
  - Displays the client ID and client secret in the UI at installation time

**What WooCommerce installation does NOT do:**
- No field mappings
- No subscription mappings
- No full syncs
- No webhook registration
- No injection of remote details

> "They don't actually utilize the integration platform apart from us setting up a key space and giving them a one API key."

This is still processed through the installation manager, but there's minimal integration platform logic involved. The third-party plugin handles all the actual syncing via the Apsis One API.

---

## Reverse Proxy Configuration Deep Dive (Deferred Topic)

### What Needs Further Explanation

[Erik Andersson raised a topic for future sessions]:

> "You have now like the squid proxy and you have the reverse proxy. So this we we just need to look at like how does the proxy work."

**Key outstanding questions:**
- When a request is made to the integration manager, how does the system know which Docker container to route it to?
- How is traffic directed to different Docker instances?
- The difference between squid proxy and the reverse proxy
- Proxy configuration details

This requires a dedicated follow-up session to examine proxy configuration files and understand the routing mechanism fully.

---

## Key Takeaways

1. **Local development is feasible for managers**: All manager services (Integration, Mappings, Delta Sync, Outbound, Broker) can be debugged locally with breakpoints in Visual Studio Code using the Docker Compose infrastructure stack.

2. **Reverse proxy bridges the IP whitelisting gap**: External service calls (Audience, Accounts, Users) are routed through a reverse proxy in the integration environment, which has whitelisted IPs. Credentials are passed through to maintain security.

3. **Docker Compose provides the full infrastructure**: A single `docker compose up` starts Postgres, Redis, Kafka, Zookeeper, LocalStack, and proxy services—everything needed except the integration platform itself.

4. **Worker debugging remains incomplete**: Workers cannot be debugged locally yet due to incomplete SQS queue integration. Current workaround is verbose logging + staging deployment (~3 min per cycle).

5. **Frontend can be pointed at localhost**: Using a `dev.appsis` localhost alias and modifying `environment.base.ts`, developers can test the frontend against local services while still using staging for non-integration services.

6. **Mock service enables CRM testing**: The mock service simulates a generic connector-compatible CRM system, allowing verification of data flows and error handling without hitting real systems.

7. **Manager debugging is extremely valuable**: The ability to set breakpoints, inspect variables, and step through code is far more efficient than the logging + deploy + wait cycle required for workers.

---

## Unresolved Questions & Follow-Up Actions

1. **Future Session Needed**: Reverse proxy and squid proxy configuration—how traffic is routed to specific Docker containers and the full proxy setup
   
2. **Team Homework Assignment** (from Lukasz Grabowski):
   - Tomasz and Michal should attempt to run local development as Erik demonstrated
   - Debug at least one manager service locally
   - Begin learning Go language (prerequisite for working in this codebase)

3. **Support Model**: Team agreed to follow the Audience team's support model—post questions/errors in a shared channel, and Erik (as tech lead attached to this team) will assist as available

4. **Potential Future Work**: Completing the worker local debugging setup, which would enable "complete local functionality for whole of integration platform"