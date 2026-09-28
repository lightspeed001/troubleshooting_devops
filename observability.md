# troubleshooting_devops
## Troubleshooting Prometheus, Grafana and Loki

Prometheus collets metrics from your Kubernetes luster. Use these steps to diagnose issues when metrics indicate a problem.

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

