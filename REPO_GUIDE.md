# demo-observability — REPO GUIDE

## Overview

Grafana + Prometheus + Tempo + OpenTelemetry Collector. Pre-provisioned dashboards showing transaction flow and trace IDs across all 3 services. Single `docker compose up` starts everything.

**Loki is NOT used.** Tempo trace attributes are the smoking gun — the trace shows `txn_id` changing between spans. This removes an entire class of log ingestion bugs (Promtail config, loki exporter deprecation, label mapping, double-encoded JSON).

## Critical Fix Notes

- **C7:** Tempo listens on `:4317` (inside container). OTel Collector exports to `tempo:4317`. No port mismatch.
- **C8:** Tempo needs a **custom `tempo.yaml`** that explicitly enables the OTLP gRPC receiver on `0.0.0.0:4317`. The bundled config does NOT enable it by default — it binds to localhost only. Without this, the OTel Collector can't push traces to Tempo and the demo's smoking gun (trace showing `txn_id` mismatch) is **gone**.
- **R8/M17:** Dropped Loki entirely. Tempo trace span attributes are the smoking gun. No Promtail, no loki exporter, no log ingestion pipeline.
- **M1:** Dropped `prometheusremotewrite` exporter — redundant with Prometheus scraping. Was failing without `--web.enable-remote-write-receiver`.
- **M2:** Prometheus scrapes `otel-collector:8889` (not `8888` which is self-telemetry).
- **M11:** Custom `tempo/` directory mounted to `/etc/tempo.yaml` — enables OTLP receiver on `0.0.0.0:4317`.
- **M13:** Dashboard JSONs must set `"uid"` explicitly so documented URLs work.
- **M12:** Dashboards should be **built in Grafana UI first**, then exported as JSON and committed. Don't hand-write dashboard JSON.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  demo-observability                   │
│                                                       │
│  ┌──────────┐  ┌────────────┐  ┌──────────────────┐  │
│  │ Grafana   │  │ Prometheus │  │   OTel Collector  │  │
│  │ :3000     │◄─┤ :9090      │  │   :4317 (gRPC)   │  │
│  │          │  │            │  │   :4318 (HTTP)   │  │
│  │ Dashboards│  │ Scrape:    │  │                  │  │
│  │ Traces   │  │ :8001/metrics│ │ ← Python (traces)│  │
│  │ (Tempo)  │  │ :8002/metrics│ │ ← TS (traces)    │  │
│  │          │  │ :8003/metrics│ │ ← Go (traces)    │  │
│  │          │  │ :8889 (OTel) │ │                  │  │
│  └────┬─────┘  └─────┬──────┘  └────────┬─────────┘  │
│       │               │                  │             │
│  ┌────┴─────┐         │                  │             │
│  │  Tempo   │◄────────┴──────────────────┘             │
│  │  :3200   │  (collector exports traces to Tempo)     │
│  │  :4317   │  (OTLP receiver, bundled config)         │
│  └──────────┘                                         │
└─────────────────────────────────────────────────────┘
```

## Directory Structure

```
demo-observability/
├── REPO_GUIDE.md
├── docker-compose.yml
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── datasources.yml      # Prometheus + Tempo
│   │   └── dashboards/
│   │       ├── dashboards.yml       # Dashboard provider config
│   │       ├── transaction-flow.json    # Built in UI, then exported
│   │       └── error-rate.json         # Built in UI, then exported
│   └── grafana.ini
├── prometheus/
│   └── prometheus.yml               # Scrape config (all services + collector)
├── otel-collector/
│   └── otel-collector.yml           # OTel pipeline (traces → Tempo, metrics → Prometheus)
├── tempo/
│   └── tempo.yaml                  # Minimal config — explicitly enables OTLP receiver on 0.0.0.0:4317
└── README.md
```

> **Custom `tempo/tempo.yaml` is required.** The bundled config does NOT enable OTLP on `0.0.0.0:4317` — it binds to localhost only. Without the custom config, the OTel Collector can't push traces and the smoking gun is gone.

## File-by-File Specification

### `docker-compose.yml`

```yaml
services:
  # ─── Grafana ──────────────────────────────────────────────
  grafana:
    image: grafana/grafana:10.3.1
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - grafana-data:/var/lib/grafana
    networks:
      - demo-network
    depends_on:
      - prometheus
      - tempo

  # ─── Prometheus ──────────────────────────────────────────
  prometheus:
    image: prom/prometheus:v2.48.1
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    networks:
      - demo-network

# ─── Tempo (trace storage) ───────────────────────────────
  # B3 fix: Mount custom tempo.yaml that explicitly enables OTLP gRPC receiver on 0.0.0.0:4317
  # The bundled config does NOT enable it by default (binds localhost only)
  tempo:
    image: grafana/tempo:2.4.1
    ports:
      - "3200:3200"   # Tempo API (for Grafana queries)
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./tempo/tempo.yaml:/etc/tempo.yaml
      - tempo-data:/data/tempo
    networks:
      - demo-network

  # ─── OTel Collector ─────────────────────────────────────
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.91.0
    ports:
      - "4317:4317"   # OTel gRPC (Python + Go send traces here)
      - "4318:4318"   # OTel HTTP (TypeScript sends traces here)
      - "8889:8889"   # Prometheus exporter (scraped by Prometheus)
    volumes:
      - ./otel-collector/otel-collector.yml:/etc/otelcol/config.yaml
    networks:
      - demo-network
    depends_on:
      - tempo

volumes:
  grafana-data:
  prometheus-data:
  tempo-data:

networks:
  demo-network:
    external: true
```

> **C8 fix:** Custom `tempo/tempo.yaml` mounted to `/etc/tempo.yaml`. This explicitly enables OTLP gRPC receiver on `0.0.0.0:4317` — the bundled config doesn't enable it by default (binds localhost only).
> **C7 fix:** No `4319:4317` port remap. Tempo container listens on `4317` internally. Collector exports to `tempo:4317`.

### `prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 5s
  evaluation_interval: 5s

scrape_configs:
  - job_name: "payment-service"
    static_configs:
      - targets: ["payment-service:8001"]
    metrics_path: /metrics

  - job_name: "order-service"
    static_configs:
      - targets: ["order-service:8002"]
    metrics_path: /metrics

  - job_name: "ledger-service"
    static_configs:
      - targets: ["ledger-service:8003"]
    metrics_path: /metrics

  # M2 fix: scrape collector on 8889 (prometheus exporter port), NOT 8888 (self-telemetry)
  - job_name: "otel-collector"
    static_configs:
      - targets: ["otel-collector:8889"]
```

> **M2 fix:** `otel-collector:8889` not `8888`.

### `otel-collector/otel-collector.yml`

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1000

exporters:
  # M1 fix: prometheus exporter (scraped by Prometheus on 8889)
  # M1 fix: removed prometheusremotewrite (was failing without receiver flag)
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: "otel"

  # C7 fix: export to tempo:4317 (Tempo's bundled OTLP receiver port)
  otlp/tempo:
    endpoint: "tempo:4317"
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/tempo]

    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]

  # No logs pipeline — we dropped Loki (M17)
```

> **M1 fix:** Removed `prometheusremotewrite` exporter. Prometheus scrapes the collector's `:8889` endpoint instead.
> **M17 fix:** No `loki` exporter, no logs pipeline.
> **C7 fix:** `otlp/tempo` endpoint is `tempo:4317` — matches Tempo's bundled OTLP receiver.

### `tempo/tempo.yaml`

```yaml
# Minimal Tempo config — explicitly enables OTLP gRPC receiver on 0.0.0.0:4317
# B3 fix: The bundled config does NOT enable OTLP on 0.0.0.0 — it defaults to localhost only.
# Without this custom config, the OTel Collector can't push traces to Tempo.
# Source: Context7 Tempo docs — "include the otlp receiver in your configuration"

server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: "0.0.0.0:4317"
        http:
          endpoint: "0.0.0.0:4318"

storage:
  trace:
    backend: local
    local:
      path: /data/tempo/blocks
    wal:
      path: /data/tempo/wal
```

> **B3 fix:** Custom config explicitly enables OTLP gRPC on `0.0.0.0:4317`. Volume mount aligns: `tempo-data:/data/tempo` matches `storage.trace.local.path: /data/tempo/blocks`.

### `grafana/grafana.ini`

```ini
[paths]
provisioning = /etc/grafana/provisioning

[auth.anonymous]
enabled = true
org_role = Viewer

[server]
http_port = 3000
```

### `grafana/provisioning/datasources/datasources.yml`

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
    uid: prometheus

  - name: Tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
    editable: true
    uid: tempo
    jsonData:
      tracesToMetrics:
        datasourceUid: prometheus
      # Enable span attribute search
      search:
        hide: false
      # Show span attributes in trace view
      nodeGraph:
        enabled: true
```

> Only 2 datasources: Prometheus + Tempo. No Loki.

### `grafana/provisioning/dashboards/dashboards.yml`

```yaml
apiVersion: 1

providers:
  - name: "Demo Dashboards"
    orgId: 1
    folder: "FlowMap Demo"
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /etc/grafana/provisioning/dashboards
```

### `grafana/provisioning/dashboards/transaction-flow.json`

> **M12 fix:** Build this dashboard in the Grafana UI first, then export as JSON and commit.
>
> **M13 fix:** Set `"uid": "transaction-flow"` in the JSON.
>
> **How to build:**
> 1. Start the stack: `docker compose up -d`
> 2. Create a transaction: `curl -X POST http://localhost:8001/transactions ...`
> 3. Open Grafana → Explore → Tempo
> 4. Search by `service.name = demo-payment-service`
> 5. Click the most recent trace → see 3 spans (payment → order → ledger)
> 6. Note the `txn_id` attribute in each span — shows the mismatch
> 7. Create a new dashboard → Add panel → Tempo data source
> 8. Query: `{ service.name = "demo-payment-service" }` (or list all traces)
> 9. Visualization: Trace view
> 10. Save dashboard with UID `transaction-flow`
> 11. Export JSON → save to this file

**Panel 1:** Tempo trace view — shows all 3 spans with `txn_id` attributes

**Panel 2:** Attribute table — shows `txn_id` from each span side by side:
- Payment span: `txn_id = 334152530919428096` (correct)
- Order span: `txn_id = <corrupted value>` (rounded by IEEE 754)

- Ledger span: `txn_id = <same corrupted value>`, `found = false`
### `grafana/provisioning/dashboards/error-rate.json`

> Build in UI, export, commit. Set `"uid": "error-rate"`.

**Panel 1:** `rate(ledger_transactions_not_found_total[5m])` — "not found" spike

**Panel 2:** `payment_transactions_created_total` vs `ledger_transactions_found_total` — shows the gap (transactions created but not found)

> **Note:** `order_errors_total` is NOT included — the TS order service doesn't throw errors during the bug. It successfully forwards the (corrupted) ID. The error is silent. The relevant signal is `ledger_transactions_not_found_total`.

## Build & Run

```bash
# Create the shared network (first time only)
docker network create demo-network

# Start all observability services
cd demo-observability
docker compose up -d

# Or via umbrella compose (from demo-repos/):
# docker compose up -d

# Verify:
# Grafana:    http://localhost:3000 (admin/admin)
# Prometheus: http://localhost:9090
# Tempo:      http://localhost:3200/ready
# OTel:       http://localhost:4317 (gRPC, no web UI)
```

## Pre-Provisioned Dashboards

Once Grafana is running, navigate to:

| Dashboard | URL | What It Shows |
|-----------|-----|---------------|
| Transaction Flow | `http://localhost:3000/d/transaction-flow/transaction-flow` | Trace spans across all 3 services, `txn_id` attribute mismatch visible |
| Error Rate | `http://localhost:3000/d/error-rate/error-rate` | "transaction not found" spike, created-vs-found gap |

> **M13 fix:** URLs use explicit UIDs (`transaction-flow`, `error-rate`) set in dashboard JSON.

## Key Config

| Service | Port | Purpose |
|---------|------|---------|
| Grafana | 3000 | Dashboards, trace/log exploration |
| Prometheus | 9090 | Metrics scraping |
| Tempo | 3200 | Trace storage + API for Grafana queries |
| Tempo (internal) | 4317 | OTLP receiver (collector pushes here) |
| OTel Collector | 4317 (gRPC), 4318 (HTTP), 8889 (metrics) | Receives traces, exports to Tempo + Prometheus |

## How the Trace Shows the Bug

When all 3 services are running and a transaction is created:

1. Open Grafana → Explore → Tempo
2. Search by `service.name = demo-payment-service` (or `demo-order-service`)
3. Click the most recent trace
4. See the trace spans:

```
payment.create_transaction
  txn_id: 334152530919428096    ← CORRECT (Python, arbitrary precision)
  │
  └─ order.process_transaction
       txn_id: <corrupted value>     ← CHANGED! (JS IEEE 754 rounding)
       │
       └─ ledger.lookup_transaction
            txn_id: <same corrupted value>  ← RECEIVED WRONG ID
            found: false
            error: "transaction not found"
```

**The "aha" in Grafana:** You can literally see the `txn_id` change between the payment span and the order span. The trace tells the story.

**Why only tracing can see this:**
- **Logs** show the already-corrupted value (JS `console.log` also uses IEEE 754)
- **Metrics** show "not found" but not *why*
- **Traces** show the `txn_id` as a span attribute in each service — the original (Python) vs corrupted (TS) are side by side

This is why the demo story is: **"Tracing was the only thing that could see this."**