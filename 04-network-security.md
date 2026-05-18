# Network Security & Gateways

## Network Security Groups (NSG)
A distributed firewall. You apply rules at the Subnet or NIC level to allow/deny traffic based on IP, Port, and Protocol. VFP executes these rules on the physical host.

## Gateways: Application Gateway vs Load Balancer

| Feature | Load Balancer (LB) | Application Gateway (AppGW) |
|---------|--------------------|-----------------------------|
| **Layer** | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| **Intelligence** | Fast, simple packet forwarding based on Hash | Smart routing (URL path, SSL termination, WAF) |
| **Subnet Requirement** | None | Must have its own dedicated subnet |
| **Underlying Infra** | Software-defined networking | Managed Virtual Appliances (VMs) |

- **AppGW Dedicated Subnet**: Azure spins up dedicated virtual machines for AppGW. The dedicated subnet ensures no other resources interfere with management traffic.

## Azure Bastion
Azure Bastion provides secure RDP and SSH access directly through the Azure Portal over SSL (Port 443).

- **How it Works**: Your browser connects via HTTPS (443) to Bastion's Public IP. The Bastion Service then establishes a private RDP/SSH connection to the target VM over the internal network.
- **Subnet Requirement**: Requires a dedicated `AzureBastionSubnet`.
- **Why it matters**: Your VMs (like AKS nodes) remain 100% private with no public IPs, eliminating brute-force internet attacks.

## VPN Gateway
A Virtual Network Gateway sends encrypted traffic between an Azure VNet and an on-premises location over the public internet.

- **Use Cases**: Site-to-Site VPN (Hybrid Cloud), Point-to-Site VPN (Secure Developer Access).
- **Subnet Requirement**: Requires a dedicated `GatewaySubnet`.
- **Underlying Infra**: Azure provisions dedicated VMs in high availability (Active-Active/Standby).
- **Security (Diffie-Hellman & IPSec)**: 
  - *IKE Phase 1*: Authenticates using a Pre-Shared Key (PSK) and uses Diffie-Hellman to generate a master key.
  - *IKE Phase 2*: Generates Data Encryption Keys (using AES) to actually encrypt your traffic. Keys are periodically regenerated (Rekeying) for Perfect Forward Secrecy (PFS).

## Private Link / Endpoints & Service Endpoints
- **Private Link / Endpoints**: Allows access to Azure PaaS (SQL, Key Vault) over a private IP in your VNet. Data never leaves your VNet.
- **Service Endpoints**: Secures PaaS to your subnet, but the traffic still uses the PaaS's public IP address space (though routed via the Azure backbone).
