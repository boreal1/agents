# Kubernetes Observability Patterns (LGTM Stack)

Detailed patterns for the LGTM stack: Loki, Grafana, Tempo, Mimir/Prometheus, plus
OpenTelemetry. Read this when you're actively setting up observability for a cluster.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prometheus / kube-prometheus-stack](#prometheus--kube-prometheus-stack)
3. [ServiceMonitor and PodMonitor](#servicemonitor-and-podmonitor)
4. [Loki Setup](#loki-setup)
5. [Tempo Setup](#tempo-setup)
6. [Grafana Datasources and Dashboards](#grafana-datasources-and-dashboards)
7. [OpenTelemetry Collector](#opentelemetry-collector)
8. [Alerting](#alerting)

## Architecture Overview

The recommended observability stack:

```
                 ┌──────────────────┐
   App pods ───▶ │  OTel Collector  │ ──▶ Tempo (traces)
                 │  (DaemonSet +    │ ──▶ Loki (logs)
                 │   Deployment)    │ ──▶ Mimir (metrics)
                 └──────────────────┘
                        ▲
                        │
                 ┌──────────────────┐
                 │   Prometheus     │ (scrapes /metrics endpoints)
                 └──────────────────┘
                        │
                        ▼
                 ┌──────────────────┐
                 │     Grafana      │ (unified UI for all three)
                 └──────────────────┘
```

The Prometheus operator provides CRDs (`ServiceMonitor`, `PodMonitor`,
`PrometheusRule`) that make scrape config and alerting declarative.

## Prometheus / kube-prometheus-stack

The `kube-prometheus-stack` Helm chart (formerly Prometheus Operator) is the
standard install. It bundles Prometheus, Grafana, Alertmanager, node-exporter,
and kube-state-metrics.

```yaml
# values.yaml for kube-prometheus-stack
fullnameOverride: prom

prometheus:
  prometheusSpec:
    retention: 15d
    retentionSize: "50GiB"
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        memory: 4Gi
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: managed-csi-premium  # AKS Premium SSD
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    # Allow scraping ServiceMonitors and PodMonitors from any namespace
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false

    # Remote write to Mimir for long-term storage
    remoteWrite:
      - url: http://mimir-distributor.observability:8080/api/v1/push
        writeRelabelConfigs:
          - sourceLabels: [__name__]
            regex: '(up|node_.*|kube_.*|container_.*)'
            action: keep

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: managed-csi
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
  config:
    route:
      receiver: 'slack'
      group_by: ['alertname', 'cluster', 'severity']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
    receivers:
      - name: 'slack'
        slack_configs:
          - api_url_file: /etc/alertmanager/secrets/slack/url
            channel: '#alerts'
            title: '{{ template "slack.title" . }}'

grafana:
  enabled: false  # Use the standalone Grafana below for LGTM
```

## ServiceMonitor and PodMonitor

For applications exposing Prometheus metrics:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  labels:
    app.kubernetes.io/name: myapp
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
  endpoints:
    - port: metrics       # the named port from the Service
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_node_name]
          targetLabel: node
        - sourceLabels: [__meta_kubernetes_namespace]
          targetLabel: namespace
```

For pods without a Service (e.g., DaemonSets):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-daemonset
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: my-daemonset
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
```

Always name the metrics port in your Service, not just specify a number — the
ServiceMonitor references the port name.

## Loki Setup

Use the `loki` Helm chart in microservices mode for production, or simple-scalable
mode for medium clusters:

```yaml
# values.yaml for loki
deploymentMode: SimpleScalable

loki:
  auth_enabled: false
  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: s3  # or azure
        schema: v13
        index:
          prefix: loki_index_
          period: 24h

  # Azure Blob storage for AKS
  storage:
    type: azure
    azure:
      account_name: mylokistorage
      container_name: loki

  limits_config:
    retention_period: 30d
    ingestion_rate_mb: 10
    ingestion_burst_size_mb: 20

backend:
  replicas: 3
  persistence:
    enabled: true
    size: 50Gi

read:
  replicas: 3

write:
  replicas: 3
  persistence:
    enabled: true
    size: 50Gi
```

For log collection, use Grafana Alloy (the modern replacement for Promtail):

```yaml
# alloy values
controller:
  type: daemonset

alloy:
  configMap:
    content: |-
      loki.source.kubernetes "pods" {
        targets    = discovery.kubernetes.pods.targets
        forward_to = [loki.write.default.receiver]
      }

      loki.write "default" {
        endpoint {
          url = "http://loki-gateway.observability/loki/api/v1/push"
        }
      }
```

## Tempo Setup

```yaml
# values.yaml for tempo
storage:
  trace:
    backend: azure  # or s3
    azure:
      container_name: tempo
      storage_account_name: mytempostorage

distributor:
  replicas: 3
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

ingester:
  replicas: 3
  persistence:
    enabled: true
    size: 50Gi

querier:
  replicas: 2

queryFrontend:
  replicas: 2

compactor:
  replicas: 1
```

## Grafana Datasources and Dashboards

Provision datasources via ConfigMap (recognized by Grafana sidecar):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  labels:
    grafana_datasource: "1"
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        access: proxy
        url: http://prom-prometheus.observability:9090
        isDefault: true
        jsonData:
          exemplarTraceIdDestinations:
            - name: trace_id
              datasourceUid: tempo
      - name: Loki
        type: loki
        access: proxy
        url: http://loki-gateway.observability
        jsonData:
          derivedFields:
            - datasourceUid: tempo
              matcherRegex: 'trace_id=(\w+)'
              name: TraceID
              url: '$${__value.raw}'
      - name: Tempo
        type: tempo
        access: proxy
        url: http://tempo-query-frontend.observability:3100
        uid: tempo
        jsonData:
          tracesToLogsV2:
            datasourceUid: 'loki'
            tags:
              - key: 'service.name'
                value: 'service_name'
          tracesToMetrics:
            datasourceUid: 'prometheus'
```

This config wires up cross-signal correlation: clicking a trace ID in logs jumps
to Tempo, clicking a span goes back to Loki for the surrounding logs.

## OpenTelemetry Collector

Deploy the collector as both a Deployment (gateway) and DaemonSet (agent):

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: gateway
spec:
  mode: deployment
  replicas: 3
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        timeout: 10s
        send_batch_size: 1024
      memory_limiter:
        check_interval: 1s
        limit_percentage: 80
      resource:
        attributes:
          - key: cluster
            value: prod-aks
            action: upsert

    exporters:
      otlp/tempo:
        endpoint: tempo-distributor.observability:4317
        tls:
          insecure: true
      prometheusremotewrite:
        endpoint: http://mimir-distributor.observability:8080/api/v1/push
      loki:
        endpoint: http://loki-gateway.observability/loki/api/v1/push

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, resource, batch]
          exporters: [otlp/tempo]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, resource, batch]
          exporters: [prometheusremotewrite]
        logs:
          receivers: [otlp]
          processors: [memory_limiter, resource, batch]
          exporters: [loki]
```

Apps point their `OTEL_EXPORTER_OTLP_ENDPOINT` at the gateway service.

## Alerting

PrometheusRule CRDs let you define alerts declaratively:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
spec:
  groups:
    - name: myapp.rules
      interval: 30s
      rules:
        - alert: MyAppHighErrorRate
          expr: |
            sum(rate(http_requests_total{app="myapp",status=~"5.."}[5m]))
              /
            sum(rate(http_requests_total{app="myapp"}[5m]))
              > 0.05
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "High error rate on myapp"
            description: "Error rate is {{ $value | humanizePercentage }} over 5m."
            runbook_url: "https://runbooks.example.com/myapp/high-error-rate"

        - alert: MyAppHighLatency
          expr: |
            histogram_quantile(0.99,
              sum by(le)(rate(http_request_duration_seconds_bucket{app="myapp"}[5m]))
            ) > 1
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High p99 latency on myapp"

        - alert: MyAppPodCrashLooping
          expr: |
            rate(kube_pod_container_status_restarts_total{pod=~"myapp-.*"}[15m]) > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"
```

The standard SRE alerting pattern uses RED metrics:
- **R**ate — requests per second
- **E**rror rate — % of failed requests
- **D**uration — request latency (p50, p95, p99)

Plus USE for resources:
- **U**tilization — % of capacity used
- **S**aturation — work queued
- **E**rrors — error count
