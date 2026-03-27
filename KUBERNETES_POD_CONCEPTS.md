# Kubernetes Pod Fundamentals

## Table of Contents
1. [Pod Fundamentals](#pod-fundamentals)
2. [Multi-Container Pods (Sidecar Pattern)](#multi-container-pods-sidecar-pattern)
3. [Pod Lifecycle Phases](#pod-lifecycle-phases)
4. [Init Containers](#init-containers)

---

## Pod Fundamentals

### What is a Pod?

A **Pod** is the smallest deployable unit in Kubernetes. It's essentially a wrapper around one or more containers (usually Docker containers) that run together on the same node.

#### Key Characteristics:
- Containers in a Pod share the **same network namespace** (same IP address, can communicate via localhost)
- Containers can share **storage volumes**
- Pods are **ephemeral** — they're created and destroyed dynamically
- Typically runs **one container per Pod**, but can run multiple for sidecar patterns

### Why is a Pod the Smallest Deployable Unit?

- You cannot deploy a bare container in Kubernetes—only Pods
- It provides isolation and networking configuration
- It abstracts container complexity while enabling multi-container patterns

### Basic Pod Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: arunpod
  labels:
    app: student
spec:
  containers:
    - name: my-nginx
      image: nginx
      ports:
        - containerPort: 80
```

This creates a Pod with a single nginx container running on port 80.

---

## Multi-Container Pods (Sidecar Pattern)

A Pod can contain **multiple containers** that work together. The most common pattern is the **Sidecar Pattern**.

### What is the Sidecar Pattern?

A sidecar is a secondary container that enhances or extends the functionality of the main container.

#### Common Use Cases:
- **Logging sidecar**: Collects logs from the main app and forwards to a centralized logging system
- **Monitoring sidecar**: Monitors the main app and exports metrics
- **Security proxy**: Acts as a service mesh proxy
- **Data sync**: Synchronizes data with external systems

### Example: Main App + Logging Sidecar

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-logging
spec:
  containers:
  # Main application container
  - name: app-container
    image: my-app:1.0
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  
  # Sidecar logging container
  - name: logging-sidecar
    image: fluentd:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  
  # Shared volume
  volumes:
  - name: logs
    emptyDir: {}
```

### How Containers Communicate in a Pod

```bash
# Both containers share the same network
# Main app listens on localhost:8080
# Sidecar can reach it via: localhost:8080
# Both share IP address (e.g., 10.0.0.5)
```

**Communication Methods:**
- **localhost**: Direct port-to-port communication within the same Pod
- **Shared volumes**: Exchange data through mounted storage
- **IPC namespace**: Shared inter-process communication (optional)

---

## Pod Lifecycle Phases

When a Pod is created, it goes through several phases:

### Pod Phases

| Phase | Description |
|-------|------------|
| **Pending** | Pod created but not running yet. Scheduler finding node, pulling image, etc. |
| **Running** | All containers started successfully |
| **Succeeded** | All containers exited successfully (for batch jobs) |
| **Failed** | One or more containers exited with error |
| **Unknown** | Cannot determine Pod state |

### Pod Status Conditions

Pods also have conditions that provide more details:

- **PodScheduled**: Pod assigned to a node
- **Ready**: Pod ready to accept traffic
- **Initialized**: Init containers completed
- **ContainersReady**: All containers ready

### Pod Lifecycle Flow

```
1. Pod Created
   ↓
2. Assigned to Node (PodScheduled=True)
   ↓
3. Container Images Pulled
   ↓
4. Init Containers Run (if any)
   ↓
5. Main Containers Start
   ↓
6. Ready Condition Set (Ready=True)
   ↓
7. Pod Running (or Completed)
```

### Checking Pod Lifecycle

```bash
# Create Pod
kubectl apply -f pod.yml

# Check pod status (shows phase and conditions)
kubectl get pod arunpod

# Get detailed information
kubectl describe pod arunpod
```

**Output Example:**
```
Name: arunpod
Status: Running
Phase: Running

Conditions:
  Type             Status
  PodScheduled     True
  Ready            True
  ContainersReady  True
  Initialized      True

Events:
  Type    Reason                 Message
  ----    ------                 -------
  Normal  Scheduled              Successfully assigned default/arunpod to node-1
  Normal  Pulling                Pulling image "nginx"
  Normal  Pulled                 Successfully pulled image "nginx"
  Normal  Created                Created container my-nginx
  Normal  Started                Started container my-nginx
```

### Watching Pod Status Changes

```bash
# Watch pod status in real-time
kubectl get pods -w

# Get detailed events for debugging
kubectl describe pod arunpod
kubectl logs arunpod
```

---

## Init Containers

**Init Containers** are specialized containers that run before the main app containers start. They must complete successfully before the main containers can run.

### Use Cases for Init Containers

- **Setup prerequisites**: Wait for dependent services to be ready
- **Download/prepare data**: Fetch config files or data
- **Initialization scripts**: Set up environment
- **Permission configuration**: Initialize file permissions or ownership
- **Wait for external services**: Database, message queues, APIs

### Key Characteristics of Init Containers

- Run **sequentially** (one after another, in order defined)
- Must exit with status 0 (success)
- Always run to completion (even if restart policy is Never)
- Main containers won't start until **all** init containers succeed
- If an init container fails, the Pod is restarted (respecting restart policy)

### Basic Example: Init Container for Setup

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  # Init containers run first
  initContainers:
  - name: wait-for-db
    image: busybox:1.28
    command: ['sh', '-c', 'echo "Waiting for DB..."; sleep 5; echo "DB ready"']
  
  - name: setup-config
    image: busybox:1.28
    command: ['sh', '-c', 'echo "Setting up configuration..."']
  
  # Main application containers run after init containers complete
  containers:
  - name: app
    image: my-app:1.0
    ports:
    - containerPort: 8080
```

### Execution Flow

```
1. Pod created
   ↓
2. wait-for-db init container runs → completes successfully
   ↓
3. setup-config init container runs → completes successfully
   ↓
4. Main app container starts
```

### Real-World Example: Wait for Database

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-db
  labels:
    app: myapp
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.28
    command:
    - 'sh'
    - '-c'
    - 'until nc -z postgres-db 5432; do echo "Waiting for database..."; sleep 2; done; echo "Database is ready!"'
  
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DB_HOST
      value: postgres-db
    - name: DB_PORT
      value: "5432"
    - name: DB_NAME
      value: myapp_db
    ports:
    - containerPort: 8080
```

### Real-World Example: Initialize Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
spec:
  initContainers:
  - name: init-config
    image: busybox:1.28
    command:
    - 'sh'
    - '-c'
    - |
      echo "Downloading configuration..."
      wget -O /config/app.conf https://config-server/app.conf
      echo "Configuration initialized successfully"
    volumeMounts:
    - name: config
      mountPath: /config
  
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: config
      mountPath: /etc/app
  
  volumes:
  - name: config
    emptyDir: {}
```

### Init Container with Shared Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-shared-volume
spec:
  initContainers:
  - name: setup
    image: alpine
    command:
    - 'sh'
    - '-c'
    - |
      echo "Setting up application files..."
      cp /init/files/* /app/data/
      chmod -R 755 /app/data/
    volumeMounts:
    - name: app-data
      mountPath: /app/data
  
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: app-data
      mountPath: /app/data
  
  volumes:
  - name: app-data
    emptyDir: {}
```

### Debugging Init Containers

```bash
# View init container logs
kubectl logs pod-name -c init-container-name

# Check init container completion status
kubectl describe pod pod-name

# Look at events to understand failures
kubectl get events --sort-by='.lastTimestamp'
```

---

## Complete Pod Reference Structure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: complete-pod
  namespace: default
  labels:
    app: myapp
    version: v1
  annotations:
    description: "Complete Pod with all components"
spec:
  # Init containers (run before main containers)
  initContainers:
  - name: init-setup
    image: busybox:1.28
    command: ['sh', '-c', 'echo "Init setup completed"']
    volumeMounts:
    - name: init-volume
      mountPath: /init
  
  # Main containers
  containers:
  - name: app
    image: myapp:1.0
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 8080
      name: http
      protocol: TCP
    
    # Environment variables
    env:
    - name: ENV
      value: production
    - name: LOG_LEVEL
      value: info
    
    # Volume mounts
    volumeMounts:
    - name: app-data
      mountPath: /app/data
    - name: config
      mountPath: /etc/app
    
    # Resource requests and limits
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    
    # Liveness probe
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    
    # Readiness probe
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
  
  # Shared volumes
  volumes:
  - name: app-data
    emptyDir: {}
  - name: config
    configMap:
      name: app-config
  - name: init-volume
    emptyDir: {}
  
  # Pod-level settings
  restartPolicy: Always
  terminationGracePeriodSeconds: 30
  dnsPolicy: ClusterFirst
```

---

## Quick Comparison Table

| Aspect | Details |
|--------|---------|
| **Smallest Unit** | Pod (not container) |
| **Container Count** | 1 or more |
| **Network** | Shared IP, localhost communication |
| **Storage** | Shared volumes possible |
| **Init Containers** | Run sequentially before main containers |
| **Sidecar Pattern** | Secondary container for enhancement |
| **Lifecycle** | Pending → Running → Succeeded/Failed |
| **Ephemeral** | Created and destroyed dynamically |

---

## Best Practices

1. **Single Container Pods**: Use one container per Pod unless there's a specific need (logging, monitoring, security proxy)
2. **Init Container Ordering**: Put critical setup in init containers to ensure Pod readiness
3. **Resource Limits**: Define resource requests and limits for all containers
4. **Probes**: Implement liveness and readiness probes for reliability
5. **Labels**: Use labels for Pod selection and organization
6. **Graceful Shutdown**: Set appropriate `terminationGracePeriodSeconds`
7. **Design for Failure**: Assume Pods can fail and design accordingly

---

## Summary

- **Pods** are the fundamental Kubernetes unit, wrapping containers with networking and storage
- **Multi-container Pods** (sidecars) enable separation of concerns and enhanced functionality
- **Pod Lifecycle** tracks the Pod from creation to completion
- **Init Containers** prepare the Pod environment before main application starts

Understanding these concepts is essential for effective Kubernetes deployment and management.
