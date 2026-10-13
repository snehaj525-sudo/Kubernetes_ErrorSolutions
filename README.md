# Kubernetes_ErrorSolutions

                                            **Docker and Kubernetes Troubleshooting: Resolving CrashLoopBackOff**

    **The CrashLoopBackOff error indicates a pod continuously restarting after failures.**

Step 1: Examine container logs
# Failing pod logs
kubectl logs <pod-name> --previous
# Logs from all containers in pod
kubectl logs <pod-name> --all-containers=true
# Follow logs in real-time
kubectl logs -f <pod-name>

Step 2: Check events
kubectl describe pod <pod-name> | grep -A 20 Events

           Cause	                                                      Diagnostic	                                     Solution
Application crashing at startup	                                   Logs show exception	                            Fix application code
Missing environment variables	                                     Configuration error	                           Check ConfigMaps/Secrets
Dependency unavailable	                                           Connection refused	                        Wait for dependency (initContainer)
Insufficient resources	                                           OOMKilled in events	                                 Increase limits

# Example: initContainer to wait for dependency
initContainers:
- name: wait-for-db
image: busybox:1.36
command: ['sh', '-c', 'until nc -z postgres-svc 5432; do sleep 2; done']

**How to Diagnose ImagePullBackOff in Kubernetes**
ImagePullBackOff means Kubernetes cannot download the specified Docker image.

Essential checks
# Error details
kubectl describe pod <pod-name> | grep -A 5 "Warning"

# Manually test pull
docker pull <image-name>:<tag>
Causes and resolutions
1. Non-existent image or incorrect tag

# Verify image existence
docker manifest inspect <image>:<tag>
2. Authentication required for private registry

# Create registry secret
kubectl create secret docker-registry regcred \
--docker-server=registry.example.com \
--docker-username=user \
--docker-password=pass

# Reference in pod
spec:
imagePullSecrets:
- name: regcred
3. Network problem to registry

# Test connectivity from debug pod
kubectl run debug --image=busybox --rm -it -- wget -O- https://registry.example.com/v2/
Key takeaway: Always use explicit tags (v1.2.3) rather than latest to ensure reproducibility.
See Kubernetes vs Docker Swarm, ECS and Nomad comparison to understand how each orchestrator handles images.

    **Resolving Memory Problems: OOMKilled**
    
OOMKilled (Out Of Memory Killed) indicates the container exceeded its memory limit.

Diagnostic
# Identify OOMKilled pods
kubectl get pods --field-selector=status.phase=Failed

# Check consumed resources
kubectl top pod <pod-name>

# Examine configured limits
kubectl describe pod <pod-name> | grep -A 5 Limits
Resource configuration
resources:
requests:
memory: "256Mi"
cpu: "250m"
limits:
memory: "512Mi"
cpu: "500m"
Sizing rules:

Requests: guaranteed resources, used for scheduling
Limits: maximum ceiling before kill
Recommended ratio: limits = 2x requests for variable applications
Docker and Kubernetes Troubleshooting: Network Errors
Network problems represent 30% of incidents in Kubernetes environments.

Pod not communicating with Service
# Verify Service exists and has endpoints
kubectl get endpoints <service-name>

# Test DNS resolution from a pod
kubectl run dns-test --image=busybox --rm -it -- nslookup <service-name>

# Verify pod labels match Service selector
kubectl get pods --show-labels
kubectl describe svc <service-name> | grep Selector
Ingress not routing traffic
# Check Ingress configuration
kubectl describe ingress <ingress-name>

# Ingress controller logs
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller

# Test from outside
curl -H "Host: example.com" http://<ingress-ip>/
Key takeaway: The NGINX Ingress Controller will be retired in March 2026 according to InfoQ. Plan your migration to Gateway API.
For quick answers, see the Docker and Kubernetes FAQ.

    **Essential Tools for Advanced Troubleshooting**
    
kubectl debug

# Create ephemeral container for debug
kubectl debug <pod-name> -it --image=busybox --target=<container-name>

# Copy pod with debug image
kubectl debug <pod-name> --copy-to=debug-pod --image=ubuntu
Cluster event analysis
# Recent events sorted by date
kubectl get events --sort-by='.lastTimestamp'

# Filter by type
kubectl get events --field-selector type=Warning
Real-time metrics
# Pod CPU/memory usage
kubectl top pods --all-namespaces

# Node usage
kubectl top nodes

    **Quick Troubleshooting Checklist**

1	kubectl get pods -o wide	   -Global pod status
2	kubectl describe pod	       -Events and configuration
3	kubectl logs --previous	     -Logs before crash
4	kubectl top pod	             -Resource consumption
5	kubectl get events	         -Recent cluster events
