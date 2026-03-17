# LoxiLB Enterprise Grafana Dashboard Setup Guide

Production-grade monitoring setup for LoxiLB Enterprise using Prometheus and Grafana.

---

## Quick Start

### 1. Import Dashboard

**Option A: Import via UI**
```
1. Open Grafana UI (http://grafana:3000)
2. Navigate to Dashboards → Import
3. Upload: docs/monitoring/grafana-loxilb-dashboard.json
4. Select Prometheus datasource
5. Click Import
```

**Option B: Import via API**
```bash
# Upload dashboard JSON
curl -X POST http://admin:admin@grafana:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d @docs/monitoring/grafana-loxilb-dashboard.json
```

---

## Dashboard Overview

### Panels & Metrics

#### **System Overview Section**
- **CPU Utilization Gauge**: Real-time system CPU usage [0-100%]
- **Memory Utilization Gauge**: Real-time system memory usage [0-100%]
- **Disk Utilization Gauge**: Root filesystem disk usage [0-100%]
- **Active Connections Graph**: TCP/UDP/SCTP flow breakdown over time

**Use Case**: Quick health check for production operators

---

#### **Traffic & Performance Section**
- **Bandwidth by Protocol**: TCP/UDP/SCTP bytes per second
- **Packet Rate by Protocol**: Protocol-specific packet rates
- **Request & Error Rate**: Total request rate with error overlay
- **Error Percentage**: Error rate as percentage with alert thresholds

**Use Case**: Real-time traffic analysis and error detection

---

#### **Service Health Section**
- **Healthy Endpoints Stat**: Current count with color coding (red <1, yellow 1-2, green ≥2)
- **Unhealthy Endpoints Stat**: Current count with color coding (green 0, yellow 1, red ≥2)
- **Endpoint Availability**: Percentage gauge showing backend health ratio
- **Top 10 Services (Pie Chart)**: Request rate distribution across services
- **Top 10 Services by Bandwidth (Stacked Bar)**: Bandwidth consumption ranking
- **Top 5 Services by Error Rate (Stacked Bar)**: Error-prone services identification

**Use Case**: Service health monitoring, SLA compliance

---

#### **Load Distribution Section**
- **Endpoint Load Distribution (Pie Chart)**: Traffic ratio per endpoint (filtered by $service variable)
- **Endpoint Bandwidth (Line Graph)**: Bandwidth per endpoint over time
- **Top 20 Clients by Packet Rate (Line Graph)**: Client traffic analysis, top talkers

**Use Case**: Load balancing effectiveness, traffic pattern analysis

---

#### **Security & Firewall Section**
- **Total Firewall Drops Stat**: Current total with color thresholds (green <100, yellow <1000, red ≥1000)
- **Firewall Drop Rate (Line Graph)**: Drops per second trend
- **Top 10 Firewall Rules (Pie Chart)**: Most active firewall rules

**Use Case**: Security monitoring, attack detection

---


#### **Advanced Flow-Level Analysis Section** ⭐
- **Client-to-Endpoint Traffic Heatmap**: Visual heatmap showing bandwidth intensity between all client IPs (sip) and endpoint IPs (dip)
- **Top 20 Flows by Bandwidth**: Line graph ranking highest bandwidth client-endpoint flows
- **Top 20 Flows by Packet Rate**: Line graph ranking highest packet rate flows
- **Average Packet Size per Flow**: Application behavior analysis (small packets = control traffic, large packets = streaming)
- **Client IP Traffic Distribution (Pie Chart)**: Identifies elephant flows and top talkers (top 15 clients)
- **Active Flow Table**: Detailed table showing all client-endpoint pairs with bandwidth, packet rate, and average packet size
- **Client-Endpoint Network Graph**: Node graph visualization mapping actual connection topology
- **Clients per Endpoint (Stacked Bar)**: Endpoint-centric view showing which clients connect to each backend

**Use Case**: 
- **Elephant Flow Detection**: Identify clients consuming >10MB/s bandwidth
- **Application Profiling**: Analyze packet size patterns (HTTP vs streaming vs gaming)
- **Anomaly Detection**: Spot unusual traffic patterns indicating DDoS or misuse
- **Capacity Planning**: Per-client and per-endpoint resource consumption tracking
- **Network Topology**: Visualize actual client-to-endpoint connection patterns

**Key Metrics**:
- `lb_rule_interaction_bytes{service, sip, dip}`: Cumulative bytes per flow
- `lb_rule_interaction_packets{service, sip, dip}`: Cumulative packets per flow
- **Label Dimensions**: service (load balancer rule), sip (source IP), dip (destination IP)

---

#### **Real-Time Rate Indicators Section** ⭐ NEW (grafana-loxilb-dashboard.json)
- **RPS Gauge**: Instantaneous requests per second (`loxilb_rps_requests`)
- **Bps Gauge**: Bits per second throughput (`loxilb_rps_bps`)
- **Pps Gauge**: Packets per second processing rate (`loxilb_rps_pps`)
- **Eps Gauge**: Errors per second (`loxilb_rps_eps`)
- **1-min Avg RPS**: Smoothed 1-minute average RPS via recording rule
- **1-min Peak RPS**: 1-minute maximum RPS for burst analysis

**Use Case**: Production operators need instant, at-a-glance rate indicators without running expensive range queries.

---

#### **Security Rate Limiting Section** ⭐ NEW (grafana-dashboards-simple2.json)
- **SYN Blocked Rate**: Time-series graph of `rate(security_syn_blocked_total[1m])` — SYN flood mitigation activity
- **Conn-Rate Blocked Rate**: `rate(security_conn_blocked_total[1m])` — rapid-connect abuse blocking
- **UDP Flood Blocked Rate**: `rate(security_udp_blocked_total[1m])` — volumetric UDP attack counter
- **IP Filter Blocked Packets**: `rate(ipfilter_blacklist_packets_total[1m])` — blocklist effectiveness
- **IP Filter Allowed Packets**: `rate(ipfilter_whitelist_packets_total[1m])` — allowlist passthrough

**Use Case**: Real-time DDoS/attack detection with per-vector visibility.

---

### New Dashboards (Phases 6-7) ⭐ NEW

#### `loxilb-ai-gateway-dashboard.json`
**UID**: `loxilb-ai-gateway`  
**Title**: LoxiLB AI Gateway  
**Panels**: 21  
**Template Variables**: `$model`, `$tenant`

| Row | Panels | Key Metrics |
|-----|--------|-------------|
| AI Request Overview | Request rate, latency gauges (p50/p95/p99), error rate | `loxilb_ai_requests_total`, recording rules |
| Token Usage | Input/output token rates, cumulative totals | `loxilb_ai_tokens_total` |
| Active Streams | Live streaming session count, RPS | `loxilb_ai_active_streams` |
| Rate Limiting & ACL | Rate limit hits, model ACL denials | `loxilb_ai_rate_limit_hits_total`, `loxilb_ai_model_not_allowed_total` |
| P/D Disaggregation | Prefill/decode latency, KV hit rate, session routing split | P/D histograms, `loxilb_ai_kv_params_*` |

**Access**: Grafana → Dashboards → LoxiLB → **LoxiLB AI Gateway**

---

#### `loxilb-observability-dashboard.json`
**UID**: `loxilb-observability`  
**Title**: LoxiLB Observability & OPA  
**Panels**: 12  

| Row | Panels | Key Metrics |
|-----|--------|-------------|
| OPA L4 Watcher | Sync rate, sync latency (p95), circuit breaker state, rule count | `loxilb_opa_*` |
| P/D & Internal Diagnostics | P/D sessions, trie nodes, fallback rate, skipped flows, dropped metrics | `loxilb_pd_*`, `loxilb_skipped_*`, `loxilb_metrics_dropped_backpressure_total` |

**Access**: Grafana → Dashboards → LoxiLB → **LoxiLB Observability & OPA**

---


## Variables

### `$DS_PROMETHEUS`
- **Type**: Datasource
- **Description**: Prometheus datasource selector
- **Default**: Auto-detected Prometheus instance

### `$service`
- **Type**: Query Variable
- **Query**: `label_values(total_requests_per_service, service)`
- **Description**: Service filter for load distribution panels
- **Multi-select**: No
- **Include All**: Yes
- **Usage**: Filter endpoint distribution and client traffic by service

**Example**: Select `192.168.1.100:80` to view endpoint load distribution for that service

### `$endpoint`
- **Type**: Query Variable
- **Query**: `label_values(lb_rule_interaction_bytes, dip)`
- **Description**: Endpoint filter for flow-level analysis panels
- **Multi-select**: No
- **Include All**: Yes
- **Usage**: Filter "Clients per Endpoint" panel to show traffic to specific backend IP

**Example**: Select `192.168.10.5` to view all client connections to that specific endpoint

### `$model` ⭐ NEW (AI Gateway dashboard)
- **Type**: Query Variable
- **Query**: `label_values(loxilb_ai_requests_total, model)`
- **Description**: Filter AI panels by LLM model (e.g., `gpt-4`, `llama3-70b`)
- **Multi-select**: No
- **Include All**: Yes
- **Usage**: Isolate metrics for a specific model to compare performance

### `$tenant` ⭐ NEW (AI Gateway dashboard)
- **Type**: Query Variable
- **Query**: `label_values(loxilb_ai_requests_total, tenant)`
- **Description**: Filter AI panels by tenant for multi-tenant deployments
- **Multi-select**: No
- **Include All**: Yes
- **Usage**: View per-tenant token consumption and rate limit events

### `$instance` ⭐ NEW (Real-Time Rate Indicators)
- **Type**: Query Variable
- **Query**: `label_values(loxilb_rps_requests, instance)`
- **Description**: Filter rate indicator panels by specific LoxiLB node instance
- **Multi-select**: No
- **Include All**: Yes
- **Usage**: Compare per-node RPS when running multiple LoxiLB instances

## Alert Integration

### Built-in Alert Thresholds

| Panel | Warning | Critical | Action |
|-------|---------|----------|--------|
| CPU Utilization | 70% | 85% | Check system load, scale resources |
| Memory Utilization | 75% | 90% | Check memory leaks, restart if needed |
| Disk Utilization | 80% | 95% | Clean logs, expand disk |
| Unhealthy Endpoints | 1 | 2+ | Check backend health, investigate failures |
| Error Percentage | 3% | 5% | Investigate application errors, check logs |
| Firewall Drops | 100/s | 1000/s | Possible attack, review firewall rules |

---

## Production Setup

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 10s
  evaluation_interval: 10s
  external_labels:
    cluster: 'production'
    environment: 'prod'

rule_files:
  - "recording-rules.yml"   # Pre-computed AI quantiles and RPS aggregations

scrape_configs:
  - job_name: 'loxilb-enterprise'
    scrape_interval: 10s
    scrape_timeout: 5s
    metrics_path: '/netlox/v1/metrics'
    static_configs:
      - targets:
          - 'llb1-loxilb.io:11111'
          - 'llb2-loxilb.io:11111'
        labels:
          region: 'ap-northeast-2'
          datacenter: 'seoul'

  # For Kubernetes deployments
  - job_name: 'loxilb-kubernetes'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - kube-system
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: loxilb
        action: keep
      - source_labels: [__meta_kubernetes_pod_ip]
        target_label: __address__
        replacement: ${1}:8080
```

---

### Grafana Data Source Setup

```yaml
# grafana-datasources.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      timeInterval: "10s"
      queryTimeout: "60s"
      httpMethod: POST
```

---

### Docker Compose Setup

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    ports:
      - "9090:9090"
    networks:
      - monitoring
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana-datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml
      - ./grafana-loxilb-dashboard.json:/etc/grafana/provisioning/dashboards/loxilb.json
      - ./grafana-dashboards.yml:/etc/grafana/provisioning/dashboards/dashboards.yml
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=http://grafana.example.com
    ports:
      - "3000:3000"
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - prometheus

  loxilb:
    image: ghcr.io/loxilb-io/loxilb:latest
    container_name: loxilb
    privileged: true
    network_mode: host
    command:
      - --prometheus=true
    restart: unless-stopped

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus-data:
  grafana-data:
```

**Dashboard Provisioning File** (`grafana-dashboards.yml`):
```yaml
apiVersion: 1

providers:
  - name: 'LoxiLB Dashboards'
    orgId: 1
    folder: 'LoxiLB'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
```

---

### Kubernetes Deployment

```yaml
# prometheus-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 10s
    scrape_configs:
      - job_name: 'loxilb'
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - kube-system
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            regex: loxilb
            action: keep

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:latest
          args:
            - '--config.file=/etc/prometheus/prometheus.yml'
            - '--storage.tsdb.retention.time=30d'
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: config
              mountPath: /etc/prometheus
            - name: storage
              mountPath: /prometheus
      volumes:
        - name: config
          configMap:
            name: prometheus-config
        - name: storage
          persistentVolumeClaim:
            claimName: prometheus-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  selector:
    app: prometheus
  ports:
    - port: 9090
      targetPort: 9090
  type: ClusterIP

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:latest
          env:
            - name: GF_SECURITY_ADMIN_PASSWORD
              value: "admin"
          ports:
            - containerPort: 3000
          volumeMounts:
            - name: storage
              mountPath: /var/lib/grafana
      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: grafana-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  selector:
    app: grafana
  ports:
    - port: 3000
      targetPort: 3000
  type: LoadBalancer
```

---

## Customization

### Adding Custom Panels

```json
// Example: Add custom service availability panel
{
  "datasource": {
    "type": "prometheus",
    "uid": "${DS_PROMETHEUS}"
  },
  "fieldConfig": {
    "defaults": {
      "unit": "percent"
    }
  },
  "targets": [
    {
      "expr": "avg_over_time(healthy_endpoints_count[24h]) / (avg_over_time(healthy_endpoints_count[24h]) + avg_over_time(unhealthy_endpoints_count[24h])) * 100",
      "legendFormat": "24h Availability"
    }
  ],
  "title": "Service Availability (24h)",
  "type": "stat"
}
```

---

### Custom Alert Rules

Create `loxilb-alerts.yml`:

```yaml
groups:
  - name: loxilb_critical_alerts
    interval: 30s
    rules:
      - alert: AllEndpointsDown
        expr: healthy_endpoints_count == 0
        for: 1m
        labels:
          severity: critical
          service: loxilb
        annotations:
          summary: "All backend endpoints are down"
          description: "LoxiLB has no healthy endpoints available"
          runbook_url: "https://docs.loxilb.io/runbooks/endpoints-down"

      - alert: HighCPUUsage
        expr: system_cpu_utilization > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "CPU usage critically high"
          description: "CPU utilization at {{ $value }}%"

      - alert: HighErrorRate
        expr: rate(total_errors[5m]) / rate(total_requests[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error rate above 5%"
          description: "Current error rate: {{ $value | humanizePercentage }}"

  - name: loxilb_warning_alerts
    interval: 1m
    rules:
      - alert: UnhealthyEndpoints
        expr: unhealthy_endpoints_count > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Unhealthy endpoints detected"
          description: "{{ $value }} endpoints are unhealthy"

      - alert: HighFirewallDrops
        expr: rate(total_fw_drops[5m]) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High firewall drop rate"
          description: "Firewall dropping {{ $value }} packets/sec"
```

Add to Prometheus config:
```yaml
rule_files:
  - 'recording-rules.yml'   # Pre-computed recording rules (already included)
  - 'loxilb-alerts.yml'     # Alert rules (optional)

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'alertmanager:9093'
```

---

## Performance Tuning

### Grafana Settings

```ini
# grafana.ini
[database]
type = postgres
host = postgres:5432
name = grafana
user = grafana
password = your_password

[server]
protocol = http
http_port = 3000
domain = grafana.example.com
root_url = https://grafana.example.com

[auth.anonymous]
enabled = false

[security]
admin_user = admin
admin_password = your_secure_password

[users]
allow_sign_up = false
auto_assign_org = true
auto_assign_org_role = Viewer

[alerting]
enabled = true
execute_alerts = true

[metrics]
enabled = true
interval_seconds = 10
```

---

### Prometheus Optimization

```yaml
# prometheus.yml
global:
  scrape_interval: 10s
  scrape_timeout: 5s
  evaluation_interval: 10s

# Storage optimization
storage:
  tsdb:
    retention.time: 30d
    retention.size: 50GB
    wal-compression: true

# Query optimization
query:
  max_concurrency: 20
  timeout: 2m
```

---

## Troubleshooting

### Dashboard Not Loading

**Issue**: Dashboard shows "No data"

**Solution**:
```bash
# Check Prometheus scrape status
curl http://prometheus:9090/api/v1/targets | jq '.data.activeTargets[] | select(.labels.job=="loxilb-enterprise")'

# Verify LoxiLB metrics endpoint
curl http://loxilb:8080/metrics | grep -E "active_conntrack|total_requests"

# Check Grafana datasource connection
curl -X GET http://admin:admin@grafana:3000/api/datasources
```

---

### High Memory Usage

**Issue**: Grafana/Prometheus consuming excessive memory

**Solution**:
```yaml
# Reduce retention period
prometheus:
  args:
    - '--storage.tsdb.retention.time=7d'

# Limit query range
grafana:
  env:
    - GF_DATAPROXY_TIMEOUT=30
    - GF_DATAPROXY_MAX_IDLE_CONNECTIONS=100
```

---

### Missing Metrics

**Issue**: Some panels show "N/A"

**Check**:
1. LoxiLB version supports all metrics (see PROMETHEUS_METRICS.md)
2. Prometheus scrape interval matches dashboard refresh rate
3. Service variable selection is valid

```promql
# List all available metrics
{__name__=~".+"}

# Check specific metric availability
count(active_conntrack_count)
```

---

## Security Best Practices

### Authentication

```yaml
# Enable Grafana authentication
grafana:
  env:
    - GF_AUTH_ANONYMOUS_ENABLED=false
    - GF_AUTH_BASIC_ENABLED=true
    - GF_AUTH_DISABLE_LOGIN_FORM=false

# Use OAuth2 for enterprise
    - GF_AUTH_GENERIC_OAUTH_ENABLED=true
    - GF_AUTH_GENERIC_OAUTH_CLIENT_ID=your_client_id
    - GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET=your_secret
```

---

### TLS/HTTPS

```yaml
# Enable HTTPS for Grafana
grafana:
  env:
    - GF_SERVER_PROTOCOL=https
    - GF_SERVER_CERT_FILE=/etc/grafana/ssl/cert.pem
    - GF_SERVER_CERT_KEY=/etc/grafana/ssl/key.pem
  volumes:
    - ./ssl:/etc/grafana/ssl:ro
```

---

### Access Control

```yaml
# Grafana RBAC
grafana:
  env:
    - GF_USERS_VIEWERS_CAN_EDIT=false
    - GF_USERS_EDITORS_CAN_ADMIN=false
    - GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer
```

---

## Maintenance

### Backup Dashboard

```bash
# Export dashboard JSON
curl -H "Authorization: Bearer <api_key>" \
  http://grafana:3000/api/dashboards/uid/loxilb-enterprise-prod > backup.json

# Backup Prometheus data
docker exec prometheus tar czf /tmp/prometheus-backup.tar.gz /prometheus
docker cp prometheus:/tmp/prometheus-backup.tar.gz ./prometheus-backup.tar.gz
```

---

### Update Dashboard

```bash
# Method 1: UI
# Navigate to Dashboard → Settings → JSON Model → Paste updated JSON

# Method 2: API
curl -X POST http://admin:admin@grafana:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d @docs/monitoring/grafana-loxilb-dashboard.json
```

---

## Support & Resources

- **Metrics Documentation**: [PROMETHEUS_METRICS.md](./PROMETHEUS_METRICS.md)
- **LoxiLB Documentation**: https://loxilb-io.github.io/loxilbdocs/
- **Grafana Documentation**: https://grafana.com/docs/
- **Prometheus Documentation**: https://prometheus.io/docs/

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01-09 | Initial dashboard and setup guide |
| 2.0 | 2026-03-17 | Added Real-Time Rate Indicators and Security Rate Limiting rows to existing dashboards; added loxilb-ai-gateway and loxilb-observability new dashboards; added $model/$tenant/$instance variables; updated Prometheus config with rule_files and corrected endpoint paths (phases 1-9) |

---

## Quick Reference

### Essential URLs
- **Grafana UI**: http://localhost:3000 (default: admin/admin)
- **Prometheus UI**: http://localhost:9090
- **LoxiLB Metrics**: http://localhost:8080/metrics

### Quick Commands
```bash
# Start monitoring stack
docker-compose -f docker-compose.monitoring.yml up -d

# Check Prometheus targets
curl http://localhost:9090/api/v1/targets

# View dashboard
open http://localhost:3000/d/loxilb-enterprise-prod

# Stop monitoring stack
docker-compose -f docker-compose.monitoring.yml down
```
