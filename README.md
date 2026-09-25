# k8s-troubleshooting
# 🚀 Kubernetes Troubleshooting Guide

A practical, hands-on **Kubernetes troubleshooting playbook** for DevOps, Cloud Engineers, SREs, and Kubernetes administrators.

This repository contains troubleshooting commands, diagnostic workflows, common failures, production-style scenarios, and step-by-step solutions for Kubernetes workloads.

---

## 📌 What You'll Learn

* Kubernetes Pod troubleshooting
* Deployment & ReplicaSet failures
* Service & Endpoint troubleshooting
* Kubernetes DNS troubleshooting
* ConfigMap & Secret issues
* PersistentVolume / PVC troubleshooting
* Node failures
* Scheduling problems
* Resource & OOM issues
* ImagePullBackOff / ErrImagePull
* CrashLoopBackOff
* HPA & Metrics Server troubleshooting
* Ingress troubleshooting
* NetworkPolicy troubleshooting
* RBAC troubleshooting
* StatefulSet troubleshooting
* DaemonSet troubleshooting
* Job & CronJob troubleshooting
* Kubernetes API troubleshooting
* Container runtime issues
* Helm troubleshooting
* Argo CD troubleshooting
* Istio / Service Mesh troubleshooting
* Prometheus / Grafana troubleshooting
* Production incident debugging

---

# 🏗️ Kubernetes Troubleshooting Workflow

When a Kubernetes application is not working, follow this order:

```text
                    ┌─────────────────────┐
                    │   User/Application  │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │       Ingress       │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │       Service       │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │      Endpoints      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │        Pods         │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │    Containers       │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ DB / External APIs  │
                    └─────────────────────┘
```

### Basic investigation

```bash
kubectl get pods -A
kubectl get nodes
kubectl get svc -A
kubectl get endpoints -A
kubectl get events -A --sort-by=.lastTimestamp
```

Then investigate:

```bash
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous
```

---

# 🔎 1. Pod Troubleshooting

## Check Pod Status

```bash
kubectl get pods -n <namespace>
```

Example:

```text
NAME                       READY   STATUS             RESTARTS
frontend-7c8d9f6d5f-x2abc  0/1     CrashLoopBackOff   8
```

Check details:

```bash
kubectl describe pod frontend-7c8d9f6d5f-x2abc -n <namespace>
```

Check logs:

```bash
kubectl logs frontend-7c8d9f6d5f-x2abc -n <namespace>
```

Previous container logs:

```bash
kubectl logs frontend-7c8d9f6d5f-x2abc \
  -n <namespace> \
  --previous
```

---

# 💥 2. CrashLoopBackOff

### Symptoms

```text
CrashLoopBackOff
```

### Troubleshooting

```bash
kubectl describe pod <pod> -n <namespace>

kubectl logs <pod> -n <namespace>

kubectl logs <pod> -n <namespace> --previous
```

Check container state:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*].state}'
```

Check restart count:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o custom-columns=NAME:.metadata.name,RESTARTS:.status.containerStatuses[*].restartCount
```

### Common causes

* Application startup failure
* Incorrect environment variables
* Missing Secret
* Missing ConfigMap
* Database unavailable
* Incorrect command/arguments
* Failed health probe
* Permission problems
* OOMKilled

---

# 🐳 3. ImagePullBackOff / ErrImagePull

Check:

```bash
kubectl describe pod <pod> -n <namespace>
```

Look at:

```text
Events:
Failed to pull image
```

Verify image:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o jsonpath='{.spec.containers[*].image}'
```

Check Deployment:

```bash
kubectl get deployment <deployment> \
  -n <namespace> \
  -o yaml
```

### Common causes

```text
Wrong image name
Wrong image tag
Private registry authentication
Registry unavailable
Image does not exist
ImagePullSecret missing
```

Check secrets:

```bash
kubectl get secrets -n <namespace>
```

---

# 📦 4. Pod Pending

```bash
kubectl get pods -n <namespace>
```

If:

```text
STATUS: Pending
```

Run:

```bash
kubectl describe pod <pod> -n <namespace>
```

Check scheduling events:

```bash
kubectl get events \
  -n <namespace> \
  --sort-by=.lastTimestamp
```

Check nodes:

```bash
kubectl get nodes
```

Check resources:

```bash
kubectl describe node <node>
```

### Common causes

* Insufficient CPU
* Insufficient memory
* Node taints
* Node affinity
* Pod anti-affinity
* Node selector
* PVC unavailable
* ResourceQuota
* LimitRange

---

# 🧠 5. OOMKilled

Check:

```bash
kubectl describe pod <pod> -n <namespace>
```

Look for:

```text
Reason: OOMKilled
Exit Code: 137
```

Check resources:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o yaml
```

Check node memory:

```bash
kubectl top nodes
```

Check pod memory:

```bash
kubectl top pods -n <namespace>
```

Example resource configuration:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"

  limits:
    cpu: "500m"
    memory: "512Mi"
```

---

# 🌐 6. Service Troubleshooting

Check Service:

```bash
kubectl get svc -n <namespace>
```

```bash
kubectl describe svc <service> -n <namespace>
```

Check endpoints:

```bash
kubectl get endpoints <service> -n <namespace>
```

Modern Kubernetes:

```bash
kubectl get endpointslice \
  -n <namespace>
```

### Critical check

If you see:

```text
ENDPOINTS: <none>
```

the Service currently has no ready matching backend endpoints.

Check Service selector:

```bash
kubectl get svc <service> \
  -n <namespace> \
  -o yaml
```

Check Pod labels:

```bash
kubectl get pods \
  -n <namespace> \
  --show-labels
```

Example:

```yaml
selector:
  app: frontend
```

Pod:

```yaml
labels:
  app: frontend
```

The selectors must match.

---

# 🔥 7. Service → Pod Connectivity

Start a temporary debugging container:

```bash
kubectl run debug \
  -n <namespace> \
  --rm -it \
  --image=curlimages/curl \
  -- sh
```

Test Service:

```bash
curl http://frontend
```

Test port:

```bash
curl http://frontend:80
```

Test DNS:

```bash
nslookup frontend
```

Test Kubernetes DNS:

```bash
nslookup kubernetes.default
```

---

# 🧭 8. Kubernetes DNS Troubleshooting

Check CoreDNS:

```bash
kubectl get pods -n kube-system \
  -l k8s-app=kube-dns
```

Check service:

```bash
kubectl get svc -n kube-system kube-dns
```

Check CoreDNS logs:

```bash
kubectl logs \
  -n kube-system \
  -l k8s-app=kube-dns
```

Test DNS:

```bash
kubectl run dns-test \
  --rm -it \
  --image=busybox:1.36 \
  -- nslookup kubernetes.default
```

Test application Service:

```bash
kubectl run dns-test \
  --rm -it \
  --image=busybox:1.36 \
  -- nslookup frontend.default.svc.cluster.local
```

---

# 🔐 9. ConfigMap Troubleshooting

Check:

```bash
kubectl get configmap -n <namespace>
```

```bash
kubectl describe configmap <configmap> \
  -n <namespace>
```

Get YAML:

```bash
kubectl get configmap <configmap> \
  -n <namespace> \
  -o yaml
```

Check Pod environment:

```bash
kubectl exec -it <pod> \
  -n <namespace> \
  -- env
```

---

# 🔑 10. Secret Troubleshooting

```bash
kubectl get secrets -n <namespace>
```

Describe:

```bash
kubectl describe secret <secret> \
  -n <namespace>
```

Decode a value:

```bash
kubectl get secret <secret> \
  -n <namespace> \
  -o jsonpath='{.data.password}' | base64 -d
```

> Avoid exposing secrets in shell history, CI/CD logs, Git repositories, or public issue reports.

---

# 💾 11. PVC / Storage Troubleshooting

Check PVC:

```bash
kubectl get pvc -n <namespace>
```

Check details:

```bash
kubectl describe pvc <pvc> \
  -n <namespace>
```

Check PV:

```bash
kubectl get pv
```

Check StorageClasses:

```bash
kubectl get storageclass
```

Common states:

```text
Pending
Bound
Lost
```

Check CSI components:

```bash
kubectl get pods -A | grep -i csi
```

---

# 🗄️ 12. StatefulSet Troubleshooting

```bash
kubectl get statefulsets -n <namespace>
```

```bash
kubectl describe statefulset <name> \
  -n <namespace>
```

Check Pods:

```bash
kubectl get pods \
  -n <namespace> \
  -l app=<application>
```

Check PVCs:

```bash
kubectl get pvc -n <namespace>
```

For a database:

```bash
kubectl logs <database-pod> -n <namespace>
```

Check database connectivity from another Pod:

```bash
kubectl run mysql-client \
  -n <namespace> \
  --rm -it \
  --image=mysql:8 \
  -- mysql -h mysql -u root -p
```

---

# 🖥️ 13. Node Troubleshooting

Check nodes:

```bash
kubectl get nodes -o wide
```

Detailed information:

```bash
kubectl describe node <node>
```

Check resource usage:

```bash
kubectl top nodes
```

Check conditions:

```bash
kubectl get node <node> \
  -o jsonpath='{.status.conditions}'
```

Common conditions:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

---

# 🚧 14. Node Taints

Check:

```bash
kubectl describe node <node> | grep -i taint
```

Example:

```text
CriticalAddonsOnly=true:NoSchedule
```

A Pod without a matching toleration may remain:

```text
Pending
```

Check Pod events:

```bash
kubectl describe pod <pod> -n <namespace>
```

---

# ⚖️ 15. Requests and Limits

Check:

```bash
kubectl get pod <pod> \
  -n <namespace> \
  -o jsonpath='{.spec.containers[*].resources}'
```

Check actual usage:

```bash
kubectl top pods -n <namespace>
```

Compare:

```text
CPU request
CPU limit
Memory request
Memory limit
Actual usage
```

---

# ❤️ 16. Readiness / Liveness Probe

Check:

```bash
kubectl describe pod <pod> -n <namespace>
```

Example:

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080

livenessProbe:
  httpGet:
    path: /health
    port: 8080
```

Test manually:

```bash
kubectl exec -it <pod> \
  -n <namespace> \
  -- wget -qO- http://localhost:8080/health
```

A failed readiness probe can cause:

```text
Pod Running
Pod Ready = 0/1
Service ENDPOINTS = none
```

---

# 🔐 17. RBAC Troubleshooting

Check current identity:

```bash
kubectl auth whoami
```

Check permissions:

```bash
kubectl auth can-i get pods
```

Check namespace:

```bash
kubectl auth can-i get pods \
  -n roboshop
```

Check another user/service account:

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:roboshop:default \
  -n roboshop
```

Check RoleBindings:

```bash
kubectl get rolebinding -A
```

Check ClusterRoleBindings:

```bash
kubectl get clusterrolebinding
```

---

# 🛡️ 18. NetworkPolicy Troubleshooting

List policies:

```bash
kubectl get networkpolicy -A
```

Inspect:

```bash
kubectl describe networkpolicy <policy> \
  -n <namespace>
```

If traffic suddenly stops after applying a policy, check:

```text
Source Pod
↓
Destination Pod
↓
Destination Port
↓
Namespace Selector
↓
Pod Selector
↓
Ingress/Egress rules
```

Test connectivity:

```bash
kubectl run net-test \
  -n <namespace> \
  --rm -it \
  --image=curlimages/curl \
  -- sh
```

---

# 🌍 19. Ingress Troubleshooting

Check:

```bash
kubectl get ingress -A
```

```bash
kubectl describe ingress <ingress> \
  -n <namespace>
```

Check backend:

```bash
kubectl get svc -n <namespace>
```

Check endpoints:

```bash
kubectl get endpoints -n <namespace>
```

Check ingress controller:

```bash
kubectl get pods -A | grep -i ingress
```

Common issues:

```text
Wrong hostname
Wrong service name
Wrong service port
No endpoints
TLS certificate problem
IngressClass mismatch
DNS problem
Load balancer problem
```

---

# 📈 20. HPA Troubleshooting

Check:

```bash
kubectl get hpa -A
```

Detailed:

```bash
kubectl describe hpa <hpa> \
  -n <namespace>
```

Check Metrics API:

```bash
kubectl top pods
```

```bash
kubectl top nodes
```

Check:

```bash
kubectl get apiservice | grep metrics
```

Common issue:

```text
Metrics API unavailable
```

---

# 🧰 21. Events — One of the Most Useful Commands

```bash
kubectl get events -A \
  --sort-by=.lastTimestamp
```

Namespace:

```bash
kubectl get events \
  -n <namespace> \
  --sort-by=.lastTimestamp
```

Events frequently reveal:

```text
FailedScheduling
FailedMount
FailedAttachVolume
FailedPull
BackOff
Unhealthy
Killing
FailedCreate
```

---

# 🔍 22. Deployment Troubleshooting

```bash
kubectl get deployment -n <namespace>
```

```bash
kubectl describe deployment <deployment> \
  -n <namespace>
```

Check ReplicaSets:

```bash
kubectl get rs -n <namespace>
```

Check rollout:

```bash
kubectl rollout status deployment/<deployment> \
  -n <namespace>
```

History:

```bash
kubectl rollout history deployment/<deployment> \
  -n <namespace>
```

Rollback:

```bash
kubectl rollout undo deployment/<deployment> \
  -n <namespace>
```

---

# 🧪 23. Debugging Without Tools Inside the Application Container

Many production containers don't contain:

```text
curl
wget
nslookup
ping
bash
```

Use an ephemeral debugging container:

```bash
kubectl debug -it <pod> \
  -n <namespace> \
  --image=nicolaka/netshoot
```

Or create a temporary Pod:

```bash
kubectl run netshoot \
  -n <namespace> \
  --rm -it \
  --image=nicolaka/netshoot \
  -- bash
```

Useful commands:

```bash
curl
wget
dig
nslookup
nc
ping
tcpdump
traceroute
```

---

# 🧵 24. Container Runtime / Node Debugging

Check node:

```bash
kubectl describe node <node>
```

Check system Pods:

```bash
kubectl get pods -n kube-system
```

Check kubelet logs on the node:

```bash
journalctl -u kubelet
```

Containerd:

```bash
journalctl -u containerd
```

Container runtime:

```bash
crictl ps
```

```bash
crictl pods
```

---

# ☸️ 25. Kubernetes API Server Troubleshooting

Check cluster:

```bash
kubectl cluster-info
```

Check API endpoint:

```bash
kubectl config view
```

Check API resources:

```bash
kubectl api-resources
```

Check API versions:

```bash
kubectl api-versions
```

Check control-plane components where applicable:

```bash
kubectl get pods -n kube-system
```

---

# ⛵ 26. Helm Troubleshooting

List releases:

```bash
helm list -A
```

Get values:

```bash
helm get values <release> \
  -n <namespace>
```

Get manifest:

```bash
helm get manifest <release> \
  -n <namespace>
```

Check release status:

```bash
helm status <release> \
  -n <namespace>
```

Render templates:

```bash
helm template <release> ./chart \
  -f values.yaml
```

Debug installation:

```bash
helm install <release> ./chart \
  -n <namespace> \
  --create-namespace \
  --debug \
  --dry-run
```

---

# 🔄 27. Argo CD Troubleshooting

Check applications:

```bash
argocd app list
```

Application status:

```bash
argocd app get <application>
```

Sync:

```bash
argocd app sync <application>
```

Hard refresh:

```bash
argocd app get <application> \
  --hard-refresh
```

Kubernetes-side investigation:

```bash
kubectl get applications \
  -n argocd
```

Check Argo CD components:

```bash
kubectl get pods -n argocd
```

Logs:

```bash
kubectl logs \
  -n argocd \
  deploy/argocd-server
```

Repository problems:

```bash
kubectl logs \
  -n argocd \
  deploy/argocd-repo-server
```

---

# 🕸️ 28. Istio Troubleshooting

Check Istio installation:

```bash
istioctl version
```

Check proxy status:

```bash
istioctl proxy-status
```

Analyze configuration:

```bash
istioctl analyze -n <namespace>
```

Check proxy configuration:

```bash
istioctl proxy-config cluster <pod> \
  -n <namespace>
```

```bash
istioctl proxy-config listeners <pod> \
  -n <namespace>
```

```bash
istioctl proxy-config routes <pod> \
  -n <namespace>
```

Check Envoy logs:

```bash
kubectl logs <pod> \
  -c istio-proxy \
  -n <namespace>
```

---

# 🌐 29. Istio Ambient Mesh Troubleshooting

Check waypoints:

```bash
kubectl get waypoint -A
```

Check waypoint Pods:

```bash
kubectl get pods -A \
  -l gateway.networking.k8s.io/gateway-name
```

Check Istio resources:

```bash
istioctl analyze -A
```

Check policies:

```bash
kubectl get authorizationpolicy -A
```

Check policy status:

```bash
kubectl describe authorizationpolicy <policy> \
  -n <namespace>
```

For connectivity errors such as:

```text
upstream connect error
connection termination
reset before headers
```

investigate:

```text
Service
↓
Service endpoints
↓
Waypoint binding
↓
AuthorizationPolicy
↓
ServiceAccount
↓
ztunnel
↓
Waypoint
↓
Destination application
```

---

# 📊 30. Prometheus / Grafana Troubleshooting

Check Prometheus:

```bash
kubectl get pods -n monitoring
```

Check services:

```bash
kubectl get svc -n monitoring
```

Check Prometheus targets from its UI/API.

Check Grafana:

```bash
kubectl logs \
  -n monitoring \
  deploy/grafana
```

Typical investigation:

```text
Application
↓
Metrics endpoint
↓
ServiceMonitor / PodMonitor
↓
Prometheus target
↓
Prometheus metric
↓
Grafana datasource
↓
Grafana dashboard
```

---

# 🧯 31. Production Incident Checklist

When production is down:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get svc -A
kubectl get endpoints -A
kubectl get ingress -A
kubectl get events -A --sort-by=.lastTimestamp
```

Then check:

```bash
kubectl top nodes
kubectl top pods -A
```

Application:

```bash
kubectl logs <pod> -n <namespace> --previous
```

Networking:

```bash
kubectl run debug \
  -n <namespace> \
  --rm -it \
  --image=nicolaka/netshoot \
  -- bash
```

---

# 🧠 Troubleshooting Decision Tree

```text
Application not reachable
        │
        ▼
Is Pod Running?
   │             │
  NO            YES
   │             │
   ▼             ▼
Check Pod     Is Pod Ready?
status           │
                 ├── NO
                 │    │
                 │    ▼
                 │ Check probes/logs/events
                 │
                 └── YES
                       │
                       ▼
                  Service exists?
                       │
                       ├── NO → Create/fix Service
                       │
                       └── YES
                            │
                            ▼
                       Endpoints exist?
                            │
                            ├── NO
                            │    │
                            │    ▼
                            │ Check selectors
                            │
                            └── YES
                                 │
                                 ▼
                            DNS working?
                                 │
                                 ├── NO → Check CoreDNS
                                 │
                                 └── YES
                                      │
                                      ▼
                              NetworkPolicy?
                                      │
                                      ├── YES → Check rules
                                      │
                                      └── NO
                                           │
                                           ▼
                                    Check application
                                    / database / external
                                    dependencies
```

---

# 🧪 Real-World Labs

This repository will contain hands-on troubleshooting labs.

## Lab 01 — CrashLoopBackOff

```text
Application
    ↓
CrashLoopBackOff
    ↓
kubectl describe
    ↓
kubectl logs --previous
    ↓
Identify configuration problem
    ↓
Fix Deployment
    ↓
Verify rollout
```

## Lab 02 — Service ENDPOINTS `<none>`

```text
Pod Running
      ↓
Service
      ↓
ENDPOINTS <none>
      ↓
Compare selectors
      ↓
Fix labels
      ↓
Verify EndpointSlice
```

## Lab 03 — Pod Pending

Investigate:

```text
CPU
Memory
Taints
Tolerations
Affinity
PVC
ResourceQuota
```

## Lab 04 — DNS Failure

Investigate:

```text
Pod DNS configuration
CoreDNS
kube-dns Service
NetworkPolicy
Service FQDN
```

## Lab 05 — Database Connectivity

Investigate:

```text
Application
   ↓
Service
   ↓
Endpoints
   ↓
DNS
   ↓
Port
   ↓
NetworkPolicy
   ↓
Database
```

## Lab 06 — Ingress 502

Investigate:

```text
DNS
 ↓
Load Balancer
 ↓
Ingress Controller
 ↓
Ingress
 ↓
Service
 ↓
Endpoints
 ↓
Pod
```

## Lab 07 — HPA Not Scaling

Investigate:

```text
HPA
 ↓
Metrics API
 ↓
Metrics Server
 ↓
Pod resource requests
 ↓
CPU/Memory metrics
```

## Lab 08 — Istio Connection Reset

Investigate:

```text
Client
 ↓
ztunnel
 ↓
Waypoint
 ↓
AuthorizationPolicy
 ↓
Service
 ↓
Endpoint
 ↓
Application
```

---

# 🛠️ Essential Command Cheat Sheet

```bash
# Cluster
kubectl cluster-info
kubectl get nodes -o wide
kubectl get events -A --sort-by=.lastTimestamp

# Pods
kubectl get pods -A -o wide
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous

# Deployment
kubectl get deploy -A
kubectl rollout status deployment/<name> -n <ns>
kubectl rollout history deployment/<name> -n <ns>

# Service
kubectl get svc -A
kubectl describe svc <svc> -n <ns>
kubectl get endpoints -A
kubectl get endpointslice -A

# Networking
kubectl run debug --rm -it \
  --image=nicolaka/netshoot -- bash

# Resources
kubectl top nodes
kubectl top pods -A

# Storage
kubectl get pv
kubectl get pvc -A
kubectl get storageclass

# RBAC
kubectl auth can-i get pods
kubectl auth whoami

# Ingress
kubectl get ingress -A
kubectl describe ingress <name> -n <ns>

# HPA
kubectl get hpa -A
kubectl describe hpa <name> -n <ns>

# Helm
helm list -A
helm status <release> -n <ns>
helm get values <release> -n <ns>

# Istio
istioctl version
istioctl analyze -A
istioctl proxy-status
```

---

# 📁 Suggested Repository Structure

```text
kubernetes-troubleshooting/
│
├── README.md
│
├── 01-pods/
│   ├── crashloopbackoff.md
│   ├── imagepullbackoff.md
│   ├── pending.md
│   └── oomkilled.md
│
├── 02-deployments/
│   ├── rollout-failure.md
│   └── replica-failure.md
│
├── 03-services/
│   ├── service-no-endpoints.md
│   ├── selector-mismatch.md
│   └── connectivity.md
│
├── 04-dns/
│   ├── coredns.md
│   └── dns-debugging.md
│
├── 05-storage/
│   ├── pvc-pending.md
│   ├── pv-binding.md
│   └── storageclass.md
│
├── 06-networking/
│   ├── networkpolicy.md
│   ├── ingress.md
│   └── service-connectivity.md
│
├── 07-rbac/
│   ├── roles.md
│   └── serviceaccounts.md
│
├── 08-hpa/
│   └── metrics-server.md
│
├── 09-helm/
│   └── helm-troubleshooting.md
│
├── 10-argocd/
│   └── argocd-troubleshooting.md
│
├── 11-istio/
│   ├── istio-troubleshooting.md
│   └── ambient-mesh.md
│
├── 12-observability/
│   ├── prometheus.md
│   ├── grafana.md
│   └── logging.md
│
└── labs/
    ├── lab-01-crashloop/
    ├── lab-02-service/
    ├── lab-03-dns/
    ├── lab-04-storage/
    ├── lab-05-ingress/
    └── lab-06-istio/
```

---

# 🎯 Troubleshooting Methodology

Use the following approach during incidents:

### 1. Observe

```bash
kubectl get pods -A
kubectl get nodes
kubectl get svc -A
```

### 2. Describe

```bash
kubectl describe pod <pod> -n <namespace>
```

### 3. Read Logs

```bash
kubectl logs <pod> -n <namespace>
```

### 4. Check Events

```bash
kubectl get events -A \
  --sort-by=.lastTimestamp
```

### 5. Test Connectivity

```bash
curl
wget
nc
dig
nslookup
```

### 6. Validate Configuration

Check:

```text
Labels
Selectors
Ports
Environment Variables
ConfigMaps
Secrets
Resources
Probes
RBAC
NetworkPolicies
```

### 7. Fix

Make the smallest safe change.

### 8. Verify

```bash
kubectl get pods
kubectl get endpoints
kubectl rollout status deployment/<name>
```

### 9. Document

Record:

```text
Incident
Root Cause
Evidence
Fix
Verification
Preventive Action
```

---

# 📝 Incident Report Template

````markdown
# Incident: Application Unavailable

## Date
YYYY-MM-DD

## Impact
Describe affected application/users.

## Symptoms

- Pods restarting
- Service unavailable
- HTTP 503
- Database connection failure

## Investigation

### Step 1

Command:

```bash
kubectl get pods -n application
````

Result:

```text
...
```

### Step 2

Command:

```bash
kubectl describe pod ...
```

Result:

```text
...
```

## Root Cause

Describe the actual root cause.

## Resolution

Describe the change made.

## Verification

```bash
kubectl get pods
kubectl get endpoints
```

## Preventive Actions

* Improve monitoring
* Add alerts
* Improve health probes
* Add resource limits
* Add automated testing

````

---

# 🚀 Goal

The goal of this repository is to build a practical Kubernetes troubleshooting reference that can be used for:

- 👨‍💻 DevOps Engineers
- ☁️ Cloud Engineers
- 🧑‍💻 SRE Engineers
- 🛠️ Kubernetes Administrators
- 🎯 Production Support Engineers
- 📚 Kubernetes Interview Preparation
- 🧪 Hands-on Kubernetes Labs

---

## ⭐ Topics Covered

```text
Kubernetes
├── Pods
├── Deployments
├── ReplicaSets
├── StatefulSets
├── DaemonSets
├── Services
├── Endpoints
├── EndpointSlices
├── DNS
├── ConfigMaps
├── Secrets
├── Storage
├── Nodes
├── Scheduling
├── Resources
├── Probes
├── RBAC
├── NetworkPolicy
├── Ingress
├── HPA
├── Helm
├── Argo CD
├── Istio
├── Prometheus
├── Grafana
└── Production Troubleshooting
````

---

## 📌 Contribution

Contributions are welcome.

If you discover a new Kubernetes failure scenario, add:

1. Problem
2. Symptoms
3. Investigation commands
4. Root cause
5. Resolution
6. Verification
7. Preventive action

---

## 👨‍💻 Author

**Venkata Ram Vanimina**

Cloud & DevOps Engineer | Kubernetes | AWS | Azure | Terraform | Docker | Helm | Istio | Argo CD

---

⭐ If this repository helps you troubleshoot Kubernetes issues, consider starring the repository.
