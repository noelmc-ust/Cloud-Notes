# Identity & Access Management (Entra ID)

## Microsoft Entra ID
Formerly known as Azure Active Directory (Azure AD). It is Microsoft’s cloud-based identity and access management (IAM) service.

- **Authentication**: Verifies who you are (Passwords, MFA, Biometrics).
- **Authorization**: Manages what you can do (RBAC).
- **Differences from On-Prem AD**: Uses HTTP-based protocols (OAuth2, SAML, OpenID Connect) and a flat structure (Users/Groups), unlike on-prem AD which uses Kerberos/LDAP and OUs.

## Core Concepts
- **Tenant (The Organization)**: A dedicated instance of Microsoft Entra ID. It's the "Apartment Building" that belongs to your company.
- **Directory (The Database)**: The identity store inside the tenant. Contains Users, Groups, and Applications.
- **Service Principal**: An identity for an application/code (e.g., CI/CD pipelines).
- **Managed Identity**: An automated Service Principal. Allows Azure resources (like VMs or AKS pods) to authenticate to other resources (like Key Vault) without managing passwords.

## Management Hierarchy
1. **Tenant / Directory** (Organization Level)
2. **Management Groups** (Governance and Policy)
3. **Subscriptions** (Billing and Management Unit)
4. **Resource Groups** (Folders for related resources)
5. **Resources** (VMs, VNets, DBs)

## Cross-Tenant VNet Peering
You can peer VNets across different Subscriptions and different Entra ID Tenants.
- **Mechanism**: The admin of Tenant A invites a user/Service Principal from Tenant B as a Guest User. The guest is granted `Network Contributor` access to the VNet in Tenant A. Authentication via CLI/PowerShell establishes the link.
