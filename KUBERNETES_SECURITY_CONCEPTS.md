# Kubernetes Security Concepts

## Table of Contents
1. [Overview](#overview)
2. [RBAC (Role-Based Access Control)](#rbac-role-based-access-control)
3. [Service Accounts](#service-accounts)
4. [Network Policies](#network-policies)
5. [Security Context & PodSecurity Admission](#security-context--podsecurity-admission)
6. [Image Scanning Basics](#image-scanning-basics)
7. [Security Best Practices](#security-best-practices)

---

## Overview

**Kubernetes Security** addresses multiple layers:
- **Authentication**: Who are you? (User/Service Account)
- **Authorization**: What can you do? (RBAC)
- **Admission Control**: Is this allowed? (Policies)
- **Container Security**: How does the container run? (Security Context)
- **Network Security**: What traffic is allowed? (Network Policies)
- **Image Security**: Is the container safe? (Scanning)

### Security Layers

```
┌─────────────────────────────────────────────┐
│         External Access                      │
│  (User requests to API Server)              │
└────────────────┬────────────────────────────┘
                 │
                 ▼
        ┌──────────────────┐
        │ Authentication   │ ← Who are you?
        │ (Certificates,   │   
        │  Tokens)         │
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │ RBAC             │ ← What can you do?
        │ (Roles,          │
        │  Permissions)    │
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │ Admission        │ ← Is this allowed?
        │ Controllers      │   (PodSecurity,
        │ (Policies)       │    NetworkPolicy)
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │ Container        │ ← How runs?
        │ Security         │   (User, Privileges,
        │ Context          │    Read-only FS)
        └──────────────────┘
```

---

## RBAC (Role-Based Access Control)

**RBAC** determines what authenticated users/service accounts can do in the cluster.

### RBAC Components

| Component | Scope | Purpose |
|-----------|-------|---------|
| **Role** | Namespace | Permissions in single namespace |
| **RoleBinding** | Namespace | Bind Role to user/SA in namespace |
| **ClusterRole** | Cluster | Permissions across all namespaces |
| **ClusterRoleBinding** | Cluster | Bind ClusterRole to user/SA globally |

### RBAC Decision Flow

```
User/ServiceAccount makes request
        │
        ▼
Find matching RoleBinding/ClusterRoleBinding
        │
        ▼
Get referenced Role/ClusterRole
        │
        ▼
Check if request verb/resource in rules
        │
        ├─ YES → Allow request
        └─ NO → Deny request (403 Forbidden)
```

### Role (Namespace-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
# Rule 1: Can read pods
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

# Rule 2: Can read pod logs
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]

# Rule 3: Can read pods in specific namespace (resourceNames)
- apiGroups: [""]
  resources: ["pods"]
  resourceNames: ["my-pod"]  # Only specific pod
  verbs: ["get"]
```

### RoleBinding (Namespace-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
roleRef:
  # Bind to Role (in same namespace)
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
# User
- kind: User
  name: alice@example.com
  apiGroup: rbac.authorization.k8s.io

# Group
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io

# ServiceAccount
- kind: ServiceAccount
  name: myapp-sa
  namespace: default
```

### ClusterRole (Cluster-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: admin-role
rules:
# All resources, all verbs
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# Or more specific
- apiGroups: [""]
  resources: ["pods", "pods/logs", "pods/exec"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Non-resource URLs
- nonResourceURLs: ["/metrics", "/api", "/logs"]
  verbs: ["get"]
```

### ClusterRoleBinding (Cluster-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: global-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin-role
subjects:
- kind: User
  name: admin@example.com
  apiGroup: rbac.authorization.k8s.io
```

### RBAC Verbs

| Verb | Action |
|------|--------|
| **get** | Get single resource |
| **list** | List resources |
| **watch** | Watch for changes |
| **create** | Create new resource |
| **update** | Update existing resource |
| **patch** | Partial update |
| **delete** | Delete resource |
| **deletecollection** | Delete multiple resources |
| **exec** | Execute commands in Pod |
| **logs** | Read Pod logs |
| **port-forward** | Port forward to Pod |

### Common Resources

```yaml
# Core resources
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch"]

# Apps resources
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Batch resources
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch"]

# Storage resources
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses", "volumeattachments"]
  verbs: ["get", "list"]

# RBAC resources
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
  verbs: ["get", "list"]
```

### Practical Example: Developer Role

```yaml
---
# Role for developers
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
# View resources
- apiGroups: [""]
  resources: ["pods", "pods/logs", "services"]
  verbs: ["get", "list", "watch"]

# Deploy and manage
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Manage ConfigMaps and Secrets
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]

# Execute in pods (debugging)
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]

# View events
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]

---
# Bind developer role to developers group
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers
  namespace: development
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer
subjects:
- kind: Group
  name: developers@company.com
  apiGroup: rbac.authorization.k8s.io
```

### Checking RBAC Permissions

```bash
# Can current user perform action?
kubectl auth can-i get pods
# Output: yes/no

# Can specific user perform action?
kubectl auth can-i get pods --as=alice@example.com

# Can specific service account perform action?
kubectl auth can-i get pods --as=system:serviceaccount:default:myapp-sa

# List all permissions for user
kubectl auth can-i list all --as=alice@example.com

# Dry run to see if request would work
kubectl get pods --as=alice@example.com --dry-run=client
```

### Pre-built Roles

Kubernetes provides built-in ClusterRoles:

```bash
# View built-in roles
kubectl get clusterroles | grep system:

# Common built-in roles:
# - system:masters
# - system:kube-scheduler
# - system:kube-proxy
# - system:kube-controller-manager
# - view (read-only)
# - edit (modify permissions)
# - admin (full permissions within namespace)
# - cluster-admin (full cluster permissions)
```

**Using built-in edit role:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-edit
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit              # Built-in edit role
subjects:
- kind: User
  name: alice@example.com
  apiGroup: rbac.authorization.k8s.io
```

---

## Service Accounts

**Service Accounts** are Kubernetes identities for Pods and applications running inside the cluster.

### Service Account Concepts

```
┌─────────────────────────────────┐
│      Pod                        │
│  ├─ Mounted Token: /var/run/.. │
│  ├─ CA Certificate              │
│  └─ Namespace info              │
└────────────────┬────────────────┘
                 │
                 ▼
   Service Account (Identity)
                 │
        ┌────────┴──────────┐
        │                   │
    Roles          Secret (Token)
    Bindings       
```

### Creating Service Accounts

#### Method 1: Declarative

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default
```

#### Method 2: Imperative

```bash
kubectl create serviceaccount myapp-sa
```

### ServiceAccount Auto-mounting

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      serviceAccountName: myapp-sa  # Use specific SA
      
      # Auto-mount SA token (default: true)
      automountServiceAccountToken: true
      
      containers:
      - name: app
        image: myapp:latest
```

### Accessing Kubernetes API from Pod

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-reader
  namespace: default

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: api-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-reader
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: api-reader
subjects:
- kind: ServiceAccount
  name: api-reader
  namespace: default

---
apiVersion: v1
kind: Pod
metadata:
  name: api-client
spec:
  serviceAccountName: api-reader
  containers:
  - name: client
    image: curlimages/curl:latest
    command:
    - /bin/sh
    - -c
    - |
      # Token mounted at default path
      TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
      CA_CERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
      
      # Query Kubernetes API
      curl -H "Authorization: Bearer $TOKEN" \
           --cacert $CA_CERT \
           https://kubernetes.default.svc/api/v1/namespaces/$NAMESPACE/pods
```

### Service Account Token Projection

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default

---
apiVersion: v1
kind: Pod
metadata:
  name: projected-token-pod
spec:
  serviceAccountName: myapp-sa
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: sa-token
      mountPath: /var/run/secrets/tokens
  
  volumes:
  - name: sa-token
    projected:
      sources:
      - serviceAccountToken:
          audience: api
          expirationSeconds: 3600
          path: token
```

---

## Network Policies

**Network Policies** control traffic flow between Pods and to external endpoints.

(Note: Network Policies covered extensively in Services & Networking guide. This is a quick reference.)

### Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}          # Applies to all pods
  policyTypes:
  - Ingress
  ingress: []              # No traffic allowed
```

### Allow Specific Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Allow Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress
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
  
  # Allow backend
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
```

---

## Security Context & PodSecurity Admission

**Security Context** defines privilege/access control settings for Pods and containers.

### Security Context Levels

1. **Pod-level**: Applies to all containers
2. **Container-level**: Overrides pod-level settings

### Pod Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-pod
spec:
  # Pod-level security context
  securityContext:
    runAsUser: 1000           # Run as UID 1000
    runAsGroup: 3000          # Run as GID 3000
    fsGroup: 2000             # Volume ownership GID
    seLinuxOptions:
      level: "s0:c123,c456"
    windowsOptions:           # Windows-specific
      gmsaCredentialSpecName: "gmsa-pod-spec"
  
  containers:
  - name: app
    image: myapp:latest
    
    # Container-level security context (overrides pod-level)
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
    
    volumeMounts:
    - name: data
      mountPath: /data
  
  volumes:
  - name: data
    emptyDir: {}
```

### Common Security Context Settings

| Setting | Purpose |
|---------|---------|
| **runAsUser** | UID to run container |
| **runAsGroup** | GID for file access |
| **runAsNonRoot** | Enforce non-root |
| **fsGroup** | Volume ownership |
| **readOnlyRootFilesystem** | Immutable root FS |
| **allowPrivilegeEscalation** | Prevent privilege escalation |
| **capabilities** | Linux capabilities |
| **seccompProfile** | Seccomp policy |

### Read-Only Root Filesystem

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-fs-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      readOnlyRootFilesystem: true
    
    # Must mount writable volumes for dynamic data
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache
  
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

### Running as Non-Root

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nonroot-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
```

### Linux Capabilities

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: capabilities-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      capabilities:
        add:
        - NET_BIND_SERVICE    # Can bind to port < 1024
        - SYS_TIME            # Can set system time
        drop:
        - ALL                 # Drop all by default
```

### Seccomp Profile

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault    # Use kubelet defaults
  
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      seccompProfile:
        type: Localhost
        localhostProfile: my-profile.json
```

### PodSecurity Admission Controller

**PodSecurity** replaces PodSecurityPolicy and enforces security standards.

#### Pod Security Standards

| Level | Restrictions |
|-------|--------------|
| **restricted** | Strictest, all hardening options required |
| **baseline** | Minimal restrictions, prevents known vulnerabilities |
| **privileged** | No restrictions (default for system pods) |

#### Applying PodSecurity Policy

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-ns
  labels:
    # Enforce restricted policy
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    
    # Audit violations
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    
    # Warn on violations
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

#### Restricted Policy Requirements

```yaml
# Pod must meet all these requirements:
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
      runAsNonRoot: true
    
    # Must set resource limits
    resources:
      limits:
        memory: "256Mi"
        cpu: "500m"
      requests:
        memory: "128Mi"
        cpu: "100m"
```

#### Baseline Policy (Less Restrictive)

```yaml
# Baseline allows more flexibility
apiVersion: v1
kind: Pod
metadata:
  name: baseline-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    # Can have privilege escalation
    # Can run as root (but not recommended)
```

---

## Image Scanning Basics

**Image Scanning** identifies vulnerabilities in container images before deployment.

### Image Scanning Tools

| Tool | Purpose |
|------|---------|
| **Trivy** | Fast vulnerability scanner |
| **Anchore** | Supply chain security |
| **Snyk** | Developer-focused scanning |
| **Clair** | Open source registry |
| **Aqua** | Container security platform |

### Trivy Scanning

```bash
# Install Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh

# Scan local image
trivy image myapp:latest

# Scan with severity filter
trivy image --severity HIGH,CRITICAL myapp:latest

# Scan and export JSON
trivy image --format json --output results.json myapp:latest

# Scan with ignore list
trivy image --ignorefile .trivyignore myapp:latest
```

### Image Scanning in CI/CD Pipeline

```yaml
# GitHub Actions example
name: Image Scan
on:
  push:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Run Trivy scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: myapp:latest
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload to GitHub Security
      uses: github/codeql-action@v1
      with:
        sarif_file: 'trivy-results.sarif'
```

### Container Image Best Practices

```dockerfile
# Bad - Large, vulnerable image
FROM ubuntu:latest
RUN apt-get update && apt-get install -y \
    python3 \
    curl \
    wget \
    git
COPY app.py /app/
CMD ["python3", "/app/app.py"]

# Better - Minimal, hardened image
FROM python:3.11-slim
RUN useradd -m -u 1000 appuser
WORKDIR /app
COPY app.py .
RUN pip install --no-cache-dir -r requirements.txt
USER appuser
CMD ["python3", "app.py"]
```

### Admission Controller for Image Scanning

```yaml
# ImagePolicy admission controller
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/image-policy-webhook/kubeconfig.yaml
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 5
      defaultAllow: false
```

### Image Pull Policies

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: image-pull-policy-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    # Pull policies:
    # Always: Always pull (default for latest tag)
    # IfNotPresent: Pull only if not on node
    # Never: Never pull
    imagePullPolicy: Always
```

### Private Registry with Image Scanning

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-registry-pod
spec:
  # Kubernetes verifies image through scanning service
  imagePullSecrets:
  - name: registry-credentials
  
  containers:
  - name: app
    image: private-registry.com/myapp:v1.0.0
    securityContext:
      readOnlyRootFilesystem: true
      runAsNonRoot: true
```

---

## Security Best Practices

### RBAC
- ✅ Use least-privilege principle
- ✅ Create specific roles for different teams
- ✅ Use namespaces to isolate permissions
- ✅ Regularly audit role bindings
- ❌ Don't make everyone cluster-admin
- ❌ Don't use wildcard resources

### Service Accounts
- ✅ Use dedicated SA for each application
- ✅ Disable auto-mounting if not needed
- ✅ Use short-lived tokens
- ✅ Rotate tokens regularly
- ❌ Don't share service account tokens
- ❌ Don't mount default SA token

### Network Policies
- ✅ Start with deny-all policy
- ✅ Explicitly allow needed traffic
- ✅ Use label selectors for targeting
- ✅ Audit network policy rules
- ❌ Don't leave all traffic open
- ❌ Don't rely only on network policies

### Security Context
- ✅ Run containers as non-root
- ✅ Use read-only root filesystem
- ✅ Drop unnecessary capabilities
- ✅ Use seccomp profiles
- ❌ Don't run as root
- ❌ Don't allow privilege escalation

### Image Security
- ✅ Scan images before deployment
- ✅ Use minimal base images
- ✅ Keep base images updated
- ✅ Sign and verify images
- ✅ Use specific image tags (not latest)
- ❌ Don't use unscanned images
- ❌ Don't use untrusted registries

---

## Practical Security Example: Secure Application

```yaml
---
# Namespace with pod security policy
apiVersion: v1
kind: Namespace
metadata:
  name: secure-app
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# Service account for application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: secure-app

---
# Role with minimal permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-reader
  namespace: secure-app
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
  resourceNames: ["app-config"]

---
# Bind role to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-reader-binding
  namespace: secure-app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-reader
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: secure-app

---
# Network policy: allow from ingress only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-ingress-only
  namespace: secure-app
spec:
  podSelector:
    matchLabels:
      app: myapp
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
  # Allow DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53

---
# Secure deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: secure-app
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
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      
      # Service account
      serviceAccountName: app-sa
      automountServiceAccountToken: true
      
      # Pod anti-affinity
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
      - name: app-container
        image: myapp:v1.0.0
        imagePullPolicy: Always
        
        # Security context
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          capabilities:
            drop:
            - ALL
        
        # Resource limits
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        
        # Writable volumes for application
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
        - name: config
          mountPath: /etc/config
          readOnly: true
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
      - name: config
        configMap:
          name: app-config
          defaultMode: 0400
```

---

## Troubleshooting Security Issues

```bash
# Check RBAC permissions
kubectl auth can-i get pods --as=alice@example.com
kubectl auth can-i create deployments --as=dev-user

# View role details
kubectl describe role pod-reader
kubectl describe clusterrole edit

# Check service account
kubectl get serviceaccount myapp-sa -o yaml
kubectl get secret myapp-sa-token -o yaml

# View network policies
kubectl get networkpolicies
kubectl describe networkpolicy deny-all-ingress

# Check security context
kubectl get pod secure-pod -o yaml | grep -A 20 securityContext

# Verify image scan results
kubectl get imagepolicies
trivy image --severity HIGH myapp:latest

# Check pod security admission
kubectl get pods --field-selector=status.phase=Failed \
  -o yaml | grep -i security

# Audit logs
kubectl logs -n kube-system -l component=kube-apiserver | grep -i denied
```

---

## Summary

- **RBAC**: Fine-grained access control using Roles, RoleBindings, ClusterRoles, ClusterRoleBindings
- **Service Accounts**: Pod identities for API access and authentication
- **Network Policies**: Control Pod-to-Pod and external traffic
- **Security Context**: Container privilege and access control settings
- **PodSecurity Admission**: Policy enforcement for container security
- **Image Scanning**: Vulnerability detection before deployment

Implementing comprehensive security practices ensures a hardened, compliant Kubernetes cluster!
