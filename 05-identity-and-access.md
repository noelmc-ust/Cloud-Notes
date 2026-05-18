# Identity & Access Management (Entra ID)

## Microsoft Entra ID
Microsoft Entra ID is the new name for **Azure Active Directory (Azure AD)**. It is Microsoft’s cloud-based Identity and Access Management (IAM) service. It acts as the "brain" that handles logins, permissions, and security policies for everything in the Microsoft cloud.

### Entra ID vs Traditional On-Prem AD
Do not confuse Entra ID with old-school Windows Server Active Directory.
- **Protocol**: On-prem AD uses Kerberos and LDAP. Entra ID uses modern HTTP-based protocols (OAuth2, SAML, OpenID Connect).
- **Structure**: On-prem uses complex Forests and Organizational Units (OUs). Entra ID uses a flat structure of Users and Groups.
- **Primary Use**: On-prem manages local desktops. Entra ID manages cloud apps, mobile devices, SaaS (Office 365, Salesforce), and Azure APIs.

---

## Core Concepts & Hierarchy
To understand Azure management, you must understand the hierarchy of the "containers" that hold your resources.

1. **The Tenant (The Organization)**
   - Think of a Tenant as a secure, isolated Apartment Building that belongs only to your company. 
   - It is a dedicated instance of Microsoft Entra ID, identified by a unique GUID and a domain (e.g., `company.onmicrosoft.com`).
2. **The Directory (The Database)**
   - The Directory is the identity store (the "phone book") inside the Tenant. It contains all the identities: Users, Groups, and Applications. If a user isn't in your directory, they don't exist in your cloud.
3. **Management Groups**
   - A container for multiple subscriptions. Used to apply governance and policies across many subscriptions at once.
4. **Subscriptions**
   - A logical billing and management unit. A Tenant can have many subscriptions (e.g., "Production", "Development").
5. **Resource Groups**
   - Folders inside a subscription to group related resources for a specific project.
6. **Resources**
   - The actual VMs, VNets, and Databases.

---

## IAM: Identity and Access Management
IAM ensures the right people have the right access to resources, primarily handled through **Role-Based Access Control (RBAC)**.
- **Authentication (AuthN)**: Are you who you say you are? (Handled by the Directory via Passwords, MFA).
- **Authorization (AuthZ)**: What are you allowed to do? (Handled by RBAC Roles attached to your identity).

### Non-Human Identities (DevOps Crucials)
As a DevOps engineer, you will heavily use non-human identities:
- **Service Principal**: An identity for an application or code. It acts like a "user account" for a script or CI/CD pipeline (e.g., GitHub Actions) so it can deploy to Azure without a human password.
- **Managed Identity**: An automated Service Principal. It allows Azure resources (like an AKS Pod, a VM, or an Azure Function) to authenticate to other Azure services (like Key Vault or PostgreSQL) automatically, entirely eliminating the need for you to store hardcoded secrets in your application code.

---

## Cross-Tenant & Cross-Subscription VNet Peering
You can peer Virtual Networks across different Subscriptions and even across different Entra ID Tenants.

### 1. Cross-Subscription (Same Tenant)
If you own two subscriptions (e.g., Dev and Prod) under the same Entra ID Tenant, you just need the `Network Contributor` role on both VNets to peer them.

### 2. Cross-Tenant (Different Directories)
This allows connectivity between entirely different Azure accounts (e.g., a Service Provider peering with a Client VNet).
Because Tenant A and Tenant B are isolated "Buildings", IAM rules in Tenant A do not apply to Tenant B.
- **The Process**:
  1. The administrator of Tenant B must invite a user or Service Principal from Tenant A as a **Guest User** in Tenant B's Directory.
  2. The admin assigns the `Network Contributor` role to that Guest User for the specific VNet in Tenant B.
  3. You authenticate via Azure CLI or PowerShell using those cross-tenant credentials to establish the peering link.
