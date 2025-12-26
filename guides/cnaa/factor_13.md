# Factor XIII. Observability

- [Factor XIII. Observability](#factor-xiii-observability)
  - [Probes](#probes)
    - [Health Checks](#health-checks)
    - [Readiness Checks](#readiness-checks)
    - [Startup Checks](#startup-checks)
    - [Status Checks](#status-checks)
  - [Graceful Degradation](#graceful-degradation)
  - [Metrics](#metrics)
    - [Minimal metrics set](#minimal-metrics-set)
  - [Tracing](#tracing)
  - [Checklist](#checklist)
    - [Health Probes Checklist](#health-probes-checklist)
    - [Status Checklist](#status-checklist)
    - [Metrics Checklist](#metrics-checklist)
      - [gRPC/REST metrics](#grpcrest-metrics)
      - [Database Metrics](#database-metrics)
      - [Secrets Management Metrics](#secrets-management-metrics)

> [!NOTE]
>
> **Principle:** The application must be completely observable. This means it should provide metrics, traces, and logs that allow understanding its state and behavior in real-time.

Translate next text to english, with saving all hidden meanings and style of original text

All markdown translation MUST be cuddled in code block to copy easily.

## Probes

When we run an application, we have components that automatically check the state of the deployment, for example, Kubernetes (or any other orchestrator). To it, any application is just a black box running in a container. It doesn't know what's going on inside: whether it's a Spring Boot application waiting for a database connection to come up, or a Go binary that's ready to accept requests in 10 milliseconds.

The orchestrator needs to communicate with this box somehow. It needs to ask simple questions to understand what to do with the application. Probes are this very communication mechanism. It's as if a manager were to walk up to an employee and ask:

- "Are you okay? Not passed out?" (Liveness Probe)
- "Ready to take on new tasks?" (Readiness Probe)
- "Are you still setting up your workspace? Should I wait?" (Startup Probe)

### Health Checks

`/healthz` This is the most basic, yet most dangerous probe.

Its sole purpose is to answer the question: "Is the application completely frozen?". It checks if your process has entered a state from which it cannot recover. A classic example is a deadlock, where two threads are waiting for each other, and the entire application hangs.

**What happens if the probe fails?**

This is the key point. If the liveness probe doesn't receive a response or gets an error, Kubernetes decides: "The patient is more dead than alive. It can't be helped. The only solution is a restart." And it mercilessly kills (SIGKILL) your container and starts a new one.

This probe must be as lightweight and fast as possible. It **must not** have external dependencies. That is, checking a database connection or the availability of another service from it is a **gross mistake**.

Why? Let's say our database goes down. All 50 instances of the application can't reach it. Their liveness probes start to fail. What does Kubernetes do? It starts restarting all 50 instances. The database comes back up, and it's immediately hit by 50 new applications trying to establish a connection at the same time. This is called a "cascading failure" or "thundering herd." Thus, the problem has been made even worse.

A proper liveness probe is a simple HTTP endpoint `/healthz` that returns 200 OK if the **main loop** of the application is running. It simply confirms that the process is not hung. By "the process is not hung," it is assumed that we will check the internal state of the application, whether there are locks that have been held for more than N amount of time, whether the server is running, etc. But we do not check external dependencies.

**Real-life analogy:** A manager walks up to a barista who is frozen and staring at one spot. He gives him a light nudge. If the barista doesn't react, he's sent home and a replacement is called. The manager doesn't care if the coffee machine is working or if there's milk. What's important to him is that the barista is conscious.

### Readiness Checks

This is the **most** important and frequently used probe for traffic management.

Its job is to answer the question: "Is the application ready to handle requests right now?". An application can be completely "alive" (liveness is fine), but temporarily not ready to work.

Examples of when an application is alive but not ready:

- It has just started and is warming up its internal cache.
- It has lost connection to an external service and is trying to restore it.
- The database connection pool is exhausted, and it cannot get a new one.

**What happens if the probe fails?**

Kubernetes does NOT kill the container. It does something much smarter: it temporarily removes this instance from the load balancer. New requests simply won't go to it, and the application can recover peacefully without extra traffic. Kubernetes will patiently check it with the readiness probe. As soon as the probe starts returning `200 OK`, it will return the instance to the rotation, and traffic will flow to it again.

**How to do it correctly?**

This is exactly where you should put all the checks for external dependencies! The `/readyz` endpoint should check:

- Is there a connection to the database?
- Is Kafka available?
- Have all the necessary caches been loaded into memory for operation?

**Real-life analogy:** A manager approaches a barista (who is conscious, liveness ok). The manager asks: "Ready to take an order?". The barista replies: "No, I don't have any cups right now, wait 2 minutes". The manager tells the cashier: "Okay, don't send customers to this barista for now". After 2 minutes, cool disposable cups are brought, the barista nods, and the manager directs the queue to him again. No one is fired, the process is not disrupted.

### Startup Checks

This is a helper probe that solves the problem of slow starts, for example, for a Kafka consumer, it's necessary to wait for the moment when the consumer registers with the group and the broker rebalances the partitions.

Its purpose is to give the application enough time for its initial startup, without letting the liveness probe kill it. This is relevant for applications that need several minutes just to start, register with some component, and prepare all the necessary resources.

**What happens if the probe fails?**

It works like this: in the Kubernetes config, you specify a configuration — "Listen, this application is slow. Give it 5 minutes to start, and during these 5 minutes, poll it with the `startup` probe. If it doesn't respond within this time, it means something went wrong, so kill it and restart it."

As soon as the `startup` probe succeeds for the first time, Kubernetes stops using it and switches to the regular liveness and readiness probes. That is, it only runs once at the beginning of the container's lifecycle.

How to do it correctly?
Usually, the startup probe simply points to the same endpoint as the readiness probe. The main thing is to correctly configure the timeouts in the Kubernetes manifest to give the application enough time.

Real-life analogy: A new trainee barista comes to the coffee shop and is very slowly and meticulously arranging all his tools. The manager knows this and says to himself: "Okay, I won't bother him and ask if he's conscious (liveness) for the first 10 minutes. I'll just watch from a distance to see if the process is moving along (startup). If he's still not ready after 10 minutes, I'll fire him. But if he gets it done sooner, I'll start treating him like everyone else."

### Status Checks

Okay, we've figured out `probes`. These are signals for the robot-orchestrator, which operates on a "yes/no" principle. But when something goes wrong, the robot doesn't care _why_ the service isn't ready. But you do. You need to find out the reason what happened to the service, and quickly.

This is exactly what a `Status Check` (or sometimes called a `Health Check API`) is for. It's a special, information-rich endpoint (for example, `/status` or `/health`) that is intended not for Kubernetes, but for a monitoring system (or a person, to see a summary dashboard for the service).

If a `readiness` probe is a red light on the dashboard, then a `status check` is a detailed report from the auto shop that says: "The problem isn't the engine, it's the oxygen sensor, part number XYZ".

**Key differences from probes:**

- **Purpose:** Not an automatic action (restart), but diagnostics. To give an engineer maximum information to quickly solve the problem.
- **Format:** Always a detailed, structured JSON. No simple 200 OKs.
- **Consumer:** A person (via curl or a browser) or a monitoring system (for example, Grafana).
- **Security:** It's usually hidden on a separate management port or protected by authentication, so as not to expose the internal workings to the outside.

A good `status check` returns something like this:

```json
{
  "status": "DEGRADED",
  "serviceVersion": "2.5.1",
  "dependencies": {
    "database": {
      "status": "UP",
      "responseTime": "34ms"
    },
    "redis_cache": {
      "status": "DOWN",
      "error": "Connection refused"
    }
  }
}
```

In the end, having such an endpoint transforms the nightmare of "Service X is down!" into a specific task: "Redis for service X is down, we need to deal with it." This saves a ton of time and nerves.

## Graceful Degradation

As described in Factor IV, external dependencies can be unavailable. The application we are considering can also be someone else's external dependency. Therefore, we also need a convenient way to tell other services how much our application is suffering from load or simple unavailability.

A little earlier, we talked about how important readiness probes are. They allow Kubernetes to not send traffic to a service that is not ready to handle it. However, relying only on "working/not working" statuses is like judging a person's well-being by their temperature and pulse. It's useful, but it doesn't give the full picture. The most accurate information about one's own state is known only by the person themselves (or their doctor). The same goes for an application.

An advanced, fault-tolerant architecture assumes that the application does not just passively respond to probes, but **actively reports its ability to handle load.**

Of course, it's not always possible to notify the load balancer that the application cannot handle requests. What if our application *is* the load balancer? Therefore, in addition to the *ability* (but not the obligation!) to report relative readiness, the application itself must be able to **gracefully degrade**. That is, it must be able to handle requests even under harsh conditions where it is being "spammed" with requests endlessly. For this, the mechanism of Load Shedding exists.

Load shedding is not the tool for reducing load itself, but rather the reason why this message needs to be sent.

You can imagine it like this:

- **Load Shedding:** A bouncer at the entrance of a crowded club tells the next visitor: "Sorry, we're full."
- **Message to the balancer:** The bouncer radios the organizer at the main entrance: "Don't send people my way for the next 15 minutes, we're packed."

We will talk more about the implementation of load shedding in Factor XIV, as this mechanism relates to resource limits. For now, we need to understand in detail how the application can use that very radio the bouncer is talking through.

There are two main approaches to implementing such communication: Pull vs. Push.

- **Pull model** — this is when the application provides a special, lightweight endpoint (e.g., `/load_status`), and the platform (load balancer, service mesh) polls it periodically.

  This is a simple and reliable approach: the service does not need to adhere to the contract of a specific platform; it is solely the responsibility of the DevOps engineer to configure the collector to retrieve the service's state.

  **Important nuance:** Unlike a heavy metrics endpoint (`/metrics`), which is polled infrequently (every 15-60 seconds), the load status endpoint **must be as lightweight as possible** because it can be polled by the balancer very frequently (every 1-5 seconds). This ensures a sufficient reaction speed.
- **Push model** — this is when the application not only recognizes its load capacity but also sends a notification to the platform itself when its state changes. The key benefit is an instant reaction without polling delay. However, this approach requires the application to have knowledge of the infrastructure and complicates its code with delivery and authentication logic.

While the Pull model is quite straightforward, to understand how and why to use the Push model, one must understand its difference from logs.

First, let's recall the key idea of logs: **a log is an event**. A notification about the load status is also an event. But the key difference lies in their **purpose and consumer**.

Second, the consumer of a log is typically an aggregation system (like Loki, Elasticsearch) and **ultimately a human** (developer, SRE) who analyzes past events for debugging or auditing. The consumer of a push notification about status is **always an automated system** that must make an immediate decision.

Events remain events — this doesn't negate the fact that when the load state changes, the application should also send events to the log. But the consumer of this event is not a human, which means they ultimately won't be looking through logs to find the necessary event. This is why push notifications are built on a separate notification system, which is used in addition to logs and metrics.

The ideal approach for notifying about load status is to use both models together. The Pull model does not depend on the load balancer's implementation and also always guarantees to indicate the current state of the service, albeit once every N seconds. The Push model, in turn, very quickly notifies the balancer of a parameter change, although not with a 100% guarantee. This allows the application to be as responsive as possible. Of course, it's not necessary to do everything at once, and depending on the system's requirements, one of three approaches can be chosen:

1. **Basic (Pull model)**.

   This is the main and most universal pattern. The application implements a lightweight endpoint that returns its current load status. The platform is configured to poll it frequently. **This is sufficient for most systems.**
2. **Advanced (Hybrid model)**.

   This is a combination of the two approaches to achieve the best of both worlds.

   - The Pull model works as a reliable state reconciliation mechanism.
   - The Push model works as a best-effort optimization: the application tries to send an instant notification (for example, via a configured callback URL), but is **not obligated** to guarantee its delivery. If the notification is not delivered, the next poll from the pull model will still correct the situation.
3. **Idiomatic "Service Mesh Native" (Push via Sidecar)**.

   This is the cleanest and most architecturally correct approach in modern systems.

   - The application sends a push notification to its local sidecar (for example, on localhost:1234).
   - The sidecar, as part of the infrastructure, takes on all the complex and insecure work of communicating with the control plane.

    This approach provides an instant reaction while keeping the application code simple and isolated from infrastructure concerns.

Regardless of the model, the application must be able to report the following information about itself (for example, in JSON format):

- **Load Signal:**

  - `weight_percent` (from 0 to 100): Relative performance. 50 means "give me half as much traffic."
  - `max_concurrency` (number): The maximum number of concurrent requests that the service is ready to handle right now. If the value is -1, it means the service does not limit itself and is ready to handle as much as it can. However, it's worth remembering that this is not an ideal practice, because the load balancer might go crazy and start sending all requests to this overconfident service. What if?
- **Operational State Signal:**
  - `state` (only `OK`, `DRAINING`, `MAINTENANCE`): The current operational status, important for zero-downtime deployments.
    - `OK` — the service is operating normally.
    - `DRAINING` — the service is temporarily not accepting new requests, but is processing those already received. This is useful during deployment or scaling.
    - `MAINTENANCE` — the service is under maintenance and should not process requests at all.
- **Degradation Signal:**
  - `status` (only `HEALTHY`, `DEGRADED`): The overall status, indicating whether the service is operating at full capacity or if some of its dependencies have failed.
  - `message` (string): A human-readable message for debugging, for example: `"Cache dependency is unavailable. Operating in fallback mode."`

Example response from the `/load_status` endpoint:

```json
{
  "status": "DEGRADED",
  "load": {
    "weight_percent": 75
  },
  "state": "OK",
  "message": "Cache dependency is unavailable"
}
```

A similar response can be returned by the callback for push notifications, if they are configured at all.

However, that's not all! We haven't mentioned a very important aspect that is also very important for this endpoint!

**mTLS by default:** The state of the service is sensitive information. If we have a system with callbacks, we need to know who the author of the message is (hello, zero trust!). Therefore, the callback must be sent with a client certificate, and the recipient of this message must verify it to ensure that it is indeed the service they expect. If an attacker sends the callback, they can fake that a service instance has failed, thereby deceiving the load balancer, which can lead to a cascading failure. Therefore, callbacks must be verified.

## Metrics

In metrics, there are four "golden signals" that should be monitored in every service:

- **Latency** — how long it takes to process a request. We separately monitor latencies for successful executions and for errors. Successful execution is an obvious metric, but we would also like to receive errors faster.
- **Traffic** — the overall picture of the service's load from requests. For an API, this is the number of requests per second. For a music streaming service, it's the volume of data transferred per second.
- **Errors**. Errors can be explicit (e.g., 500-series error codes) or implicit. An implicit error is when we receive a response, but not with the expected content or with too much delay.
- **Saturation**, or the load on our service. How much more load can it handle? Usually, a service is limited by CPU or memory resources. These limits must be monitored. In a database, for example, you need to monitor free space.

### Minimal metrics set

- Runtime
  - CPU Usage
  - Memory Usage
  - Disk Usage
  - Network Usage
- API
  - Total requests
  - Latency (time spent processing the request)
  - Failed requests (errors `4xx`, `5xx`)
  - Request size
  - Response size
  - Retry Attempts (number of retry attempts, measured on idempotent requests)
- AuthN
  - Total attempts
  - AuthN latency (time spent on authentication)
  - Failed Auth Attempts (JWT token validation, etc)
- AuthZ
  - similar as AuthN, but for authorization
  - Action audit (which actions were performed, which permissions were checked)
- Caching
  - Cache Hits/Misses
- External API
  - Total requests
  - Latency
  - Failed requests (errors `4xx`, `5xx`)
  - Request size
  - Response size
- Database
  - Connection pool (amount of active connections, amount of available connections)
  - Total queries (splitted by categories)
  - Queries latency
  - Transactions total
  - Transactions latency
  - Transactions failed
- Feature Flags
  - Total flag evaluations
  - Flags latency (time spent on flag evaluations)
  - Failed flags requests
- Tracing
  - Injected trace ids (number of requests in which a trace id was passed)

## Tracing

Besides logs and metrics, there is a third pillar of observability — tracing. If **metrics** tell you "what" broke (latency increased), and **logs** tell you "why" (DB connection error), then **traces** tell you "where exactly" in the distributed system it happened.

A trace is the story of a single request that travels through multiple microservices. It shows how much time the request spent in each service and where the "bottlenecks" were. This is invaluable for debugging complex distributed systems. The `OpenTelemetry` standard is used for this.

**What a trace provides:**

- **Finding bottlenecks:** It's immediately clear that service A responded in 20 ms, service B in 30 ms, and service C in 2 seconds. The problem is in service C.
- **Understanding dependencies:** You can see who calls whom and in what order.
- **Debugging errors:** You can find all logs and metrics related to one specific "slow" request just by its `trace.id`.

## Checklist

- [ ] The application **must** explicitly provide the ability to configure observability endpoints via environment variables in URL format, for example, `MYAPP_METRICS_ADDR=http://0.0.0.0:8080/metrics`:
  - [ ] It **must** be possible to configure both the listener address and the port.
  - [ ] The scheme **must** always be either `http` or `https`.
  - [ ] Depending on the scheme, the endpoint **must** be secured with TLS.
  - [ ] The configuration **must** allow specifying a path prefix, for example, `/metrics`.
  - [ ] It **should** be forbidden to host the business logic service and metrics on the same port.
  - [ ] The configuration **may** include the option for authorization via mTLS.
- [ ] The service **must** explicitly indicate the path to observability endpoints with an INFO level, for example:

  ```json
  {
    "level": "INFO",
    "event_id": "metrics.started",
    "addr": "http://0.0.0.0:8080/metrics",
    "message": "Metrics server started at http://0.0.0.0:8080/metrics"
  }
  ```

- [ ] The application **must** first start the observability endpoints, and only after the server has started accepting traffic, continue setting up the main service, not the other way around.

  > **NOTE:** This avoids a situation where the service is already accepting traffic, but the metrics are not yet ready. In such a case, the orchestrator will not be able to correctly check the service's state and may start restarting it, leading to a cascading failure.

### Health Probes Checklist

- [ ] The service **must** provide all three: `liveness`, `readiness`, and `startup` probes.
- [ ] Liveness Probe: `/healthz`
  - [ ] The Liveness Probe **must not** check external dependencies (e.g., database or other services).
  - [ ] The Liveness Probe **must** only respond with `200 OK` and no other codes.

    > **NOTE:** If `204 No Content` is sent, some Kubernetes implementations (e.g., Google GKE and Azure AKS) may consider the service unresponsive and start restarting it. Therefore, it is better to always return `200 OK`.

  - [ ] The Liveness Probe **should not** include any data in the response body.

    > **NOTE:** The data would be practically useless, as almost no one ever processes it, even though Hashicorp Nomad, for example, allows interpreting the received data.

- [ ] Readiness Probe: `/readyz`
  - [ ] The Readiness Probe **must not** check external dependencies.
  - [ ] The Readiness Probe **must** return only `200 OK` if the service is ready to accept traffic, and `503 Service Unavailable` if it is not.

    > **NOTE:** This is the same situation as with the Liveness Probe. If `204` is sent, some Kubernetes implementations may consider the service not ready on a `204 No Content` response.

  - [ ] The Readiness Probe **must** incorporate a `Circuit Breaker` pattern.
  - [ ] The Circuit Breaker **must** have configuration settings via environment variables for precise behavior tuning:
    - [ ] `threshold` (int): The TOTAL number of errors on the server controller that must occur before it starts considering the service not ready.
    - [ ] `threshold_window_interval` (duration): The size of the window (**in duration format!**) during which errors will be summed. For example, if `threshold` = 10 and `threshold_window_interval` = 1s, then if 10 errors occur within 1 second, the service will be considered not ready.
    - [ ] `reset_timeout` (duration): The amount of time (**in duration format!**) that must pass after the circuit opens before the service checks the half-open state.
    - [ ] `half_open_threshold` (int): The TOTAL number of requests that must pass in the half-open state before the service is considered ready. For example, if `half_open_threshold` = 5, then after the service was closed, it must process 5 requests in the half-open state before returning to the open state.

  - [ ] The Readiness Probe **must** respond with `503 Service Unavailable` only in the `Open` state.

    > **NOTE:** The orchestrator needs to return the service to rotation to send test requests.
- [ ] Startup Probe: `/startupz`
  - [ ] The Startup Probe **must** check every external dependency, such as the database, cache, etc.
  - [ ] The probe **must** return only `200 OK` if the service is ready for work, and `503 Service Unavailable` if it is not.
  - [ ] The Startup Probe **must** transition to a ready state only when the Readiness Probe starts returning `200 OK`.
  - [ ] The Startup Probe **may** return a request body, but only in the case of a `503 Service Unavailable` code if the service is not ready.

    > **NOTE:** This can be useful for diagnostics, but it is not mandatory. It is important that the request body is in JSON format and contains information about which external dependencies are unavailable. You can copy the format from the `status check` endpoint.

### Status Checklist

- [ ] The service **may** provide a `/status` endpoint for state diagnostics:
  - [ ] The endpoint **should** be run on a separate port, different from the business logic and metrics port.

    **Note:** This is at least safer, and at most, it allows to implement very granular access policies, for example, allowing access to this endpoint only from the pod's internal network.

### Metrics Checklist

- [ ] The service **must** provide a `/metrics` endpoint for collecting metrics,
- [ ] Metrics **must** be in at least OpenMetrics (Prometheus) format
  - [ ] The application **may** implement metric export via OpenTelemetry, but without disabling the main `/metrics` endpoint.
- [ ] The minimum set of metrics **must** be implemented as described in the checklist below.

#### gRPC/REST metrics

---

> [!WARNING]
>
> **Important:** For gRPC specifically, this refers only to unary requests. Streaming requests, **IN ADDITION** to the specified metrics, **must** also comply with the metrics for the pub/sub block.

---

> [!WARNING]
>
> **Important:** metrics for APIs **must** be implemented not only on the server, **BUT ALSO ON THE CLIENT-SIDE APIS!**

---

- [ ] Metrics **must** be implemented for every API the service provides, including system ones, gRPC, and REST APIs, **as well as for the metrics endpoint itself**.
- [ ] Metrics **must** be implemented not only for the server but also for **EVERY** gRPC/HTTP API client.
- [ ] Labeling **must** be done by:
  - [ ] gRPC methods/HTTP endpoints (`/users/{user_id}/profile`, `package.v1.UserService.GetProfile`, etc.)
  - [ ] For REST — additional labeling by HTTP methods (GET, POST, PUT, DELETE, etc.)
  - [ ] Status codes (2xx, 4xx, 5xx, CODE_OK, CODE_NOT_FOUND, etc.)
- [ ] Summarized metrics (`_total`) **should not** be included.
- [ ] The metrics themselves **must** include the following:
  - [ ] Request Count — `counter` (total number of requests)
  - [ ] Request Latency — `histogram` (request processing time, standard buckets)
  - [ ] Request Size — `histogram` (request size, custom buckets depending on the method's logic, NOT LOGARITHMIC!)
  - [ ] Response Size — `histogram` (response size, custom buckets depending on the method's logic, NOT LOGARITHMIC!)
  - [ ] Retry Attempts — `counter` (number of retry attempts, only for idempotent requests; requests must be marked in the API schema)

#### Database Metrics

- [ ] Metrics **must** be implemented for the database to which the service connects.
- [ ] Global metrics **must** include the following:
  - [ ] Connection pool — `gauge` (number of active connections, number of free connections)
  - [ ] Active Transactions — `gauge` (number of active transactions)
- [ ] Labeling **must** be done by:
  - [ ] QueryID (for SQL/NoSQL — the name of the query, for Redis — the name of the command OR the name of the Lua script)
  - [ ] Execution status (error code/category, if applicable)
- [ ] Labeled metrics **must** include the following:
  - [ ] Total Queries — `counter` (total number of queries, labeled by QueryID)
  - [ ] Queries Latency — `histogram` (query execution time, labeled by QueryID, standard buckets)
  - [ ] If the DB uses transactions:
    - [ ] Transactions Total — `counter` (total number of transactions, labeled by QueryID and execution status)
    - [ ] Transactions Latency — `histogram` (transaction execution time, standard buckets)

#### Secrets Management Metrics

> [!WARNING]
>
> **Important:** For metrics related to secrets there is an Audit system: access logs for exact secret objects. **The purpose of metrics is not auditing, but monitoring the availability and performance of the secret management system.** Therefore, metrics should not reveal the substance of the secrets, and there **must not** be any labeling by the secrets themselves.

- [ ] Metrics **must** be implemented for the secret storage to which the service connects (e.g., HashiCorp Vault, AWS Secrets Manager, etc.).
- [ ] Global metrics **must** include the following:
  - [ ] Secrets Fetch Count — `counter` (total number of fetch requests, without separation by secrets)
  - [ ] Secrets Fetch Latency — `histogram` (time to fetch a secret, standard buckets)
