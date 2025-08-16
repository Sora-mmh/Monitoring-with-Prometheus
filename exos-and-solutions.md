# Monitoring with Prometheus — Exercises & Solutions

**Repository to use:** [GitLab – prometheus-exercises](https://gitlab.com/twn-devops-bootcamp/latest/16-prometheus/prometheus-exercises)

## Context

You and your team are running a setup in a K8s cluster with a Java application that uses MySQL DB and is accessible from a browser using Ingress. Sometimes you have issues causing MySQL DB to not be accessible or Ingress problems, meaning users can't access your Java application. 

**Current Problems:**
- Hours of troubleshooting to identify issues
- Only aware of problems when users report them
- Reactive rather than proactive problem-solving
- No visibility into cluster health

**Goal:** Implement Prometheus monitoring to increase visibility, get immediate alerts when issues happen, and enable proactive problem-fixing.

---

## Exercise 1: Deploy your Application and Prepare the Setup

**Task:**
- Create a K8s cluster
- Deploy MySQL database with 2 replicas using Helm chart
- Deploy Java Maven application with 3 replicas that talks to MySQL DB
- Deploy Nginx Ingress Controller using Helm chart
- Configure access to Java application using Ingress rule

**Solution:**

**Preparation:**
```sh
# Create K8s cluster on LKE and set kubeconfig file
chmod 400 ~/Downloads/monitoring-kubeconfig.yaml
export KUBECONFIG=~/Downloads/monitoring-kubeconfig.yaml

# Create docker-registry secret for private Docker Hub access
DOCKER_REGISTRY_SERVER=docker.io
DOCKER_USER=your_dockerID_same_as_for_docker_login
DOCKER_EMAIL=your_dockerhub_email_same_as_for_docker_login
DOCKER_PASSWORD=your_dockerhub_pwd_same_as_for_docker_login

kubectl create secret docker-registry my-registry-key \
--docker-server=$DOCKER_REGISTRY_SERVER \
--docker-username=$DOCKER_USER \
--docker-password=$DOCKER_PASSWORD \
--docker-email=$DOCKER_EMAIL
```

**Implementation:**
```sh
# Execute Ansible playbook to deploy java and mysql apps in k8s cluster
ansible-playbook 1-configure-k8s.yaml
```

**Troubleshooting:**
```sh
# If you get nginx-controller-admission webhook error:
kubectl get ValidatingWebhookConfiguration  # gives you the name
kubectl delete ValidatingWebhookConfiguration {name}
```

**Note:** You can use the Ansible playbook from Ansible exercises 7 & 8 with adjustments for this setup.

---

## Exercise 2: Start Monitoring your Applications

**Task:**
- Deploy Prometheus Operator in your cluster using Helm chart
- Configure metrics scraping for Nginx Controller
- Configure metrics scraping for MySQL
- Configure metrics scraping for Java application (port 8081, NOT /metrics endpoint)
- Verify in Prometheus UI that all three application metrics are being collected

**Solution:**

**Deploy Prometheus Stack:**
```sh
# Add Prometheus community Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace and install Prometheus stack
kubectl create namespace monitoring
helm install monitoring-stack prometheus-community/kube-prometheus-stack -n monitoring

# Access Prometheus UI to view targets
kubectl -n monitoring port-forward svc/monitoring-stack-kube-prom-prometheus 9090:9090
```

**Verify Prometheus Installation:**
```sh
# Open browser to check targets
http://127.0.0.1:9090/targets
```

**Prepare Applications for Monitoring:**
```sh
# Clean up existing installations
helm uninstall mysql-release
helm uninstall ingress-controller -n ingress

# Check ServiceMonitor selector (may vary with Prometheus stack versions)
kubectl get prometheuses.monitoring.coreos.com
kubectl get prometheuses.monitoring.coreos.com {crd-name} -o yaml | grep serviceMonitorSelector -A 2
```

**Build Java Application with Metrics:**
```sh
# Check out java app code with prometheus client
git checkout main

# Build new jar with metrics support
gradle clean build

# Build and push docker image
docker build -t {docker-hub-id}:{repo-name}:{tag} .
docker push {docker-hub-id}:{repo-name}:{tag}

# Update image name in kubernetes-manifests/java-app.yaml file
```

**Configure Metrics Scraping:**
```sh
# Add metrics scraping to nginx, mysql and java apps
# Ensure correct IP addressing in ingress YAML file
ansible-playbook 2-configure-k8s.yaml

# Verify new targets are added in Prometheus UI
http://127.0.0.1:9090/targets
```

**Note:** Some cloud-native applications have built-in metrics scraping configuration and may not require separate exporter applications. Check the application chart documentation first.

---

## Exercise 3: Configure Alert Rules

**Task:**
Configure alert rules for critical issues:
- **Nginx-ingress:** More than 5% of HTTP requests have status 4xx
- **MySQL:** All MySQL instances are down & MySQL has too many connections
- **Java application:** Too many requests
- **K8s component:** StatefulSet replicas mismatch

**Solution:**

**Check Alert Rule Selector:**
```sh
# Verify correct label for alert rules (may change with Prometheus stack versions)
kubectl get prometheuses.monitoring.coreos.com
kubectl get prometheuses.monitoring.coreos.com {crd-name} -o yaml | grep ruleSelector -A 2
```

**Apply Alert Rules:**
```sh
# Execute following to add prometheus alert rules
kubectl apply -f kubernetes-manifests/3-nginx-alert-rules.yaml
kubectl apply -f kubernetes-manifests/3-mysql-alert-rules.yaml
kubectl apply -f kubernetes-manifests/3-java-alert-rules.yaml
kubectl apply -f kubernetes-manifests/3-k8s-alert-rules.yaml
```

**Note:** We use `"release: monitoring-stack"` label to add alert rules. This label may change with newer Prometheus stack versions.

---

## Exercise 4: Send Alert Notifications

**Task:**
Configure AlertManager to send notifications:
- **Developer team:** Java or MySQL application issues → Slack channel
- **K8s administrators:** Nginx Ingress Controller or K8s component issues → Email address

**Solution:**

**Setup Slack Integration:**
```sh
# Use the following guide to set up your Slack channel:
# https://www.freecodecamp.org/news/what-are-github-actions-and-how-can-you-automate-tests-and-slack-notifications/
```

**Setup Email Configuration:**
```sh
# Configure your email account as shown in monitoring module video:
# "10 - Configure Alertmanager with Email Receiver"
```

**Apply Notification Configuration:**
```sh
# Execute following to configure alert manager notifications
kubectl apply -f kubernetes-manifests/4-email-secret.yaml
kubectl apply -f kubernetes-manifests/4-slack-secret.yaml
kubectl apply -f kubernetes-manifests/4-alert-manager-configuration.yaml
```

**Note:** In your case, this can be your own email address or your own Slack channel for testing purposes.

---

## Exercise 5: Test the Alerts

**Task:**
Simulate issues and trigger at least one alert for each notification channel (Slack and Email) to verify the entire monitoring setup works correctly.

**Solution:**

**Test StatefulSet Alert (Email notification):**
```sh
# Delete one of the MySQL StatefulSet pods
kubectl get statefulsets
kubectl delete pod mysql-0  # or any MySQL pod name

# This should trigger the "StatefulSet replicas mismatch" alert
# and send email notification to K8s administrators
```

**Test MySQL Connection Alert (Slack notification):**
```sh
# Scale down MySQL deployment to trigger "MySQL instances down" alert
kubectl scale statefulset mysql --replicas=0

# This should trigger MySQL-related alerts
# and send Slack notification to developer team
```

**Test Nginx 4xx Error Alert (Email notification):**
```sh
# Access non-existent path to generate 4xx errors
curl http://your-java-app-domain/path-that-doesnt-exist
curl http://your-java-app-domain/another-invalid-path
# Repeat multiple times to exceed 5% threshold

# This should trigger nginx 4xx error rate alert
# and send email notification to K8s administrators
```

**Test Java Application Alert (Slack notification):**
```sh
# Generate high load on Java application
# Use load testing tools like curl in a loop or wrk
for i in {1..100}; do
  curl http://your-java-app-domain/ &
done

# This should trigger "Too many requests" alert
# and send Slack notification to developer team
```

**Restore Services After Testing:**
```sh
# Scale MySQL back up
kubectl scale statefulset mysql --replicas=2

# Verify all services are running
kubectl get pods
kubectl get statefulsets
```

---

## Solutions in Code Structure

```
solutions-in-code/
├── 1-configure-k8s.yaml
├── 2-configure-k8s.yaml
├── kubernetes-manifests/
│   ├── java-app.yaml
│   ├── mysql-config.yaml
│   ├── ingress-config.yaml
│   ├── 3-nginx-alert-rules.yaml
│   ├── 3-mysql-alert-rules.yaml
│   ├── 3-java-alert-rules.yaml
│   ├── 3-k8s-alert-rules.yaml
│   ├── 4-email-secret.yaml
│   ├── 4-slack-secret.yaml
│   └── 4-alert-manager-configuration.yaml
```

## Key Monitoring Components

### **Prometheus Stack Components:**
- **Prometheus:** Metrics collection and storage
- **AlertManager:** Alert routing and notifications
- **Grafana:** Visualization and dashboards
- **ServiceMonitors:** Metrics scraping configuration

### **Application Metrics Sources:**
- **Java Application:** Custom metrics on port 8081
- **MySQL Database:** Database performance metrics
- **Nginx Ingress:** HTTP request metrics and status codes
- **Kubernetes:** Cluster and pod health metrics

### **Alert Categories:**
- **Application Performance:** Request rates, response times
- **Infrastructure Health:** Pod status, resource utilization  
- **Database Connectivity:** Connection pools, availability
- **Network Issues:** HTTP error rates, ingress problems

## Useful References

- **Prometheus Monitoring API for K8s:** [OpenShift Monitoring APIs](https://docs.openshift.com/container-platform/latest/rest_api/monitoring_apis/monitoring-apis-index.html)
- **Helm Charts:**
  - [MySQL Bitnami Chart](https://github.com/bitnami/charts/tree/master/bitnami/mysql)
  - [Nginx Ingress Chart](https://github.com/kubernetes/ingress-nginx/tree/master/charts/ingress-nginx)
  - [Kube Prometheus Stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

## Learning Outcomes

- **Observability Implementation:** Set up comprehensive monitoring for microservices
- **Proactive Alerting:** Configure intelligent alerts for critical system issues
- **Multi-Channel Notifications:** Route different alert types to appropriate teams
- **Testing & Validation:** Verify monitoring setup with controlled failure scenarios
- **Prometheus Ecosystem:** Hands-on experience with Prometheus, AlertManager, and ServiceMonitors
- **Kubernetes Monitoring:** Deep understanding of cloud-native observability patterns
