# Azure SDN & Underlay (VFP)

To understand Azure networking, you must look past the Azure Portal and understand the "Silicon and Software" layer. Azure operates entirely on a **Software-Defined Network (SDN)**. When you create subnets, peer VNets, or change route tables, you aren't actually re-cabling anything. Everything is a dance between the Control Plane (the Brain) and the Data Plane (the Muscle).

## 1. The Data Plane: VFP (Virtual Filtering Platform)
The **Virtual Filtering Platform (VFP)** is the core of Azure's Data Plane. It is a highly optimized, programmable virtual switch that runs on every single physical host (server) in every Azure datacenter, sitting inside the Hyper-V layer.

- **The Match-Action Engine**: VFP sits exactly between the physical Network Interface Card (NIC) of the server and the virtual NIC (vNIC) of your Virtual Machine (or AKS node). 
- For every single packet that leaves or enters a VM, VFP asks:
  1. **Match**: Does this packet (IP, Port, Protocol) match any rule I have?
  2. **Action**: Should I Allow, Deny, Encapsulate (wrap it for a VNet), or Translate (NAT) it?

### VFP Layers
VFP executes rules in layers, programmed by different controllers:
- **Layer 1 (ACLs)**: NSG controller checks: "Is this traffic allowed to leave?" (Firewall rules).
- **Layer 2 (VNet)**: Virtual Network controller checks: "Where is the destination? Is it in a peered VNet? If so, wrap it in a VXLAN tunnel."
- **Layer 3 (Load Balancer)**: SLB controller says: "I need to NAT this IP to the correct backend."

### Stateful Memory
VFP performs **Stateful Inspection**. If you allow an outbound request from your AKS pod to the internet (e.g., to `8.8.8.8`), VFP automatically "remembers" that connection. When the response comes back, VFP recognizes the return packet and lets it in, even if you don't have an explicit inbound rule.

---

## 2. The Control Plane: Host Agent & Network Controller

### The Azure Directory (The Phonebook)
Where does VFP get its mapping data? The mappings live in a massive, globally distributed database called the **Azure Directory Service**.
Every time a VM is created, its **Private IP**, **MAC address**, the **Physical IP of the Host Server** it sits on, and its **VNet ID (VNID)** are recorded here.

### The Network Controller (The Brain)
The Network Controller monitors the Directory. Azure uses an **Event-Driven Push Model (Subscribe/Notify)**. 
- The Controller doesn't wait for a packet to be sent to look up data. 
- When a change occurs (e.g., a VM is created, or a VNet peering is enabled), the Directory is updated.
- The Controller gets a push notification instantly. It calculates which physical hosts in the region are affected and pushes the new routing "policies".

### The Host Agent (The Local Manager)
There is one **Host Agent** per physical server. 
- It acts as the local ambassador. It receives instructions from the Network Controller and "plumbs" (injects) these rules into the VFP's local lookup tables.
- The VFP doesn't "search" the internet for a mapping; the mapping is pre-fed to it by the Host Agent so it can make routing decisions at wire speed.

---

## 3. The Packet Journey: "Layer 2.5" Encapsulation

How does a packet travel from VM-A on Server 1 to VM-B on Server 2?

1. **The Departure**: VM-A sends a packet to `10.0.2.5` (VM-B). The packet hits the vNIC and enters the VFP on Host 1.
2. **The Lookup**: VFP checks its Flow Table. It sees that `10.0.2.5` is located on Physical Host 2.
3. **Encapsulation (Layer 2.5)**: VFP takes the original packet and wraps it in a **VXLAN (or NVGRE)** envelope.
   - **Outer Header**: Source = IP of Physical Host 1; Destination = IP of Physical Host 2.
   - **Inner Header**: Source = IP of VM-A; Destination = IP of VM-B.
   - **VNet ID**: It attaches the unique ID of your VNet so the physical wires know this isn't another customer's traffic.
4. **The Wire**: The packet travels across the physical datacenter switches. The physical routers only care about the Outer header (the physical hosts). They don't even know there is a VM inside.
5. **The Arrival**: Host 2's VFP receives the packet, recognizes the VXLAN header, "unwraps" the envelope, checks the Inbound NSG rules, and places the bare packet onto VM-B’s vNIC. To VM-B, it looks like the packet just arrived locally.

*Note: For VNet Peering, the exact same process happens. The Controller just pushes the destination VNet's Host IP mappings to the source VFP. The packet is encapsulated and travels directly Host-to-Host over the Microsoft backbone without hitting a centralized router.*

---

## 4. Accelerated Networking & FPGAs (Hardware Bypass)

Normally, the VFP runs in the CPU of the physical host (Software Path), which costs CPU cycles and adds latency.

**Accelerated Networking (AccelNet)** bypasses the Host CPU entirely:
- **FPGA (SmartNIC)**: Modern Azure physical NICs have a Field Programmable Gate Array (FPGA)—a hardware chip that can be reprogrammed on the fly.
- **Hardware Offloading**: The Host Agent pushes the VFP rules directly into the FPGA silicon.
- **The Result**: When a VM sends a packet, it goes directly to the FPGA. The FPGA looks at its hardware-level Flow Table, encapsulates the packet in VXLAN, and sends it out onto the fiber. 
- **Performance**: The Host CPU never touches the packet, reducing latency from ~100 microseconds to ~15 microseconds and allowing for near-wire speed (up to 100Gbps).
