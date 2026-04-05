# How Software Gets Deployed: A History Built from Problems

## Stage 0: One Program, One Machine

You write a program. You compile it (or not). You run it on the machine where you wrote it. You are the only user. This is how most people start.

Deployment means typing `./my_program`. If it crashes, you restart it. If the machine dies, you restore from a backup (if you have one). The program reads and writes files on local disk. Everything is colocated — your code, your data, your user.

This works perfectly until it doesn't.

---

## Stage 1: Other People Need to Use It

The moment your program serves users over a network — a web application, an API, a database — two things change. First, the machine needs to be reachable. Second, it needs to stay running when you're not watching it.

**The reachability problem** means the machine needs a stable network address and a process that listens on a port. Your program goes from something you invoke to something that runs continuously as a **daemon** — a long-lived process, typically detached from any terminal session.

**The uptime problem** means you need something to restart your program if it crashes. Unix systems solved this with **init systems**: the kernel starts process ID 1 (init), and init is responsible for starting and supervising other processes. Early versions (SysV init) used shell scripts in `/etc/init.d/` that defined `start`, `stop`, and `restart` commands. You'd write a script that launched your daemon and hope it handled failures gracefully.

The problems with init scripts were real: they were brittle shell scripts that didn't track whether the process was actually running, didn't handle dependencies between services well, and had no built-in restart-on-crash behavior. Modern init systems like **systemd** replaced this: systemd tracks processes using Linux cgroups (the kernel's mechanism for grouping processes), can automatically restart crashed services, manages dependencies, and captures stdout/stderr into a structured log.

**Deployment at this stage** means: copy your program (and any configuration) to the server, write a systemd unit file (or equivalent) that tells the init system how to run it, and start the service. You probably do this by hand, over SSH.

**What's actually happening at the systems level:** your program is a process, managed by PID 1. It binds to a TCP port. The kernel routes incoming packets to that port. If the process dies, systemd notices (because the PID disappears) and spawns a new one.

**New problems:** You have one machine. If it dies, everything is down. And deployments are manual — you SSH in, copy files, restart the service. If you mess up, you're debugging in production with users watching.

---

## Stage 2: Making Deployment Repeatable

Deploying by hand works once. The fifth time you forget a step or misconfigure something. The problem is that a server's state — installed packages, config files, environment variables, user permissions — accumulates over time through ad-hoc commands. This accumulated state is impossible to reproduce reliably. People called these **snowflake servers**: each one is unique, and nobody fully understands how it got that way.

The first mitigation was **shell scripts** — just automate the SSH-and-copy workflow. But shell scripts for server configuration are fragile. They're not idempotent (running them twice might break things), they handle errors poorly, and they don't track what state the machine is currently in.

This led to **configuration management tools** (Chef, Puppet, Ansible). These tools let you declare the desired state of a machine — "this package should be installed, this file should have these contents, this service should be running" — and the tool figures out what changes are needed to reach that state. Ansible, for example, connects to machines over SSH and runs Python modules that check the current state and apply changes only if needed.

The key insight was **idempotency**: you describe *what* the state should be, not *how* to get there. Run the tool ten times, get the same result.

**Deployment at this stage:** You define your server's configuration in code (an Ansible playbook, a Chef recipe). To deploy, you run the tool, which connects to the server and converges its state to match your definition. Your application binary or code is part of that definition.

**New problems:** Even with automated configuration, your application has dependencies — specific language runtimes, libraries, system packages. These dependencies can conflict with other things on the same machine. And you still have one machine, so you have one point of failure.

---

## Stage 3: Running Multiple Things on One Machine

Servers are expensive. If you have several applications, you want to run them on the same hardware. But processes on the same machine share everything: the filesystem, the network namespace, system libraries, CPU, and memory. Application A needs Python 3.8 and Application B needs Python 3.11. Both want to listen on port 8080. Their dependencies stomp on each other.

The initial answer was careful partitioning — run each app under its own user account, use virtual environments for language dependencies, bind to different ports. This works but it's manual and fragile.

**Virtual machines** offered a more complete solution. A **hypervisor** (like KVM, built into the Linux kernel, or VMware's ESXi) runs between the hardware and the operating systems. It presents virtualized hardware to each guest OS. Each VM gets its own kernel, its own filesystem, its own network stack. From the perspective of software inside the VM, it's running on a dedicated machine.

**What's actually happening:** The hypervisor intercepts privileged CPU instructions from each guest kernel and either emulates them or (with hardware virtualization support like Intel VT-x) runs them in a special CPU mode that traps to the hypervisor on sensitive operations. Each VM's memory is mapped through a second layer of page tables. Virtual network interfaces are bridged or NATed by the hypervisor.

**The value:** Complete isolation. Each VM is a self-contained machine. You can run different operating systems, different library versions, different everything. If one VM misbehaves, others are unaffected.

**New problems:** VMs are heavy. Each one runs an entire operating system kernel, consuming hundreds of megabytes of RAM before your application even starts. Booting takes seconds to minutes. You can maybe run tens of VMs on a beefy server, not hundreds. And now you have a new deployment question — deploying *into* VMs. You need a way to create VM images (the virtual disk that contains the OS and your app), version them, and distribute them.

---

## Stage 4: Lighter Isolation — Containers

The weight of VMs came from the fact that each one runs its own kernel. But most of the time, your applications don't need different kernels — they need different *userspaces*. They need isolated filesystem views, separate network stacks, and resource limits. The Linux kernel already had mechanisms for all of this.

**Namespaces** (added to Linux incrementally from 2002–2016) let you give a process its own view of system resources. There are namespaces for:
- **Mount**: the process sees its own filesystem tree
- **PID**: the process sees its own process ID space (its PID 1 is not the host's PID 1)
- **Network**: the process gets its own network interfaces, IP addresses, and routing table
- **User**: the process can think it's root inside its namespace but map to an unprivileged user on the host
- And several others (UTS for hostname, IPC for inter-process communication, cgroup)

**Cgroups** (control groups) let you limit and account for resource usage — CPU time, memory, disk I/O, network bandwidth — for groups of processes.

**A container is a process (or group of processes) running with its own namespaces and cgroups.** There's no separate kernel, no hardware virtualization. The container's processes run as regular Linux processes, but they *see* an isolated environment.

The missing piece was the filesystem. You want each container to start from a clean, known filesystem state. **Union filesystems** (like OverlayFS) solved this by layering filesystem changes on top of a read-only base. You start with a base image (say, a minimal Debian filesystem), layer your application on top, and each container gets its own writable layer. The base layers are shared across containers, saving disk space.

**Docker** (2013) packaged all of this into a usable workflow. What Docker actually did was:
1. Defined a **Dockerfile** format — a recipe for building a filesystem image layer by layer
2. Provided a **registry** — a server that stores and distributes images
3. Gave you a CLI to build images, push them, pull them, and run containers from them

A Docker image is just a tarball of filesystem layers with metadata. A running container is just a Linux process with namespaces, cgroups, and a union filesystem mount.

**Deployment changed fundamentally.** Instead of configuring a server to run your application, you build an image that *contains* your application and all its dependencies. The server only needs a container runtime. Deploying means "pull the new image and start a container from it." The environment is identical everywhere — your laptop, the CI server, production — because the image carries the environment with it.

```
Developer's machine:  Dockerfile -> build -> image
Registry:             push/pull images
Production server:    pull image -> run container
```

**What you gained:** Sub-second startup (no kernel to boot). Hundreds of containers per machine. Reproducible environments. Clear dependency boundaries.

**New problems:** You now have potentially many containers on one or more machines. Who decides which containers run where? If a container dies, who restarts it? If you need three copies for redundancy, who ensures exactly three are running? If containers need to talk to each other, how do they find each other? You've solved the isolation and packaging problems but created an orchestration problem.

### The road not taken: Nix

Containers solved the reproducibility problem by brute force — ship the entire filesystem. But there's an alternative approach that solves it more rigorously by attacking the root cause: if builds are non-deterministic, make them deterministic.

**Nix** is a package manager built on a single idea: a package is a pure function of its inputs. The source code, every dependency, the compiler, the compiler flags — all of it gets hashed together, and the build output is stored at a path derived from that hash: `/nix/store/abc123-python-3.11.4/`. If any input changes, the hash changes and you get a different path. Two packages that need different versions of the same library coexist at different store paths. Nothing is global, nothing is mutable, nothing conflicts. You get the dependency isolation of Stage 3 (multiple things on one machine without stomping on each other) without VMs or containers — just careful path management.

This scales upward. **NixOS** applies the same principle to an entire operating system: packages, services, users, kernel modules, all declared in a single expression and built deterministically from that declaration. It's Stage 2 (configuration management) done with a rigor that Ansible and Chef couldn't achieve — instead of converging a mutable system toward a desired state, you build the entire system from scratch from its specification. Rollback means booting the previous generation, which is still sitting in the Nix store.

The mainstream didn't go this way. Containers won because they were conceptually simpler ("ship a tarball of the filesystem"), worked with existing tools and mental models, and had Docker's excellent developer experience. Nix demands a fundamentally different mental model — functional programming applied to system configuration — and has a notoriously steep learning curve. The trade-off is real: Nix solves reproducibility more rigorously (bit-for-bit identical builds, no "but it works in the container I built yesterday"), but containers were good enough for most organizations and far easier to adopt.

Where Nix shows up in practice today is often *alongside* containers rather than instead of them. You use Nix to build the container image — guaranteeing the image contents are reproducible and minimal — then deploy the image through the normal container/orchestration pipeline from Stage 5 onward. It slots in at the "build" step rather than replacing the runtime story.

---

## Stage 5: Multiple Machines

Before addressing orchestration, let's acknowledge why you end up with multiple machines in the first place.

A single machine has a ceiling: finite CPU, memory, disk I/O, and network bandwidth. When your application's load exceeds what one machine can handle, you have two options: **scale vertically** (bigger machine) or **scale horizontally** (more machines).

Vertical scaling hits hard limits quickly and gets exponentially expensive. So you go horizontal: run multiple copies of your application behind a **load balancer**.

A load balancer is a process (or dedicated hardware) that accepts incoming connections and forwards them to one of several backend servers. At the simplest level (L4 / TCP), it sees a new TCP connection and picks a backend based on an algorithm — round-robin, least connections, random, hashing the source IP. At the application level (L7 / HTTP), it can inspect the HTTP request and route based on path, headers, cookies, etc.

**What's happening at the network level:** The load balancer has a public IP address. DNS points your domain to that IP. The load balancer accepts the TCP connection, selects a backend, and either proxies the connection (maintaining two separate TCP connections) or uses NAT to rewrite the destination. The backends typically have private IPs on an internal network.

**Health checks** emerge naturally here. If one backend crashes, the load balancer shouldn't send traffic to it. So the load balancer periodically makes requests to each backend (typically `GET /health`) and removes unresponsive backends from the rotation. This gives you automatic failover — a dead backend gets no traffic until it recovers.

**Deployment now has a new dimension.** You can do **rolling deploys**: update one backend at a time, let the load balancer drain connections from the old instance, bring up the new one, verify it's healthy, move to the next. If the new version is broken, you've only affected a fraction of traffic and can roll back.

**New problems:** You're managing multiple machines manually. Each one needs the container runtime installed, needs to be monitored, needs SSH access, needs OS updates. When a machine dies, you need to provision a replacement. You're making placement decisions by hand — "this container goes on server 3 because server 2 is almost full." And your configuration management needs to handle heterogeneous machines.

---

## Stage 6: Treating Machines as a Pool — Orchestration

The core tension: you think in terms of applications ("I need three copies of my web server and two copies of my worker"), but you're managing machines ("server 1 has this running, server 2 has that"). You want to declare intent and let something else handle placement.

This is what **container orchestrators** do. The most dominant one is Kubernetes (2014, from Google), but the concept is clearer when you build it up from the problems.

What an orchestrator needs to do:
1. **Track the state of all machines** (how much CPU/memory is free, what's running where)
2. **Accept a desired state** ("run 3 copies of image X, each needing 512MB RAM and 0.5 CPU")
3. **Schedule containers onto machines** (bin-packing: find a machine with enough free resources)
4. **Monitor running containers** and reconcile with desired state (if one dies, start a replacement)
5. **Provide networking** so containers can find and talk to each other regardless of which machine they're on

The architecture typically has a **control plane** (the brain — stores desired state, makes scheduling decisions) and **worker nodes** (the machines that actually run containers).

In Kubernetes specifically:
- The control plane stores all state in **etcd** (a distributed key-value store). The **API server** is the front door — all commands go through it. The **scheduler** watches for unassigned containers and picks nodes for them. **Controllers** are loops that watch the current state, compare it to the desired state, and take action to converge.
- Each worker node runs a **kubelet** (an agent that talks to the API server, starts and stops containers as instructed) and a **kube-proxy** (handles network routing for services).

The fundamental abstraction is **declarative state reconciliation.** You don't say "start a container on node 5." You say "I want 3 replicas of this container." The system continuously works toward making reality match your declaration. If a node dies and takes a replica with it, the system notices the discrepancy and schedules a replacement somewhere else. You're managing a cluster, not machines.

**Deployment becomes:** push a new image to the registry, update the desired state ("now use image v2 instead of v1"), and the orchestrator handles the rolling update — spinning up new containers, health-checking them, draining old ones.

### The elephant in the room: state

Everything above assumes **stateless** workloads — processes that can be killed and restarted anywhere without losing anything, because all their persistent data lives somewhere else. Web servers, API backends, and workers fit this model. Databases, message queues, and object stores do not.

A stateless container is fungible. The orchestrator can reschedule it to any node, start five copies, or kill one mid-request and nobody notices (once the replacement picks up). A database container is the opposite: it has data on a local disk. If you kill it and restart it on another node, the data isn't there. If you run two copies, they need to agree on who's the primary. The orchestrator's core assumption — that containers are interchangeable cattle — breaks down.

This tension shaped real infrastructure decisions profoundly. The common resolution: **don't orchestrate your databases the same way you orchestrate your applications.** In practice, most organizations run stateful workloads in one of three ways:

1. **Managed services** — offload the problem entirely. Use the cloud provider's database (RDS, Cloud SQL), message queue (SQS, Pub/Sub), or object store (S3, GCS). The provider handles replication, failover, backups, and disk management. Your application just talks to an endpoint. This is by far the most common choice.
2. **Dedicated VMs or bare metal** — run the database on machines outside the orchestrator, managed with traditional configuration management (Ansible, etc.) or the provider's VM tooling. Proven, well-understood, but you're back to managing snowflakes for that part of the stack.
3. **StatefulSets and persistent volumes** — Kubernetes *can* run stateful workloads. StatefulSets give each pod a stable identity (pod-0, pod-1, pod-2) and stable network names, so database replicas can find each other across restarts. Persistent volume claims attach network-attached storage (like an EBS volume) that follows the pod across rescheduling. Operators (custom controllers) encode database-specific knowledge — how to initialize a replica, promote a follower, take backups. This works but adds significant complexity and demands deep expertise in both the database and the orchestrator.

The key insight: **the container/orchestration story is fundamentally a compute story.** It solved the "where do processes run" problem. The "where does data live" problem is largely orthogonal and follows a different set of trade-offs around durability, consistency, and replication. The two worlds interact — your containers need to *connect to* your data stores — but they're not managed the same way, and conflating them causes pain.

This asymmetry persists through every subsequent stage. When we talk about deployment pipelines, observability, and service meshes below, the mental model is primarily stateless services. Stateful systems have their own operational playbooks, their own failure modes, and their own upgrade strategies.

---

## Stage 7: Service Discovery and Internal Networking

With many containers scattered across many machines, a basic problem emerges: how does Service A find Service B?

You can't hardcode IP addresses — containers are ephemeral, they get rescheduled to different machines, their IPs change. You need a layer of indirection.

**Service discovery** is the mechanism by which a running service learns the network address of another service it needs to talk to. There are several approaches:

**DNS-based:** Run an internal DNS server that maps service names to current IPs. When service "user-api" scales to 3 instances on 3 different IPs, DNS returns all three. The client picks one (or DNS rotates the order). Kubernetes has a built-in DNS server that does exactly this — every service gets a DNS name like `user-api.default.svc.cluster.local`.

**Virtual IPs:** Kubernetes also creates a **cluster IP** for each service — a stable virtual IP that isn't assigned to any physical interface. When a packet is sent to this IP, **kube-proxy** on the local node intercepts it (using iptables rules or IPVS) and rewrites the destination to one of the actual container IPs. The client talks to one stable address; the routing happens transparently.

**The underlying networking challenge** is: how do containers on different machines talk to each other at all? Each container has its own network namespace with its own IP address. But these IPs are internal to the node. The solution is an **overlay network** — a virtual network that spans all nodes. Implementations vary, but the principle is: when a container on Node A sends a packet to a container on Node B, the packet gets encapsulated (wrapped inside another packet addressed from Node A's real IP to Node B's real IP), sent across the physical network, and decapsulated on the other side. From the containers' perspective, they're on the same flat network. Technologies like VXLAN do this encapsulation at the kernel level.

**New thread — securing communication between services:** On the open internet, TLS encrypts traffic. Inside your cluster, traffic between services historically went unencrypted — it was all on a private network, how bad could it be? But as organizations adopted zero-trust security postures, encrypting *all* traffic became necessary. We'll see how this gets addressed in the service mesh discussion below.

---

## Stage 8: Handling Configuration and Secrets

Your application needs configuration — database connection strings, API keys, feature flags. Baking these into the container image is a bad idea: you want the same image to run in staging and production with different config.

The traditional Unix approach is environment variables and config files. Orchestrators formalize this:

- **Config maps** (in Kubernetes): key-value pairs that get injected into the container as environment variables or mounted as files
- **Secrets**: same concept, but the values are stored with some level of access control and (depending on setup) encryption at rest

For more sensitive secrets, external **secret management** systems (like HashiCorp Vault) add audit logging, automatic rotation, dynamic credentials (e.g., generating a short-lived database password on demand), and fine-grained access policies.

---

## Stage 9: The Deployment Pipeline

At this point, the *mechanism* for deploying is solid — build an image, push it, update the desired state in the orchestrator. But the *process* around it needs structure. What triggers a deploy? How do you prevent bad code from reaching production?

**Continuous Integration (CI)** is the practice of automatically building and testing every code change. Concretely: a developer pushes to a Git branch, a server (Jenkins, GitHub Actions, GitLab CI — just programs that watch for Git events and run shell commands in defined environments) pulls the code, builds it, runs the test suite, and reports pass/fail. The CI server is just an automated build machine with some triggering logic.

**Continuous Deployment (CD)** extends this: if the build passes, automatically deploy it. The pipeline typically looks like:

```
push code -> CI builds + tests -> build container image -> push image to registry
-> update orchestrator's desired state -> rolling update in production
```

**What's actually happening:** the CI/CD system is running shell commands in sequence, with each step gated on the previous one succeeding. It's fundamentally just scripted automation — the same thing you'd do by hand, but triggered automatically and running in a controlled environment.

**Deployment strategies** get more sophisticated:
- **Rolling update**: Replace instances one at a time (the default in Kubernetes)
- **Blue-green**: Run the new version alongside the old as a complete parallel environment, then switch traffic all at once by updating the load balancer
- **Canary**: Route a small percentage of traffic (say 5%) to the new version, monitor error rates and latency, gradually increase if metrics look good

---

*At this point, the core systems problems are largely solved. You can package software reproducibly (containers), run it on pooled hardware (orchestration), route traffic to it (service discovery, load balancing), and recover from failures automatically (reconciliation loops, health checks). The remaining stages are about operating and governing what you've built — making the running system visible, consistent, and auditable.*

---

## Stage 10: Observability — Seeing What's Happening

With dozens of services running across many machines, "SSH in and check the logs" doesn't work anymore. You need centralized visibility.

**Logging:** Each container writes to stdout/stderr. The container runtime captures this. An agent on each node (like Fluentd or Filebeat) collects these logs and ships them to a central store (Elasticsearch, Loki) where you can search across all services. The key challenge is **correlation** — when a user request touches 5 services, you need to trace it across all their logs. This is solved by passing a unique **request ID** (or trace ID) through every service call, including it in every log line.

**Metrics:** Numeric time-series data — request count, latency percentiles, error rates, CPU usage, memory usage. Services expose metrics (typically via an HTTP endpoint), a collector (like Prometheus) scrapes them periodically, and you visualize them in dashboards. **Alerting** rules fire when metrics cross thresholds — error rate above 1%, latency above 500ms. Alerts go to a pager or a chat channel.

**Distributed tracing:** When a request flows through multiple services, tracing captures the timing of each hop. Each service creates a **span** (a named, timed operation) and links it to the parent span via headers propagated through the request. The result is a tree of spans showing exactly where time was spent. Tools like Jaeger or Zipkin collect and visualize these traces.

These three — logs, metrics, traces — are collectively called the **three pillars of observability**. They serve different purposes: metrics tell you *something* is wrong, traces tell you *where*, logs tell you *why*.

---

## Stage 11: Service Meshes — Infrastructure Concerns as a Layer

At this point, every service needs to handle some common concerns:
- Retries and timeouts when calling other services
- Circuit-breaking (stop calling a failing service to let it recover)
- Mutual TLS (both sides authenticate with certificates, encrypting traffic)
- Request-level metrics and tracing
- Traffic splitting for canary deployments

Implementing all of this in each service's code is wasteful and error-prone. Different services are written in different languages, by different teams. They all need the same behavior.

A **service mesh** extracts these concerns from application code into the infrastructure. The concrete mechanism: alongside every container, the orchestrator injects a **sidecar proxy** — another container running in the same network namespace. All inbound and outbound traffic for your application gets routed through this proxy (via iptables rules that redirect traffic).

The proxy (Envoy is the most common) handles mTLS termination, retries, circuit-breaking, load balancing, metric collection, and trace propagation. Your application code makes a plain HTTP call to `http://user-api:8080/users/123`. The proxy intercepts it, establishes a mutual TLS connection with the proxy on the other side, applies retry policies, records latency metrics, and forwards the request.

A **control plane** (like Istio's or Linkerd's) manages all the proxies — distributing TLS certificates, pushing configuration, collecting telemetry.

**What you gain:** Consistent cross-service behavior without library dependencies. Language-agnostic. Operators can change retry policies or traffic routing without modifying application code.

**What it costs:** Significant operational complexity. Every request now goes through two extra hops (proxy on each side). Debugging is harder because the proxy is an invisible intermediary. The control plane itself needs to be highly available.

---

## Stage 12: Infrastructure as Code

At every layer so far — VMs, networks, load balancers, Kubernetes clusters, DNS records — someone has been creating and configuring infrastructure. If this is done through web consoles or ad-hoc CLI commands, you get the same snowflake problem you had with servers.

**Infrastructure as Code (IaC)** tools let you define infrastructure in declarative configuration files and apply them. Terraform (or its successor OpenTofu) is the canonical example. You write:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.medium"
}
```

Terraform compares this declaration to the actual state of your cloud account (stored in a **state file**), computes the diff, and makes the API calls to create, modify, or destroy resources. The state file tracks what Terraform manages — without it, Terraform doesn't know what's already been created.

This means your entire infrastructure — networks, machines, databases, DNS, Kubernetes clusters, load balancers — lives in version-controlled files. You can review changes in pull requests, roll back by reverting a commit, and reproduce entire environments from scratch.

---

## Stage 13: GitOps — Git as the Source of Truth

A natural convergence: if your infrastructure configuration is in Git and your application desired state (Kubernetes manifests) is in Git, why not make Git the single source of truth for *everything* running in production?

**GitOps** is the practice of declaring your entire system's desired state in a Git repository and having an automated agent that continuously reconciles the actual state with what's in Git. Tools like ArgoCD or Flux watch a Git repo. When you push a change (update an image tag, change a config value, add a new service), the agent detects the diff and applies it to the cluster.

Deployment becomes: merge a pull request. Rollback becomes: revert a commit. Audit trail becomes: Git history.

---

## Where We Are Now

The progression, viewed from 30,000 feet:

| Era | Deploy means... | Isolation unit | Failure handling |
|---|---|---|---|
| One machine | Copy binary, restart | Process | Manual restart |
| Config management | Run playbook over SSH | Process + packages | Init system restarts process |
| VMs | Create VM image, boot it | Full OS | Hypervisor restarts VM |
| Containers | Build image, run container | Namespaced process | Orchestrator reschedules |
| Orchestrator | Declare desired state | Pod (container group) | Continuous reconciliation |
| GitOps | Merge a PR | Pod + infrastructure | Automated self-healing |

Each layer addressed real pain from the previous one, and each introduced new complexity that motivated the next. The overall trajectory is toward **declarative, version-controlled, self-healing systems** — you state what should exist, and the infrastructure converges toward it. The cost is layers of abstraction and operational complexity that require real expertise to understand and debug when things go wrong.

The fundamental operations haven't changed since Stage 0: a process runs on a CPU, reads and writes memory, sends and receives network packets. Everything above is coordination machinery to make these basic operations reliable, reproducible, and scalable.
