# Monitoring-with-Prometheus

The listed project in this module covers:

---

### **1. Data Visualization**
- **Accessed Grafana** (default admin login via port-forward).  
- **Tested CPU spike** on a pod and watched the metric jump in Grafana.

---

### **2. Custom Alerting Rules**
- **Wrote YAML PrometheusRule** objects for:
  - High CPU (> 50 %)  
  - Pod crash loops  
- **Applied rules** with `kubectl apply -f` and saw them in **Prometheus Alerts** tab.

---

### **3. Alertmanager & Email Alerts**
- **Configured Alertmanager** secret with Gmail SMTP settings.  
- **Triggered test alert** (killed a pod) → **email received** within seconds.

---

### **4. Monitor Third-Party Apps**
| App | Steps |
|-----|-------|
| **Redis** | Deploy `prometheus-redis-exporter` via Helm.  <br> Created `PrometheusRule` for `RedisDown`. <br> Imported **Grafana dashboard ID 763** for instant graphs. |

---

### **5. Instrument Your Own App**
- **Node.js app** instrumented with `prom-client`:
  ```js
  const prom = require('prom-client');
  const collectDefault = prom.collectDefaultMetrics;
  collectDefault();
  ```
- **Docker image** built & pushed → **K8s Deployment + Service** deployed.  
- **ServiceMonitor** added so Prometheus scrapes `/metrics`.  
- **Built custom Grafana dashboard** showing `http_requests_total`, `response_time`, etc.
