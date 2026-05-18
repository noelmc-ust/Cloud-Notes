# VNet and Subnets

## The Foundation: VNet and Subnets
A Virtual Network (VNet) is your private, isolated network in the Azure cloud. It forms the fundamental building block for your private environment. 
Unlike traditional on-premises networks where you need a router to move traffic between different subnets, **Azure VNets are "flat" by default**. This means all subnets within a single VNet can communicate with each other automatically without any additional configuration or routing devices, thanks to **Azure System Routes**.

### Setup Strategy & Functional Slicing
When creating a VNet, you first define a large **Address Space** using a CIDR block (e.g., `10.0.0.0/16`). After that, you carve this large space into smaller, functional slices called **Subnets**.

In an enterprise architecture—especially when deploying microservices with Azure Kubernetes Service (AKS)—you typically divide the VNet into specific subnets:
- **System Subnet**: Dedicated for the AKS nodes (the virtual machines that run your workloads).
- **User/App Subnet**: Dedicated for the pods (if you are using Azure CNI, every pod requires its own IP address from the VNet).
- **GatewaySubnet**: A specifically named subnet reserved strictly for Virtual Network Gateways (VPN or ExpressRoute).
- **AzureBastionSubnet**: A specifically named subnet reserved for the Azure Bastion service for secure management access.

> [!WARNING] 
> Subnet names like `GatewaySubnet` and `AzureBastionSubnet` are mandatory and hardcoded in Azure. Azure’s Resource Provider looks for these exact strings and applies a strict "Lock" on them. It ensures the underlying Virtual Filtering Platform (VFP) rules for that subnet are specifically tuned for those services, and it prevents you from accidentally putting a Web Server or Database in that subnet.

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

### Rule 3: Subnet Design (Inside each VNet)
Once you have your VNet block (e.g., Engineering at `10.3.0.0/16`), slice it by component tier:
- `10.3.1.0/24` → AKS nodes
- `10.3.2.0/24` → Databases
- `10.3.3.0/24` → App Gateway / Ingress
- `10.3.4.0/24` → Internal services

> [!TIP]
> **Professional Tip for AKS**: When setting up subnets for AKS, always **over-provision**. Azure CNI (Advanced Networking) reserves IP addresses for every pod the moment the node starts. If you create a `/24` subnet (251 usable IPs) and your pods are dense, you will run out of IPs incredibly fast, causing pod creation to fail. Aim for at least a `/22` for production AKS subnets.

## The "Hub and Spoke" Model
In enterprise environments, using a single VNet is a bad practice. Instead, we use the **Hub-and-Spoke topology**.

```text
                [ HUB VNet ]
          (Shared Services, Firewall, VPN, Bastion)
                /    |    \
               /     |     \
      [Spoke-1] [Spoke-2] [Spoke-3]
      (App A)   (App B)   (Dev/Test)
```

### Why do we use Hub and Spoke?
1. **Isolation**: Each department or application (e.g., Finance, HR, Engineering) gets its own Spoke VNet. 
2. **Security Boundaries**: A compromised Spoke does not inherently compromise other Spokes.
3. **Easier Scaling**: As the organization grows, you simply plug a new Spoke into the Hub.
4. **Cost Efficiency**: Instead of deploying a highly expensive Azure Firewall or VPN Gateway in every single VNet, you deploy them once in the Hub. All Spokes share these centralized resources.

### Traffic Flow in Hub and Spoke
By default, Spokes cannot talk to each other. They must route their traffic through the Hub. All outbound traffic to the internet from the Spokes is "forced" to the Hub via a **User Defined Route (UDR)** to be scrubbed by the Firewall before going to the internet.
