# Kubernetes Error Solutions

A quick-reference troubleshooting guide for common Kubernetes and Docker errors — diagnostic commands, root causes, and fixes.

## Table of Contents

- [CrashLoopBackOff](#crashloopbackoff)
- [ImagePullBackOff](#imagepullbackoff)
- [OOMKilled](#oomkilled)
- [Network Errors](#network-errors)
- [Advanced Troubleshooting Tools](#advanced-troubleshooting-tools)
- [Quick Troubleshooting Checklist](#quick-troubleshooting-checklist)

---

## CrashLoopBackOff

A pod enters `CrashLoopBackOff` when its container keeps crashing and restarting.

### Step 1: Examine Container Logs

```bash
# Logs from the previous (failed) container instance
kubectl logs <pod-name> --previous

# Logs from all containers in the pod
kubectl logs <pod-name> --all-containers=true

# Follow logs in real time
kubectl logs -f <pod-name>
```

### Step 2: Check Pod Events

```bash
kubectl describe pod <pod-name> | grep -A 20 Events
```

### Common Causes

| Cause | Diagnostic Signal | Solution |
|---|---|---|
| Application crashing at startup | Logs show an exception | Fix the application code |
| Missing environment variables | Configuration error | Check ConfigMaps / Secrets |
| Dependency unavailable | Connection refused | Wait for dependency (use an `initContainer`) |
| Insufficient resources | `OOMKilled` in events | Increase resource limits |

### Example: `initContainer` to Wait for a Dependency

```yaml
initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z postgres-svc 5432; do sleep 2; done']
```

---

## ImagePullBackOff

`ImagePullBackOff` means Kubernetes can't download the specified image.

### Essential Checks

```bash
# Error details
kubectl describe pod <pod-name> | grep -A 5 "Warning"

# Manually test the pull
docker pull <image>:<tag>
```

### Causes and Resolutions

**1. Non-existent image or incorrect tag**

```bash
docker manifest inspect <image>:<tag>
```

**2. Authentication required for a private registry**

```bash
# Create a registry secret
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass
```

```yaml
# Reference it in the pod spec
spec:
  imagePullSecrets:
    - name: regcred
```

**3. Network problem reaching the registry**

```bash
kubectl run debug --image=busybox --rm -it -- wget -O- https://registry.example.com/v2/
```

> **Key takeaway:** Always use explicit tags (`v1.2.3`) instead of `latest` to ensure reproducibility.

---

## OOMKilled

`OOMKilled` (Out Of Memory Killed) means the container exceeded its memory limit.

### Diagnostic

```bash
# Identify OOMKilled pods
kubectl get pods --field-selector=status.phase=Failed

# Check consumed resources
kubectl top pod

# Examine configured limits
kubectl describe pod <pod-name> | grep -A 5 Limits
```

### Resource Configuration

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### Sizing Rules

- **Requests** — guaranteed resources, used for scheduling
- **Limits** — maximum ceiling before the container is killed
- **Recommended ratio** — limits = 2x requests for variable workloads

---

## Network Errors

Network issues account for a significant share of Kubernetes incidents.

### Pod Not Communicating with a Service

```bash
# Verify the Service exists and has endpoints
kubectl get endpoints <service-name>

# Test DNS resolution from a pod
kubectl run dns-test --image=busybox --rm -it -- nslookup <service-name>

# Verify pod labels match the Service selector
kubectl get pods --show-labels
kubectl describe svc <service-name> | grep Selector
```

### Ingress Not Routing Traffic

```bash
# Check Ingress configuration
kubectl describe ingress <ingress-name>

# Check Ingress controller logs
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller

# Test from outside the cluster
curl -H "Host: example.com" http://<external-ip>/
```

> **Key takeaway:** The NGINX Ingress Controller is being retired — plan your migration to the Gateway API.

---

## Advanced Troubleshooting Tools

### `kubectl debug`

```bash
# Attach an ephemeral debug container
kubectl debug -it <pod-name> --image=busybox --target=<container-name>

# Copy a pod with a debug image
kubectl debug <pod-name> --copy-to=debug-pod --image=ubuntu
```

### Cluster Event Analysis

```bash
# Recent events sorted by timestamp
kubectl get events --sort-by='.lastTimestamp'

# Filter by event type
kubectl get events --field-selector type=Warning
```

### Real-Time Metrics

```bash
# Pod CPU/memory usage
kubectl top pods --all-namespaces

# Node usage
kubectl top nodes
```

---

## Quick Troubleshooting Checklist

| # | Command | Purpose |
|---|---|---|
| 1 | `kubectl get pods -o wide` | Global pod status |
| 2 | `kubectl describe pod <pod-name>` | Events and configuration |
| 3 | `kubectl logs <pod-name> --previous` | Logs before the crash |
| 4 | `kubectl top pod` | Resource consumption |
| 5 | `kubectl get events` | Recent cluster events |
