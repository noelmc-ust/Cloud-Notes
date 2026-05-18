# VNet Peering and Routing

## VNet Peering: The Global Backbone
When you have two separate VNets (e.g., a "Hub" and a "Spoke"), they are completely isolated. To connect them, you use **VNet Peering**.

### How it works:
- **Low Latency**: Traffic stays entirely on the Microsoft private backbone (the global dark fiber network). It never touches the public internet.
- **Non-Transitive**: VNet Peering is not transitive by default. If VNet A is peered with VNet B, and VNet B is peered with VNet C, **A and C cannot talk to each other**. To fix this, you must either peer A and C directly, or route their traffic through a Network Virtual Appliance (NVA) like an Azure Firewall in VNet B.
- **Underlying Infrastructure**: Azure uses Software Defined Networking (SDN). When you peer two VNets, there is **no gateway device** between them. Azure’s mapping service simply updates the underlying host physical network adapters (the Virtual Filtering Platform) to recognize that the private IP of VNet B is reachable directly via the internal fabric. It is a direct, host-to-host line at the infrastructure level.

### The Overlap Nightmare
Peered VNet address ranges **MUST NOT overlap**. If you try to use `10.0.0.0/16` for both the Hub and the Spoke, Azure will refuse to create the peering connection.
- **Why?**: The underlying SDN switch (VFP) would get confused. When a packet is destined for `10.0.0.5`, the VFP wouldn't know if that's "Internal 10.0.0.5" or "Peered 10.0.0.5".
- **Underlying Identifier**: In the physical wires, Azure tags every packet with a Virtual Network ID (VNID). VNet A might be VNID 5001, VNet B might be 7002. This allows thousands of customers to use `10.0.0.0/16` on the same physical hardware. However, when you Peer them, you are asking Azure to merge those two ID spaces for routing, hence why the IPs must be mathematically unique.

### Cross-Tenant VNet Peering
You can peer VNets across different Azure Subscriptions, and even across entirely different Microsoft Entra ID (Azure AD) Tenants (e.g., a service provider peering with a client).
- **Mechanism**: The administrator of Tenant A must grant permissions to a user or Service Principal from Tenant B as a Guest User.
- **Role**: Assign the `Network Contributor` role to that user for the specific VNet. 
- **Setup**: You must authenticate via Azure CLI or PowerShell using the cross-tenant credentials to establish the link.
- **Gateway Transit**: A powerful feature of peering. You can check "Use Remote Gateways" on the Spoke, allowing the Spoke to "borrow" the VPN tunnel located in the Hub to talk back to an on-premise office.

---

## Routing & Forced Tunneling

### Default Azure Routing
In a default Azure VNet, a Spoke naturally wants to send traffic directly to the internet. It uses the default Azure System Route (`0.0.0.0/0 → Internet`). While convenient, this is terrible for enterprise security.

### Forcing Traffic: The UDR and Firewall "Handshake"
To stop default internet routing, we use a **User Defined Route (UDR)**. A UDR is essentially a manual override of Azure's default routing table.

**The Mechanism of "Forced Tunneling":**
1. **The Route Table**: You create a Route Table resource and associate it with your AKS/Spoke subnets.
2. **The Rule**: You add a custom route with the following parameters:
   - **Address Prefix**: `0.0.0.0/0` (This means "All traffic")
   - **Next Hop Type**: `Virtual Appliance`
   - **Next Hop Address**: The Private IP of your Azure Firewall (located in the Hub).

### What is the "Next Hop"?
Think of the Next Hop as the "Instructions for the next leg of a journey."
Imagine you are in a building (The Spoke VNet) and want to mail a letter (Data Packet) to the outside world.
- **Default**: You walk out the front door yourself.
- **Forced Tunneling (UDR)**: There is a sign on your door that says: "All outgoing mail must be handed to the Security Desk (The Firewall) in Building B (The Hub)."

The Firewall is your Next Hop. Your only job is to get the packet to that specific Next Hop IP address. 

### Why do this for AKS?
When a packet leaves an AKS node destined for `google.com`, the Azure fabric looks at the UDR, ignores the default internet path, and "encapsulates" that packet to the Firewall’s internal NIC. The Firewall then decapsulates it, inspects it against your rules, and if allowed, sends it out through its own public IP.

1. **Centralized Logging**: You see every single connection attempt from every microservice in one central Firewall log.
2. **FQDN Filtering**: You can restrict your pods so they can only talk to `github.com` or `mcr.microsoft.com` (Layer 7 URL filtering). This prevents data exfiltration if a pod is compromised (which NSGs cannot do, since they only understand IPs, not URLs).
3. **IP Masquerading**: The outside world only sees the Hub Firewall's IP, completely hiding the internal structure of your Spoke network.
