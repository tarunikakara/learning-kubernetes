# Kubernetes Logging, Monitoring & Troubleshooting

## Table of Contents
1. [Overview](#overview)
2. [kubectl Commands](#kubectl-commands)
3. [Events & Pod Logs](#events--pod-logs)
4. [Health Checks (Probes)](#health-checks-probes)
5. [Metrics-Server](#metrics-server)
6. [Prometheus & Grafana Basics](#prometheus--grafana-basics)
7. [Troubleshooting Guide](#troubleshooting-guide)
8. [Best Practices](#best-practices)

---

## Overview

**Observability** in Kubernetes consists of three pillars:

```
┌─────────────────────────────────────┐
│     Observability in Kubernetes     │
├─────────────────────────────────────┤
│                                     │
│  1. Logs      → What happened?      │
│  2. Metrics   → How is it doing?    │
│  3. Traces    → Why did it happen?  │
│                                     │
└─────────────────────────────────────┘
```

### Monitoring Stack Layers

```
┌──────────────────────────────────────┐
│    Application Monitoring            │
│  (Prometheus, Grafana Dashboards)   │
└─────────────────┬────────────────────┘
                  │
┌─────────────────▼────────────────────┐
│    Kubernetes Monitoring             │
│  (Metrics-Server, kubelet metrics)  │
└─────────────────┬────────────────────┘
                  │
┌─────────────────▼────────────────────┐
│    Container Runtime Monitoring      │
│  (Docker/containerd metrics)         │
└──────────────────────────────────────┘
```

---

## kubectl Commands

### Pod Information Commands

#### Get Pods

```bash
# Get pods in default namespace
kubectl get pods

# Get pods in all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A

# Get pods with more details
kubectl get pods -o wide

# Watch pods in real-time
kubectl get pods --watch
kubectl get pods -w

# Get pods with labels
kubectl get pods --show-labels

# Filter by label
kubectl get pods -l app=myapp

# Get pods by node
kubectl get pods --field-selector spec.nodeName=node-1
```

#### Describe Pod

```bash
# Full pod details
kubectl describe pod my-pod

# Describe pod in specific namespace
kubectl describe pod my-pod -n my-namespace

# Get YAML definition
kubectl get pod my-pod -o yaml

# Get JSON definition
kubectl get pod my-pod -o json

# Get specific field
kubectl get pod my-pod -o jsonpath='{.status.phase}'
```

#### Format and Output

```bash
# Table format (default)
kubectl get pods -o table

# Wide format (adds more columns)
kubectl get pods -o wide

# YAML format
kubectl get pods -o yaml

# JSON format
kubectl get pods -o json

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# JSON path query
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

### Pod Execution and Access

#### Logs

```bash
# Get pod logs
kubectl logs my-pod

# Follow logs (like tail -f)
kubectl logs my-pod -f

# Get logs from all containers in pod
kubectl logs my-pod --all-containers=true

# Get logs from specific container
kubectl logs my-pod -c container-name

# Get logs with timestamps
kubectl logs my-pod --timestamps=true

# Get logs from previous container (container crashed)
kubectl logs my-pod --previous

# Get last 100 lines
kubectl logs my-pod --tail=100

# Get logs since 2 minutes ago
kubectl logs my-pod --since=2m

# Multi-pod logs
kubectl logs -l app=myapp
```

#### Exec/Shell Access

```bash
# Execute command in pod
kubectl exec my-pod -- ls -la

# Interactive shell in pod
kubectl exec -it my-pod -- /bin/bash
kubectl exec -it my-pod -- /bin/sh

# Exec in specific container
kubectl exec -it my-pod -c container-name -- /bin/bash

# Run debug pod
kubectl run -it debug --image=curlimages/curl --restart=Never -- /bin/sh

# Connect to pod network
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
```

#### Port Forwarding

```bash
# Forward local port to pod port
kubectl port-forward my-pod 8080:8080

# Forward specific port
kubectl port-forward my-pod 8080:3000

# Forward to service
kubectl port-forward service/my-service 8080:80

# Forward from specific address
kubectl port-forward my-pod 127.0.0.1:8080:8080
```

### Resource Inspection

#### Describe Resources

```bash
# Describe deployment
kubectl describe deployment my-deployment

# Describe service
kubectl describe service my-service

# Describe node
kubectl describe node node-1

# Describe events
kubectl describe events
```

#### Get Resource Usage

```bash
# Current pod resource usage
kubectl top pod

# Pod resource usage in all namespaces
kubectl top pod -A

# Resource usage by node
kubectl top node

# Resource limits vs actual usage
kubectl describe node | grep -A 5 "Allocated resources"
```

### Advanced kubectl Commands

```bash
# Get events (sorted by time)
kubectl get events --sort-by='.lastTimestamp'

# Get events for specific pod
kubectl get events --field-selector involvedObject.name=my-pod

# Get resource API versions
kubectl api-resources

# Get API groups
kubectl api-versions

# Explain resource field
kubectl explain pod.spec.containers

# Dry-run to validate
kubectl create deployment my-app --image=myapp:latest --dry-run=client -o yaml

# Check if user can perform action
kubectl auth can-i create pods

# Generate YAML
kubectl create deployment my-app --image=myapp:latest -o yaml --dry-run=client
```

### Debugging Commands Summary

```bash
# Get pod status
kubectl get pod my-pod -o wide

# Describe pod (events, conditions)
kubectl describe pod my-pod

# Get logs (application output)
kubectl logs my-pod

# Get logs from previous container
kubectl logs my-pod --previous

# Execute command in pod
kubectl exec my-pod -- env

# Get shell access
kubectl exec -it my-pod -- /bin/bash

# Port forward to pod
kubectl port-forward my-pod 8080:8080

# Copy files from pod
kubectl cp my-pod:/path/to/file ./local-file

# Run debug container
kubectl run -it debug --image=curlimages/curl --restart=Never -- /bin/sh
```

---

## Events & Pod Logs

### Kubernetes Events

**Events** are objects that track cluster activities and state changes.

```yaml
apiVersion: v1
kind: Event
metadata:
  name: my-pod.17c6bae6ba86b1e8
  namespace: default
involvedObject:
  apiVersion: v1
  kind: Pod
  name: my-pod
  uid: abc123
reason: Scheduled           # Event type
message: "Successfully assigned pod to node-1"
type: Normal               # Normal or Warning
count: 1
firstTimestamp: 2024-01-15T10:30:00Z
lastTimestamp: 2024-01-15T10:30:00Z
source:
  component: default-scheduler
```

#### Getting Events

```bash
# Get all events
kubectl get events

# Get events sorted by time
kubectl get events --sort-by='.lastTimestamp'

# Get events for specific pod
kubectl get events --field-selector involvedObject.name=my-pod

# Get warning events
kubectl get events --field-selector type=Warning

# Watch events in real-time
kubectl get events -w

# Get events from specific namespace
kubectl get events -n my-namespace

# Get events by reason
kubectl get events --field-selector reason=Failed

# Describe events (more detail)
kubectl describe events

# Get events for specific object type
kubectl get events --field-selector involvedObject.kind=Pod
```

#### Common Pod Events

| Event | Reason | Meaning |
|-------|--------|---------|
| Normal | Scheduled | Pod assigned to node |
| Normal | Pulling | Image being pulled |
| Normal | Pulled | Image successfully pulled |
| Normal | Created | Container created |
| Normal | Started | Container started |
| Warning | ImagePullBackOff | Failed to pull image |
| Warning | BackOff | Restarting container |
| Warning | Failed | Container failed |
| Warning | FailedScheduling | Pod couldn't be scheduled |

### Pod Logs

#### Basic Log Retrieval

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    logs:                    # Logs are in this format
    # stdout and stderr combined
    # 2024-01-15T10:30:00Z INFO: Application started
    # 2024-01-15T10:30:01Z ERROR: Connection failed
```

```bash
# Get container logs
kubectl logs app-pod

# Get logs with timestamps
kubectl logs app-pod --timestamps=true

# Get logs from last 1 hour
kubectl logs app-pod --since=1h

# Get last 50 lines
kubectl logs app-pod --tail=50

# Get logs since specific time
kubectl logs app-pod --since-time=2024-01-15T10:00:00Z
```

#### Multi-Container Pod Logs

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: app
    image: myapp:latest
  - name: sidecar
    image: sidecar:latest
  - name: logging
    image: logging-agent:latest
```

```bash
# Get logs from all containers
kubectl logs multi-container-pod --all-containers=true

# Get logs from specific container
kubectl logs multi-container-pod -c app

# Get logs from previous instance
kubectl logs multi-container-pod --previous

# Get logs from crashed container
kubectl logs multi-container-pod -c app --previous
```

#### Streaming Logs

```bash
# Follow logs (like tail -f)
kubectl logs app-pod -f

# Follow logs from multiple pods
kubectl logs -l app=myapp -f

# Follow logs without timestamps
kubectl logs app-pod -f --timestamps=false

# Follow logs from specific container
kubectl logs app-pod -c app -f
```

#### Log Analysis

```bash
# Filter logs
kubectl logs app-pod | grep ERROR

# Get only ERROR lines
kubectl logs app-pod | grep "ERROR"

# Get lines around error (context)
kubectl logs app-pod | grep -C 3 "ERROR"

# Count log lines
kubectl logs app-pod | wc -l

# Get unique errors
kubectl logs app-pod | grep "ERROR" | sort | uniq

# Get logs from multiple pods
kubectl logs -l app=myapp | grep "ERROR"
```

### Log Levels and Best Practices

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: well-logged-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        # Environment for log level
        env:
        - name: LOG_LEVEL
          value: "INFO"          # DEBUG, INFO, WARN, ERROR
        - name: LOG_FORMAT
          value: "json"          # json or text
        # Output logs to stdout/stderr (captured by kubelet)
        volumeMounts:
        - name: logs
          mountPath: /var/log
      volumes:
      - name: logs
        emptyDir: {}
```

---

## Health Checks (Probes)

**Probes** assess Pod health and availability. Three types:

```
┌─────────────────────────────────────┐
│       Kubernetes Health Checks      │
├─────────────────────────────────────┤
│                                     │
│  Liveness Probe   → Is it alive?   │
│  Readiness Probe  → Ready to serve? │
│  Startup Probe    → Ready to start? │
│                                     │
└─────────────────────────────────────┘
```

### Liveness Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-probe-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    
    # HTTP Liveness Check
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
        scheme: HTTP
      initialDelaySeconds: 10    # Wait before first check
      periodSeconds: 10          # Check every 10s
      timeoutSeconds: 5          # Wait 5s for response
      successThreshold: 1        # 1 success = healthy
      failureThreshold: 3        # 3 failures = restart
```

**Purpose**: Restart container if it becomes unhealthy

### Readiness Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-probe-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    
    # HTTP Readiness Check
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      successThreshold: 1
      failureThreshold: 2
```

**Purpose**: Remove Pod from Service endpoints if not ready (traffic stops)

### Startup Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-probe-pod
spec:
  containers:
  - name: app
    image: slow-starting-app:latest
    
    # TCP Startup Check (for slow-starting apps)
    startupProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 3
      successThreshold: 1
      failureThreshold: 30        # Allow 300s (30 * 10s) startup time
    
    # Liveness only starts after startup passes
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      periodSeconds: 10
      failureThreshold: 3
```

**Purpose**: Give slow-starting apps time to initialize

### Probe Types

#### 1. HTTP Probe

```yaml
livenessProbe:
  httpGet:
    host: localhost
    path: /health
    port: 8080
    scheme: HTTP              # HTTP or HTTPS
    httpHeaders:
    - name: X-Long-Connection
      value: "5s"
  initialDelaySeconds: 10
  periodSeconds: 10
```

#### 2. TCP Probe

```yaml
livenessProbe:
  tcpSocket:
    host: localhost
    port: 5432                # Just check if port is open
  initialDelaySeconds: 15
  periodSeconds: 20
```

#### 3. Exec Probe

```yaml
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - "curl -f http://localhost:8080/health || exit 1"
  initialDelaySeconds: 10
  periodSeconds: 10
```

### Probe Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| **initialDelaySeconds** | 0 | Delay before first probe |
| **periodSeconds** | 10 | Probe frequency |
| **timeoutSeconds** | 1 | Response timeout |
| **successThreshold** | 1 | Successes needed for ready |
| **failureThreshold** | 3 | Failures allowed before action |

### Practical Health Check Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: healthy-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
          name: http
        
        # Startup probe for app initialization
        startupProbe:
          httpGet:
            path: /startup
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
        
        # Readiness probe for traffic readiness
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        
        # Liveness probe for crash detection
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
          failureThreshold: 3
        
        # Resource limits
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

---

## Metrics-Server

**Metrics-Server** collects CPU and memory metrics from kubelet for `kubectl top` and autoscaling.

### Architecture

```
┌──────────────────────────────────┐
│    Kubernetes API Server         │
└────────────────┬─────────────────┘
                 │
                 ▼
┌──────────────────────────────────┐
│       Metrics Server             │
│  (Collects and serves metrics)   │
└────────────────┬─────────────────┘
                 │
      ┌──────────┴──────────┐
      │                     │
      ▼                     ▼
   kubelet              kubelet
   (node-1)             (node-2)
```

### Installation

```bash
# Check if metrics-server is running
kubectl get deployment -n kube-system | grep metrics-server

# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify installation
kubectl get pods -n kube-system -l k8s-app=metrics-server

# Wait for metrics to be available (takes ~1 minute)
kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system --timeout=300s
```

### Viewing Metrics

#### Node Metrics

```bash
# View node CPU and memory
kubectl top node

# View specific node
kubectl top node node-1

# View with more precision
kubectl top node --no-headers

# Export metrics
kubectl top node -o json
```

#### Pod Metrics

```bash
# View pod metrics
kubectl top pod

# View pods in all namespaces
kubectl top pod -A

# View specific pod
kubectl top pod my-pod

# View specific container
kubectl top pod my-pod -c container-name

# View by namespace
kubectl top pod -n my-namespace

# Sort by memory usage
kubectl top pod --sort-by=memory

# Sort by CPU usage
kubectl top pod --sort-by=cpu

# Show only pods using high CPU
kubectl top pod | sort -k2 -rn | head -5
```

### Metrics Format

```bash
# Example output
$ kubectl top pod
NAME                    CPU(m)    MEMORY(Mi)
my-pod-123              150       256
my-pod-456              50        128

# m = millicores (1000m = 1 CPU)
# Mi = Mebibytes

# Example node output
$ kubectl top node
NAME      CPU(m)    CPU%    MEMORY(Mi)   MEMORY%
node-1    500       50%     2048         80%
node-2    200       20%     1024         40%
```

### Hardware Resource Requests

Metrics help capacity planning:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: optimized-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        
        # Based on kubectl top metrics, set appropriate resources
        resources:
          requests:
            memory: "256Mi"      # Scheduler needs this to schedule
            cpu: "100m"
          limits:
            memory: "512Mi"      # OOMKilled if exceeded
            cpu: "500m"          # Throttled if exceeded
```

---

## Prometheus & Grafana Basics

**Prometheus** scrapes metrics from Kubernetes components. **Grafana** visualizes them.

### Architecture

```
┌──────────────────────────────────────┐
│    Kubernetes Cluster                │
│  ┌─────────────────────────────────┐ │
│  │  Prometheus                     │ │
│  │  (Scrapes metrics every 15s)   │ │
│  └──────────────┬──────────────────┘ │
│                 │                    │
│  ┌──────────────▼──────────────────┐ │
│  │  Time-series Database           │ │
│  │  (Stores metrics)               │ │
│  └─────────────────────────────────┘ │
└──────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│         Grafana                      │
│    (Visualizes metrics)              │
│    (Dashboards, Alerts)              │
└──────────────────────────────────────┘
```

### Installation with Helm

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus Stack (includes Grafana)
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace

# Verify installation
kubectl get pods -n monitoring

# Get Grafana password
kubectl get secret -n monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d
```

### Accessing Grafana

```bash
# Port forward to Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# Open browser
# http://localhost:3000
# Username: admin
# Password: (from above)
```

### Prometheus Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s      # Scrape every 15s
      evaluation_interval: 15s  # Evaluate rules every 15s
    
    scrape_configs:
    # Scrape Kubernetes API server
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
    
    # Scrape kubelet metrics
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    
    # Scrape pod metrics
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
    
    # Scrape service metrics
    - job_name: 'kubernetes-services'
      kubernetes_sd_configs:
      - role: service
```

### Prometheus Queries (PromQL)

#### Basic Queries

```promql
# Current CPU usage (all pods)
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)

# Current memory usage
container_memory_usage_bytes

# Node CPU usage percentage
(1 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))) * 100

# Pod memory as percentage of limit
(container_memory_usage_bytes / container_spec_memory_limit_bytes) * 100
```

#### Common Metrics

| Metric | Unit | Purpose |
|--------|------|---------|
| **container_cpu_usage_seconds_total** | cores | Total CPU time |
| **container_memory_usage_bytes** | bytes | Current memory |
| **container_network_receive_bytes_total** | bytes | Network received |
| **container_network_transmit_bytes_total** | bytes | Network sent |
| **node_cpu_seconds_total** | seconds | Node CPU time |
| **node_memory_MemAvailable_bytes** | bytes | Available memory |

### Grafana Dashboard Creation

#### Step 1: Create Dashboard
1. In Grafana UI: + → Dashboard → New Panel
2. Select Prometheus data source
3. Enter PromQL query

#### Step 2: Common Panels

**CPU Usage Panel**
```
Query: sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
Format: Time series
Visualization: Graph
```

**Memory Usage Panel**
```
Query: container_memory_usage_bytes / 1024 / 1024
Format: Time series
Unit: MB
Visualization: Graph
```

**Pod Restart Count**
```
Query: kube_pod_container_status_restarts_total
Format: Table
Visualization: Table
```

### ServiceMonitor for Custom Metrics

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitoring
  namespace: default
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics          # Prometheus metrics endpoint
```

### PrometheusRule for Alerts

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
  namespace: monitoring
spec:
  groups:
  - name: my-app.rules
    interval: 30s
    rules:
    # Alert if pod is not ready
    - alert: PodNotReady
      expr: kube_pod_status_ready{condition="false"} == 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} is not ready"
    
    # Alert if high memory usage
    - alert: HighMemoryUsage
      expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} memory usage is {{ $value | humanizePercentage }}"
```

---

## Troubleshooting Guide

### Pod Won't Start

```bash
# Get pod status
kubectl describe pod my-pod

# Check events
kubectl get events --field-selector involvedObject.name=my-pod

# Common scenarios:
# 1. ImagePullBackOff → Image doesn't exist or private registry
# 2. CrashLoopBackOff → Container crashes on start
# 3. Pending → Not enough resources or node selector mismatch
```

### Pod Crashes Repeatedly

```bash
# Get crash reason
kubectl describe pod my-pod

# Get logs from crashed container
kubectl logs my-pod --previous

# Check resource limits
kubectl describe pod my-pod | grep -A 5 "Limits"

# Check if OOMKilled
kubectl describe pod my-pod | grep -i "OOMKilled"
```

### Pod Can't Connect to Service

```bash
# Verify service exists
kubectl get service my-service

# Get service endpoints
kubectl get endpoints my-service

# Verify pod is running
kubectl get pods -l app=my-app

# Test DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup my-service

# Test connectivity
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://my-service:8080/health
```

### High CPU/Memory Usage

```bash
# Find offending pods
kubectl top pod -A | sort -k3 -rn | head -10

# Get resource requests/limits
kubectl describe pod my-pod | grep -A 5 "Requests"

# Check metrics history
kubectl top pod my-pod --sort-by=cpu

# Scale horizontally
kubectl scale deployment my-app --replicas=5

# Adjust requests/limits
kubectl set resources deployment my-app -c app --requests=cpu=100m,memory=256Mi
```

### Node Issues

```bash
# Check node status
kubectl get nodes

# Describe problematic node
kubectl describe node node-1

# Check node resources
kubectl top node node-1

# Check node events
kubectl get events --field-selector involvedObject.name=node-1

# Drain node for maintenance
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# Cordoning node (prevent new pods)
kubectl cordon node-1

# Uncordon node
kubectl uncordon node-1
```

### Debugging Multi-Container Pods

```bash
# List containers in pod
kubectl get pod my-pod -o jsonpath='{.spec.containers[*].name}'

# Get logs from specific container
kubectl logs my-pod -c container-name

# Exec into specific container
kubectl exec -it my-pod -c container-name -- /bin/bash

# Debug sidecar container
kubectl logs my-pod -c sidecar --previous
```

---

## Best Practices

### Logging
- ✅ Log to stdout/stderr (captured by kubelet)
- ✅ Use structured logging (JSON format)
- ✅ Include request IDs for tracing
- ✅ Use appropriate log levels
- ✅ Avoid logging sensitive data (passwords, tokens)
- ❌ Don't log to files inside container
- ❌ Don't use syslog

### Health Checks
- ✅ Implement all three probe types for production
- ✅ Set appropriate initial delay
- ✅ Use `/health` endpoint for liveness
- ✅ Use `/ready` endpoint for readiness
- ✅ Return correct HTTP status codes
- ❌ Don't just check if process exists
- ❌ Don't run external commands if possible

### Monitoring
- ✅ Set resource requests and limits
- ✅ Monitor CPU, memory, disk, network
- ✅ Set up alerting for critical metrics
- ✅ Use custom metrics for application health
- ✅ Regularly review metrics and adjust resources
- ❌ Don't set limits too low
- ❌ Don't ignore warning alerts

### Example: Production-Ready Application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      # Pod disruption budget
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: app
        image: myapp:v1.0.0
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090
        
        # Environment for logging
        env:
        - name: LOG_LEVEL
          value: "INFO"
        - name: LOG_FORMAT
          value: "json"
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        
        # Startup probe (20 seconds to start)
        startupProbe:
          httpGet:
            path: /startup
            port: http
          failureThreshold: 30
          periodSeconds: 10
        
        # Readiness probe (traffic only if ready)
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
        
        # Liveness probe (restart if hung)
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 15
          periodSeconds: 20
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Resource management
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Security context
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
        
        # Writable volumes
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
      
      # Volumes
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
      
      # Termination grace
      terminationGracePeriodSeconds: 30

---
# Pod Disruption Budget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

---

## Summary

- **kubectl Commands**: Primary tool for cluster interaction and debugging
- **Events & Logs**: Understand what happened and when through events and application logs
- **Health Checks**: Startup, Readiness, and Liveness probes for reliable applications
- **Metrics-Server**: Provides basic CPU/memory metrics via `kubectl top`
- **Prometheus & Grafana**: Comprehensive monitoring, metrics collection, and visualization
- **Troubleshooting**: Systematic approach using logs, events, metrics, and kubectl commands

Effective logging and monitoring are essential for maintaining healthy, reliable Kubernetes clusters!
