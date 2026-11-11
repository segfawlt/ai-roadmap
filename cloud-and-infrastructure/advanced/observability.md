# Observability

**Unit 5:** Microservices Observability **Topic Code:** OBS-501 **Reading Time:** ~40 minutes

------------------------------------------------------------------------

## Learning Objectives

- Define observability and explain why it is fundamentally different from traditional monitoring.

- Identify the three pillars of observability (metrics, logs, traces) and the role each plays in diagnosing production incidents.

- Describe the LGTM stack (Loki, Grafana, Tempo, Mimir/Prometheus) and how its components integrate.

- Use PromQL to express service-level indicators with the RED method (Rate, Errors, Duration).

- Explain what Service Level Objectives (SLOs), error budgets, and burn-rate alerts mean for a production service.

------------------------------------------------------------------------

## Section 1: From Monitoring to Observability

### 1.1 The Problem with Monoliths

In a classic monolithic system, debugging is relatively easy. The entire request lives inside one process, one log file, and one memory space. You can attach a debugger, tail a log, or inspect a thread dump and quickly understand what went wrong.

Once you split that monolith into dozens or hundreds of microservices, the picture changes radically. A single user request may hop across a dozen services, three databases, a message queue, and an external API. When something goes wrong at midnight, the question is no longer “what’s in my log file?” but “which of the 47 services that touched this request actually failed, and in what order?”

**Monitoring** — the classical approach — tells you whether your known failure modes are happening. You define a list of things to check (CPU, memory, HTTP 5xx rate) and alert when thresholds are breached. It works well when you already know what can break.

**Observability** is a stronger property. A system is observable when you can ask *new* questions about its behavior without having to ship new instrumentation. The system’s outputs — metrics, logs, and traces — are rich enough that you can explore novel failure modes interactively, long after the code was written.

### 1.2 The Three Pillars

``` mermaid
        block-beta
    columns 3
    block:header:3
        H["The Three Pillars of Observability"]
    end
    M["Metrics<br>(rate, latency,<br>error count)"]
    L["Logs<br>(structured<br>events)"]
    T["Traces<br>(request flow<br>across services)"]
    M1["That something<br>is wrong"]
    T1["Where it is<br>going wrong"]
    L1["Why it is<br>going wrong"]
```

- **Metrics** — numeric time-series data. Cheap to store, easy to aggregate, great for dashboards and alerting. *Answer: Is something wrong?*

- **Traces** — the path a single request takes through the system, with latency attributed to each span. *Answer: Where in the chain is the problem?*

- **Logs** — detailed event records, usually structured JSON. Expensive at high volume but rich in context. *Answer: What exactly happened?*

A production-ready service needs all three. Metrics without traces means you know something is broken but not where. Traces without logs means you can locate the bad hop but not diagnose why it failed. Logs without metrics means you’re swimming in text with no aggregate picture.

------------------------------------------------------------------------

## Section 2: The LGTM Stack

The open-source observability stack championed by Grafana Labs is commonly abbreviated **LGTM**:

- **L**oki — log aggregation

- **G**rafana — the visualization and alerting UI

- **T**empo — distributed trace storage

- **M**imir (or Prometheus) — metrics storage

All four integrate natively in Grafana so that a single dashboard can show metrics, logs, and traces from the same time range. Crucially, you can click a span in Tempo and jump straight to the matching Loki log lines — the “correlated exemplar” pattern — which turns debugging from hours into minutes.

### 2.1 Prometheus: Metrics

Prometheus uses a **pull model**. Your applications expose a `/metrics` endpoint (typically via a library like `prometheus_client`), and Prometheus scrapes that endpoint on a schedule.

``` yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "orders-service"
    static_configs:
      - targets: ["orders:8080"]
  - job_name: "payments-service"
    static_configs:
      - targets: ["payments:8080"]
```

The four metric types you will actually use:

| Type          | Behavior                 | Example                           |
|---------------|--------------------------|-----------------------------------|
| **Counter**   | Monotonically increasing | `http_requests_total`             |
| **Gauge**     | Point-in-time value      | `db_connections_active`           |
| **Histogram** | Bucketed distribution    | `http_request_duration_seconds`   |
| **Summary**   | Client-side quantiles    | Prefer histograms for aggregation |

### 2.2 Loki: Logs

Loki’s insight is that you don’t need to index every word in every log line — you just need to index *labels* (service name, level, host) and store the raw content compressed in object storage. This makes Loki dramatically cheaper than Elasticsearch for large log volumes.

Query with **LogQL**, which looks like PromQL plus `grep`:

```
# All ERROR logs from the orders service
{app="orders"} |= "ERROR"

# Parse JSON and filter on a nested field
{app="orders"} | json | status_code >= 500

# Rate of 500s per minute, by service
sum by (service) (
  rate({app=~".+"} |= "500" [1m])
)
```

### 2.3 Tempo: Traces

A **trace** is a tree of spans. Each span records a unit of work (an HTTP request handler, a database query, a downstream API call) along with its duration and attributes. Traces are emitted by instrumented applications using the **OpenTelemetry** standard, then shipped to Tempo via the OTLP protocol.

``` python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("handle_checkout") as span:
    span.set_attribute("user.id", user_id)
    span.set_attribute("cart.item_count", len(cart))
    result = process_payment(cart)
    span.set_attribute("payment.status", result.status)
```

For popular libraries (Flask, FastAPI, requests, SQLAlchemy, psycopg) you can get traces without writing any code by running your service under `opentelemetry-instrument`.

------------------------------------------------------------------------

## Section 3: The RED and USE Methods

Two lightweight frameworks cover 90% of service-level monitoring.

### 3.1 RED for services

For every HTTP or RPC service, track:

- **R**ate — requests per second

- **E**rrors — error rate (5xx, or domain-specific)

- **D**uration — latency distribution (p50, p95, p99)

```
# Request rate
rate(http_requests_total{service="orders"}[5m])

# Error rate as a fraction
sum(rate(http_requests_total{service="orders", status=~"5.."}[5m]))
  / sum(rate(http_requests_total{service="orders"}[5m]))

# p95 latency
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket{service="orders"}[5m]))
)
```

### 3.2 USE for resources

For every resource (CPU, disk, network interface, connection pool):

- **U**tilization — percent busy

- **S**aturation — queue depth or wait time

- **E**rrors — failures on the resource itself

USE catches problems that RED misses. A service can have healthy latency right up until the connection pool saturates, at which point latency explodes. A USE panel on the connection pool gives you the early warning.

------------------------------------------------------------------------

## Section 4: SLOs and Error Budgets

An **SLI** (Service Level Indicator) is a measurable property of your service — “fraction of requests that return in under 300ms and are not 5xx”.

An **SLO** (Service Level Objective) is a target for that SLI — “99.9% of requests over a 30-day rolling window”.

The **error budget** is the complement — 0.1% of requests, or about 43 minutes per month. You are *allowed* to fail that much. If you haven’t burned it, ship features. If you’ve blown it, halt feature work and invest in reliability.

### 4.1 Burn-rate Alerts

Instead of alerting when any error happens (noisy) or when the monthly budget is fully exhausted (too late), alert on **burn rate** — how fast you’re consuming the budget.

```
# 1-hour burn rate: fast burn means budget gone in 2 days
(
  1 - (
    sum(rate(http_requests_total{status!~"5.."}[1h]))
    / sum(rate(http_requests_total[1h]))
  )
) > (14.4 * (1 - 0.999))
```

The `14.4` multiplier comes from the math: if you burn at 14.4× the sustainable rate, you’ll exhaust a 30-day budget in 30 days / 14.4 ≈ 2 days. That is the standard “fast burn” alert window.

------------------------------------------------------------------------

## Section 5: Best Practices

1.  **Structured logs**, always. JSON lines with a consistent schema. Plain text logs are a tax on every future debugging session.

2.  **Include a trace ID on every log line.** This single practice turns Loki and Tempo into one system.

3.  **No secrets, no PII.** Redact before emitting. Once it’s in the log pipeline it’s very hard to fully purge.

4.  **Alert on symptoms, not causes.** Alert when users can’t check out, not when CPU is at 80% — CPU alerts are noisy and often harmless.

5.  **Dashboards per service, not per team.** A team’s on-call should be able to open one dashboard and see the state of the service they are responsible for.

6.  **Annotate deploys and incidents.** Grafana annotations overlay deploy markers on every panel, making “what changed” questions trivial.

7.  **Test your alerts.** An alert that has never fired is an alert you don’t know works. Rehearse with synthetic failures quarterly.

------------------------------------------------------------------------

## Section 6: Extended PromQL and LogQL Examples

The basics in Section 2 cover 80% of day-to-day queries. The patterns below are the ones you will reach for when you are on-call or debugging a hot incident.

### 6.1 PromQL Patterns

```
# Top 5 services by error rate
topk(5,
  sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
)

# Alert: error rate above 5% for 10 minutes
(
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))
) > 0.05
```

### 6.2 LogQL Patterns

```
# All ERROR logs from the web service in the last hour
{app="web"} |= "ERROR"

# Parse JSON and filter on a nested field
{app="web"} | json | status_code >= 500

# Rate of 500s per minute, by service
sum by (service) (
  rate({app=~".+"} |= "500" [1m])
)
```

### 6.3 OpenTelemetry: Auto-instrumentation

For popular Python libraries (Flask, FastAPI, requests, SQLAlchemy, psycopg), auto-instrumentation wraps everything with zero code changes:

``` bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap --action=install

opentelemetry-instrument --traces_exporter otlp \
  --exporter_otlp_endpoint http://tempo:4317 \
  python app.py
```

This is the fastest way to get traces into Tempo when adopting OpenTelemetry.

------------------------------------------------------------------------

## Section 7: Grafana Tips

- **Variables** (`$service`, `$env`) make one dashboard serve many.

- **Transformations** can do joins and math client-side without changing the query.

- **Alerting:** Grafana-managed alerts are easier than Prometheus Alertmanager for small teams.

- **Annotations** mark deploys, incidents, and changes on every panel — this is the fastest “what changed?” tool you have during an incident.

- **Dashboards as code:** export JSON from Grafana, check it into the repo, and import via provisioning so environments stay in sync.

------------------------------------------------------------------------

## Summary

Observability is the property of being able to ask new questions about a running system without shipping new code. In a microservices architecture it is no longer a nice-to-have — it is the only way to keep the system tractable. The LGTM stack provides production-grade tooling for all three pillars at a reasonable cost, and the RED/USE methods give you a lightweight mental model for what to measure. SLOs with burn-rate alerts connect engineering work directly to user-visible reliability.

For LLM-backed services, the same three pillars apply plus a fourth layer: trace the full agent loop (input → tool calls → LLM calls → output). Tools like LangFuse and LangSmith are purpose-built for this fourth layer — see [Observability: LangFuse & LangSmith](../../ai/llmops/llmops-observability.md).

## Practice

Use the official Grafana Labs `docker-compose` stack or the LGTM all-in-one image to avoid wiring individual services.

### 1. Instrument a FastAPI app

Add Prometheus metrics to a FastAPI app with `prometheus-fastapi-instrumentator`. Expose `/metrics`. Confirm Prometheus scrapes it by running `curl`` ``http://localhost:9090/api/v1/targets` and checking `health:`` ``"up"`.

### 2. Build a RED dashboard

In Grafana, build a dashboard that shows Request rate, Error rate, and p95 Duration for your service. Use `rate()` and `histogram_quantile()`. Load test the app with `hey` or `wrk` and watch the dashboard update.

### 3. Alert on error budget burn

Define an SLO of 99.5% success rate. Write a PromQL alert that fires when the 5-minute error rate exceeds `(1`` ``-`` ``0.995)`` ``×`` ``14.4` (fast burn). Trigger it by introducing a fault (e.g., `/error` endpoint returns 500).

### 4. Trace IDs in logs

Add OpenTelemetry auto-instrumentation to your app. Configure your logger to include `trace_id` on every log line. In Grafana, click a span in Tempo and verify the “Logs for this span” link jumps straight to the matching Loki logs. This cross-pillar linking is the payoff of a unified stack.

### 5. USE method on a database

For a Postgres instance, use `node_exporter` and `postgres_exporter` to track:

- CPU / memory utilization

- I/O saturation (disk queue depth)

- Connection errors

Build one dashboard panel per dimension. Under load, identify which resource saturates first.

## Review Questions

1.  **What are the three pillars of observability?**

    - A. Alerts, dashboards, runbooks

    - B. Metrics, logs, traces

    - C. CPU, memory, disk

    - D. DEBUG, INFO, ERROR

2.  **How does Prometheus collect metrics from targets?**

    - A. Applications push metrics to Prometheus

    - B. Prometheus scrapes a `/metrics` HTTP endpoint on each target (pull model)

    - C. Via a Kafka topic

    - D. Via direct database writes

3.  **Which PromQL function computes per-second rate from a counter?**

    - A. `sum()`

    - B. `rate()`

    - C. `delta()`

    - D. `count()`

4.  **Which metrics does the RED method track for services?**

    - A. Revenue, Expenses, Deficit

    - B. Rate, Errors, Duration

    - C. Requests, Exceptions, Downtime

    - D. Read, Execute, Delete

5.  **What is Loki’s key design decision that keeps costs low?**

    - A. It only indexes log labels, not the log content

    - B. It stores logs in RAM

    - C. It compresses logs 100x

    - D. It deletes logs after 1 day

6.  **Why should log lines include a trace ID?**

    - A. To save disk space

    - B. To jump from a failing log entry directly to the full distributed trace it belongs to

    - C. To satisfy compliance

    - D. To enable logging at DEBUG level

7.  **What does an “error budget” express?**

    - A. The dollar amount allocated for incidents

    - B. The allowable fraction of failed operations before halting feature work and investing in reliability

    - C. The number of engineers on-call

    - D. The cost of log storage

8.  **Which Prometheus metric type is best for recording request latency distributions?**

    - A. Counter

    - B. Gauge

    - C. Histogram

    - D. Summary (prefer over histogram)

9.  **OpenTelemetry is…**

    - A. A single observability vendor

    - B. The CNCF standard for instrumenting applications to emit traces, metrics, and logs, with a vendor-neutral wire format (OTLP)

    - C. A replacement for Kubernetes

    - D. A proprietary Datadog protocol

10. **Why is auto-instrumentation useful when adopting OpenTelemetry?**

    - A. It’s required for compliance

    - B. It instruments popular libraries (Flask, FastAPI, SQLAlchemy, requests) with zero code changes, giving you traces immediately

    - C. It compiles the code to machine code

    - D. It removes the need for Grafana

**View Answer Key**

1.  **B** — Metrics, logs, traces are the canonical three pillars.

2.  **B** — Prometheus pulls; you must expose a `/metrics` endpoint.

3.  **B** — `rate()` computes per-second increase over a time window.

4.  **B** — Rate, Errors, Duration — the minimum viable service dashboard.

5.  **A** — Loki indexes only labels; content is stored compressed in object storage.

6.  **B** — Cross-pillar linking is the whole point of a unified stack.

7.  **B** — Error budget = allowable failure; halt features when burned too fast.

8.  **C** — Histograms expose buckets you can aggregate; summaries compute quantiles client-side and are harder to aggregate across instances.

9.  **B** — OpenTelemetry is the CNCF standard with vendor-neutral OTLP wire format.

10. **B** — Zero-code instrumentation via `opentelemetry-instrument` is the fastest way to get traces.
