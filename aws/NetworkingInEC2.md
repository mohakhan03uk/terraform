# Networking in Amazon EC2

## Overview
1. An EC2 instance is configured with a **primary network interface (ENI)**, which is a logical virtual network card.
2. The instance receives a **primary private IPv4 address** from the subnet CIDR, and it is assigned to the **primary network interface**.
3. You can attach a **public IPv4 address** from Amazon's public IP pool.
   - This public IP remains associated **only until the instance is stopped, hibernated, or terminated**.
4. If you need a **persistent public IP address**, you must allocate an **Elastic IP (EIP)** and associate it with:
   - an EC2 instance **or**
   - a **Network Interface (ENI)**
5. You can move an **Elastic IP** from one instance to another as needed (useful for failover).
6. You can also **bring your own public IP address (BYOIP)** and use it as an Elastic IP in AWS.

---

## IP Addressing

1. Amazon EC2 and VPC support:
   - IPv4 only
   - IPv6 only
   - Dual-stack (IPv4 + IPv6)
2. When you launch an EC2 instance, you must specify:
   - a **VPC**
   - a **subnet**
   The instance receives its IP address from the **CIDR block of the subnet**.
3. You can optionally configure instances with:
   - Public IPv4 addresses
   - IPv6 addresses  
   If EC2 instances in **different VPCs communicate using public IPs**, the traffic **stays on the AWS global private network** and does **not traverse the public internet**.
4. Default public IP behavior:
   > - Instances launched in the **default VPC** receive a public IPv4 address by default.  
   > - Instances launched in **non-default subnets** do **not** receive a public IP unless explicitly enabled.
   >
   > - Auto-assigned public IPs can change when you **stop / start / hibernate / terminate** an instance.
   > - For a **persistent public IPv4**, use an **Elastic IP (EIP)**.

---
## VPC Fundamentals 
1. A **VPC (Virtual Private Cloud)** is a logically isolated virtual network in AWS.
2. A VPC contains:
   - Subnets
   - Route tables
   - Internet Gateway (IGW)
   - NAT Gateway / NAT Instance
   - Network ACLs
3. Subnets define:
   - IP address ranges
   - Availability Zone boundaries
4. Routing is controlled by **route tables**, not by ENIs.
---
## Subnets

1. A **subnet** is a **range of IP addresses** within a **VPC CIDR block**.
2. Each subnet:
   - Belongs to **exactly one Availability Zone (AZ)**
   - Cannot span multiple AZs
3. Subnets are used to:
   - Organize resources
   - Control routing
   - Isolate workloads

---

### Subnet CIDR

1. A subnet CIDR must be a **subset of the VPC CIDR**.
2. AWS reserves **5 IP addresses** in every subnet:
   - Network address
   - VPC router
   - DNS
   - Reserved for future use
   - Broadcast (not used, but reserved)
3. Example:
   - Subnet CIDR: `10.0.0.0/24`
   - Total IPs: 256
   - Usable IPs: 251

---

### Public vs Private Subnets

#### Public Subnet
A subnet is considered **public** if:
- Its route table has a route to an **Internet Gateway (IGW)**

Common use cases:
- Bastion hosts
- Internet-facing load balancers
- NAT Gateways

#### Private Subnet
A subnet is considered **private** if:
- It does **not** have a route to an Internet Gateway

Common use cases:
- Application servers
- Databases
- Internal services

> There is **no explicit “public/private” flag** — it is defined purely by routing.

---

### Subnet Route Tables

1. Each subnet must be associated with **one route table**.
2. A route table controls how traffic is routed:
   - Local VPC routing is automatic
   - Internet access requires an IGW route
   - Outbound-only internet access requires a NAT Gateway/Instance
3. Multiple subnets can share the same route table.

---

### Auto-assign Public IP Setting

1. Each subnet has a setting:
   - **Auto-assign public IPv4 address**
2. If enabled:
   - EC2 instances launched into the subnet receive a public IPv4 by default
3. Default VPC subnets:
   - Auto-assign public IP = **enabled**
4. Custom subnets:
   - Auto-assign public IP = **disabled by default**

---

### Subnet and ENI Relationship

1. A **network interface (ENI)** is created in **one subnet**.
2. Since a subnet belongs to one AZ:
   - ENIs are **AZ-scoped**
   - ENIs cannot move across AZs
3. Elastic IPs are **region-scoped** and can be reassociated across AZs.

> ENI → Subnet → Availability Zone  
> EIP → Region

---

### IPv6 in Subnets

1. IPv6 addresses are assigned from an **IPv6 CIDR block** associated with the VPC.
2. Each subnet receives a **/64 IPv6 CIDR**.
3. IPv6 addresses:
   - Are globally unique
   - Are public by default
   - Do not require NAT

---

### Subnet Use Cases

- Network segmentation
- Security isolation
- High availability across AZs
- Public/private workload separation
- Kubernetes node and pod networking (EKS)

---

## Multiple IP Addresses

1. You can assign **multiple private IPv4 and IPv6 addresses** to an EC2 instance.
2. These IPs are assigned to **network interfaces (ENIs)**.
3. The maximum number of:
   - Network interfaces
   - IPv4 addresses per ENI
   - IPv6 addresses per ENI  
   depends on the **instance type**.
4. Common use cases:
   - Hosting multiple services on one instance
   - Network appliances
   - EKS pod networking
   - Failover without DNS changes
5. Reference:
   - https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-instance-addressing.html#multiple-ip-addresses

---

## Elastic Network Interfaces (ENI)

1. An **Elastic Network Interface (ENI)** is a **logical networking component** in a VPC that represents a **virtual network card**.
2. An ENI is created in **one specific subnet**, and each subnet belongs to **exactly one Availability Zone (AZ)**.
   - Therefore, an ENI **cannot move across AZs**.
   - **Elastic IPs (EIP)** are **region-scoped**, not AZ-scoped.

   > **Relationship:**  
   > ENI → Subnet → Availability Zone (fixed)  
   > EIP → Region (movable)

3. ENIs can be:
   - Attached
   - Detached
   - Reattached to another instance (within the same AZ)
4. You can enable **VPC Flow Logs** on an ENI to capture:
   - Source/destination IP
   - Ports
   - Protocol
   - Accept/Reject status  
   Logs are published to **CloudWatch Logs** or **S3**.

---

### Network Interface Attributes

A network interface can include:

- A **primary private IPv4 address** from the subnet CIDR
- A **primary IPv6 address** (if IPv6 is enabled)
- **Secondary private IPv4 addresses**
- **Secondary IPv6 addresses**
- **One Elastic IP (IPv4)** per private IPv4 address
- One auto-assigned **public IPv4 address**
- **Security groups**
- A **MAC address**
- A **source/destination check flag**
- A **description**

---
## Elastic IP Addresses (EIP)

1. An **Elastic IP (EIP)** is a **static, public IPv4 address** that is **allocated to your AWS account** and is **region-scoped**.
2. Unlike auto-assigned public IPv4 addresses, an EIP:
   - Does **not change** when you stop/start an EC2 instance
   - Remains allocated until you explicitly release it
3. An Elastic IP can be associated with:
   - An EC2 instance
   - A specific **Elastic Network Interface (ENI)**
4. An Elastic IP is always mapped to a **private IPv4 address** on an ENI.
   > EIP → ENI → Private IPv4
5. You can **remap an Elastic IP** from one instance to another instance (or ENI) in the **same Region**.
   - This enables fast failover without DNS changes.
6. Elastic IPs are commonly used for:
   - Bastion hosts
   - Public APIs
   - NAT instances
   - High-availability architectures
7. AWS charges for:
   - Unused Elastic IPs
   - Elastic IPs associated with stopped instances
8. You can bring your own public IPv4 address range and use it as Elastic IPs (**BYOIP**).

---

## Elastic IP vs Public IP (Quick Comparison)

| Feature | Public IPv4 | Elastic IP |
|------|-------------|------------|
| Allocation | Automatic | Manual |
| Persistence | Temporary | Persistent |
| Changes on stop/start | Yes | No |
| Region scoped | No | Yes |
| Movable between instances | No | Yes |
| Cost | Free (while running) | Charged if unused |

---

## Elastic IP + ENI (Important Relationship)

- Elastic IPs are **not attached directly to EC2 instances**.
- An EIP is associated with a **private IPv4 address on an ENI**.
- If the ENI moves to another instance, the EIP **moves with it**.
- This makes **ENI-based failover** more flexible than instance-based EIP attachment.

---

## Hostnames and DNS 
1. EC2 instances can be assigned:
   - Private DNS names
   - Public DNS names (if public IP exists)
2. DNS behavior depends on VPC settings:
   - `enableDnsSupport`
   - `enableDnsHostnames`
3. Hostnames are automatically derived from the instance’s IP address.

---

## Network Bandwidth 
1. Network bandwidth depends on:
   - EC2 instance type
   - Network card configuration
2. Larger instance types provide:
   - Higher throughput
   - Higher packets per second (PPS)
3. Some instances support **multiple network cards** for higher aggregate bandwidth.

---

## Enhanced Networking 
1. Enhanced networking improves:
   - Throughput
   - Latency
   - Packet processing
2. Uses **ENA (Elastic Network Adapter)** or **SR-IOV**.
3. Enabled by default on most modern instance types.
4. Required for high-performance workloads such as:
   - Big data
   - High-frequency trading
   - Distributed systems

---

## Elastic Fabric Adapter (EFA)
1. EFA is a specialized network interface for:
   - High Performance Computing (HPC)
   - Machine learning workloads
2. Provides:
   - Very low latency
   - High bandwidth
   - OS-bypass capabilities
3. Used with MPI-based applications.
4. Supported only on specific instance types.

---

## Placement Groups
1. Placement groups influence how EC2 instances are placed on underlying hardware.
2. Types:
   - **Cluster** – low latency, high throughput (single AZ)
   - **Spread** – instances placed on distinct hardware (high availability)
   - **Partition** – large distributed workloads (e.g., HDFS, Kafka)
3. Placement groups affect **network latency and fault tolerance**.

---

## Network MTU 
1. MTU (Maximum Transmission Unit) defines the largest packet size.
2. Larger MTU can improve performance for data-intensive workloads.
3. Must be supported by:
   - Instance type
   - OS
   - Network path

---

## EC2 Topology
1. EC2 topology represents how instances are physically placed in AWS infrastructure.
2. Helps understand:
   - Network distance
   - Failure domains
   - Capacity reservations
3. Relevant for:
   - Low-latency architectures
   - HPC workloads

---

## Important Notes (Exam & Real-World)

- Every EC2 instance has **at least one ENI** (primary ENI).
- The **primary ENI cannot be detached** from the instance.
- **Source/Destination Check** must be disabled for:
  - NAT instances
  - Firewalls
  - Routers
- Many AWS services create **service-managed ENIs** automatically:
  - EKS
  - ALB / NLB
  - RDS
  - Lambda (VPC mode)
- Always prefer **EIP + ENI** for:
  - Stateful services
  - Fast failover designs
- Do **not** use EIPs unnecessarily due to cost.
- For internet-facing load-balanced apps, prefer:
  - **ALB / NLB** instead of direct EIPs
- Subnets are **AZ-specific**
- Public/private depends on **route table**, not subnet type
- ENIs inherit subnet properties
- IPv6-enabled subnets simplify internet connectivity
---

## Quick Summary

- **ENI** = Virtual NIC in a VPC
- **Private IP** = From subnet CIDR
- **Public IP** = Temporary unless Elastic IP is used
- **Elastic IP** = Persistent, movable, region-scoped
- **Multiple ENIs / IPs** = Advanced networking & HA
- **VPC Flow Logs** = Traffic visibility at ENI level
- Subnets are **AZ-specific**
- Public/private = **route table**, not subnet type
- EIP ≠ Public IP
- ENI is the foundation of EC2 networking
- Use **ALB/NLB** instead of EIP for scalable public apps
