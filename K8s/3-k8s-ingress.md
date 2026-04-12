# Kubernetes Ingress: How External Traffic Reaches Your Services

A ground-up guide covering how traffic from a user's browser reaches the right pod inside your Kubernetes cluster — from the simpler options (NodePort, LoadBalancer) to the production-standard Ingress, with practical examples and the full request trace.

---

## Table of Contents

1. [The Problem: ClusterIP Is Internal Only](#1-the-problem-clusterip-is-internal-only)
2. [Option 1: NodePort — Simple but Limited](#2-option-1-nodeport--simple-but-limited)
3. [Option 2: LoadBalancer — Better but Expensive](#3-option-2-loadbalancer--better-but-expensive)
4. [Option 3: Ingress — The Production Solution](#4-option-3-ingress--the-production-solution)
5. [How Ingress Actually Works Under the Hood](#5-how-ingress-actually-works-under-the-hood)
6. [The Full Traffic Flow: Browser to Pod](#6-the-full-traffic-flow-browser-to-pod)
7. [Practical Examples](#7-practical-examples)
8. [TLS / HTTPS with Ingress](#8-tls--https-with-ingress)
9. [Production Concerns at the Ingress Layer](#9-production-concerns-at-the-ingress-layer)
10. [Choosing an Ingress Controller](#10-choosing-an-ingress-controller)

---

## 1. The Problem: ClusterIP Is Internal Only

After creating a Service with type `ClusterIP`, your pods are reachable via a stable internal address like `10.96.45.12`. Any pod inside the cluster can call it. But a user sitting in a coffee shop with their browser cannot. That IP doesn't exist on the internet. It's a virtual address maintained only by iptables rules inside the cluster.

So the question becomes: how do you let the outside world reach your services?

There are three approaches, each building on the previous one.

---

## 2. Option 1: NodePort — Simple but Limited

A NodePort Service does everything ClusterIP does, plus it opens a specific port on every node in the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-api-svc
spec:
  type: NodePort
  selector:
    app: my-api
  ports:
  - port: 80             # internal ClusterIP port
    targetPort: 8080     # port on the actual pods
    nodePort: 30080      # opened on EVERY node in the cluster
```

After applying this, if your node's external IP is `34.120.50.10`, a user can hit `http://34.120.50.10:30080` and the traffic gets routed to one of your pods.

### How it works internally

```
User hits: http://34.120.50.10:30080
     │
     ▼
Node receives packet on port 30080
     │
     ▼
iptables rule (same as ClusterIP):
rewrite destination to a real pod IP
     │
     ▼
Pod receives request on port 8080
```

The NodePort is essentially a ClusterIP Service with an extra iptables rule that listens on the node's external interface.

### Why NodePort isn't enough for production

- **Ugly port numbers.** NodePort range is 30000-32767 only. No user wants to type `:30080` in a URL.
- **Node IP instability.** If that node dies, its IP is gone. You'd have to know another node's IP.
- **No TLS termination.** You'd have to handle HTTPS inside your application.
- **No hostname-based routing.** You can't serve `api.mycompany.com` and `dashboard.mycompany.com` from the same port.
- **No path-based routing.** You can't route `/users` to one service and `/orders` to another.
- **Port management nightmare.** Each service needs a unique port number. Managing 50 services on 50 ports is chaos.

NodePort is a building block, not a final solution. But understanding it matters because the other options are built on top of it.

---

## 3. Option 2: LoadBalancer — Better but Expensive

A LoadBalancer Service does everything NodePort does, plus it asks your cloud provider to provision an actual load balancer.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-api-svc
spec:
  type: LoadBalancer
  selector:
    app: my-api
  ports:
  - port: 80
    targetPort: 8080
```

When you apply this, the cloud controller (a Kubernetes component that talks to your cloud provider's API) provisions a real load balancer — an ALB on AWS, a Cloud Load Balancer on GCP, an Azure Load Balancer on AKS. The load balancer gets a public IP or DNS name and routes traffic to the NodePort across your nodes.

### How it works internally

```
User hits: http://ab1234-567.us-east-1.elb.amazonaws.com
     │
     ▼
Cloud load balancer (has a public IP)
     │
     ▼
Forwards to NodePort on one of the cluster nodes
     │
     ▼
iptables routes to a real pod
     │
     ▼
Pod receives request
```

### Why LoadBalancer isn't ideal at scale

- **One load balancer per Service.** If you have 10 microservices that need external access, you get 10 cloud load balancers.
- **Cost.** Cloud load balancers cost roughly $15-25/month each, plus data transfer charges. 10 services = $150-250/month just in load balancer fees, before any traffic costs.
- **No routing logic.** Each load balancer points to exactly one Service. You can't route `/api` to one service and `/web` to another through the same load balancer.
- **No shared TLS.** Each load balancer manages its own certificate separately.
- **DNS complexity.** Each service gets its own DNS name or IP. Your users would need different URLs for each service.

LoadBalancer Services are fine when you have 1-2 services to expose. But most production systems have many services — and that's where Ingress comes in.

---

## 4. Option 3: Ingress — The Production Solution

Ingress gives you one entry point that routes traffic to many Services based on rules you define — hostnames, URL paths, or both.

### The basic concept

Instead of each service having its own load balancer, you have one load balancer that fronts an Ingress Controller. The controller reads your routing rules and sends each request to the right backend Service.

```
Without Ingress (3 services = 3 load balancers):
  LB-1 (api.mycompany.com)        → users-svc
  LB-2 (orders.mycompany.com)     → orders-svc
  LB-3 (dashboard.mycompany.com)  → dashboard-svc
  Cost: 3 × $20/month = $60/month

With Ingress (3 services = 1 load balancer):
  LB-1 → Ingress Controller → routes by host/path:
         api.mycompany.com/users   → users-svc
         api.mycompany.com/orders  → orders-svc
         dashboard.mycompany.com   → dashboard-svc
  Cost: 1 × $20/month = $20/month
```

### The Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: users-svc
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: orders-svc
            port:
              number: 80
  - host: dashboard.mycompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dashboard-svc
            port:
              number: 80
```

In plain English: "If the request is for `api.mycompany.com/users`, send it to the users Service. If it's for `api.mycompany.com/orders`, send it to the orders Service. If it's for `dashboard.mycompany.com`, send it to the dashboard Service."

Three services, one load balancer, one IP address, hostname routing, path routing — all in one place.

### Critical thing most people miss

The Ingress YAML by itself does nothing. It's just a set of routing rules stored in etcd, like any other Kubernetes object. You need an Ingress Controller — a running pod inside your cluster — to read those rules and make them real.

---

## 5. How Ingress Actually Works Under the Hood

### What is an Ingress Controller?

It's a pod running inside your cluster that contains a reverse proxy — most commonly nginx, Traefik, or HAProxy. You install it separately; Kubernetes doesn't ship with one by default.

The Ingress Controller does three things:

**1. Watches the API server** for Ingress objects, Services, and Endpoints. Same watch pattern as every other Kubernetes controller.

**2. Generates proxy configuration from your Ingress rules.** For the NGINX Ingress Controller, it reads your Ingress YAML and dynamically generates an `nginx.conf` file. Your rules become:

```nginx
# Auto-generated — you never write this directly

server {
    listen 443 ssl;
    server_name api.mycompany.com;
    ssl_certificate     /etc/tls/tls.crt;
    ssl_certificate_key /etc/tls/tls.key;

    location /users {
        proxy_pass http://users-svc.default.svc.cluster.local:80;
    }
    location /orders {
        proxy_pass http://orders-svc.default.svc.cluster.local:80;
    }
}

server {
    listen 443 ssl;
    server_name dashboard.mycompany.com;

    location / {
        proxy_pass http://dashboard-svc.default.svc.cluster.local:80;
    }
}
```

**3. Provisions a cloud load balancer for itself** by creating a LoadBalancer-type Service. This is the one and only load balancer. All external traffic flows through it into the Ingress Controller pod, which then routes to the correct backend.

### When things change

When you update an Ingress rule, add a new Service, or pod Endpoints change, the controller detects the change via its API server watch, regenerates `nginx.conf`, and reloads nginx with zero downtime. This is why you can add new routing rules without any infrastructure changes — just `kubectl apply` a new Ingress YAML.

---

## 6. The Full Traffic Flow: Browser to Pod

Let's trace exactly what happens when a user types `https://api.mycompany.com/users` in their browser.

### Step 1: DNS resolution

The user's browser needs to turn `api.mycompany.com` into an IP address. Your DNS provider (Route53, Cloudflare, etc.) has a record pointing `api.mycompany.com` to the cloud load balancer's public IP: `34.120.50.10`.

You configure this DNS record yourself when you set up the Ingress. The load balancer's IP is assigned by the cloud provider when the Ingress Controller provisions it.

### Step 2: TLS handshake

The browser connects to `34.120.50.10:443` (HTTPS). TLS can be terminated at two places depending on your configuration:

- **At the load balancer.** The cloud LB decrypts the request, then forwards plain HTTP to the Ingress Controller. Common on AWS ALB.
- **At the Ingress Controller.** The load balancer passes encrypted traffic through, and the nginx inside the controller decrypts it using the TLS certificate from the Kubernetes Secret referenced in the Ingress YAML. Common with NGINX Ingress Controller.

### Step 3: Load balancer forwards to Ingress Controller

The cloud load balancer forwards the request to one of the Ingress Controller pods (you usually run 2-3 replicas for high availability). It reaches the controller via a NodePort on one of the cluster nodes.

### Step 4: Ingress Controller inspects hostname and path

nginx inside the controller reads the HTTP request:

```
Host: api.mycompany.com
GET /users/123 HTTP/1.1
```

It matches this against the Ingress rules:
- Host matches `api.mycompany.com` — check.
- Path `/users/123` matches prefix `/users` — check.
- Backend: `users-svc:80`.

### Step 5: Traffic hits the internal Service

The Ingress Controller forwards the request to `users-svc:80`. From here, it's the exact same flow we covered in the networking guide — ClusterIP, iptables DNAT, packet reaches a real pod.

### Step 6: Pod responds

The response travels back through the chain: pod → Service → Ingress Controller (which can add response headers, compress the response, handle CORS) → load balancer → internet → user's browser.

### The complete path

```
User browser
  │  DNS: api.mycompany.com → 34.120.50.10
  ▼
Cloud Load Balancer (34.120.50.10:443)
  │  TLS termination (optional)
  ▼
Ingress Controller Pod (nginx)
  │  Reads Host header + path
  │  Matches Ingress rules
  │  Decides: → users-svc:80
  ▼
users-svc (ClusterIP: 10.96.45.12)
  │  iptables DNAT to real pod IP
  ▼
users pod (10.244.1.15:8080)
  │  Processes request, sends response
  ▼
Response travels back the same path in reverse
```

---

## 7. Practical Examples

### Example 1: Simple single-service Ingress

The simplest case — one service, one hostname, catch-all path:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-svc
            port:
              number: 80
```

All traffic to `myapp.example.com` goes to `myapp-svc`.

### Example 2: Path-based routing (API gateway pattern)

Multiple microservices behind one hostname, split by URL path:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: users-svc
            port:
              number: 80
      - path: /orders(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: orders-svc
            port:
              number: 80
      - path: /payments(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: payments-svc
            port:
              number: 80
```

The `rewrite-target` annotation strips the prefix before forwarding. So `api.example.com/users/123` arrives at the users service as `/123`, not `/users/123`. This matters because the backend service usually doesn't know it's being served under a prefix.

### Example 3: Multi-hostname routing (multiple products)

Different hostnames routing to completely different services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-svc
            port:
              number: 80
```

Three completely separate applications, all served from the same load balancer and IP address. DNS for all three hostnames points to the same load balancer IP.

### Example 4: Default backend (catch-all for unmatched requests)

What happens when someone hits a hostname or path that doesn't match any rule? By default, they get a 404. You can configure a default backend:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: with-default
spec:
  defaultBackend:
    service:
      name: default-svc
      port:
        number: 80
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: users-svc
            port:
              number: 80
```

Any request that doesn't match `api.example.com/users` goes to `default-svc`. This is useful for serving a custom 404 page or redirecting to your main app.

---

## 8. TLS / HTTPS with Ingress

In production, all external traffic should be HTTPS. Ingress handles this through TLS configuration.

### Step 1: Create or obtain a TLS certificate

You need a certificate and private key. You can get these from Let's Encrypt (free, automated), your company's certificate authority, or a paid provider.

### Step 2: Store the certificate as a Kubernetes Secret

```bash
kubectl create secret tls my-tls-cert \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

This creates a Secret in your cluster containing the certificate and private key.

### Step 3: Reference the Secret in your Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - api.example.com
    - dashboard.example.com
    secretName: my-tls-cert
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
  - host: dashboard.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dashboard-svc
            port:
              number: 80
```

The `tls` section tells the Ingress Controller: "Use the certificate from `my-tls-cert` Secret for these hostnames." The `ssl-redirect` annotation tells it to redirect HTTP to HTTPS automatically.

### Automating certificate management with cert-manager

In practice, nobody manually creates and rotates certificates. You install cert-manager — a Kubernetes controller that automatically provisions and renews certificates from Let's Encrypt:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auto-tls-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-auto    # cert-manager creates this automatically
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
```

cert-manager sees this annotation, requests a certificate from Let's Encrypt, stores it as a Secret, and renews it before it expires. You never manually touch certificates again.

---

## 9. Production Concerns at the Ingress Layer

The Ingress Controller is the front door to your cluster. Several important production concerns live here:

### Rate limiting

Protect your services from abuse. The Ingress Controller can throttle requests per IP before they reach your application:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"          # 10 requests per second per IP
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"  # allow burst of 50
```

### Request size limits

Prevent users from sending massive payloads that could overwhelm your services:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"    # max 10MB request body
```

### Timeouts

Control how long the Ingress Controller waits for your backend to respond:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "5"     # 5 seconds to connect
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"       # 60 seconds to respond
```

### CORS headers

If your frontend is on a different domain than your API, the Ingress Controller can handle CORS:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE"
```

### Authentication

The Ingress Controller can enforce authentication before requests reach your services. External auth example:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-url: "https://auth.example.com/verify"
    nginx.ingress.kubernetes.io/auth-signin: "https://auth.example.com/login"
```

Every request first goes to your auth service. Only if it returns 200 does the request proceed to the backend.

### Canary deployments

Some Ingress Controllers support traffic splitting for gradual rollouts:

```yaml
# Main Ingress (serves 95% of traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-main
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 80

---
# Canary Ingress (serves 5% of traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "5"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 80
```

5% of traffic goes to v2. If metrics look good, you gradually increase the weight. If something breaks, set it to 0. This is more controlled than Kubernetes' built-in rolling update.

### Observability

The Ingress Controller is the single point through which all external traffic flows. This makes it the natural place to collect:

- **Access logs** — who is hitting what, from where, how often
- **Latency metrics** — how long each backend takes to respond
- **Error rates** — which services are returning 5xx errors
- **Traffic volume** — requests per second per service

Most Ingress Controllers expose Prometheus metrics out of the box. This data is invaluable for debugging production issues and for capacity planning discussions.

---

## 10. Choosing an Ingress Controller

| Controller | What it is | Best for | Notes |
|---|---|---|---|
| NGINX Ingress Controller | nginx configured from Ingress rules | General purpose, most teams | Most widely used, extensive annotation support |
| AWS ALB Ingress Controller | Provisions AWS Application Load Balancers | EKS-heavy teams | Uses AWS-native load balancing, supports WAF integration |
| Traefik | Lightweight reverse proxy | Smaller setups, auto-discovery | Built-in Let's Encrypt support, nice dashboard |
| HAProxy Ingress | HAProxy configured from Ingress rules | High-performance needs | Lower latency than nginx for some workloads |
| Istio Gateway | Part of the Istio service mesh | Teams already using Istio | More powerful but more complex, uses Envoy proxy |
| Kong Ingress | Kong API gateway | Teams needing API gateway features | Built-in plugins for auth, rate limiting, transforms |

For most teams starting out, the NGINX Ingress Controller is the safe default. It's well-documented, widely used, and has annotations for almost everything you'd need. Switch to something else only when you hit a specific limitation.

### Key thing to remember

The Ingress object YAML is the same regardless of which controller you use — the `spec.rules` section is standard Kubernetes. The differences are in the annotations (controller-specific configuration) and the features each controller supports beyond the basic spec.

---

## Quick Reference: The Three Service Types + Ingress

| Type | What it does | External access | Use case |
|---|---|---|---|
| ClusterIP | Stable internal IP + DNS | No | Service-to-service within cluster |
| NodePort | ClusterIP + opens port on every node | Yes, via node IP:port | Development, testing |
| LoadBalancer | NodePort + cloud load balancer | Yes, via LB public IP | Single service needing external access |
| Ingress | Shared LB + host/path routing | Yes, via shared LB IP | Multiple services, production HTTP traffic |

Each type builds on the previous one. Ingress uses a LoadBalancer Service (for the controller), which uses NodePort, which uses ClusterIP. Understanding this stack is what lets you debug networking issues in production — you know exactly which layer to inspect.

---

*Last updated: April 2026*