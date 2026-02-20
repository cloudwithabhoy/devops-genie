# Azure — Basic Questions

---

## 1. What is a VNet and NSG in Azure?

These are two foundational Azure networking concepts — VNet is your private network, NSG is the firewall that controls traffic within it.

**VNet (Virtual Network):**

A VNet is Azure's equivalent of AWS VPC — a logically isolated network inside Azure where your resources (VMs, AKS clusters, App Services, databases) run. You define the IP address space and divide it into subnets.

```
VNet: 10.0.0.0/16
├── Subnet: frontend    10.0.1.0/24  → Web servers, load balancers
├── Subnet: backend     10.0.2.0/24  → App servers, AKS nodes
└── Subnet: database    10.0.3.0/24  → Azure SQL, Cosmos DB
```

**Key VNet concepts:**

- **Subnets** — subdivisions of the VNet. Each subnet can have its own NSG and route table. Resources in the same VNet can talk to each other by default.
- **VNet Peering** — private connection between two VNets (even across regions). Traffic stays on the Azure backbone, doesn't go through the internet. Used to connect the app VNet to the shared-services VNet (where monitoring, logging, and DevOps tooling live).
- **VNet Integration** — allows App Services and Functions to make outbound calls into a VNet (to reach databases in private subnets).
- **Private Endpoints** — bring Azure PaaS services (Azure SQL, Storage, ACR, Key Vault) into your VNet with a private IP. Traffic stays entirely private — no internet exposure.

```
Before Private Endpoint: App → Internet → Azure SQL public endpoint
After Private Endpoint:  App → VNet private IP → Azure SQL (no internet traversal)
```

---

**NSG (Network Security Group):**

An NSG is a stateful firewall applied at the **subnet** or **NIC** (network interface) level. It contains inbound and outbound security rules that allow or deny traffic based on source/destination IP, port, and protocol.

```
NSG Rule:
Priority: 100
Name:     Allow-HTTPS-Inbound
Direction: Inbound
Protocol:  TCP
Source:    * (any)
Destination: *
Port:      443
Action:    Allow

Priority: 65500  ← Always last, Azure-managed
Name:     DenyAllInbound
Action:   Deny
```

Lower priority number = evaluated first. Azure has default deny-all rules at the end (priority 65500). You add explicit Allow rules.

**In production — how we use NSGs:**

```
Internet → Azure Load Balancer → frontend subnet (NSG: allow 80/443 inbound)
                                        ↓
                              backend subnet (NSG: allow 8080 from frontend subnet only)
                                        ↓
                              database subnet (NSG: allow 1433 from backend subnet only)
```

Each NSG enforces that traffic flows only in the expected direction. The database subnet's NSG explicitly denies any traffic that doesn't come from the backend subnet — a compromised frontend pod can't directly reach the database.

**NSG vs Azure Firewall:**

NSG is layer 4 (IP/port). Azure Firewall is layer 7 — it can inspect HTTP headers, do URL filtering, and enforce FQDN-based rules. We use NSGs for VNet-internal traffic control and Azure Firewall for centralized egress control (all outbound internet traffic from the VNet goes through Azure Firewall for logging and filtering).

**Real scenario:** We deployed a new microservice in the backend subnet and it couldn't reach our Azure Service Bus. The service's NSG allowed all outbound traffic from the backend subnet, but we'd added a route table that sent all egress through Azure Firewall. Azure Firewall's application rules didn't have a rule for `*.servicebus.windows.net`. Traffic was silently dropped. Fix: added an Azure Firewall application rule allowing HTTPS to `*.servicebus.windows.net`. NSG alone wasn't enough because Azure Firewall sat in the egress path too.

---

## 2. What is Azure AD (Microsoft Entra ID) and why is it used?

Azure AD (now officially called Microsoft Entra ID) is Microsoft's cloud-based identity and access management service. It's the central identity system for Azure — every Azure resource access, every Azure Portal login, and every Azure CLI command is authenticated through it.

**What it does:**

- **Authentication** — verifies who you are (username/password, MFA, passwordless)
- **Authorization** — controls what you can do (via Azure RBAC roles assigned in Azure AD)
- **Single Sign-On (SSO)** — one login for Azure, Microsoft 365, and thousands of SaaS applications (Salesforce, GitHub Enterprise, Slack)
- **Application registration** — service accounts for applications (Service Principals) so code can authenticate to Azure resources without using a username/password

**Why it's used in DevOps contexts:**

**1. CI/CD authentication to Azure.**

Instead of long-lived credentials (service account passwords that never rotate), we use a **Service Principal** or **Managed Identity**:

```bash
# Service Principal for CI/CD pipeline
az ad sp create-for-rbac \
  --name "github-actions-deploy" \
  --role "Contributor" \
  --scopes "/subscriptions/xxxxx/resourceGroups/prod-rg"

# Output: appId, password, tenant
# These go into GitHub Secrets — pipeline authenticates to Azure with these credentials
```

Better (zero secrets): **Workload Identity Federation** — GitHub Actions gets a short-lived OIDC token and exchanges it with Azure AD for an access token. No long-lived passwords stored anywhere.

```yaml
# GitHub Actions — OIDC-based Azure login (no secrets)
- uses: azure/login@v2
  with:
    client-id: ${{ vars.AZURE_CLIENT_ID }}
    tenant-id: ${{ vars.AZURE_TENANT_ID }}
    subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
```

**2. AKS and Azure RBAC.**

AKS clusters integrate with Azure AD. Engineers authenticate to the cluster using their Azure AD credentials — no separate kubeconfig with embedded certs:

```bash
az aks get-credentials --resource-group prod-rg --name prod-aks
# kubectl now authenticates via Azure AD — your az login identity is used
```

Azure RBAC roles (like `Azure Kubernetes Service Cluster User` or `Azure Kubernetes Service RBAC Admin`) are assigned in Azure AD. When an engineer's role is removed, their cluster access is revoked automatically — no manual kubeconfig rotation needed.

**3. Managed Identity — no credentials at all for VMs and services.**

A Managed Identity is an identity in Azure AD automatically managed by Azure. A VM with a Managed Identity can authenticate to Azure Key Vault or Azure Storage without any credentials in code:

```python
# In app code on an Azure VM with Managed Identity
from azure.identity import ManagedIdentityCredential
from azure.keyvault.secrets import SecretClient

credential = ManagedIdentityCredential()
client = SecretClient(vault_url="https://my-vault.vault.azure.net/", credential=credential)
secret = client.get_secret("db-password")
# No username, no password, no client ID in the code
```

**4. Conditional Access.**

Azure AD Conditional Access policies can require MFA for admin access from outside the corporate network, block logins from high-risk countries, require compliant devices for production access. We enforce: "all access to prod Azure resources from outside the office network requires MFA." Azure AD enforces this at the identity layer — the application doesn't need to implement it.

**Azure AD vs on-premises Active Directory:**

On-prem AD manages Windows domain users, computers, and Group Policies. Azure AD manages cloud identities, SaaS app access, and OAuth2/OIDC-based authentication. Azure AD Connect syncs on-prem AD users to Azure AD — engineers log in with their corporate Windows credentials to access Azure resources (hybrid identity).

**Real scenario:** We had a CI/CD pipeline using a Service Principal password stored in a secret. The password was rotated every 90 days manually. When someone forgot to update the secret in the pipeline, the pipeline broke at midnight. After migrating to Workload Identity Federation: no passwords, no rotation, no expiry. The pipeline authenticates via OIDC token exchange. Zero secret management overhead, and more secure.
