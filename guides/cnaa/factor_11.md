# Factor XI. Logging

[![CC BY-ND 4.0][cc-by-nc-nd-shield]][cc-by-nc-nd]

> [!IMPORTANT] 
>
> **Principle:** Logging is first and foremost about **events**, not just text strings. Logs must be structured, immutable, and timestamped. They must contain sufficient context to understand the event that occurred, but without being redundant.

<p align="center"><b>Logs are not just strings for terminal — it's a full-fledged event system.</b></p>

Each log entry is a structured, immutable, and timestamped description of an event that occurred in the system. It is not just a `printf` for debugging. It is a "black box", the flight recorder of your service. You must treat its design as seriously as API design.

Each log event **must** contain comprehensive context, yet not too redundant. Messages like "Error processing request" are more harmful than helpful. Each entry must be a self-contained document that does not depend on other logs, and which fully and holistically describes the event that occurred in the system.

## Log details

Logs require a certain minimal context that must be present in every entry. Field names are intentionally omitted, as the key point is to focus on the content.

### Minimal set of fields

- **Timestamp**: (Obvious but important) Exact timestamp in ISO 8601 format with milliseconds and timezone **ONLY IN UTC** (e.g., 2025-06-20T21:58:40.123Z).
- **Log level**: (Also quite obvious) The importance level of the event. We will discuss the semantics of levels below.
- **Event identifier**: (**Most important key!**) A unique, stable identifier of the event type itself. Not to be confused with trace ID. For example: `user.authentication.success`, `database.query.failed`, `payment.provider.request.sent`. The application defines the list of available events by itself and their identifiers. This allows easy filtering of events by type and quickly finding the necessary events in the logs.
  - **Why?** Reliable metrics and alerts are built on this field (`count(event_type='database.query.failed'`)). It does not change from message to message, unlike `message`.
- **Service name and version**: (`users-api@1.2.5-<commit-hash>` or something else) Who generated the event. Critically important in a microservice environment. This is effectively the source of the event.
- **The message itself**: (mandatory) A human-readable description of the event. It should be brief but informative. Must be in English to avoid internationalization issues. For example: "User authentication successful", "Database query failed: connection timeout".

  Before some time there was a strong belief that the message itself should be the immutable part of the log by which events can be filtered. However, this is not entirely true. The message is primarily intended for humans, not machines. Therefore, it should be brief and understandable, but not necessarily static and not even required to be predictable. Uniqueness is ensured by the event identifier.

- **trace_id & span_id**: identifiers from tracing system. They allow you to find all logs related to a single request that passed through 5 services with one click.
- **host.name / pod.name**: The specific instance that generated the log. Helps when the problem is related to a "sick" node (you can match data by metrics on a specific host and determine the cause of the problem).

Metrics above are mandatory in any case. By meaning, this list resembles CloudEvents — since a log is an event, it must have resource parameters like an event. However, unlike CloudEvents, logs have many other parameters without which their meaning is lost.

### Context data

These fields are added depending on the specific event. They are the "core" of the log.

- **Actor**: Who initiated the action? Usually, this is the identifier of the resource that initiated the event. This could be a user resource, a cron timer resource, an event from Kafka, and so on. Typically, a resource has its own unique identifier that also describes its type (e.g., `types.example.com/users/12345`). It is highly recommended to use resource identifiers rather than just names. This allows for easy filtering of events by both resource type and its identifier, while simultaneously providing an audit log.
- **Object**: What was the action performed on? This is also typically a resource identifier, like the actor, but in this case, it is the object on which the action was performed. For example, if user 12345 deleted post 67890, the actor would be `types.example.com/users/12345`, and the object would be `types.example.com/posts/67890`. If the action is not associated with a specific resource, the field may be optional.

  **Important:** NEVER, under any circumstances, you **should not** log whole object! Just an identifier.
- **Action & Result**: How was the action performed, and what was the result? This is a very heterogeneous field because different actions on different resources can have very different input and output data.

  A classic example is an HTTP request: suppose we have an HTTP API and we receive a request that we want to show in the log. In this case, we need to log the following input data:

  - `method`: The request method (`GET`, `POST`, `PUT`, `DELETE`, etc.)
  - `path`: The request path (e.g., `/v1/users/12345`)

  And the output would be:

  - `status_code`: The HTTP status code of the response (`200`, `404`, `500`, etc.)

  You may notice that not much data is required for an HTTP request. This is because a) there are already traces that allow finding all logs by trace ID, and b) there are already metrics that will tell how long the request was processed and how many requests were processed. Therefore, it is sufficient to simply specify the method, path, and status code in the log.

As a result, an "ideal" log might look like this:

```json
{
  "timestamp": "2025-06-20T21:58:40.123Z",
  "level": "ERROR",
  "service_name": "orders-api@v2.1.0",
  "event_type": "database.query.failed",
  "message": "Failed to update order status in database for order 'ord_9a8b7c6d'",
  "trace_id": "a1b2c3d4e5f6",
  "hostname": "k8s-pod-orders-api-7f7f7f7f7f-abcde",
  "context": {
    "resource": {
      "type": "order",
      "id": "ord_9a8b7c6d"
    },
    "actor": {
      "user_id": "usr_5e4f3g2h"
    },
    "db": {
      "statement": "UPDATE orders SET status = ? WHERE id = ?",
      "host": "prod-db.cluster-ro.internal"
    },
    "error": {
      "type": "database.TimeoutError",
      "message": "Query timeout after 3000ms",
      "stack_trace": "..."
    }
  }
}
```

Of course, the perfect log does not exist, and in reality, logs can be much more complex. But the key thing to remember is: a log is an event, not just a string. It must be complete in information, but not excessively so. Because the rest of the information can be obtained from traces and metrics. **A log is not just debugging information, but a full-fledged tool for monitoring and auditing.**

## Logging levels

The log level defines not "importance", but the **expected reaction to an event**.

The key, and most common, mistake is to use `INFO` and `DEBUG` to log everything that happens in the system. For example, "user logged in" or "API request completed". Of course, these are important events, and they should be recorded somewhere, but for logs, there is the principle **"A log is not a string, it's an event"**. Events need to be **reacted to**, not just recorded.

### `DEBUG`

> [!NOTE]
> "For the developer of this service during debugging".

**When to use**: Logic of branching, variable values, execution tracing within a complex algorithm.

**Rule**: In production, this level should be turned off by default. It **may** be dynamically enabled for a short time on one instance for diagnosing a problem (even in production!). Never alert.

### `INFO`

> [!TIP]
> "The system is operating normally."

**When to use**: Significant lifecycle points: service started/stopped, background task started/completed, **an important but routine business decision was made** (e.g., `payment.processed`).

**Rule**: INFO log is the pulse of the application. There should be enough to understand the overall flow of work, but not too much to create noise. Does not alert.

### `WARN`

> [!WARNING]
>
> "Something unexpected happened, but I handled it.".

**When to use**: Retry connecting to an API, use a fallback value because the primary one is unavailable, detect an outdated API format in a client request.
**Rule**: WARN is a signal of a potential problem or "technical debt" in the system. It says: "The system is stable, but pay attention to me in the future." It can be the basis for dashboards and delayed alerts (rate(warnings) > X).

### `ERROR`

> [!CAUTION]
>
> "I could not complete the operation. Human attention is required."

**When to use**: Failed to connect to the database after all retries, request ended with a 5xx error, critical business logic was not executed.
**Rule**: Every ERROR log is an incident. It must be actionable. If no one can or should respond to the error, it is not ERROR, but WARN or INFO. Almost always, alerts are set up based on ERROR logs. **Must** contain a stack trace.

### `FATAL`

> [!CAUTION]
>
> "I am dying. Farewell, and thanks for all the fish."

**When to use**: Failed to read mandatory configuration at startup, ran out of memory, encountered an irreparable internal state violation.
**Rule**:  Immediately after writing this log, the application must terminate. This is a signal to the orchestrator (Kubernetes) to restart. Always alert.

Since a `FATAL` event is essentially the application's suicide note, its content must be as informative as possible so that the post-mortem investigation can be conducted quickly and efficiently. The goal is to give the engineer who is woken up by an alert at 3 AM all the necessary information to understand the cause of the failure, without forcing them to connect to the machine and dig through the system.

**The philosophy of `FATAL`-log: Post-mortem report, written automatically in advance.**

A `FATAL` level log must be a self-sufficient report of the failure. It answers four key questions:

- **Who crashed?** (Service identification)
- **Why did it crash?** (Specific, irreparable error)
- **What was it doing at the time?** (Execution context)
- **What was its environment?** (State snapshot)

**Group 1: Identification ("Who crashed?").**

These are baseic fields, that must be in any log, but at this level, they are critically important:

- **service name**
- **service version**
- **host/pod name**

**Why?** To know exactly which service instance caused the failure.

**Group 2: Cause of death ("Why did it crash?").**

This is the core of the message. Maximum detail here saves hours of investigation.

- **error type**: (Required) The programmatic error type. Not a string, but the actual class/type name. For example, `ConfigurationError`, `DatabaseConnectionFatalError`, `OutOfMemoryError`.
- **message**: (Required) A human-readable, unambiguous message about the root cause. Not "Error starting application," but "Failed to connect to database: connection refused" or "Failed to bind to port 8080: address already in use".
- **stack trace**: (**Absolutely required**) The full stack trace that led to the error. Without it, debugging is practically impossible.
- **error chain**: (Highly recommended) If the error was "wrapped" multiple times, the log should contain the entire chain. This helps to understand the root cause. Example:

  ```text
  FATAL: Failed to start web server
  Caused by: Failed to initialize database component
  Caused by: Failed to connect to database master node
  Caused by: Invalid database password (authentication failed)
  ```

**Group 3: Execution context ("What was it doing?").**

It is necessary to record at which stage of the lifecycle the failure occurred.

- **error stage**: (Key field for `FATAL`) The stage at which the failure occurred. Possible values:
  - `startup`: Failure during initialization (the most common case for FATAL). For example, failed to read config, connect to database, or bind to network port.
  - `runtime`: Failure during normal operation. For example, out of memory, or a background guardian thread detected an irreparable state violation.
  - `shutdown`: Failure during an attempt to shut down gracefully.
  - any other values, as long as they are not `unknown` or `unspecified`.
- **trace.id**: If the failure occurred in the context of processing a request (which is rarer for `FATAL`, but possible), these IDs are invaluable. They link the crash to a specific trace. If the failure occurred at the `startup` stage, these fields will be empty, and that is normal.

**Group 4: Environment snapshot ("What was its environment?").**

These are blackbox data. What was the world around the process in the last moments of its life?

- **Configuration and its source**: The source from which the configuration was loaded.
  If the error is related to a specific parameter, it should be indicated (with secrets masking!).

  Configuration should also include information about the environment, paths to secrets (but not the secrets themselves, be careful!), feature flags, etc.
- **Last-gasp metrics**: Key indicators of the process state right before exit:
  - `memory.heap.used_bytes`: 512345678 (helps diagnose memory leaks).
  - `process.uptime_ms`: 1250 (shows how quickly the service crashed after startup).
  - `goroutines.count`: (for Golang, as an example. For Java or TypeScript applications runtime fields might be different).

  Metrics are useful regardless of the cause of the crash. They provide context and help understand what was happening in the system at the moment of failure.

**Why write so much in `FATAL` if there is Sentry?**


Error tracking systems (Sentry, Bugsnag, Rollbar) **do not replace** FATAL logs, but **complement** them. The ideal implementation is to **do both**. They serve different purposes and different audiences.

**The ideal scenario: Synergy of two systems.**

 Imagine that at 3 AM your service starts crashing in a loop (`CrashLoopBackOff`).

- 3:05 AM: SRE receives two alerts almost simultaneously:
  - From Sentry: "A new fatal error `DatabaseConnectionRefused` appeared in the `payment-gateway` service of release `v3.4.1`."
  - From the logging system (`Alertmanager`): "High rate of `FATAL` logs for the `payment-gateway` service."
- 3:06 AM (SRE investigation):
  - SRE opens the log dashboard. They see a `FATAL` log. But they also build a graph and see that at the same time the `payment-gateway` crashed, the `active_connections` metric for the `PostgreSQL` database hit the ceiling, and the `user-profile` service experienced a sharp increase in requests.
  - **SRE conclusion:** The problem is not just in the code. It seems that the `user-profile` service started DDoSing the database, causing no free connections left for `payment-gateway`. SRE temporarily limits the rate for `user-profile` and stabilizes the system until a fix is applied.
- 9:00 AM (Developer investigation):
  - The developer opens an incident in Sentry, which was automatically created for them. They don't need to dig through logs. They see the exact line of code where the crash occurs, the full stack trace, and understand that the code lacks handling for the situation when the database connection pool is exhausted.
  - **Developer conclusion:** More elegant error handling is needed (e.g., a `circuit breaker`) so that the service doesn't crash but temporarily stops accepting requests.

Sending `FATAL` events to Sentry is mandatory. But this should be done in addition to writing canonical, detailed logs to a centralized system.

- Sentry — for developers and code debugging.
- Logs — for SRE and overall system state analysis.

Together they create a powerful, layered defense system that allows for the fastest and most effective response to the most critical failures.

## Pretty printing

For logs to be processed by a log collector, they must be structured.

The problem is that if you start developing a system for pretty printing, you'll suddenly find that it involves an incredible number of architectural decisions:

- What format will the logs be?
- Will they be colored?
- What colors will be used?
- Should "alerts" be highlighted?
- Should logs be interactive?
- Should emoji be used in logs?

And this is not the full list of questions that need to be answered. Therefore, in most cases, it's best to use off-the-shelf logging solutions that support structured logs and allow for easy output format customization.

Therefore, the application **should not** support pretty-printing functionality, as this is not its primary task. Instead, it is better to concentrate on making the logs structured and easily processable.

However, the need for beautiful log output does not disappear. So what should be done in this case?

Pretty printing functionality **must** be supported by a separate launcher component if it is shipped with the application. For example, you can use [hl](https://github.com/pamburus/hl), which integrates quite conveniently, albeit at the cost of an additional dependency.

## Tooling

- [hl](https://github.com/pamburus/hl) — pretty printing for json logs (just add `2>&1 | hl -P` after your command and magic will happen 🪄)

## Checklist

- [ ] The application **must** provide a way to enable the debug logging level without restarting the application (**can** use a feature flag).
- [ ] The application **must** log events ONLY in JSON format.
- [ ] The application **must not** handle "pretty-printing" of logs itself. The output must always be a single JSON string for machine processing.
- [ ] Logs **must** be sent only to `stderr`.
- [ ] `event_type` **must** be a unique and immutable event identifier.
- [ ] Each `event_type` **must** be documented in the service so that other teams can rely on it.
- [ ] A single `event_type` **must** be tied to a specific logging level, and the logging level **must** be documented in the list.
- [ ] Logs **must not, under any circumstances,** contain sensitive data (secrets, passwords, PII) in plain text. Such data **must** be completely removed from logs, not just masked.
  - [ ] Logs **must not** contain full authorization tokens or API keys, even if they are temporary.
  - [ ] However, it **is acceptable** to log token metadata (e.g., `key_id` or `jti`) to find it, **but not the token itself.**
- [ ] `WARN`, `ERROR`, and `FATAL` level logs **must** be actionable. That is, they **must** clarify the next step for an engineer who will investigate them. If a log does not make it clear what to do next, it is not a `WARN`, `ERROR`, or `FATAL` log, but `INFO`/`DEBUG`.
- [ ] `INFO` level logs **must** be used only for key lifecycle events (service start/stop, start/end of an important task), not for debugging execution flow in production.
- [ ] `INFO` level logs **must not** create excessive noise. The number of logs should follow the rule of "no more than 6 logs per minute per instance" (or "one log no more frequently than every 10 seconds"). This rule **must** be followed under any service load, including peak hours.

  **Note:** If there are more logs, it means they are not informative and need to be reduced.
- [ ] The application **must** not only write `ERROR` and `FATAL` logs to `stderr` but also send them to a configured error tracking system (e.g., `Sentry`), **if available**.
- [ ] Logs **must** have the following fields:
  - [ ] `timestamp` — timestamp in `ISO 8601` format (yyyy-MM-ddTHH:mm:ss.SSSTZD) **ONLY IN UTC**.
  - [ ] `level` — logging level: only `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL`.
  - [ ] `service_name` — a constant service identifier (**the application name, not the deployment name!**), as well as its version (e.g., `users-api@1.2.5-<commit-hash>`).
  - [ ] `event_type` — IMMUTABLE event identifier. **Must** be exported as a constant in the application to avoid duplicate identifiers, and also **must** be documented with a brief description of the event's essence.
    - [ ] All possible `event_type`s **must** be listed in the service's documentation so other teams and systems can rely on them.
    - [ ] `event_type` **must** adhere to backward compatibility rules. If an `event_type` needs to be changed, a new one must be created, and the old one marked as deprecated.
  - [ ] `message` — a human-readable message, only in English, in any format, but not empty.
  - [ ] `trace_id` — trace identifier, in [W3C Trace Context](https://www.w3.org/TR/trace-context/) format (e.g., `4bf92f3577b34da6a3ce929d0e0e4736`). **Must** be included in the log, but only if it can be found in the execution context. If not, the field **may** be empty.

    **Do not confuse with `trace_parent` and `span_id`.**
  - [ ] `host_name` — the value of the `HOSTNAME` environment variable, OR the attached hostname provided by Service Discovery.
- [ ] Logs **should** have the following fields depending on the context:
  - [ ] `context` — An object in any format that contains the event context. If present, there **must** be a schema clearly tied to the `event_type`.

    Important: If the event type indicates an error, then `context` **is the error structure itself.** However, the context **must not** contain a stack trace.
  - [ ] `actor_name` — The resource that initiated the event. **Must** be in the resource identifier format specified in [Google AIP-122](https://google.aip.dev/122).
  - [ ] `resource_name` — The resource **on which** the action was performed. **Must** be in the resource identifier format specified in [Google AIP-122](https://google.aip.dev/122).
  - [ ] `action` — The action that was performed. It **may** vary depending on the context, but the `event_type` **must** have a fixed format for it.
- [ ] Logs **may** include the following fields:
  - [ ] `span_id` — span identifier, if it exists in the execution context. **Must** be in [W3C Trace Context](https://www.w3.org/TR/trace-context/) format.
  - [ ] `metrics` — An object containing metrics related to the event. For example, operation execution time, number of processed records, etc.

- [ ] Specific log levels have specific requirements:
  - [ ] `DEBUG`
    - [ ] **must not** contain a stack trace (the reason is that correct use of debug would generate an excessive amount of data).
  - [ ] `INFO`
  - [ ] `WARN`
    - [ ] If a `WARN` is related to using a fallback or a retry, the context **should** specify which mechanism was triggered. This helps to understand how often the system operates in a non-standard mode.
  - [ ] `ERROR`
    - [ ] **must** contain a `context` that describes the error context, as well as an `error` field containing information about the error.
  - [ ] `FATAL`
    - [ ] **must** contain a `stack_trace` with the full error stack trace.
    - [ ] `event_type` **must** specify the error type; the log must contain the programmatic error type (e.g., `system.configuration_failure`, `system.out_of_memory`). These types **must** be clearly documented as to when they occur.
    - [ ] The context **must** contain information about the lifecycle stage at which the failure occurred (e.g., `startup`, `runtime`, or `shutdown`).
    - [ ] The context **may** contain key state indicators just before the crash (last-gasp metrics), such as memory usage or process uptime.
- [ ] The log **must not** contain any other fields than those listed above. Any additional fields **must** be documented as a schema associated with the `event_type`.
