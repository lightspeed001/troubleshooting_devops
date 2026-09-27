# troubleshooting_devops
## Troubleshooting K8s

### 1. Cluster Level Checks

> **Cluster Health**

- Run `kubectl get nodes` to verify all nodes are `Ready`
- Check for taints or unschedulable nodes: `kubectl describe nodes | grep -i taint`
- Verify cluster capacity: `kubectl top nodes` (check CPU/memory pressure)

> **API Server & Etcd**

- Check API server logs (if you have access)

```bash
kubectl logs -n kube-system kube-spiserver-<node-name>

```
- Verify Etcd health (if managing Etcd directly)

```bash
ETCDCTL_API=3 etcdctl endpoint health --endpoints=<etcd-endpoint>

```
> **Networking**

- Test inter-pod connectivity:

```bash
kubectl run -it --rm --image=alpine/network -- sh

# inside the pod

ping <service-name>.<namespace>.svc.cluster.local

```

- Check CNI plugin logs (eg. Calico, Cilium, Flannel)

```bash
kubectl logs -n kube-system <cni-pod-name>

```

### 2. Namespace & Resource Checks

> **Namespace Existence**

- Ensure the namespace exists: `kubectl get ns <you-namespace>`
- If missing, recreate it: `kubectl create ns <your-namespace>`

> **Resource Limits**

- Check for resource constaints:

```bash
kubectl describe nodes | grep -A 5 "Allocated resources"

```

- Verify pod resource requests/limits int the deployment:

```bash
kubectl get deploy <deployment-name> -o yaml |grep -A 10 "resources"

```

> **Quotas**

- Check if ResourceQuotas are blocking deployment:

```bash
kubectl get quota -n <your-namespace>

```

### 3. Deployment & Pod-Specific Checks

> **Deployment Status**

- Check deployment status:

```bash
kubectl get deploy <deployment-name> -o wide

```

- Verify rollout status:

```bash
kubectl rollout status deploy/<deployment-status>

```

> **Pod Status**

- List pods and their statuses:

```bash
kubectl get pods -n <your-namespace> -o wide

```

- Check pod events:

```bash
kubectl describe pod <pod-name> -n <your-namespace>

```

- Look for `CrashLoopBackOff`, `Pending` or `ImagePullBackOff`

> **Pod Logs**

- Featch logs for the problematic pod:

```bash
kubectl logs <pod-name> -n <your-namespace> --previous  # for crashed pods

```

- Stream logs in real time:

```bash
kubectl logs -f <pod-name> -n <your-namespace>

```

> **Init Containers**

- Check if init containers are failing:

```bash
kubectl describe pod <pod-name> | grep -A 20 "Init Containers"

```

### 4. Configuration Checks

> **ConfigMaps & Secrets**

- Verify ConfigMaps/Secrets exist:

```bash
kubectl get configmap <config-name> -n <your-namespace>
kubectl get secret <secret-name> -n <your-namespace>

```
- Check if they're correctly mounted in the pod:

```bash
kubectl described pod <pod-name> | grep -A 5 "Volumes"

```

> **Environment Variables**

- Verify env vars are correctly set:

```bash
kubectl get deploy <deployment-name> -o yaml | grep -A 20 "env:"

```

> **Image Pull Issues**

- Check if the image exists and is accessible:

```bash
kubectl describe pod <pod-name> | grep -i "image pull"

```

- Verify image pull secrets:

```bash
kubectl get secret <secret-name> -n <your-namespace> -o yaml

```

### 5. Networking & Service Checks

> **Service Endpoints**

- Verifying the service has endpoints:

```bash
kubectl get endpoints <service-name> -n <your-namespace>

```

- Check if pods are correctly labeled to match the service selector:

```bash
kubectl get pods -n <your-namespace> --show-labels

```

> **Ingress & Routes**

- Check Ingress rules:

```bash
kubectl get ingress -n <your-namespace>
kubectl describe ingress <ingress-name> -n <your-namespace>

```

- Verify DNS resolution for the ingress host.

> **Network Policies**

- Check if Network Policies are blocking traffic:

```bash
kubectl get networkpolicy -n <your-namespace>

```

### 6. Storage Checks

> **Persistent VolumeClaims** (PVCs)

- Verify PVCs are bound:

```bash
kubectl get pvc -n <your-namespace>

```

- Check for errors in PVC events:

```bash
kubectl describe pvc <pvc-name> -n <your-namespace>

```

> **StorageClass**

- Ensure the StorageClass exists and is accessible:

```bash
kubectl get storageclass

```

### RBAC & Permissions

> **ServiceAccount Permissions**

- Check if the pod's ServiceAccount has the required permissions:

```bash
kubectl describe sa <service-account-name> -n <your-namespace>

```

- Verify RoleBindings/ClusterRoleBindings:

```bash
kubectl get rolebindings,clusterrolebindings -n <your-namespace>

```

> **Pod Security Policies** (PSP)/QPA Gatekeeper

- Check if PSPs or policies are blocking are the pod:

```bash
kubectl get psp # If PSPs are enabled
kubect; get k8spsp... # If using DFA Gatekeeper

```

### 8. Custom Resource Definitions (CRDs)

> **CRD Status**

- If using CRDs (eg. Istio, Prometheus Operator), check their status

```bash
kubectl get <crd-name> -n <your-namespace>
kubectl describe <crd-name> <instance-name> -n <your-namespace>

```

### 9. Control Plane & Operator Issues

> **Operator Logs**

- If using an operator (eg. Helm Operator, ArgoCD), check its logs:

```bash
kubectl logs -n <operator-namespace> <operator-pod-name>

```

> **Kube-Controller-Manager**

- Check controller logs (if accessible):

```bash
kubectl logs -n kube-system kube-controller-manager-<node-name>

```

### 10. Advanced Debugging

> **Exec into Pod**

- If the pod is running exec into it:

```bash
kubectl exec -it <pod-name> -n <yout-namespace> -- sh

```
- Check filesystem, network, or application-specific issues.

> **Debug Sidecar Containers**

- If using (eg. Istio, Linkerd), check their logs:

```bash
kubectl logs <pod-name> -c <side-container-name> -n <your-namespace>

```

> **Ephemeral Debug Containers**

- Use `ephemeral debug containers` (kubernetes > 1.25) to inspect a failing pod:

```bash
kubectl debug -it <pod-name> --image=busybox --target=<container-name> -n <your-namespace>

```

### 11. External Dependencies

> **Database/External Services**

- Verify connectivity to external dependencies (eg. databases, APIs):

```bash
kubectl run -t --rm --image=alpine -- sh
# Inside the pod:
nc -zv <external-service> <port>

```

> **Cloud Provider Issues**

- Check cloud provider logs (eg. AWS EKS, GKE, AKS) for node or load balancer issues.

### 12. Common Pitfalls

> **Time Synchronization**: Ensure all nodes have synchronized time (NTP).
> **DNS Resolution**: Check CoreDNS logs:

```bash
kubectl logs -n kube-system <coredns-pod-name>

```

> **Node Affinity/Anti-Affinity**: Verify node selectors or taints/tolerations:

```bash
kubectl get nodes --show-labels
kubectl describe node <node-name> | grep -i taint

```

### Final steps :spiral_notepad:

> **Reproduce the Issue**: If possible, recreste the isssue in a staging environment.

> **Check Events**: Look for warining/errors in cluster-wide events:

```bash
kubectl get events -A --sort-by='.metadata.creationTimestamp'

```

> **Compare with Working Deployments**: If you have a working deployment, diff its configuration:

```bash
kubectl get deploy <working-deploy> -n yaml > working.yaml
kubectl get deploy <failing-deploy> -o yaml > failing.yaml
diff wrking.yaml failing.yaml

```

### Tools to Help

> `kubectl debug` : for inspecting failing pods.
> `stern`: Multi-pod log tailing:

```bash
stern <pod-pattern> -n <your-namespace>

```
- `k8s`: Terminal-based UI for Kubernetes.
- `kubectl-neat`: Clean up `kubectl` output for readibility:

```bash
kubectl get deploy <deployment-name> -o yaml | kubectl neat

```

