# VNet and Subnets

## The Foundation: VNet and Subnets
A Virtual Network (VNet) is your private, isolated network in the Azure cloud. It forms the fundamental building block for your private environment, acting as the cloud version of your on-premises network. 

Unlike traditional on-premises networks where you need a physical router to move traffic between different subnets, **Azure VNets are "flat" by default**. This means all subnets within a single VNet can communicate with each other automatically without any additional configuration or routing devices. This is achieved through built-in **Azure System Routes**.

### Why Azure Doesn't Support an "Internet Gateway" Resource
If you come from AWS, you might be looking for an "Internet Gateway" (IGW) to attach to your VNet. Azure does not have this specific resource.
- **Inbuilt Capabilities of Azure Networking**: Azure handles internet routing implicitly at the SDN level. The default system route `0.0.0.0/0` points directly to the internet. If a VM has a Public IP, the Azure SDN automatically performs the Network Address Translation (NAT) and sends it out. You do not need to build a manual gateway to the internet unless you are forcing traffic through a firewall.

---

## IP Address Space and CIDR Planning
Designing your network space correctly from day one is critical because changing it later is extremely difficult. 

### Rule 1: Start BIG
Allocate a massive address space for the entire company, and then carve it out for different VNets.
**Example:**
- **Company VNet space**: `10.0.0.0/8` (or `/12`)
- **Hub VNet**: `10.0.0.0/16`
- **Finance VNet**: `10.1.0.0/16`
- **HR VNet**: `10.2.0.0/16`
- **Engineering VNet**: `10.3.0.0/16`

### Rule 2: Leave gaps for the future
Do not use sequential blocks without leaving room. 
- `10.4.0.0/16` → reserved for future acquisitions or new departments
- `10.5.0.0/16` → reserved

### System Reserved IP Addresses
When you create a subnet, Azure always reserves 5 IP addresses that you cannot use for your VMs:
1. `x.x.x.0`: Network address.
2. `x.x.x.1`: Reserved by Azure for the default gateway.
3. `x.x.x.2`, `x.x.x.3`: Reserved by Azure to map the Azure DNS IPs to the VNet space.
4. `x.x.x.255`: Network broadcast address.
*Example: In a `/24` subnet, you have 256 total IPs, but only 251 usable IPs.*

*(Self Study Note: IANA - Internet Assigned Numbers Authority is the global organization that oversees global IP address allocation, managing which blocks are public and which are private like 10.x.x.x).*

---

## Creating Subnets and Segmentation

### Subnet Design (Inside each VNet)
Once you have your VNet block (e.g., Engineering at `10.3.0.0/16`), slice it by component tier. In subnets, we try to create segments to group related resources for security and routing boundaries.
- `10.3.1.0/24` → AKS nodes (System Subnet)
- `10.3.2.0/24` → Databases
- `10.3.3.0/24` → App Gateway / Ingress
- `10.3.4.0/24` → Internal services

> [!WARNING] 
> Subnet names like `GatewaySubnet` (for VPNs) and `AzureBastionSubnet` (for Bastion) are mandatory and hardcoded in Azure. Azure’s Resource Provider looks for these exact strings and applies a strict "Lock" on them to prevent you from putting normal VMs inside.

> [!TIP]
> **Professional Tip for AKS**: Always **over-provision**. Azure CNI reserves IP addresses for every pod the moment the node starts. If you create a `/24` subnet (251 usable IPs) and your pods are dense, you will run out of IPs incredibly fast. Aim for at least a `/22` for production AKS subnets.

### Lab Scenario: Public vs Private Subnets
**Setup**: You have 1 VNet with 2 subnets:
- Public Subnet: `10.0.1.0/24` (VM-1)
- Private Subnet: `10.0.2.0/24` (VM-2)

**How to expose them?**
- **App in Public Subnet**: To expose VM-1, you assign it a Public IP address directly (or put a Public Load Balancer in front of it) and open port 80/443 in the NSG.
- **App in Private Subnet**: VM-2 has no Public IP. 
  - *Inbound*: To let users access the app on VM-2, you must deploy an Application Gateway or Load Balancer in the public subnet, which forwards traffic to VM-2's private IP.
  - *Outbound*: To let VM-2 reach the internet (to download updates), you **Create a NAT Gateway and attach it to the Private Subnet**. The NAT Gateway gives the subnet a shared public IP for outbound traffic without exposing the VMs to inbound internet attacks.

---

## The "Hub and Spoke" Model
In enterprise environments, using a single VNet is a bad practice. Instead, we use the **Hub-and-Spoke topology**.

```text
                [ HUB VNet ]
          (Shared Services, Firewall, VPN Gateway)
                /    |    \
               /     |     \
      [Spoke-1] [Spoke-2] [Spoke-3]
      (App A)   (App B)   (Dev/Test)
```

### Why do we use Hub and Spoke?
1. **Isolation**: Each department or application (e.g., Finance, HR, Engineering) gets its own Spoke VNet. 
2. **Security Boundaries**: A compromised Spoke does not inherently compromise other Spokes.
3. **Easier Scaling**: As the organization grows, you simply plug a new Spoke into the Hub.
4. **Cost Efficiency**: You deploy highly expensive shared services (Firewall, DNS servers, ExpressRoute) once in the Hub.

**Traffic**: All traffic from the Spoke is "forced" to the Hub via a UDR to be scrubbed by the Firewall before going to the internet.
