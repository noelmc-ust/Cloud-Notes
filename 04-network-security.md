# Network Security & Gateways

## Network Security Groups (NSG)
An NSG is a distributed firewall. You apply rules to the Subnet or the Network Interface (NIC) to allow/deny traffic based on IP, Port, and Protocol. 
- **Under the Hood**: NSG rules are pushed by the Network Controller to the Host Agent, which compiles them into the VFP Flow Table on the physical host. When you "deny" Port 80, the VFP drops those packets before they ever reach the VM.

---

## Azure Bastion: The Secure "Jumpbox"
In legacy setups, managing private VMs meant giving a VM a public IP and opening port 22 (SSH) or 3389 (RDP) to the internet. This is a massive security risk due to brute-force attacks.

**Azure Bastion** is a managed PaaS that provides secure RDP/SSH access directly through the Azure Portal over SSL (Port 443).

### How it works:
1. **The Subnet**: You must create a dedicated subnet named exactly `AzureBastionSubnet`.
2. **The Infrastructure**: When you deploy Bastion, Azure spins up a Scale Set of hardened, minimal Linux VMs in that subnet. Microsoft manages their patching and OS updates; you never see them in your VM list.
3. **The Connection Flow**:
   - You click "Connect" in the portal. Your browser establishes an HTTPS (443) connection to the Bastion's Public IP.
   - The Bastion service verifies your Entra ID (MFA) token, and translates the HTTPS traffic into a private RDP/SSH session.
   - The VFP on the Bastion's physical host creates a tunnel to the target VM's private IP.
4. **Why this is Secure**: The only port open to the internet on Bastion is 443. Your actual worker VMs (AKS nodes, DBs) remain 100% private. The VFP completely blocks any direct internet access to your worker VMs, only allowing SSH traffic if it originates from the specific internal IP of the Bastion scale set.

---

## VPN Gateway & IPSec
A VPN Gateway is used to send encrypted traffic between an Azure VNet and an on-premises location (Site-to-Site) or a developer laptop (Point-to-Site) over the public Internet.

- **Implementation**: Requires a dedicated `GatewaySubnet`. Setting this up takes up to 45 minutes because Azure is physically provisioning a highly available (Active-Active or Active-Standby) cluster of VMs and configuring the networking stack to support an encrypted tunnel.

### How the Encryption Works (IKE and Diffie-Hellman)
A common misconception is that the Pre-Shared Key (PSK) you type in the portal is the encryption key. **It is not.**

1. **IKE Phase 1 (Authentication & DH Exchange)**: 
   - Azure and your on-prem router perform a **Diffie-Hellman (DH)** key exchange to mathematically calculate a massive, shared master secret (SKEYID). 
   - The **PSK** is only used to *authenticate* the gateways to each other to prevent a Man-in-the-Middle attack. 
2. **IKE Phase 2 (Data Encryption Keys)**: 
   - Using the Phase 1 master key, the gateways negotiate actual Data Encryption Keys (using AES-256). 
   - Your data is encrypted using these dynamic keys, not the PSK.
3. **Rekeying & Perfect Forward Secrecy (PFS)**: 
   - Keys expire (e.g., every 1 hour). 
   - With PFS enabled, the gateways run a brand-new Diffie-Hellman exchange from scratch for Phase 2. This ensures that even if an attacker compromises an old key, they cannot mathematically deduce the new keys.

---

## Load Balancer vs Application Gateway

| Feature | Load Balancer (LB) | Application Gateway (AppGW) |
|---------|--------------------|-----------------------------|
| **Layer** | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| **Intelligence** | Fast, simple packet forwarding (IP/Port). | Smart routing (URL path `/api/orders`), SSL Termination, Web Application Firewall (WAF). |
| **Subnet Requirement** | Can sit anywhere (or no subnet). | **Must have its own dedicated subnet.** |
| **Underlying Infra** | Purely SDN/VFP (Hash-based routing). | Managed Virtual Appliances (VMs running NGINX/IIS). |

### Why does Application Gateway need its own Subnet?
When you deploy an AppGW, Azure literally spins up dedicated virtual machines that act as Worker Nodes to handle SSL termination. If you autoscale from 2 to 10 instances, it grabs 10 IPs from that subnet.
Azure requires this subnet to isolate the gateway so no other resources interfere with Microsoft's management traffic. 
*(Pro-Tip: Ensure your NSG on this subnet allows Azure's "Gateway Manager" traffic, or the AppGW will fail).*

### Load Balancer Mechanics
To balance between VMs, you configure:
1. **Backend Pool**: The target VM network interfaces (NICs).
2. **Health Probe**: The LB constantly pings your VMs (e.g., Port 80 every 5s). If a VM crashes, the probe fails, and the LB instantly stops sending it traffic.
3. **Balancing Rule**: Connects the frontend Public IP to the Backend Pool.
Under the hood, the **VFP** performs Destination NAT (DNAT), translating the Load Balancer IP to the Private IP of the specific healthy node, and encapsulates the packet directly to that node's physical host.

---

## Private Link vs Service Endpoints
- **Private Link / Endpoints**: Allows you to access Azure PaaS services (like Azure SQL or Key Vault) over a private IP from your VNet. The data never leaves your VNet, providing the highest security.
- **Service Endpoints**: A simpler way to secure PaaS services. It forces traffic from your subnet to the PaaS service to stay on the Azure backbone, but the service is still accessed via its public IP address space.
