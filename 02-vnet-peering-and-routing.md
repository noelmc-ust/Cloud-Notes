# VNet Peering and Routing

## VNet Peering: The Global Backbone
When you have two separate VNets (e.g., a "Hub" and a "Spoke"), they are completely isolated by default. To connect them, you use **VNet Peering**. 

### Why we need VNet Peering (Realtime Use Cases)
- **Shared Services**: Allowing multiple departmental Spokes to reach a central DNS server or Active Directory Domain Controller in a Hub.
- **Database Replication**: Peering a Web VNet in one region to a Database VNet in another region to keep backend data synced.
- **Hybrid Cloud**: Allowing Spokes to use a single VPN Gateway located in a Hub to talk to an on-premise datacenter.

### Regional vs Global VNet Peering
- **Regional VNet Peering**: Connects VNets within the *same* Azure region (e.g., East US to East US).
- **Global VNet Peering**: Connects VNets across *different* Azure regions (e.g., London to New York). 

### How we are bypassing public routing (Microsoft Backbone)
What is meant by the Microsoft Backbone providing the infrastructure? 
When you peer VNets across the globe, the traffic **never touches the public internet**. It travels exclusively across Microsoft's massive private dark fiber network. You are bypassing the unpredictability and security risks of public ISP routing and going along with private routing.

#### Microsoft "Cold Potato" Routing
Microsoft uses "Cold Potato" routing. This means they keep your traffic on their private fiber backbone for as long as possible before handing it off to the destination, ensuring the highest security, lowest latency, and best stability.

*Example*: **When you peer VNet A (London) with VNet B (New York):**
1. The Host Agent in London is told: "To reach the New York IP range, use this specific tunnel ID."
2. The VFP encapsulates the packet.
3. The packet is handed off to the Microsoft Cold Potato Routing system, traveling under the ocean on private Microsoft cables all the way to the New York datacenter.

### How it Works Structurally:
- **Non-Transitive**: VNet Peering is not transitive by default. If VNet A is peered with VNet B, and VNet B is peered with VNet C, **A and C cannot talk to each other**. To fix this, you must either peer A and C directly, or route their traffic through a Network Virtual Appliance (NVA).
- **Underlying Infrastructure**: Azure uses Software Defined Networking (SDN). There is **no gateway device** between peered VNets. Azure’s mapping service simply updates the Virtual Filtering Platform (VFP) to recognize that the private IP of VNet B is reachable directly via the internal fabric. It is a direct, host-to-host line.

### The Overlap Nightmare
Peered VNet address ranges **MUST NOT overlap**. If you try to use `10.0.0.0/16` for both the Hub and the Spoke, Azure will refuse to create the peering connection.
- **Why?**: The underlying SDN switch (VFP) would get confused. When a packet is destined for `10.0.0.5`, the VFP wouldn't know if that's "Internal 10.0.0.5" or "Peered 10.0.0.5".
- **Underlying Identifier**: In the physical wires, Azure tags every packet with a Virtual Network ID (VNID) to keep overlapping customer networks separate. When you Peer, you ask Azure to merge those two ID spaces for routing, hence why the IPs must be mathematically unique.

---

## Routing & Forced Tunneling

### Default Azure Routing
In a default Azure VNet, a Spoke naturally wants to send traffic directly to the internet. It uses the default Azure System Route (`0.0.0.0/0 → Internet`). While convenient, this is terrible for enterprise security.

### Forcing Traffic: The UDR and Firewall "Handshake"
To stop default internet routing, we use a **User Defined Route (UDR)**. A UDR is essentially a manual override of Azure's default routing table.

**The Mechanism of "Forced Tunneling":**
1. **The Route Table**: You create a Route Table resource and associate it with your subnets.
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
When a packet leaves an AKS node destined for `google.com`, the Azure fabric looks at the UDR, ignores the default internet path, and "encapsulates" that packet to the Firewall’s internal NIC. 
1. **Centralized Logging**: You see every single connection attempt in one Firewall log.
2. **FQDN Filtering**: Restrict pods so they can only talk to specific URLs (Layer 7 filtering), preventing data exfiltration if a pod is compromised.
3. **IP Masquerading**: The outside world only sees the Hub Firewall's IP.
