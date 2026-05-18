# Compute & Microservices Architecture

## Azure Compute Options
From most control to least control (more automation):
- **Virtual Machines (IaaS)**: Legacy / full control and responsibility.
- **App Service (PaaS)**: Managed runtime, great for simple web apps.
- **Azure Container Instances**: Quick container runs.
- **Azure Container Apps**: Serverless microservices.
- **AKS (Azure Kubernetes Service)**: Full microservices platform.
- **Azure Functions (FaaS)**: Event-driven architecture.

## Microservices Architecture in AKS
Architecting for Zero Trust and Scalability:

- **Networking**: Use Azure CNI (Advanced Networking) so every pod gets a real IP from your VNet.
- **Control Plane**: Use a Private AKS Cluster. The API server is accessed via Private Endpoint.
- **Nodes**: Placed in a Private Subnet (no public IPs).
- **Egress**: Use an Azure NAT Gateway. Provides a static public IP for outbound internet traffic, preventing SNAT port exhaustion.
- **Ingress Layer 4**: Azure Load Balancer for TCP/UDP.
- **Ingress Layer 7**: Application Gateway (AGIC) acts as a Web Application Firewall (WAF) and handles SSL termination and URL path routing.
