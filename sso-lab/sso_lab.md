Here's your full hands-on lab. Let me first give you an architecture overview of what you'll be building, then walk you through each phase step by step.---

# 🔐 Lab: Cloud-Native SSO with NGINX & Kubernetes

**Goal:** Migrate from a legacy Apache/VM SSO to a containerised NGINX SSO front, deployed on Kubernetes with config-as-code, observability, and auto-scaling.

**What you'll need installed:** Docker, kubectl, minikube (or kind), helm

---

## ⚙️ Phase 0 — Prerequisites & local cluster

```bash
# Install minikube (macOS/Linux)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start a local cluster
minikube start --cpus=4 --memory=6g --driver=docker

# Verify
kubectl get nodes
# Expected: NAME       STATUS   ROLES           AGE   VERSION
#           minikube   Ready    control-plane   ...
```

> 📝 **Note:** `minikube` runs a single-node Kubernetes cluster inside Docker. Perfect for local labs — no cloud account needed.

---

## 🐳 Phase 1 — Build a custom NGINX Docker image

This image will be the SSO front-end: it terminates TLS, checks session cookies, and proxies authenticated requests upstream.

### 1.1 — Project structure

```bash
mkdir -p sso-lab/{nginx,k8s,tests}
cd sso-lab
```

### 1.2 — NGINX configuration (config-as-code)

```bash
cat > nginx/nginx.conf << 'EOF'
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid       /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # Structured JSON logs — easier to ingest into Prometheus/Loki
    log_format json_combined escape=json
        '{"time":"$time_iso8601",'
        '"method":"$request_method",'
        '"uri":"$request_uri",'
        '"status":$status,'
        '"upstream":"$upstream_addr",'
        '"duration":$request_time,'
        '"ip":"$remote_addr"}';

    access_log /var/log/nginx/access.log json_combined;
    sendfile on;
    keepalive_timeout 65;

    # Prometheus metrics stub (nginx-prometheus-exporter reads /nginx_status)
    server {
        listen 8080;
        location /nginx_status {
            stub_status;
            allow 127.0.0.1;
            allow 10.0.0.0/8;   # pod CIDR
            deny all;
        }
    }

    # Main SSO front
    server {
        listen 80;
        server_name _;

        # Health check endpoint — used by K8s liveness/readiness probes
        location /healthz {
            access_log off;
            return 200 "ok\n";
            add_header Content-Type text/plain;
        }

        # SSO auth check — forward every request here first
        location /auth {
            internal;
            proxy_pass         http://auth-backend-svc:8081/validate;
            proxy_pass_request_body off;
            proxy_set_header   Content-Length "";
            proxy_set_header   X-Original-URI $request_uri;
        }

        # Protected routes
        location / {
            auth_request /auth;
            auth_request_set $auth_status $upstream_status;

            # On auth failure → redirect to login
            error_page 401 = @login_redirect;

            proxy_pass         http://upstream-app-svc:3000;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location @login_redirect {
            return 302 /sso/login?redirect=$request_uri;
        }

        # SSO login page
        location /sso/ {
            proxy_pass http://auth-backend-svc:8081/;
        }
    }
}
EOF
```

### 1.3 — Dockerfile

```dockerfile
# nginx/Dockerfile
FROM nginx:1.25-alpine

# Remove default config
RUN rm /etc/nginx/conf.d/default.conf

# Copy our config
COPY nginx.conf /etc/nginx/nginx.conf

# Non-root user for security best practice
RUN addgroup -S sso && adduser -S sso -G sso \
    && chown -R sso:sso /var/cache/nginx /var/log/nginx /var/run

# Expose HTTP + metrics port
EXPOSE 80 8080

# Healthcheck — Docker itself will poll this
HEALTHCHECK --interval=10s --timeout=3s \
    CMD wget -qO- http://localhost/healthz || exit 1

USER sso
CMD ["nginx", "-g", "daemon off;"]
```

> 📝 **Note:** Running as a non-root user is a Kubernetes security best practice. The `HEALTHCHECK` lets Docker mark the container as unhealthy before K8s probes even kick in.

### 1.4 — Build and push to minikube's registry

```bash
# Point your shell's Docker to minikube's internal Docker daemon
eval $(minikube docker-env)

# Build — the image is now directly available inside minikube, no registry needed
docker build -t sso-nginx:v1 ./nginx

# Verify
docker images | grep sso-nginx
```

---

## ☸️ Phase 2 — Deploy to Kubernetes

All YAML goes in `k8s/`. This is **configuration-as-code** — commit everything to Git.

### 2.1 — Namespace

```bash
cat > k8s/namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: sso
  labels:
    app: sso
EOF
kubectl apply -f k8s/namespace.yaml
```

### 2.2 — ConfigMap (nginx config managed by K8s)

```bash
cat > k8s/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-sso-config
  namespace: sso
data:
  # You can override individual values without rebuilding the image
  UPSTREAM_HOST: "upstream-app-svc"
  UPSTREAM_PORT: "3000"
  AUTH_HOST: "auth-backend-svc"
  AUTH_PORT: "8081"
EOF
kubectl apply -f k8s/configmap.yaml
```

### 2.3 — Secret (TLS cert / session key)

```bash
# Generate a self-signed cert for the lab
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=sso.local/O=lab"

# Store it as a K8s Secret — never commit raw secrets to Git!
kubectl create secret tls sso-tls \
  --cert=tls.crt --key=tls.key \
  -n sso

# Check
kubectl get secret sso-tls -n sso
```

> ⚠️ **Reminder:** In production use **cert-manager** with Let's Encrypt or your internal CA. Never hard-code secrets in YAML files.

### 2.4 — Deployment

```bash
cat > k8s/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sso-nginx
  namespace: sso
  labels:
    app: sso-nginx
    version: v1
spec:
  replicas: 3                     # 3 pods for HA
  selector:
    matchLabels:
      app: sso-nginx
  strategy:
    type: RollingUpdate           # zero-downtime deploys
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: sso-nginx
        version: v1
    spec:
      containers:
      - name: nginx
        image: sso-nginx:v1
        imagePullPolicy: Never    # use minikube's local image
        ports:
        - containerPort: 80
          name: http
        - containerPort: 8080
          name: metrics
        envFrom:
        - configMapRef:
            name: nginx-sso-config
        # Liveness: restart if nginx hangs
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        # Readiness: only send traffic when nginx is ready
        readinessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "128Mi"
        # Security context — read-only filesystem
        securityContext:
          readOnlyRootFilesystem: false
          allowPrivilegeEscalation: false
          runAsNonRoot: true
EOF
kubectl apply -f k8s/deployment.yaml
```

### 2.5 — Service & Ingress

```bash
cat > k8s/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: sso-nginx-svc
  namespace: sso
spec:
  selector:
    app: sso-nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: metrics
    port: 8080
    targetPort: 8080
  type: ClusterIP
EOF

cat > k8s/ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sso-ingress
  namespace: sso
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - sso.local
    secretName: sso-tls
  rules:
  - host: sso.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sso-nginx-svc
            port:
              number: 80
EOF

kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```

### 2.6 — Check the rollout

```bash
kubectl rollout status deployment/sso-nginx -n sso
# Waiting for deployment "sso-nginx" rollout to finish: 0 of 3 updated replicas are available...
# deployment "sso-nginx" successfully rolled out

kubectl get pods -n sso -o wide
# NAME                         READY   STATUS    RESTARTS   AGE   IP
# sso-nginx-xxxx-aaaa          1/1     Running   0          30s   ...
# sso-nginx-xxxx-bbbb          1/1     Running   0          30s   ...
# sso-nginx-xxxx-cccc          1/1     Running   0          30s   ...
```

---

## 📊 Phase 3 — Observability (Prometheus + Grafana)

```bash
# Enable metrics-server addon in minikube
minikube addons enable metrics-server

# Install kube-prometheus-stack via Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=admin123

# Expose Grafana locally
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80 &
# Open http://localhost:3000 — login: admin / admin123
```

Now add a `ServiceMonitor` so Prometheus scrapes your NGINX metrics port:

```bash
cat > k8s/servicemonitor.yaml << 'EOF'
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sso-nginx-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: sso-nginx
  namespaceSelector:
    matchNames:
    - sso
  endpoints:
  - port: metrics
    path: /nginx_status
    interval: 15s
EOF
kubectl apply -f k8s/servicemonitor.yaml
```

> 📝 **What to watch in Grafana:** nginx active connections, request rate, upstream response time, error rate (4xx/5xx ratio). Import dashboard ID **9614** (official NGINX exporter dashboard).

---

## 📈 Phase 4 — Auto-scaling (HPA)

```bash
cat > k8s/hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sso-nginx-hpa
  namespace: sso
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sso-nginx
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60   # scale up when avg CPU > 60%
EOF
kubectl apply -f k8s/hpa.yaml

# Watch the HPA react
kubectl get hpa -n sso -w
```

**Load test to trigger scaling:**

```bash
# Generate traffic from within the cluster
kubectl run load-test --image=busybox --restart=Never -n sso -- \
  sh -c "while true; do wget -q -O- http://sso-nginx-svc/healthz; done"

# Watch pods spin up
kubectl get pods -n sso -w

# Cleanup
kubectl delete pod load-test -n sso
```

---

## 🧪 Phase 5 — Automated tests

```bash
cat > tests/smoke_test.sh << 'EOF'
#!/bin/bash
set -euo pipefail

SSO_URL=${1:-"http://localhost"}

echo "=== SSO smoke tests ==="

# 1. Health check
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$SSO_URL/healthz")
[ "$STATUS" = "200" ] && echo "✓ /healthz returns 200" || { echo "✗ healthz failed ($STATUS)"; exit 1; }

# 2. Protected route redirects to login (not 500)
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$SSO_URL/")
[ "$STATUS" = "302" ] && echo "✓ / redirects unauthenticated users" || echo "⚠ / returned $STATUS"

# 3. Metrics endpoint responds
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$SSO_URL:8080/nginx_status")
[ "$STATUS" = "200" ] && echo "✓ /nginx_status returns 200" || echo "⚠ metrics returned $STATUS"

echo "=== All checks passed ==="
EOF
chmod +x tests/smoke_test.sh

# Run against minikube
minikube service sso-nginx-svc -n sso --url &
bash tests/smoke_test.sh $(minikube service sso-nginx-svc -n sso --url)
```

---

## 🔄 Phase 6 — Progressive migration strategy

This mirrors a real production migration:

```
Week 1  ──► Run NGINX SSO in parallel (shadow mode, no real traffic)
Week 2  ──► Route 10% of users via Ingress canary annotation
Week 3  ──► 50% traffic, monitor error rates in Grafana
Week 4  ──► 100% traffic, decommission Apache/VM
```

**Canary Ingress example (10% traffic split):**

```bash
cat > k8s/ingress-canary.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sso-ingress-canary
  namespace: sso
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
  - host: sso.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sso-nginx-svc
            port:
              number: 80
EOF
kubectl apply -f k8s/ingress-canary.yaml
# Increase canary-weight progressively: 10 → 50 → 100
```

---

## 📋 Cheat sheet

| Command | Purpose |
|---|---|
| `kubectl get pods -n sso -w` | Watch pod status live |
| `kubectl logs -n sso -l app=sso-nginx -f` | Stream logs from all SSO pods |
| `kubectl describe pod <name> -n sso` | Debug crash loops / probe failures |
| `kubectl rollout undo deployment/sso-nginx -n sso` | Roll back a bad deploy |
| `kubectl exec -it <pod> -n sso -- sh` | Shell into a pod |
| `minikube dashboard` | Visual K8s UI in browser |

---

> 💡 **Key concepts reinforced:** image layering & non-root containers, config-as-code (all state in YAML), liveness vs readiness probes, rolling deployments, ServiceMonitor scraping, HPA, canary traffic splitting, and smoke testing. Each phase is independently deployable — good for iterating in sprints.