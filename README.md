


# 🗺️ Conceptual Map: Large-Scale Scraping Architecture

### 1. Orchestration and Code Patterns (Scraping Flow)

- **Resilient Pipeline:** Replaces coupled logic with strict sequential steps (setup -> navigate -> bypass -> interact -> extract) using middlewares to inject captcha resolution without breaking the main flow.

- **Selectors Hot-Reload:** Moving CSS/XPath Selectors from the code to **Redis** to allow real-time updates without redeploying pods.

- **AI Auto-Healing (Detective):** If a provider's failure rate increases, a special worker (Vision/LLM) is launched to analyze the broken DOM, deduce the new selector, and update Redis automatically.

### 2. Cache and Network Management (Avoiding Bottlenecks)

- **Initial Problem:** Cache Stampede / Thundering Herd (many workers spinning up and trying to download the same heavy `.js` bundles at once, burning proxies).

- **Solution (Redlock):** Using **Distributed Locks** by file name in Redis. The first worker locks, downloads, and saves. The rest wait and read directly from Redis.

- **Cache Invalidation (Event-Driven):** If a worker detects that a cached file is no longer valid, it deletes it and emits an event via **Redis Pub/Sub** (`cache.invalidated`). Since Node.js is single-threaded, the workers listen to the event and clear their internal RAM/buffer without race conditions.

### 3. Scaling in Kubernetes (KEDA + HPA)

- **Safe Downscaling (Graceful Shutdown):** Kubernetes sends `SIGTERM`. The worker stops accepting messages from RabbitMQ (`channel.cancel()`), finishes the current context, saves to the DB, and executes `process.exit(0)`. Zero half-finished jobs.

- **Synchronous Scaling (Prometheus Scaler):** Do not scale by queue size (it is reactive and hurts latency). Scale by measuring the **Average Saturation** of the workers (e.g., % of active contexts vs maximum limit). Target: 70%.

- **Asynchronous Scaling (Demand-Driven Scaling):** To respect limits and avoid wasting resources. Metric calculated in Redis: `Total = SUM( MIN(user_pending_tasks, user_max_concurrency) )`. Guarantees spinning up only the strictly necessary pods.

### 4. Resilience and Degradation Mitigation (Hedged Requests)

- **Eager Hedged Requests:** If the Gateway detects that the provider's average response time (Grok) has exceeded the threshold (degraded), it switches to proactive mode.

- **RPC Pattern (Scatter-Gather in RabbitMQ):** The Gateway injects 3 identical tasks at once with the same `correlation_id` but a different `attemptId` ("A", "B", "C"). It waits in the `reply_to` queue for the first one to respond.

- **Clean Cancellation (AbortController):** As soon as the Gateway receives the first success, it returns HTTP 200 and emits a `Cancel` event via Pub/Sub. The "losing" workers look up that ID in their internal `Map<id, AbortController>`, execute `.abort()`, send an `ACK` to RabbitMQ to kill the message, and close their browser cleanly.

### 5. CPU and RAM Optimization (Worker Level)

- **Resource Blocking:** Aggressive network interception blocking fonts, images, and telemetry (Analytics) to reduce the load on V8/Chromium.

- **Warm Browsers:** Keeping a single heavy `Browser` instance open and orchestrating isolation through `BrowserContexts` (like incognito tabs) to cheapen startup costs.

- **Periodic Recycling (Memory Leaks):** To mitigate the intrinsic memory leaks of Playwright/Chromium, force a complete restart of the process/browser every "X" successful requests, forcing the Garbage Collector to clean up.
  
  
### 6. "The Scribe" and State Traceability (State Machine)


*   **Concept:** Orchestrator microservice (Audit & Ledger). It is the **only one** with write permissions (INSERT/UPDATE) in the Postgres database. It works 100% by listening to events (Pub/Sub or RabbitMQ) linked to a `Trace ID`.
*   **The Lifecycle and who publishes each state:**
    1.  **`QUEUED` (Asynchronous Only):** Published by the **Gateway** right before dropping the task into the asynchronous RabbitMQ queue.
    2.  **`PROCESSING`:** Published by the **Worker** at the exact millisecond it picks up the message from the queue (both synchronous and asynchronous) and starts opening the browser.
    3.  **`COMPLETED` / `ERROR`:** Published by the **Worker** upon finishing the extraction, attaching the success JSON or the error log (reason for failure, timeout, etc.).
*   **Business Value:** We have an immutable history. If a client complains, we look up the `Trace ID` and see exactly in which millisecond it changed state or why it failed.


### 7. Credit Charging Strategy ("Dual Write" Mitigation)


*   **Business Rule:** The charge is issued *just before* the network delivery, to ensure we don't give away the scraping for free if there's a micro-disconnection. The Scribe catches this event and deducts it in Postgres.
*   **Charging on Synchronous Request (The Gateway):**
    *   The Worker finishes and passes the JSON to the Gateway.
    *   The Gateway publishes the charging event to a durable RabbitMQ queue: *"Charge 1 credit to this TraceID"*.
    *   *Immediately after*, the Gateway sends the `res.send(200)` to the client with the JSON.
*   **Charging on Asynchronous Request (The Notifier):**
    *   The Worker finishes and saves the result.
    *   The **Notifier** picks it up to send it via Webhook.
    *   The Notifier publishes the charging event in RabbitMQ.
    *   *Immediately after*, the Notifier makes the `POST` request to the client's server. If the client's server returns a 500 Error or is down, **the credit is already charged**. Our job was done successfully; the Notifier will retry (*Exponential Backoff*), logging each failed attempt to justify the charge if the client complains.

### 8. Concurrency Control and Auto-Cleanup (Redis ZSETs)


*   **Concept:** Preventing a client from saturating the synchronous system. Managed in ultra-fast memory using Redis **Sorted Sets (ZSET)**.
*   **ZSET Structure:** There is a ZSET for each client. The `Member` is the request's `Trace ID`, and the `Score` is the exact entry **Timestamp**.
*   **Self-Healing Algorithm (Lock Cleanup):** 
    *   Problem: If a worker crashes catastrophically and doesn't report that it has finished, the request gets stuck, "blocking" a user's concurrency slot forever (*Stuck Lock*).
    *   Solution: Every time the Gateway validates if the user has a free slot, it first executes a `ZREMRANGEBYSCORE` command. This automatically deletes any `Trace ID` with a timestamp older than our maximum SLA (e.g., older than 3 minutes).
    *   Result: Automatic and silent cleanup. We count how many elements remain in the ZSET; if it's less than the client's limit, we let them through and add their new `Trace ID`.
