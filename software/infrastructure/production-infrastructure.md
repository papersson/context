# Production Infrastructure from the Ground Up

A practical guide to how production infrastructure works at a company with 10-50 services and a small platform team (2-5 people). Concrete tools, configs, and commands rather than abstract descriptions.

---

## 1. Cloud Infrastructure with Terraform/OpenTofu

### The problem

Infrastructure changes need to be reviewable, reproducible, and reversible. Clicking around in the AWS console gives you none of those properties. Terraform lets you declare what infrastructure should exist, track what currently exists (state), and compute the diff.

### Real directory structure

```
infra/
├── modules/                    # Reusable building blocks
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── eks-cluster/
│   ├── rds-postgres/
│   └── s3-bucket/
├── environments/
│   ├── production/
│   │   ├── main.tf            # Composes modules
│   │   ├── variables.tf
│   │   ├── terraform.tfvars   # Env-specific values
│   │   ├── backend.tf         # Where state lives
│   │   └── providers.tf       # AWS provider config
│   ├── staging/
│   │   ├── main.tf
│   │   ├── ...
│   └── dev/
│       ├── main.tf
│       └── ...
└── global/                    # Shared across environments
    ├── iam-roles/
    ├── dns-zones/
    └── ecr-repos/
```

The key insight: **modules are reusable definitions**, environments are **specific instantiations**. Your VPC module defines "a VPC with public and private subnets." Your production environment says "give me a VPC in us-east-1 with these CIDRs."

### What a module actually looks like

```hcl
# modules/vpc/main.tf

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.azs[count.index]

  tags = {
    Name = "${var.environment}-private-${var.azs[count.index]}"
    # This tag is how EKS discovers subnets for internal LBs
    "kubernetes.io/role/internal-elb" = "1"
  }
}

resource "aws_subnet" "public" {
  count                   = length(var.public_subnet_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.environment}-public-${var.azs[count.index]}"
    "kubernetes.io/role/elb" = "1"
  }
}

# NAT Gateway — one per AZ for HA, or one shared to save money
resource "aws_eip" "nat" {
  count  = var.ha_nat ? length(var.azs) : 1
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count         = var.ha_nat ? length(var.azs) : 1
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}
```

```hcl
# modules/vpc/variables.tf

variable "environment" {
  type = string
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "private_subnet_cidrs" {
  type = list(string)
}

variable "public_subnet_cidrs" {
  type = list(string)
}

variable "azs" {
  type = list(string)
}

variable "ha_nat" {
  type    = bool
  default = false
  # true = one NAT per AZ (~$100/mo), false = single NAT (~$35/mo)
}
```

```hcl
# modules/vpc/outputs.tf

output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}
```

### How an environment uses modules

```hcl
# environments/production/main.tf

module "vpc" {
  source = "../../modules/vpc"

  environment          = "production"
  vpc_cidr             = "10.0.0.0/16"
  azs                  = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnet_cidrs  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  ha_nat               = true  # prod gets HA, dev doesn't
}

module "eks" {
  source = "../../modules/eks-cluster"

  cluster_name       = "production"
  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.private_subnet_ids
  kubernetes_version = "1.29"

  node_groups = {
    general = {
      instance_types = ["m6i.xlarge"]
      min_size       = 3
      max_size       = 20
      desired_size   = 5
    }
    spot = {
      instance_types = ["m6i.xlarge", "m5.xlarge", "m5a.xlarge"]
      capacity_type  = "SPOT"
      min_size       = 0
      max_size       = 30
      desired_size   = 3
    }
  }
}

module "rds" {
  source = "../../modules/rds-postgres"

  identifier     = "production-main"
  engine_version = "15.4"
  instance_class = "db.r6g.xlarge"
  multi_az       = true  # automatic failover replica

  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.private_subnet_ids
  allowed_cidr_blocks = module.vpc.private_subnet_cidrs
}
```

### State management

Terraform state is **the mapping between your config and real resources**. If you lose it, Terraform can't manage existing infrastructure. If two people modify it simultaneously, you corrupt it.

```hcl
# environments/production/backend.tf

terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"   # DynamoDB provides locking
    encrypt        = true
  }
}
```

The DynamoDB table is critical: it prevents two engineers from running `terraform apply` simultaneously. When you run apply, Terraform acquires a lock in DynamoDB. If someone else tries to apply, they get:

```
Error: Error acquiring the state lock
Lock Info:
  ID:        a1b2c3d4-...
  Who:       alice@mycompany.com
  Operation: OperationTypeApply
  Created:   2026-04-05 10:30:00.000000 UTC
```

### Multi-account setup

Most companies use separate AWS accounts per environment, managed through an organization:

```
AWS Organization
├── Management Account (billing, org policies)
├── Production Account
├── Staging Account
├── Dev Account
├── Security Account (CloudTrail, GuardDuty)
└── Shared Services Account (ECR, DNS, CI/CD)
```

You access them by assuming roles:

```hcl
# environments/production/providers.tf

provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformAdmin"
  }

  default_tags {
    tags = {
      Environment = "production"
      ManagedBy   = "terraform"
      Repository  = "mycompany/infra"
    }
  }
}
```

### What a PR review looks like

Your CI pipeline runs `terraform plan` and posts the output to the PR:

```
Terraform will perform the following actions:

  # module.eks.aws_eks_node_group.general will be updated in-place
  ~ resource "aws_eks_node_group" "general" {
      ~ scaling_config {
          ~ desired_size = 5 -> 8
            # (2 unchanged attributes hidden)
        }
    }

  # module.rds.aws_db_instance.main will be updated in-place
  ~ resource "aws_db_instance" "main" {
      ~ instance_class = "db.r6g.xlarge" -> "db.r6g.2xlarge"
        # This will cause a reboot during the maintenance window
    }

Plan: 0 to add, 2 to change, 0 to destroy.
```

Reviewers look for:
- **Destroys** — anything being deleted is scrutinized heavily
- **Replacement** vs **in-place update** — a replace means downtime
- **Security group changes** — is someone opening port 22 to 0.0.0.0/0?
- **Cost implications** — going from xlarge to 2xlarge doubles that line item
- **State moves** — if you refactor module names, resources get destroyed and recreated unless you use `terraform state mv`

### Common mistakes

**Mistake 1: hardcoding values instead of using variables.** You end up with `cidr_block = "10.0.1.0/24"` scattered everywhere, and changing it means find-and-replace across files.

**Mistake 2: monolithic state.** Putting your entire company's infrastructure in one state file means every `plan` takes 5 minutes and touches everything. Split by environment and by concern (networking, compute, databases).

**Mistake 3: not pinning provider versions.**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"   # Pin to major.minor
    }
  }
  required_version = ">= 1.6"
}
```

Without this, a provider update can change behavior and cause unexpected changes across your infrastructure.

**Mistake 4: running apply locally.** Production applies should run in CI only, with the plan output reviewed. Local applies lead to "oops I was in the production workspace."

---

## 2. Kubernetes Cluster Setup

### The problem

You have containers. You need something to run them across multiple machines with self-healing, service discovery, load balancing, secrets management, and rolling deployments. Kubernetes is the industry standard for this.

### Managed vs self-managed

**Managed (EKS/GKE/AKS):** The cloud provider runs the control plane (API server, etcd, scheduler, controller manager). You manage worker nodes and everything that runs on them.

| Concern | Managed (EKS) | Self-managed (kubeadm/RKE2) |
|---------|---------------|---------------------------|
| Control plane uptime | AWS's problem | Your problem |
| etcd backups | Automatic | You manage this |
| K8s version upgrades | One button (with caveats) | Manual, multi-step |
| Cost | ~$75/mo per cluster + nodes | Just the nodes |
| Customization | Limited | Full control |
| When to choose | Default choice for cloud | On-prem, air-gapped, edge |

For a 10-50 service company: **always go managed unless you have a specific reason not to** (on-prem requirement, regulatory, air-gapped).

### EKS cluster via Terraform

```hcl
# modules/eks-cluster/main.tf

resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  version  = var.kubernetes_version
  role_arn = aws_iam_role.cluster.arn

  vpc_config {
    subnet_ids              = var.subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = true   # for kubectl from dev laptops
    public_access_cidrs     = var.allowed_cidrs  # restrict by office IP
  }

  # Enable control plane logging
  enabled_cluster_log_types = [
    "api", "audit", "authenticator", "controllerManager", "scheduler"
  ]
}

resource "aws_eks_node_group" "main" {
  for_each = var.node_groups

  cluster_name    = aws_eks_cluster.main.name
  node_group_name = each.key
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.subnet_ids
  instance_types  = each.value.instance_types
  capacity_type   = lookup(each.value, "capacity_type", "ON_DEMAND")

  scaling_config {
    min_size     = each.value.min_size
    max_size     = each.value.max_size
    desired_size = each.value.desired_size
  }

  update_config {
    max_unavailable_percentage = 25  # rolling update: 25% at a time
  }

  labels = {
    role = each.key
  }

  # Taint spot nodes so only tolerating workloads get scheduled there
  dynamic "taint" {
    for_each = lookup(each.value, "capacity_type", "ON_DEMAND") == "SPOT" ? [1] : []
    content {
      key    = "spot"
      value  = "true"
      effect = "NO_SCHEDULE"
    }
  }
}
```

### What you install on top of the cluster (day-2 components)

The bare cluster can run pods, but production needs more. This is typically managed via Helm charts and/or GitOps:

```
cluster-addons/
├── ingress-nginx/           # Incoming HTTP traffic
│   ├── values-production.yaml
│   └── values-staging.yaml
├── cert-manager/            # Automatic TLS certificates
│   └── values.yaml
├── external-dns/            # Auto-creates DNS records from ingress
│   └── values.yaml
├── cluster-autoscaler/      # Adds/removes nodes based on demand
│   └── values.yaml
├── metrics-server/          # Pod resource metrics (for HPA)
│   └── values.yaml
├── kube-prometheus-stack/   # Prometheus + Grafana
│   └── values.yaml
├── loki/                    # Log aggregation
│   └── values.yaml
├── external-secrets/        # Sync secrets from AWS Secrets Manager
│   └── values.yaml
└── argocd/                  # GitOps controller
    └── values.yaml
```

Example: ingress-nginx Helm values:

```yaml
# cluster-addons/ingress-nginx/values-production.yaml

controller:
  replicaCount: 3
  # This creates an AWS NLB (Network Load Balancer)
  service:
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
      service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      memory: 512Mi
  # Spread across nodes/AZs
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
```

### Day-2 concerns

**Cluster upgrades:** Kubernetes releases every ~4 months and supports 3 minor versions. You're always on a clock. The process:

1. Read the changelog for deprecations and breaking changes
2. Upgrade the control plane (managed = one API call, takes ~20 min)
3. Upgrade node groups one at a time (rolling replacement of nodes)
4. Update cluster add-ons to compatible versions

```bash
# EKS upgrade sequence
aws eks update-cluster-version --name production --kubernetes-version 1.30
# Wait for control plane...
aws eks update-nodegroup-version --cluster-name production --nodegroup-name general
# This cordons old nodes, drains pods, launches new nodes
```

**Autoscaling** operates at two levels:
- **Horizontal Pod Autoscaler (HPA):** scales the number of pods based on CPU/memory/custom metrics
- **Cluster Autoscaler (or Karpenter):** scales the number of nodes when pods can't be scheduled

```yaml
# HPA example
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5 min before scaling down
```

**etcd backups (self-managed only):** If etcd dies and you don't have a backup, you've lost your entire cluster state — every deployment, service, secret, configmap. Managed services handle this, but for self-managed:

```bash
# Snapshot etcd — run this on a cron every hour
ETCDCTL_API=3 etcdctl snapshot save /backups/etcd-$(date +%Y%m%d-%H%M).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

## 3. On-Prem and Hybrid

### The problem

Not everything can be in the cloud. Reasons: data sovereignty regulations, latency requirements (factory floor, trading), existing hardware investments, egress cost at scale (moving terabytes out of AWS is expensive), or simply organizational inertia.

### Bare metal provisioning

When you rack a new server, it has no OS. You need automated provisioning:

**PXE boot flow:**
1. Server powers on, BIOS/UEFI set to network boot
2. Server broadcasts DHCP request
3. DHCP server responds with IP + address of TFTP server
4. Server downloads bootloader via TFTP
5. Bootloader downloads kernel + initrd + a kickstart/preseed/cloud-init config
6. OS installs unattended
7. On first boot, runs configuration management (Ansible/Salt) to join the cluster

**MAAS (Metal as a Service)** from Canonical abstracts this into a UI/API. You register machines by MAC address, and MAAS handles the PXE -> OS install -> commissioning pipeline. You can then "deploy" a machine to a specific OS with an API call, similar to how you'd launch an EC2 instance.

```bash
# MAAS CLI example
maas admin machines allocate system_id=abc123
maas admin machine deploy abc123 distro_series=jammy
# Machine PXE boots, installs Ubuntu 22.04, and becomes available
```

### vSphere/VMware

Most enterprise on-prem runs VMware. The stack:
- **ESXi**: Hypervisor that runs directly on hardware
- **vCenter**: Centralized management of multiple ESXi hosts
- **vSAN**: Software-defined storage across local disks
- **NSX**: Software-defined networking (overlay networks, firewalling)

You can manage vSphere with Terraform:

```hcl
provider "vsphere" {
  vsphere_server = "vcenter.internal.company.com"
}

resource "vsphere_virtual_machine" "k8s_worker" {
  count            = 5
  name             = "k8s-worker-${count.index + 1}"
  resource_pool_id = data.vsphere_resource_pool.pool.id
  datastore_id     = data.vsphere_datastore.ssd.id

  num_cpus = 8
  memory   = 32768  # 32 GB

  network_interface {
    network_id = data.vsphere_network.vm_network.id
  }

  disk {
    label            = "disk0"
    size             = 200
    thin_provisioned = true
  }

  clone {
    template_uuid = data.vsphere_virtual_machine.ubuntu_template.id
    customize {
      linux_options {
        host_name = "k8s-worker-${count.index + 1}"
        domain    = "internal.company.com"
      }
      network_interface {
        ipv4_address = "10.20.30.${count.index + 10}"
        ipv4_netmask = 24
      }
      ipv4_gateway = "10.20.30.1"
    }
  }
}
```

### Networking differences

| Cloud | On-prem |
|-------|---------|
| VPCs are isolated by default, you explicitly allow traffic | Flat network by default, you segment with VLANs |
| Security groups (stateful L4 firewalls) | Physical firewalls, NSX distributed firewall, or ACLs on switches |
| Route tables are API-managed | BGP peering between routers, sometimes manual static routes |
| Load balancers are a managed service | F5 Big-IP, HAProxy, or MetalLB |
| DNS is Route53/Cloud DNS | Bind, PowerDNS, or Active Directory DNS |

**VLANs** are the on-prem equivalent of subnets: you tag switch ports with a VLAN ID to isolate traffic. A server might have VLAN 100 for application traffic and VLAN 200 for storage replication:

```yaml
# /etc/netplan/01-config.yaml (on Ubuntu)
network:
  ethernets:
    eno1: {}
  vlans:
    vlan100:
      id: 100
      link: eno1
      addresses: [10.100.0.15/24]
    vlan200:
      id: 200
      link: eno1
      addresses: [10.200.0.15/24]
```

**BGP** is how routers exchange routes. In on-prem Kubernetes, you use BGP to advertise service IPs to the network. MetalLB (a bare-metal load balancer for K8s) peers with your router via BGP:

```yaml
# MetalLB configuration
apiVersion: metallb.io/v1beta1
kind: BGPPeer
metadata:
  name: router
  namespace: metallb-system
spec:
  myASN: 64500         # Your cluster's autonomous system number
  peerASN: 64501       # Your router's ASN
  peerAddress: 10.0.0.1
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: service-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.0.100.0/24    # IPs MetalLB can assign to LoadBalancer services
```

### Running workloads across cloud and on-prem

The platform team's job is to **make both environments look the same to application developers.** Strategies:

1. **Same Kubernetes API everywhere.** Whether the cluster runs on EKS or RKE2 on bare metal, developers write the same Deployment YAML. The platform team handles the differences (storage classes, load balancer implementations, ingress controllers).

2. **Cluster federation** (Rancher, or fleet management): a single control plane manages multiple clusters. You deploy an application once and it gets placed in the right cluster(s).

3. **GitOps as the abstraction**: ArgoCD watches a git repo and applies manifests to each cluster. The repo structure encodes what goes where:

```
gitops/
├── base/                      # Shared manifests
│   └── api-service/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
├── overlays/
│   ├── cloud-production/      # AWS-specific overrides
│   │   ├── kustomization.yaml
│   │   └── ingress-patch.yaml # Uses ALB ingress
│   └── onprem-production/     # On-prem-specific overrides
│       ├── kustomization.yaml
│       └── ingress-patch.yaml # Uses nginx + MetalLB
```

### What the platform team abstracts away

Application developers should NOT need to know:
- Which cluster or region their service runs in
- Whether the underlying node is EC2, bare metal, or a VM
- How the load balancer works (ALB vs MetalLB vs F5)
- How secrets get from Vault/Secrets Manager into their pods
- How logs and metrics get collected

They SHOULD know:
- How to write a Deployment/Service YAML (or use a template)
- How to set resource requests/limits
- How to configure their service's specific alerts
- How to read dashboards and logs

---

## 4. Networking in Depth

### VPC architecture

A production VPC typically has three tiers of subnets across multiple availability zones:

```
VPC: 10.0.0.0/16

+---------------------------------------------------------+
|                      AZ-a         AZ-b         AZ-c     |
|                                                          |
|  Public subnets:   10.0.1.0/24  10.0.2.0/24  10.0.3.0/24
|  (ALB, NAT GW)                                          |
|                                                          |
|  Private subnets:  10.0.11.0/24 10.0.12.0/24 10.0.13.0/24
|  (App pods, EKS)                                        |
|                                                          |
|  Data subnets:     10.0.21.0/24 10.0.22.0/24 10.0.23.0/24
|  (RDS, ElastiCache)                                     |
+---------------------------------------------------------+
```

**Security groups** are stateful firewalls attached to resources. They're the primary access control mechanism:

```hcl
# RDS security group — only allows traffic from EKS nodes
resource "aws_security_group" "rds" {
  name   = "rds-postgres"
  vpc_id = module.vpc.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [module.eks.node_security_group_id]
    # Only EKS nodes can reach port 5432 — not the public subnets,
    # not other services, not even bastion hosts (unless you add them)
  }
}
```

**NAT gateways** let private subnets reach the internet (for pulling container images, calling external APIs) without being reachable FROM the internet. They're expensive (~$35/month + $0.045/GB processed), which is why dev environments often share one while production gets one per AZ.

**VPC peering / Transit Gateway:** When you have multiple VPCs (one per environment, or shared services), they need to communicate. Peering is point-to-point (fine for 2-3 VPCs), Transit Gateway is a hub (necessary when you have 5+):

```hcl
resource "aws_ec2_transit_gateway" "main" {
  description = "Central hub for all VPCs"
  auto_accept_shared_attachments = "enable"
}

resource "aws_ec2_transit_gateway_vpc_attachment" "production" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = module.production_vpc.vpc_id
  subnet_ids         = module.production_vpc.private_subnet_ids
}

resource "aws_ec2_transit_gateway_vpc_attachment" "shared_services" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = module.shared_services_vpc.vpc_id
  subnet_ids         = module.shared_services_vpc.private_subnet_ids
}
```

### Full request path: internet to pod

Tracing a request to `api.example.com/users/123` through every hop:

```
1. CLIENT: Browser resolves api.example.com
   -> DNS query to Route53

2. ROUTE53: Returns the ALB's IP (or CNAME to ALB DNS name)
   -> api.example.com -> ALIAS -> k8s-prod-ingress-abc123.us-east-1.elb.amazonaws.com

3. ALB (Application Load Balancer): Receives HTTPS request
   - Terminates TLS (holds the certificate)
   - Evaluates listener rules
   - Forwards to a target group (the NodePort on EKS nodes)
   -> Sends HTTP to 10.0.11.45:32080 (a random EKS node, port 32080)

4. NODE (kube-proxy / iptables): Receives on NodePort 32080
   - iptables rules (managed by kube-proxy) match this to a Service
   - DNAT to a pod IP: 10.0.11.45:32080 -> 172.16.5.23:8080
   -> Forwards to pod IP 172.16.5.23:8080

5. CNI (Container Network Interface): Routes to the pod
   - If the pod is on this node: routes directly via veth pair
   - If the pod is on another node: encapsulates (VXLAN) or uses VPC routing
   -> Packet arrives at the pod's network namespace

6. POD: Your application container receives the HTTP request
   - Sees src=10.0.11.45 (the node) dst=172.16.5.23:8080
   - Processes request, returns response
   - Response follows the reverse path
```

With an **ingress controller** (nginx or AWS ALB Ingress Controller), step 3-4 changes slightly: the ALB routes directly to pod IPs (using the AWS VPC CNI plugin, which assigns real VPC IPs to pods), skipping the NodePort hop.

### How the Ingress resource works

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls  # cert-manager fills this
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

The ingress controller watches for Ingress resources and reconfigures its nginx (or envoy) accordingly. It's a pod running inside the cluster that acts as a reverse proxy.

### Pod-to-pod networking

Every pod gets a unique IP. Pod A on Node 1 needs to reach Pod B on Node 2. How?

**AWS VPC CNI (native routing):** Each pod gets an actual VPC IP address from the node's subnet. The VPC route table already knows how to route between subnets. No overlay network needed.

```
Pod A (172.16.5.23) on Node 1 (10.0.11.45)
  -> Packet to Pod B (172.16.5.87) on Node 2 (10.0.11.46)
  -> Leaves Node 1 via ENI (Elastic Network Interface)
  -> VPC routing delivers it to Node 2
  -> Node 2's CNI routes it to Pod B
```

This is fast (no encapsulation overhead) but has a limitation: you can run out of IPs. Each EC2 instance type has a max number of ENIs and IPs per ENI. An `m5.xlarge` can support ~58 pod IPs.

**Overlay networks (Calico VXLAN, Flannel, Cilium):** Used on-prem or when you want pod IPs independent of the underlying network. Pods get IPs from a cluster-internal range (e.g., 192.168.0.0/16). When a packet crosses nodes, it gets encapsulated:

```
Pod A (192.168.1.5) -> Pod B (192.168.2.10)
  -> Node 1 wraps the packet: outer IP = Node 1 (10.0.11.45),
     inner IP = 192.168.1.5 -> 192.168.2.10
  -> Normal network routing delivers the outer packet to Node 2
  -> Node 2 unwraps it and delivers to Pod B
```

**Cilium with eBPF** is increasingly the default choice: it replaces iptables (slow at scale) with eBPF programs in the kernel, supports native routing (no overlay when possible), and adds network policy enforcement, observability, and encryption — all in one CNI.

---

## 5. Storage and Stateful Workloads

### Why databases are hard in Kubernetes

Kubernetes is designed around **cattle, not pets**: pods are ephemeral, can be rescheduled anywhere, and local disk dies with the pod. Databases need the opposite: stable identity, persistent storage, careful ordering of start/stop, and often manual failover decisions.

**StatefulSets** (not Deployments) solve some of this:
- Pods get stable names: `postgres-0`, `postgres-1`, `postgres-2`
- Pods start and stop in order
- Each pod gets its own PersistentVolumeClaim that follows it

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          resources:
            requests:
              cpu: "2"
              memory: 8Gi
            limits:
              memory: 8Gi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3-encrypted
        resources:
          requests:
            storage: 100Gi
```

### Storage classes and CSI drivers

A **StorageClass** tells Kubernetes how to provision volumes:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
provisioner: ebs.csi.aws.com    # CSI driver for AWS EBS
parameters:
  type: gp3
  encrypted: "true"
  throughput: "125"              # MB/s
  iops: "3000"
reclaimPolicy: Retain            # Don't delete the volume when PVC is deleted
volumeBindingMode: WaitForFirstConsumer  # Don't provision until a pod needs it
allowVolumeExpansion: true       # Can resize later
```

The **CSI (Container Storage Interface)** driver is a set of pods that talk to the cloud provider's storage API. When a pod needs a volume, Kubernetes calls the CSI driver, which calls the AWS API to create an EBS volume, attaches it to the EC2 instance, and mounts it.

On-prem, the CSI driver might talk to:
- **Ceph/Rook**: Software-defined storage using local disks across nodes
- **NetApp/Pure Storage**: Enterprise SAN/NAS
- **Longhorn**: Lightweight distributed storage for smaller deployments

### Managed vs self-hosted databases

**Use managed databases (RDS, Cloud SQL) when:**
- You don't have a dedicated DBA
- You need automated backups, point-in-time recovery, failover
- The database isn't performance-critical enough to justify tuning
- You want the vendor to handle security patches and version upgrades

**Self-host when:**
- Regulatory requirements (some industries require data on specific hardware)
- Performance: you need specific kernel/storage tuning, or io_uring, or huge pages
- Cost: at very high scale, managed databases become expensive ($20K+/mo for large RDS)
- On-prem: you don't have a choice

```hcl
# Typical RDS setup in Terraform
module "rds" {
  source = "../../modules/rds-postgres"

  identifier           = "production-main"
  engine_version       = "15.4"
  instance_class       = "db.r6g.xlarge"   # 4 vCPU, 32 GB RAM
  allocated_storage    = 500                # GB, gp3
  max_allocated_storage = 1000             # auto-scaling

  multi_az             = true              # synchronous standby in another AZ
  deletion_protection  = true              # prevents accidental terraform destroy

  backup_retention_period = 30             # days of automated backups
  backup_window          = "03:00-04:00"   # UTC

  # Network
  db_subnet_group_name   = aws_db_subnet_group.data.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  # Monitoring
  performance_insights_enabled    = true
  monitoring_interval             = 60     # enhanced monitoring (seconds)
  monitoring_role_arn             = aws_iam_role.rds_monitoring.arn
}
```

### Backup strategies

For managed databases, backups are mostly handled for you. For self-hosted:

1. **Logical backups** (`pg_dump`): Portable, human-readable, slow to restore at scale
2. **Physical backups** (pgBackRest, WAL-G): File-level copy of data directory + WAL files. Fast restore, supports point-in-time recovery

```yaml
# A CronJob to backup a self-hosted Postgres using WAL-G
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 */6 * * *"   # every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: wal-g/wal-g:latest
              command: ["wal-g", "backup-push", "/var/lib/postgresql/data"]
              env:
                - name: AWS_S3_BUCKET
                  value: "mycompany-postgres-backups"
                - name: WALG_COMPRESSION_METHOD
                  value: "lz4"
          restartPolicy: OnFailure
```

**Test your restores regularly.** A backup you haven't restored is just a hypothesis. Many teams have a weekly automated job that restores the latest backup to a scratch database and runs a basic sanity check.

---

## 6. Observability Infrastructure

### The problem

When something breaks at 3 AM, you need to answer three questions fast:
1. **What** is broken? (Alerts from metrics)
2. **Where** is it broken? (Traces showing which service/dependency)
3. **Why** is it broken? (Logs from the relevant timeframe)

### The three pillars, concretely

**Metrics (Prometheus + Grafana):**

```yaml
# kube-prometheus-stack values.yaml (Helm)
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3-encrypted
          resources:
            requests:
              storage: 200Gi
    # Scrape all pods with a prometheus.io/scrape annotation
    additionalScrapeConfigs:
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)

grafana:
  persistence:
    enabled: true
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: default
          folder: ''
          type: file
          options:
            path: /var/lib/grafana/dashboards
```

Applications expose metrics at `/metrics` in Prometheus format:

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",path="/api/users",status="200"} 45892
http_requests_total{method="GET",path="/api/users",status="500"} 12

# HELP http_request_duration_seconds Request latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{path="/api/users",le="0.1"} 42000
http_request_duration_seconds_bucket{path="/api/users",le="0.5"} 45000
http_request_duration_seconds_bucket{path="/api/users",le="1.0"} 45800
http_request_duration_seconds_bucket{path="/api/users",le="+Inf"} 45892
```

**Logging (Loki or ELK):**

Loki is the Prometheus-equivalent for logs: it indexes metadata (labels) not content, making it much cheaper than Elasticsearch at scale.

```yaml
# promtail values.yaml
config:
  clients:
    - url: http://loki-gateway/loki/api/v1/push
  snippets:
    pipelineStages:
      - docker: {}       # Parse Docker JSON log format
      - json:            # Parse app JSON logs
          expressions:
            level: level
            msg: msg
      - labels:
          level:         # Add log level as a label for filtering
```

Query logs in Grafana using LogQL:

```
# Show errors from the api-service in the last hour
{namespace="production", app="api-service"} |= "error" | json | level = "ERROR"

# Count errors per minute
sum(rate({app="api-service"} |= "error" [1m])) by (pod)
```

**Tracing (Jaeger or Tempo):**

Distributed tracing follows a single request through multiple services:

```
Request: GET /api/orders/123
+- api-gateway (12ms)
|  +- auth-service.verify-token (3ms)
|  +- order-service.get-order (45ms)
|     +- postgres.query (8ms)
|     +- inventory-service.check-stock (120ms)  <-- SLOW
|     |  +- redis.get (2ms)
|     +- pricing-service.calculate (5ms)
+- Total: 185ms
```

Applications instrument with OpenTelemetry, which sends spans to a collector:

```yaml
# OpenTelemetry Collector config
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      batch:
        timeout: 5s
    exporters:
      otlp/tempo:
        endpoint: tempo-distributor:4317
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlp/tempo]
```

### Platform team vs service team responsibilities

| Concern | Platform team provides | Service team configures |
|---------|----------------------|----------------------|
| Metrics collection | Prometheus + scrape config | `/metrics` endpoint, custom metrics |
| Dashboards | Cluster-wide dashboards, templates | Service-specific dashboards |
| Logging | Promtail/Loki pipeline | Structured log format, useful log messages |
| Tracing | OTel collector, Tempo | OTel SDK instrumentation in their code |
| Alerting | Alert routing, PagerDuty integration | Service-specific alert rules |
| On-call | Cluster/infra on-call rotation | Service on-call rotation |

### Alert routing

```yaml
# Alertmanager config — routes alerts to the right team
route:
  receiver: platform-team-slack
  group_by: [alertname, namespace]
  group_wait: 30s        # wait to batch related alerts
  group_interval: 5m
  repeat_interval: 4h

  routes:
    # Critical alerts page the on-call engineer
    - match:
        severity: critical
      receiver: pagerduty-oncall
      continue: true     # also send to Slack

    # Service-specific alerts go to the owning team
    - match:
        namespace: payments
      receiver: payments-team-slack
    - match:
        namespace: search
      receiver: search-team-slack

receivers:
  - name: pagerduty-oncall
    pagerduty_configs:
      - service_key: <key>
        severity: '{{ .CommonLabels.severity }}'
  - name: platform-team-slack
    slack_configs:
      - channel: '#platform-alerts'
        title: '{{ .CommonLabels.alertname }}'
        text: '{{ .CommonAnnotations.description }}'
```

Example alert rule:

```yaml
groups:
  - name: api-service
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)
          > 0.05
        for: 5m    # must be true for 5 minutes before firing
        labels:
          severity: critical
        annotations:
          description: >-
            {{ $labels.service }} has >5% error rate
            ({{ $value | humanizePercentage }})
```

---

## 7. Cost Management

### How cloud costs actually work

The big three cost categories:

**Compute (~50-70% of bill):**
- EC2/GCE instances billed per second
- An `m6i.xlarge` (4 vCPU, 16 GB) costs ~$140/month on-demand
- EKS nodes are just EC2 instances + a $75/month cluster fee

**Storage (~10-20%):**
- EBS (block storage): gp3 = $0.08/GB/month. 100GB across 50 services = $400/month. Not expensive until you over-provision.
- S3: $0.023/GB/month for standard. Cheap. But lifecycle policies matter — terabytes of old logs add up.
- RDS storage: $0.115/GB/month for gp3. The instance cost dominates though.

**Networking (~10-20%, and this is where surprises live):**
- Ingress (data IN): **free**
- Egress (data OUT to internet): $0.09/GB. This is where companies get surprised.
- Cross-AZ traffic: $0.01/GB each way. If your service talks to a database in another AZ at high throughput, this adds up fast.
- NAT Gateway processing: $0.045/GB. Pulling container images through NAT gets expensive.

### What surprises people

1. **NAT gateway costs.** A cluster pulling images, calling external APIs, and downloading packages through a NAT gateway can easily generate $500-2000/month in NAT fees alone. Mitigations: VPC endpoints for S3/ECR (free), pull-through caches, pre-baked AMIs.

2. **Cross-AZ traffic.** A chatty microservice architecture where Service A (in AZ-a) calls Service B (in AZ-b) 10,000 times per second with 10KB payloads: that's ~$260/month for just that one pair. Multiply by many services and it's significant.

3. **Idle resources.** Dev/staging environments running 24/7 when they're only used during business hours. A staging cluster of 5 `m5.xlarge` costs $700/month just sitting there on weekends.

4. **EBS volumes outliving their instances.** Terraform creates a volume, instance gets terminated, volume stays (especially with `Retain` reclaim policy). Orphaned volumes accumulate.

5. **Log storage.** CloudWatch Logs ingestion is $0.50/GB. A verbose application logging 10GB/day costs $150/month just for ingestion, plus storage. This is why teams use Loki or self-hosted ELK instead.

### Cost optimization strategies

**Reserved Instances / Savings Plans:**

```
On-demand m6i.xlarge:  $0.192/hr = $140/month
1-year reserved (no upfront): $0.121/hr = $88/month (37% savings)
3-year reserved (all upfront): $0.076/hr = $55/month (60% savings)
```

Reserve your baseline compute (the minimum nodes you always run). Pay on-demand or spot for burst capacity.

**Spot instances** are unused EC2 capacity at 60-90% discount, but AWS can terminate them with 2 minutes notice. Perfect for:
- Batch processing, CI/CD jobs
- Stateless workloads that can tolerate interruption
- Kubernetes pods with graceful shutdown handling

```yaml
# Karpenter provisioner for spot
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-workloads
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values:
            # Multiple types for better spot availability
            - m6i.xlarge
            - m5.xlarge
            - m5a.xlarge
            - m6a.xlarge
            - c6i.xlarge
      taints:
        - key: spot
          value: "true"
          effect: NoSchedule
  limits:
    cpu: 200           # max 200 vCPUs of spot
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 30s
```

**Cost tracking per team/service:**

Use Kubernetes labels + a cost allocation tool (Kubecost, OpenCost):

```yaml
# Every deployment gets cost-tracking labels
metadata:
  labels:
    team: payments        # which team owns it
    service: payment-api  # which service
    environment: production
```

Kubecost reads these labels plus cloud billing data and produces per-service cost breakdowns: "The payments team's services cost $3,200/month: $1,800 compute, $400 RDS, $600 storage, $400 network."

**Right-sizing** means matching resource requests to actual usage:

```bash
# Check actual vs requested resources
kubectl top pods -n production
# NAME                          CPU(cores)   MEMORY(bytes)
# api-service-7d4f5c8b9-x2k4l  45m          256Mi
# api-service-7d4f5c8b9-r9m2p  52m          248Mi

# But the deployment requests 500m CPU and 1Gi memory — 10x over-provisioned
# This wastes capacity and inflates autoscaling
```

---

## 8. The Platform Team's Actual Work

### What a 2-5 person platform team does

A week might look like:

**Monday:**
- Review and merge Terraform PRs from last week
- Cluster-autoscaler is thrashing (scaling up and down rapidly) — investigate and tune stabilization window
- Onboarding a new service team: set up their namespace, RBAC, CI pipeline template

**Tuesday:**
- Kubernetes 1.29 -> 1.30 upgrade on staging
  - Read changelog for deprecated APIs
  - Run `pluto detect-all-in-cluster` to find deprecated resources
  - Upgrade control plane, then node groups
  - Run integration tests, verify all add-ons still work
- Respond to developer question: "why is my pod OOMKilled?" -> they set a 256Mi memory limit on a Java app

**Wednesday:**
- Incident: production latency spike
  - PagerDuty alert fires at 09:14
  - Check Grafana dashboards — one node group is at 95% CPU
  - Cluster autoscaler can't add nodes — ASG max limit reached
  - Short-term: increase max from 20 to 30 nodes
  - Long-term: right-size over-requesting services, consider Karpenter
  - Write incident postmortem

**Thursday:**
- Build a Helm chart template (internal "golden path") so new services don't need to write their own Deployment YAML from scratch
- Code review on a PR adding a new RDS instance for the recommendations team
- Investigate why Loki is dropping logs — the ingestion rate is hitting limits

**Friday:**
- Capacity planning for next quarter: projected growth, reserved instance coverage, any new services launching
- Rotate SSH keys on bastion hosts
- Update runbooks for common incidents

### Developer self-service

The platform team's goal is to minimize how often developers need to talk to them. This means:

**1. Internal developer platform (IDP):**

A `service.yaml` or CRD that developers fill out, and automation handles the rest:

```yaml
# What a developer submits to create a new service
apiVersion: platform.company.com/v1
kind: Service
metadata:
  name: recommendation-engine
  namespace: recommendations
spec:
  team: ml-team
  language: python
  port: 8080

  resources:
    cpu: "1"
    memory: 2Gi

  scaling:
    min: 2
    max: 20
    targetCPU: 70

  dependencies:
    - type: postgres
      size: small        # maps to db.t3.medium, 50GB
    - type: redis
      size: small        # maps to cache.t3.micro

  ingress:
    host: recs.internal.company.com
    public: false        # internal only

  alerts:
    oncall: ml-team-pagerduty
```

This triggers automation that creates: Kubernetes namespace, RBAC, deployment templates, RDS instance, ElastiCache instance, DNS record, monitoring dashboard, CI/CD pipeline, and cost allocation tags.

**2. CI pipeline templates:**

```yaml
# .github/workflows/deploy.yaml — provided as a reusable workflow
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: company/platform-workflows/.github/workflows/build-and-deploy.yaml@v2
    with:
      service-name: api-service
      cluster: production
      namespace: api
    secrets: inherit
    # The platform team maintains this reusable workflow.
    # It handles: build Docker image, push to ECR, update ArgoCD,
    # wait for rollout, run smoke tests, alert on failure.
```

### On-call rotation

Typical structure:

```
Primary on-call: 1 person, 1-week rotation
Secondary on-call: 1 person, escalation after 15 min

Rotation: A -> B -> C -> D -> A (4 people, each on primary every 4 weeks)

Alert escalation:
  0 min  -> Slack notification to #platform-alerts
  5 min  -> PagerDuty pages primary on-call (phone + push notification)
  15 min -> PagerDuty pages secondary
  30 min -> PagerDuty pages engineering manager
```

Common alerts and their typical causes:

| Alert | Likely cause | Action |
|-------|-------------|--------|
| `NodeNotReady` | Node hardware/kernel issue, kubelet OOM | Cordon + drain the node, investigate |
| `PodCrashLoopBackOff` | App bug, misconfigured env var, missing secret | Check pod logs, pass to service team |
| `HighMemoryPressure` | Memory leak or under-provisioned limits | Identify which pods, increase limits or fix leak |
| `CertificateExpiringSoon` | cert-manager renewal failed | Check cert-manager logs, usually a DNS challenge issue |
| `EtcdHighCommitDuration` | Slow disk I/O on etcd nodes | Move etcd to faster storage |
| `PrometheusTargetDown` | Service crashed or metrics endpoint changed | Check if service is up, fix scrape config |
| `DiskPressure` | Logs or images filling the node disk | Clean old images, check for verbose logging |

### Common failure modes

**1. The 3 AM cascading failure:** One service starts returning 500s, callers retry, retry storms overwhelm the downstream service, which overwhelms the database. Fix: circuit breakers, retry budgets, and rate limiting.

**2. "It works on my machine" -> broken in prod:** The CI builds a different image than what ran locally. Fix: developers test with the same Docker image, not just `python main.py`.

**3. Gradual resource exhaustion:** Memory leak that only manifests after 3 days. Pod doesn't OOM because the limit is set high. Metrics show slow creep. Fix: memory profiling, and alerts on memory growth rate (not just absolute).

**4. DNS resolution failures during deployment:** A new deployment creates new pods. Old pod IPs linger in DNS caches (kube-dns caching, application connection pools). Services get connection errors for 30-60 seconds. Fix: proper readiness probes, graceful shutdown (preStop hooks), and application-level connection retry.

**5. Terraform state drift:** Someone made a change in the console. Terraform plan now shows unexpected changes. Fix: policy that all changes go through Terraform, use AWS Config or Drift Detection to catch manual changes.

**6. Certificate renewal failure at 3 AM on a Saturday:** cert-manager uses DNS-01 challenges. The IAM role for Route53 access got rotated. Nobody noticed until the cert expired. Fix: alert on certificates expiring in <14 days (not just <1 day).

---

## Putting it all together

For a company with 10-50 services and a 3-person platform team, the realistic stack:

```
Cloud: AWS (single region, 3 AZs)
Accounts: production, staging, dev, shared-services
IaC: Terraform with S3 backend, modules in a monorepo
Compute: EKS (managed K8s), mix of on-demand + spot nodes
Networking: VPC with public/private subnets, ALB for ingress
Databases: RDS Postgres for most services, ElastiCache Redis for caching
CI/CD: GitHub Actions -> build Docker image -> ArgoCD deploys to K8s
GitOps: ArgoCD watches a manifests repo, auto-syncs to clusters
Secrets: AWS Secrets Manager -> External Secrets Operator -> K8s Secrets
Monitoring: Prometheus + Grafana (kube-prometheus-stack Helm chart)
Logging: Promtail -> Loki -> Grafana
Tracing: OpenTelemetry -> Tempo -> Grafana
Alerting: Alertmanager -> PagerDuty + Slack
Cost: Kubecost for allocation, monthly review meeting
Developer experience: Helm chart template, reusable CI workflows, namespace self-service
```

> **Sidenote: Nix.** One cross-cutting tool worth knowing about is [Nix](https://nixos.org). It's a purely functional package manager that gives bit-for-bit reproducible environments. Its strongest fit in this stack is **developer environments**: a `flake.nix` at the repo root pins exact versions of terraform, kubectl, helm, awscli, etc. — `nix develop` and every engineer gets identical tooling with no manual setup. It can also build minimal Docker images (no shell, no package manager — just the app and its dependencies) and, via NixOS, provide an entire server OS with atomic rollbacks (useful for on-prem bare metal). The tradeoff is a steep learning curve and a small hiring pool. For a small platform team: dev shells are worth adopting incrementally; NixOS for servers is a bigger commitment that only pays off if someone on the team already knows Nix.

The platform team's value proposition is simple: **application developers should be able to deploy a new service by writing a Dockerfile and some YAML, without understanding any of the infrastructure underneath.** The platform team builds and maintains the machinery that makes that possible.
