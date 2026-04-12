# Kubernetes Deployment: The Complete Mental Model

A ground-up guide covering everything that happens when you deploy an application to Kubernetes — from the Linux kernel to a running, healthy pod serving traffic. Written for experienced software engineers building staff-level systems understanding.

---

## Table of Contents

1. [Foundation: The Linux Kernel](#1-foundation-the-linux-kernel)
2. [Foundation: Containers — Isolated Filesystem and Network](#2-foundation-containers--isolated-filesystem-and-network)
3. [Foundation: What is a Kubernetes Cluster?](#3-foundation-what-is-a-kubernetes-cluster)
4. [The Deployment Flow: 7 Steps](#4-the-deployment-flow-7-steps)
   - [Step 1: You Write a YAML File](#step-1-you-write-a-yaml-file)
   - [Step 2: kubectl apply — The File Reaches the API Server](#step-2-kubectl-apply--the-file-reaches-the-api-server)
   - [Step 3: The Deployment Controller Wakes Up](#step-3-the-deployment-controller-wakes-up)
   - [Step 4: The ReplicaSet Controller Creates Pod Objects](#step-4-the-replicaset-controller-creates-pod-objects)
   - [Step 5: The Scheduler Assigns Pods to Nodes](#step-5-the-scheduler-assigns-pods-to-nodes)
   - [Step 6: Kubelet Starts the Containers](#step-6-kubelet-starts-the-containers)
   - [Step 7: Health Checks — The Pod Proves It's Ready](#step-7-health-checks--the-pod-proves-its-ready)
5. [Deep Dives](#5-deep-dives)
   - [etcd: The Cluster's Brain](#etcd-the-clusters-brain)
   - [Resource Management: Requests, Limits, and QoS](#resource-management-requests-limits-and-qos)
   - [Scaling: HPA, VPA, and Cluster Autoscaler](#scaling-hpa-vpa-and-cluster-autoscaler)
   - [Rolling Updates: What Actually Happens](#rolling-updates-what-actually-happens)
6. [Glossary: Cluster vs Node vs Pod vs Deployment](#6-glossary-cluster-vs-node-vs-pod-vs-deployment)
7. [The One Mental Model That Ties Everything Together](#7-the-one-mental-model-that-ties-everything-together)

---

## 1. Foundation: The Linux Kernel

Before you can understand containers or Kubernetes, you need to understand what the Linux kernel is and why it exists.

### The Problem It Solves

Your computer has hardware: CPU, RAM, disk, network card. And you have programs that want to use that hardware: a web browser, nginx, a Python script, a database.

You can't let programs talk directly to hardware. If your Python script could write to any memory address it wanted, it could overwrite another program's data or crash the whole machine. If two programs tried to write to the disk at the same time without coordination, they'd corrupt each other's files.

The kernel is the piece of software that sits between programs and hardware, managing all access. It's the first thing that runs when you power on the machine, and it stays running the entire time. Every other program runs on top of it.

### The Four Big Jobs

**1. Process Scheduling — "Who gets the CPU right now?"**

Your machine might have 4 CPU cores but 200 running processes. They can't all run at once. The kernel's scheduler gives each process a tiny slice of CPU time (a few milliseconds), then pauses it and gives the next process a turn. It switches so fast — thousands of times per second — that every program thinks it has the CPU to itself. When you run `top` and see a process using "25% CPU," that means the scheduler is giving it one full core's worth of time slices.

**2. Memory Management — "Each program gets its own address space"**

When your Python app allocates a variable, it gets a memory address like `0x7ffd4a2b`. But that's a virtual address, not a real physical location in RAM. The kernel maintains a mapping table (called page tables) that translates virtual addresses to physical RAM addresses. Each process gets its own virtual address space, so process A's address `0x1000` maps to a completely different physical location than process B's `0x1000`. They can't see or corrupt each other's memory. If a process tries to access memory it hasn't been given, the kernel kills it immediately — that's the "segmentation fault" error.

**3. Filesystem — "Files are an abstraction over raw disk blocks"**

A physical SSD doesn't know what a "file" is. It just has billions of numbered blocks (usually 4KB each) that you can read and write. The kernel's filesystem layer creates the illusion of files and directories on top of this. When you do `open("/etc/config.txt")`, the kernel looks up the file's metadata (called an inode), finds which disk blocks contain the data, reads those blocks from the SSD, and hands the data back to your program. Your program never touches the disk directly.

**4. Network Stack — "Speaking TCP/IP to the world"**

When nginx wants to listen on port 80, it doesn't configure the network card directly. It asks the kernel to create a socket — an endpoint for network communication. The kernel's network stack handles everything: assembling IP packets, managing TCP connections (handshake, retransmission, ordering), routing packets to the right interface. When a packet arrives from the internet, the network card interrupts the kernel, the kernel processes the packet through its TCP/IP stack, matches it to the right socket, and wakes up the program that's waiting for data on that socket.

### How Programs Talk to the Kernel — System Calls

Programs can't just reach into kernel memory. There's a hard boundary enforced by the CPU itself. The CPU has two modes:

- **Kernel mode (ring 0):** Full access to everything — all memory, all hardware, all instructions. Only the kernel runs here.
- **User mode (ring 3):** Restricted. Can't touch hardware, can't access other programs' memory, can't execute privileged instructions.

When your program needs the kernel to do something — read a file, open a network connection, create a new process — it makes a system call. This is a special CPU instruction that temporarily switches from user mode to kernel mode, runs the kernel's code for that operation, then switches back. Common system calls:

- `open()` — open a file
- `read()` / `write()` — read from or write to a file or socket
- `fork()` — create a new process (a copy of the current one)
- `exec()` — replace the current process with a new program
- `socket()` — create a network endpoint
- `mmap()` — map a file into memory

Every time you write `file = open("data.txt")` in Python, Python calls the C library, which calls the `open()` system call, which switches into kernel mode, which looks up the file, allocates a file descriptor, and returns it to your program. All of that happens in microseconds.

### There Is Only One Linux Kernel

There's one codebase, maintained by one project, at kernel.org. Linus Torvalds started it in 1991 and still oversees it. When people say "Linux," they mean this kernel. Everything else — Ubuntu, Debian, Red Hat, Alpine — those are distributions, not different kernels. A distribution is a packaging of the Linux kernel plus a bunch of userspace tools, package managers, default configurations, and libraries on top.

| Distribution | Package Manager | C Library | Base Image Size | Init System |
|---|---|---|---|---|
| Ubuntu | apt | glibc | ~800MB | systemd |
| Alpine | apk | musl | ~5MB | OpenRC |
| Red Hat (RHEL) | yum/dnf | glibc | ~200MB | systemd |

They all run the same kernel — different versions of it, but the same codebase. The distribution maintainers choose which kernel version to ship. Cloud providers (AWS, GCP, Azure) add another layer — they pick node images with specific kernel versions tested for their platform.

### Why This Matters for Containers

Containers are not virtual machines. They don't run their own kernel. Every container on a host shares the same Linux kernel. What makes containers isolated is that the kernel has features specifically designed to partition its own resources:

- **Namespaces** — the kernel gives a process a restricted view. A PID namespace makes the process think it's PID 1 (the only process). A network namespace gives it its own IP and ports. A mount namespace gives it its own filesystem tree. The process calls the same system calls, the same kernel handles them — but the kernel returns different answers depending on which namespace the process is in.
- **Cgroups (control groups)** — the kernel limits how much resource a process gets. "This group of processes can use at most 2 CPU cores and 1GB of RAM." This is exactly what Kubernetes resource `requests` and `limits` translate into.

---

## 2. Foundation: Containers — Isolated Filesystem and Network

### What "Isolated Filesystem" Means

A container gets its own root filesystem — a completely separate directory tree that looks like a full Linux system to the process inside it. The process sees `/bin`, `/etc`, `/usr`, `/tmp` — but these are not the host's real directories. They come from a container image (like `nginx:1.21`), which is essentially a tarball of a pre-built filesystem.

When your containerised nginx writes to `/var/log/nginx/access.log`, it's writing to a file that only exists inside that container's filesystem. The host machine and other containers can't see it (unless you explicitly mount a shared volume). And the container can't see or modify the host's `/etc/passwd` — it's sandboxed.

Under the hood, this uses Linux mount namespaces combined with overlay filesystems. The image layers are read-only, and the container gets a thin writable layer on top. When it modifies a file, it copies that file into the writable layer (copy-on-write). 10 containers from the same image share the read-only layers; only the diffs are unique.

### What "Isolated Network" Means

Each container gets its own network namespace — its own set of network interfaces, its own IP address, its own port space.

Without isolation, if you run three copies of nginx on a host, they'd all fight over port 80. With network namespaces, each container thinks it has the entire port space to itself. All three can bind to port 80 inside their own namespace because they each have a separate network stack.

### Key Networking Concepts

**NIC (Network Interface Card):** The hardware (or virtual equivalent) that connects a machine to a network. Think of it as the mailbox of your machine — it has a network address and can send/receive data. Your laptop's WiFi chip is a NIC. Containers get virtual NICs — software-based interfaces that behave like real ones but have no physical cable.

**Port:** A numbered address within a single machine's network stack. It identifies which program should receive incoming data. The IP address gets the packet to the right machine; the port number gets it to the right application on that machine. There are 65,535 ports available. Well-known ones: port 80 (HTTP), port 443 (HTTPS), port 22 (SSH), port 5432 (PostgreSQL).

**Bridge Network:** A virtual network switch that lives inside the Linux kernel. Docker creates one called `docker0`. It connects multiple virtual NICs so containers can talk to each other, just like a physical switch in a server room connects multiple cables.

**Can two containers have the same IP?** On the same host — no. Each gets a unique IP from the bridge's subnet. On different hosts — yes, which is a real problem. Kubernetes solves this by requiring every pod to get a cluster-wide unique IP.

### What Happens When a User Types a URL That Reaches Your Container

Let's say your VM's public IP is `34.120.50.10`, and you started a container with `docker run -p 80:80 nginx`.

1. **Browser creates a TCP connection** to `34.120.50.10:80`. The packet travels across the internet and arrives at your VM's physical NIC.
2. **VM kernel receives the packet.** Nothing on the host is listening on port 80.
3. **iptables NAT kicks in.** When you started the container with `-p 80:80`, Docker added an iptables rule: "any packet arriving at host port 80, rewrite the destination to `172.17.0.2:80` and forward it." The kernel rewrites the packet's destination.
4. **Packet crosses the bridge.** The rewritten packet goes through `docker0`, down the veth pair (virtual ethernet cable), and arrives at the container's virtual eth0 interface.
5. **nginx receives the request** inside the container, processes it, and sends the response back.
6. **Response goes back** the reverse path. iptables rewrites the source IP back to `34.120.50.10` so the browser doesn't see the internal `172.17.0.x` address.

The `-p 80:80` flag is the port mapping. Format: `hostPort:containerPort`. Without it, there's no iptables rule and the container is invisible from outside. You can map different host ports to run multiple containers that all use port 80 internally:

```
Container A (nginx):  -p 8080:80   → host:8080 → container:80
Container B (nginx):  -p 8081:80   → host:8081 → container:80
Container C (api):    -p 3000:8080 → host:3000 → container:8080
```

---

## 3. Foundation: What is a Kubernetes Cluster?

### The Concept

A "cluster" is a group of machines working together as one system, running the Kubernetes software. It consists of:

- One or more **control plane machines** (running API server, etcd, scheduler, controller manager)
- One or more **worker node machines** (running kubelet, your application pods)
- A **network** connecting them all

### Managed vs Self-Hosted

You can get a Kubernetes cluster in several ways:

| Method | Who manages control plane | Example |
|---|---|---|
| Cloud managed | Cloud provider | EKS (AWS), GKE (Google), AKS (Azure) |
| Self-hosted | Your team | kubeadm on bare metal or VMs |
| Local dev | Your laptop | minikube, kind, Docker Desktop |

EKS, GKE, and AKS are managed Kubernetes services. "Managed" means the cloud provider runs the control plane — the API server, etcd, scheduler, controllers. You don't see these machines, don't SSH into them, don't back up etcd yourself. The provider handles all of that.

The Kubernetes API, YAML files, kubectl commands — all identical across these. If you learn Kubernetes on EKS, you can walk into a company using GKE and everything works the same way.

Companies choose managed because running the control plane yourself is genuinely hard: etcd backups, version upgrades without downtime, certificate rotation, high availability. Managed services handle all of that for a per-cluster hourly fee.

### The Architecture

```
Control Plane (the brain):
├── API Server      — Central gateway. Only component that talks to etcd.
│                     Everything goes through it: kubectl, scheduler, controllers, kubelet.
├── etcd            — Distributed key-value store. Holds ALL cluster state.
├── Scheduler       — Watches for unassigned pods, picks the best node for each one.
└── Controller Mgr  — Runs dozens of control loops. Each watches a resource type
                      and reconciles desired state with actual state.

Worker Nodes (the muscle):
├── Kubelet         — Agent on every node. Watches for pods assigned to its node,
│                     tells containerd to start/stop containers.
├── Kube-proxy      — Handles networking rules for Service routing.
└── Container runtime (containerd) — Actually runs containers.
```

---

## 4. The Deployment Flow: 7 Steps

Everything below traces what happens from the moment you decide to deploy an application to the moment it's serving real user traffic.

---

### Step 1: You Write a YAML File

Everything in Kubernetes starts with a declaration. You write a file that describes what you want, not how to do it.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-api
  template:
    metadata:
      labels:
        app: my-api
    spec:
      containers:
      - name: my-api
        image: my-company/api-server:v2.1
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "500m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
```

In plain English: "I want 3 copies of my container `my-company/api-server:v2.1` running at all times. Each one listens on port 8080. Each one needs at least half a CPU core and 256MB of memory guaranteed, and should never use more than 1 core or 512MB."

This file doesn't do anything yet. It's just a declaration sitting on your laptop. Nothing is running. You're not telling Kubernetes "start 3 containers on node X." You're saying "I want 3 of these to exist" and Kubernetes figures out the rest.

**Important:** This YAML is an application-level configuration (Layer 2). The cluster infrastructure — how many nodes, how many etcd instances, what CNI plugin to use — is configured at a completely different level (Layer 1), usually via Terraform, cloud console, or kubeadm, before any application is deployed.

---

### Step 2: kubectl apply — The File Reaches the API Server

When you type `kubectl apply -f deployment.yaml`, here's what happens:

**kubectl** reads your YAML file, converts it to JSON, and sends an HTTPS POST request to the Kubernetes API server:

```
POST https://your-cluster.example.com:6443/apis/apps/v1/namespaces/default/deployments
Authorization: Bearer <your-token>
Content-Type: application/json
Body: { ...your YAML converted to JSON... }
```

**The API server** receives this request and runs it through a pipeline of checks, in this exact order:

1. **Authentication** — "Who are you?" It checks your token or certificate. If invalid, 401 error. Nothing happens.

2. **Authorization (RBAC)** — "Are you allowed to create a Deployment in this namespace?" Your user or service account needs the right role bindings. If not, 403 Forbidden.

3. **Admission Controllers** — Plugins that can modify or reject your request before it's saved. Examples:
   - `LimitRanger` might inject default resource limits if you didn't set any.
   - `ResourceQuota` might reject the request if the namespace has hit its limits.
   - Custom webhooks might enforce "every container must come from our approved registry."

4. **Validation** — Structural correctness. Does `replicas` have a valid number? Is `image` present? Is `containerPort` an integer?

If all four checks pass, the API server writes your Deployment object to **etcd** — the cluster's key-value store.

kubectl gets back a success response: `deployment.apps/my-api created`.

**Nothing is running yet.** No containers have started. No nodes have been chosen. All that's happened is your desired state has been recorded in etcd. Like filing a work order — the paperwork exists, but no one has started the job.

---

### Step 3: The Deployment Controller Wakes Up

This step is NOT triggered by you or by kubectl. It happens automatically.

The Deployment controller is one of many control loops running inside the controller manager. It has been sitting there since the cluster started, doing one thing: watching the API server for any changes to Deployment objects.

The moment the API server wrote your Deployment to etcd, etcd notified the API server, and the API server notified the Deployment controller: "a new Deployment called `my-api` just appeared."

The Deployment controller reads the object and asks: "Does a ReplicaSet already exist for this Deployment?" Since this is brand new, the answer is no.

The Deployment controller creates a ReplicaSet object:

```yaml
kind: ReplicaSet
metadata:
  name: my-api-7f8b4c2d         # auto-generated name
  ownerReferences:
    - kind: Deployment
      name: my-api               # "I belong to this Deployment"
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: my-api
        image: my-company/api-server:v2.1
```

It sends this to the API server, the API server validates it and writes it to etcd.

**Still nothing is running.** No containers, no node selection. We just have two objects recorded in etcd — a Deployment and a ReplicaSet.

**Why this extra layer?** The answer is rolling updates. When you later deploy v2.2, the Deployment controller creates a second ReplicaSet for v2.2 and gradually scales it up while scaling the v2.1 ReplicaSet down:

```
Deployment: my-api
├── ReplicaSet: my-api-7f8b4c2d (v2.1) → 2 pods (scaling down)
└── ReplicaSet: my-api-9a3e1f5b (v2.2) → 2 pods (scaling up)
```

If v2.2 is broken, the Deployment controller scales the old ReplicaSet back up — that's a rollback. The ReplicaSet layer makes this possible. Each ReplicaSet represents one specific version of your app.

**Why not have the Deployment directly manage pods?** Because each object type has a different controller with a different job. The Deployment controller only manages versions and transitions between them. The ReplicaSet controller (step 4) only manages pod counts. Keeping them separate makes each one simple. Smart behavior emerges from simple controllers chained together through etcd.

---

### Step 4: The ReplicaSet Controller Creates Pod Objects

Same pattern. The ReplicaSet controller has been watching for changes to ReplicaSet objects.

It sees the new ReplicaSet and asks: "How many pods should exist, and how many actually exist?"

```
desired: 3
actual:  0
difference: 3 pods need to be created
```

It creates 3 Pod objects by making 3 API calls:

```yaml
kind: Pod
metadata:
  name: my-api-7f8b4c2d-xk9p2        # random suffix
  labels:
    app: my-api
  ownerReferences:
    - kind: ReplicaSet
      name: my-api-7f8b4c2d           # "I belong to this ReplicaSet"
spec:
  containers:
  - name: my-api
    image: my-company/api-server:v2.1
    ports:
    - containerPort: 8080
    resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
      limits:
        cpu: "1000m"
        memory: "512Mi"
  nodeName: ""                         # empty — no node assigned yet
```

The API server writes these to etcd. We now have 3 Pod objects in **Pending** state. They exist as database records but no machine has been chosen to run them.

Note the ownership chain:

```
Deployment: my-api
  └── owns → ReplicaSet: my-api-7f8b4c2d
                └── owns → Pod: my-api-7f8b4c2d-xk9p2
                └── owns → Pod: my-api-7f8b4c2d-ab3m7
                └── owns → Pod: my-api-7f8b4c2d-qr8n1
```

If you delete the Deployment, Kubernetes garbage-collects the ReplicaSet, which garbage-collects the pods. Everything cleans up because each object knows who created it.

**Four steps in and still zero containers running — just database records in etcd.**

---

### Step 5: The Scheduler Assigns Pods to Nodes

The scheduler watches for pods with `nodeName: ""` — pods that exist but haven't been assigned to a machine.

It picks up each pod and decides where it should run, in two phases:

**Phase 1: Filtering — "Which nodes CAN run this pod?"**

The scheduler eliminates nodes that can't possibly work:

1. **Resource availability** — Your pod requests 500m CPU and 256Mi memory. If a node only has 200m CPU free (based on what's already requested, not what's actually used), it's eliminated.
2. **Taints and tolerations** — GPU nodes might have a taint that says "only GPU workloads." If your pod doesn't tolerate it, eliminated.
3. **Node selectors / affinity** — If your pod says `nodeSelector: disk=ssd`, only nodes with that label survive.
4. **Node health** — If kubelet has stopped reporting or node is `NotReady`, eliminated.
5. **Port conflicts** — If your pod needs a specific host port already in use, eliminated.

If zero nodes survive filtering, the pod stays in Pending. This is exactly what happens when a cluster is fully packed (see Rolling Updates section below).

**Phase 2: Scoring — "Of the eligible nodes, which is best?"**

Each surviving node gets scored 0-100 on:

- **Resource balance** — prefer evening out CPU/memory usage across nodes
- **Spreading** — prefer different nodes/zones for replicas of the same app
- **Affinity** — prefer nodes running related pods if affinity rules are set
- **Least requested** — prefer nodes with more free resources

The node with the highest total score wins.

**The scheduler writes its decision** — it updates the pod, setting `nodeName`:

```yaml
# Before
spec:
  nodeName: ""

# After
spec:
  nodeName: "worker-2"
```

That's all the scheduler does. It doesn't contact the node. It doesn't start any container. It just writes "this pod belongs on worker-2" to etcd through the API server.

**Still no containers running. Just database records with node assignments.**

---

### Step 6: Kubelet Starts the Containers

This is the moment something actually runs on a real machine.

Kubelet is an agent running on every worker node. It watches the API server for pods assigned to its node. When the scheduler wrote `nodeName: "worker-2"`, the kubelet on worker-2 gets notified and works through this sequence:

**1. Pull the container image.**
Kubelet tells containerd to pull `my-company/api-server:v2.1`. If the image isn't cached locally, it downloads from the container registry (Docker Hub, ECR, GCR, or your private registry). This is often the slowest step — a large image might take 30-60 seconds. If the pull fails (wrong image name, auth failure, network issue), the pod goes into `ImagePullBackOff` and kubelet retries with increasing delays.

**2. Set up the pod's network namespace.**
Kubelet calls the CNI (Container Network Interface) plugin (Calico, Cilium, AWS VPC CNI). The plugin creates a new network namespace and assigns a unique cluster-wide IP to the pod.

**3. Set up storage volumes.**
If the pod mounts ConfigMaps, Secrets, or persistent disks, kubelet sets those up now.

**4. Configure cgroups.**
The resource requests and limits from your YAML become real kernel-level enforcement:

```
Your YAML:                     Kernel cgroup:
  requests.cpu: 500m        →  "Guaranteed 0.5 CPU cores"
  limits.cpu: 1000m         →  "Can burst up to 1.0 cores, then throttled"
  requests.memory: 256Mi    →  "Reserved 256Mi"
  limits.memory: 512Mi      →  "If exceeds 512Mi, OOM-killed immediately"
```

This is where the Kubernetes YAML connects directly to the Linux kernel concepts from section 1.

**5. Start the container.**
Kubelet tells containerd: "Run this image with this network namespace, these volumes, this cgroup, this command." Containerd creates the container and starts the process. Your application code is now running.

**6. Kubelet reports back.**
It updates the pod status:

```yaml
status:
  phase: Running
  podIP: 10.244.2.15
  containerStatuses:
  - name: my-api
    state:
      running:
        startedAt: "2026-04-12T14:32:01Z"
    ready: false          # not yet — health checks haven't passed
```

The container is running but `ready: false`. Kubernetes doesn't trust it to receive traffic yet.

---

### Step 7: Health Checks — The Pod Proves It's Ready

The container is running, but your application might need a few seconds to load configuration, connect to a database, warm up a cache. Sending traffic during this window would cause errors.

There are three types of probes, defined in the pod spec:

```yaml
containers:
- name: my-api
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 10
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 3
    periodSeconds: 5
  startupProbe:
    httpGet:
      path: /healthz
      port: 8080
    failureThreshold: 30
    periodSeconds: 5
```

**Readiness Probe — "Should this pod receive traffic?"**

Kubelet hits `http://localhost:8080/ready` every 5 seconds, starting 3 seconds after launch. Your `/ready` endpoint should return 200 only when the app is genuinely ready — database connected, config loaded.

Until it returns 200, the pod stays `ready: false`. Kubernetes keeps a list of ready pod IPs for each Service (called Endpoints). Only pods with `ready: true` get traffic. If readiness fails later (database drops), the pod is removed from the list — traffic stops, but the pod keeps running. When the probe passes again, it's added back.

**Liveness Probe — "Is this pod hopelessly stuck?"**

Kubelet hits `/healthz` every 10 seconds. If it fails several consecutive times (default 3), kubelet kills the container and restarts it. This handles deadlocks, infinite loops, or unrecoverable states.

Critical mistake to avoid: don't make liveness check dependencies. If it checks the database and the database goes down, Kubernetes kills and restarts all your pods. They come back, still can't reach the database, get killed again — a restart loop that makes things worse. Liveness should check "is my process fundamentally healthy," not "can I reach everything I depend on." Save dependency checks for readiness.

**Startup Probe — "Is this pod still booting?"**

Optional but important for slow-starting apps (Java with large classpaths, 60-90 second startup). Tells kubelet: "Don't run liveness or readiness checks until this probe passes." Prevents liveness from killing a container that's just slow to start.

**The moment the pod becomes ready:**

Once readiness returns 200, kubelet updates the pod:

```yaml
status:
  conditions:
  - type: Ready
    status: "True"
```

The Endpoints controller sees this and adds the pod's IP to the Service's endpoint list. Traffic flows.

**Your app is live.**

---

## 5. Deep Dives

### etcd: The Cluster's Brain

etcd is a distributed key-value store. Every single thing Kubernetes knows — every deployment, pod, node, config — is a key-value entry:

```
Key                                        Value
/registry/deployments/default/my-api   →   {full JSON of deployment spec}
/registry/pods/default/my-api-abc123   →   {JSON: status, nodeName, etc.}
/registry/nodes/worker-1               →   {JSON: capacity, conditions}
```

**Why etcd instead of PostgreSQL?**

Three properties matter for a cluster control plane:

1. **Strong consistency.** etcd runs 3 or 5 instances using Raft consensus. When data is written to the leader, it replicates to a majority of followers before the write is confirmed. No stale reads. The scheduler never sees outdated state.

2. **Watch mechanism.** Any component can subscribe to changes on a key prefix and get notified in real time. The scheduler watches for unassigned pods. Kubelet watches for pods on its node. Controllers watch their resource types. No polling — etcd pushes changes instantly. This is what makes the whole reconciliation architecture possible.

3. **Simplicity and speed.** Only key-value operations: get, put, delete, list by prefix, watch. No SQL, no joins. Reads take microseconds.

**How Raft consensus works:**

One instance is elected leader. All writes go through the leader:

1. Leader writes entry to its own log (not committed yet)
2. Leader sends entry to both followers
3. Each follower records it and replies "got it"
4. Once a majority confirms (leader + at least 1 follower = 2 of 3), the write is committed
5. Leader tells followers to apply it
6. Leader responds to the API server: "success"

This is why you use odd numbers (3 or 5). With 3 instances, you can lose 1 and still have a majority (2). With 5, you can lose 2. With an even number like 4, you can still only lose 1 (need 3 for majority), so 4 gives no advantage over 3 but costs more.

**Where etcd config lives:** On managed Kubernetes (EKS, GKE, AKS), you never see it — the provider handles it. On self-hosted clusters, it's in files on the control plane nodes (e.g., `/etc/kubernetes/manifests/etcd.yaml`), completely separate from your application YAML.

**Only the API server talks to etcd directly.** Everything else goes through the API server, which handles auth, authorization, and validation as a gatekeeper.

---

### Resource Management: Requests, Limits, and QoS

**Requests** are guarantees. The scheduler uses requests to decide if a pod fits on a node. A node with 4 CPU cores and 3.5 already requested will only accept a pod requesting 0.5 or less. The capacity is reserved whether or not the pod actually uses it.

**Limits** are ceilings. If a pod exceeds its memory limit, the kernel's OOM killer terminates it immediately. CPU limits work differently — the kernel throttles the process using CFS bandwidth control. The pod slows down but doesn't die.

**Critical asymmetry: CPU is compressible, memory is not.** Hit CPU limit → runs slower (bad but survivable). Hit memory limit → killed and restarted. Be generous with memory limits and tighter with CPU.

```yaml
resources:
  requests:
    cpu: "500m"       # 0.5 cores guaranteed
    memory: "256Mi"   # 256MB guaranteed
  limits:
    cpu: "1000m"      # can burst to 1.0 core
    memory: "512Mi"   # killed if exceeds 512MB
```

**QoS Classes** — assigned automatically based on how you set requests and limits:

| Class | Condition | Eviction Priority |
|---|---|---|
| Guaranteed | requests == limits for both CPU and memory | Last to be evicted |
| Burstable | requests < limits | Evicted before Guaranteed |
| BestEffort | No requests or limits set | First to die under pressure |

Never run BestEffort in production.

**Node allocatable vs capacity:** A node with 4 CPU cores doesn't give you 4 for pods. Kubelet reserves resources for itself (`--kube-reserved`), the OS (`--system-reserved`), and an eviction threshold buffer. Actual allocatable might be ~3.5 cores. Account for this in capacity planning.

**Resource Quotas and LimitRanges:** ResourceQuota caps total requests/limits across all pods in a namespace. LimitRange sets defaults and maximums per pod. These prevent one team from consuming the whole cluster and protect against developers forgetting to set resource specs.

---

### Scaling: HPA, VPA, and Cluster Autoscaler

**Horizontal Pod Autoscaler (HPA)** — adjusts replica count based on metrics (CPU, memory, custom metrics). Runs every 15 seconds (default). Formula:

```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / targetMetricValue))
```

If you have 3 pods at 80% CPU and your target is 50%: `ceil(3 × 80/50) = ceil(4.8) = 5`. HPA scales to 5 replicas. New pods go through the same step 3-7 flow.

**Vertical Pod Autoscaler (VPA)** — adjusts CPU/memory requests on existing pods. Trickier because you can't change a running pod's resources — VPA evicts and recreates it with new values.

**Cluster Autoscaler** — adds or removes nodes (actual machines). Watches for pods stuck in Pending because no node has capacity. Provisions a new VM from the cloud provider (takes 1-3 minutes). If a node is underutilized for a sustained period, it drains pods and terminates the node to save cost.

---

### Rolling Updates: What Actually Happens

When you deploy a new version, the Deployment has a rollout strategy (defaults shown):

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%          # extra pods above desired count allowed temporarily
    maxUnavailable: 25%    # pods that can be down during update
```

**The danger scenario — a fully packed cluster:**

```
Cluster: 2 CPU cores total
Pod A (v1): requests 1 core
Pod B (v1): requests 1 core
Available: 0 cores
```

With default strategy (maxSurge=1, maxUnavailable=0 for 2 replicas):
- Kubernetes tries to create a v2 pod first (surge), but there's no room.
- It can't kill a v1 pod because maxUnavailable=0.
- Deadlock. The v2 pod sits in Pending forever.

**Solutions:**

1. **Allow unavailability:** Set `maxSurge: 0, maxUnavailable: 1`. Kubernetes kills one v1 pod first, freeing a core, then schedules v2 there. Trade-off: briefly running at half capacity.

2. **Cluster autoscaler:** A new node gets provisioned for the surge pod. Full availability but slower (1-3 min) and costs more.

3. **Don't pack at 100%:** The real answer. Keep nodes at 70-80% request utilization. The extra 20-30% is your rolling update budget. This is a staff-level insight for capacity planning discussions.

---

## 6. Glossary: Cluster vs Node vs Pod vs Deployment

From biggest to smallest:

| Concept | What it is | Analogy |
|---|---|---|
| **Cluster** | The whole Kubernetes environment — control plane + workers | A factory |
| **Node** | A single machine (VM or physical) inside the cluster | A workbench in the factory |
| **Deployment** | A declaration: "I want N copies of app version X" | A work order pinned to the wall |
| **ReplicaSet** | Manages a specific version's pod count (created by Deployment) | A batch ticket for version 2.1 |
| **Pod** | Smallest deployable unit, wraps 1+ containers with shared network | A tray on the workbench |
| **Container** | The actual running process (nginx, your API, etc.) | The product being assembled |

How they nest:

```
Cluster
├── Node (worker-1, 4 cores, 16GB)
│   ├── Pod (my-api-xk9p2)
│   │   └── Container (my-company/api-server:v2.1)
│   ├── Pod (my-api-ab3m7)
│   │   └── Container (my-company/api-server:v2.1)
│   └── Pod (redis-cache-m4k2)
│       └── Container (redis:7.0)
│
├── Node (worker-2, 4 cores, 16GB)
│   ├── Pod (my-api-qr8n1)
│   │   └── Container (my-company/api-server:v2.1)
│   └── Pod (postgres-0)
│       └── Container (postgres:15)
│
└── Node (control-plane-1)
    ├── API server
    ├── etcd
    ├── Scheduler
    └── Controller manager
```

---

## 7. The One Mental Model That Ties Everything Together

Everything in Kubernetes follows the same pattern:

1. **You declare desired state** (write YAML, kubectl apply)
2. **API server stores it in etcd** (the single source of truth)
3. **Independent controllers watch etcd** and react when desired state ≠ actual state
4. **Each controller does one small job** and writes the result back to etcd
5. **The next controller in the chain picks it up** and does its small job

Nobody orchestrates the chain. Nobody calls controller B after controller A finishes. Each controller independently watches for conditions it cares about and acts. The smart behavior emerges from simple components reacting to shared state.

This is the reconciliation loop pattern, and it applies to everything:

- Deployment controller: "Does a ReplicaSet exist for this Deployment? No → create one"
- ReplicaSet controller: "Are there 3 pods? Only 2 → create one more"
- Scheduler: "Is there a pod with no node? → assign it to the best node"
- Kubelet: "Is there a pod assigned to me that isn't running? → start it"
- HPA: "Is CPU above target? → increase replica count on the ReplicaSet"
- Cluster autoscaler: "Is there a Pending pod with no room? → add a node"

Once you internalize this pattern, you can reason about any Kubernetes behavior by asking three questions:

1. **Which controller is watching this?**
2. **What's the desired state vs actual state?**
3. **What action does it take to close the gap?**

That's the framework that lets you think through Kubernetes problems in real time during a design discussion, rather than relying on memorized commands.

---

*Last updated: April 2026*