# Azure Arc-enabled Servers & Entra ID SSH Login for Linux

Notes from:
- Doc: [Azure Arc-enabled servers overview](https://learn.microsoft.com/en-us/azure/azure-arc/servers/overview) — Microsoft Learn
- Doc: [Connected Machine agent overview](https://learn.microsoft.com/en-us/azure/azure-arc/servers/agent-overview) — Microsoft Learn
- Doc: [Connected Machine agent prerequisites](https://learn.microsoft.com/en-us/azure/azure-arc/servers/prerequisites) — Microsoft Learn
- Doc: [Connected Machine agent network requirements](https://learn.microsoft.com/en-us/azure/azure-arc/servers/network-requirements) — Microsoft Learn
- Doc: [Manage agent proxy settings](https://learn.microsoft.com/en-us/azure/azure-arc/servers/manage-agent-proxy-settings) — Microsoft Learn
- Doc: [SSH access to Azure Arc-enabled servers](https://learn.microsoft.com/en-us/azure/azure-arc/servers/ssh-arc-overview) — Microsoft Learn
- Doc: [Troubleshoot SSH access to Arc-enabled servers](https://learn.microsoft.com/en-us/azure/azure-arc/servers/ssh-arc-troubleshoot) — Microsoft Learn
- Doc: [Azure Arc-enabled servers security overview](https://learn.microsoft.com/en-us/azure/azure-arc/servers/security-overview) — Microsoft Learn
- Doc: [VM extension management with Arc-enabled servers](https://learn.microsoft.com/en-us/azure/azure-arc/servers/manage-vm-extensions) — Microsoft Learn
- Doc: [Sign in to a Linux VM in Azure using Microsoft Entra ID and OpenSSH](https://learn.microsoft.com/en-us/entra/identity/devices/howto-vm-sign-in-azure-ad-linux) — Microsoft Learn
- Doc: [az ssh (Azure CLI reference)](https://learn.microsoft.com/en-us/cli/azure/ssh?view=azure-cli-latest) — Microsoft Learn
- Doc: [Azure built-in roles for Compute](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/compute) — Microsoft Learn
- Doc: [What is Azure RBAC?](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview) — Microsoft Learn
- Page: [Azure Arc pricing](https://azure.microsoft.com/en-us/pricing/details/azure-arc/core-control-plane/) — Microsoft Azure

Related notes: [Microsoft Entra ID](../../../security/identity/entra-id/entra-id.md) ·
[RBAC (Azure RBAC section)](../../../security/access-control/rbac/rbac.md#worked-example-azure-rbac) ·
[Mesh VPN](../../../networking/vpn/mesh-vpn/mesh-vpn.md)

## What it is

**Azure Arc** extends Azure's control plane to things that are **not** running in
Azure: on-prem servers, VMs in other clouds, Kubernetes clusters, databases.
**Arc-enabled servers** are physical or virtual Windows/Linux machines onto which you
install the **Azure Connected Machine agent**. The machine then appears in Azure as a
resource (`Microsoft.HybridCompute/machines`) in a resource group, with:

- a **managed identity** (the machine can authenticate to Azure services),
- **VM extensions** (install software remotely, e.g. the Entra SSH login extension),
- Azure RBAC, Azure Policy, tags, inventory — like a native Azure VM.

Why it matters for identity: it's the mechanism that lets you log into a **non-Azure
Linux server with your Entra ID account**, with access controlled by Entra groups and
Azure RBAC instead of per-server `authorized_keys` files.

> Prerequisite that surprises people: Arc needs an **Azure subscription**. The resource
> providers (`Microsoft.HybridCompute`, `Microsoft.GuestConfiguration`,
> `Microsoft.HybridConnectivity`) are registered in it and the machines live in its
> resource groups. An Entra tenant alone isn't enough.

## Architecture at a glance

```
         Admin laptop                         Azure                          Server (on-prem / LAB)
 ┌──────────────────────┐       ┌───────────────────────────────┐      ┌─────────────────────────────┐
 │ az login  (Entra)    │──────▶│ Entra ID: auth + MFA          │      │ Connected Machine agent     │
 │ az ssh arc / az ssh  │       │ Azure RBAC: login roles       │◀─────│  (himds, outbound 443 only) │
 │   cert → OpenSSH     │──┐    │ Arc control plane (ARM)       │      │ AADSSHLoginForLinux ext.    │
 └──────────────────────┘  │    │ Relay (Azure Service Bus)     │◀─────│  (PAM/NSS: aadsshlogin)     │
                           └───▶│                               │─────▶│ sshd                        │
                                └───────────────────────────────┘      └─────────────────────────────┘
```

**Outbound only**: the server needs no public IP and no inbound port — it opens outbound
HTTPS connections to Azure, and SSH through Arc travels over a relay the server itself
connected to.

## Network requirements

Everything is **outbound TCP 443** (TLS 1.2/1.3):

| Destination | Purpose |
|---|---|
| `packages.microsoft.com` | Agent and `aadsshlogin` packages, updates |
| `login.microsoftonline.com`, `*.login.microsoft.com` | Entra ID: agent auth, and PAM-based Entra SSH auth |
| `pas.windows.net` | Azure RBAC check at SSH login time |
| `management.azure.com` | Azure Resource Manager (connect/disconnect) |
| `*.his.arc.azure.com` | Metadata and hybrid identity (managed identity) |
| `*.guestconfiguration.azure.com` | Extension management |
| `guestnotificationservice.azure.com`, `*.guestnotificationservice.azure.com` | Notifications |
| `*.servicebus.windows.net` | Notifications and the **SSH relay** |

- For IP-based firewalls, Microsoft publishes **service tags** instead of IP lists:
  `AzureActiveDirectory`, `AzureTrafficManager`, `AzureResourceManager`,
  `AzureArcInfrastructure`, `Storage`, `AzureFrontDoor.Frontend`.
- An **HTTP proxy** is supported by the agent (`azcmagent config set proxy.url …`),
  with per-service bypass. Some extensions don't inherit the agent's proxy; for the
  Entra SSH extension on Arc, proxy behaviour is not documented.
- The **Arc gateway** can reduce the number of endpoints to allow.
- Traffic is light: one persistent WebSocket, heartbeat every 5 minutes.

## Entra ID SSH login

Installing the **`AADSSHLoginForLinux`** extension (packages `aadsshlogin*`) hooks
Entra into the server's **PAM/NSS** stack. The user doesn't use a password or a
long-lived key: the Azure CLI gets a **short-lived OpenSSH certificate signed by
Entra**, and sshd accepts it after checking Azure RBAC.

Prerequisites (Linux): supported distro (e.g. Ubuntu 16.04 → 24.04; full list in the
Entra Linux sign-in doc), `sshd` running, agent ≥ 1.31 for SSH via Arc. The docs state architectures
for the agent (x86-64, ARM64 with some features missing) but **not** for the SSH
extension.

### Who can log in: Azure RBAC

Access is **not** given by Owner/Contributor on the machine — that's an "intentional
(and audited) separation" between managing and logging in. Instead, dedicated roles
using `dataActions`:

| Built-in role | Effect on the Linux box |
|---|---|
| **Virtual Machine Administrator Login** | Log in as your Entra user **with passwordless `sudo`** |
| **Virtual Machine User Login** | Log in as your Entra user, **no sudo** |
| **Virtual Machine Local User Login** | Connect **through the Arc relay** as an existing **local** Linux user (key/password) |

- Assign to **Entra groups** at management group / subscription / **resource group**
  scope, not per machine. Nested group members inherit (Azure RBAC is transitive).
- Only two privilege levels from Azure (sudo or not); finer `sudoers` rules for Entra
  users aren't documented.
- Assigning these roles requires `Microsoft.Authorization/roleAssignments/write`
  (e.g. *Role Based Access Control Administrator* or *Virtual Machine Data Access
  Administrator*). Onboarding a machine requires *Azure Connected Machine Onboarding*
  or Contributor on the resource group.

### How to connect

| Path | Command | Needs |
|---|---|---|
| **Through Azure (Arc relay)** | `az ssh arc --resource-group RG --name SERVER` | Azure CLI + `ssh` extension, `az login`; server reaches `*.servicebus.windows.net`. No inbound port, no VPN |
| **Direct** (LAN, VPN, [mesh VPN](../../../networking/vpn/mesh-vpn/mesh-vpn.md)) | `az ssh cert` or `az ssh config --ip …`, then plain `ssh` | Network reachability to the server. Documented for Azure VMs; for Arc servers it isn't explicitly documented |
| **Local user** | `az ssh arc … --local-user admin` or plain `ssh admin@server` | Relay: *VM Local User Login* role. Direct: nothing from Azure |

`az ssh config` writes an OpenSSH config file so VS Code Remote-SSH, `scp`, `rsync`
etc. work transparently.

### MFA and policies

- Tier-independent: on Entra Free, **security defaults** force MFA for Azure CLI /
  ARM sign-ins, which is how the admin gets the SSH certificate. (Whether that MFA
  specifically covers the certificate request isn't stated.) Security defaults also
  block device-code login, so `az login` uses the browser flow.
- **Conditional Access** (P1) can target the *Azure Linux VM Sign-In* app: MFA,
  location, compliant device (device compliance only works with Azure CLI on
  Windows/macOS).
- **PIM** (P2/Governance) can make the login roles just-in-time.

## When things are down

| Situation | Behaviour |
|---|---|
| Server can't reach Entra / `pas.windows.net` | Entra login fails. **No offline or cached-credential mode is documented** |
| Admin can't reach Entra | No new certificate → no Entra login |
| Azure or Internet down | Arc relay path unavailable |
| Agent offline 15–30 min | Resource shows **Disconnected** |
| Agent offline > 45 days | May become **Expired** (managed identity credential valid 90 days, renewed every 45) → disconnect and reconnect |
| Azure region outage | Automatic failover within the geography; agent shows Disconnected meanwhile |
| Local users | Keep working with plain `ssh` (they don't depend on Azure) |

Takeaway: keep **local admin accounts with keys stored outside the IdP** as the
fallback — Arc + Entra is a convenience and audit layer, not the only door.

## Costs

| Item | Cost |
|---|---|
| Arc core control plane: inventory, **SSH via Arc**, Run Command, Custom Script Extension | Free |
| Azure RBAC | Free with the subscription |
| Entra tier for SSH login + group role assignment | Free is enough (CA = P1, PIM = P2 optional) |
| Entra SSH login extension | Not listed on the pricing page; no charge found |
| Optional add-ons: Azure Policy guest configuration (~$6/server/month), Update Manager, Defender for Servers, Azure Monitor, Sentinel | Charged per server when enabled |
| The Azure subscription itself | Required; pay-as-you-go bills consumption only |

Checked Sept 2026 — recheck the pricing page.

## Gotchas

- `AllowGroups` / `DenyGroups` in `sshd_config` make the **first** Entra login fail.
  Remove them or confine them to a `Match` block that excludes Entra users.
- **Cloned machines / golden images** already onboarded share an identity — re-onboard
  each clone.
- Remote access through Arc can be **disabled per server**
  (`azcmagent config set incomingconnections.enabled false`), and extension allowlists
  are recommended to limit what Azure can install.
- Entra sign-in logs are kept 7 days on Free (30 on P1/P2); the server's own
  `/var/log/auth.log` remains the local audit trail.
- Undocumented: Entra user naming/UID mapping on the server, clashes with existing local
  users of the same name, certificate lifetime.

## Quiz

Self-test on this material: **[Infra Access Quiz](https://claude.ai/artifact/XUcm5e6T6dHWsueoa9Txbp)**
— draws 10 random questions from a pool of 30 each time, so it's reusable. Shared with the sibling note: covers the Arc agent, outbound 443 flows, Entra SSH roles, outage behaviour, WireGuard, NetBird components, and groups and policies.
Supports skip / previous / jump-to-question navigation, and keeps your progress in the browser if you refresh.
