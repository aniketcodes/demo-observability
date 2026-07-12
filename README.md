# Demo Observability Stack

Observability infrastructure for the demo microservices architecture.

## Components

### Grafana (port 3000)
- Dashboards for transaction flow and error rates
- Anonymous auth enabled for demo
- Auto-provisioned datasources

### Prometheus (port 9090)
- Metrics collection from OTel collector
- 15-second scrape interval
- Monitor Prometheus itself

### Tempo (port 3200)
- Distributed trace storage
- OTLP receiver for traces
- Local storage for demo

### Loki (port 3100)
- Log aggregation
- Local filesystem backend
- 7-day retention

### Promtail (port 9080)
- Log shipping to Loki
- Scrape /var/log/demo/ directory
- Extract service labels from structured logs

### OTel Collector (ports 4317, 4318)
- OTLP gRPC and HTTP receivers
- Batch processor for efficiency
- Tempo exporter for traces
- Prometheus exporter for metrics

### Grafana MCP
- MCP server for AI integration
- Search, datasource, prometheus, loki tools

## Quick Start
```bash
docker network create demo-network
docker-compose up -d
```

## Dashboards
- **Transaction Flow**: Payments Created, Orders Processed, Ledger Found/Not Found
- **Error Rate**: Not Found Rate, Order Errors

## Known Issues
- Local storage only (data lost on restart)
- Anonymous auth enabled (no login required)
