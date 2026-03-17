# LoxiLB Sockproxy Metrics Documentation

Comprehensive guide to Prometheus metrics exported by LoxiLB's sockproxy module for AI/LLM workload monitoring and observability.

---

## Table of Contents
- [Overview](#overview)
- [Quick Start](#quick-start)
- [Architecture](#architecture)
- [Metric Categories](#metric-categories)
- [Tier 1: Critical AI/LLM Metrics](#tier-1-critical-aillm-metrics)
- [Tier 2: Important Observability Metrics](#tier-2-important-observability-metrics)
- [Tier 3: Operational Debugging Metrics](#tier-3-operational-debugging-metrics)
- [Tier 4: L7 HTTP Metrics](#tier-4-l7-http-metrics-)
- [Query Examples](#query-examples)
- [Grafana Dashboard Examples](#grafana-dashboard-examples)
- [Alerting Rules](#alerting-rules)
- [Troubleshooting Guide](#troubleshooting-guide)
- [Performance Tuning](#performance-tuning)

---

## Overview

LoxiLB's sockproxy module provides specialized metrics for monitoring AI/LLM workloads with a focus on:
- **Cache Backpressure Management**: Adaptive flow control for high-bandwidth streaming
- **Session Affinity**: Conversation-based routing for stateful AI sessions
- **HTTP/2 Multiplexing**: Connection reuse efficiency for concurrent AI requests
- **SSL/TLS Performance**: Encrypted AI API traffic monitoring
- **Streaming Detection**: Chunked transfer encoding for real-time AI responses

### Design Principles
- **Low Overhead**: C11 atomic counters with zero-lock contention
- **Real-time Collection**: 10-second intervals for responsive monitoring
- **Delta Calculation**: Overflow-protected cumulative counter processing
- **Tier-based Priority**: Critical metrics (Tier 1) vs debugging metrics (Tier 3)

---

## Quick Start

### Accessing Sockproxy Metrics

```bash
# Filter only sockproxy metrics
curl http://localhost:8080/metrics | grep "^proxy_"

# Check specific metric
curl http://localhost:8080/metrics | grep proxy_cache_backpressure_ratio
```

### Prometheus Scrape Configuration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'loxilb-sockproxy'
    scrape_interval: 10s
    static_configs:
      - targets: ['loxilb:8080']
    metric_path: '/metrics'
    relabel_configs:
      - source_labels: [__name__]
        regex: 'proxy_.*'
        action: keep
```

### Quick Health Check

```promql
# AI session routing effectiveness (target: >0.95)
proxy_session_affinity_hit_rate

# Cache pressure indicator (alert if >0.3)
proxy_cache_backpressure_ratio

# HTTP/2 efficiency (target: >10)
proxy_http2_connection_reuse_avg
```

---

## Architecture

### Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│  sockproxy.c (eBPF/userspace C)                             │
│  • C11 atomic counters (_Atomic uint64_t)                   │
│  • 10 instrumentation points                                │
│  • proxy_get_metrics() export function                      │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ↓ CGO bridge (every 10s)
┌─────────────────────────────────────────────────────────────┐
│  sockproxy_metrics.go                                       │
│  • RunSockproxyMetrics() goroutine                          │
│  • Delta calculation for counters                           │
│  • Ratio calculation for gauges                             │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ↓ promauto.NewGauge/Counter
┌─────────────────────────────────────────────────────────────┐
│  Prometheus /metrics endpoint                               │
│  • 13 metrics (7 gauges + 6 counters)                       │
│  • Standard Prometheus exposition format                    │
└─────────────────────────────────────────────────────────────┘
```

### Instrumentation Points in C Code

| Location | Metric | Trigger Condition |
|----------|--------|-------------------|
| sockproxy.c:465 | cache_high_water_events | Cache exceeds high watermark |
| sockproxy.c:744 | cache_drain_partial | EAGAIN/EWOULDBLOCK during flush |
| sockproxy.c:1195 | chunked_responses | Transfer-Encoding: chunked detected |
| sockproxy.c:1737 | conversation_hits | Session affinity lookup success |
| sockproxy.c:1740 | conversation_misses | Session affinity lookup failure |
| sockproxy.c:2391 | conversation_ttl_expired | Session TTL timeout |
| sockproxy.c:4855, 4958 | peer_eof_graceful | Graceful close with pending drain |
| sockproxy_h2.c:156 | h2_total_streams | HTTP/2 stream creation |
| sockproxy_h2.c:1526 | h2_sessions | HTTP/2 session establishment |

---

## Metric Categories

| Tier | Purpose | Update Interval | Metrics Count | Alert Priority |
|------|---------|----------------|---------------|----------------|
| **Tier 1** | Critical AI/LLM performance | 10s | 4 | High |
| **Tier 2** | Important observability | 10s | 5 | Medium |
| **Tier 3** | Operational debugging | 10s | 4 | Low |
| **Tier 4** | L7 HTTP & P/D metrics | 10s | 4 | Medium |

### Collection Pattern
All metrics use **10-second collection intervals** via `RunSockproxyMetrics()` goroutine following the `RunSecurityRateStats()` pattern from `prometheus.go`.

---

## Tier 1: Critical AI/LLM Metrics

These metrics directly impact AI workload performance and should be monitored continuously with real-time alerts.

---

### `proxy_cache_backpressure_ratio`
**Type**: Gauge  
**Range**: 0.0 to 1.0  
**Update**: Every 10 seconds  
**Description**: Ratio of active connections experiencing cache backpressure (adaptive flow control)

#### Purpose
Measures the percentage of connections where sockproxy has activated backpressure due to downstream congestion. High values indicate that clients are sending data faster than backends can process, triggering adaptive rate limiting.

#### Formula
```
proxy_cache_backpressure_ratio = cache_backpressure_active / active_connections
```

#### Thresholds
- **Healthy**: < 0.1 (10% of connections)
- **Warning**: 0.1 - 0.3 (10-30% of connections)
- **Critical**: > 0.3 (30%+ of connections)

#### Query Examples

```promql
# Current backpressure ratio
proxy_cache_backpressure_ratio

# Alert on sustained backpressure
proxy_cache_backpressure_ratio > 0.3

# Backpressure trend (increasing = worsening)
deriv(proxy_cache_backpressure_ratio[5m])

# Percentage of time under backpressure in last hour
avg_over_time(proxy_cache_backpressure_ratio[1h]) * 100
```

#### Grafana Visualization
- **Panel Type**: Time series graph with threshold lines
- **Y-axis**: 0.0 to 1.0 (percentage)
- **Thresholds**: Warning at 0.1, Critical at 0.3
- **Color**: Green (< 0.1), Yellow (0.1-0.3), Red (> 0.3)

#### Troubleshooting
| Symptom | Cause | Action |
|---------|-------|--------|
| Ratio > 0.5 | Backends overloaded | Scale backend capacity |
| Ratio > 0.3 sustained | Network congestion | Check network latency/bandwidth |
| Sudden spike to 1.0 | Backend failure | Check `unhealthy_endpoints_count` |
| Oscillating 0.0-0.5 | Insufficient cache size | Increase `CACHE_MAX_SIZE` |

---

### `proxy_session_affinity_hit_rate`
**Type**: Gauge  
**Range**: 0.0 to 1.0  
**Update**: Every 10 seconds  
**Description**: Ratio of conversation mapping hits vs total lookups (session affinity effectiveness)

#### Purpose
Measures how effectively sockproxy routes subsequent requests from the same client to the same backend, critical for stateful AI conversations (e.g., ChatGPT sessions, context-aware APIs).

#### Formula
```
proxy_session_affinity_hit_rate = conversation_hits / (conversation_hits + conversation_misses)
```

#### Thresholds
- **Excellent**: > 0.95 (95%+ hit rate)
- **Good**: 0.90 - 0.95 (90-95% hit rate)
- **Poor**: 0.80 - 0.90 (80-90% hit rate)
- **Critical**: < 0.80 (below 80% hit rate)

#### Query Examples

```promql
# Current hit rate
proxy_session_affinity_hit_rate

# Alert on low hit rate
proxy_session_affinity_hit_rate < 0.90

# Hit rate over last 5 minutes
avg_over_time(proxy_session_affinity_hit_rate[5m])

# Calculate miss rate
1 - proxy_session_affinity_hit_rate

# Total hits in last hour
increase(proxy_conversation_hits_total[1h])

# Total misses in last hour
increase(proxy_conversation_misses_total[1h])
```

#### Grafana Visualization
- **Panel Type**: Stat panel with gauge
- **Display**: Percentage (0-100%)
- **Thresholds**: Red (< 80%), Yellow (80-95%), Green (> 95%)
- **Sparkline**: Show trend over last 30 minutes

#### Troubleshooting
| Symptom | Cause | Action |
|---------|-------|--------|
| Hit rate < 0.8 | TTL too short | Increase `CONVERSATION_TTL` |
| Hit rate drops suddenly | Hash table collision | Check `conversation_sessions` capacity |
| Hit rate = 0.0 | Session affinity disabled | Enable conversation mapping |
| Gradual decrease | Memory pressure | Check `proxy_conversation_ttl_expired_total` |

#### AI Workload Impact
- **< 80% hit rate**: Context loss, increased latency, redundant model loading
- **> 95% hit rate**: Optimal performance, consistent context, efficient resource use

---

### `proxy_http2_connection_reuse_avg`
**Type**: Gauge  
**Range**: 0.0 to unbounded (typical: 1-100)  
**Update**: Every 10 seconds  
**Description**: Average number of streams per HTTP/2 session (connection multiplexing efficiency)

#### Purpose
Measures how effectively sockproxy reuses HTTP/2 connections through stream multiplexing. Higher values indicate better resource utilization and reduced TLS handshake overhead, critical for high-throughput AI APIs.

#### Formula
```
proxy_http2_connection_reuse_avg = h2_total_streams / h2_sessions
```

#### Thresholds
- **Excellent**: > 10 (10+ streams per connection)
- **Good**: 5 - 10 (moderate multiplexing)
- **Poor**: 1 - 5 (limited reuse)
- **Critical**: < 1 (no multiplexing, possible issue)

#### Query Examples

```promql
# Current reuse average
proxy_http2_connection_reuse_avg

# Alert on low multiplexing
proxy_http2_connection_reuse_avg < 5

# Reuse efficiency trend
rate(proxy_http2_total_streams_total[5m]) / rate(proxy_http2_sessions_total[5m])

# Total streams created in last hour
increase(proxy_http2_total_streams_total[1h])

# Total sessions in last hour
increase(proxy_http2_sessions_total[1h])

# Peak multiplexing over 24 hours
max_over_time(proxy_http2_connection_reuse_avg[24h])
```

#### Grafana Visualization
- **Panel Type**: Time series graph
- **Y-axis**: Logarithmic scale (1-100)
- **Target Line**: Draw reference line at 10
- **Annotation**: Mark HTTP/2 deployment events

#### Troubleshooting
| Symptom | Cause | Action |
|---------|-------|--------|
| Avg < 2 | Clients not reusing connections | Check client keep-alive settings |
| Avg = 1 exactly | HTTP/1.1 fallback | Verify ALPN negotiation |
| Sudden drop | Connection closures | Check `proxy_graceful_close_total` |
| Avg > 100 | Long-lived connections | Normal for streaming workloads |

#### Performance Impact
- **Avg = 1**: 1 TLS handshake per request (expensive)
- **Avg = 10**: 1 TLS handshake per 10 requests (90% overhead reduction)
- **Avg = 50**: 1 TLS handshake per 50 requests (98% overhead reduction)

---

### `proxy_cache_high_water_events_total`
**Type**: Counter (monotonic)  
**Update**: Every 10 seconds  
**Description**: Total number of cache backpressure activations

#### Purpose
Cumulative count of how many times sockproxy triggered adaptive backpressure due to cache exceeding the high watermark threshold. Frequent events indicate sustained pressure requiring capacity adjustments.

#### Query Examples

```promql
# Total events since startup
proxy_cache_high_water_events_total

# Events per minute
rate(proxy_cache_high_water_events_total[1m]) * 60

# Events in last hour
increase(proxy_cache_high_water_events_total[1h])

# Alert on frequent triggers (>10/min sustained)
rate(proxy_cache_high_water_events_total[5m]) * 60 > 10

# Events per active connection
rate(proxy_cache_high_water_events_total[5m]) / proxy_active_connections
```

#### Grafana Visualization
- **Panel Type**: Stat panel + Time series
- **Stat Panel**: Total events (big number)
- **Time Series**: Rate over time (events/min)
- **Alert Annotation**: Mark threshold crossings

#### Troubleshooting
| Rate | Diagnosis | Action |
|------|-----------|--------|
| 0 events/min | Healthy, no backpressure | No action needed |
| 1-5 events/min | Occasional spikes | Monitor, likely normal |
| 5-10 events/min | Sustained pressure | Increase cache size or backend capacity |
| >10 events/min | Constant backpressure | Critical: Scale backends immediately |

#### Related Metrics
```promql
# Backpressure correlation
proxy_cache_high_water_events_total and proxy_cache_backpressure_ratio > 0.2

# Impact on partial drains
rate(proxy_cache_high_water_events_total[5m]) and rate(proxy_cache_drain_partial_total[5m])
```

---

## Tier 2: Important Observability Metrics

These metrics provide essential visibility into sockproxy operations and capacity utilization.

---

### `proxy_active_connections`
**Type**: Gauge  
**Range**: 0 to unbounded  
**Update**: Every 10 seconds  
**Description**: Current number of active proxy connections (all types: HTTP/1.1, HTTP/2, SSL/TLS)

#### Purpose
Real-time count of all active connections handled by sockproxy, used for capacity planning and load monitoring.

#### Query Examples

```promql
# Current active connections
proxy_active_connections

# Peak connections in last 24 hours
max_over_time(proxy_active_connections[24h])

# Average connections over last hour
avg_over_time(proxy_active_connections[1h])

# Connection growth rate
rate(proxy_active_connections[5m])

# Alert on high connection count (assuming 10k limit)
proxy_active_connections > 8000

# Connection utilization percentage (assuming 10k limit)
(proxy_active_connections / 10000) * 100
```

#### Grafana Visualization
- **Panel Type**: Time series + Stat
- **Stat Panel**: Current value with max/min
- **Time Series**: Show daily/weekly patterns
- **Threshold**: Warning at 80% capacity, critical at 90%

---

### `proxy_active_ssl_connections`
**Type**: Gauge  
**Range**: 0 to unbounded  
**Update**: Every 10 seconds  
**Description**: Current number of active SSL/TLS connections

#### Purpose
Monitors encrypted connection count for performance and security analysis. SSL/TLS connections have higher CPU overhead due to encryption/decryption.

#### Query Examples

```promql
# Current SSL connections
proxy_active_ssl_connections

# SSL ratio (percentage of encrypted traffic)
(proxy_active_ssl_connections / proxy_active_connections) * 100

# Alert on SSL connection limit
proxy_active_ssl_connections > 5000

# Non-SSL connections
proxy_active_connections - proxy_active_ssl_connections

# SSL growth rate
rate(proxy_active_ssl_connections[5m])
```

#### Grafana Visualization
- **Panel Type**: Pie chart (SSL vs non-SSL) + Time series
- **Pie Chart**: Show distribution
- **Time Series**: Track SSL adoption over time

---

### `proxy_conversation_sessions`
**Type**: Gauge  
**Range**: 0 to unbounded  
**Update**: Every 10 seconds  
**Description**: Current number of active conversation mappings (session affinity state)

#### Purpose
Tracks how many unique client-to-backend session mappings are currently maintained in memory for AI conversation routing.

#### Query Examples

```promql
# Current conversation sessions
proxy_conversation_sessions

# Sessions per active connection
proxy_conversation_sessions / proxy_active_connections

# Alert on session table capacity (assuming 50k limit)
proxy_conversation_sessions > 45000

# Session growth rate
rate(proxy_conversation_sessions[5m])

# Peak sessions in last week
max_over_time(proxy_conversation_sessions[7d])
```

#### Troubleshooting
| Symptom | Cause | Action |
|---------|-------|--------|
| Sessions >> connections | Long TTL, slow expiration | Reduce `CONVERSATION_TTL` |
| Sessions ≈ connections | Good equilibrium | No action needed |
| Sessions = 0 | Session affinity disabled | Enable if needed for AI workloads |

---

### `proxy_chunked_responses_total`
**Type**: Counter (monotonic)  
**Update**: Every 10 seconds  
**Description**: Total number of HTTP responses using chunked transfer encoding

#### Purpose
Detects streaming AI workloads (e.g., GPT-4 streaming, real-time translations). High values indicate clients receiving incremental responses.

#### Query Examples

```promql
# Total chunked responses since startup
proxy_chunked_responses_total

# Chunked responses per second
rate(proxy_chunked_responses_total[1m])

# Chunked responses in last hour
increase(proxy_chunked_responses_total[1h])

# Percentage of responses that are chunked (vs total requests)
rate(proxy_chunked_responses_total[5m]) / rate(total_requests[5m]) * 100

# Alert on high streaming load
rate(proxy_chunked_responses_total[5m]) > 100
```

#### Grafana Visualization
- **Panel Type**: Time series + Stat
- **Stat Panel**: Total count + rate
- **Time Series**: Show streaming workload patterns
- **Correlation**: Overlay with `proxy_cache_backpressure_ratio`

#### AI Workload Indicators
- **High rate (>50/s)**: Heavy streaming AI usage (GPT-4, Claude)
- **Low rate (<10/s)**: Batch/synchronous AI APIs
- **Zero**: No streaming workloads

---

### `proxy_http2_active_sessions`
**Type**: Gauge  
**Range**: 0 to unbounded  
**Update**: Every 10 seconds  
**Description**: Current number of active HTTP/2 sessions

#### Purpose
Tracks HTTP/2 session count for multiplexing analysis. Compare with `proxy_http2_connection_reuse_avg` to understand connection efficiency.

#### Query Examples

```promql
# Current HTTP/2 sessions
proxy_http2_active_sessions

# HTTP/2 adoption rate (vs total connections)
(proxy_http2_active_sessions / proxy_active_connections) * 100

# Average streams per session (alternative calculation)
proxy_http2_total_streams_total / proxy_http2_active_sessions

# Alert on low HTTP/2 usage
(proxy_http2_active_sessions / proxy_active_connections) < 0.5

# HTTP/2 session growth
rate(proxy_http2_active_sessions[5m])
```

---

## Tier 3: Operational Debugging Metrics

These metrics help diagnose operational issues and fine-tune sockproxy configuration.

---

### `proxy_cache_drain_partial_total`
**Type**: Counter (monotonic)  
**Update**: Every 10 seconds  
**Description**: Total partial cache drain events (EAGAIN/EWOULDBLOCK during flush)

#### Purpose
Counts how many times sockproxy encountered flow control when flushing cached data to clients. High values indicate network congestion or slow clients.

#### Query Examples

```promql
# Total partial drains since startup
proxy_cache_drain_partial_total

# Partial drains per minute
rate(proxy_cache_drain_partial_total[1m]) * 60

# Partial drains in last hour
increase(proxy_cache_drain_partial_total[1h])

# Correlation with backpressure
rate(proxy_cache_drain_partial_total[5m]) and proxy_cache_backpressure_ratio > 0.2

# Alert on excessive partial drains
rate(proxy_cache_drain_partial_total[5m]) * 60 > 50
```

#### Troubleshooting
| Rate | Diagnosis | Action |
|------|-----------|--------|
| 0-10/min | Normal operation | No action needed |
| 10-50/min | Moderate flow control | Monitor network conditions |
| >50/min | Excessive blocking | Check client network bandwidth, investigate slow clients |

---

### `proxy_graceful_close_total`
**Type**: Counter (monotonic)  
**Update**: Every 10 seconds  
**Description**: Total graceful connection closes with pending cache drain

#### Purpose
Counts connections that closed gracefully while still having data in the cache buffer. Ensures no data loss during shutdown.

#### Query Examples

```promql
# Total graceful closes since startup
proxy_graceful_close_total

# Graceful closes per minute
rate(proxy_graceful_close_total[1m]) * 60

# Graceful closes in last hour
increase(proxy_graceful_close_total[1h])

# Percentage of closes that are graceful (vs total connection close rate)
rate(proxy_graceful_close_total[5m]) / rate(proxy_active_connections[5m])

# Alert on high graceful close rate (potential issue)
rate(proxy_graceful_close_total[5m]) * 60 > 20
```

#### Interpretation
- **High rate**: Normal for long-lived streaming connections closing with pending data
- **Zero rate**: Either no closes or abrupt terminations (check error logs)
- **Sudden spike**: Possible mass client disconnect event

---

### `proxy_conversation_ttl_expired_total`
**Type**: Counter (monotonic)  
**Update**: Every 10 seconds  
**Description**: Total conversation mappings expired due to TTL timeout

#### Purpose
Counts how many session affinity mappings were removed due to inactivity timeout. Helps tune `CONVERSATION_TTL` settings.

#### Query Examples

```promql
# Total expirations since startup
proxy_conversation_ttl_expired_total

# Expirations per minute
rate(proxy_conversation_ttl_expired_total[1m]) * 60

# Expirations in last hour
increase(proxy_conversation_ttl_expired_total[1h])

# Expiration rate vs session creation rate
rate(proxy_conversation_ttl_expired_total[5m]) / rate(proxy_conversation_sessions[5m])

# Alert on excessive expirations (>100/min)
rate(proxy_conversation_ttl_expired_total[5m]) * 60 > 100
```

#### Tuning Guide
| Expiration Rate | Action | Recommended TTL |
|-----------------|--------|-----------------|
| Very high (>100/min) | Increase TTL | 300-600s (5-10 min) |
| Moderate (10-100/min) | Current TTL okay | 180-300s (3-5 min) |
| Low (<10/min) | Consider decreasing TTL | 60-180s (1-3 min) |

#### Impact on Session Affinity
```promql
# Calculate optimal TTL based on expiration rate
# Goal: Keep expiration rate < 10% of session creation rate
(rate(proxy_conversation_ttl_expired_total[5m]) / rate(proxy_conversation_sessions[5m])) < 0.10
```

---

## Tier 4: L7 HTTP Metrics ⭐ NEW

These metrics expose HTTP-layer visibility added in phase 4 of the metrics expansion. They complement the existing connection-level metrics with response-level observability.

---

### `proxy_http_responses_total`
**Type**: Counter (monotonic)  
**Update**: Every 10 seconds  
**Description**: Total number of HTTP responses proxied through sockproxy

#### Query Examples

```promql
# HTTP response rate
rate(proxy_http_responses_total[1m])

# Total responses in last hour
increase(proxy_http_responses_total[1h])
```

---

### `proxy_http_responses_by_status_total`
**Type**: Counter (monotonic)  
**Labels**: `status_class` (`2xx`, `4xx`, `5xx`)  
**Update**: Every 10 seconds  
**Description**: HTTP responses grouped by status class for error rate tracking

#### Query Examples

```promql
# HTTP 5xx error rate
rate(proxy_http_responses_by_status_total{status_class="5xx"}[5m])

# Error percentage
rate(proxy_http_responses_by_status_total{status_class="5xx"}[5m]) /
rate(proxy_http_responses_total[5m]) * 100

# 4xx vs 5xx comparison
sum by (status_class) (rate(proxy_http_responses_by_status_total[5m]))
```

#### Thresholds
- **5xx rate > 1%**: Warning
- **5xx rate > 5%**: Critical

---

### `proxy_http_ttfb_seconds`
**Type**: Histogram  
**Update**: Every 10 seconds  
**Description**: Time-to-first-byte latency distribution for HTTP responses through sockproxy. Key metric for AI streaming workload responsiveness.

#### Query Examples

```promql
# TTFB p50 (median)
histogram_quantile(0.50, rate(proxy_http_ttfb_seconds_bucket[5m]))

# TTFB p95
histogram_quantile(0.95, rate(proxy_http_ttfb_seconds_bucket[5m]))

# TTFB p99
histogram_quantile(0.99, rate(proxy_http_ttfb_seconds_bucket[5m]))

# Alert on slow TTFB (>2s p95)
histogram_quantile(0.95, rate(proxy_http_ttfb_seconds_bucket[5m])) > 2
```

#### Thresholds
- **p95 < 0.5s**: Excellent
- **p95 < 1.0s**: Good
- **p95 < 2.0s**: Acceptable
- **p95 > 2.0s**: Investigate backend latency

---

### `proxy_pd_kv_params_overflow_total`
**Type**: Counter (monotonic)  
**Update**: Every 10 seconds  
**Description**: Events where P/D KV parameters exceeded buffer capacity. Should remain 0 under normal operation.

#### Query Examples

```promql
# Overflow events per minute
rate(proxy_pd_kv_params_overflow_total[1m]) * 60

# Alert on any overflow
proxy_pd_kv_params_overflow_total > 0
```

---

## Query Examples

### AI Workload Health Dashboard

```promql
# Overall AI performance score (0-100)
(
  (proxy_session_affinity_hit_rate * 40) +
  ((1 - proxy_cache_backpressure_ratio) * 30) +
  (min(proxy_http2_connection_reuse_avg / 10, 1) * 30)
) * 100

# Streaming workload intensity
rate(proxy_chunked_responses_total[5m]) * 60

# SSL/TLS overhead ratio
(proxy_active_ssl_connections / proxy_active_connections) * 100

# Session affinity effectiveness
proxy_session_affinity_hit_rate > 0.95
```

---

### Capacity Planning

```promql
# Connection headroom (assuming 10k limit)
10000 - proxy_active_connections

# Session table headroom (assuming 50k limit)
50000 - proxy_conversation_sessions

# Estimated time to capacity (hours)
(10000 - proxy_active_connections) / deriv(proxy_active_connections[1h]) / 3600

# HTTP/2 resource efficiency gain (vs HTTP/1.1)
proxy_http2_connection_reuse_avg * proxy_http2_active_sessions
```

---

### Performance Bottleneck Detection

```promql
# Identify bottleneck type
label_replace(
  label_replace(
    label_replace(
      max(proxy_cache_backpressure_ratio > 0.3), "bottleneck", "Cache Pressure", "", ""
    ), "bottleneck", "Low Session Affinity", "", ""
  ), "bottleneck", "Poor HTTP/2 Reuse", "", ""
)

# Backend saturation indicator
proxy_cache_backpressure_ratio > 0.3 and rate(proxy_cache_high_water_events_total[5m]) > 0.1

# Client-side bottleneck indicator
rate(proxy_cache_drain_partial_total[5m]) > 0.5 and proxy_cache_backpressure_ratio < 0.2
```

---

### SLA Monitoring

```promql
# Session affinity SLA (target: 95% over 30 days)
avg_over_time(proxy_session_affinity_hit_rate[30d]) > 0.95

# Backpressure SLA (target: <10% over 30 days)
avg_over_time(proxy_cache_backpressure_ratio[30d]) < 0.10

# HTTP/2 efficiency SLA (target: >5 over 30 days)
avg_over_time(proxy_http2_connection_reuse_avg[30d]) > 5
```

---

## Grafana Dashboard Examples

### Dashboard 1: AI Workload Overview

```json
{
  "dashboard": {
    "title": "LoxiLB Sockproxy - AI Workload Overview",
    "rows": [
      {
        "panels": [
          {
            "title": "Session Affinity Hit Rate",
            "type": "gauge",
            "targets": [
              {
                "expr": "proxy_session_affinity_hit_rate"
              }
            ],
            "fieldConfig": {
              "defaults": {
                "max": 1,
                "min": 0,
                "thresholds": {
                  "steps": [
                    { "value": 0, "color": "red" },
                    { "value": 0.80, "color": "yellow" },
                    { "value": 0.95, "color": "green" }
                  ]
                }
              }
            }
          },
          {
            "title": "Cache Backpressure Ratio",
            "type": "gauge",
            "targets": [
              {
                "expr": "proxy_cache_backpressure_ratio"
              }
            ],
            "fieldConfig": {
              "defaults": {
                "max": 1,
                "min": 0,
                "thresholds": {
                  "steps": [
                    { "value": 0, "color": "green" },
                    { "value": 0.1, "color": "yellow" },
                    { "value": 0.3, "color": "red" }
                  ]
                }
              }
            }
          },
          {
            "title": "HTTP/2 Connection Reuse",
            "type": "stat",
            "targets": [
              {
                "expr": "proxy_http2_connection_reuse_avg"
              }
            ],
            "fieldConfig": {
              "defaults": {
                "thresholds": {
                  "steps": [
                    { "value": 0, "color": "red" },
                    { "value": 5, "color": "yellow" },
                    { "value": 10, "color": "green" }
                  ]
                }
              }
            }
          }
        ]
      }
    ]
  }
}
```

---

### Dashboard 2: Connection Statistics

**Panels**:
1. **Active Connections** (Time Series)
   - Query: `proxy_active_connections`
   - Visualization: Area graph with max line
   
2. **SSL vs Non-SSL** (Pie Chart)
   - Queries:
     - SSL: `proxy_active_ssl_connections`
     - Non-SSL: `proxy_active_connections - proxy_active_ssl_connections`
   
3. **HTTP/2 Sessions** (Time Series)
   - Query: `proxy_http2_active_sessions`
   - Overlay: `proxy_http2_total_streams_total` (secondary Y-axis)
   
4. **Conversation Sessions** (Stat + Sparkline)
   - Query: `proxy_conversation_sessions`
   - Sparkline: Last 30 minutes

---

### Dashboard 3: Streaming AI Workloads

**Panels**:
1. **Chunked Response Rate** (Time Series)
   - Query: `rate(proxy_chunked_responses_total[1m]) * 60`
   - Unit: responses/min
   
2. **Streaming Percentage** (Gauge)
   - Query: `(rate(proxy_chunked_responses_total[5m]) / rate(total_requests[5m])) * 100`
   - Unit: Percentage
   
3. **Cache Pressure During Streaming** (Time Series)
   - Queries:
     - Primary: `proxy_cache_backpressure_ratio`
     - Secondary: `rate(proxy_chunked_responses_total[1m])`
   - Correlation panel

---

### Dashboard 4: Troubleshooting

**Panels**:
1. **Cache High Water Events** (Time Series)
   - Query: `rate(proxy_cache_high_water_events_total[1m]) * 60`
   - Unit: events/min
   - Alert threshold line
   
2. **Partial Cache Drains** (Time Series)
   - Query: `rate(proxy_cache_drain_partial_total[1m]) * 60`
   - Unit: drains/min
   
3. **Graceful Closes** (Time Series)
   - Query: `rate(proxy_graceful_close_total[1m]) * 60`
   - Unit: closes/min
   
4. **Session Expirations** (Time Series)
   - Query: `rate(proxy_conversation_ttl_expired_total[1m]) * 60`
   - Unit: expirations/min

---

### Dashboard 5: Performance Heatmap

**Panel Type**: Heatmap  
**Query**:
```promql
# Backpressure intensity over time
proxy_cache_backpressure_ratio
```

**Configuration**:
- X-axis: Time
- Y-axis: Backpressure ratio buckets (0.0-0.1, 0.1-0.2, ..., 0.9-1.0)
- Color: Intensity (green = low, red = high)

---

## Alerting Rules

### Critical Alerts

```yaml
groups:
  - name: sockproxy_critical
    interval: 30s
    rules:
      - alert: HighCacheBackpressure
        expr: proxy_cache_backpressure_ratio > 0.3
        for: 5m
        labels:
          severity: critical
          component: sockproxy
        annotations:
          summary: "High cache backpressure detected"
          description: "{{ $value | humanizePercentage }} of connections under backpressure"
          runbook: "https://docs.loxilb.io/sockproxy-backpressure"

      - alert: LowSessionAffinityHitRate
        expr: proxy_session_affinity_hit_rate < 0.80
        for: 10m
        labels:
          severity: critical
          component: sockproxy
        annotations:
          summary: "Session affinity hit rate below 80%"
          description: "Current hit rate: {{ $value | humanizePercentage }}"
          impact: "AI conversation context loss, increased latency"

      - alert: NoHTTP2Multiplexing
        expr: proxy_http2_connection_reuse_avg < 2
        for: 15m
        labels:
          severity: critical
          component: sockproxy
        annotations:
          summary: "HTTP/2 connection reuse critically low"
          description: "Average streams per connection: {{ $value }}"
          impact: "Excessive TLS handshakes, high CPU usage"

      - alert: FrequentCacheHighWaterTriggers
        expr: rate(proxy_cache_high_water_events_total[5m]) * 60 > 10
        for: 5m
        labels:
          severity: critical
          component: sockproxy
        annotations:
          summary: "Frequent cache backpressure triggers"
          description: "{{ $value }} events per minute"
          action: "Scale backend capacity immediately"
```

---

### Warning Alerts

```yaml
  - name: sockproxy_warnings
    interval: 1m
    rules:
      - alert: ElevatedCacheBackpressure
        expr: proxy_cache_backpressure_ratio > 0.1
        for: 10m
        labels:
          severity: warning
          component: sockproxy
        annotations:
          summary: "Elevated cache backpressure"
          description: "{{ $value | humanizePercentage }} of connections affected"

      - alert: SessionAffinityDegraded
        expr: proxy_session_affinity_hit_rate < 0.90
        for: 15m
        labels:
          severity: warning
          component: sockproxy
        annotations:
          summary: "Session affinity hit rate degraded"
          description: "Current hit rate: {{ $value | humanizePercentage }}"

      - alert: LowHTTP2Efficiency
        expr: proxy_http2_connection_reuse_avg < 5
        for: 20m
        labels:
          severity: warning
          component: sockproxy
        annotations:
          summary: "HTTP/2 connection reuse below target"
          description: "Average streams per connection: {{ $value }}"

      - alert: HighPartialDrainRate
        expr: rate(proxy_cache_drain_partial_total[5m]) * 60 > 50
        for: 10m
        labels:
          severity: warning
          component: sockproxy
        annotations:
          summary: "High partial cache drain rate"
          description: "{{ $value }} drains per minute"
          cause: "Network congestion or slow clients"

      - alert: ExcessiveSessionExpirations
        expr: rate(proxy_conversation_ttl_expired_total[5m]) * 60 > 100
        for: 10m
        labels:
          severity: warning
          component: sockproxy
        annotations:
          summary: "High session expiration rate"
          description: "{{ $value }} expirations per minute"
          action: "Consider increasing CONVERSATION_TTL"
```

---

### Informational Alerts

```yaml
  - name: sockproxy_info
    interval: 5m
    rules:
      - alert: HighStreamingWorkload
        expr: rate(proxy_chunked_responses_total[5m]) * 60 > 100
        for: 30m
        labels:
          severity: info
          component: sockproxy
        annotations:
          summary: "High streaming AI workload detected"
          description: "{{ $value }} chunked responses per minute"

      - alert: HighConnectionCount
        expr: proxy_active_connections > 8000
        for: 10m
        labels:
          severity: info
          component: sockproxy
        annotations:
          summary: "High connection count"
          description: "{{ $value }} active connections (80% of capacity)"

      - alert: LowSSLAdoption
        expr: (proxy_active_ssl_connections / proxy_active_connections) < 0.5
        for: 1h
        labels:
          severity: info
          component: sockproxy
        annotations:
          summary: "Low SSL/TLS adoption"
          description: "Only {{ $value | humanizePercentage }} of connections encrypted"
```

---

## Troubleshooting Guide

### High Cache Backpressure

**Symptoms**: `proxy_cache_backpressure_ratio > 0.3`

**Diagnosis Steps**:
1. Check backend health: `healthy_endpoints_count`
2. Check backend load: `endpoint_traffic_bytes` distribution
3. Check network latency: RTT metrics
4. Check cache configuration: `CACHE_MAX_SIZE`, `CACHE_HIGH_WATER`

**Resolution**:
```bash
# Scale backends
kubectl scale deployment backend --replicas=10

# Increase cache size (if memory available)
# Edit sockproxy.c: #define CACHE_MAX_SIZE (256 * 1024)

# Adjust high water mark
# Edit sockproxy.c: #define CACHE_HIGH_WATER (192 * 1024)
```

---

### Low Session Affinity Hit Rate

**Symptoms**: `proxy_session_affinity_hit_rate < 0.90`

**Diagnosis Steps**:
1. Check TTL expirations: `rate(proxy_conversation_ttl_expired_total[5m])`
2. Check session count: `proxy_conversation_sessions`
3. Check hash collision rate (logs)
4. Verify client IP stability

**Resolution**:
```bash
# Increase TTL (sockproxy.c)
#define CONVERSATION_TTL 300  // 5 minutes

# Increase hash table size
#define CONVERSATION_TABLE_SIZE 65536  // 64k entries

# Enable sticky session logging
export SOCKPROXY_DEBUG=1
```

---

### Poor HTTP/2 Multiplexing

**Symptoms**: `proxy_http2_connection_reuse_avg < 5`

**Diagnosis Steps**:
1. Verify HTTP/2 is enabled: `proxy_http2_active_sessions > 0`
2. Check ALPN negotiation (logs)
3. Check client HTTP/2 support
4. Verify connection keep-alive settings

**Resolution**:
```bash
# Client-side fix (example: curl)
curl --http2 --keepalive-time 60 https://api.example.com

# Server-side: Increase HTTP/2 max streams
# Edit sockproxy_h2.c: nghttp2_settings_entry settings[] = {
#   {NGHTTP2_SETTINGS_MAX_CONCURRENT_STREAMS, 128}
# }

# Verify ALPN
openssl s_client -connect api.example.com:443 -alpn h2
```

---

### Excessive Partial Drains

**Symptoms**: `rate(proxy_cache_drain_partial_total[5m]) * 60 > 50`

**Diagnosis Steps**:
1. Check client bandwidth: Network monitoring tools
2. Check TCP socket buffer sizes: `sysctl net.ipv4.tcp_wmem`
3. Check for slow clients: Client-side metrics
4. Verify no network congestion: Packet loss metrics

**Resolution**:
```bash
# Increase socket send buffer
sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"

# Enable TCP window scaling
sysctl -w net.ipv4.tcp_window_scaling=1

# Increase cache drain timeout
# Edit sockproxy.c: #define CACHE_DRAIN_TIMEOUT_MS 5000
```

---

## Performance Tuning

### Optimizing Session Affinity

**Goal**: Achieve >95% hit rate

```c
// sockproxy.c tuning parameters
#define CONVERSATION_TTL 300           // 5 minutes (increase for long sessions)
#define CONVERSATION_TABLE_SIZE 65536  // 64k entries (increase for high concurrency)
#define CONVERSATION_CLEANUP_INTERVAL 60  // 1 minute (decrease for faster cleanup)
```

**Validation**:
```promql
# Target hit rate
proxy_session_affinity_hit_rate > 0.95

# Verify TTL is sufficient (expirations should be <10% of sessions)
rate(proxy_conversation_ttl_expired_total[5m]) / rate(proxy_conversation_sessions[5m]) < 0.10
```

---

### Optimizing Cache Backpressure

**Goal**: Minimize backpressure ratio <10%

```c
// sockproxy.c tuning parameters
#define CACHE_MAX_SIZE (256 * 1024)      // 256KB (increase for high-bandwidth)
#define CACHE_HIGH_WATER (192 * 1024)    // 75% of max (decrease for earlier backpressure)
#define CACHE_LOW_WATER (128 * 1024)     // 50% of max (hysteresis zone)
```

**Validation**:
```promql
# Target backpressure
proxy_cache_backpressure_ratio < 0.10

# Verify high water events are infrequent
rate(proxy_cache_high_water_events_total[5m]) * 60 < 5
```

---

### Optimizing HTTP/2 Multiplexing

**Goal**: Achieve >10 streams per session

```c
// sockproxy_h2.c tuning parameters
nghttp2_settings_entry settings[] = {
    {NGHTTP2_SETTINGS_MAX_CONCURRENT_STREAMS, 128},  // Increase from default 100
    {NGHTTP2_SETTINGS_INITIAL_WINDOW_SIZE, 65535},   // Default
    {NGHTTP2_SETTINGS_MAX_FRAME_SIZE, 16384}         // Default
};
```

**Validation**:
```promql
# Target reuse average
proxy_http2_connection_reuse_avg > 10

# Verify HTTP/2 adoption
(proxy_http2_active_sessions / proxy_active_connections) > 0.7
```

---

## Best Practices

### Metric Collection
- **Scrape Interval**: Use 10-second intervals matching `RunSockproxyMetrics()` update frequency
- **Retention**: Keep high-resolution data for 7 days, downsample to 1m for 30 days
- **Cardinality**: Sockproxy metrics have zero labels (low cardinality), safe for long-term storage

### Alerting Strategy
- **Tier 1 Metrics**: Alert immediately on threshold crossings (5-10 minute grace period)
- **Tier 2 Metrics**: Alert on sustained trends (15-30 minute grace period)
- **Tier 3 Metrics**: Use for debugging, no proactive alerts (log-based analysis)

### Dashboard Design
- **Executive View**: Show Tier 1 metrics only (3 gauges)
- **Operations View**: Show Tier 1 + Tier 2 (8 panels)
- **Debug View**: Show all metrics + correlations (15+ panels)

### Query Optimization
```promql
# Use recording rules for complex calculations
- record: sockproxy_ai_health_score
  expr: |
    (
      (proxy_session_affinity_hit_rate * 40) +
      ((1 - proxy_cache_backpressure_ratio) * 30) +
      (min(proxy_http2_connection_reuse_avg / 10, 1) * 30)
    ) * 100

# Then query the recording rule
sockproxy_ai_health_score
```

---

## Integration Examples

### Kubernetes Annotations

```yaml
apiVersion: v1
kind: Service
metadata:
  name: loxilb-sockproxy
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
    prometheus.io/interval: "10s"
```

---

### Prometheus Recording Rules

```yaml
groups:
  - name: sockproxy_recording_rules
    interval: 30s
    rules:
      # AI workload health score
      - record: sockproxy_ai_health_score
        expr: |
          (
            (proxy_session_affinity_hit_rate * 40) +
            ((1 - proxy_cache_backpressure_ratio) * 30) +
            (min(proxy_http2_connection_reuse_avg / 10, 1) * 30)
          ) * 100

      # Connection efficiency ratio
      - record: sockproxy_connection_efficiency
        expr: |
          proxy_http2_connection_reuse_avg * proxy_http2_active_sessions /
          proxy_active_connections

      # Session affinity miss rate
      - record: sockproxy_session_miss_rate
        expr: 1 - proxy_session_affinity_hit_rate

      # Streaming workload ratio
      - record: sockproxy_streaming_ratio
        expr: |
          rate(proxy_chunked_responses_total[5m]) /
          rate(total_requests[5m])
```

---

### Grafana Alerting (Unified Alerting)

```yaml
apiVersion: 1
groups:
  - name: sockproxy
    interval: 1m
    rules:
      - uid: sockproxy_backpressure
        title: High Cache Backpressure
        condition: A
        data:
          - refId: A
            queryType: ''
            relativeTimeRange:
              from: 300
              to: 0
            datasourceUid: prometheus
            model:
              expr: proxy_cache_backpressure_ratio > 0.3
              intervalMs: 1000
              maxDataPoints: 43200
        noDataState: NoData
        execErrState: Alerting
        for: 5m
        annotations:
          description: "{{ $value | humanizePercentage }} of connections under backpressure"
        labels:
          severity: critical
```

---

## Appendix: Metric Reference

| Metric Name | Type | Tier | Unit | Target/Threshold |
|-------------|------|------|------|------------------|
| `proxy_cache_backpressure_ratio` | Gauge | 1 | Ratio (0-1) | < 0.1 |
| `proxy_session_affinity_hit_rate` | Gauge | 1 | Ratio (0-1) | > 0.95 |
| `proxy_http2_connection_reuse_avg` | Gauge | 1 | Count | > 10 |
| `proxy_cache_high_water_events_total` | Counter | 1 | Events | <5/min |
| `proxy_active_connections` | Gauge | 2 | Count | < 80% capacity |
| `proxy_active_ssl_connections` | Gauge | 2 | Count | Monitor trend |
| `proxy_conversation_sessions` | Gauge | 2 | Count | < 90% capacity |
| `proxy_chunked_responses_total` | Counter | 2 | Responses | Monitor trend |
| `proxy_http2_active_sessions` | Gauge | 2 | Count | Monitor trend |
| `proxy_cache_drain_partial_total` | Counter | 3 | Events | <50/min |
| `proxy_graceful_close_total` | Counter | 3 | Closes | Monitor trend |
| `proxy_conversation_ttl_expired_total` | Counter | 3 | Expirations | <100/min |
| `proxy_http_responses_total` | Counter | 4 | Responses | Monitor trend |
| `proxy_http_responses_by_status_total` | Counter | 4 | Responses | 5xx < 1% |
| `proxy_http_ttfb_seconds` | Histogram | 4 | Seconds | p95 < 1.0s |
| `proxy_pd_kv_params_overflow_total` | Counter | 4 | Events | 0/min |

---

## Support

For issues or questions:
- **GitHub**: https://github.com/loxilb-io/loxilb
- **Documentation**: https://loxilb-io.github.io/loxilbdocs/
- **Slack**: loxilb.slack.com
- **Metrics Design**: `SOCKPROXY_METRICS_COMPACT_DESIGN.md`
- **Code**: `api/prometheus/sockproxy_metrics.go`, `loxilb-ebpf/common/sockproxy.c`

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01-17 | Initial sockproxy metrics documentation |
| 2.0 | 2026-03-17 | Added Tier 4: L7 HTTP metrics (proxy_http_responses_total, proxy_http_responses_by_status_total, proxy_http_ttfb_seconds, proxy_pd_kv_params_overflow_total); updated Metric Categories table and Appendix |

---

**Document Status**: Production Ready  
**Last Updated**: 2026-03-17  
**Maintained By**: LoxiLB Enterprise Team
