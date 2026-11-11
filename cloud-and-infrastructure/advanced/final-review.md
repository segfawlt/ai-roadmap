# Review & E2E Debugging

**Unit 7:** Review & E2E Debugging **Topic Code:** MS-U7-T1 **Reading Time:** ~40 minutes

------------------------------------------------------------------------

## Learning Objectives

- Review the complete microservices architecture built throughout the course.

- Apply the three pillars of observability (Logs, Traces, Metrics) in a holistic debugging scenario.

- Trace a request’s lifecycle from the API Gateway to the database and back using Jaeger.

- Correlate traces from Jaeger with detailed logs in Kibana using a shared `trace_id`.

- Diagnose performance bottlenecks and errors by analyzing data from Grafana, Jaeger, and Kibana in concert.

------------------------------------------------------------------------

## Section 1: Concept/overview

### 1.1 Introduction

In the world of monolithic architecture, debugging is relatively simple. You have a single codebase, a single log file (or a few files), and can use a debugger to follow every execution step of a request. Everything sits inside a single “box”. However, when moving to microservices architecture, we trade this simplicity for scalability, flexibility, and team independence. A user request is no longer handled by a single application, but travels through a chain of services, each with its own codebase, logs, and even database.

The problem arises immediately: “What happens when a request fails or runs slowly?”. Is the error in the API Gateway, the authentication service, the product service, or the payment service? Or does the problem lie in the network connecting them? Searching through dozens of separate log files to piece together the story of a single request is like looking for a needle in a haystack. This is the problem that End-to-End (E2E) Debugging and the concept of **Observability** were born to solve. Instead of “guessing,” we equip the system with the ability to “tell its story,” allowing us to observe, ask questions, and understand its internal state from the data it emits.

### 1.2 Formal Definition

**End-to-End (E2E) Debugging** in microservices is the process of monitoring, analyzing, and diagnosing the processing flow of a single request across its entire journey, from initialization by the client, passing through services, databases, message queues, until returning the final response.

This process cannot be performed effectively without **Observability**. Observability is not merely monitoring. Monitoring tells you *when* the system has a problem (e.g., CPU usage 90%), while Observability helps you answer *why* the system has a problem. It is built upon three main pillars (The Three Pillars of Observability):

1.  **Logs:** Record discrete events with timestamps. Logs provide the most detailed information about what happened at a specific moment in a specific service. Example: “User ‘alice’ logged in successfully at 2023-10-27T10:00:00Z”.

2.  **Metrics:** Numerical data aggregated over time, providing an overview of system health and performance. Example: number of requests per second, error rate, average latency.

3.  **Traces:** Record the entire journey of a request as it passes through multiple services. A trace is made up of multiple **spans**, each representing a unit of work within a service (e.g., an API call, a database query). Traces are the most powerful tool for E2E Debugging.

By combining all three pillars, we can build a complete picture, from detecting issues at a high level (Metrics), identifying the problematic service (Traces), and finally finding the root cause in the code (Logs).

### 1.3 Analogy

Imagine you are investigating a lost package case in a complex logistics system.

- **Metrics** are like the logistics company’s summary report. It tells you: “Today, 98% of packages were delivered on time”, “Loss rate at hub X is 0.5%”, “Average processing time at warehouse Y is 2 hours”. Metrics help you detect “hotspots” (e.g., why is warehouse Y processing unusually slowly?).

- **Traces** are like the security camera system tracking the journey of *a specific package*. It shows you: “Package A was received at the post office at 9 AM, arrived at hub X at 11 AM, left warehouse Y at 3 PM, and was handed to the shipper at 4 PM”. If the package was stuck at warehouse Y for 4 hours, the trace will indicate exactly that. This is E2E tracking.

- **Logs** are the detailed notebooks of each employee at each point. When the trace shows the package stuck at warehouse Y, you look at warehouse Y’s logs during that time. You might see notes like: “13:05 - Barcode scanning system error”, “14:10 - Unexpected power outage”, “14:30 - Employee Z reported package A wrapper torn”. Logs provide detailed context to explain *why* the delay happened.

Only by combining all three can you go from “loss rate increased” (metric) to “package A stuck at warehouse Y” (trace) and finally “due to barcode system error” (log) to thoroughly resolve the problem.

------------------------------------------------------------------------

## Section 2: Core Components

### 2.1 Architecture overview

To perform E2E Debugging, we need an “Observability Stack”. Below is the reference architecture we have built throughout the course.

``` mermaid
        flowchart TD
    Client --> GW["API Gateway"]
    GW --> Order["Order Service"]
    Order --> Payment["Payment Service"]

    Order --> OTel["OpenTelemetry Collector / Agents"]
    Payment --> OTel
    GW --> OTel

    OTel -- Traces --> Jaeger
    OTel -- Metrics --> Prometheus
    OTel -- Logs --> ES["Elasticsearch"]

    Grafana --> Jaeger
    Grafana --> Prometheus
    Kibana --> ES
    Dev["Developer / SRE"] --> Grafana
```

Workflow:

1.  **Request Flow**: The Client sends a request to the API Gateway. The Gateway routes to microservices (e.g., Order Service, Payment Service).

2.  **Data Generation**: Each service is equipped with the OpenTelemetry SDK. This SDK automatically “instruments” HTTP calls, DB queries, etc., to generate Traces, Metrics, and Logs.

3.  **Data Collection**: OpenTelemetry Collector (or agent) collects data from services.

4.  **Data Storage & Processing**:

- **Traces** are sent to **Jaeger** for storage and querying.

- **Metrics** are scraped and stored by **Prometheus** as time-series data.

- **Logs** (usually JSON formatted) are pushed to **Elasticsearch** via a log forwarder like Fluentd/Logstash.

1.  **Data Visualization & Analysis**:

- **Grafana** connects to Prometheus to graph metrics and to Jaeger to display traces.

- **Kibana** provides a powerful interface to search, analyze, and visualize logs from Elasticsearch.

- **Jaeger UI** is specialized for deep analysis of distributed traces.

### 2.2 Key Components

\*\*Component 1: Structured Logs & `trace_id**`

- **Definition:** Logs written in a machine-readable format (like JSON) instead of plain text. Each log line contains key-value pairs, making search and filtering extremely efficient. Most importantly, every log line related to a specific request must contain the `trace_id` of that request.

- **Role:** The `trace_id` acts like a “foreign key,” allowing us to link dozens or hundreds of log lines from multiple different services together, all belonging to the same request.

- **Syntax (Example JSON Log in Kibana):**

``` json
{
  "@timestamp": "2023-10-27T10:15:30.123Z",
  "log.level": "INFO",
  "message": "Processing payment for order",
  "service.name": "payment-service",
  "trace_id": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6",
  "span_id": "f1e2d3c4b5a6f7e8",
  "order_id": "ORD-98765",
  "amount": 99.99,
  "currency": "USD"
}
```

**Component 2: Distributed Traces (Jaeger)**

- **Definition:** A trace represents the entire journey of a request. It is a collection of **spans**.

- **Trace**: The entire request flow, identified by a unique `trace_id`.

- **Span**: A unit of work within a service (e.g., processing a function, calling another API, querying DB). Each span has its own `span_id`, a common `trace_id`, and can have a `parent_span_id` to form a parent-child relationship tree.

- **Role:** Helps visualize the “Gantt chart” of a request. We can see which service calls which service, how long each step takes, and where the performance bottlenecks are.

- **Syntax (Example query in Jaeger UI):** Based on the interface, you can filter by `service.name`, `operation.name`, `tags` (e.g., `http.status_code=500`), and time.

**Component 3: Application Metrics (Prometheus & Grafana)**

- **Definition:** Indicators measuring application performance and health. Important metrics often follow the **RED** standard:

- **Rate**: Number of requests per second.

- **Errors**: Rate of failed requests.

- **Duration**: Request processing time (latency), usually measured in percentiles like p50, p95, p99.

- **Role:** Provides a 30,000-foot overview. The Grafana dashboard is the first place you look to know “Is something wrong” (e.g., error rate spiking).

- **Syntax (Example PromQL in Grafana):**

```
# Calculate the 99th percentile latency for order-service over the last 5 minutes
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket{service="order-service"}[5m])) by (le))
```

### 2.3 Comparison of Approaches

| Approach | Pros | Cons | When to use |
|----|----|----|----|
| **Logs Only** | Easy to implement initially; Detailed, specific. | Hard to correlate between services; Hard to visualize request flow; High storage cost. | Debugging specific issues within a service *after* identifying that service has a problem. |
| **Metrics Only** | Good overview; Storage efficient; Good for alerting. | Lack of detail; Doesn’t show root cause; Hard to debug a single request. | Monitoring system health, detecting abnormal trends, establishing SLOs/SLIs. |
| **Traces Only** | Perfect for understanding request flow; Easy to identify bottlenecks. | Can be resource-intensive (overhead); Sampling might miss rare errors. | Diagnosing latency issues and errors in distributed systems. |
| **Combine All Three (Observability)** | Provides full picture, from overview to detail; Accelerates diagnosis and resolution. | Requires more complex setup and integration; Needs new culture and skills in the team. | Any microservices system operating in production. This is the gold standard. |

------------------------------------------------------------------------

## Section 3: Implementation

### Level 1 - Basic (Beginner)

**Goal:** Add `trace_id` to a Python (Flask) service’s logs to allow searching all logs for a request.

We will create a simple middleware to generate a `trace_id` if it doesn’t exist and put it into the request context. Then, we configure the logger to automatically add this `trace_id` to every log line.

``` python
# app.py
import logging
import uuid
from flask import Flask, request, g

# 1. Custom logging formatter to include our trace_id
class RequestFormatter(logging.Formatter):
    def format(self, record):
        # Check if we have trace_id in the global context 'g'
        record.trace_id = g.get('trace_id', 'no-trace-id')
        return super().format(record)

# 2. Setup Flask app and logging
app = Flask(__name__)
handler = logging.StreamHandler()
# Add trace_id to our log format
formatter = RequestFormatter(
    '[%(asctime)s] [%(levelname)s] [%(service_name)s] [%(trace_id)s] %(message)s'
)
handler.setFormatter(formatter)
# Add a filter to inject service_name
handler.addFilter(lambda record: setattr(record, 'service_name', 'order-service') or True)
app.logger.addHandler(handler)
app.logger.setLevel(logging.INFO)


# 3. Create a middleware to inject trace_id for each request
@app.before_request
def before_request_func():
    # Get trace_id from header or generate a new one
    trace_id = request.headers.get('X-Request-ID') or str(uuid.uuid4())
    # Store it in Flask's global context 'g' for this request
    g.trace_id = trace_id


@app.route('/order')
def create_order():
    user_id = request.args.get('user_id')
    if not user_id:
        app.logger.warning("Request received without user_id")
        return "user_id is required", 400

    app.logger.info(f"Creating order for user_id: {user_id}")
    # ... business logic here ...
    app.logger.info(f"Order for user_id: {user_id} created successfully")
    return {"status": "success", "trace_id": g.trace_id}

if __name__ == '__main__':
    app.run(port=5001)
```

**How to run and check:**

1.  Run the file `python`` ``app.py`.

2.  Send request: `curl`` ``"http://localhost:5001/order?user_id=123"`

3.  Send another request with custom `trace_id`: `curl`` ``-H`` ``"X-Request-ID:`` ``my-custom-trace-id"`` ``"http://localhost:5001/order?user_id=456"`

**Expected Output (in terminal):**

```
[2023-10-27 11:30:05,123] [INFO] [order-service] [a1b2c3d4-e5f6-a7b8-c9d0-e1f2a3b4c5d6] Creating order for user_id: 123
[2023-10-27 11:30:05,124] [INFO] [order-service] [a1b2c3d4-e5f6-a7b8-c9d0-e1f2a3b4c5d6] Order for user_id: 123 created successfully
[2023-10-27 11:30:15,456] [INFO] [order-service] [my-custom-trace-id] Creating order for user_id: 456
[2023-10-27 11:30:15,457] [INFO] [order-service] [my-custom-trace-id] Order for user_id: 456 created successfully
```

*You will see all logs of the same request sharing a common `trace_id`.*

**Common Errors:**

- **Error:** `AttributeError:`` ``'g'`` ``object`` ``has`` ``no`` ``attribute`` ``'trace_id'`.

- **Reason:** Middleware `before_request` hasn’t run or is misconfigured, causing `g.trace_id` to not be set when logger tries to access it.

- **Fix:** Ensure decorator `@app.before_request` is placed correctly and function is executed. Add default value when calling `g.get('trace_id',`` ``'default-value')` to avoid error.

### Level 2 - Intermediate

**Goal:** Propagate `trace_id` (context propagation) from `Order`` ``Service` to `Payment`` ``Service`.

We will extend the previous example. `Order`` ``Service` will call an API of `Payment`` ``Service`, and we need to ensure `trace_id` is passed via HTTP Header.

**`order_service.py` (modified from above)**

``` python
# order_service.py
# ... (logging setup is the same as Level 1) ...
import requests

@app.route('/order')
def create_order():
    user_id = request.args.get('user_id')
    app.logger.info(f"Creating order for user_id: {user_id}")

    # Propagate the trace_id to the downstream service
    headers = {'X-Request-ID': g.trace_id}
    try:
        app.logger.info("Calling Payment Service to process payment")
        # Call payment service, passing the header
        payment_response = requests.get(
            'http://localhost:5002/pay',
            headers=headers,
            params={'order_id': 'ORD123'}
        )
        payment_response.raise_for_status() # Raise exception for 4xx/5xx status
        app.logger.info("Payment successful")
    except requests.exceptions.RequestException as e:
        app.logger.error(f"Payment service call failed: {e}")
        return {"status": "error", "message": "payment failed"}, 500

    app.logger.info(f"Order for user_id: {user_id} created successfully")
    return {"status": "success", "trace_id": g.trace_id}
```

**`payment_service.py` (new service)**

``` python
# payment_service.py
# ... (logging setup is the same as Level 1, but change service_name to 'payment-service') ...
# Note: The logging setup code needs to be copied here as well.
# Make sure to change 'order-service' to 'payment-service' in the filter.
# (For brevity, the setup code is omitted here)

@app.route('/pay')
def process_payment():
    order_id = request.args.get('order_id')
    # The middleware `before_request` will automatically extract 'X-Request-ID'
    app.logger.info(f"Processing payment for order_id: {order_id}")
    # ... payment logic ...
    app.logger.info(f"Payment for order_id: {order_id} processed")
    return {"status": "payment_processed"}
```

**How to run and check:**

1.  Run `payment_service.py` on one terminal: `python`` ``payment_service.py`` ``--port`` ``5002`.

2.  Run `order_service.py` on another terminal: `python`` ``order_service.py`` ``--port`` ``5001`.

3.  Send request to `Order`` ``Service`: `curl`` ``"http://localhost:5001/order?user_id=123"`.

**Expected Output (combined from both terminals):**

```
# Terminal 1: order-service
[2023-10-27 12:00:10,100] [INFO] [order-service] [b1c2d3e4-f5g6-h7i8-j9k0-l1m2n3o4p5q6] Creating order for user_id: 123
[2023-10-27 12:00:10,101] [INFO] [order-service] [b1c2d3e4-f5g6-h7i8-j9k0-l1m2n3o4p5q6] Calling Payment Service to process payment
[2023-10-27 12:00:10,250] [INFO] [order-service] [b1c2d3e4-f5g6-h7i8-j9k0-l1m2n3o4p5q6] Payment successful
[2023-10-27 12:00:10,251] [INFO] [order-service] [b1c2d3e4-f5g6-h7i8-j9k0-l1m2n3o4p5q6] Order for user_id: 123 created successfully

# Terminal 2: payment-service
[2023-10-27 12:00:10,150] [INFO] [payment-service] [b1c2d3e4-f5g6-h7i8-j9k0-l1m2n3o4p5q6] Processing payment for order_id: ORD123
[2023-10-27 12:00:10,151] [INFO] [payment-service] [b1c2d3e4-f5g6-h7i8-j9k0-l1m2n3o4p5q6] Payment for order_id: ORD123 processed
```

*Success! `trace_id` `b1c2...` appeared in logs of both services, allowing us to piece together the story perfectly.*

### Level 3 - Advanced

**Goal:** Use OpenTelemetry to automatically generate and propagate Traces, while integrating `trace_id` into Logs and creating custom Metrics.

This is the professional and production-ready way. We no longer manage `trace_id` manually but let the library handle it.

``` python
# advanced_app.py
import logging
from flask import Flask, request
import requests

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator

# 1. --- OpenTelemetry Setup ---
# Setup provider and processor. We'll export to console for this example.
# In production, you would use OTLPSpanExporter to send to Jaeger/Collector.
provider = TracerProvider()
processor = BatchSpanProcessor(ConsoleSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

# Get a tracer for this service
tracer = trace.get_tracer(__name__)

# 2. --- Flask App Setup ---
app = Flask(__name__)

# Instrument Flask to automatically create spans for each request
FlaskInstrumentor().instrument_app(app)
# Instrument requests library to propagate context and create spans for outgoing calls
RequestsInstrumentor().instrument()

# 3. --- Logging Setup with Trace/Span ID injection ---
def get_trace_id():
    # Helper to get current trace_id from the context
    span = trace.get_current_span()
    if span and span.get_span_context().is_valid:
        return format(span.get_span_context().trace_id, '032x')
    return 'no-trace-id'

# Custom formatter to add trace_id to logs
# (This is similar to Level 1, but gets trace_id from OpenTelemetry)
# ... logging setup code using get_trace_id() ...

@app.route('/order')
def create_order_advanced():
    # A new span will be automatically created for this request by the instrumentor
    current_span = trace.get_current_span()

    user_id = request.args.get('user_id')
    current_span.set_attribute("user.id", user_id) # Add custom attribute to span

    app.logger.info("Starting order creation")

    # This HTTP call will also be automatically traced
    # The trace context (trace_id) is propagated automatically!
    requests.get('https://api.example.com/inventory/check', params={'item': 'book'})

    # Create a custom inner span for a specific business logic
    with tracer.start_as_current_span("process_internal_logic") as logic_span:
        logic_span.add_event("Starting complex calculation")
        # ... do some work ...
        logic_span.add_event("Finished complex calculation")

    app.logger.info("Order created")
    return {"status": "created", "trace_id": get_trace_id()}


if __name__ == '__main__':
    # ... Add logging setup here for brevity ...
    app.run(port=5001)
```

**Expected Output (in terminal, from ConsoleSpanExporter):** You will see JSON output describing the generated spans.

```
{
    "name": "GET /order",
    "context": {
        "trace_id": "0x4a1db...e4c",
        "span_id": "0x2b...f9",
        ...
    },
    "kind": "SpanKind.SERVER",
    "parent_id": null,
    "start_time": "...",
    "end_time": "...",
    "attributes": {
        "http.method": "GET",
        "http.route": "/order",
        "user.id": "123"
    },
    ...
    "children": [
        {
            "name": "GET",
            "context": { "trace_id": "0x4a1db...e4c", "span_id": "0x3c...a1", ... },
            "parent_id": "0x2b...f9",
            "attributes": { "http.url": "https://api.example.com/inventory/check", ... }
        },
        {
            "name": "process_internal_logic",
            "context": { "trace_id": "0x4a1db...e4c", "span_id": "0x4d...b2", ... },
            "parent_id": "0x2b...f9",
            "events": [ { "name": "Starting complex calculation" }, ... ]
        }
    ]
}
```

*This output when sent to Jaeger will be visualized as a waterfall chart, showing span `GET`` ``/order` is parent of span `GET` (calling inventory) and span `process_internal_logic`.*

------------------------------------------------------------------------

## Section 4: Best Practices

### ✅ DO’s - Recommended

| Practice | Why | Example |
|----|----|----|
| **Use Structured Logging (JSON)** | Allows logs to be queried, filtered, and aggregated automatically by tools like Kibana. Structured data is easier to analyze than plain text. | `{"level":`` ``"error",`` ``"message":`` ``"DB`` ``connection`` ``failed",`` ``"db.host":`` ``"pg.prod"}` instead of `ERROR:`` ``DB`` ``connection`` ``failed`` ``to`` ``pg.prod`. |
| **Use OpenTelemetry** | It is an open standard, supported by a large community. Provides SDKs and automatic instrumentations for many languages/frameworks, reducing setup effort. | `FlaskInstrumentor().instrument_app(app)` automatically creates traces for every Flask endpoint. |
| **Add context to Spans and Logs** | Adding business attributes (`user_id`, `order_id`, `tenant_id`) to traces and logs makes debugging much faster. | `current_span.set_attribute("order.id",`` ``order_id)` |
| **Configure Sampling in Production** | Tracing 100% requests at high traffic creates large overhead. Sampling (e.g., 1-10% traffic) allows collecting enough data for analysis without impacting performance. | Use `TraceIdRatioBasedSampler(0.05)` in OpenTelemetry to sample 5% of requests. |

### ❌ DON’Ts - Avoid

| Anti-pattern | Consequence | How to avoid |
|----|----|----|
| **Log sensitive info (PII)** | Leaking user personal data (passwords, credit cards, emails), violating security regulations like GDPR. | Filter and encrypt sensitive fields before logging. Use log scanning tools to detect PII. |
| **Create Metrics with High Cardinality** | Putting values with too many variations (like `user_id`, `request_id`) into metric labels will “explode” the number of time series, overloading Prometheus. | Only use labels with finite values (e.g., `http_method`, `status_code`, `endpoint`). Use logs/traces for unique IDs. |
| **Rely on a single pillar** | Looking only at logs won’t show bottlenecks. Looking only at metrics won’t reveal causes. Looking only at traces lacks detail. | Build a culture of using Grafana, Jaeger, and Kibana simultaneously during troubleshooting. |
| **Log too much or too little** | Too little log gives insufficient info to debug. Too much log (especially DEBUG level in production) wastes storage and obscures important info. | Set appropriate log levels per environment. Use INFO for key events, WARN/ERROR for issues needing attention. |

### 🔒 Security Considerations

- **PII Leakage**: Absolutely do not put sensitive info like passwords, API keys, tokens, personal info into logs or span attributes. Mask or remove them.

- **Access Control**: Observability stack UIs (Grafana, Kibana, Jaeger) must be protected by authentication and authorization. Do not expose them to the Internet without protection.

- **Injection Attacks**: Be careful when logging user input. Although less common, improperly formatted log strings can cause display issues or even vulnerabilities if processed by another tool.

### ⚡ Performance Tips

- **Asynchronous Exporting**: Always use asynchronous “exporters” (e.g., `BatchSpanProcessor`) so telemetry data sending doesn’t block the application’s main thread.

- **Sampling**: In high traffic environments, apply sampling. Common strategies are `tail-based`` ``sampling` (decide to keep trace after completion, based on error presence) or `ratio-based`` ``sampling` (keep a certain percentage).

- **Collector Architecture**: Deploy OpenTelemetry Collector as an agent on each node or as a centralized gateway to offload the application and optimize data sending.

------------------------------------------------------------------------

## Section 5: Case Study

### 5.1 Scenario

**Company/Project:** “E-Shop Now”, a growing e-commerce site. **Requirements:** Payment system must successfully process 99.9% of requests with latency under 500ms (p99). **Constraints:** System built on microservices architecture including `Frontend`` ``Gateway`, `Order`` ``Service`, `Payment`` ``Service`, and `Notification`` ``Service`. Recently, the team received many user complaints about the checkout page “spinning” for a long time, occasionally reporting errors.

### 5.2 Problem Analysis

Support team confirmed the issue but didn’t know the root cause.

- **Hypothesis 1 (Dev A):** “Maybe `Payment`` ``Service` database is slow.”

- **Hypothesis 2 (Dev B):** “Maybe `Notification`` ``Service` sending confirmation emails timed out.”

- **Hypothesis 3 (SRE):** “I see `Order`` ``Service` CPU spiking occasionally, could be that.”

Instead of guessing, the team decided to use the observability stack to investigate systematically.

### 5.3 Solution Design

3-step investigation plan, using 3 pillars:

1.  **Step 1: The “What” (Metrics - Grafana):** Confirm the issue and impact scope. View overview dashboard to answer: “Is the problem real? Where and when does it happen?”.

2.  **Step 2: The “Where” (Traces - Jaeger):** Find a specific slow or failed request. Analyze its trace to identify exactly which service and operation is the “bottleneck”.

3.  **Step 3: The “Why” (Logs - Kibana):** With `trace_id` from Jaeger, search all related logs in Kibana to read the detailed story and find the root cause.

### 5.4 Implementation

**Step 1: Analyze Metrics on Grafana**

Team opens “Service Health” dashboard. They see two abnormal graphs:

- `Payment`` ``Service` endpoint `/charge` “API Latency (p99)” graph skyrocketed from ~200ms to ~3000ms.

- Same endpoint’s “Error Rate” graph also increased from 0.1% to 5%.

**Conclusion:** Problem is definitely in `Payment`` ``Service`. Dev B and SRE hypotheses are less likely.

**Step 2: Tracing on Jaeger**

Team goes to Jaeger UI, filters traces for `payment-service` with `operation` `POST`` ``/charge`, `status=error`, and `duration` \> 2s. They immediately find many traces like this:

```
payment-service: POST /charge (3050ms)
├── validate-request (10ms)
├── get-user-data (50ms)
├── call-external-psp (3000ms)  <-- BOTTLENECK
|   └── ... (timeout error)
└── save-transaction-record (5ms)
```

Trace shows almost entire time (3000ms) consumed by span `call-external-psp` (calling third-party payment provider). This span ends with tag `error=true`.

**Conclusion:** Bottleneck is the API call to payment partner. Dev A’s hypothesis (DB) is eliminated.

**Step 3: Analyze Logs on Kibana**

Team copies `trace_id` from slow trace on Jaeger (e.g., `e8a9f0b1c2d3e4f5...`). They paste it into Kibana search bar with query:

```
trace.id: "e8a9f0b1c2d3e4f5..."
```

Result returns a sequence of logs from `Payment`` ``Service` for that request:

``` json
{"@timestamp": "...", "service.name": "payment-service", "trace.id": "e8a9...", "message": "Starting payment charge for order ORD-456"}
{"@timestamp": "...", "service.name": "payment-service", "trace.id": "e8a9...", "message": "Calling external PSP API at https://psp.vendor.com/v2/charge"}
{"@timestamp": "...", "service.name": "payment-service", "trace.id": "e8a9...", "log.level": "ERROR", "message": "Failed to call PSP API", "error.details": "ConnectTimeout: HTTPSConnectionPool(host='psp.vendor.com', port=443): Max retries exceeded with url: /v2/charge (Caused by ConnectTimeoutError(<urllib3.connection.HTTPSConnection object at 0x...>, 'Connection to psp.vendor.com timed out. (connect timeout=3)'))"}
```

Last log line pointed out the root cause: `ConnectTimeoutError`. Their service could not connect to partner API within 3 seconds.

### 5.5 Results & Lessons Learned

- **Root Cause:** Payment partner (PSP) API performance degraded, causing timeout.

- **Immediate Action:** Team temporarily switched to backup payment provider and contacted main partner to report incident.

- **Metrics Improvement:**

- Latency p99 of endpoint `/charge` returned to \< 200ms.

- Error rate dropped to near 0%.

- **Lessons Learned:**

1.  **Observability is mandatory:** Without this stack, the team could have lost days or weeks finding the issue, severely impacting revenue and reputation.

2.  **Importance of `trace_id`:** `trace_id` is the red thread connecting everything. It allows seamless transition from “what” (metric) to “where” (trace) and “why” (log).

3.  **Set Reasonable Timeouts:** Must have strict timeouts for external calls and fallback mechanisms so system doesn’t “hang” with partner.

------------------------------------------------------------------------

## References

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)

- [Jaeger Tracing Documentation](https://www.jaegertracing.io/docs/1.41/)

- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)

- [Elasticsearch & Kibana Guide](https://www.elastic.co/guide/index.html)

- [SRE Google Book - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
