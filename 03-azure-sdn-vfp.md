# Azure SDN & Underlay (VFP)

Azure operates on a Software-Defined Network (SDN). You aren't actually re-cabling anything when you peer VNets.

## VFP (Virtual Filtering Platform) and Host Agents
Every physical server in an Azure Datacenter runs a specialized agent and a technology called Virtual Filtering Platform (VFP).

- **VFP (Data Plane)**: The "Virtual Switch" code inside the Hyper-V switch that inspects every packet. It sits between the physical NIC of the server and the virtual NIC of your VM.
- **Host Agent (Control Plane)**: The "Local Manager" (one per physical server) that receives instructions from the global Azure Network Controller and programs the VFP.

### The Azure Directory & Network Controller
- **Azure Directory Service**: The "Phonebook." Records the VM ID, Private IP, MAC address, and Physical Host IP.
- **Network Controller**: The "Brain." Subscribes to the Directory. When a change occurs (e.g., VNet Peering created, VM moved), it instantly sends a tiny message to the Host Agents of the affected physical servers.

### The Packet Journey (Internal VNet)
1. **Departure**: VM-A sends a packet to VM-B. Packet hits the VFP on Host 1.
2. **Lookup**: VFP checks its Flow Table (programmed by the Host Agent) and finds VM-B's Physical Host IP.
3. **Encapsulation (Layer 2.5)**: VFP wraps the packet in a VXLAN (or NVGRE) envelope.
   - *Outer Header*: Host 1 IP -> Host 2 IP.
   - *Inner Header*: VM-A IP -> VM-B IP.
   - *VNet ID*: Identifies your specific VNet.
4. **The Wire**: The packet travels across the datacenter over the Microsoft physical backbone.
5. **Arrival**: Host 2's VFP unwraps the VXLAN header, checks inbound NSG rules, and places the packet onto VM-B’s vNIC.

### Accelerated Networking (AccelNet) & FPGAs
Normally, VFP runs on the Host CPU, costing cycles and adding latency.
- **FPGA (SmartNIC)**: With Accelerated Networking, VFP rules are offloaded directly into the FPGA chip on the physical NIC.
- **Hardware Bypass**: The Host CPU never touches the packet. The hardware NIC executes the VFP lookup and encapsulation at wire speed, reducing latency from ~100us to ~15us.
