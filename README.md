# Azure Cloud Infrastructure Project
### VM Provisioning · Networking · Blob Storage

A hands-on Azure project demonstrating core cloud engineering skills — provisioning virtual machines, designing a secure network topology, configuring Network Security Groups, and managing blob storage via Azure CLI.

---

## Architecture Overview

```
Internet
    │
    │ HTTP/HTTPS (80, 443)
    │ SSH (Port 22 — restricted to developer IP only)
    ▼
┌─────────────────────────────────────────────────────────┐
│ Resource Group: rg-webproject (Central India)           │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Virtual Network: vnet-main (10.0.0.0/16)         │   │
│  │                                                  │   │
│  │  ┌─────────────────┐   ┌──────────────────────┐  │   │
│  │  │ subnet-public   │   │ subnet-private        │  │   │
│  │  │ 10.0.1.0/24     │   │ 10.0.2.0/24          │  │   │
│  │  │                 │   │                      │  │   │
│  │  │  ┌───────────┐  │   │  ┌────────────────┐  │  │   │
│  │  │  │  vm-web   │──┼───┼─▶│    vm-app      │  │  │   │
│  │  │  │ Ubuntu LTS│  │   │  │  Ubuntu LTS    │  │  │   │
│  │  │  │Public IP  │  │   │  │ No Public IP   │  │  │   │
│  │  │  └───────────┘  │   │  └────────────────┘  │  │   │
│  │  │  NSG: 22,80,443 │   │  NSG: 22 (internal)  │  │   │
│  │  └─────────────────┘   └──────────────────────┘  │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Storage Account: stwebproject2026                 │   │
│  │  📁 web-assets (HTML files)                      │   │
│  │  📁 $logs     (Analytics logs)                   │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Resources Provisioned

| Resource | Name | Details |
|---|---|---|
| Resource Group | `rg-webproject` | Central India |
| Virtual Network | `vnet-main` | 10.0.0.0/16 |
| Subnet (Public) | `subnet-public` | 10.0.1.0/24 |
| Subnet (Private) | `subnet-private` | 10.0.2.0/24 |
| VM (Web) | `vm-web` | Ubuntu 22.04 LTS, Standard_B2ats_v2, Public IP |
| VM (App) | `vm-app` | Ubuntu 22.04 LTS, Standard_B2ats_v2, No Public IP |
| NSG (Web) | `vm-web-nsg` | Ports 22 (restricted), 80, 443 |
| NSG (App) | `vm-app-nsg` | Port 22 from subnet-public only |
| Storage Account | `stwebproject2026` | Standard LRS, Private blobs |
| Blob Container | `web-assets` | Static file hosting |
| Blob Container | `$logs` | Storage analytics logs |

---

## Security Design

### Network Security Groups (NSGs)

**vm-web-nsg** — Web tier (public-facing):

| Priority | Rule | Port | Source | Action |
|---|---|---|---|---|
| 100 | Allow-HTTP | 80 | Any | Allow |
| 200 | Allow-HTTPS | 443 | Any | Allow |
| 300 | SSH | 22 | Developer IP only | Allow |
| 65500 | DenyAllInBound | Any | Any | Deny |

**vm-app-nsg** — App tier (private):

| Priority | Rule | Port | Source | Action |
|---|---|---|---|---|
| 100 | Allow-SSH-FromWebSubnet | 22 | 10.0.1.0/24 | Allow |
| 65500 | DenyAllInBound | Any | Any | Deny |

### Key Security Decisions

- `vm-app` has **no public IP** — unreachable directly from the internet
- SSH into `vm-app` is only possible by first connecting to `vm-web` (bastion-like pattern)
- SSH on `vm-web` is restricted to a **single developer IP** via NSG rule
- Blob storage has **anonymous access disabled** — all access requires authentication
- Blobs uploaded using **Azure RBAC** (Storage Blob Data Contributor role) and account key authentication
- TLS 1.2 enforced on storage account

> **Production note:** In a production environment, Azure Bastion would replace the public SSH endpoint to eliminate port 22 exposure entirely, and Azure Firewall would be placed at the VNet perimeter for centralised traffic inspection. These were not provisioned here due to cost constraints on a free trial account.

---

## Connectivity Proof

The network topology was validated by establishing an SSH tunnel through the VMs:

```bash
# Step 1 — From developer laptop to vm-web (public subnet)
ssh -i vm-web-key.pem azureuser@<vm-web-public-ip>

# Step 2 — From vm-web to vm-app (private subnet, internal only)
ssh -i ~/.ssh/vm-app-key.pem azureuser@10.0.2.4
```

This confirms:
- `vm-app` is **not reachable directly** from the internet
- Internal VNet routing between subnets works correctly
- NSG rules are enforced as expected

---

## Storage Account — File Upload via Azure CLI

An HTML file was created on `vm-web` and uploaded to Azure Blob Storage using the Azure CLI:

```bash
# Install Azure CLI on vm-web
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Authenticate
az login --use-device-code

# Upload file to blob container
az storage blob upload \
  --account-name stwebproject2026 \
  --account-key "<storage-account-key>" \
  --container-name web-assets \
  --file index.html \
  --name index.html
```

Upload confirmed at **100%** with server-side encryption enabled.

---

## Key Concepts Demonstrated

| Concept | Implementation |
|---|---|
| VM Provisioning | Two Ubuntu VMs with different network tiers |
| VNet Design | /16 address space with /24 public and private subnets |
| Network Security | NSGs with least-privilege inbound rules |
| Defense in Depth | Public VM → Private VM access pattern |
| Azure RBAC | Storage Blob Data Contributor role assignment |
| Blob Storage | Container creation, CLI-based file upload |
| Cost Awareness | VMs deallocated when idle; LRS redundancy chosen |
| Principle of Least Privilege | SSH restricted to single IP; app VM has no public IP |

---

## Tools & Technologies

- **Azure Portal** — Resource provisioning and configuration
- **Azure CLI** — Storage operations and authentication
- **Ubuntu 22.04 LTS** — VM operating system
- **SSH / SCP** — Secure VM access and file transfer
- **PowerShell / Git Bash** — Local terminal for SSH connectivity

---

## Cost Management

All resources were provisioned on an Azure Free Trial account ($200 credits):

- VMs deallocated when not in use (no compute charges while idle)
- Standard_B2ats_v2 chosen as the most cost-effective VM size
- LRS replication selected for storage (lowest redundancy cost)
- Azure Bastion and Azure Firewall deliberately excluded due to high hourly cost (~$140–$900/month respectively)

---

## Author

Uthera J  
Cloud Operations Professional | Chennai, India  
LinkedIn: https://www.linkedin.com/in/uthera-jaganathan/
Github: https://github.com/utherajagan
