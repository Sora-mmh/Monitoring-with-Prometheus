# Monitoring with Prometheus 

Prometheus is an **open-source monitoring system and alerting toolkit**, designed especially for dynamic containerized environments like Kubernetes and Docker Swarm, but also perfectly suitable for traditional server monitoring.

***

## 1. Why Use Prometheus?

- Monitors highly dynamic infrastructures comprising hundreds or thousands of containers and services.
- Provides consistent visibility across multiple layers: infrastructure, platform, and application.
- Helps quickly identify the root cause of issues rather than manually troubleshooting distributed components.
- Continuously monitors services in real-time to detect degradation or failure.
- Triggers alerts on fault conditions, potentially before end-users experience issues.
- Enables proactive problem resolution, improving system reliability.

***

## 2. Prometheus Architecture Overview

### Main Components

- **Prometheus Server**  
  The core component responsible for data collection (scraping) and storage. It pulls metrics from configured targets periodically according to the `scrape_interval`.

- **Targets**  
  Applications, services, nodes, or containers exposing metrics on HTTP endpoints (typically `/metrics`).

- **Metrics**  
  Time series data points with types:  
  - *Counter*: cumulative count of events (e.g., request count).  
  - *Gauge*: current value, e.g., CPU usage or memory usage.  
  - *Histogram*: distribution of observed values, e.g., request durations.

- **Exporters**  
  Services that expose metrics in Prometheus format for applications or systems that do not natively expose `/metrics`. Examples: node exporter for Linux OS metrics, MySQL exporter for databases.

- **Push Gateway**  
  For short-lived jobs (e.g., batch jobs), which cannot be scraped reliably, metrics can be pushed to the Push Gateway, which Prometheus scrapes later.

- **Time Series Database (TSDB)**  
  Efficient local storage of scraped time series data with support for retention policies and data compaction.

- **Alertmanager**  
  Manages alerts sent from Prometheus, handling deduplication, grouping, routing, silencing, and dispatching notifications to channels like email, Slack, PagerDuty.

- **Client Libraries**  
  Provided for different programming languages to instrument your own applications and expose custom metrics.

- **PromQL**  
  A powerful query language to select, aggregate, and analyze time series data.

***

### Data Flow

1. Prometheus scrapes metrics endpoints from Targets or Exporters.
2. Stores data as time series in TSDB.
3. Applies rules and alerting conditions.
4. Sends alerts to Alertmanager if conditions are met.
5. Alertmanager manages notification routing.

***

## 3. Sample Prometheus Configuration (`prometheus.yml`)

```yaml
global:
  scrape_interval: 15s         # scrape targets every 15 seconds
  evaluation_interval: 15s     # evaluate rules every 15 seconds

rule_files:
  - first.rules                # aggregate or alerting rules
  - second.rules

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']   # Prometheus self-monitoring

  - job_name: node_exporter
    scrape_interval: 1m
    scrape_timeout: 1m
    static_configs:
      - targets: ['localhost:9100']   # Node Exporter for OS metrics
```

- Default for each job:  
  - `metrics_path: "/metrics"`  
  - `scheme: "http"`

***

## 4. Using Exporters for External Systems

- Exporters convert existing application/system metrics into Prometheus format.
- Often deployed as Docker containers or Kubernetes sidecars.
- Examples:  
  - Node Exporter (Linux OS stats)  
  - MySQL Exporter  
- Service discovery ensures Prometheus discovers and scrapes new exporters dynamically.
- Kubernetes `ServiceMonitor` CRDs extend Prometheus Operator to detect new exporters automatically.

***

## 5. Alerting and Alertmanager

### Prometheus Alerts

- Defined with **PromQL expressions** that specify when an alert should fire.
- Alerts have metadata: severity, description, duration before firing (`for`).
- Example alert rule:

```yaml
groups:
- name: example_alerts
  rules:
  - alert: HighCPUUsage
    expr: node_cpu_seconds_total > 0.8
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High CPU usage detected"
      description: "CPU usage is over 80% for more than 5 minutes."
```

### Alertmanager

- Receives alerts from Prometheus when firing.
- Deduplicates, groups, silences, and routes alerts.
- Routing is configured based on labels (e.g., severity).
- Supports receivers like email, Slack, PagerDuty.
- Configuration is YAML-driven and includes global settings, routes, inhibit rules, and receivers.

***

## 6. Installing Prometheus in Kubernetes (EKS)

- Three main ways:  
  - Manual YAML manifests (laborious).  
  - Prometheus Operator (manages lifecycle).  
  - Helm charts (simplifies installing an operator + components).
  
- Operator manages Prometheus servers, Alertmanager instances, exporters, and config.

***

## 7. Visualizing Data – Prometheus UI and Grafana

### Prometheus UI

- Provides metric explorer and query execution.
- Lists active scrape targets.
- Displays current configuration and alert status.

### Grafana

- Powerful dashboarding and visualization tool.
- Integrates with Prometheus as a data source.
- Supports complex panels with PromQL queries.
- Dashboards can be created, edited, cloned, and shared.
- Provides alert visualization.

***

## 8. Creating Dashboards and Testing Anomalies

- Create a new dashboard in Grafana and add panels with Prometheus queries.
- Use PromQL filters, for example to focus on certain status codes or pods.
- Test anomalies by generating load (e.g., repeated HTTP requests) to cause CPU spikes.
- Dashboards aid in pinpointing the resource usage per microservice/component.

***

## 9. Managing Users and Data Sources in Grafana

- Manage users and teams via Grafana UI.
- Configure multiple data sources (e.g., Prometheus, Alertmanager).
- Assign permissions and roles for access control.

***

## 10. Official Best Practices and Resources

For effective Prometheus monitoring, follow these official guidelines:

- Metric and Label Naming:  
  https://prometheus.io/docs/practices/naming/

- Instrumentation Guidelines:  
  https://prometheus.io/docs/practices/instrumentation/

- Consoles and Dashboards best practices:  
  https://prometheus.io/docs/practices/consoles/

- Alerting best practices:  
  https://prometheus.io/docs/practices/alerting/

- Pushgateway usage guidelines:  
  https://prometheus.io/docs/practices/pushing/
