# Kubernetes + Microservices — Production Prep Guide

> Three-service setup. Each section covers: **what it is → why it matters → how it works → production tips**.
> Services used throughout: **Frontend**, **API Service**, **Auth Service**.

---

## The Architecture We're Building

```
                        Internet
                            |
                      [ Ingress ]           ← single entry point
                     /     |     \
              /frontend  /api   /auth
                 |         |        |
          [Frontend    [API       [Auth
           Service]    Service]   Service]
                            |        |
                       [ConfigMap] [Secret]
```

All three services run inside Kubernetes (k8s). Users hit one URL — the Ingress routes them to the right service.

---

## 1. Core Concepts — The Basics

### What Kubernetes Does
K8s takes your containerised apps and handles:
- **Running them** (scheduling containers onto nodes)
- **Healing them** (restarts crashed containers)
- **Scaling them** (adds/removes replicas based on load)
- **Networking them** (gives every pod an IP, routes traffic)

Think of it as an operating system for a cluster of machines.

### The Mental Model

```
Cluster
  └── Node (a VM or physical machine)
        └── Pod (one or more containers, same IP)
              └── Container (your Docker image)
```

| Term | One-liner |
|---|---|
| **Cluster** | The whole k8s environment |
| **Node** | A machine (VM) in the cluster |
| **Pod** | Smallest deployable unit — wraps one or more containers |
| **Deployment** | Manages a set of identical Pods; handles rollouts |
| **Service** | Stable network endpoint for a group of Pods |
| **Ingress** | HTTP routing from the outside world into Services |
| **Namespace** | Logical partition — like folders inside the cluster |
| **ConfigMap** | Non-secret config values |
| **Secret** | Sensitive config (passwords, tokens) — base64 encoded |

---

## 2. Namespaces — Organise First

Split your services by namespace so they don't pollute each other.

```yaml
# namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

Apply it:
```bash
kubectl apply -f namespaces.yaml
kubectl get namespaces
```

> **Rule:** Always deploy to a named namespace. Never use `default` in production.

---

## 3. ConfigMaps & Secrets — Config Before Code

### ConfigMap — non-sensitive values

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: production
data:
  LOG_LEVEL: "info"
  API_BASE_URL: "https://api.myapp.com"
  ALLOWED_ORIGINS: "https://myapp.com"
```

### Secret — sensitive values

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
  namespace: production
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=      # base64 encoded
  JWT_SECRET: c3VwZXJzZWNyZXQ=
```

Encode a value:
```bash
echo -n "mypassword" | base64
```

> **Production rule:** Never hardcode secrets in your Docker image or code. Always inject via Secrets.  
> Even better: Use **External Secrets Operator** to pull from AWS Secrets Manager / HashiCorp Vault instead of storing secrets in k8s at all.

---

## 4. Deployments — Running Your Services

A Deployment tells k8s: *"Run 3 copies of this container. If one dies, restart it. Roll out updates without downtime."*

### Service 1: Frontend

```yaml
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: production
  labels:
    app: frontend
spec:
  replicas: 2                          # run 2 copies
  selector:
    matchLabels:
      app: frontend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                      # spin up 1 extra during update
      maxUnavailable: 0                # never take pods down before new ones are ready
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: myrepo/frontend:1.4.2  # always pin a version, never use :latest
          ports:
            - containerPort: 3000
          resources:                    # always set these in production
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          readinessProbe:               # k8s only sends traffic when this passes
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:                # k8s restarts the pod if this fails
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
          envFrom:
            - configMapRef:
                name: api-config        # inject all ConfigMap values as env vars
```

### Service 2: API Service

```yaml
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: production
  labels:
    app: api-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api-service
    spec:
      containers:
        - name: api-service
          image: myrepo/api-service:2.1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 30
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: api-secrets
                  key: DB_PASSWORD
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: api-secrets
                  key: JWT_SECRET
          envFrom:
            - configMapRef:
                name: api-config
```

### Service 3: Auth Service

```yaml
# auth-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: production
  labels:
    app: auth-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auth-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
        - name: auth-service
          image: myrepo/auth-service:1.0.5
          ports:
            - containerPort: 9000
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          readinessProbe:
            httpGet:
              path: /ready
              port: 9000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /ready
              port: 9000
            initialDelaySeconds: 15
            periodSeconds: 20
          env:
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: api-secrets
                  key: JWT_SECRET
```

---

## 5. Services — Internal Networking

A **Kubernetes Service** gives a stable IP/DNS name to a group of Pods. Pods come and go (they get new IPs), but the Service IP stays the same.

### Three Service Types

| Type | Accessible from | Use case |
|---|---|---|
| **ClusterIP** | Inside cluster only | Internal service-to-service calls |
| **NodePort** | External (via node IP + port) | Dev/testing only |
| **LoadBalancer** | External (cloud LB) | Exposing one service directly — use Ingress instead |

**For microservices: use ClusterIP for everything. Let Ingress handle external traffic.**

### Frontend Service
```yaml
# frontend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: production
spec:
  selector:
    app: frontend              # matches pods with this label
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

### API Service
```yaml
# api-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: production
spec:
  selector:
    app: api-service
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### Auth Service
```yaml
# auth-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-svc
  namespace: production
spec:
  selector:
    app: auth-service
  ports:
    - port: 80
      targetPort: 9000
  type: ClusterIP
```

Internal DNS — how services call each other:
```
http://api-svc.production.svc.cluster.local
http://auth-svc.production.svc.cluster.local

# Short form works within same namespace:
http://api-svc
http://auth-svc
```

---

## 6. Ingress — Inbound Traffic from the Internet

Ingress is a reverse proxy that sits at the edge of your cluster and routes HTTP/HTTPS traffic to the right Service based on host/path rules.

### Step 1: Install an Ingress Controller

The Ingress object is just a config. You need a controller (the actual proxy) running in the cluster. Most common: **NGINX Ingress Controller** or **AWS ALB Controller**.

```bash
# Install NGINX Ingress Controller (using Helm)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### Step 2: Create the Ingress Resource

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"   # auto TLS cert
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.com
        - api.myapp.com
      secretName: myapp-tls                              # cert stored here by cert-manager
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80

    - host: api.myapp.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /auth
            pathType: Prefix
            backend:
              service:
                name: auth-svc
                port:
                  number: 80
```

### Step 3: Auto TLS with cert-manager

```bash
# Install cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

```yaml
# cluster-issuer.yaml — tells cert-manager to use Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@myapp.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

Traffic flow end to end:
```
User → myapp.com → DNS → Cloud Load Balancer → Ingress Controller Pod
         → routes to frontend-svc → Frontend Pod
                    api-svc → API Pod
                    auth-svc → Auth Pod
```

---

## 7. Horizontal Pod Autoscaler (HPA) — Scale on Demand

HPA automatically adds/removes pod replicas based on CPU or memory load.

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60    # scale out when avg CPU > 60%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
```

> **Requirement:** The metrics-server must be installed for HPA to work.
> ```bash
> kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
> ```

---

## 8. PodDisruptionBudget — Safe Rollouts & Node Drains

Ensures k8s never takes down too many pods at once during updates or node maintenance.

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: production
spec:
  minAvailable: 2                # always keep at least 2 pods running
  selector:
    matchLabels:
      app: api-service
```

---

## 9. Network Policies — Zero Trust Networking

By default, all pods in a cluster can talk to each other. Network Policies lock this down — only explicitly allowed communication works.

```yaml
# networkpolicy.yaml
# API Service can only receive traffic from Frontend and Auth Service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-only-frontend-auth
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
        - podSelector:
            matchLabels:
              app: auth-service
      ports:
        - port: 8080
```

> **Production rule:** Start with a "deny all" policy, then add explicit allows.

---

## 10. RBAC — Who Can Do What in the Cluster

RBAC controls which humans and service accounts can interact with the k8s API.

```yaml
# rbac.yaml
# A role that lets an app read secrets in 'production' namespace only
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["secrets", "configmaps"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-secret-reader
  namespace: production
subjects:
  - kind: ServiceAccount
    name: api-service-account
    namespace: production
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 11. Resource Checklist — Production Grade

### Health probes — must have on every container

| Probe | Fails → | Use for |
|---|---|---|
| **readinessProbe** | Stop sending traffic to pod | App started but not ready (DB connecting) |
| **livenessProbe** | Restart the pod | App is deadlocked / stuck |
| **startupProbe** | Give slow apps time before liveness kicks in | Java apps with long boot |

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30        # allow up to 30 × 10s = 5 min to start
  periodSeconds: 10
```

### Resource requests vs limits

```yaml
resources:
  requests:
    cpu: "100m"       # guaranteed — k8s schedules based on this
    memory: "128Mi"
  limits:
    cpu: "500m"       # hard cap — CPU throttled at this
    memory: "256Mi"   # OOMKill if exceeded
```

> **Rule:** Always set both. No limits = a single rogue pod can eat an entire node.

### Image best practices

```dockerfile
# ✅ Do this
FROM node:20-alpine             # minimal base image
COPY --chown=node:node . .
USER node                       # never run as root
```

```yaml
# ✅ In deployment
image: myrepo/api-service:2.1.0  # pinned version
imagePullPolicy: IfNotPresent
```

---

## 12. Observability — Logs, Metrics, Traces

### Logs
Use structured JSON logs from your app. In k8s, logs go to stdout/stderr and are collected by the logging stack.

```bash
kubectl logs -n production deployment/api-service --follow
kubectl logs -n production -l app=api-service --tail=100
```

**Production stack:** FluentBit → Elasticsearch/OpenSearch → Kibana  
or: FluentBit → CloudWatch / Datadog / Grafana Loki

### Metrics
**Prometheus + Grafana** is the standard.

```yaml
# Add this annotation so Prometheus scrapes your pod
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: "/metrics"
    prometheus.io/port: "8080"
```

### Distributed Tracing
Use **OpenTelemetry** to instrument your services. Traces show how a request flows through Frontend → API → Auth.

Export to: Jaeger, Tempo, Datadog, AWS X-Ray.

---

## 13. Deploying — The Workflow

```bash
# 1. Build and push your image
docker build -t myrepo/api-service:2.1.1 .
docker push myrepo/api-service:2.1.1

# 2. Update the image in the deployment
kubectl set image deployment/api-service \
  api-service=myrepo/api-service:2.1.1 \
  -n production

# 3. Watch the rollout
kubectl rollout status deployment/api-service -n production

# 4. Check pods
kubectl get pods -n production

# 5. If something goes wrong — roll back instantly
kubectl rollout undo deployment/api-service -n production
```

### Using Helm (recommended for production)

Helm packages all your k8s YAML into a chart — handles versioning, templating, and environment differences.

```bash
# Install your app as a Helm release
helm install myapp ./charts/myapp \
  --namespace production \
  --values values.production.yaml

# Upgrade
helm upgrade myapp ./charts/myapp \
  --namespace production \
  --values values.production.yaml

# Rollback
helm rollback myapp 1 --namespace production
```

---

## 14. Useful kubectl Commands — Day to Day

```bash
# See everything in a namespace
kubectl get all -n production

# Describe a pod (events, config, errors)
kubectl describe pod <pod-name> -n production

# Shell into a running container
kubectl exec -it <pod-name> -n production -- /bin/sh

# Port-forward to test a service locally
kubectl port-forward svc/api-svc 8080:80 -n production

# See resource usage
kubectl top pods -n production
kubectl top nodes

# Watch pods in real time
kubectl get pods -n production -w

# Get events (great for debugging)
kubectl get events -n production --sort-by='.lastTimestamp'

# Apply all YAML in a directory
kubectl apply -f ./k8s/ --recursive
```

---

## 15. Folder Structure — Keep It Organised

```
k8s/
├── namespaces.yaml
├── configmaps/
│   └── api-config.yaml
├── secrets/
│   └── api-secrets.yaml          # never commit real secrets to git
├── deployments/
│   ├── frontend-deployment.yaml
│   ├── api-deployment.yaml
│   └── auth-deployment.yaml
├── services/
│   ├── frontend-service.yaml
│   ├── api-service.yaml
│   └── auth-service.yaml
├── ingress/
│   ├── ingress.yaml
│   └── cluster-issuer.yaml
├── autoscaling/
│   └── api-hpa.yaml
├── policies/
│   ├── network-policies.yaml
│   └── pdb.yaml
└── rbac/
    └── rbac.yaml
```

---

## Quick Reference — Production Checklist

| Item | Done? |
|---|---|
| Namespace created (not using `default`) | ✅ |
| All secrets in k8s Secrets (not ConfigMap, not code) | ✅ |
| Image pinned to a version tag (not `:latest`) | ✅ |
| Resource requests and limits set on every container | ✅ |
| readinessProbe on every container | ✅ |
| livenessProbe on every container | ✅ |
| RollingUpdate strategy with `maxUnavailable: 0` | ✅ |
| HPA configured (minReplicas ≥ 2) | ✅ |
| PodDisruptionBudget in place | ✅ |
| Ingress with TLS (cert-manager + Let's Encrypt) | ✅ |
| Network Policies restricting inter-service traffic | ✅ |
| RBAC — least privilege service accounts | ✅ |
| Containers not running as root | ✅ |
| Structured logging to stdout | ✅ |
| Metrics exposed for Prometheus | ✅ |

---

## One-liners to Memorise

- **Pod** = smallest unit; **Deployment** = manages pods; **Service** = stable network endpoint.
- **ClusterIP** = internal only; use Ingress for external HTTP traffic.
- **Ingress** = L7 routing (host + path rules); needs an Ingress Controller running.
- **ConfigMap** = non-secret config; **Secret** = sensitive values — never hardcode either.
- **readinessProbe** = gates traffic; **livenessProbe** = gates pod restart.
- **requests** = scheduled guarantee; **limits** = hard cap — always set both.
- **HPA** = autoscale pods on CPU/memory; needs metrics-server.
- **PDB** = guarantees minimum available pods during disruptions.
- **NetworkPolicy** = pod-level firewall; default is allow-all — lock it down.
- **RBAC** = who can call the k8s API; always use least-privilege service accounts.
- **Helm** = package manager for k8s YAML; use it to manage releases and rollbacks.
- **Rolling update** = zero-downtime deploys; `rollout undo` = instant rollback.
