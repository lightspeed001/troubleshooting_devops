# troubleshooting_devops
## Troubleshooting Prometheus, Grafana and Loki

Prometheus collects metrics from your Kubernetes luster. Use these steps to diagnose issues when metrics indicate a problem.

### 1. Prometheus: Metrics-Based Troubleshooting

**A. Check Prometheus Targets**

> Verify Prometheus is scraping targets:

```bash
# Access Prometheus UI (port-forward if needed)
kubectl port-forward svc/prometheus-operated -n monitoring 9090:9090

```

- Navigate to Status > Targets in the Prometheus UI.
- Look for targets in `UP` (healthy) or `DOWN` (unhealthy) state/
Common Issues

- Scrape errors: Check logs for `scrape_timeout` or `scrape_failure`
- Missing labels: Ensure `JOB` and `pod` labels are orretly set.

> Check scrape config

```bash
kubectl get prometheus -n monitoring -o yaml | grep -A 20 "scrape_configs"

```

**B. Key Metrics to Investigate**

> 1. Pod/Container Resource Usage

- CPU/Memory Pressure:

```promql
# High CPU usage (pod-level)
sum(rate(container_cpu_usage_seconds_total{namespace="<your-namespace>"}[5m])) by (pod) > 0.8

# High memory usage (pod-level)
sum(container_memory_working_set_bytes{namespace="<your-namespace>"}) by (pod) / sum(container_spec_memory_limit_bytes{namespace="<your-namespace>"}) by (pod) > 0.9

```
- Action: Scale up resources, optmisze application, or kill misbehaving pods.

> 2. Pod Restarts & Crashes

- CrashLoopBackOff:

```promql
increase(kube_pod_container_status_restarts_total{namespace="<your-namespace>"}[1h]) > 0

```
- Action: Check pod logs (`kubectl logs <pod-name> --previous`).

> 3. Network Latency & Errors

- High Latency (HTTP/TCP):

```promql
# HTTP request latency (service-level)
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{namespace="<your-namespace>"}[5m])) by (le, service))

# TCP connection errors
sum(rate(tcp_connections_total{namespace="<your-namespace>", state="error"}[5m])) by (pod)

```
- Action: Check service mesh (Istio/Linkerd) or CNI logs.

> 4. Disk I/O Bottlenecks

- High Disk Latency:

```promql
# Pod-level disk latency
rate(container_fs_reads_total{namespace="<your-namespace>"}[5m]) > 100

```
- Action: Check PVC performance or node disk health

> 5. API Server Latency

- Slow Kubernetes API Calls

```promql
# API server latency (99th percentile)
histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket{verb!="WATCH"}[5m])) by (le, resource))

```
- Action: Check `kube-apiserver` logs for throttling or slow queries.

**C. Prometheus Alerts**

- Check if prometheus has triggered alerts:

```promql
kubectl get prometheus -n monitoring

```
Common alerts to investigate:

- `KubePodCrashLooping`
- `KubeContaierOOMKilled`
- `KubeNodeNotReady`
- `HighCPUUsage`

---

### 2. Grafana: Visualizing Metrics

Grafana provides dashbords to visualize Prometheus metrics. Use these steps to correlate issues.

**A. Access Grafana**

- Port-forward Grafana:

```bash
kubectl port-forward svc/grafana -n monitory 3000:3000

```
- Log in (default credentials: `admin/admin` or check your Helm values).

**B. Key Dashboards to Check**

- Kubernetes/Compute Resources/Namespace (Pods) - CPU/Memory usage per pod.
- Kubernetes/Networking/Namespace- Networking traffic, errors and latency
- Kubernetes/Persistent Volumes = Disk I/O and storage erformance.
- Kubernetes/API Server: API server latency and error rates.
- Custom Application Dashboards: Metrics specific to your app (eg. HTTP requests, database queries).

**C. Correlate Metrics with Issues**

> Example: If a pod is crashing (`CrashLoopBackOff`), check:

- CPU/Memory: Is the pod OOMKilled or CPU throttled?
- Network: Are there connection errors to dependencies?
- Disk: Is the PVC full or slow?
- Logs: Check Loki for application errors.

---

### 3. Loki: Log-Based Troubleshooting

Loki aggregates logs from from your Kubernetes cluster. Use these steps when reveal the issue.

**A. Access Loki**

> Port-forward Loki:

```bash
kubectl port-forward svc/loki -n 3100:3100

```
> Use Grafana Explore (select Loki as the data source) or `logcli`

```bash
logcli query '{namespace="<your-namespace>", pod=~"<pod-name>.*"}' --addr=http://loki:3100

```

**B. Key Log Queries**

- Application Errors: `{namespace="<your-namespace>"} |~ "error|exception|failed"`
- Pod Crash Reasons: `{namespace="<your-namespace>"} |~ "OOMKilled|SIGKILL|CrashLoopBackOff"`
- HTTP 5xx Errors:`{namespace="<your-namespace>"} |~ "HTTP/1.1\" 5[0-9]{2}"`
- Slow Requests: `{namespace="<your-namespace>"} | duration > "1s"`
- Database Connection Issues: `{namespace="<your-namespace>"} |~ "connection refused|timeout"`


**C. Correlate Logs with Metrics**

> Examples: If Prometheus shows high katency.

- Query Loki for slow requests:

```loki
{namespace="<your-namespace>"} | duration > "1s"

```
- Check if these ogs correlate with CPU/memory usage.

---

### 4. End-to-End Observability Workflow

When a problem is immediately obvious (eg. high latency, errors or crashes ), follow this workflow:

**Step 1: Identify the Symptom**

- Prometheus: High CPU, ,emory, latency or errors.
- Grafana: Visual anomalies in dashboards.
- Loki: Error logs or slow requests.

**Step 3: Correlate Data**

- Prometheus: High resource usage, error rates, latency spikes.
- Grafana: Visual confirmation of anomalies in dashborads.
- Loki: Error messages, stack traces, or slow requests.
- Kubernetes: Pod status, events, and logs (`kubectl describe`, `kubectl logs`)

**Step 4: Take Action**

> Scale Up: If CPU/Memory is high, increase resources or optimize the app.

> Restart Pods: If a pod is stuck in `CrashLoopBackOff`, delete it to forcea restart.

> Check dependencies: If latency is high, verify database/API health.

> Rollback: If a recent change caused the issue, roll back the deployment.

**Example Scenario: High Latency**

> 1. Prometheus

- Query: `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service=""}[5m])) by (le))`
- Result: 95th percentile latency is 2s (normal is 200ms).

> 2. Grafana

- Dashboard shows spikes in latency for `<your-service>`

> 3. Loki

- Query: `{service="<your-service>"} | duration > "1s"`
- Result: Logs show slow database queries

> 4. Action

- Optimize database queries or scale the database.

---

### 5. Common Observability Pitfalls

> 1. Missing Metrics

- Ensure Prometheus is scraping your app (check `scrape_configs`)
- Add custom metrics if needed (eg. `prometheus-adapter`)

> 2. Log Sampling

- Loki may sample logs under high load. Adjust `limits_config` Loki values

> 3. Metric Cardinality Explosion

- Too many labels (eg. `pod=.*`) can overwhelm Prometheus. Use `keep_common` or `drop` in relabeling.

> 4. Grafana Dashboard Lag

- Refresh dashboards or check for slow queries in Grafana logs.

---

### 6. Tools to Enhance Observability

> kube-prometheus-stack: Per-configured Prometheus, Grafana and Alertmanager for Kubernetes.

> Tempo: Distributed tracing (integrate with Prometheus/Grafana)

> Grafana OnCall: Alerting and incident Prometheus alternative.

> VictoriaMetrics: High-performance Prometheus alternative.

> Mimir: Long-term metrics storage for Prometheus.

---

### Final Checklist :spiral_notepad:

> 1. Prometheus

- Check target health.
- Query key metrics (CPU, memory, latency, errors)
- Review alerts

> 2. Grafana

- Inspect relevant dashboards
- Correlate metrics with visual anomalies.

> 3. Loki

- Query logs for errors or slow requests.
- Correlate logs with metrics.

> 4. Kubernetes

- Check pod status, events and logs.

> 5. Action

- Scale, restart or optimize base on findings.


