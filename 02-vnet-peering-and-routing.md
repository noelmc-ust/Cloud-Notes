# VNet Peering and Routing

## VNet Peering: The Global Backbone
When you have two separate VNets (e.g., a "Hub" and a "Spoke"), you connect them via VNet Peering.

### How it works:
- **Low Latency**: Traffic stays on the Microsoft private backbone. It never touches the public internet.
- **Non-Transitive**: If VNet A is peered with VNet B, and B is peered with C, A and C cannot talk unless you peer them directly or use a Network Virtual Appliance (NVA) in the middle.
- **Underlying Infrastructure**: Azure uses Software Defined Networking (SDN). Azure’s mapping service updates the underlying host physical network adapters (NICs) to recognize that the private IP of the peered VNet is reachable. There is no gateway device between them; it is a direct line.

### Overlap Nightmare
Peered VNet address ranges **MUST NOT overlap**. If you try to use `10.0.0.0/16` for both, the peering will fail. Azure needs unique CIDRs to route packets accurately.

## Routing
### Default vs Forced Routing
- **Default Behavior**: A packet going to the internet travels directly out (via Azure System Route `0.0.0.0/0 → Internet`).
- **Forced Tunneling (UDR)**: Uses a User Defined Route to override the default path.

### User Defined Routes (UDR)
A UDR is essentially a manual override of Azure's default routing table.
- **Route Table**: Created and associated with your subnets.
- **Rule Example**: 
  - **Address Prefix**: `0.0.0.0/0` (All traffic)
  - **Next Hop Type**: `Virtual Appliance`
  - **Next Hop Address**: Private IP of your Azure Firewall (located in the Hub).

### The Next Hop
Think of the Next Hop as "Instructions for the next leg of a journey."
When you apply a UDR, instead of going to the internet, traffic is encapsulated and sent to the "Security Desk" (the Firewall's internal NIC). The Firewall decapsulates it, inspects it, and if allowed, sends it out through its own public IP.

**Why do this for AKS?**
- **Centralized Logging**: You see every single connection attempt in one Firewall log.
- **FQDN Filtering**: Prevent data exfiltration by restricting pod access (e.g., only allow `github.com`).
- **IP Masquerading**: Hides the internal structure of your Spoke network from the outside world.
