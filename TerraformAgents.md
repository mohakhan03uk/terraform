# HCP Terraform Agents

> **Scope:** Self-hosted agents for HCP Terraform/Terraform Cloud that execute runs inside private networks/on‑prem while HCP Terraform orchestrates. [Overview](https://developer.hashicorp.com/terraform/cloud-docs/agents)\
> Each agent process runs a single Terraform run at a time.\
> You may run multiple agents… up to the organization’s purchased agent limit\
> Free Tier : No agents allowed (Agents are a Business-tier feature)\
> Standard Tier : Agents allowed but quantity depends on plan entitlement\
> Business Tier : Highest limit; number of agents allowed is tied to your purchased concurrent runs\
> Terraform Enterprise (Self‑hosted) : Unlimited agents. 
---

## 1) Why Agents Can Manage On‑Prem Systems (vSphere, Nutanix, OpenStack)

HCP Terraform Agents run *inside your private network*. That means the **Terraform providers** (e.g., **vSphere**, **Nutanix**, **OpenStack**, **Cisco ACI**, **F5 BIG‑IP**) execute **API calls to private endpoints** from within your security boundary. The agent itself is only a **Terraform runner** — it **does not** contain VMware logic — but when it runs your Terraform code, the relevant provider libraries (like the vSphere provider) perform the actual operations against vCenter/ESXi APIs.  
Sources: [Agents overview](https://developer.hashicorp.com/terraform/cloud-docs/agents), [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents)

### ✔ How the Agent Creates VMware VMs or Networking (Step‑by‑Step)
1. **You write Terraform code** using the **vSphere provider** (`hashicorp/vsphere`).  
2. **HCP Terraform** orchestrates a run and assigns it to the **Agent Pool** you configured.  
3. The **Agent** (inside your network) **polls** HCP Terraform and **receives the job** (pull model).  
4. The **Agent executes** `terraform init/plan/apply` **locally**: it downloads the vSphere provider and uses your workspace variables/secrets.  
5. The **vSphere provider** calls **vCenter/ESXi APIs** (e.g., `https://vcenter.local/sdk`) to **create VMs, portgroups, networks, datastores, NICs**, etc.  
6. **Results** (logs, state updates) are sent back to HCP Terraform **over outbound HTTPS (443)**.  
Sources: [Agents (pull‑based execution)](https://docs.devnetexperttraining.com/static-docs/Terraform/docs/cloud/agents/index.html), [Install & run agents](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents)

### Minimal vSphere Example
```hcl
provider "vsphere" {
  user                 = var.vcenter_user
  password             = var.vcenter_password
  vsphere_server       = var.vcenter_server
  allow_unverified_ssl = true
}

resource "vsphere_virtual_machine" "web01" {
  name   = "web01"
  num_cpus = 2
  memory   = 4096
  resource_pool_id = data.vsphere_compute_cluster.cluster.resource_pool_id
  datastore_id     = data.vsphere_datastore.ds.id

  network_interface {
    network_id = data.vsphere_network.net.id
  }

  disk {
    label = "disk0"
    size  = 40
  }
}
```
This works because the **provider runs on the agent host**, which can reach **private vCenter**.  
Sources: [Agents overview](https://developer.hashicorp.com/terraform/cloud-docs/agents), [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents)

### Visual Flow Diagram
```
               +-------------------------------+
               |      HCP Terraform (SaaS)     |
               |  Plan/Apply Orchestration     |
               +-------------------------------+
                      ^                 |
         Outbound Poll|                 | Run Results, State, Logs
                      |                 v
              +--------------------------------+
              |   HCP Terraform Agent          |
              |   (inside your network)        |
              | - Executes Terraform locally   |
              | - Loads providers (vsphere,    |
              |   openstack, nutanix, etc.)    |
              +--------------------------------+
                            |
                            | Private Network (no inbound ports)
                            v
                +-------------------------------+
                | vCenter / ESXi / NSX / APIs   |
                +-------------------------------+
```
Agents use a **pull/outbound‑only model (HTTPS/443)**; no inbound firewall holes are required.  
Sources: [Networking & pull model](https://docs.devnetexperttraining.com/static-docs/Terraform/docs/cloud/agents/index.html), [Agents overview](https://developer.hashicorp.com/terraform/cloud-docs/agents)

---

## 2) Architecture & Execution Model
- **Pull‑based:** Agents **poll** HCP Terraform for work; you do **not** open inbound ports.  
- **Local execution:** Each run uses a **temporary directory** with a clean environment; write **stateless/idempotent** code.  
- **Concurrency:** One run **per agent process**; you can run multiple processes on the same host (license limit permitting).  
- **Resilience:** Run under a **process supervisor** (systemd/K8s/Nomad) to auto‑restart.  
- **Auto‑updates:** Default **minor** auto‑update; tune via `TFC_AGENT_AUTO_UPDATE` (`minor`, `patch`, `disabled`).  
Sources: [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents), [Networking overview](https://docs.devnetexperttraining.com/static-docs/Terraform/docs/cloud/agents/index.html)

---

## 3) Requirements (OS/CPU, Terraform, Hardware, Networking)
- **OS/Arch:** Linux **x86_64** and **ARM64** supported; pool arch is set by the **first agent**; keep pools homogeneous.  
- **Terraform versions:** x86_64: **0.12+**; ARM64: **0.13.5+** (workspaces <0.12 cannot use agents).  
- **Hardware (reference):** **≥4 GB** free disk; **≥2 GB** RAM (add **~250 MB** if request forwarding).  
- **Networking:** Ensure outbound **HTTPS/443** to HCP Terraform and access to provider endpoints (e.g., releases/providers APIs).  
Sources: [Requirements](https://developer.hashicorp.com/terraform/cloud-docs/agents/requirements), [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents)

---

## 4) Install & Run Agents
### A) Docker (Official Image)
```bash
export TFC_AGENT_TOKEN="<agent_pool_token>"   # required
export TFC_AGENT_NAME="east-dc-1"             # optional label

docker run --rm \
  -e TFC_AGENT_TOKEN \
  -e TFC_AGENT_NAME \
  hashicorp/tfc-agent:latest
```
Sources: [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents), [Docker Hub](https://hub.docker.com/r/hashicorp/tfc-agent)

### B) Linux Binary
Provision a Linux host, download the agent, set `TFC_AGENT_TOKEN` (and optionally `TFC_AGENT_NAME`), and run the binary under a supervisor (**systemd**).  
Sources: [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents)

---

## 5) Manage Agent Pools
- **Concept:** Pools group agents; workspaces/Stacks configured for agents consume any **available agent** in the pool.  
- **Permissions:** Manage at org level; **Admin** on workspace/Stack to switch execution mode.  
- **Create & Scope:** Settings → **Agents** → create pool + **Create Token**; then **scope** to projects/workspaces/Stacks (unassign from active ones before removing).  
- **IaC:** Use **TFE provider** (`tfe_agent_pool`) or **HCP Terraform Operator for Kubernetes** to declaratively manage pools/tokens and scale agents.  
Sources: [Agent pools](https://developer.hashicorp.com/terraform/cloud-docs/agents/agent-pools), [TFE provider](https://registry.terraform.io/providers/hashicorp/tfe/latest/docs/resources/agent_pool), [Operator docs](https://github.com/hashicorp/hcp-terraform-operator/blob/main/docs/agentpool.md)

---

## 6) Configure Workspaces/Stacks to Use Agents
- In workspace/Stack settings, set **Execution Mode → Agents** and choose the **agent pool**. HCP Terraform dispatches to the **first available** agent in the pool.  
Sources: [Agent pools](https://developer.hashicorp.com/terraform/cloud-docs/agents/agent-pools), [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents)

---

## 7) Operational Considerations & Best Practices
- **Stateless/idempotent:** avoid absolute paths and machine state.  
- **Single‑execution mode:** use if you want strictly one workload per agent instance.  
- **Homogeneity:** same OS/arch/hardware across pool members to avoid performance variance.  
- **Auto‑update policy:** Decide between `minor` (default), `patch`, or `disabled`.  
Sources: [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents), [Requirements](https://developer.hashicorp.com/terraform/cloud-docs/agents/requirements)

---

## 8) Request Forwarding (Overview)
Enabling request forwarding lets the agent relay specific HTTP requests for the run to private endpoints; plan for **extra memory (~250 MB)**.  
Sources: [Requirements](https://developer.hashicorp.com/terraform/cloud-docs/agents/requirements)

---

## 9) Logging, Metrics & Tracing (Pointers)
Agents expose operational telemetry (logging/metrics/tracing). Integrate with your logging stack; see related docs pages under the Agents section.  
Sources: [Agents docs index](https://developer.hashicorp.com/terraform/cloud-docs/agents)

---

## 10) Extended Examples

### A) vSphere — VM from Template with Network
```hcl
provider "vsphere" {
  user           = var.vcenter_user
  password       = var.vcenter_password
  vsphere_server = var.vcenter_server
  allow_unverified_ssl = true
}

data "vsphere_datacenter" "dc" { name = var.vsphere_dc }

data "vsphere_datastore" "ds" { name = var.vsphere_datastore }

data "vsphere_compute_cluster" "cluster" { name = var.vsphere_cluster }

data "vsphere_network" "net" { name = var.vsphere_portgroup }

data "vsphere_virtual_machine" "tpl" { name = var.template_name }

resource "vsphere_virtual_machine" "vm" {
  name             = var.vm_name
  resource_pool_id = data.vsphere_compute_cluster.cluster.resource_pool_id
  datastore_id     = data.vsphere_datastore.ds.id
  num_cpus         = 2
  memory           = 4096
  guest_id         = data.vsphere_virtual_machine.tpl.guest_id

  network_interface {
    network_id = data.vsphere_network.net.id
  }

  disk {
    label = "disk0"
    size  = 40
  }

  clone {
    template_uuid = data.vsphere_virtual_machine.tpl.id
  }
}
```
Run this in a workspace configured to use **Agents**; the agent host can reach vCenter.  
Sources: [Agents overview](https://developer.hashicorp.com/terraform/cloud-docs/agents), [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents)

### B) Nutanix — Minimal (conceptual)
```hcl
provider "nutanix" {
  username = var.ntnx_user
  password = var.ntnx_pass
  endpoint = var.ntnx_endpoint # private Prism
}
# resources: nutanix_virtual_machine, network, image, etc.
```
The agent reaches **Nutanix Prism** inside the network; the provider handles API calls.  
Reference: Nutanix provider docs (conceptual; use official provider docs for full resource blocks).

### C) OpenStack — VM with Cloud‑Init (conceptual)
```hcl
provider "openstack" {
  auth_url    = var.os_auth_url   # private Keystone
  user_name   = var.os_user
  password    = var.os_password
  tenant_name = var.os_project
  region      = var.os_region
}
# resources: openstack_compute_instance_v2, network, volume, etc.
```
Agent reaches **Keystone/Neutron/Nova** endpoints on your private OpenStack control plane.  
Reference: OpenStack provider docs.

---

## 11) Tips & Tricks
1. **Multiple agent processes per host** to increase throughput (subject to license).  
2. **Human‑friendly names** (e.g., `tfc-east-rack-01`) for easier ops/log review.  
3. **Scope pools** only to workspaces/projects/Stacks that require private access.  
4. **Kubernetes Operator** to declaratively manage pools/tokens and scale agents as pods.  
5. **Docker image help:** `docker run hashicorp/tfc-agent:latest help`.  
Sources: [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents), [Agent pools](https://developer.hashicorp.com/terraform/cloud-docs/agents/agent-pools), [Docker Hub](https://hub.docker.com/r/hashicorp/tfc-agent), [Operator docs](https://github.com/hashicorp/hcp-terraform-operator/blob/main/docs/agentpool.md)

---

## 12) Troubleshooting Checklist
- **Agent not registering:** Verify `TFC_AGENT_TOKEN` and outbound **443** to `app.terraform.io`.  
- **Runs queued/no agent:** Agent is running; pool is **scoped** to target workspace/Stack; execution mode set to **Agents**.  
- **Performance variance:** Standardize pool **hardware/OS/arch**; check disk/RAM baselines (**≥4 GB**, **≥2 GB**).  
- **Statefulness:** Remove absolute paths and external machine state assumptions.  
- **Update surprises:** Confirm `TFC_AGENT_AUTO_UPDATE` policy.  
Sources: [Agent pools](https://developer.hashicorp.com/terraform/cloud-docs/agents/agent-pools), [Requirements](https://developer.hashicorp.com/terraform/cloud-docs/agents/requirements), [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents)

---

## 13) MCQs (with answers)
1. **Agents require which network access from your private environment?**  
   a) Inbound TCP/22 from HCP Terraform  
   b) Inbound TCP/443 from HCP Terraform  
   c) **Outbound TCP/443 to HCP Terraform**  
   d) Bi‑directional VPN  
   **Answer:** c.  
   Sources: [Networking overview](https://docs.devnetexperttraining.com/static-docs/Terraform/docs/cloud/agents/index.html), [Agents overview](https://developer.hashicorp.com/terraform/cloud-docs/agents)

2. **A workspace using Terraform 0.11 can use agents?**  
   a) Yes, with ARM64  
   b) **No; agent mode requires Terraform ≥ 0.12 (ARM64 ≥ 0.13.5)**  
   c) Yes, if Docker is used  
   d) Only in Enterprise  
   **Answer:** b.  
   Source: [Requirements](https://developer.hashicorp.com/terraform/cloud-docs/agents/requirements)

3. **Which statement about agent pools is true?**  
   a) They automatically support mixed x86/ARM hosts  
   b) **They group agents; the first agent sets architecture for the pool**  
   c) They expose inbound ports to HCP Terraform  
   d) They can run multiple runs per agent process  
   **Answer:** b.  
   Source: [Requirements](https://developer.hashicorp.com/terraform/cloud-docs/agents/requirements)

4. **Default update behavior for agents is:**  
   a) Disabled  
   b) Patch‑only  
   c) **Minor auto‑update**  
   d) Canary builds  
   **Answer:** c.  
   Source: [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents)

5. **Recommended disk space for an agent host is:**  
   a) 1 GB  
   b) **≥ 4 GB**  
   c) 8 MB  
   d) 512 MB  
   **Answer:** b.  
   Source: [Requirements](https://developer.hashicorp.com/terraform/cloud-docs/agents/requirements)

---

## 14) Quick Start Checklist
- [ ] Create **Agent Pool**; **generate token** and store securely.  
- [ ] Deploy **agent** (Docker/binary) with `TFC_AGENT_TOKEN` (optional `TFC_AGENT_NAME`).  
- [ ] Workspace/Stack: set **Execution Mode → Agents**, select **agent pool**.  
- [ ] Verify outbound **HTTPS/443** connectivity and provider endpoints.  
- [ ] Ensure Terraform version compatibility (≥0.12 x86, ≥0.13.5 ARM).  
- [ ] Hardware sizing (≥4 GB disk, ≥2 GB RAM; +250 MB if request forwarding).  
- [ ] Decide **auto‑update** policy (`minor`/`patch`/`disabled`).  
Sources: [Agent pools](https://developer.hashicorp.com/terraform/cloud-docs/agents/agent-pools), [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents), [Requirements](https://developer.hashicorp.com/terraform/cloud-docs/agents/requirements)

---

## 15) Ready‑to‑Use Snippets

### A) systemd service (Linux binary)
```ini
# /etc/systemd/system/tfc-agent.service
[Unit]
Description=HCP Terraform Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Environment=TFC_AGENT_TOKEN=YOUR_AGENT_POOL_TOKEN
Environment=TFC_AGENT_NAME=agent-linux-01
# Optional: restrict auto-update behavior
# Environment=TFC_AGENT_AUTO_UPDATE=patch

User=terraform
Group=terraform
ExecStart=/usr/local/bin/tfc-agent
Restart=always
RestartSec=5

# Hardening (tune per environment)
NoNewPrivileges=true
ProtectSystem=full
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now tfc-agent
sudo systemctl status tfc-agent
```
Source: [Install & run](https://developer.hashicorp.com/terraform/cloud-docs/agents/agents)

### B) Kubernetes (HCP Terraform Operator) — minimal AgentPool CR
```yaml
apiVersion: app.terraform.io/v1alpha2
kind: AgentPool
metadata:
  name: agent-pool-demo
  namespace: default
spec:
  organization: your-org
  token:
    secretKeyRef:
      name: tfc-operator
      key: token
  name: agent-pool-demo
  agentTokens:
    - name: default-token
  agentDeployment:
    replicas: 2
    spec:
      containers:
        - name: tfc-agent
          image: hashicorp/tfc-agent:latest
          env:
            - name: TFC_AGENT_TOKEN
              valueFrom:
                secretKeyRef:
                  name: agent-pool-demo-default-token
                  key: token
```
Source: [Operator docs](https://github.com/hashicorp/hcp-terraform-operator/blob/main/docs/agentpool.md)

---

## 16) References
- HCP Terraform Agents (overview): https://developer.hashicorp.com/terraform/cloud-docs/agents  
- Install & run agents: https://developer.hashicorp.com/terraform/cloud-docs/agents/agents  
- Agent requirements: https://developer.hashicorp.com/terraform/cloud-docs/agents/requirements  
- Manage agent pools: https://developer.hashicorp.com/terraform/cloud-docs/agents/agent-pools  
- Docker Image: https://hub.docker.com/r/hashicorp/tfc-agent  
- Kubernetes Operator (AgentPool CR): https://github.com/hashicorp/hcp-terraform-operator/blob/main/docs/agentpool.md  
- Networking/pull model (illustrative): https://docs.devnetexperttraining.com/static-docs/Terraform/docs/cloud/agents/index.html

---

*Last updated: 2025‑12‑19*
