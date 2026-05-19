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

How UDR Works
Azure's default routing behavior follows a system table. For example, if VM A wants to talk to VM B, Azure routes it directly through its internal fabric.

When you create a UDR, you are building a custom Route Table resource, defining specific paths, and then associating that Route Table to one or more subnets.

Every custom route you define in a UDR requires three main pieces of information:

Address Prefix: The destination IP range you want to target (e.g., 10.0.0.0/16 or 0.0.0.0/0 for all traffic).

Next Hop Type: The mechanism or destination that will handle the traffic.

Next Hop Address: The specific IP address of the appliance handling the traffic (only required for certain hop types).

How Azure Chooses a Route (The Selection Order)
If multiple routes exist to a destination, Azure uses the following hierarchy to decide which one wins:

Longest Prefix Match: The most specific route always wins. A route for 10.0.1.0/24 will take precedence over 10.0.0.0/16.

User-Defined Routes (UDR): If the prefixes are an exact match, a UDR will always override Azure's default System Routes.

BGP Routes: Routes learned via ExpressRoute or VPN Gateways using BGP.

System Routes: The default fallback routes provided by Azure.

What All We Can Do With UDRs (Common Next Hop Types)
The power of a UDR comes down to what you set as the Next Hop Type. Here are the primary architectural maneuvers you can accomplish:

1. Force Traffic Through a Firewall (Virtual Appliance)
Next Hop Type: Virtual Appliance

What it does: You specify the private IP address of a Network Virtual Appliance (NVA)—like an Azure Firewall, Palo Alto, or Fortinet NVA.

Use Case: You want to ensure that all outbound traffic from your database subnet to the internet (0.0.0.0/0) must first pass through a firewall for deep packet inspection and filtering.

2. Bypass or Change VNet Peering Slugs (VNet Local)
Next Hop Type: Virtual Network

What it does: Forces traffic to stay within the VNet boundary or routes it cleanly over VNet Peerings.

Use Case: If you want to temporarily drop or reroute traffic between peered networks for maintenance or isolation, you can manually override the default peering paths.

3. Drop Traffic Completely (None / Blackhole)
Next Hop Type: None

What it does: Acts as a network "blackhole." Any traffic matching this destination prefix is silently dropped.

Use Case: Security isolation. If you have a compromised subnet or a legacy environment that should absolutely never communicate with a specific database subnet, you can create a UDR to drop that specific internal traffic.

4. Force Tunneling to On-Premises (Virtual Network Gateway)
Next Hop Type: Virtual Network Gateway

What it does: Routes traffic directly to your Azure VPN or ExpressRoute gateway.

Use Case: Forced Tunneling. Many enterprises have strict compliance rules stating that cloud resources cannot access the internet directly. You can create a UDR with a destination of 0.0.0.0/0 and a next hop of Virtual Network Gateway. This forces all internet-bound traffic to go back to your on-premises corporate firewall before hitting the web.

5. Direct Internet Access
Next Hop Type: Internet

What it does: Forces traffic directly out to the Azure infrastructure's public routing.

Use Case: If you have an NVA or a proxy server that needs an uninhibited, direct pipeline to the internet to fetch updates without hitting other internal firewalls, you can explicitly grant it using this hop.

Important Rules to Keep in Mind
Subnet Level Only: You associate a Route Table to a subnet, not to individual Virtual Machines or NICs. Every resource inside that subnet will inherit those UDR rules.

Gateway Subnet Limitations: You can associate a UDR to the GatewaySubnet (where your VPN/ExpressRoute lives), but it only supports specific scenarios, like steering incoming on-prem traffic straight to a firewall.

Asymmetric Routing Risk: If you route traffic out of a subnet via an NVA/Firewall using a UDR, you must ensure the return traffic is also steered correctly. If the return packet takes a default system shortcut and bypasses the firewall on the way back, the firewall will drop the connection because it never saw the stateful handshake.

1. Industry Standard: The Separation of Firewall and VPN GatewayIn a production environment, bundling a massive firewall and a high-throughput VPN endpoint into a single virtual appliance introduces a bottleneck and a single point of failure. Instead, enterprises adopt the Hub-and-Spoke Network Topology.In this architecture:The VPN Gateway (The Router): This is a dedicated, cloud-native resource (like an Azure VPN Gateway or ExpressRoute Gateway). Its only job is to handle the heavy mathematical lifting of the IPsec tunneling, BGP routing, and cryptographic handshakes (Phase 1 and Phase 2) with your on-premise datacenter or multi-cloud router.The Firewall (The Inspector): This is a separate, highly scalable security appliance (like Azure Firewall, Palo Alto, or Fortinet NVA). Its only job is stateful packet inspection, deep packet inspection (DPI), IDPS, and web/URL filtering.  Why do we separate them?Blast Radius: If the VPN gateway gets flooded with a massive influx of cross-premises data, your internal firewall capabilities remain unhindered.Asymmetric Scaling: VPN throughput needs scale differently than security inspection rules. You can upgrade your firewall SKU without having to tear down and rebuild your stable production IPsec tunnels.Zero Trust Integration: The VPN allows traffic into the cloud, but the Firewall decides which workloads that traffic can actually talk to.2. Common Enterprise Use Cases for UDRsBecause the Firewall and VPN Gateway are physically separate resources sitting in different subnets, Azure's default system routing will naturally try to bypass the firewall to save hops. We use UDRs to force the traffic through the firewall anyway.  Here are the three standard enterprise use cases where UDRs are mandatory:Use Case A: East-West Traffic (Spoke-to-Spoke Inspection)Scenario: You have a Production-Spoke VNet and a Shared-Services-Spoke VNet. By default, if they are peered, they talk directly to each other.The UDR: You attach a UDR to the Production Subnet stating: "If you want to talk to the Shared Services IP range, your Next Hop is the Private IP of the Hub Firewall."Use Case B: North-South Egress (Secure Internet Access)Scenario: Your internal database servers in a Spoke VNet need to download an OS patch from the internet, but they cannot have public IP addresses.The UDR: You attach a UDR to the Database Subnet: 0.0.0.0/0 (All Internet Traffic) $\rightarrow$ Next Hop: Private IP of the Hub Firewall.Use Case C: Ingress Inspection from On-PremisesScenario: Traffic arrives from your on-premises datacenter through the VPN Gateway. By default, the Gateway knows how to route directly to the Spokes, bypassing the firewall.  The UDR: You attach a UDR directly to the GatewaySubnet: "If traffic is heading toward any Spoke VNet range, the Next Hop is the Private IP of the Hub Firewall."  3. How it Setup Step-by-Step (The Traffic Flow)To understand exactly how the industry sets this up using UDRs, let's track a packet traveling from an On-Premises Server to an Azure Workload VM and back.

[On-Prem Server] 
       │
 (IPsec VPN Tunnel over Internet)
       ▼
[Hub VNet: GatewaySubnet] (Terminates IPsec, reads Gateway UDR)
       │
       ▼  Next Hop: Firewall Private IP
[Hub VNet: AzureFirewallSubnet] (Inspects packet, allows via Network Rule)
       │
 (VNet Peering Fabric)
       ▼
[Spoke VNet: Workload Subnet] (Packet reaches Azure Workload VM)

Step 1: The Inbound Path (On-Prem to Azure)The On-Prem server sends a packet targeting the Azure VM (10.2.1.4).The On-Prem router encrypts it, sends it through the IPsec tunnel, and the Azure VPN Gateway decrypts it inside the GatewaySubnet.The VPN Gateway looks at its routing table. Because we applied a Gateway UDR, the system route is overridden. The UDR says: To reach 10.2.0.0/16 (Spokes), send the traffic to 10.0.0.4 (the Firewall).The packet moves to the AzureFirewallSubnet. The Firewall evaluates its Layer 4/Layer 7 rules, confirms it's safe, and forwards it across the VNet Peering to the Workload VM.Step 2: The Outbound Return Path (Azure back to On-Prem)For the connection to work, the asymmetric trap must be avoided. The return path must mirror the entry path perfectly:The Azure VM replies to the On-Prem server IP (192.168.1.50).The Workload Subnet has its own Spoke UDR attached. This UDR states: To reach 192.168.1.0/24 (On-Prem), your Next Hop is 10.0.0.4 (the Hub Firewall).  The reply packet goes back to the Firewall first. The Firewall verifies the stateful connection match.The Firewall then hands the packet over to the VPN Gateway, which re-encrypts it into the IPsec tunnel and sends it back home.

### Why do this for AKS?
When a packet leaves an AKS node destined for `google.com`, the Azure fabric looks at the UDR, ignores the default internet path, and "encapsulates" that packet to the Firewall’s internal NIC. 
1. **Centralized Logging**: You see every single connection attempt in one Firewall log.
2. **FQDN Filtering**: Restrict pods so they can only talk to specific URLs (Layer 7 filtering), preventing data exfiltration if a pod is compromised.
3. **IP Masquerading**: The outside world only sees the Hub Firewall's IP.
