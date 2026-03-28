# Kubernetes Custom Resources & Operators + Service Mesh

## Table of Contents
1. [Custom Resources](#custom-resources)
2. [CRDs (Custom Resource Definitions)](#crds-custom-resource-definitions)
3. [Operators](#operators)
4. [Operator Pattern](#operator-pattern)
5. [Service Mesh Fundamentals](#service-mesh-fundamentals)
6. [Istio Basics](#istio-basics)
7. [Linkerd Basics](#linkerd-basics)
8. [Sidecar Proxies](#sidecar-proxies)
9. [mTLS in Service Mesh](#mtls-in-service-mesh)
10. [Traffic Management](#traffic-management)
11. [Observability in Service Mesh](#observability-in-service-mesh)
12. [Best Practices](#best-practices)

---

# Custom Resources

## Custom Resources Overview

**Custom Resources** extend Kubernetes API with domain-specific resources beyond built-in objects (Pod, Service, etc).

### Problem with Native Resources

```
Built-in Kubernetes Resources:
├── Pod, Deployment, StatefulSet → Pod management
├── Service, Ingress → Networking
├── PV, PVC → Storage
├── Secret, ConfigMap → Configuration
└── RBAC objects → Security

What about domain-specific needs?
├── Database clusters
├── Message queues
├── Machine learning models
├── Certificate management
└── Custom workflows
```

### Custom Resources Solution

```yaml
# Define custom resource
apiVersion: databases.example.com/v1alpha1
kind: PostgresCluster
metadata:
  name: my-database
spec:
  size: 3
  version: "13.0"
  storage: 10Gi

# Use like native resources
kubectl get postgresclusters
kubectl describe postgesclusters my-database
kubectl delete postgesclusters my-database
```

### Custom Resources vs StaticMan Pods

| Approach | Complexity | Flexibility | Management |
|----------|-----------|-------------|-----------|
| **Manual Pods** | Low | Limited | Manual |
| **Deployment** | Low | Limited | kubectl |
| **Custom Resource** | Medium-High | High | Operator |

---

# CRDs (Custom Resource Definitions)

**CRDs** define the schema and behavior of custom resources.

### CRD Structure

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.databases.example.com
spec:
  names:
    kind: PostgresCluster          # Resource type
    plural: postgresclusters       # Plural name
    singular: postgrescluster      # Singular name
    shortNames:
    - pg                           # kubectl get pg
  scope: Namespaced               # Namespaced or Cluster
  group: databases.example.com    # API group
  
  versions:
  - name: v1alpha1                 # Version
    served: true                   # Available to users
    storage: true                  # Storage version
    
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size:
                type: integer
                minimum: 1
                maximum: 100
                description: "Number of replicas"
              
              version:
                type: string
                pattern: '^\d+\.\d+\.\d+$'
                description: "PostgreSQL version"
              
              storage:
                type: string
                pattern: '^\d+Gi$'
                description: "Storage size"
            
            required:
            - size
            - version
            - storage
          
          status:
            type: object
            properties:
              ready:
                type: boolean
              replicas:
                type: integer
              phase:
                type: string
                enum:
                - Creating
                - Ready
                - Scaling
                - Error
```

### CRD Validation

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.content.example.com
spec:
  names:
    kind: Website
    plural: websites
  scope: Namespaced
  group: content.example.com
  
  versions:
  - name: v1
    served: true
    storage: true
    
    schema:
      openAPIV3Schema:
        type: object
        properties:
          apiVersion:
            type: string
          kind:
            type: string
          metadata:
            type: object
          
          spec:
            type: object
            properties:
              domain:
                type: string
                pattern: '^[a-z0-9][a-z0-9-]*[a-z0-9]$'
              
              replicas:
                type: integer
                minimum: 1
                maximum: 50
              
              tls:
                type: boolean
              
              tags:
                type: array
                items:
                  type: string
            
            required:
            - domain
            - replicas
          
          status:
            type: object
            properties:
              ready:
                type: boolean
              url:
                type: string
              phase:
                type: string
```

### Custom Resource Instance

```yaml
apiVersion: content.example.com/v1
kind: Website
metadata:
  name: my-blog
  namespace: default
spec:
  domain: myblog.com
  replicas: 3
  tls: true
  tags:
  - blog
  - cms
status:
  ready: true
  url: https://myblog.com
  phase: Ready
```

### CRD Commands

```bash
# List CRDs
kubectl get crd

# Describe CRD
kubectl describe crd postgresclusters.databases.example.com

# Create CRD
kubectl apply -f postgres-crd.yaml

# Create custom resource instance
kubectl apply -f postgres-instance.yaml

# Get custom resources
kubectl get postgresclusters
kubectl get pg  # Using short name

# Describe instance
kubectl describe postgesclusters my-database

# Delete CRD (cascades to instances)
kubectl delete crd postgresclusters.databases.example.com
```

### CRD Versions and Upgrades

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  names:
    kind: Database
    plural: databases
  group: example.com
  scope: Namespaced
  
  # Multiple versions
  versions:
  - name: v1alpha1
    served: false               # Old version, not served
    storage: false
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size: { type: integer }
  
  - name: v1beta1
    served: true                # Current version
    storage: true               # New storage version
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas: { type: integer }
              config: { type: string }
    
    # Convert old version to new
    conversion:
      strategy: None
```

---

# Operators

## Operator Pattern

**Operators** are Kubernetes-native applications that automate custom resource management.

### Operator Architecture

```
┌────────────────────────────────────────┐
│      Kubernetes Cluster                │
├────────────────────────────────────────┤
│                                        │
│  CRD Definition                        │
│  └─ PostgresCluster                    │
│                                        │
│  Custom Resources (Instances)          │
│  ├─ my-database                        │
│  ├─ prod-database                      │
│  └─ backup-database                    │
│                                        │
│  Operator Pod                          │
│  └─ Watches for changes                │
│     Reconciles desired → actual state  │
│                                        │
│  Generated Resources                   │
│  ├─ StatefulSets                       │
│  ├─ Services                           │
│  ├─ ConfigMaps                         │
│  └─ PVCs                               │
│                                        │
└────────────────────────────────────────┘
```

### Operator Reconciliation Loop

```
Change detected
    │
    ▼
Read current resource spec
    │
    ▼
Check actual state
    │
    ├─ Matches desired? → Done
    │
    └─ Doesn't match?
        │
        ▼
    Create/Update/Delete
    managed resources
        │
        ▼
    Update status/conditions
        │
        ▼
    Requeue if needed
```

### Popular Operators

| Operator | Purpose | Manages |
|----------|---------|---------|
| **Prometheus** | Monitoring | ServiceMonitor, PrometheusRule |
| **Etcd** | Database | EtcdCluster |
| **PostgreSQL** | Database | PostgresCluster |
| **cert-manager** | Certificates | Certificate, ClusterIssuer |
| **ArgoCD** | GitOps | Application, AppProject |

---

## Deploying Operators

### PostgreSQL Operator Example

```bash
# Install via Helm
helm repo add zalando https://charts.zalando.dev
helm install postgres-operator zalando/postgres-operator \
  -n postgres-system \
  --create-namespace

# Verify installation
kubectl get pods -n postgres-system
kubectl get crd | grep zalando
```

### PostgreSQL Cluster via Operator

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-postgres
  namespace: default
spec:
  instances: 3                    # Number of replicas
  
  primaryUpdateStrategy: unsupervised
  
  imageName: ghcr.io/cloudnative-pg/cloudnative-pg:15.2
  
  # PostgreSQL configuration
  postgresql:
    parameters:
      max_parallel_workers_per_gather: "4"
      max_parallel_workers: "4"
  
  # Storage configuration
  storage:
    size: 20Gi
  
  # Restart strategy
  failoverDelay: 300
  
  # Monitoring
  monitoring:
    enabled: true
    podMonitorSpec:
      interval: 30s
  
  # Backup configuration
  backup:
    barmanObjectStore:
      wal:
        maxParallel: 4
      compression: gzip
      endpointURL: "https://s3.example.com"
      destinationPath: "s3://backup-bucket"
      s3Credentials:
        accessKeyId:
          name: aws-creds
          key: access_key_id
        secretAccessKey:
          name: aws-creds
          key: secret_access_key
  
  # Bootstrap from backup
  bootstrap:
    recovery:
      source: my-postgres-backup
      recoveryTarget:
        timeline: latest

---
# Manager configures operator behavior
apiVersion: postgresql.cnpg.io/v1
kind: OperatorConfiguration
metadata:
  name: cnpg
spec:
  # Operator watches for CRD changes
  monitoringConfiguration:
    metricsEnabled: true
    portMetrics: 8432
```

### Verifying Operator-Created Resources

```bash
# View created resources
kubectl get statefulsets
kubectl get services -l cnpg.io/cluster=my-postgres
kubectl get pvc

# Check cluster status
kubectl describe cluster my-postgres

# View cluster pods
kubectl get pods -l cnpg.io/cluster=my-postgres

# Check operator logs
kubectl logs -n postgres-system -l app.kubernetes.io/name=cloudnative-pg
```

---

# Service Mesh Fundamentals

## What is a Service Mesh?

**Service Mesh** is a dedicated infrastructure layer for managing service-to-service communication.

### Problems Service Mesh Solves

```
Without Service Mesh:
├─ Each service implements network logic
│  ├─ Retries
│  ├─ Circuit breaking
│  ├─ Authentication/Authorization
│  └─ Distributed tracing
├─ Error-prone across languages
├─ Increases app complexity
└─ Hard to change policies

With Service Mesh:
├─ Centralized policy management
├─ Transparent to applications
├─ Language-agnostic
├─ Consistent observability
└─ Easy policy updates
```

### Service Mesh Architecture

```
┌─────────────────────────────────────────────┐
│           Kubernetes Cluster                │
├─────────────────────────────────────────────┤
│                                             │
│  ┌──────────────┐        ┌──────────────┐ │
│  │   Service A  │        │   Service B  │ │
│  └───────┬──────┘        └──────┬───────┘ │
│          │                      │         │
│          ▼                      ▼         │
│  ┌──────────────┐        ┌──────────────┐ │
│  │ Sidecar Proxy│        │ Sidecar Proxy│ │
│  │  (Envoy)     │        │  (Envoy)     │ │
│  └──────────────┘        └──────────────┘ │
│          ▲                      ▲         │
│          │                      │         │
│          └──────────┬───────────┘         │
│                     │                     │
│          ┌──────────▼──────────┐          │
│          │  Service Mesh       │          │
│          │  Control Plane      │          │
│          │  (Istiod/Linkerd)   │          │
│          └─────────────────────┘          │
│                                             │
└─────────────────────────────────────────────┘
```

### Key Components

| Component | Purpose |
|-----------|---------|
| **Sidecar Proxy** | Intercepts traffic, applies policies |
| **Control Plane** | Manages proxy configuration |
| **CRDs** | Traffic management resources |
| **Observability** | Metrics, traces, logs |

---

# Istio Basics

**Istio** is a popular open-source service mesh for Kubernetes.

## Istio Installation

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.x.x

# Install with demo profile
./bin/istioctl install --set profile=demo -y

# Verify installation
kubectl get pods -n istio-system
kubectl get crd | grep istio

# Enable sidecar injection for namespace
kubectl label namespace default istio-injection=enabled

# Check injection
kubectl get namespace default --show-labels
```

## Istio Traffic Management

### VirtualService

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-vs
spec:
  hosts:
  - api                           # Service name
  http:
  - match:
    - uri:
        prefix: /v1
    route:
    - destination:
        host: api
        port:
          number: 8080
        subset: v1                # Subset (canary)
      weight: 90                  # 90% traffic
    - destination:
        host: api
        port:
          number: 8080
        subset: v2                # New version
      weight: 10                  # 10% traffic (canary)
    timeout: 30s
    retries:
      attempts: 3
      perTryTimeout: 10s
```

### DestinationRule

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-dr
spec:
  host: api                       # Service to configure
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    loadBalancer:
      simple: ROUND_ROBIN        # Load balancing strategy
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

### Gateway and Route

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: api-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - api.example.com

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-routing
spec:
  hosts:
  - api.example.com
  gateways:
  - api-gateway
  http:
  - route:
    - destination:
        host: api
        port:
          number: 80
```

---

# Linkerd Basics

**Linkerd** is a lightweight service mesh focused on reliability.

## Linkerd Installation

```bash
# Download Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# Add to PATH
export PATH=$PATH:$HOME/.linkerd2/bin

# Install Linkerd
linkerd install | kubectl apply -f -

# Verify installation
kubectl get pods -n linkerd
linkerd check

# Enable sidecar injection
kubectl annotate namespace default linkerd.io/inject=enabled

# Check injection
kubectl get namespace default --show-labels
```

## Linkerd Traffic Policy

```yaml
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata:
  name: api-server
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: api
  port: 8080
  protocol: HTTP

---
apiVersion: policy.linkerd.io/v1beta1
kind: ServerAuthorization
metadata:
  name: api-authz
  namespace: default
spec:
  server:
    selector:
      matchLabels:
        app: api
  client:
    meshTLS:
      unauthenticatedTLS: false
    networks:
    - cidr: 10.0.0.0/8

---
apiVersion: policy.linkerd.io/v1beta1
kind: HTTPRoute
metadata:
  name: api-route
  namespace: default
spec:
  parentRefs:
  - name: api
    kind: Service
    port: 8080
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api
      port: 8080
```

---

# Sidecar Proxies

**Sidecar Proxies** are lightweight containers that intercept and manage network traffic.

### How Sidecar Proxies Work

```
Application Container     Sidecar Proxy
        │                      │
        │──── Traffic ────────>│
        │                      │
        │  Apply policies      │
        │  - mTLS              │
        │  - Retry logic       │
        │  - Rate limiting     │
        │                      │
        │<──── Response ───────│
        │                      │
```

### Istio Sidecar Injection

#### Automatic Injection

```bash
# Enable auto-injection
kubectl label namespace staging istio-injection=enabled

# Deploy pod (injection automatic)
kubectl apply -f app.yaml

# View injected pod
kubectl get pod -o yaml | grep -A 10 "containers:"
```

#### Manual Injection

```bash
# Inject into deployment
istioctl kube-inject -f app.yaml | kubectl apply -f -

# Check injection
kubectl describe pod my-pod
# Look for istio-proxy container
```

### Sidecar Configuration

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: app-sidecar
spec:
  workloadSelector:
    labels:
      app: my-app
  
  # Ingress listeners
  ingress:
  - port:
      number: 8080
      protocol: HTTP
      name: http
    defaultEndpoint: 127.0.0.1:3000    # Where traffic goes
  
  # Egress listeners
  egress:
  - hosts:
    - "./*"                             # Same namespace
    - "istio-system/*"                  # Other namespaces
    port:
      number: 15001
      protocol: TCP
  
  # Outbound traffic policy
  outboundTrafficPolicy:
    mode: ALLOW_ANY                     # ALLOW_ANY or REGISTRY_ONLY
```

---

# mTLS in Service Mesh

**mTLS (mutual TLS)** encrypts and authenticates all communication between services.

### mTLS Flow

```
Client                                Server
  │                                     │
  ├─ Request Certificate ───────────>  │
  │                                     │
  │  <─ Certificate ──────────────────  │
  │                                     │
  ├─ Verify Certificate ────────────>  │
  │                                     │
  ├─ Send Encrypted Data ────────────> │
  │                                     │
  │  <─ Encrypted Response ───────────  │
  │                                     │
  └─ Verify Response & Decrypt ────── │
```

### Istio mTLS Configuration

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT                # STRICT: mTLS required
                                 # PERMISSIVE: mTLS or plaintext
                                 # DISABLE: plaintext only

---
# Per-namespace mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: namespace-mtls
  namespace: production
spec:
  mtls:
    mode: STRICT

---
# Per-workload mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: api-mtls
spec:
  selector:
    matchLabels:
      app: api
  mtls:
    mode: STRICT
    
  # Specific ports can use different modes
  portLevelMtls:
    8090:
      mode: DISABLE             # Health check port
```

### Authorization Policies

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-api
spec:
  selector:
    matchLabels:
      app: api
  
  # Allow rules
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/default/sa/frontend"
    to:
    - operation:
        methods: ["GET"]
        paths: ["/health", "/api/v1/*"]
  
  # Deny rules (evaluated first)
  - to:
    - operation:
        ports: ["admin"]
      name: "admin-port"
    when:
    - key: source.ip
      values:
      - "192.168.1.0/24"
    deny: [{}]
```

---

# Traffic Management

## Canary Deployments

```yaml
---
# Deployment with version label
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
      version: v1
  template:
    metadata:
      labels:
        app: api
        version: v1
    spec:
      containers:
      - name: api
        image: api:v1.0.0

---
# New version (canary)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
      version: v2
  template:
    metadata:
      labels:
        app: api
        version: v2
    spec:
      containers:
      - name: api
        image: api:v2.0.0

---
# Service routes to both versions
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080

---
# Istio VirtualService: 90% v1, 10% v2 (canary)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
spec:
  hosts:
  - api
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: api
        subset: v1
      weight: 90
    - destination:
        host: api
        subset: v2
      weight: 10

---
# DestinationRule: Define subsets
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api
spec:
  host: api
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

## Circuit Breaker

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-circuit-breaker
spec:
  host: api
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutive5xxErrors: 5      # Eject after 5 errors
      interval: 30s                 # Check interval
      baseEjectionTime: 30s         # Minimum ejection time
      maxEjectionPercent: 50        # Max % of hosts to eject
      minRequestVolume: 5           # Minimum requests to analyze
      splitExternalLocalOriginErrors: true
```

## Retry Policy

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-with-retries
spec:
  hosts:
  - api
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: api
        port:
          number: 8080
    
    # Retry configuration
    retries:
      attempts: 3                   # Number of retries
      perTryTimeout: 10s            # Timeout per attempt
      retryOn: "5xx,reset,connection-failure"
```

---

# Observability in Service Mesh

## Metrics

Service mesh provides automatic metrics collection.

### Prometheus Integration

```bash
# Service mesh exposes metrics
kubectl port-forward -n istio-system svc/prometheus 9090:9090

# Access Prometheus
# http://localhost:9090

# Query examples:
# istio_request_total
# istio_request_duration_milliseconds_bucket
# istio_requests_total{destination_service="api"}
```

### Grafana Dashboards

```bash
# Prometheus data source configured
# Port forward Grafana
kubectl port-forward -n istio-system svc/grafana 3000:3000

# Browse pre-made dashboards
# - Service Mesh Performance
# - Service Mesh
# - Istio Workload Dashboard
```

## Distributed Tracing

### Jaeger Integration

```bash
# Jaeger deployed with Istio
kubectl port-forward -n istio-system svc/jaeger-query 16686:16686

# View traces
# http://localhost:16686

# Trace shows request path through services
# with latency at each hop
```

### Adding Trace Headers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: traced-app
spec:
  containers:
  - name: app
    image: myapp:latest
    # Application passes trace headers
    # X-Trace-ID
    # X-Span-ID
    # X-Parent-Span-ID
```

## Access Logs

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: access-logs
spec:
  accessLogging:
  - providers:
    - name: envoy
    state: ON
    match:
      ALL_REQUESTS: {}

---
# Check access logs in sidecar
# kubectl logs <pod> -c istio-proxy
```

### Access Log Format

```
[2024-01-15T10:30:00.123Z] 10.0.0.50:54321 api-v1:8080
GET /api/users HTTP/1.1 200 OK 45 123ms 456ms
```

---

## Example: Complete Observability Stack

```yaml
---
# Telemetry configuration
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: complete-observability
spec:
  # Metrics
  metrics:
  - providers:
    - name: prometheus
    state: ON
    overrides:
    - match:
        ALL_REQUESTS: {}
      metrics:
      - REQUEST_PATH
      - REQUEST_HOST
      - RESPONSE_CODE
      - RESPONSE_TIME
  
  # Tracing
  tracing:
  - providers:
    - name: jaeger
    randomSamplingPercentage: 100.0
    useRequestIdForTraceSampling: true
  
  # Access logs
  accessLogging:
  - providers:
    - name: envoy
    state: ON

---
# Custom metric labels
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: custom-metrics
spec:
  metrics:
  - providers:
    - name: prometheus
    dimensions:
    - REQUEST_HEADER_USER_ID
    - DESTINATION_CLUSTER
    - SOURCE_NAMESPACE
```

---

## Best Practices

### Custom Resource Best Practices

- ✅ Define clear schema with validation
- ✅ Include meaningful status fields
- ✅ Use appropriate scopes (Namespaced vs Cluster)
- ✅ Version your CRDs for evolution
- ✅ Document via OpenAPI schema
- ✅ Implement proper validation rules
- ❌ Don't make schema too permissive
- ❌ Don't break backward compatibility

### Operator Best Practices

- ✅ Implement idempotent operations
- ✅ Use status conditions for state
- ✅ Handle errors gracefully
- ✅ Implement proper RBAC
- ✅ Add comprehensive logging
- ✅ Use finalizers for cleanup
- ✅ Test operator upgrades
- ❌ Don't block reconciliation
- ❌ Don't assume external service availability

### Service Mesh Best Practices

- ✅ Start with PERMISSIVE mTLS mode
- ✅ Gradually migrate to STRICT
- ✅ Use canary deployments for mesh rollouts
- ✅ Monitor mesh metrics continuously
- ✅ Implement network policies alongside mesh
- ✅ Use observability for troubleshooting
- ✅ Plan for mesh performance overhead
- ✅ Test failover scenarios
- ❌ Don't enable mesh for all namespaces immediately
- ❌ Don't ignore resource consumption
- ❌ Don't substitute mesh for proper application design

### Production Deployment Example

```yaml
---
# Namespace with controlled mesh injection
apiVersion: v1
kind: Namespace
metadata:
  name: production-apps
  labels:
    istio-injection: enabled
    pod-security.kubernetes.io/enforce: restricted

---
# Strict mTLS throughout namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: strict-mtls
  namespace: production-apps
spec:
  mtls:
    mode: STRICT
  portLevelMtls:
    8090:
      mode: PERMISSIVE             # Health check port

---
# Network Policy supporting mesh
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-network-policy
  namespace: production-apps
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector: {}
    - namespaceSelector: {}

---
# Application with complete observability
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: production-apps
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
      version: v1
  template:
    metadata:
      labels:
        app: api
        version: v1
      annotations:
        sidecar.istio.io/inject: "true"
        prometheus.io/scrape: "true"
    spec:
      # Service account for mesh
      serviceAccountName: api-sa
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: api
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: api
        image: api:v1.0.0
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 8081
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        
        # Resource management
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        
        # Security
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
        
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      
      volumes:
      - name: tmp
        emptyDir: {}

---
# VirtualService with full traffic management
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
  namespace: production-apps
spec:
  hosts:
  - api.production-apps.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: /api
    route:
    - destination:
        host: api.production-apps.svc.cluster.local
        port:
          number: 8080
    timeout: 30s
    retries:
      attempts: 3
      perTryTimeout: 10s
      retryOn: 5xx,reset,connection-failure

---
# DestinationRule with circuit breaking
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api
  namespace: production-apps
spec:
  host: api.production-apps.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 50
```

---

## Summary

**Custom Resources & Operators**:
- CRDs extend Kubernetes with domain-specific resources
- Operators automate management of custom resources
- Reconciliation loop maintains desired state
- Enables infrastructure as code patterns

**Service Mesh**:
- Centralized management of service-to-service communication
- Sidecar proxies intercept and manage traffic
- mTLS provides encrypted and authenticated communication
- Istio and Linkerd are leading implementations
- Observability provides metrics, traces, and logs
- Enables advanced traffic management (canary, circuit breaker, retries)

Custom resources and service meshes enable enterprise-grade Kubernetes deployments with advanced traffic management and observability!
