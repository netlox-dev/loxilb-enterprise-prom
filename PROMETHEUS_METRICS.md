# LoxiLB Enterprise Prometheus Metrics Documentation

Comprehensive guide to all Prometheus metrics exported by LoxiLB Enterprise for production monitoring and observability.

---

## Table of Contents
- [Quick Start](#quick-start)
- [Metric Categories](#metric-categories)
- [Connection Tracking Metrics](#connection-tracking-metrics)
- [Load Balancer Metrics](#load-balancer-metrics)
- [Endpoint Health Metrics](#endpoint-health-metrics)
- [Traffic Processing Metrics](#traffic-processing-metrics)
- [System Utilization Metrics](#system-utilization-metrics)
- [Firewall Metrics](#firewall-metrics)
- [Distribution Metrics](#distribution-metrics)
- [RPS Metrics](#rps-metrics)
- [LCU Capacity Metrics](#lcu-capacity-metrics)
- [Query Examples](#query-examples)
- [Alerting Rules](#alerting-rules)

---

## Quick Start

### Accessing Metrics

```bash
# Default metrics endpoint
curl http://localhost:8080/metrics

# With authentication
curl -H "Authorization: Bearer <token>" https://loxilb:8080/metrics
```

### Scrape Configuration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'loxilb-enterprise'
    scrape_interval: 10s
    static_configs:
      - targets: ['loxilb:8080']
    metric_path: '/metrics'
```

---

## Metric Categories

| Category | Update Interval | Priority | Count |
|----------|----------------|----------|-------|
| Connection Tracking | 10s | Critical | 6 |
| Load Balancer | 10s | Critical | 6 |
| Endpoint Health | 10s | Critical | 3 |
| Traffic Processing | 10s | Important | 9 |
| System Utilization | 10s | Critical | 3 |
| Firewall | 10s | Critical | 3 |
| Distribution | 2min | Operational | 6 |
| RPS | 10s | Important | 6 |
| LCU | 10s | Historical | 1 |

---

## Connection Tracking Metrics

### `active_conntrack_count`
**Type**: Gauge
**Description**: Total number of active established connections from clients to targets
**Use Case**: Monitor current connection load and capacity planning

```promql
# Current active connections
active_conntrack_count

# Connection growth rate
rate(active_conntrack_count[5m])

# Peak connections in last hour
max_over_time(active_conntrack_count[1h])
```

---

### `active_flow_count_tcp`
**Type**: Gauge
**Description**: Number of concurrent TCP flows (connections) from clients to targets
**Use Case**: Protocol-specific load monitoring

```promql
# TCP connection percentage
active_flow_count_tcp / active_conntrack_count * 100
```

---

### `active_flow_count_udp`
**Type**: Gauge
**Description**: Number of concurrent UDP flows from clients to targets
**Use Case**: UDP traffic analysis for QUIC, DNS, gaming protocols

```promql
# UDP vs TCP ratio
active_flow_count_udp / active_flow_count_tcp
```

---

### `active_flow_count_sctp`
**Type**: Gauge
**Description**: Number of concurrent SCTP flows (critical for telco/5G workloads)
**Use Case**: Telco service monitoring, 5G N2/N4 interface tracking

```promql
# SCTP connection health
active_flow_count_sctp > 0
```

---

### `inactive_flow_count`
**Type**: Gauge
**Description**: Number of closed/inactive flows in conntrack table
**Use Case**: Connection cleanup monitoring, potential memory leaks

```promql
# Cleanup efficiency
inactive_flow_count / (active_conntrack_count + inactive_flow_count)

# Alert on excessive inactive flows
inactive_flow_count > 10000
```

---

### `new_flow_count`
**Type**: Gauge
**Description**: Number of new connections established in current collection interval
**Use Case**: Connection rate monitoring, DDoS detection

```promql
# New connections per second
rate(new_flow_count[1m])

# Connection burst detection
increase(new_flow_count[30s]) > 1000
```

---

## Load Balancer Metrics

### `lb_rule_count`
**Type**: Gauge
**Description**: Total number of active load balancer rules configured
**Use Case**: Configuration audit, capacity management

```promql
# Total LB rules
lb_rule_count

# Alert on rule limit
lb_rule_count > 256
```

---

### `total_requests`
**Type**: Counter
**Description**: Cumulative total number of requests (new flows) processed
**Use Case**: Traffic volume tracking, billing, capacity planning

```promql
# Requests per second
rate(total_requests[1m])

# Total requests in last hour
increase(total_requests[1h])

# Request growth trend
deriv(total_requests[5m])
```

---

### `total_requests_per_service`
**Type**: Counter (with label: `service`)
**Description**: Total requests per individual service
**Use Case**: Per-service traffic analysis, hot service identification

```promql
# Top 10 services by request rate
topk(10, rate(total_requests_per_service[5m]))

# Requests for specific service
rate(total_requests_per_service{service="192.168.1.100:80"}[1m])

# Service request distribution
sum by (service) (rate(total_requests_per_service[5m]))
```

---

### `total_errors`
**Type**: Counter
**Description**: Total number of connection errors (h/e, closed-wait, err, abort states)
**Use Case**: Error rate monitoring, SLA compliance

```promql
# Error rate
rate(total_errors[1m])

# Error percentage
rate(total_errors[5m]) / rate(total_requests[5m]) * 100

# Alert on high error rate
rate(total_errors[5m]) > 10
```

---

### `total_errors_per_service`
**Type**: Counter (with label: `service`)
**Description**: Error count per individual service
**Use Case**: Service health monitoring, problematic service identification

```promql
# Services with highest error rates
topk(5, rate(total_errors_per_service[5m]))

# Error rate for specific service
rate(total_errors_per_service{service="192.168.1.100:80"}[1m])
```

---

### `healthy_endpoints_count`
**Type**: Gauge
**Description**: Number of healthy target endpoints currently available
**Use Case**: Backend health monitoring, capacity availability

```promql
# Current healthy endpoints
healthy_endpoints_count

# Alert on low healthy endpoints
healthy_endpoints_count < 2

# Endpoint availability percentage
healthy_endpoints_count / (healthy_endpoints_count + unhealthy_endpoints_count) * 100
```

---

### `unhealthy_endpoints_count`
**Type**: Gauge
**Description**: Number of unhealthy target endpoints
**Use Case**: Health check failure detection, maintenance alerts

```promql
# Current unhealthy endpoints
unhealthy_endpoints_count

# Alert on any unhealthy endpoint
unhealthy_endpoints_count > 0
```

---

## Traffic Processing Metrics

### `processed_bytes_total`
**Type**: Counter
**Description**: Total bytes processed by load balancer (including TCP/IP headers)
**Use Case**: Bandwidth monitoring, billing, network capacity planning

```promql
# Bytes per second (bandwidth)
rate(processed_bytes_total[1m])

# Gigabytes processed in last hour
increase(processed_bytes_total[1h]) / 1024 / 1024 / 1024

# Peak bandwidth in last 24h
max_over_time(rate(processed_bytes_total[1m])[24h:1m])
```

---

### `processed_packets_total`
**Type**: Counter
**Description**: Total packets processed by load balancer
**Use Case**: Packet rate monitoring, DDoS detection

```promql
# Packets per second
rate(processed_packets_total[1m])

# Average packet size
rate(processed_bytes_total[1m]) / rate(processed_packets_total[1m])
```

---

### `processed_tcp_bytes`
**Type**: Counter
**Description**: Total TCP bytes processed
**Use Case**: Protocol-specific bandwidth analysis

```promql
# TCP bandwidth
rate(processed_tcp_bytes[1m])

# TCP percentage of total traffic
rate(processed_tcp_bytes[5m]) / rate(processed_bytes_total[5m]) * 100
```

---

### `processed_udp_bytes`
**Type**: Counter
**Description**: Total UDP bytes processed
**Use Case**: UDP traffic analysis for QUIC, DNS, gaming

```promql
# UDP bandwidth
rate(processed_udp_bytes[1m])
```

---

### `processed_sctp_bytes`
**Type**: Counter
**Description**: Total SCTP bytes processed (telco/5G traffic)
**Use Case**: Telco workload monitoring, 5G interface traffic

```promql
# SCTP bandwidth
rate(processed_sctp_bytes[1m])

# SCTP traffic ratio
rate(processed_sctp_bytes[5m]) / rate(processed_bytes_total[5m]) * 100
```

---

### `processed_tcp_packets`
**Type**: Counter
**Description**: Total TCP packets processed
**Use Case**: TCP packet rate monitoring, attack detection

```promql
# TCP packets per second
rate(processed_tcp_packets[1m])

# Average TCP packet size
rate(processed_tcp_bytes[1m]) / rate(processed_tcp_packets[1m])
```

---

### `processed_udp_packets`
**Type**: Counter
**Description**: Total UDP packets processed
**Use Case**: UDP packet rate analysis

```promql
# UDP packets per second
rate(processed_udp_packets[1m])
```

---

### `processed_sctp_packets`
**Type**: Counter
**Description**: Total SCTP packets processed
**Use Case**: SCTP packet analysis for telco workloads

```promql
# SCTP packets per second
rate(processed_sctp_packets[1m])
```

---

## System Utilization Metrics

### `system_cpu_utilization`
**Type**: Gauge
**Description**: Total system CPU utilization percentage [0-100]
**Use Case**: Resource monitoring, autoscaling triggers

```promql
# Current CPU usage
system_cpu_utilization

# Alert on high CPU
system_cpu_utilization > 80

# CPU usage trend
deriv(system_cpu_utilization[5m])
```

---

### `system_memory_utilization`
**Type**: Gauge
**Description**: Total system memory utilization percentage [0-100]
**Use Case**: Memory pressure detection, OOM prevention

```promql
# Current memory usage
system_memory_utilization

# Alert on high memory
system_memory_utilization > 85
```

---

### `system_disk_utilization`
**Type**: Gauge
**Description**: Root filesystem disk utilization percentage [0-100]
**Use Case**: Disk space monitoring, log rotation triggers

```promql
# Current disk usage
system_disk_utilization

# Alert on low disk space
system_disk_utilization > 90
```

---

## Firewall Metrics

### `total_fw_drops`
**Type**: Gauge
**Description**: Total number of packets dropped by firewall rules
**Use Case**: Security monitoring, attack detection

```promql
# Current firewall drops
total_fw_drops

# Firewall drop rate
rate(total_fw_drops[1m])

# Alert on sudden drop increase
increase(total_fw_drops[1m]) > 1000
```

---

### `total_fw_drops_per_rule`
**Type**: Gauge (with label: `fw_rule`)
**Description**: Packet drops per individual firewall rule
**Use Case**: Rule effectiveness analysis, attack pattern identification

```promql
# Top rules by drop count
topk(10, total_fw_drops_per_rule)

# Drops for specific rule
total_fw_drops_per_rule{fw_rule="block_icmp"}
```

---

### `firewall_rules_count`
**Type**: Gauge
**Description**: Total number of active firewall rules
**Use Case**: Configuration audit

```promql
# Total firewall rules
firewall_rules_count
```

---

## Distribution Metrics

### `service_traffic_bytes`
**Type**: Counter (with label: `service`)
**Description**: Total bytes per service
**Use Case**: Service-level traffic analysis

```promql
# Bytes per second per service
rate(service_traffic_bytes[1m])

# Top services by bandwidth
topk(10, rate(service_traffic_bytes[5m]))
```

---

### `endpoint_traffic_bytes`
**Type**: Counter (with labels: `service`, `dip`)
**Description**: Total bytes per endpoint per service
**Use Case**: Endpoint load distribution analysis

```promql
# Traffic distribution across endpoints
sum by (dip) (rate(endpoint_traffic_bytes{service="192.168.1.100:80"}[5m]))

# Endpoint bandwidth ranking
topk(10, rate(endpoint_traffic_bytes[5m]))
```

---

### `client_traffic_packets`
**Type**: Counter (with labels: `service`, `sip`)
**Description**: Total packets per client per service
**Use Case**: Client traffic pattern analysis, top talkers

```promql
# Top clients by packet count
topk(20, rate(client_traffic_packets[5m]))

# Client request distribution for service
sum by (sip) (rate(client_traffic_packets{service="192.168.1.100:80"}[5m]))
```

---

### `lb_rule_interaction_bytes`
**Type**: Counter (with labels: `service`, `sip`, `dip`)
**Description**: Bytes exchanged between load balancer and IPs
**Use Case**: Detailed flow-level traffic analysis

```promql
# Total interaction bytes rate
sum(rate(lb_rule_interaction_bytes[1m]))

# Traffic between specific client and endpoint
rate(lb_rule_interaction_bytes{service="192.168.1.100:80",sip="10.0.0.5",dip="192.168.2.10"}[1m])
```

---

### `lb_rule_interaction_packets`
**Type**: Counter (with labels: `service`, `sip`, `dip`)
**Description**: Packets exchanged between load balancer and IPs
**Use Case**: Flow-level packet analysis

```promql
# Packets per second for specific flow
rate(lb_rule_interaction_packets{service="192.168.1.100:80"}[1m])
```

---

### `total_load_dists_per_service`
**Type**: Gauge (with label: `service`)
**Description**: Total traffic distribution ratio across all endpoints for service
**Use Case**: Load balancing effectiveness

```promql
# Service distribution ratios
total_load_dists_per_service
```

---

### `endpoint_load_dists_per_service`
**Type**: Gauge (with labels: `service`, `dip`)
**Description**: Ratio of traffic distribution for each endpoint per service
**Use Case**: Load balancing fairness analysis

```promql
# Distribution across endpoints for service
endpoint_load_dists_per_service{service="192.168.1.100:80"}

# Check for unbalanced distribution
stddev by (service) (endpoint_load_dists_per_service)
```

---

## RPS Metrics

### `rps_requests`
**Type**: Gauge
**Description**: Current requests per second
**Use Case**: Real-time load monitoring

```promql
# Current RPS
rps_requests

# RPS trend
deriv(rps_requests[5m])
```

---

### `rps_1m_avg`
**Type**: Gauge (with label: `service`)
**Description**: Average RPS over 1 minute per service
**Use Case**: Service performance tracking

```promql
# Average RPS per service
rps_1m_avg

# Top services by RPS
topk(10, rps_1m_avg)
```

---

### `rps_1m_peak`
**Type**: Gauge (with label: `service`)
**Description**: Peak RPS over 1 minute per service
**Use Case**: Capacity planning, burst detection

```promql
# Peak RPS per service
rps_1m_peak

# Peak-to-average ratio
rps_1m_peak / rps_1m_avg
```

---

### `rps_bps`
**Type**: Gauge
**Description**: Current bytes per second
**Use Case**: Real-time bandwidth monitoring

```promql
# Current bandwidth
rps_bps

# Bandwidth in Mbps
rps_bps * 8 / 1024 / 1024
```

---

### `rps_pps`
**Type**: Gauge
**Description**: Current packets per second
**Use Case**: Real-time packet rate monitoring

```promql
# Current packet rate
rps_pps
```

---

### `rps_eps`
**Type**: Gauge
**Description**: Current errors per second
**Use Case**: Real-time error rate monitoring

```promql
# Current error rate
rps_eps

# Error percentage
rps_eps / rps_requests * 100
```

---

## LCU Capacity Metrics

### `consumed_lcus`
**Type**: Gauge
**Description**: Total Load Balancer Capacity Units (LCUs) consumed
**Formula**: `(active_flows / 2,160,000) + (rules / 1,000) + (processed_bytes_gbps / 3600)`
**Use Case**: Billing, capacity planning, AWS ALB equivalent metrics

```promql
# Current LCU consumption
consumed_lcus

# LCU trend
deriv(consumed_lcus[10m])

# Alert on high LCU usage
consumed_lcus > 100
```

---

## Query Examples

### Service Health Dashboard

```promql
# Overall service health score
(healthy_endpoints_count / (healthy_endpoints_count + unhealthy_endpoints_count)) *
(1 - (rate(total_errors[5m]) / rate(total_requests[5m])))

# Service availability percentage
avg_over_time(healthy_endpoints_count[24h]) /
(avg_over_time(healthy_endpoints_count[24h]) + avg_over_time(unhealthy_endpoints_count[24h])) * 100
```

---

### Performance Analysis

```promql
# Average response time estimate (total bytes / total requests)
rate(processed_bytes_total[5m]) / rate(total_requests[5m])

# Protocol distribution
sum by (protocol) (
  label_replace(
    label_replace(
      label_replace(
        rate(processed_tcp_bytes[5m]), "protocol", "TCP", "", ""
      ), "protocol", "UDP", "", ""
    ), "protocol", "SCTP", "", ""
  )
)
```

---

### Capacity Planning

```promql
# Connection capacity remaining (assuming 100k limit)
100000 - active_conntrack_count

# CPU headroom percentage
100 - system_cpu_utilization

# Predicted time to capacity (hours)
(100000 - active_conntrack_count) / deriv(active_conntrack_count[1h]) / 3600
```

---

### SLA Monitoring

```promql
# 99.9% availability requirement
avg_over_time(
  (healthy_endpoints_count > 0)[30d:]
) / count_over_time((healthy_endpoints_count >= 0)[30d:]) * 100

# Error budget remaining (0.1% error threshold)
1 - (increase(total_errors[30d]) / increase(total_requests[30d])) / 0.001
```

---

## Alerting Rules

### Critical Alerts

```yaml
groups:
  - name: loxilb_critical
    interval: 30s
    rules:
      - alert: AllEndpointsDown
        expr: healthy_endpoints_count == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "All backend endpoints are unhealthy"
          description: "No healthy endpoints available for load balancing"

      - alert: HighCPUUsage
        expr: system_cpu_utilization > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "CPU usage above 90%"
          description: "CPU utilization: {{ $value }}%"

      - alert: HighMemoryUsage
        expr: system_memory_utilization > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Memory usage above 90%"
          description: "Memory utilization: {{ $value }}%"

      - alert: HighErrorRate
        expr: rate(total_errors[5m]) / rate(total_requests[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error rate above 5%"
          description: "Current error rate: {{ $value | humanizePercentage }}"
```

---

### Warning Alerts

```yaml
  - name: loxilb_warnings
    interval: 1m
    rules:
      - alert: HighConnectionCount
        expr: active_conntrack_count > 80000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High active connection count"
          description: "Active connections: {{ $value }}"

      - alert: UnhealthyEndpoints
        expr: unhealthy_endpoints_count > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Unhealthy backend endpoints detected"
          description: "{{ $value }} endpoints are unhealthy"

      - alert: HighDiskUsage
        expr: system_disk_utilization > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Disk usage above 85%"
          description: "Disk utilization: {{ $value }}%"

      - alert: HighFirewallDrops
        expr: rate(total_fw_drops[5m]) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High firewall drop rate"
          description: "Firewall dropping {{ $value }} packets/sec"
```

---

## Best Practices

### Query Optimization
- Use `rate()` for counters, not `increase()` for alerts
- Aggregate before rate for better performance: `sum(rate(...))`
- Use recording rules for complex repeated queries
- Set appropriate evaluation intervals (10s for critical, 1m for warnings)

### Label Usage
- Filter by service: `{service="192.168.1.100:80"}`
- Filter by endpoint: `{dip="192.168.2.10"}`
- Combine filters: `{service="...", dip="..."}`

### Retention
- High-resolution (10s): 7 days
- Medium-resolution (1m): 30 days
- Low-resolution (5m): 1 year

---

## Troubleshooting

### Missing Metrics
```bash
# Check Prometheus scrape status
curl http://prometheus:9090/api/v1/targets

# Verify LoxiLB metrics endpoint
curl http://loxilb:8080/metrics | grep -E "(active_conntrack|total_requests)"
```

### High Cardinality
- Monitor label cardinality: `count by (__name__) ({__name__=~".+"})`
- Limit per-flow metrics collection intervals
- Use aggregation for high-cardinality labels (client IPs)

### Query Performance
```promql
# Check query execution time
topk(10, count by (__name__) ({__name__=~".+"}))

# Identify slow queries in Prometheus logs
grep "query_duration_seconds" /var/log/prometheus/prometheus.log
```

---


---

## Advanced Flow-Level Analysis

### Overview
LoxiLB Enterprise provides granular flow-level metrics through `lb_rule_interaction_bytes` and `lb_rule_interaction_packets` with multi-dimensional labels (service, sip, dip) for deep traffic visibility.

### Key Use Cases
- **Elephant Flow Detection**: Identify high-bandwidth clients consuming disproportionate resources
- **Application Profiling**: Analyze packet size patterns to understand application behavior
- **Anomaly Detection**: Detect unusual client-endpoint traffic patterns
- **Capacity Planning**: Per-client and per-endpoint resource consumption tracking
- **Network Topology Visualization**: Map actual client-to-endpoint connections

---

### `lb_rule_interaction_bytes`
**Type**: Counter (CounterVec with labels)
**Labels**: `service`, `sip` (source IP), `dip` (destination IP)
**Description**: Cumulative bytes transferred per unique client-endpoint flow
**Update Frequency**: Real-time (every conntrack update)

#### Query Examples

```promql
# Top 20 flows by bandwidth
topk(20, rate(lb_rule_interaction_bytes{service=~"$service"}[1m]))

# Client-to-endpoint traffic heatmap
sum by (sip, dip) (rate(lb_rule_interaction_bytes{service=~"$service"}[1m]))

# Elephant flows (>10MB/s)
rate(lb_rule_interaction_bytes[1m]) > 10000000

# Per-client total bandwidth
sum by (sip) (rate(lb_rule_interaction_bytes{service=~"$service"}[5m]))

# Per-endpoint total bandwidth
sum by (dip) (rate(lb_rule_interaction_bytes{service=~"$service"}[5m]))

# Service bandwidth across all flows
sum(rate(lb_rule_interaction_bytes{service="frontend"}[1m]))
```

---

### `lb_rule_interaction_packets`
**Type**: Counter (CounterVec with labels)
**Labels**: `service`, `sip` (source IP), `dip` (destination IP)
**Description**: Cumulative packets transferred per unique client-endpoint flow
**Update Frequency**: Real-time (every conntrack update)

#### Query Examples

```promql
# Top 20 flows by packet rate
topk(20, rate(lb_rule_interaction_packets{service=~"$service"}[1m]))

# Per-client packet rate distribution
topk(15, sum by (sip) (rate(lb_rule_interaction_packets{service=~"$service"}[5m])))

# Small packet attack detection (>10k pps with small packets)
rate(lb_rule_interaction_packets[1m]) > 10000
  and
(rate(lb_rule_interaction_bytes[1m]) / rate(lb_rule_interaction_packets[1m])) < 100
```

---

### Advanced Analysis Queries

#### Average Packet Size per Flow
**Use Case**: Application behavior profiling (HTTP vs streaming vs gaming)

```promql
# Average packet size per flow
rate(lb_rule_interaction_bytes{service=~"$service"}[1m]) 
  / 
rate(lb_rule_interaction_packets{service=~"$service"}[1m])

# Classify by application type
# Small packets (<100 bytes) = ACKs, control messages
# Medium packets (100-1400 bytes) = HTTP, API calls
# Large packets (>1400 bytes) = Streaming, file transfer
```

**Interpretation**:
- `<64 bytes`: TCP control packets, ACKs
- `64-100 bytes`: Small API responses, DNS queries
- `100-500 bytes`: Typical HTTP requests/responses
- `500-1400 bytes`: Rich API payloads, web content
- `>1400 bytes`: Video streaming, large file transfers

---

#### Client Traffic Distribution (Elephant Flows)
**Use Case**: Identify top talkers consuming bandwidth

```promql
# Top 15 clients by bandwidth share
topk(15, sum by (sip) (rate(lb_rule_interaction_bytes{service=~"$service"}[5m])))

# Clients responsible for >10% of traffic
(
  sum by (sip) (rate(lb_rule_interaction_bytes{service=~"$service"}[5m]))
  /
  sum(rate(lb_rule_interaction_bytes{service=~"$service"}[5m]))
) > 0.10
```

---

#### Endpoint Load Balancing Effectiveness
**Use Case**: Verify load distribution across backends

```promql
# Bandwidth distribution across endpoints
sum by (dip) (rate(lb_rule_interaction_bytes{service=~"$service"}[1m]))

# Coefficient of variation (lower = better distribution)
stddev(sum by (dip) (rate(lb_rule_interaction_bytes{service=~"$service"}[5m])))
  /
avg(sum by (dip) (rate(lb_rule_interaction_bytes{service=~"$service"}[5m])))

# Detect overloaded endpoints (>2x average)
sum by (dip) (rate(lb_rule_interaction_bytes{service=~"$service"}[5m]))
  > 2 *
avg(sum by (dip) (rate(lb_rule_interaction_bytes{service=~"$service"}[5m])))
```

---

#### Active Flow Count
**Use Case**: Monitor concurrent client-endpoint connections

```promql
# Total active flows for a service
count(rate(lb_rule_interaction_bytes{service=~"$service"}[1m]) > 0)

# Active flows per client
count by (sip) (rate(lb_rule_interaction_bytes{service=~"$service"}[1m]) > 0)

# Active flows per endpoint
count by (dip) (rate(lb_rule_interaction_bytes{service=~"$service"}[1m]) > 0)
```

---

#### Flow Duration and Persistence Analysis
**Use Case**: Session persistence and connection stability

```promql
# Long-lived flows (active for >5min)
count(rate(lb_rule_interaction_bytes{service=~"$service"}[5m]) > 0)

# Short-lived flows (active <1min but not in last 5min)
count(rate(lb_rule_interaction_bytes{service=~"$service"}[1m]) > 0)
  -
count(rate(lb_rule_interaction_bytes{service=~"$service"}[5m]) > 0)
```

---

### Grafana Panel Examples

#### Traffic Heatmap
**Visualization**: Heatmap panel
**Query**: `sum by (sip, dip) (rate(lb_rule_interaction_bytes{service=~"$service"}[1m]))`
**Purpose**: Visualize traffic intensity between all clients and endpoints

**Configuration**:
- X-axis: dip (endpoint IP)
- Y-axis: sip (client IP)
- Color intensity: Bandwidth (Bps)

---

#### Network Graph
**Visualization**: Node Graph panel
**Query**: `topk(50, rate(lb_rule_interaction_bytes{service=~"$service"}[1m]))`
**Purpose**: Map client-endpoint topology with bandwidth as edge weight

**Transformation**:
```
Organize Fields:
  - source: sip
  - target: dip
  - bandwidth: Value
```

---

#### Flow Table
**Visualization**: Table panel
**Queries**:
- `rate(lb_rule_interaction_bytes{service=~"$service"}[1m])`
- `rate(lb_rule_interaction_packets{service=~"$service"}[1m])`

**Calculated Fields**:
- Avg Packet Size = Bandwidth / Packet Rate

**Columns**:
- Service, Client IP, Endpoint IP, Bandwidth (Bps), Packet Rate (pps), Avg Packet Size (bytes)

---

### Alerting Rules for Flow Analysis

```yaml
# Elephant flow detected
- alert: ElephantFlowDetected
  expr: rate(lb_rule_interaction_bytes[1m]) > 100000000  # 100 MB/s
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Elephant flow from {{$labels.sip}} to {{$labels.dip}}"
    description: "Flow consuming {{$value | humanize}}B/s"

# Small packet attack (potential DDoS)
- alert: SmallPacketAttack
  expr: |
    rate(lb_rule_interaction_packets[1m]) > 10000
      and
    (rate(lb_rule_interaction_bytes[1m]) / rate(lb_rule_interaction_packets[1m])) < 100
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Potential DDoS from {{$labels.sip}}"
    description: "{{$value | humanize}} pps with avg packet size <100 bytes"

# Endpoint overloaded
- alert: EndpointOverloaded
  expr: |
    sum by (dip) (rate(lb_rule_interaction_bytes{service=~".*"}[5m]))
      > 2 *
    avg(sum by (dip) (rate(lb_rule_interaction_bytes{service=~".*"}[5m])))
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Endpoint {{$labels.dip}} receiving 2x average traffic"
    description: "Load balancing may be ineffective"

# Unbalanced client load
- alert: UnbalancedClientLoad
  expr: |
    (
      sum by (sip) (rate(lb_rule_interaction_bytes{service=~".*"}[5m]))
      /
      sum(rate(lb_rule_interaction_bytes{service=~".*"}[5m]))
    ) > 0.30  # Single client >30% of traffic
  for: 10m
  labels:
    severity: info
  annotations:
    summary: "Client {{$labels.sip}} consuming {{$value | humanizePercentage}} of traffic"
```

---

### Performance Considerations

**Label Cardinality**:
- `lb_rule_interaction_bytes` and `lb_rule_interaction_packets` create one time series per unique (service, sip, dip) combination
- High cardinality = more memory usage in Prometheus
- Typical cardinality: `services × clients × endpoints`

**Example**:
- 10 services × 1000 clients × 5 endpoints = 50,000 time series
- Memory usage: ~50MB for 1 day retention (estimated)

**Optimization**:
- Use `topk()` in queries to limit results
- Set appropriate retention policies
- Use recording rules for frequently used aggregations

**Recording Rules Example**:
```yaml
groups:
  - name: loxilb_flow_aggregates
    interval: 30s
    rules:
      # Pre-aggregate per-client bandwidth
      - record: client_bandwidth_bps
        expr: sum by (sip, service) (rate(lb_rule_interaction_bytes[1m]))
      
      # Pre-aggregate per-endpoint bandwidth
      - record: endpoint_bandwidth_bps
        expr: sum by (dip, service) (rate(lb_rule_interaction_bytes[1m]))
      
      # Pre-calculate avg packet size
      - record: flow_avg_packet_size
        expr: |
          rate(lb_rule_interaction_bytes[1m]) 
          / 
          rate(lb_rule_interaction_packets[1m])
```

---


## Integration Examples

### Grafana Variables
```
# Service selector
label_values(total_requests_per_service, service)

# Endpoint selector
label_values(endpoint_traffic_bytes{service="$service"}, dip)
```

### Export to External Systems
```promql
# JSON export for billing
{__name__=~"consumed_lcus|processed_bytes_total|total_requests"}

# CSV export for reporting
increase(total_requests[24h])
increase(processed_bytes_total[24h])
avg_over_time(system_cpu_utilization[24h])
```

---

## Metric Collection Architecture

```
eBPF Data Plane (conntrack, packets, bytes)
          ↓
Prometheus Go Module (every 10s)
          ↓
     ┌────┴────┐
     ↓         ↓
Prometheus  Database
Scraper    (Time Series)
     ↓         ↓
  Grafana   API Query
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01-09 | Initial metrics documentation |
| 1.1 | 2025-01-09 | Added protocol-specific packet metrics |
| 1.2 | 2025-01-09 | Added traffic distribution metrics |
| 1.3 | 2025-01-09 | Added advanced flow-level analysis section |

---

## Support

For issues or questions:
- GitHub: https://github.com/loxilb-io/loxilb
- Documentation: https://loxilb-io.github.io/loxilbdocs/
- Slack: loxilb.slack.com
