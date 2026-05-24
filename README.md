# Secure Azure Hub-and-Spoke Topology with Centralized Firewall Transit Routing

## 📌 Architectural Overview
This repository documents the structural engineering and validation of a secure, zero-trust **Hub-and-Spoke Virtual Network (VNet) topology** deployed natively within Microsoft Azure. The architecture isolates discrete workload environments—**Production, Staging, and Development**—while enforcing mandatory, stateful packet inspection for all cross-spoke (East-West) traffic via a centralized security appliance.

### Design Objectives
* **Zero Trust Boundary Isolation:** Complete segregation of spoke environments at the network layer. No public IP addresses are assigned to any workload Virtual Machines (VMs).
* **Centralized Egress & Transit Control:** Elimination of direct spoke-to-spoke VNet peering to prevent lateral movement. All cross-environment traffic is forced through an enterprise-grade firewalled transit path.
* **Secure Operational Management:** Decoupled administrative access utilizing a centralized Azure Bastion host provisioned exclusively inside the Hub management boundary.

---

## 🛠️ Infrastructure Specifications

### 1. Networking Matrix
| Virtual Network | Address Space (CIDR) | Dedicated Subnets | Component Role |
| :--- | :--- | :--- | :--- |
| **Hub-VNet** | `10.0.0.0/16` | `AzureFirewallSubnet` (`10.0.1.0/24`) <br> `AzureBastionSubnet` (`10.0.2.0/24`) | Centralized Transit, Management & Security |
| **Development-Spoke-VNet** | `10.1.0.0/16` | `Workload-Subnet` (`10.1.1.0/24`) | Isolated Non-Prod Workloads |
| **Staging-Spoke-VNet** | `10.2.0.0/16` | `Workload-Subnet` (`10.2.1.0/24`) | Pre-Production Staging Phase |
| **Production-Spoke-VNet** | `10.3.0.0/16` | `Workload-Subnet` (`10.3.1.0/24`) | Mission-Critical Production Assets |

### 2. Core Security & Routing Components
* **Central Appliance:** Azure Firewall (Standard/Premium Policy-Driven) acting as the Layer 3/4 transit router.
* **Ingress Management:** Azure Bastion deployed inside the Hub to safely jump-box into spoke workloads via private RFC 1918 space.
* **Routing Control:** User-Defined Routes (UDRs) bound to all spoke subnets overriding default Azure System Routing tables.

---

## 🛑 Root Cause Analysis (RCA): The Asymmetric Routing Loop

### The Initial Engineering Failure
During the baseline deployment phase, User-Defined Route (UDR) tables were attached to all Spoke subnets with a generic catch-all default route:
* **Destination:** `0.0.0.0/0`
* **Next Hop Type:** `Virtual Appliance`
* **Next Hop IP:** `10.0.1.4` (Azure Firewall VIP)

Despite both Outbound and Inbound Network Security Groups (NSGs) being explicitly set to `Allow`, and Azure Firewall network rules configured with an unrestrictive Any-to-Any (`*` to `*`) rule collection, **cross-spoke SSH/RDP connectivity requests timed out completely.**

### Triage & Diagnostic Strategy
To systematically isolate the failure layer, a clean-room state was established:
1. Custom route tables were temporarily detached from the subnets to isolate the cloud fabric provider's base routing layer.
2. Synthetic packet probing was initiated via **Azure Network Watcher Connection Troubleshoot** from `Development-VM` to `Production-VM` over Port 22.

### Diagnostic Findings
The hop-by-hop packet inspection revealed a structural routing conflict inherent to Azure's software-defined networking (SDN) fabric:

1. **Ingress Invalidation:** The egress packet from the source VM successfully checked the UDR, bypassed the local NSG, and matched the firewall policy rule. The firewall permitted transit and handed the packet off to the destination spoke subnet.
2. **The Return Path Trap:** The packet arrived at the destination VM retaining its original client source IP (e.g., `10.1.1.4`). When the target operating system generated the return handshake packet destined back to the source IP, the platform evaluated local routing options.
3. **Route Longest-Prefix Match (LPM) Conflict:** Azure automatically injects an implicit, immutable system route for all direct VNet peerings. 
   * The destination subnet evaluated two valid paths to return the packet:
     * System Peering Route: `10.1.0.0/16` ➔ `VNet Peering` (Direct Fabric)
     * Custom UDR: `0.0.0.0/0` ➔ `Virtual Appliance` (`10.0.1.4`)
   * Because a more specific network prefix (`/16`) *always* takes precedence over a generic default route (`/0`), the destination VM bypassed the firewall entirely, routing the reply back over the direct peering fabric.
4. **Stateful Stateful Inspection Drop:** The source VM received an un-encapsulated packet directly from the destination VM, bypassing the firewall state engine it initiated the call with. Detecting a tracking state mismatch, the client OS dropped the packet, causing an asymmetric routing loop.

---

##  Engineering Resolution & Implementation

Symmetric traffic flow was achieved by remediating both the peering transport layer and the firewall policy translation mechanics.

### Phase 1: Transit Peering Authorization
By default, VNet peering drops packets passing through intermediate networks. The Hub-and-Spoke links were reconfigured to explicitly allow transit:
* **Hub-VNet Edge Settings:** Activated `Allow 'Hub-VNet' to receive forwarded traffic from 'Spoke-VNets'`.
* **Spoke-VNet Edge Settings:** Activated `Allow 'Spoke-VNet' to receive forwarded traffic from 'Hub-VNet'`.

### Phase 2: Enforcing Flow Symmetry via Source NAT (SNAT)
To force the destination VMs to safely return traffic through the central firewall without managing highly complex, shifting routing matrices, the **Azure Firewall Policy** was updated to override default private routing assumptions:
* **Configuration:** Adjusted **Private IP Ranges (SNAT)** settings from *IANA defined private IP addresses* to **Always SNAT**.

#### Optimized Traffic Flow Post-Remediation:
1. `Development-VM` (`10.1.1.4`) transmits an outbound packet to `Production-VM` (`10.3.1.4`).
2. The Spoke UDR catches the packet and forwards it to the Azure Firewall (`10.0.1.4`).
3. The Firewall executes policy validation, strips out the source IP (`10.1.1.4`), translates it to its own internal VIP (`10.0.1.4`), and forwards the frame to the destination Spoke.
4. `Production-VM` receives the packet seeing the *Firewall* as the source handler.
5. When `Production-VM` replies, it targets `10.0.1.4`. This matches the local Hub network directly, bypassing the direct-peering specific route. 
6. The Firewall receives the return frame, reverses the NAT table state mapping, and safely routes the traffic back to the originating development subnet—establishing 100% network symmetry.

---

## 🚀 Advanced Production Alternatives (Enterprise Scale)

While modifying the appliance posture to **Always SNAT** is an effective architectural pattern for rapid deployment and development labs, real-world enterprise deployments frequently require native source-IP retention for centralized security information and event management (SIEM) ingestion, compliance logging, and identity-aware application architectures.

To implement this model at an enterprise scale without source translation, the design should be evolved to use **Deterministic Longest-Prefix Routing (No-SNAT Mode)**:

Instead of assigning a generic `0.0.0.0/0` default block, engineers deploy highly explicit Route Tables containing the exact classless inter-domain routing (CIDR) configurations of the opposing spoke environments:
* **Route Entry 1:** `10.1.0.0/16` (Dev Spoke) ➔ Next Hop: `Virtual Appliance (10.0.1.4)`
* **Route Entry 2:** `10.2.0.0/16` (Staging Spoke) ➔ Next Hop: `Virtual Appliance (10.0.1.4)`
* **Route Entry 3:** `10.3.0.0/16` (Prod Spoke) ➔ Next Hop: `Virtual Appliance (10.0.1.4)`

**Why this is the industry standard for large infrastructures:**
Because these custom entries exactly match the `/16` network boundaries of the peering networks, they match the evaluation priority weight of Azure's default peering paths. Since custom UDRs possess a higher placement priority over native system routes when the mask length matches, the destination VM is forced to return traffic through the firewall naturally, ensuring complete security visibility without hiding real infrastructure IPs behind source NAT.

---

## 📂 Project Assets & Verification Directory
All design proofs, platform validation outputs, and topology mappings are available for reference within the repository directories:
* `/architecture-diagrams/` — High-fidelity conceptual end-to-end traffic flow schematics.
* `/deployment-proof/` — Verified Network Watcher runtime states, routing tables, and policy configuration captures from the Azure management plane.