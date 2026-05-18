# Compute & Microservices Architecture

## Azure Compute Spectrum
Azure provides a spectrum of compute options, trading off between the level of control you need versus the amount of infrastructure management you want to offload to Azure.

From **Highest Control** (Work More) to **Highest Automation** (Work Less):
1. **Virtual Machines (IaaS)**: Legacy or lift-and-shift workloads. You have full control and full responsibility for the OS, patching, and networking. It is the cloud version of an on-prem server.
2. **App Service (PaaS)**: Managed runtime. Great for simple web apps. You have no VM management; you just focus on your code and deployment.
3. **Azure Container Instances (ACI)**: Serverless containers. Ideal for quick, one-off container runs or batch jobs without provisioning a cluster.
4. **Azure Container Apps**: Serverless microservices platform built on AKS, but without the complexity of managing Kubernetes yourself.
5. **AKS (Azure Kubernetes Service)**: Full microservices orchestration platform. Provides immense power and scalability but requires deep networking and operational knowledge.
6. **Azure Functions (FaaS)**: Event-driven architecture. "Run this code when something happens." You don't think about infrastructure at all.

---

## Enterprise Microservices Architecture in AKS
When deploying a microservices application in AKS for an enterprise, you don't just "hit deploy." You must architect the cluster for **Zero Trust** and massive scalability.

### 1. Networking: Azure CNI
You should use **Azure CNI (Advanced Networking)** instead of kubenet. 
- **Why?**: Azure CNI gives every single Pod a real, routable IP address from your VNet. This makes it significantly easier to connect your microservices to on-premise databases or other Azure PaaS services because the pods appear as first-class citizens on the Azure network.

### 2. Private vs Public Isolation
- **Control Plane**: Deploy a **Private AKS Cluster**. This ensures that the Kubernetes API server (the brain of the cluster) is only accessible via a Private Endpoint located within your VNet, completely hiding it from the public internet.
- **Worker Nodes**: Place the node pools in a **Private Subnet**. Your VMs should never have public IPs assigned to them.

### 3. Outbound Connectivity (Egress)
If your nodes are private, how do your pods reach the internet to pull images or call external APIs?
- **Azure NAT Gateway**: You attach a NAT Gateway to your Private Subnet. The NAT Gateway is assigned a static Public IP. 
- **The Problem it Solves**: Without it, large AKS clusters suffer from "SNAT port exhaustion" when thousands of pods try to use the default Azure load balancer to reach the internet simultaneously. NAT Gateway dynamically allocates SNAT ports and prevents connection drops.

### 4. Inbound Connectivity (Ingress)
How does user traffic reach your pods securely?
- **Layer 4 (L4) Ingress**: An internal Azure Load Balancer handles TCP/UDP traffic and routes packets based purely on IP and Port.
- **Layer 7 (L7) Ingress**: Use the **Azure Application Gateway (AGIC - Application Gateway Ingress Controller)**. It acts as a Web Application Firewall (WAF), handles SSL termination (decrypting HTTPS traffic), and performs smart routing based on URL paths (e.g., routing `/api/orders` to the orders microservice and `/api/users` to the users microservice).
