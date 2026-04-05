# Infrastructure architectures for AI agent compute in production

**Firecracker microVMs have emerged as the dominant isolation primitive for production agent workloads, with every major agent platform — Cursor, E2B, Fly.io Sprites, AWS Bedrock AgentCore — converging on hardware-virtualized sandboxes that boot in under 200ms.** The newly released `kubernetes-sigs/agent-sandbox` CRD (March 2026) signals Kubernetes ecosystem recognition that agents represent a fundamentally new workload type requiring purpose-built abstractions. For self-hosted deployments, a spectrum of viable architectures exists ranging from gVisor-sandboxed containers with warm pools to full Firecracker microVM stacks — with the right choice depending on security requirements, operational maturity, and scale. At 500 concurrent sessions, self-hosted infrastructure costs **5–15× less** than managed agent sandbox platforms, but demands significant engineering investment in orchestration, snapshot lifecycle, and warm-pool management.

---

## 1. The agent compute landscape at a glance

The infrastructure landscape for agent workloads has crystallized into four distinct layers, each addressing different organizational needs and maturity levels.

**Production agent companies** (Cursor, Devin, GitHub Copilot, Replit) build custom orchestration on top of cloud primitives. Cursor confirmed using **AWS Firecracker microVMs** with a Rust-based orchestrator called "Anyrun." Devin runs isolated cloud VMs on Azure. GitHub Copilot's coding agent repurposes **GitHub Actions runners** — ephemeral containers with a network firewall layer and zero-secret architecture. Replit uses Docker containers on GCP VMs with a custom copy-on-write storage fabric called "Bottomless Storage." Bolt.new takes a radically different approach: WebContainers run Node.js entirely in-browser via WebAssembly, achieving **millisecond cold starts with zero server-side compute**.

**Agent sandbox platforms** (E2B, Fly.io Sprites, Modal, Daytona, Blaxel) provide purpose-built APIs for creating, managing, and destroying isolated execution environments. These differ from generic container orchestration in three critical ways: sub-second creation from snapshots, per-sandbox lifecycle management with pause/resume, and SDK-driven programmatic control. E2B leads on developer experience and open-source availability (Apache 2.0), Fly.io Sprites leads on the persistent-state philosophy, and Modal leads on Python ML ecosystem integration.

**Remote development platforms** (Coder, Gitpod/Ona) have pivoted aggressively toward agent workloads. Coder now explicitly markets to agent use cases with purpose-built features (Tasks, Mux, Agent Boundaries). Anthropic uses Coder for Claude Code environments. Gitpod rebranded entirely as **Ona** in September 2025, dropping the developer environment framing for an agent-first platform. These platforms provide mature workspace lifecycle management, warm pools via prebuilds, and enterprise governance that purpose-built sandbox platforms lack.

**Kubernetes-native tooling** arrived with `kubernetes-sigs/agent-sandbox`, released under SIG Apps in March 2026. This project provides Sandbox, SandboxTemplate, SandboxWarmPool, and SandboxClaim CRDs that directly model agent session lifecycle as Kubernetes-native resources, supporting gVisor and Kata Containers RuntimeClasses for isolation.

---

## 2. How production agent companies architect their compute

The most revealing architectural detail from production systems is the convergence on **full environment isolation** — every major agent platform gives each session its own kernel or VM boundary, not just process-level separation.

**Cursor's architecture is the most documented.** Background agents run in Firecracker microVMs on AWS, orchestrated by a Rust service called "Anyrun" that handles "launching agents in the cloud, securely and with the right process isolation." Each agent clones a GitHub repo, sets up the environment (from Dockerfile or agent-driven discovery), caches the disk state after installation, then operates with full shell access across tmux terminals. Cursor supports **10–20 parallel agents per user**, each on separate VMs. Disk state caching means subsequent sessions on the same repo skip installation — a snapshot-based bootstrapping pattern. Environment configuration lives in `.cursor/environment.json`.

**GitHub Copilot's coding agent** leverages the existing Actions infrastructure at massive scale — **40 million daily jobs** on weekdays. The security architecture uses a "substrate layer" resting on an Actions runner VM with several trusted containers providing kernel-enforced communication boundaries. The agent container has tightly controlled egress with configurable domain allowlists, an MCP gateway in a separate trusted container, and a zero-secret design where LLM authentication tokens are isolated in an API proxy the agent container cannot access. This is architecturally notable: GitHub chose to **repurpose existing CI/CD infrastructure** rather than build bespoke agent compute.

**Replit's custom storage system** is uniquely powerful for agent workloads. Their "Bottomless Storage" infrastructure uses virtual block devices via the Network Block Device protocol, backed by Google Cloud Storage with lazy loading. Filesystems are stored as **16 MiB immutable chunks**, and copying is manifest-only — constant-time regardless of filesystem size. This enables instant filesystem forks for parallel agent experimentation and rollback to any checkpoint. Replit's "Parallel Sampling" feature runs multiple agents in forked environments simultaneously, selecting the best result.

| Platform | Isolation boundary | Key technology | Cold start | State management |
|---|---|---|---|---|
| **Cursor** | Firecracker microVM | Rust "Anyrun" orchestrator, AWS | ~125ms boot + env setup | Disk state caching after install |
| **GitHub Copilot** | Actions runner container | Firewall proxy, zero-secret arch | Actions runner startup | Ephemeral (no idle state) |
| **Devin** | Cloud VM (Azure) | gRPC/WebSocket control plane | Improved in v2.2 | Session pause/resume via VM |
| **Replit** | Docker container on GCP VM | CoW block storage, Nix, Goval | Seconds (GCP VM allocation) | Forkable filesystems + databases |
| **Bolt.new** | Browser/WebAssembly | WebContainers, Service Workers | **Milliseconds** | IndexedDB (client-side) |
| **Amazon Q** | Managed sandbox + Devfile | Strands SDK, Bedrock AgentCore | Not disclosed | Ephemeral per task |

---

## 3. Agent sandbox platforms compared in depth

The agent sandbox category has exploded from essentially zero in 2023 to **7+ competing platforms** in early 2026. The fundamental design tension is between **ephemeral sandboxes** (E2B, Modal) and **persistent environments** (Fly.io Sprites, Blaxel).

### E2B: the open-source Firecracker standard-bearer

E2B provides Firecracker microVM sandboxes with **<200ms startup from snapshot**, a Python/TypeScript SDK, and full filesystem/terminal/browser access per sandbox. Each sandbox gets its own Linux kernel via KVM, providing hardware-level isolation. Custom environments are defined via Dockerfiles converted into snapshotted microVM templates. E2B's **pause/resume preserves complete memory state** including running processes, loaded variables, and filesystem — not just a filesystem snapshot. Paused sandboxes incur no compute charges and persist up to 30 days on the Pro plan.

E2B's infrastructure is **fully open-source under Apache 2.0** (`e2b-dev/infra`), self-hostable via Terraform on AWS and GCP. This makes it the only production-grade agent sandbox with a self-hosting path. The managed service prices at ~**$0.05/vCPU-hour**, with Pro at $150/month supporting 100 concurrent sandboxes (expandable to 1,100). Major users include Hugging Face (tens of thousands of concurrent sandboxes), Manus, and reportedly 88% of Fortune 100 companies.

The key limitations: **24-hour maximum session length** on Pro, no built-in outbound network policies (egress filtering must be implemented externally), and self-hosting requires managing the full E2B control plane.

### Fly.io Sprites: the persistent-state challenger

Launched January 2026, Sprites represent a **philosophical break** from ephemeral sandboxes. CEO Kurt Mackey argues: "Claude doesn't want a stateless container. Claude wants a computer." Each Sprite gets a persistent **100GB NVMe-backed filesystem** that survives between sessions — no Docker images, no container rebuilds. Sprites create in 1–12 seconds, auto-sleep after 30 seconds of inactivity, and resume in under 1 second.

The storage architecture uses fast local NVMe for low-latency I/O plus durable object storage for persistence. Checkpoint/restore captures entire environment state in **~300ms**, storing the last 5 checkpoints. Billing is granular: **$0.07/CPU-hour**, $0.04375/GB-hour memory, with no charges while asleep. A 4-hour coding session costs approximately $0.44. Sprites bill only for storage blocks actually written, not the full 100GB allocation.

Sprites are **not open-source** (though Fly.io's Thomas Ptacek indicated an open-source local version is planned). The product is designed primarily for individual developer workflows rather than platform-scale multi-tenancy. REST API and SDKs (JavaScript, Go) enable programmatic control, with L3 network policies providing configurable outbound domain filtering.

### Modal: gVisor-based serverless with ML ecosystem depth

Modal uses **gVisor** (Google's user-space kernel intercepting ~68 syscalls) rather than Firecracker microVMs. This provides stronger-than-container isolation but uses a shared host kernel — a weaker security boundary than hardware virtualization. Modal's strength is its Python-first serverless model with autoscaling from zero to **20,000+ concurrent containers** and built-in GPU support (T4 through B200).

Sandbox pricing runs ~**$0.142/core-hour** (3× the base function rate), making it the most expensive option for sustained agent workloads. Memory snapshots capture container state at user-controlled points, but restoring creates a *new* sandbox rather than resuming the original. Modal is **proprietary with no self-hosting or BYOC option**, and SDK-defined images (no arbitrary OCI images) create vendor lock-in.

### Emerging entrants

**Daytona** pivoted from developer environments to agent infrastructure in early 2025. Using Docker containers by default, it achieves **sub-90ms cold starts** (27ms best case) — faster than any microVM — at the cost of weaker default isolation. Enhanced isolation via Kata Containers is available but requires explicit configuration. Open-source under AGPL 3.0.

**Blaxel** (YC Spring 2025) claims industry-leading **<25ms resume from standby** with scale-to-zero after 1 second of inactivity. It co-locates agent logic alongside sandboxes to eliminate network overhead. Managed-only, no self-hosting.

**Vercel Sandbox** uses Firecracker microVMs with a novel "active CPU pricing" model at $0.128/vCPU-hour, charging only during CPU execution (not I/O wait) — claiming up to 95% savings for idle-heavy workloads. Maximum 5-hour sessions on Pro/Enterprise.

| Platform | Isolation | Cold start | Snapshot/restore | Self-host | Pricing model |
|---|---|---|---|---|---|
| **E2B** | Firecracker microVM | ~150ms | Full memory state | ✅ Apache 2.0 | ~$0.05/vCPU-hr |
| **Fly Sprites** | Firecracker microVM | 1–12s (create), <1s (resume) | ~300ms checkpoint | ❌ (planned) | $0.07/CPU-hr |
| **Modal** | gVisor | 2–4s | New from snapshot | ❌ | ~$0.14/core-hr |
| **Daytona** | Docker (default) | Sub-90ms | OCI snapshot | ✅ AGPL 3.0 | ~$0.067/hr |
| **Blaxel** | MicroVM | <25ms resume | Full state | ❌ | ~$0.08/hr |
| **Vercel** | Firecracker microVM | Not disclosed | Filesystem only | ❌ | $0.128/vCPU-hr (active only) |

---

## 4. Remote dev environments pivoting to agent infrastructure

The most architecturally mature option for self-hosted agent compute may not be an agent-specific platform at all. **Coder** (AGPL v3, `coder/coder`, 17K+ GitHub stars) has emerged as the leading open-source platform for agent workloads, with Anthropic confirmed as a production user for Claude Code environments.

Coder's architecture centers on **Terraform-based workspace provisioning**: templates written in Terraform define infrastructure resources (K8s pods, Docker containers, cloud VMs), and provisioner daemons execute `terraform apply` during workspace creation. This provides unmatched flexibility — workspaces can target any Terraform provider. The control plane (`coderd`) exposes a full REST API for programmatic lifecycle management, and **prebuilt workspaces** maintain a warm pool of ready-to-claim environments. When an agent requests a workspace, if a matching prebuild exists, ownership transfers instantly — eliminating cold-start latency.

Purpose-built agent features now include **Coder Tasks** (running terminal-based agents like Claude Code in background workspaces), **Mux** (parallel agent orchestration in governed workspaces), **Agent Boundaries** (process-level firewall), and **AI Bridge** (infrastructure-level API key and MCP tool injection). The validated architecture supports **up to 3,000+ concurrent users** on Kubernetes. The community edition includes prebuilds, autostop, and unlimited workspaces; premium adds dormancy management, resource quotas, multi-org, and HA.

**Gitpod rebranded as Ona** in September 2025, abandoning the developer environment framing entirely. Ona claims its agents "co-authored 60% of PRs merged on main." The new platform uses **VM-level isolation** (each environment gets a dedicated VM with containers inside), self-hosted runners in customer AWS VPCs, and environment classes from 2 vCPU/8GB to 32 vCPU/128GB. The critical difference from Coder: Ona bundles its own proprietary agents, while Coder provides infrastructure for *any* agent. Ona's new platform is **closed-source**, a significant departure from Gitpod's AGPL-licensed classic codebase.

**Eclipse Che** (EPL 2.0) is fully Kubernetes-native — workspaces are DevWorkspace CRDs managed by the DevWorkspace Operator. This makes it controllable via `kubectl` or any Kubernetes client. However, Che lacks prebuilds, warm pools, and agent-specific lifecycle management. Repurposing it for agent workloads would require building a prebuild system, agent-aware lifecycle controller, and headless workspace templates — essentially recreating much of what Coder already provides.

---

## 5. Firecracker microVMs and the isolation spectrum

Firecracker, written in Rust by AWS for Lambda and Fargate, handles **tens of trillions of function invocations** and has become the reference implementation for agent sandboxing. Its security posture derives from an intentionally minimal design: ~50K lines of Rust, only 5 emulated devices (virtio-net, virtio-block, virtio-vsock, serial console, minimal keyboard controller), and a jailer that applies **six layers of defense**: KVM hardware virtualization, chroot, Linux namespaces, cgroups, seccomp-BPF (whitelisting only 24 syscalls), and privilege dropping.

**Cold boot completes in ≤125ms** from the InstanceStart API call to guest `/sbin/init`. Memory overhead is **<5 MiB per microVM** (VMM threads only). A single host can create **up to 150 microVMs per second**. CPU performance exceeds **95% of bare metal**. AWS runs production oversubscription ratios as high as 10×, tested at over 20×, enabled by demand fault paging — memory files are MAP_PRIVATE mapped and pages load on demand.

**Snapshot/restore** is the critical capability for idle-heavy agent workloads. A full snapshot captures CPU registers, memory contents, and device state. Restore times vary by implementation: **28ms** achieved by the ForgeVM project, **~400ms** by CodeSandbox (with compressed memory), **"hundreds of milliseconds"** reported by Fly.io. Snapshot size equals allocated RAM by default, but memory balloon inflation before snapshot can dramatically reduce this — CodeSandbox reclaims freed pages to minimize snapshot files.

**Cloud Hypervisor** is the leading alternative: also Rust-based (~50K LOC), sharing `rust-vmm` crates with Firecracker, but with more features — CPU/memory hotplugging, VFIO GPU passthrough, live migration, and Windows guest support. Boot time is ~200ms (vs Firecracker's ~125ms). Northflank chose Cloud Hypervisor over Firecracker for broader compatibility in their production deployment handling **2M+ isolated workloads monthly**.

**Kata Containers** provides the most mature path for integrating microVMs into Kubernetes via RuntimeClass. It supports Cloud Hypervisor (recommended), Firecracker, and QEMU as VMM backends. Boot time is 150–300ms with ~30–50MB memory overhead per guest OS. Production users include Baidu, Ant Group, Microsoft Azure (Confidential Containers on AKS), and Northflank. A maintainer acknowledged in 2025 that "lack of production maturity and a steep learning curve are the two main challenges" for widespread adoption.

For self-hosted Kubernetes, **Kata Containers with Cloud Hypervisor as the VMM backend** is the most viable microVM path. Direct Firecracker-on-Kubernetes integration via `firecracker-containerd` exists but is not fully production-ready. Running Firecracker outside Kubernetes (E2B's approach) with a custom orchestrator provides the most control over snapshot/resume lifecycle.

---

## 6. Lifecycle management that actually works in production

The **80% idle profile** of agent sessions makes lifecycle management — specifically, the ability to suspend sessions without losing state and resume them quickly — the single most impactful optimization axis.

**Firecracker snapshot/restore is production-ready** for full snapshots (GA status). The workflow: pause microVM → serialize memory/CPU/device state → store snapshot files → later, MAP_PRIVATE mmap of memory file with on-demand page loading. Network connections **do not survive** snapshot/restore — applications must handle reconnection. Diff snapshots (tracking only dirty pages) remain in "developer preview." E2B, Fly.io, and CodeSandbox all use this mechanism in production.

**CRIU (Checkpoint/Restore In Userspace)** reached **beta in Kubernetes v1.30** via the Container Checkpointing feature gate. It works by freezing a process tree via ptrace, collecting process state from `/proc`, and saving memory pages, file descriptors, TCP connections (via "TCP repair mode"), and process hierarchy as protobuf files. Restore converts checkpoint archives to OCI images that CRI-O or containerd (v2.0+) detect and restore instead of cold-starting. However, CRIU cannot guarantee consistent preservation of shared memory segments and IPC channels, has security implications (checkpoint data contains all memory including secrets), and requires `CAP_CHECKPOINT_RESTORE` capability. It is **better suited for migration than high-frequency suspend/resume cycles**.

**Warm pool patterns** are more immediately practical than suspend/resume for many deployments:

- **Snapshot-based cloning**: Create a "golden" Firecracker snapshot after environment initialization. Clone from snapshot for each new session. MAP_PRIVATE mapping enables efficient memory sharing between clones.
- **Pre-created pool with claim**: Maintain N ready environments (booted, initialized, waiting). On request, assign from pool and replenish in background. The `agent-sandbox` SandboxWarmPool CRD implements exactly this pattern for Kubernetes.
- **Placeholder pod pattern** (from JupyterHub): Deploy low-priority pods that reserve capacity. Real workloads arrive with higher PriorityClass, Kubernetes preempts placeholders. This provides O(seconds) allocation without custom infrastructure.
- **Tiered warmth**: Hot (running, assigned) → Warm (running, unassigned) → Cold (snapshot on disk) → Frozen (snapshot in object storage). Promote/demote based on demand signals.

**Application-level checkpointing** — serializing agent conversation state to a database and reconstructing on a fresh environment — works for conversation/reasoning state (inherently serializable as JSON) but fails for execution environment state (installed packages, running servers, compiled artifacts, browser sessions). The ideal approach for **30–120 minute agent sessions is hybrid**: infrastructure snapshot for environment state, application checkpoint for conversation context, enabling cross-infrastructure portability while preserving execution environment fidelity.

---

## 7. Kubernetes-native patterns with agent-sandbox

The **`kubernetes-sigs/agent-sandbox`** project, released under SIG Apps with its v0.1.0 release and a formal Kubernetes blog announcement on March 20, 2026, provides the first Kubernetes-native abstraction purpose-built for agent workloads. Google is the primary contributor, with GKE documentation already published for deploying agent-sandbox with gVisor.

The **core Sandbox CRD** manages a single, stateful pod with a stable hostname, persistent storage (PVC), and lifecycle management including scale-to-zero (setting replicas to 0 deletes the pod but retains the Sandbox resource and PVC, allowing later recreation). The **extensions module** adds SandboxTemplate (reusable environment definitions), SandboxClaim (consumers request a sandbox from a template), and SandboxWarmPool (maintains N pre-provisioned pods for instant allocation). A Python SDK provides a programmatic interface abstracting Kubernetes complexity.

For organizations not yet on agent-sandbox, **established Kubernetes patterns translate directly**:

**gVisor (runsc) RuntimeClass** is the most accessible sandboxing option for standard Kubernetes clusters. Performance benchmarks show **<3% overhead for CPU-bound work** (compilation, computation), with Ant Group reporting 70% of production applications running at <1% overhead. File I/O was historically gVisor's weakest area, but the rootfs overlay optimization reduced fsstress benchmarks from 262s to 3.18s. gVisor implements ~70–80% of Linux syscalls — sufficient for shell, git, compilers, npm/pip, and typical agent tooling. GKE Autopilot enables gVisor automatically for all pods.

**KEDA with custom external scalers** enables scaling agent worker deployments from 0 to N based on queue depth. A ScaledJob resource creates one Kubernetes Job per pending task — ideal for per-task agent sessions. The critical caveat: when KEDA scales to zero, all in-memory state is lost. Combining KEDA with agent-sandbox's hibernation capability (saving state to PV before scale-down) addresses this.

**Per-session PVC management** should use dynamic CSI provisioning with `volumeBindingMode: WaitForFirstConsumer` for topology-aware allocation. PVC lifecycle should be tied to session lifecycle via `ownerReferences` on the PVC pointing to the Sandbox CR, ensuring automatic garbage collection. For performance-sensitive workloads, local NVMe via `local-path-provisioner` provides the lowest latency; network SSDs (gp3, pd-ssd) are suitable for most agent workloads.

**Defense-in-depth security** for agent containers on Kubernetes should combine: RuntimeDefault seccomp profile (blocks ~44 of 300+ syscalls), drop ALL Linux capabilities, read-only root filesystem with writable emptyDir overlay, per-session NetworkPolicy restricting egress to HTTPS + DNS while blocking cluster-internal ranges, and non-root user execution. For shell/git/compiler workloads, **no special capabilities are needed** when running as non-root.

---

## 8. Scaling analysis across four architectural approaches

The choice of architecture depends heavily on scale, security requirements, and engineering investment tolerance. The following analysis uses a **64-core/256GB reference node** and assumes per-session resource profiles of ~0.2 vCPU average (1–2 vCPU burst) with 512MB–1GB RAM working set.

**Approach A: Shared fat containers** pack multiple agent sessions as processes within a single container. Cold start is **~10ms** (process fork). Maximum density reaches ~500–800 sessions with aggressive overcommit. However, the failure blast radius is **catastrophic** — any crash, OOM, or security breach affects all sessions — and process-level isolation is insufficient for executing untrusted LLM-generated code. This approach is only viable when all agents are trusted and belong to the same security domain.

**Approach B: Per-session containers with warm pool** gives each session its own OCI container. The containerd-shim adds **~10–12 MiB RSS per container** (verified via containerd GitHub issue #7878); at 800 containers, shim overhead alone reaches ~10GB. Warm pool startup is **~50–200ms** (assign existing container). Using JupyterHub-style placeholder pods with PriorityClass preemption, real session allocation happens in seconds. Maximum density: **200–300 sessions without overcommit, 400–600 with 2× memory overcommit**. This is the default recommendation for most Kubernetes deployments.

**Approach C: Per-session Firecracker microVMs** boot in **≤125ms cold, 28–200ms from snapshot**. VMM overhead is <5 MiB per microVM, but guest OS adds ~50 MiB, totaling ~55 MiB overhead per session. AWS has tested memory oversubscription ratios over 20× in production, with 10× as the operational norm for Lambda. With demand fault paging and memory balloon devices, idle VMs consume minimal physical memory. Maximum density: **400–1,000 sessions per node with 5–10× overcommit**. Alibaba's RunD achieved 2,500 sandboxes on a 384GB node. This provides the strongest security boundary but requires Firecracker-capable infrastructure (bare metal or nested virtualization).

**Approach D: Managed serverless platforms** offer the lowest operational overhead. Fly.io with suspend/resume is dramatically cheaper for idle-heavy workloads: at 80% suspension, 100 concurrent sessions costs approximately **$182/month** versus ~$3,900/month for E2B or ~$11,300/month for Modal Sandbox. Self-hosted infrastructure (cloud VM) at the same scale costs ~$1,600/month but requires engineering investment in orchestration.

| Metric | Shared container | Per-session container | Firecracker microVM | Serverless (Fly.io) |
|---|---|---|---|---|
| Cold start | ~10ms | 1–5s | 125ms | 1–12s (create) |
| Warm start | ~10ms | 50–200ms | 28–200ms (snapshot) | <1s (resume) |
| Overhead per session | 10–50 MiB | 15–25 MiB | 35–55 MiB | Platform-managed |
| Max density (256GB, 5× overcommit) | ~2,000+ | ~600 | ~1,000 | Elastic |
| Security isolation | Weakest | Medium | Strong (hardware VM) | Strong (Firecracker) |
| Failure blast radius | All sessions | Single session | Single session | Single session |
| Suspend/resume | N/A | CRIU (beta) | Native snapshot | Native |
| Monthly cost (100 sessions, self-hosted) | ~$1,600 | ~$1,600 | ~$1,600 | ~$182 |

**The critical insight: at 80% idle, aggressive overcommit is safe and essential.** JupyterHub (an analogous idle-heavy, interactive workload) routinely overcommits CPU at 5–10× on production clusters. Set low Kubernetes resource requests (0.1 CPU, 256 MiB) with higher limits (2 CPU, 2 GiB) for Burstable QoS. Memory overcommit above 2× requires monitoring — running out of CPU causes slowdowns, running out of memory causes OOM kills.

---

## 9. Architecture recommendations for three deployment tiers

### Tier 1: Self-hosted Kubernetes, minimal operational overhead

**Use per-session containers with gVisor RuntimeClass, the agent-sandbox CRD, and a warm pool.**

Deploy `kubernetes-sigs/agent-sandbox` with gVisor as the RuntimeClass. Define SandboxTemplates for your agent environment images. Configure SandboxWarmPool with replicas matching expected demand (e.g., 10 warm pods for 50 expected concurrent sessions). Use KEDA with an external scaler to dynamically adjust warm pool size based on queue depth. Apply defense-in-depth: RuntimeDefault seccomp, drop ALL capabilities, read-only root filesystem, per-session NetworkPolicy.

For workspace lifecycle, configure autostop after 30 minutes of inactivity (destroy pod, retain PVC). Use dynamic PVC provisioning with ownerReferences for automatic cleanup on session deletion. Pre-pull agent images on all nodes via DaemonSet.

**At 5–20 sessions**: Single node, minimal warm pool (3–5 pods), no autoscaling needed.
**At 20–100 sessions**: 1–2 nodes, warm pool of 10–20, KEDA for queue-based scaling, Cluster Autoscaler or Karpenter for node scaling.
**At 100–500 sessions**: 2–4 nodes, warm pool of 20–50, aggressive CPU overcommit (5×), memory overcommit (1.5–2×), JupyterHub-style user scheduler for bin-packing.

**Limitations**: gVisor provides weaker isolation than hardware VMs (shared host kernel, ~70–80% syscall coverage). No infrastructure-level snapshot/restore for session hibernation. CRIU is beta and may not reliably checkpoint agent workloads with running subprocesses.

### Tier 2: Self-hosted Kubernetes with additional tooling

**Use Coder (open-source) with Terraform templates provisioning gVisor/Kata-sandboxed pods, prebuilt workspaces as warm pools, and the full Coder lifecycle management stack.**

Deploy Coder via Helm on Kubernetes. Author Terraform templates targeting Kubernetes pods with Kata Containers RuntimeClass (for full syscall compatibility and VM-level isolation) or gVisor (for lower overhead). Enable prebuilt workspaces with `prebuild_count` to maintain warm pools. Use Coder's autostop with activity detection (won't stop workspaces with active SSH/agent connections). For agent orchestration, use Coder's REST API (`/api/v2/workspaces`) to programmatically create, start, stop, and delete agent sessions.

**Key advantages over Tier 1**: Coder provides enterprise lifecycle management (dormancy, quotas, audit logging on Premium), a web dashboard for monitoring, Agent Boundaries for network/process-level agent restrictions, AI Bridge for injecting API keys, and validated architecture documentation for scaling to 3,000+ users. The Terraform provisioning model means the same templates can target K8s pods today and Firecracker microVMs tomorrow, providing an upgrade path to Tier 3.

**At 100+ sessions**: Enable Kata Containers with Cloud Hypervisor as VMM for hardware-level isolation on nodes supporting KVM. This provides full syscall compatibility with ~150–300ms boot time and ~30–50MB memory overhead per guest OS.

### Tier 3: Unconstrained (Firecracker available, budget for tooling)

**Use Firecracker microVMs with snapshot/restore, either via E2B's open-source stack (self-hosted) or Fly.io Sprites (managed), with Coder as the orchestration layer.**

For **self-hosted**: Deploy E2B's open-source infrastructure (`e2b-dev/infra`) via Terraform on bare metal or cloud instances with KVM support. Build Firecracker microVM templates from Dockerfiles. Implement warm pools via pre-snapshotted microVMs. Use snapshot/restore for session hibernation — pause idle sessions after 5 minutes, restore in <200ms on demand. At 500 sessions on 2 bare-metal nodes, this costs ~$1,000/month (hardware amortization + hosting) versus $18,700+/month for managed E2B.

For **managed with minimal ops**: Use Fly.io Sprites for persistent agent environments. At 80% idle utilization with auto-sleep, 100 concurrent sessions costs ~$182/month. Sprites provide 100GB persistent NVMe, ~300ms checkpoint/restore, and Firecracker hardware isolation. Use the REST API for programmatic lifecycle management. For burst overflow, Sprites create in 1–12 seconds.

For **hybrid**: Run Coder as the unified control plane. Terraform templates provision Sprites or self-hosted Firecracker VMs as workspace backends. Coder handles lifecycle management, access control, and audit logging. Agents interact via Coder's API. This provides the strongest isolation (Firecracker), the most mature lifecycle management (Coder), and the best cost profile (suspend/resume eliminates idle waste).

The **ideal architecture at scale** is a warm pool of pre-snapshotted Firecracker microVMs with tiered state management: hot (running, assigned) → warm (snapshot on local NVMe) → cold (snapshot in object storage). Sessions idle for >5 minutes get snapshotted and stopped. Sessions idle for >1 hour get moved to object storage. Resume from local snapshot in <200ms; from object storage in 1–5 seconds. This pattern — proven by E2B, Fly.io, and AWS Lambda — extracts maximum density from idle-heavy agent workloads while maintaining hardware-level isolation.

---

## Conclusion: where agent compute is heading

Three tectonic shifts are reshaping this landscape. First, **Kubernetes is formally absorbing agent workloads** — the agent-sandbox CRD under SIG Apps is not an edge project but a Google-backed initiative with GKE integration, signaling that agent compute will be a first-class Kubernetes primitive alongside Deployments and StatefulSets. Second, **the ephemeral-vs-persistent debate is resolving in favor of persistent environments** — Fly.io's Sprites thesis ("agents want computers, not containers") is gaining traction as teams discover that environment rebuild costs dominate session latency for complex projects. Third, **remote dev environment platforms are converging with agent platforms** — Coder's pivot to agent support and Gitpod's total rebrand as Ona demonstrate that workspace lifecycle management *is* agent lifecycle management.

The most underappreciated technical capability is **Firecracker's demand fault paging combined with aggressive memory oversubscription**. AWS has proven 10× overcommit ratios in production with Firecracker, meaning a 256GB node can theoretically host microVMs claiming 2.5TB of aggregate memory — only pages actually touched occupy physical RAM. For 80%-idle agent workloads, this transforms the economics entirely: the binding constraint shifts from memory capacity to active compute burst capacity.

For teams making architectural decisions today: start with Tier 1 (agent-sandbox + gVisor on existing K8s) to validate workload patterns and session characteristics. Graduate to Tier 2 (Coder) when you need enterprise lifecycle management and multi-tenancy. Move to Tier 3 (Firecracker) when security requirements demand hardware isolation or scale economics justify the infrastructure investment. The portable Terraform provisioning model in Coder means this progression can happen incrementally, without rearchitecting the control plane.
