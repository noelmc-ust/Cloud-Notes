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


Step 1: The Interesting Traffic Trigger
An IPsec tunnel doesn't just stay active all the time by default; it usually needs a reason to spin up.

What happens: A computer on Network A tries to send a packet to a computer on Network B (e.g., pinging a private IP across the cloud/on-prem boundary).

The Match: The local VPN gateway intercepts this packet and checks its Crypto ACL (Access Control List) or Traffic Selector.

The Result: If the source and destination IPs match the policy, the router marks this as "Interesting Traffic" and realizes it needs to build a secure tunnel to forward it.

Step 2: IKE Phase 1 – Negotiation (The Proposal)
Before the two gateways can talk securely, they have to agree on how they are going to talk. The initiating gateway sends a list of cryptographic options to the responding gateway.

They must find a matching match for four main parameters (often remembered by the acronym HAGLE):

Hash: The algorithm used for data integrity (e.g., SHA-256).

Authentication: How they will prove their identities (e.g., Pre-Shared Key or Digital Certificates).

Group: The Diffie-Hellman (DH) group, which determines the strength of the keys they will make (e.g., Group 14, Group 19).

Lifetime: How long this management session will last before expiring.

Encryption: The algorithm used to protect this management traffic (e.g., AES-256).

Step 3: IKE Phase 1 – Diffie-Hellman Key Exchange & Authentication
Once they agree on the cryptographic parameters, they build the secure management channel.

The DH Exchange: The gateways exchange public Diffie-Hellman values. Using these public values and their own private values, both sides mathematically calculate the exact same secret master key (SKEYID). No keys are actually sent over the internet.

Authentication: The gateways use the configured Pre-Shared Key (PSK) or certificates to sign a hash of the exchange. If both sides can successfully decrypt and verify the signature, they have authenticated each other.

The Result: A secure, bi-directional management tunnel called the IKE Phase 1 SA (Security Association) or ISAKMP SA is established. All future management messages are now encrypted.

Step 4: IKE Phase 2 – Negotiating the Data Tunnel (Quick Mode)
Now that the gateways have a secure "private room" to talk in, they negotiate the actual tunnel that will carry the real user data. This is called Quick Mode.

Inside the protected Phase 1 tunnel, they negotiate:

IPsec Protocol: Usually ESP (Encapsulating Security Payload), which encrypts the data, rather than AH (Authentication Header), which only signs it.

Encryption & Hash: The specific AES and SHA variations used for the actual user data (these can be different from Phase 1).

PFS (Perfect Forward Secrecy): If enabled, the gateways will force a brand-new Diffie-Hellman key exchange right now to ensure Phase 2 keys are completely independent of Phase 1 keys.

Step 5: Establishing the IPsec Data Tunnel
Once Phase 2 negotiations are complete, the gateways create two uni-directional tunnels (one for inbound traffic, one for outbound traffic). These are called IPsec SAs.

Each IPsec SA is assigned a unique identifier called an SPI (Security Parameter Index). When a gateway encrypts a packet, it stamps it with the SPI so the receiving gateway instantly knows which cryptographic keys and parameters to use to decrypt it.

Step 6: Data Transfer (The Active Tunnel)
The tunnel is now fully established and functional.

The original packet from Step 1 is encapsulated.

The ESP header is attached (which includes the SPI).

The entire original packet (payload and original IP headers) is completely encrypted.

A New IP Header is slapped on the outside, using the public IP addresses of the two VPN gateways as the source and destination.

The packet travels across the public internet safely. When it hits the remote gateway, it is decrypted and forwarded to its final destination network.

Step 7: Tunnel Teardown or Rekeying
The tunnel will not stay open indefinitely.

Rekeying: Before the Phase 1 or Phase 2 lifetimes expire (or after a certain amount of gigabytes have passed), the gateways will quietly spin up a new negotiation in the background to create fresh keys, ensuring data flow is never interrupted.

Teardown: If no traffic passes through the tunnel for an extended period (Idle Timeout), or if DPD (Dead Peer Detection) notices that the other side has gone offline, the gateways will tear down the SAs and clear the memory caches until "Interesting Traffic" triggers the process all over again.
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
