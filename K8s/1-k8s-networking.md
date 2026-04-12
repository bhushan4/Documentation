# Kubernetes Networking: How Traffic Reaches Your Pods

A ground-up guide covering everything that happens when traffic needs to reach your application inside Kubernetes — from the fundamental problem of ephemeral pod IPs to Services, ClusterIP, DNS resolution, iptables routing, and the Endpoints controller that keeps it all in sync.

---

## Table of Contents

1. [The Problem: Pod IPs Are Throwaway](#1-the-problem-pod-ips-are-throwaway)
2. [The Solution: Services](#2-the-solution-services)
3. [How Traffic Routes to the Right Pod — Step by Step](#3-how-traffic-routes-to-the-right-pod--step-by-step)
4. [The Endpoints Controller: Keeping It All In Sync](#4-the-endpoints-controller-keeping-it-all-in-sync)
5. [The Complete Chain: How Everything Connects](#5-the-complete-chain-how-everything-connects)

---

## 1. The Problem: Pod IPs Are Throwaway

After a Deployment creates pods, each one gets a unique cluster IP:

```
Pod my-api-xk9p2  →  IP: 10.244.1.15
Pod my-api-ab3m7  →  IP: 10.244.2.22
Pod my-api-qr8n1  →  IP: 10.244.1.18
```

You might think another service (say a frontend) could just call `http://10.244.1.15:8080` to reach your API. Technically it works — right now. But these IPs are not stable:

- If a pod crashes and kubelet restarts it, it gets a **new IP**.
- If you scale from 3 to 5 replicas, the new pods get **new IPs**.
- If a rolling update happens, old pods die and new ones come up with **different IPs**.
- You have 3 pods — which one should the frontend call? You need **load balancing** across all of them.

Pod IPs are like phone numbers that change every few hours. You can never hardcode them. You need a stable address — like a phone directory entry — that always points to the right set of healthy pods, no matter how many there are or how often they restart.

---

## 2. The Solution: Services

A Service is a stable network endpoint that sits in front of a group of pods. You create it the same way you create anything in Kubernetes — with a YAML file and `kubectl apply`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-api-svc
spec:
  selector:
    app: my-api          # "find all pods with this label"
  ports:
  - port: 80             # the port the Service listens on
    targetPort: 8080     # the port on the actual pods
  type: ClusterIP
```

When you apply this, the API server validates it and writes it to etcd (same pipeline as any other Kubernetes object). The Service gets assigned a **ClusterIP** — a virtual IP address like `10.96.45.12`.

### What "virtual" means

This IP doesn't belong to any machine. No NIC has it. No pod has it. No network interface anywhere is configured with this address. It's a fiction maintained entirely by networking rules (iptables) on every node. Despite being fictional, it works — any pod in the cluster can send traffic to `10.96.45.12:80` and it will arrive at one of the real pods behind the Service.

### The port mapping

Notice the two port numbers:

```yaml
ports:
- port: 80             # external-facing: what other pods use to reach the Service
  targetPort: 8080     # internal: the port your actual containers listen on
```

Other pods call `my-api-svc:80`. The Service translates that to port `8080` on the target pod. This decoupling means your pods can listen on whatever port they want internally — the Service presents a clean, standard port to the rest of the cluster.

### DNS — you don't even need the IP

Kubernetes runs a DNS service (CoreDNS) inside the cluster. Every Service automatically gets a DNS entry:

```
my-api-svc                              → 10.96.45.12  (within same namespace)
my-api-svc.default                      → 10.96.45.12  (namespace explicit)
my-api-svc.default.svc.cluster.local    → 10.96.45.12  (fully qualified)
```

Every pod's `/etc/resolv.conf` is configured to use CoreDNS as its DNS server. So your frontend code can just do `http://my-api-svc:80` — the OS resolves the name to the ClusterIP automatically.

---

## 3. How Traffic Routes to the Right Pod — Step by Step

Here's exactly what happens when your frontend pod sends a request to `http://my-api-svc:80`.

### Step 1: DNS resolves the name to the ClusterIP

The frontend application code makes an HTTP request to `my-api-svc:80`. The operating system's DNS resolver (inside the pod's network namespace) sends a DNS query to CoreDNS. CoreDNS looks up the Service and responds:

```
my-api-svc.default.svc.cluster.local → 10.96.45.12
```

The frontend now has an IP address to connect to.

### Step 2: The packet leaves the frontend pod

The frontend creates a TCP connection. The outgoing packet looks like:

```
Source:      10.244.2.5:43210    (frontend pod's IP, random ephemeral source port)
Destination: 10.96.45.12:80     (the ClusterIP — a virtual address)
```

The packet leaves the pod's network namespace and enters the node's network stack.

### Step 3: iptables intercepts and rewrites the packet (DNAT)

This is where the magic happens. `kube-proxy` — which runs on every node — has been watching the API server for Service and Endpoints changes. When you created the Service and when pods became ready, kube-proxy programmed iptables rules on every node in the cluster.

These rules say:

```
IF destination is 10.96.45.12:80 THEN
  randomly pick ONE of these real pod IPs:
    → 10.244.1.15:8080  (pod 1)
    → 10.244.2.22:8080  (pod 2)
    → 10.244.1.18:8080  (pod 3)
  rewrite the packet's destination to the chosen pod
```

The kernel intercepts the packet before it goes anywhere on the network. It rewrites the destination from the virtual `10.96.45.12:80` to a real pod address like `10.244.2.22:8080`. This is called DNAT — Destination Network Address Translation. It's the same concept Docker uses for port mapping (`-p 80:80`), but applied cluster-wide by kube-proxy.

The frontend pod has no idea this happened. From its perspective, it sent a packet to `10.96.45.12` and got a response from `10.96.45.12`. The rewriting is invisible.

### Step 4: The packet reaches the actual pod

After the destination is rewritten to a real pod IP, the packet is routed through the cluster network:

- **If the target pod is on the same node:** The packet goes through the local bridge/virtual network and arrives at the pod's network namespace directly. No physical network hop.
- **If the target pod is on a different node:** The packet travels across the cluster network. The CNI plugin (Calico, Cilium, Flannel, AWS VPC CNI) handles this — it knows how to route packets between nodes to reach any pod IP in the cluster. The packet arrives at the target node, enters that node's network stack, and is delivered to the target pod's network namespace.

The application inside the target pod receives the request on port 8080, processes it, and sends a response. The response travels the reverse path — iptables on the original node rewrites the source address back to `10.96.45.12:80` so the frontend sees a consistent address.

### Why kube-proxy programs EVERY node

You might wonder: why does kube-proxy put these iptables rules on all nodes, not just the node where the frontend pod runs?

Because pods can be scheduled on any node. Tomorrow the frontend might get rescheduled to a different node after a rolling update. That new node needs to already have the iptables rules in place. By programming every node, Kubernetes ensures that no matter where a pod runs, it can always reach any Service.

### The load balancing is random (by default)

The iptables rules use probability-based selection. With 3 pods, each gets roughly 1/3 of the traffic. This is simple but effective for most workloads. It's not round-robin, it's not least-connections, it's just random distribution. For more sophisticated load balancing, you'd use a service mesh like Istio or Linkerd, which replaces this mechanism with proxy-based routing.

---

## 4. The Endpoints Controller: Keeping It All In Sync

The iptables rules need an up-to-date list of healthy pod IPs. Who maintains that list? The **Endpoints controller** — another one of those independent controllers watching etcd and reacting.

It watches two things simultaneously:

1. **The Service** — specifically its `selector` field (`app: my-api`)
2. **All pods in the cluster** — specifically their labels and readiness status

It continuously asks: "Which pods match this Service's selector AND have `ready: true`?" The answer becomes an **Endpoints object**:

```yaml
kind: Endpoints
metadata:
  name: my-api-svc          # same name as the Service
subsets:
- addresses:
  - ip: 10.244.1.15         # pod 1 — ready
    targetRef:
      kind: Pod
      name: my-api-7f8b4c2d-xk9p2
  - ip: 10.244.2.22         # pod 2 — ready
    targetRef:
      kind: Pod
      name: my-api-7f8b4c2d-ab3m7
  - ip: 10.244.1.18         # pod 3 — ready
    targetRef:
      kind: Pod
      name: my-api-7f8b4c2d-qr8n1
  ports:
  - port: 8080
```

### The live update chain

Here's how everything stays current:

1. **A pod becomes ready** (readiness probe passes) → kubelet updates the pod's status in the API server → the Endpoints controller sees the change → adds the pod's IP to the Endpoints object → kube-proxy sees the updated Endpoints → reprograms iptables on every node → traffic starts flowing to the new pod.

2. **A pod fails its readiness probe** (database connection drops, dependency becomes unhealthy) → kubelet updates `ready: false` → Endpoints controller removes the pod's IP from the list → kube-proxy updates iptables → traffic stops going to that pod. The pod is NOT killed (that's the liveness probe's job). It just stops receiving traffic until it becomes ready again.

3. **A pod is killed** (crash, rolling update, scale-down) → pod is deleted from etcd → Endpoints controller removes its IP → kube-proxy updates iptables → traffic is redistributed among remaining pods.

4. **A new pod comes up** (scale-up, rolling update replacement) → goes through steps 1-7 of the deployment flow → readiness probe passes → added to Endpoints → starts receiving traffic.

This is why the readiness probe from step 7 of the deployment flow matters so much. It's the gate that controls whether a pod's IP appears in the Endpoints list, which controls whether iptables sends any traffic to it. A pod can be `Running` (container is alive) but not `Ready` (not receiving traffic). The distinction is critical.

### What the Endpoints object looks like during a rolling update

During a rolling update from v1 to v2, the Endpoints list changes in real time:

```
Time 0 (before update):
  Endpoints: [pod-v1-a, pod-v1-b, pod-v1-c]     ← all v1

Time 1 (v2 pod starts, readiness passes):
  Endpoints: [pod-v1-a, pod-v1-b, pod-v1-c, pod-v2-a]  ← mixed

Time 2 (one v1 pod killed):
  Endpoints: [pod-v1-b, pod-v1-c, pod-v2-a]      ← mixed

Time 3 (another v2 up, another v1 down):
  Endpoints: [pod-v1-c, pod-v2-a, pod-v2-b]       ← mixed

Time 4 (update complete):
  Endpoints: [pod-v2-a, pod-v2-b, pod-v2-c]       ← all v2
```

From the frontend's perspective, nothing changed. It's still calling `my-api-svc:80`. The Service's ClusterIP didn't change. The DNS entry didn't change. Only the backend pod IPs in the iptables rules shifted — completely invisible to the caller. This is how zero-downtime deployments work.

---

## 5. The Complete Chain: How Everything Connects

Here's the full picture of how every component works together to route in-cluster traffic:

```
Frontend pod code:  fetch("http://my-api-svc:80/data")
         │
         ▼
    Pod's /etc/resolv.conf points to CoreDNS
         │
         ▼
    CoreDNS responds: my-api-svc → 10.96.45.12  (ClusterIP)
         │
         ▼
    Pod creates packet: dst = 10.96.45.12:80
         │
         ▼
    Packet enters node's network stack
         │
         ▼
    iptables rule (programmed by kube-proxy):
    "10.96.45.12:80 → randomly pick from Endpoints list"
         │
         ▼
    Packet rewritten: dst = 10.244.2.22:8080  (a real pod)
         │
         ▼
    CNI plugin routes packet to the target node (if different)
         │
         ▼
    Pod receives request on port 8080, sends response
         │
         ▼
    Response travels reverse path, iptables rewrites source
    back to 10.96.45.12:80 so frontend sees consistent address
```

### What keeps each piece updated

| Component | What it watches | What it does |
|---|---|---|
| CoreDNS | Service objects in API server | Maintains DNS entries for each Service |
| Endpoints controller | Services + Pods (labels + readiness) | Maintains list of healthy pod IPs per Service |
| kube-proxy | Service + Endpoints objects | Programs iptables rules on every node |
| iptables (kernel) | Nothing — it's just rules | Rewrites packet destinations based on rules |
| CNI plugin | Pod creation/deletion events | Assigns pod IPs, sets up routes between nodes |

### The reconciliation pattern (again)

This is the same pattern from the deployment flow. No orchestrator coordinates these components. Each one independently watches for changes and reacts:

- Endpoints controller doesn't tell kube-proxy "hey, I updated the list." It just writes the new Endpoints object to the API server.
- kube-proxy is watching Endpoints objects. It sees the change and reprograms iptables.
- If kube-proxy crashes and restarts, it reads the current Endpoints from the API server and rebuilds all iptables rules from scratch.

Everything is driven by desired state in etcd, watched by independent controllers. The same pattern, applied to networking.

---

### What we haven't covered yet

Everything above is about **internal cluster traffic** — pod to pod. For traffic from the outside world (a user in their browser), you need:

- **NodePort Service** — opens a port on every node in the cluster. External traffic hits any node's IP on that port and gets routed to a pod. Simple but limited.
- **LoadBalancer Service** — provisions a cloud load balancer (ALB on AWS, Cloud Load Balancer on GCP) that sends traffic to your pods. The standard approach for production.
- **Ingress** — a higher-level abstraction that routes HTTP traffic based on hostname and URL path. "Requests to `api.mycompany.com/users` go to the users Service, requests to `api.mycompany.com/orders` go to the orders Service." This is what most production setups use for HTTP traffic.

---

*Last updated: April 2026*