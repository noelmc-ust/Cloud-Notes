# VNet and Subnets

## The Foundation: VNet and Subnets
A Virtual Network (VNet) is your private isolated network in the Azure cloud. Unlike on-premises networks, Azure VNets are "flat" by default—meaning all subnets within a single VNet can talk to each other without a router, thanks to Azure System Routes.

### Setup Strategy:
- **Address Space**: Define a CIDR block (e.g., `10.0.0.0/16`).
- **Subnetting**: Carve this into functional slices. 

In an enterprise AKS (Azure Kubernetes Service) setup, you typically need:
- **System Subnet**: For the AKS nodes.
- **User/App Subnet**: For the pods (if using Azure CNI).
- **GatewaySubnet**: Specifically for VPN Gateways.
- **AzureBastionSubnet**: A dedicated, named subnet for secure management.

## IP Address Space and CIDR Planning
### Rule 1: Start BIG
Example:
- Company VNet space: `10.0.0.0/8` (or `/12`)
Then divide:
- Hub: `10.0.0.0/16`
- Finance: `10.1.0.0/16`
- HR: `10.2.0.0/16`
- Engineering: `10.3.0.0/16`

### Rule 2: Leave gaps for future
- `10.4.0.0/16` → reserved
- `10.5.0.0/16` → reserved

### Step 3: Subnet Design (Inside each VNet)
Example: Engineering VNet (`10.3.0.0/16`)
- `10.3.1.0/24` → AKS nodes
- `10.3.2.0/24` → Databases
- `10.3.3.0/24` → App Gateway / Ingress
- `10.3.4.0/24` → Internal services

> [!TIP]
> **Professional Tip**: When setting up subnets for AKS, always over-provision. Azure CNI reserves IPs for every pod the moment the node starts. Aim for at least a `/22` for production AKS subnets.

## The "Hub and Spoke" Model
In enterprise environments, we rarely use one VNet. We use a Hub-and-Spoke topology.

```text
                [ HUB VNet ]
          (shared services, firewall)
                /    |    \
               /     |     \
      [Spoke-1] [Spoke-2] [Spoke-3]
      (App A)   (App B)   (Dev/Test)
```

- **Hub**: Contains the Firewall, VPN Gateway (to your office), and Bastion.
- **Spoke**: Contains your applications, like an AKS Cluster. Each department/app gets its own VNet for isolation and security boundaries.

**Traffic**: All traffic from the Spoke can be "forced" to the Hub via a UDR to be scrubbed by the Firewall before going to the internet.
