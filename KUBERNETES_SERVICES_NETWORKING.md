# Kubernetes Services & Networking

## Table of Contents
1. [Overview](#overview)
2. [Service Types](#service-types)
3. [Service Discovery & DNS](#service-discovery--dns)
4. [Network Policies](#network-policies)
5. [CNI Plugins](#cni-plugins)
6. [Comparison & Decision Matrix](#comparison--decision-matrix)

---

## Overview

**Services** in Kubernetes provide stable network endpoints for accessing Pods. Since Pods are ephemeral (created and destroyed), Services abstract the underlying Pod IPs and provide:

- **Stable DNS names**: Pods can be discovered reliably
- **Load balancing**: Distribute traffic across multiple Pods
- **Service discovery**: Apps discover each other automatically
- **Network abstraction**: Decouple client from Pod locations

### Core Service Concepts

| Concept | Purpose |
|---------|---------|
| **Service** | Stable endpoint to access Pods |
| **Selector** | Identifies which Pods receive traffic |
| **Port** | Service port (external or within cluster) |
| **TargetPort** | Pod port receiving actual traffic |
| **Endpoint** | Actual Pod IP:Port this Service points to |
| **Session Affinity** | Sticky sessions (client → same Pod) |

### Service Architecture

```
┌─────────────────────────────────────────┐
│         External Client / Pod            │
│     Requests to Service IP:Port          │
└────────────────┬────────────────────────┘
                 │
                 ▼
        ┌────────────────────┐
        │     Service        │
        │  (Stable IP:Port)  │
        └─────────┬──────────┘
                  │ Load Balance
         ┌────────┴─────────┬─────────────┐
         │                  │             │
    ┌────▼──────┐    ┌─────▼──────┐  ┌──▼──────────┐
    │   Pod 1    │    │   Pod 2    │  │   Pod 3    │
    │ 10.0.0.5   │    │ 10.0.0.6   │  │ 10.0.0.7   │
    └────────────┘    └────────────┘  └────────────┘
```

---

## Service Types

### 1. ClusterIP (Default)

**ClusterIP** exposes the Service on an internal cluster IP. The Service is only accessible from within the cluster.

#### Characteristics
- **Scope**: Internal cluster only
- **IP Type**: Virtual cluster IP (not routable externally)
- **Use Case**: Internal communication between pods/services
- **Default**: If no type specified, ClusterIP is used

#### Basic Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP              # Optional (default)
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80                   # Service port
    targetPort: 80             # Pod port
```

#### DNS Resolution

```bash
# Within cluster, access via DNS
# Format: <service-name>.<namespace>.svc.cluster.local

nginx-service               # Within same namespace
nginx-service.default       # With namespace
nginx-service.default.svc.cluster.local  # Full FQDN
```

#### YAML with Multiple Ports

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: metrics
    port: 9090
    targetPort: 9090
```

#### Accessing from Pod

```bash
# From within a Pod in the cluster:
curl http://nginx-service:80
curl http://nginx-service.default.svc.cluster.local:80

# Environment variables are also injected
# Access via: $NGINX_SERVICE_SERVICE_HOST:$NGINX_SERVICE_SERVICE_PORT
```

#### Session Affinity (Sticky Sessions)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  sessionAffinity: ClientIP      # Stick client to same Pod
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800      # 3 hours
```

---

### 2. NodePort

**NodePort** exposes the Service on a static port on each node. Externally accessible via `NodeIP:NodePort`.

#### Characteristics
- **Scope**: Accessible from outside cluster
- **Port Range**: 30000-32767 (default)
- **IP Type**: Every node gets same port
- **Use Case**: External access without load balancer
- **Limitations**: Limited scalability (can't use port 80, 443 directly)

#### Basic Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80           # Service port (within cluster)
    targetPort: 8080   # Pod port
    nodePort: 30080    # Node port (optional, auto-assigned if omitted)
```

#### How NodePort Works

```
External Client: 192.168.1.100
         │
         ▼
    Node 1: 192.168.1.10:30080
         │
         ├──► Pod 1 (10.0.0.5:8080) ✓
         │
    Node 2: 192.168.1.11:30080
         │
         └──► Pod 2 (10.0.0.6:8080) ✓
```

#### Access Method

```bash
# From external machine
curl http://<node-ip>:<nodePort>
curl http://192.168.1.10:30080

# Find node IP and port
kubectl get nodes -o wide
kubectl get svc web-service -o wide
```

#### Multiple Ports with NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-nodeport
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
    nodePort: 30080
  - name: https
    port: 443
    targetPort: 8443
    nodePort: 30443
```

#### Your Deployment Example with NodePort

From your `deployment.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: serv-taruni
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - nodePort: 32428        # External access on all nodes
    protocol: TCP
    port: 80               # Internal service port
    targetPort: 80         # Pod container port
```

**Access:**
```bash
http://<any-node-ip>:32428
```

---

### 3. LoadBalancer

**LoadBalancer** exposes the Service externally using a cloud provider's load balancer (AWS ELB, Google Cloud LB, Azure LB, etc.).

#### Characteristics
- **Scope**: Accessible from outside cluster with external IP
- **Cloud Integration**: Requires cloud provider integration
- **External IP**: Gets real external IP (not node IP)
- **Use Case**: Production external access
- **Provisioning**: Automatically provisions cloud LB

#### Basic Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lb-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
```

#### How LoadBalancer Works

```
Internet Client: 203.0.113.123
         │
         ▼
    Cloud Load Balancer (203.0.113.50:80)
         │
    ┌────┴────┬────────┐
    │          │        │
    ▼          ▼        ▼
  Node 1    Node 2    Node 3
    │          │        │
    └──────┬───┴────┬───┘
           │        │
        Pod 1    Pod 2
```

#### Accessing LoadBalancer Service

```bash
# Wait for external IP to be assigned
kubectl get svc lb-service -w

# Once EXTERNAL-IP is assigned (e.g., 203.0.113.50)
curl http://203.0.113.50:80

# Check status
kubectl describe svc lb-service
```

#### Cloud-Specific Annotations

```yaml
apiVersion: v1
kind: Service
metadata:
  name: aws-lb-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # Network Load Balancer
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 443
    targetPort: 8443
    protocol: TCP
```

#### LoadBalancer with Multiple Ports

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-protocol-lb
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: grpc
    port: 50051
    targetPort: 50051
```

---

### 4. ExternalName

**ExternalName** maps a Service to an external DNS name (outside cluster). No endpoints or load balancing involved.

#### Characteristics
- **Scope**: Maps to external service
- **No Load Balancing**: Direct CNAME mapping
- **Use Case**: Access external APIs/services via DNS
- **DNS Type**: CNAME record

#### Use Cases
- Access external databases (managed RDS, Cloud SQL)
- Integrate legacy systems
- Reference external APIs
- Multi-cluster communication

#### Basic Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: my-database.example.com
  ports:
  - port: 3306
```

#### How ExternalName Works

```
┌─────────────────────────┐
│    Pod in Cluster       │
│ Requests to:            │
│ external-db:3306        │
└────────────┬────────────┘
             │
      CNAME Resolution
             │
             ▼
   my-database.example.com
        (External)
```

#### Accessing ExternalName Service

```bash
# From within Pod
mysql -h external-db -P 3306 -u user -p

# Pods automatically get DNS names in environment
# EXTERNAL_DB_SERVICE_HOST=my-database.example.com
# EXTERNAL_DB_SERVICE_PORT=3306
```

#### MultiCluster ExternalName

```yaml
# In Cluster A, reference service in Cluster B
apiVersion: v1
kind: Service
metadata:
  name: remote-api
spec:
  type: ExternalName
  externalName: api-service.default.svc.cluster2.local
  ports:
  - port: 8080
    targetPort: 8080
```

---

## Service Discovery & DNS

### How DNS Works in Kubernetes

Kubernetes cluster runs **CoreDNS** (formerly kube-dns) which provides DNS resolution for Services.

#### DNS Names

Every Service gets automatic DNS names:

```
Short name (same namespace):
<service-name>

With namespace:
<service-name>.<namespace>

Fully qualified:
<service-name>.<namespace>.svc.cluster.local

Pod DNS (StatefulSet):
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

#### DNS Resolution Examples

```bash
# Namespace: default, Service: nginx-service

# Within same namespace:
nginx-service
nginx-service:80

# From different namespace (monitoring):
nginx-service.default
nginx-service.default.svc.cluster.local

# StatefulSet Pod DNS (my-db-0 in my-db StatefulSet):
my-db-0.my-db-service.default.svc.cluster.local
```

### Service Environment Variables

When a Pod is created, Kubernetes injects Service information as environment variables:

```bash
# For service: nginx-service

NGINX_SERVICE_SERVICE_HOST=10.0.0.100
NGINX_SERVICE_SERVICE_PORT=80
NGINX_SERVICE_SERVICE_PORT_HTTP=80
```

#### Accessing via Environment Variables

```bash
# In Pod, access service via injected variables
curl http://$NGINX_SERVICE_SERVICE_HOST:$NGINX_SERVICE_SERVICE_PORT
```

### DNS A Records

```bash
# Service DNS resolves to Cluster IP
nslookup nginx-service.default.svc.cluster.local

# Output:
# Name: nginx-service.default.svc.cluster.local
# Address: 10.0.0.100

# Headless Service (ClusterIP: None) resolves to Pod IPs
nslookup my-db.my-db-service.default.svc.cluster.local

# Output: Multiple A records (one per Pod)
# Address: 10.0.0.5
# Address: 10.0.0.6
# Address: 10.0.0.7
```

### Troubleshooting DNS

```bash
# From within Pod, test DNS resolution
kubectl exec -it <pod-name> -- /bin/sh
nslookup nginx-service
nslookup nginx-service.default
nslookup nginx-service.default.svc.cluster.local

# Check DNS server
cat /etc/resolv.conf
# nameserver 10.96.0.10  (CoreDNS service IP)

# Query specific service attributes
dig nginx-service.default.svc.cluster.local
dig SRV nginx-service.default.svc.cluster.local

# Check DNS pod logs
kubectl logs -n kube-system -l k8s-app=kube-dns
```

#### DNS Entry Format

```
# Service DNS:
<service>.<namespace>.svc.cluster.local A <cluster-ip>

# Pod DNS (StatefulSet):
<pod>.<service>.<namespace>.svc.cluster.local A <pod-ip>

# SRV Records (headless service):
_<service>._tcp.<namespace>.svc.cluster.local SRV <pod>.<service>.<namespace>.svc.cluster.local
```

---

## Network Policies

**Network Policies** define ingress and egress traffic rules for Pods. They act as firewall rules for Pod-to-Pod communication.

### Key Concepts

- **Ingress**: Incoming traffic TO Pods
- **Egress**: Outgoing traffic FROM Pods
- **Pod Selector**: Which Pods the policy applies to
- **Network Isolation**: By default, all traffic allowed (if no policy exists)
- **Additive Rules**: Multiple policies can apply to same Pod

### Basic Network Policy Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-http
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 80
```

### Example 1: Allow Ingress from Specific Pod

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
spec:
  # This policy applies to backend pods
  podSelector:
    matchLabels:
      app: backend
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    # Allow traffic from frontend pods
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Effect:**
- Pods with label `app: backend` only receive traffic from `app: frontend`
- All other traffic to backend is blocked
- Pods without this label are isolated

### Example 2: Allow Multiple Sources

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-multiple-sources
spec:
  podSelector:
    matchLabels:
      app: database
  
  policyTypes:
  - Ingress
  
  ingress:
  # Allow from backend in same namespace
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
  
  # Allow from monitoring in different namespace
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 5432
```

### Example 3: Deny All (Explicit Whitelist)

```yaml
# Deny all ingress traffic (default deny)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  # ingress: [] means no traffic allowed
```

Then allow specific traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
```

### Example 4: Egress Control (Allow DNS and HTTP only)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-limited
spec:
  podSelector:
    matchLabels:
      app: frontend
  
  policyTypes:
  - Egress
  
  egress:
  # Allow DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
  
  # Allow HTTP to backend
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 8080
  
  # Allow HTTPS to external APIs
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 443
```

### Example 5: Namespace Isolation

```yaml
# Allow traffic only within same namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
```

### Network Policy Operators

```yaml
ingress:
- from:
  - podSelector:
      matchExpressions:
      # In: pod label value must be in list
      - key: app
        operator: In
        values: ["frontend", "web"]
      
      # NotIn: pod label value must NOT be in list
      - key: tier
        operator: NotIn
        values: ["database"]
      
      # Exists: key must exist
      - key: version
        operator: Exists
      
      # DoesNotExist: key must NOT exist
      - key: deprecated
        operator: DoesNotExist
```

### Network Policy Limitations

```bash
# NetworkPolicy limitations:
# ❌ Cannot target by IP addresses directly
# ✅ But can target by Pod/Namespace selectors
# ❌ Cannot bind to specific port range directly
# ✅ But can specify multiple port rules
# ❌ No egress capability (must use CNI that supports it)
```

### Checking Network Policies

```bash
# List all NetworkPolicies
kubectl get networkpolicies
kubectl get netpol

# Describe a NetworkPolicy
kubectl describe networkpolicy/allow-http

# Check which pods are affected
kubectl get pods --show-labels

# Test connectivity
kubectl exec -it pod-a -- nc -vz pod-b-service 8080
```

---

## CNI Plugins

**CNI (Container Network Interface)** plugins manage Pod networking. They assign IP addresses, manage routing, and implement network policies.

### What CNI Does

- **IP Assignment**: Allocate unique IP to each Pod
- **Routing**: Create network routes between Pods and nodes
- **Network Policies**: Enforce traffic rules
- **Overlay Networks**: Virtual network on top of physical
- **Network Monitoring**: Packet capture and logging

### CNI Plugin Architecture

```
┌──────────────────────────────────────┐
│      kubelet (on each node)          │
└────────────┬─────────────────────────┘
             │ Calls CNI plugin
             ▼
      ┌────────────────┐
      │  CNI Plugin    │
      │ (e.g., Calico) │
      └────────┬───────┘
               │
        ┌──────┴────────────┐
        │                   │
   IP Allocation      Route Configuration
```

---

### 1. Calico

**Calico** provides networking and network policy features for Kubernetes using pure IP routing.

#### Characteristics
- **Approach**: Pure IP routing (no overlay)
- **Network Model**: BGP routing protocol
- **Performance**: Low latency, high throughput
- **Scaling**: Good for large clusters
- **Policy Support**: Full eBPF-based network policies
- **Encryption**: Optional WireGuard encryption

#### Installation

```bash
# Install Calico CNI plugin
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/master/manifests/tigera-operator.yaml
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/master/manifests/custom-resources.yaml

# Check Calico pods
kubectl get pods -n calico-system
```

#### Calico Network Policy

```yaml
apiVersion: crd.projectcalico.org/v1
kind: NetworkPolicy
metadata:
  name: calico-policy
  namespace: production
spec:
  selector: app == 'web'
  order: 100
  ingress:
  - action: Allow
    protocol: TCP
    destination:
      ports:
      - 80
      - 443
      selector: app == 'frontend'
  egress:
  - action: Allow
    destination:
      selector: app == 'backend'
```

#### Configuration

```yaml
# Calico ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: calico-config
  namespace: calico-system
data:
  # Enable IP-in-IP encapsulation
  calico_backend: bird
  
  # BGP as number
  AS_NUMBER: "64512"
  
  # Node IP automatically detected
  NODENAME: ""
```

#### Benefits
- ✅ High performance (no overlay encapsulation overhead)
- ✅ Fine-grained network policies
- ✅ Layer 3 routing (BGP)
- ✅ Good for on-premises/hybrid clouds

#### Drawbacks
- ❌ Requires BGP setup (more complex)
- ❌ Not ideal for cloud environments with security groups

---

### 2. Flannel

**Flannel** provides a simple layer 3 network fabric. Uses VXLAN or host-gw for tunneling.

#### Characteristics
- **Approach**: Simple overlay network
- **Tunneling**: VXLAN or host-gw
- **Performance**: Moderate (overhead from tunneling)
- **Policy Support**: No native network policies
- **Simplicity**: Easy to deploy and manage
- **Use Case**: Small to medium clusters

#### Installation

```bash
# Install Flannel
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# Check Flannel pods
kubectl get pods -n kube-flannel
```

#### Flannel Configuration

```yaml
# Flannel ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    }
  
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

#### Backend Options

| Backend | Method | Performance | Encryption |
|---------|--------|-------------|-----------|
| **VXLAN** | Software tunneling | Moderate | Optional |
| **host-gw** | Layer 2 direct | High | No |
| **UDP** | UDP tunneling | Slow | No |
| **IPIP** | IP-in-IP tunneling | Moderate | No |

#### Benefits
- ✅ Simple and lightweight
- ✅ Works easily across cloud providers
- ✅ Low memory footprint
- ✅ Mature and stable

#### Drawbacks
- ❌ No built-in network policies
- ❌ Performance overhead from tunneling
- ❌ Less feature-rich than Calico

---

### 3. Cilium

**Cilium** is a modern eBPF-based networking and security plugin. Advanced features on top of BPF programs.

#### Characteristics
- **Approach**: eBPF (extended Berkeley Packet Filter)
- **Performance**: Very high (kernel-level)
- **Intelligence**: Application-aware policies
- **Observability**: Advanced networking insights
- **Security**: Micro-segmentation
- **Encryption**: Built-in encryption

#### Installation

```bash
# Install Cilium CLI
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin

# Install Cilium
cilium install

# Check Cilium status
cilium status
```

#### Cilium Network Policy (Namespace scoped)

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "app-policy"
spec:
  description: "L7 policy for app"
  selector:
    matchLabels:
      app: web
  
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: GET
          path: "^/api/.*"
  
  egress:
  - toEndpoints:
    - matchLabels:
        app: backend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
```

#### L7 Policy Example (Application-aware)

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "l7-policy"
spec:
  selector:
    matchLabels:
      app: api
  
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: client
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/health"
        - method: "POST"
          path: "/submit"
          headerMatches:
          - name: "X-API-Token"
            value: "token123"
```

#### Hubble (Cilium's Observability)

```bash
# Enable Hubble for observability
cilium hubble enable

# View real-time traffic flows
hubble observe

# Get flow logs
hubble flows get

# Monitor specific pod
hubble observe -n default -p pod/web-deployment-xxxxx
```

#### Benefits
- ✅ Highest performance (eBPF kernel-level)
- ✅ L7 (Application layer) policies
- ✅ Excellent observability (Hubble)
- ✅ Service mesh ready
- ✅ Advanced security features

#### Drawbacks
- ❌ More complex to configure
- ❌ Steeper learning curve
- ❌ Requires kernel 4.9+ with BPF support
- ❌ More resource intensive

---

## Comparison & Decision Matrix

### CNI Plugin Comparison

| Feature | Calico | Flannel | Cilium |
|---------|--------|---------|--------|
| **Setup Complexity** | Medium | Easy | Hard |
| **Performance** | Very High | Moderate | Very High |
| **Network Policies** | Yes (native) | No | Yes (L7) |
| **Encryption** | Yes (WireGuard) | Optional | Yes (built-in) |
| **Cluster Size** | Large | Small-Medium | Medium-Large |
| **Observability** | Basic | Basic | Advanced (Hubble) |
| **Learning Curve** | Medium | Easy | Steep |
| **Support** | Excellent | Good | Excellent |
| **Scaling** | ✅ ✅ ✅ | ✅ ✅ | ✅ ✅ ✅ |
| **Cost** | Free | Free | Free |

### Service Type Decision Tree

```
START
  │
  ├─ Need internal access only?
  │   ├─ YES → ClusterIP ✓
  │   └─ NO → Continue
  │
  ├─ Need external access without LB?
  │   ├─ YES → NodePort ✓
  │   └─ NO → Continue
  │
  ├─ Have cloud provider (AWS/GCP/Azure)?
  │   ├─ YES → LoadBalancer ✓
  │   └─ NO → Continue
  │
  └─ Need to map to external service/DNS?
      ├─ YES → ExternalName ✓
      └─ NO → Reconsider requirements
```

### CNI Plugin Selection

```
START
  │
  ├─ Need highest performance (bare metal/on-prem)?
  │   ├─ YES with BGP knowledge → Calico ✓
  │   └─ YES without BGP → Continue
  │
  ├─ Need simple setup (cloud/dev)?
  │   ├─ YES → Flannel ✓
  │   └─ NO → Continue
  │
  ├─ Need advanced policies & observability?
  │   ├─ YES, and kernel supports eBPF → Cilium ✓
  │   └─ YES, kernel doesn't support eBPF → Continue
  │
  └─ Need network policies with routing?
      ├─ YES → Calico ✓
      └─ Reconsider
```

---

## Practical Examples

### Example 1: Web Tier with Internal Database

```yaml
---
# Database Service (ClusterIP - internal only)
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432

---
# Backend Service (ClusterIP - internal only)
apiVersion: v1
kind: Service
metadata:
  name: backend-api
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080

---
# Frontend Service (LoadBalancer - external)
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000

---
# Network Policy: Allow backend to DB only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-access
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432

---
# Network Policy: Allow frontend to backend only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-access
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Example 2: Multi-Tier with External Database

```yaml
---
# External managed database (RDS/Cloud SQL)
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: mydb.c5oe1axfx1234.us-west-2.rds.amazonaws.com
  ports:
  - port: 3306

---
# Access from application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: myapp:latest
        env:
        - name: DB_HOST
          value: external-db
        - name: DB_PORT
          value: "3306"
        ports:
        - containerPort: 8080

---
# Frontend service
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

### Example 3: Microservices with Istio

```yaml
# Using Cilium for advanced L7 policies
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: microservices-policy
spec:
  # Allow Frontend -> Backend
  - selector:
      matchLabels:
        app: frontend
    ingress:
    - fromEndpoints:
      - matchLabels:
          app: client
      toPorts:
      - ports:
        - port: "3000"
  
  # Allow Backend -> Payment Service
  - selector:
      matchLabels:
        app: payment
    ingress:
    - fromEndpoints:
      - matchLabels:
          app: backend
      toPorts:
      - ports:
        - port: "8080"
      rules:
        http:
        - method: POST
          path: "/pay"
  
  # Allow Backend -> Inventory Service
  - selector:
      matchLabels:
        app: inventory
    ingress:
    - fromEndpoints:
      - matchLabels:
          app: backend
      toPorts:
      - ports:
        - port: "8081"
```

---

## Best Practices

### Services
- ✅ Use ClusterIP for internal services
- ✅ Avoid NodePort in production (use LoadBalancer)
- ✅ Use ExternalName for external integrations
- ✅ Set sessionAffinity if needed
- ✅ Implement appropriate health checks
- ❌ Don't expose all services externally

### Network Policies
- ✅ Start with explicit deny-all, then whitelist
- ✅ Use namespace labels for organization
- ✅ Test policies before applying in production
- ✅ Document your network architecture
- ✅ Monitor with network observability tools
- ❌ Don't rely only on network policies for security

### CNI Selection
- ✅ Calico for large/on-prem clusters with BGP
- ✅ Flannel for simple/dev environments
- ✅ Cilium for advanced observability/policies
- ✅ Consider performance requirements
- ✅ Plan for future scaling
- ❌ Don't change CNI after cluster creation (very difficult)

---

## Troubleshooting

```bash
# Check Service endpoints
kubectl get endpoints <service-name>

# Test DNS from pod
kubectl exec -it <pod> -- nslookup <service-name>

# Check Service status
kubectl describe service <service-name>

# See traffic flow
kubectl logs <service-pod> -c service

# Test connectivity between pods
kubectl exec -it <pod-a> -- nc -vz <pod-b-ip> <port>

# Check network policies applied
kubectl get networkpolicies
kubectl describe networkpolicy <policy-name>

# Check CNI plugin status
kubectl get daemonset -n kube-system

# View pod IP assignment
kubectl get pods -o wide
```

---

## Summary

- **Services**: Create stable network endpoints abstracting Pod ephemeralness
- **ClusterIP**: Internal cluster communication
- **NodePort**: External access via node ports (limited use)
- **LoadBalancer**: Production external access with cloud LB
- **ExternalName**: Map to external DNS names
- **DNS**: Built-in service discovery via CoreDNS
- **Network Policies**: Firewall rules for Pod-to-Pod traffic
- **CNI Plugins**: Choose Calico (performance), Flannel (simplicity), or Cilium (advanced)

Understanding Services and Networking is crucial for building scalable, secure Kubernetes applications!
