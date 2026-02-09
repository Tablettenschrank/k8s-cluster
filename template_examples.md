# Kubernetes Resource Templates & Examples

Comprehensive documentation with template examples for common Kubernetes resources with detailed explanations of each field and when to use them.

## 📑 Table of Contents

### Core Concepts
- [Understanding Metadata vs Spec](#understanding-metadata-vs-spec)

### Basic Resources
1. [Namespace](#1-namespace) - Isolation boundary
2. [Deployment](#2-deployment) - Manages ReplicaSets and Pods (stateless apps)
3. [Service](#3-service) - Stable network endpoint for Pods
4. [PersistentVolumeClaim](#4-persistentvolumeclaim) - Request storage
5. [ConfigMap](#5-configmap) - Non-sensitive configuration data
6. [Secret](#6-secret) - Sensitive data (passwords, tokens, keys)

### Gateway API (Modern Routing)
7. [Gateway](#7-gateway) - Entry point for traffic
8. [HTTPRoute](#8-httproute) - HTTP routing rules

### Advanced Resources
9. [StatefulSet](#9-statefulset) - Stateful applications (databases)
10. [DaemonSet](#10-daemonset) - One pod per node
11. [Job](#11-job) - Run-to-completion tasks
12. [CronJob](#12-cronjob) - Scheduled jobs

### Traditional & Security
13. [Ingress](#13-ingress) - Traditional HTTP routing (older than Gateway API)
14. [NetworkPolicy](#14-networkpolicy) - Firewall rules for pods
15. [HorizontalPodAutoscaler](#15-horizontalpodautoscaler) - Auto-scale based on metrics

### Configuration Management
16. [Kustomization](#16-kustomization) - Manage multiple resources together

### Best Practices
- [Common Patterns & Best Practices](#common-patterns--best-practices)

---

## Understanding Metadata vs Spec

Every Kubernetes resource has a similar structure:

```yaml
apiVersion: v1
kind: ResourceType
metadata:           # WHO/WHERE - Identity, location, and how others find you
  name: resource-name
  namespace: resource-namespace
  labels: {}
  annotations: {}
spec:              # WHAT/HOW - Configuration and desired behavior
  # Resource-specific configuration
status:            # CURRENT STATE - Managed by Kubernetes (read-only)
  # Runtime information
```

**Key Concepts:**
- **metadata**: Identity and attributes - "WHO am I, WHERE do I live, how can others FIND me?"
- **spec**: Desired state and behavior - "WHAT should this resource DO and HOW should it behave?"
- **status**: Current state - Kubernetes manages this automatically

---

## 1. Namespace

**Purpose:** Isolation boundary (like folders/apartments) for organizing resources.

**When to use:**
- Isolate different apps, teams, or environments
- Apply resource quotas or network policies per namespace
- Organize multi-tenant clusters

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app-namespace              # REQUIRED: Unique namespace name
  labels:                             # OPTIONAL: Tags for organization/selection
    environment: production
    team: backend
  annotations:                        # OPTIONAL: Extra metadata (non-identifying)
    description: "Namespace for my application"
```

---

## 2. Deployment

**Purpose:** Manages ReplicaSets and Pods for stateless applications.

**When to use:**
- Stateless applications (web servers, APIs)
- Apps that can be replaced/restarted without data loss
- When you need rolling updates

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment             # REQUIRED: Deployment name
  namespace: my-app-namespace         # REQUIRED: Where it lives (or use kustomization)
  labels:                             # OPTIONAL: Tags for this deployment
    app: my-app
    tier: frontend
  annotations:
    description: "Main application deployment"

spec:
  replicas: 3                         # REQUIRED: How many pods to run
  revisionHistoryLimit: 10            # OPTIONAL: How many old ReplicaSets to keep (default: 10)
  
  selector:                           # REQUIRED: How to find pods to manage
    matchLabels:
      app: my-app                     # Must match template.metadata.labels
      version: v1
  
  strategy:                           # OPTIONAL: How to update pods
    type: RollingUpdate               # RollingUpdate (default) or Recreate
    rollingUpdate:
      maxUnavailable: 1               # Max pods unavailable during update
      maxSurge: 1                     # Max extra pods during update
  
  template:                           # REQUIRED: Pod template
    metadata:
      labels:                         # REQUIRED: Must match selector.matchLabels
        app: my-app
        version: v1
      annotations:
        prometheus.io/scrape: "true"  # Example: Prometheus monitoring
    
    spec:                             # Pod specification
      containers:
      - name: my-app-container        # REQUIRED: Container name
        image: nginx:1.21.0           # REQUIRED: Container image
        imagePullPolicy: IfNotPresent # IfNotPresent (default), Always, Never
        
        ports:
        - name: http                  # OPTIONAL: Port name (useful for Service)
          containerPort: 80           # REQUIRED: Port the container listens on
          protocol: TCP               # TCP (default) or UDP
        
        env:                          # OPTIONAL: Environment variables
        - name: DATABASE_HOST
          value: "db.example.com"
        - name: DATABASE_PASSWORD     # From Secret
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        
        resources:                    # OPTIONAL but RECOMMENDED: Resource limits
          requests:                   # Minimum resources needed
            memory: "128Mi"
            cpu: "100m"
          limits:                     # Maximum resources allowed
            memory: "256Mi"
            cpu: "500m"
        
        volumeMounts:                 # OPTIONAL: Mount volumes
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
        - name: data-volume
          mountPath: /data
        
        livenessProbe:                # OPTIONAL: Health check (restart if fails)
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:               # OPTIONAL: Ready check (remove from service if fails)
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      
      volumes:                        # OPTIONAL: Define volumes
      - name: config-volume
        configMap:
          name: app-config
      - name: data-volume
        persistentVolumeClaim:
          claimName: app-data-pvc
```

---

## 3. Service

**Purpose:** Stable network endpoint for Pods (load balancer).

**Service Types:**
- **ClusterIP**: Only accessible within cluster (default)
- **NodePort**: Accessible on each node's IP at a static port
- **LoadBalancer**: Cloud load balancer (or MetalLB)
- **ExternalName**: DNS CNAME redirect

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service                # REQUIRED: Service name
  namespace: my-app-namespace
  labels:
    app: my-app
  annotations:
    metallb.universe.tf/loadBalancerIPs: "10.20.91.10"  # Example: MetalLB IP

spec:
  type: ClusterIP                     # ClusterIP (default), NodePort, LoadBalancer, ExternalName
  
  selector:                           # REQUIRED: Which pods to send traffic to
    app: my-app                       # Matches Pod labels
    version: v1
  
  ports:                              # REQUIRED: Port mappings
  - name: http                        # OPTIONAL: Port name
    protocol: TCP                     # TCP (default) or UDP
    port: 8080                        # REQUIRED: Service port (what clients connect to)
    targetPort: 80                    # REQUIRED: Pod port (where traffic goes)
    # targetPort: http                # Can use container port name instead
  - name: https
    protocol: TCP
    port: 8443
    targetPort: 443
  
  sessionAffinity: None               # None (default) or ClientIP (sticky sessions)
  sessionAffinityConfig:              # OPTIONAL: Session affinity settings
    clientIP:
      timeoutSeconds: 10800
```

**How Service finds Pods:**
The `selector` matches Pod `labels` - Service automatically discovers and load-balances to all matching Pods.

---

## 4. PersistentVolumeClaim

**Purpose:** Request storage for persistent data.

**When to use:**
- Persistent data that survives pod restarts
- Databases, file uploads, logs
- Pair with StatefulSet for stateful apps

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc                  # REQUIRED: PVC name
  namespace: my-app-namespace
  labels:
    app: my-app

spec:
  accessModes:                        # REQUIRED: How volume can be mounted
  - ReadWriteOnce                     # RWO: Single node read-write
  # - ReadOnlyMany                    # ROX: Multiple nodes read-only
  # - ReadWriteMany                   # RWX: Multiple nodes read-write
  
  resources:                          # REQUIRED: Storage size request
    requests:
      storage: 10Gi
  
  storageClassName: nfs-client        # OPTIONAL: Which storage class to use
  
  volumeMode: Filesystem              # Filesystem (default) or Block
```

**Access Modes:**
- **ReadWriteOnce (RWO)**: One node can mount read-write
- **ReadOnlyMany (ROX)**: Many nodes can mount read-only
- **ReadWriteMany (RWX)**: Many nodes can mount read-write

---

## 5. ConfigMap

**Purpose:** Store non-sensitive configuration data.

**Use in Pods:**
1. As environment variables (`env.valueFrom.configMapKeyRef`)
2. As volume mount (`volumes.configMap`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config                    # REQUIRED: ConfigMap name
  namespace: my-app-namespace

data:                                 # Key-value pairs
  database.host: "db.example.com"
  database.port: "5432"
  app.properties: |                   # Multi-line config file
    setting1=value1
    setting2=value2
  nginx.conf: |                       # Example: nginx config
    server {
      listen 80;
      server_name example.com;
    }
```

**Example - Using ConfigMap in Pod:**
```yaml
# As environment variable
env:
- name: DB_HOST
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: database.host

# As volume mount
volumes:
- name: config-volume
  configMap:
    name: app-config
```

---

## 6. Secret

**Purpose:** Store sensitive data (passwords, tokens, keys).

**Secret Types:**
- **Opaque**: Generic secrets (default)
- **kubernetes.io/tls**: TLS certificates
- **kubernetes.io/dockerconfigjson**: Docker registry auth

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret                    # REQUIRED: Secret name
  namespace: my-app-namespace

type: Opaque                          # Opaque (default), kubernetes.io/tls, etc.

data:                                 # Base64-encoded values
  username: YWRtaW4=                  # "admin" in base64
  password: cGFzc3dvcmQxMjM=          # "password123" in base64

stringData:                           # OPTIONAL: Plain text (auto-encoded)
  api-token: "my-secret-token-123"
```

**Usage:** Same as ConfigMap (environment variables or volume mounts).

**Creating base64 values:**
```bash
echo -n "admin" | base64
# Output: YWRtaW4=
```

---

## 7. Gateway

**Purpose:** Entry point for traffic using the Gateway API (modern replacement for Ingress).

**Gateway vs Ingress:**
- Gateway API is newer, more flexible, and standardized
- Supports multiple protocols (HTTP, HTTPS, TCP, UDP, gRPC)
- Better multi-tenancy and cross-namespace references

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway                    # REQUIRED: Gateway name
  namespace: gateway-system           # REQUIRED: Namespace for gateway infrastructure
  labels:
    environment: production
  annotations:
    metallb.universe.tf/loadBalancerIPs: "10.20.91.5"

spec:
  gatewayClassName: nginx             # REQUIRED: Which gateway controller (nginx, traefik, kong, etc.)
  
  listeners:                          # REQUIRED: At least one listener
  - name: http                        # REQUIRED: Listener name (for sectionName reference)
    protocol: HTTP                    # REQUIRED: HTTP, HTTPS, TCP, UDP
    port: 80                          # REQUIRED: Port to listen on
    hostname: "*.example.com"         # OPTIONAL: Hostname filter (supports wildcards)
    
    allowedRoutes:                    # OPTIONAL: Which routes can attach
      namespaces:
        from: All                     # All, Same, Selector
        # selector:                   # If using Selector
        #   matchLabels:
        #     gateway: enabled
      kinds:                          # OPTIONAL: Which route types
      - group: gateway.networking.k8s.io
        kind: HTTPRoute
  
  - name: https                       # HTTPS listener example
    protocol: HTTPS
    port: 443
    hostname: "*.example.com"
    
    tls:                              # REQUIRED for HTTPS
      mode: Terminate                 # Terminate (decrypt) or Passthrough
      certificateRefs:                # REQUIRED: TLS certificate secret
      - name: example-tls-cert
        # namespace: cert-namespace   # OPTIONAL: If cert in different namespace
    
    allowedRoutes:
      namespaces:
        from: All
  
  - name: tcp-custom                  # TCP listener example
    protocol: TCP
    port: 2525
    allowedRoutes:
      namespaces:
        from: All
      kinds:
      - group: gateway.networking.k8s.io
        kind: TCPRoute
```

**Important:** The `allowedRoutes.namespaces.from` setting determines which namespaces can attach routes:
- **Same**: Only routes in the same namespace as the Gateway
- **All**: Routes from any namespace (best for multi-tenant setups)
- **Selector**: Only namespaces matching specific labels

---

## 8. HTTPRoute

**Purpose:** Define HTTP routing rules for a Gateway.

**Common Patterns:**
- **Path-based routing**: `/api` → api-service, `/` → frontend-service
- **Host-based routing**: `api.example.com` → api-service, `web.example.com` → web-service
- **Canary deployments**: 90% → stable, 10% → canary
- **Cross-namespace routing**: Route in app namespace → Gateway in gateway namespace

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route                  # REQUIRED: Route name
  namespace: my-app-namespace         # REQUIRED: Where the route lives
  labels:
    app: my-app

spec:
  parentRefs:                         # REQUIRED: Which gateway(s) to attach to
  - name: my-gateway                  # REQUIRED: Gateway name
    namespace: gateway-system         # REQUIRED if gateway in different namespace
    sectionName: http                 # OPTIONAL: Specific listener name (required if multiple listeners)
  
  hostnames:                          # OPTIONAL but RECOMMENDED: Which domains
  - "app.example.com"
  - "www.example.com"
  
  rules:                              # REQUIRED: At least one routing rule
  - matches:                          # OPTIONAL: Traffic matching (default: all traffic)
    - path:                           # Path-based routing
        type: PathPrefix              # PathPrefix, Exact, RegularExpression
        value: /api
    - headers:                        # Header-based routing
      - name: X-Custom-Header
        value: special
    - queryParams:                    # Query parameter routing
      - name: version
        value: v2
    - method: GET                     # HTTP method
    
    filters:                          # OPTIONAL: Modify requests/responses
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: X-Custom-Header
          value: added-value
        remove:
        - X-Remove-This
    - type: RequestRedirect           # Redirect example
      requestRedirect:
        hostname: new.example.com
        statusCode: 301
    
    backendRefs:                      # REQUIRED: Where to send traffic
    - name: my-app-service            # REQUIRED: Service name
      namespace: my-app-namespace     # OPTIONAL: If service in different namespace
      port: 8080                      # REQUIRED: Service port
      weight: 90                      # OPTIONAL: Traffic weight (for A/B testing)
    - name: my-app-service-v2         # Second backend for canary/blue-green
      port: 8080
      weight: 10                      # 10% traffic to v2
  
  - matches:                          # Second rule example
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: my-app-service
      port: 8080
```

**Critical: Cross-Namespace References**

When HTTPRoute and Gateway are in **different namespaces**, you **MUST** specify the namespace in `parentRefs`:

```yaml
parentRefs:
  - name: gateway-nginx
    namespace: gateway-nginx    # ← REQUIRED for cross-namespace
```

**When to use `sectionName`:**
- ❌ **Not required** if Gateway has only ONE listener
- ✅ **Required** if Gateway has MULTIPLE listeners (to specify which one)

---

## 9. StatefulSet

**Purpose:** Manage stateful applications with stable network identities and persistent storage.

**StatefulSet vs Deployment:**
- **StatefulSet**: Stable pod names (my-database-0, my-database-1), ordered deployment/scaling
- **Deployment**: Random pod names, can deploy in parallel
- **Use StatefulSet for**: Databases, Kafka, ZooKeeper, Redis, etc.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-database                   # REQUIRED: StatefulSet name
  namespace: my-app-namespace

spec:
  serviceName: my-database-headless   # REQUIRED: Headless service name
  replicas: 3                         # REQUIRED: Number of pods
  
  selector:                           # REQUIRED: Pod selector
    matchLabels:
      app: my-database
  
  template:                           # Pod template (same as Deployment)
    metadata:
      labels:
        app: my-database
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  
  volumeClaimTemplates:               # REQUIRED: PVC template for each pod
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
      storageClassName: nfs-client
```

**Features:**
- Each pod gets a unique name: `my-database-0`, `my-database-1`, `my-database-2`
- Each pod gets its own PersistentVolumeClaim
- Pods are created/deleted in order
- Stable network identity via headless service

---

## 10. DaemonSet

**Purpose:** Run one pod per node (monitoring, logging, system daemons).

**When to use:**
- Monitoring agents (Prometheus node-exporter)
- Log collectors (Fluentd, Filebeat)
- Storage daemons
- Network plugins

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter                 # REQUIRED: DaemonSet name
  namespace: monitoring

spec:
  selector:                           # REQUIRED: Pod selector
    matchLabels:
      app: node-exporter
  
  template:                           # Pod template
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true               # OPTIONAL: Use host's network namespace
      hostPID: true                   # OPTIONAL: Use host's PID namespace
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
```

**Behavior:** Automatically deploys one pod to each node (including new nodes added later).

---

## 11. Job

**Purpose:** Run tasks to completion (one-time or batch processing).

**When to use:**
- Database migrations
- Batch processing
- Data imports/exports
- One-time tasks

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: database-migration            # REQUIRED: Job name
  namespace: my-app-namespace

spec:
  completions: 1                      # OPTIONAL: How many successful completions needed
  parallelism: 1                      # OPTIONAL: How many pods to run in parallel
  backoffLimit: 3                     # OPTIONAL: Retries before marking failed
  
  template:                           # Pod template
    spec:
      restartPolicy: OnFailure        # OnFailure or Never (required for Job)
      containers:
      - name: migration
        image: my-app:v1.2.0
        command: ["./migrate-db.sh"]
```

**RestartPolicy:**
- **OnFailure**: Restart pod if it fails
- **Never**: Don't restart, just mark as failed

---

## 12. CronJob

**Purpose:** Run Jobs on a schedule (like cron).

**Cron Format:**
```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6, Sunday=0)
│ │ │ │ │
* * * * *
```

**Examples:**
- `0 2 * * *` - Daily at 2am
- `*/5 * * * *` - Every 5 minutes
- `0 0 * * 0` - Weekly on Sunday at midnight

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-database               # REQUIRED: CronJob name
  namespace: my-app-namespace

spec:
  schedule: "0 2 * * *"               # REQUIRED: Cron format (2am daily)
  successfulJobsHistoryLimit: 3       # OPTIONAL: Keep 3 successful jobs
  failedJobsHistoryLimit: 1           # OPTIONAL: Keep 1 failed job
  
  jobTemplate:                        # Job template
    spec:
      template:                       # Pod template
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:14
            command: ["pg_dump", "-h", "db-host", "-U", "user"]
```

---

## 13. Ingress

**Purpose:** Traditional HTTP/HTTPS routing (older than Gateway API).

**Ingress vs Gateway API:**
- **Ingress**: Simpler, HTTP/HTTPS only
- **Gateway API**: More powerful, supports TCP/UDP, better multi-tenancy

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress                # REQUIRED: Ingress name
  namespace: my-app-namespace
  annotations:                        # Controller-specific annotations
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt

spec:
  ingressClassName: nginx             # REQUIRED: Which ingress controller
  
  tls:                                # OPTIONAL: TLS configuration
  - hosts:
    - app.example.com
    secretName: app-tls-cert
  
  rules:                              # REQUIRED: Routing rules
  - host: app.example.com             # OPTIONAL: Hostname
    http:
      paths:
      - path: /api
        pathType: Prefix              # Prefix, Exact, ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

---

## 14. NetworkPolicy

**Purpose:** Firewall rules for pods (network segmentation).

**Default Behavior:**
- **No NetworkPolicy**: All traffic allowed
- **With NetworkPolicy**: Default deny, only explicitly allowed traffic passes

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend     # REQUIRED: Policy name
  namespace: my-app-namespace

spec:
  podSelector:                        # REQUIRED: Which pods this applies to
    matchLabels:
      tier: backend
  
  policyTypes:                        # REQUIRED: Ingress, Egress, or both
  - Ingress
  - Egress
  
  ingress:                            # OPTIONAL: Incoming traffic rules
  - from:
    - podSelector:                    # Allow from pods with these labels
        matchLabels:
          tier: frontend
    - namespaceSelector:              # Allow from specific namespaces
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 8080
  
  egress:                             # OPTIONAL: Outgoing traffic rules
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
```

**Example Scenario:**
This policy allows:
- ✅ Frontend pods → Backend pods (port 8080)
- ✅ Monitoring namespace → Backend pods (port 8080)
- ✅ Backend pods → Database pods (port 5432)
- ❌ All other traffic is blocked

---

## 15. HorizontalPodAutoscaler

**Purpose:** Automatically scale pods based on CPU, memory, or custom metrics.

**When to use:**
- Variable traffic patterns
- Cost optimization
- Automatic scaling based on load

**Requirements:** Metrics-server must be installed in the cluster.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa                    # REQUIRED: HPA name
  namespace: my-app-namespace

spec:
  scaleTargetRef:                     # REQUIRED: What to scale
    apiVersion: apps/v1
    kind: Deployment
    name: my-app-deployment
  
  minReplicas: 2                      # REQUIRED: Minimum pods
  maxReplicas: 10                     # REQUIRED: Maximum pods
  
  metrics:                            # REQUIRED: Scaling metrics
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70        # Scale when avg CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80        # Scale when avg memory > 80%
```

**How it works:**
- HPA monitors metrics every 15 seconds (default)
- If average CPU > 70%, it adds more pods
- If average CPU < 70%, it removes pods (after cooldown period)
- Respects min/max replica limits

---

## 16. Kustomization

**Purpose:** Manage and customize multiple Kubernetes resources together.

**File:** Save as `kustomization.yaml` (separate file, not a Kubernetes resource).

**Deploy with:** `kubectl apply -k <directory>`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: my-app-namespace           # OPTIONAL: Apply to all resources

resources:                            # REQUIRED: List of resource files
- namespace.yaml
- deployment.yaml
- service.yaml
- route-nginx.yaml
- pvc.yaml

commonLabels:                         # OPTIONAL: Add labels to all resources
  app: my-app
  managed-by: kustomize

configMapGenerator:                   # OPTIONAL: Generate ConfigMaps
- name: app-config
  files:
  - config.properties

secretGenerator:                      # OPTIONAL: Generate Secrets
- name: app-secret
  literals:
  - password=secret123

images:                               # OPTIONAL: Override image tags
- name: my-app
  newTag: v1.2.3

patches:                              # OPTIONAL: Patch specific resources
- path: patch-resources.yaml
  target:
    kind: Deployment
    name: my-app-deployment
```

**Benefits:**
- Group related resources together
- Apply common configurations to multiple resources
- Environment-specific customizations (dev, staging, prod)
- No templating needed

---

## Common Patterns & Best Practices

### 1. Cross-Namespace References

Always specify namespace when referencing resources in different namespaces.

**Example: HTTPRoute → Gateway (different namespace)**
```yaml
parentRefs:
  - name: gateway-nginx
    namespace: gateway-nginx      # ← REQUIRED
```

### 2. Labels & Selectors

Use consistent labels for selector matching.

**Example: Deployment selector must match Pod template labels**
```yaml
selector:
  matchLabels:
    app: my-app                   # ← Must match
template:
  metadata:
    labels:
      app: my-app                 # ← Must match
```

**Example: Service selector must match Pod labels**
```yaml
# Service
selector:
  app: my-app                     # ← Matches pods with this label

# Deployment creates pods with:
template:
  metadata:
    labels:
      app: my-app                 # ← Service finds these pods
```

### 3. Resource Requests & Limits

Always set for production workloads to ensure proper scheduling and resource allocation.

```yaml
resources:
  requests:                       # What you need (for scheduling)
    memory: "128Mi"
    cpu: "100m"
  limits:                         # Maximum allowed (for protection)
    memory: "256Mi"
    cpu: "500m"
```

**CPU Units:**
- `100m` = 0.1 CPU (10% of one CPU core)
- `1` or `1000m` = 1 full CPU core

**Memory Units:**
- `128Mi` = 128 Mebibytes (128 * 1024 * 1024 bytes)
- `1Gi` = 1 Gibibyte

### 4. Health Checks

Use both liveness and readiness probes for robust deployments.

```yaml
livenessProbe:                    # Restart if unhealthy
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:                   # Remove from service if not ready
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

**Difference:**
- **Liveness**: Checks if pod is alive (restarts if fails)
- **Readiness**: Checks if pod is ready to receive traffic (removes from service if fails)

### 5. Security Best Practices

```yaml
securityContext:
  runAsNonRoot: true              # Don't run as root
  runAsUser: 1000                 # Run as specific user
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true    # Make filesystem read-only
  capabilities:
    drop:
    - ALL                         # Drop all Linux capabilities
```

**Additional Security:**
- Use Secrets for sensitive data (not ConfigMaps)
- Keep images updated with latest security patches
- Use NetworkPolicies to restrict traffic
- Scan images for vulnerabilities
- Use RBAC for access control

### 6. Gateway API Best Practices

**Multi-Gateway Setup (like your project):**
```yaml
# NGINX Gateway for *.n.k8s.tablettenschrank.de
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-nginx
  namespace: gateway-nginx
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    hostname: "*.n.k8s.tablettenschrank.de"
    allowedRoutes:
      namespaces:
        from: All                 # ← THE KEY SETTING for cross-namespace

---
# Traefik Gateway for *.t.k8s.tablettenschrank.de
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-traefik
  namespace: gateway-traefik
spec:
  gatewayClassName: traefik
  listeners:
  - name: web
    port: 8000
    hostname: "*.t.k8s.tablettenschrank.de"
    allowedRoutes:
      namespaces:
        from: All
```

**Key Points:**
- One Gateway per ingress controller
- Use `allowedRoutes.namespaces.from: All` for multi-tenant
- Use hostname filters to route to different gateways
- Always specify namespace in cross-namespace `parentRefs`
- Use `sectionName` when gateway has multiple listeners

### 7. Namespace Organization

**Recommended Structure:**
```
gateway-nginx/        # Gateway infrastructure
  └── Gateway, Pods
gateway-traefik/      # Gateway infrastructure
  └── Gateway, Pods
my-app/               # Application resources
  └── Deployment, Service, HTTPRoute
monitoring/           # Monitoring stack
  └── Prometheus, Grafana
```

**Benefits:**
- Clear separation of concerns
- Easy to apply different policies per namespace
- Better security boundaries
- Easier to manage RBAC

### 8. Deployment Strategies

**Rolling Update (Zero Downtime):**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0             # Keep all pods running during update
    maxSurge: 1                   # Create 1 extra pod for updates
```

**Blue-Green Deployment:**
```yaml
# Use two deployments with different labels
# Switch Service selector to route to new version
selector:
  app: my-app
  version: blue    # Change to "green" when ready
```

**Canary Deployment:**
```yaml
# Use HTTPRoute with multiple backendRefs
backendRefs:
- name: my-app-stable
  weight: 90                      # 90% to stable
- name: my-app-canary
  weight: 10                      # 10% to canary
```

---

## Quick Reference

### Resource Selection Guide

| Need | Use This |
|------|----------|
| Stateless web app | Deployment + Service |
| Stateful database | StatefulSet + PVC |
| One-time task | Job |
| Scheduled task | CronJob |
| System daemon | DaemonSet |
| Expose app externally | Gateway + HTTPRoute (modern) or Ingress (traditional) |
| Configuration | ConfigMap |
| Secrets | Secret |
| Persistent storage | PersistentVolumeClaim |
| Network isolation | NetworkPolicy |
| Auto-scaling | HorizontalPodAutoscaler |

### Common kubectl Commands

```bash
# Apply resources
kubectl apply -f deployment.yaml
kubectl apply -k ./my-app/              # Kustomize

# Get resources
kubectl get pods -n my-app-namespace
kubectl get gateway -A
kubectl get httproute -n nginx-website

# Describe (detailed info)
kubectl describe pod my-pod -n my-app
kubectl describe gateway gateway-nginx -n gateway-nginx

# Logs
kubectl logs my-pod -n my-app
kubectl logs -f my-pod -n my-app        # Follow logs

# Delete resources
kubectl delete -f deployment.yaml
kubectl delete pod my-pod -n my-app

# Execute commands in pod
kubectl exec -it my-pod -n my-app -- /bin/bash
```

---

## Your Multi-Gateway Setup Example

Based on your working configuration:

```yaml
# Gateway in gateway-nginx namespace
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-nginx
  namespace: gateway-nginx
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "*.n.k8s.tablettenschrank.de"
    allowedRoutes:
      namespaces:
        from: All                     # ← Allows routes from any namespace

---
# Route in nginx-website namespace
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginx-route-nginx
  namespace: nginx-website            # ← Different namespace
spec:
  parentRefs:
  - name: gateway-nginx
    namespace: gateway-nginx          # ← MUST specify namespace
    # sectionName: http               # ← Optional (only 1 listener)
  hostnames:
  - "web.n.k8s.tablettenschrank.de"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: nginx-deployment          # ← Same namespace (nginx-website)
      port: 8080
```

**Why this works:**
1. ✅ Gateway has `allowedRoutes.namespaces.from: All`
2. ✅ HTTPRoute specifies `namespace: gateway-nginx` in parentRefs
3. ✅ HTTPRoute finds Service in same namespace (no namespace needed)

---

**End of Documentation**

For more information, see the official Kubernetes documentation: https://kubernetes.io/docs/
