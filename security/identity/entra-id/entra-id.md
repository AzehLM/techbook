# Microsoft Entra ID

Notes from:
- Doc: [What is Microsoft Entra ID?](https://learn.microsoft.com/en-us/entra/fundamentals/whatis) — Microsoft Learn
- Doc: [Microsoft Entra licensing](https://learn.microsoft.com/en-us/entra/fundamentals/licensing) — Microsoft Learn
- Page: [Microsoft Entra plans and pricing](https://www.microsoft.com/en-us/security/business/microsoft-entra-pricing) — Microsoft
- Doc: [Security defaults](https://learn.microsoft.com/en-us/entra/fundamentals/security-defaults) — Microsoft Learn
- Doc: [MFA versions and consumption plans](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-mfa-licensing) — Microsoft Learn
- Doc: [Conditional Access overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview) — Microsoft Learn
- Doc: [Manage users and groups assignment to an application](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/assign-user-or-group-access-portal) — Microsoft Learn
- Doc: [Manage access to apps](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/what-is-access-management) — Microsoft Learn
- Doc: [Rules for dynamic membership groups](https://learn.microsoft.com/en-us/entra/identity/users/groups-dynamic-membership) — Microsoft Learn
- Doc: [Set up self-service group management](https://learn.microsoft.com/en-us/entra/identity/users/groups-self-service-management) — Microsoft Learn
- Doc: [Microsoft Entra service limits and restrictions](https://learn.microsoft.com/en-us/entra/identity/users/directory-service-limits-restrictions) — Microsoft Learn
- Doc: [License self-service password reset](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-sspr-licensing) — Microsoft Learn
- Doc: [Microsoft Entra data retention](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/reference-reports-data-retention) — Microsoft Learn
- Doc: [External ID pricing](https://learn.microsoft.com/en-us/entra/external-id/external-identities-pricing) — Microsoft Learn

Related notes: [SSO & federation](../sso-federation/sso-federation.md) ·
[RBAC](../../access-control/rbac/rbac.md) ·
[Azure Arc (Entra SSH to Linux)](../../../cloud/azure/azure-arc/azure-arc.md)

## What it is

**Microsoft Entra ID** (formerly *Azure Active Directory*, "Azure AD") is Microsoft's
cloud **identity and access management (IAM)** service. It is the directory behind
every Microsoft 365 and Azure tenant: it holds users and groups, authenticates them
(password, MFA, passwordless), and issues tokens so they can sign in to apps —
Microsoft's own and any third-party SaaS that speaks SAML or OpenID Connect.

In SSO vocabulary, Entra ID is the **identity provider (IdP)**: apps trust it to say
"this is Alice, she passed MFA, she is in groups X and Y" and decide access from that
(see [SSO & federation](../sso-federation/sso-federation.md)).

> Despite the old name, Entra ID is **not** cloud-hosted Active Directory Domain
> Services: no LDAP, no Kerberos, no GPOs, no domain join in the classic sense. It is a
> web-protocol IdP (SAML, OIDC, OAuth 2.0). Hybrid setups sync on-prem AD into it.

## Core building blocks

| Object | What it is | Notes |
|---|---|---|
| **Tenant** | One dedicated instance of Entra ID for an organisation | Created with the first Microsoft 365 / Azure subscription; has a `*.onmicrosoft.com` domain + your verified domains |
| **User (member)** | An account that belongs to the tenant | Your employees |
| **Guest (B2B)** | An external account invited into the tenant | Signs in with their own home identity; billed per monthly active user (MAU), see below |
| **Group** | A collection of users (and/or other groups, devices, service principals) | Security groups for access; Microsoft 365 groups for collaboration |
| **App registration** | The *definition* of an app that uses Entra for sign-in (client ID, redirect URIs, secrets/certs) | What a developer creates |
| **Enterprise application** (service principal) | The *instance* of an app in your tenant: who can use it, SSO settings, provisioning | What an admin configures for a SaaS |
| **Directory roles** | Admin roles over Entra itself (Global Administrator, User Administrator, Application Administrator…) | Different from Azure RBAC roles over Azure resources — see [RBAC](../../access-control/rbac/rbac.md) |
| **Administrative unit** | A container to scope admin roles to a subset of users/groups | Creating AUs is free; AU-scoped admins need P1 |

### Groups: membership types

| Type | How members get in | Licence |
|---|---|---|
| **Assigned** | An admin (or group owner) adds them by hand | Free |
| **Dynamic** | A rule on user attributes, e.g. `user.department -eq "Engineering"` | P1 for every user who ends up in a dynamic group |
| **Self-service** | Users create groups; owners manage members | Free. "Request to join" + owner approval needs P1 |

**Nested groups** (a group inside a group) behave differently depending on *who reads
the membership* — a frequent source of surprises:

| Consumer | Sees nested members? |
|---|---|
| Assigning a group to an enterprise app | **No** — only direct members get access |
| Group claims in SAML/OIDC tokens | **Yes** (transitive) |
| Conditional Access user/group scope | **Yes** |
| Automatic provisioning (SCIM) scoped by group | **No** |
| Azure RBAC role assignments | **Yes** ("role assignments are transitive for groups") |

## Giving users access to an app

Once an enterprise app is configured for SSO, two settings decide *who* can sign in:

- **Assignment required?**
  - **Yes** → only users assigned to the app (directly, or through a group) can sign
    in. Everyone else is refused by Entra before reaching the app.
  - **No** → any user of the tenant can sign in; the app itself must decide what they
    can do (often from group claims in the token).
- **Assign users or groups** — assigning **users** one by one works on every tier;
  assigning **groups** requires **P1**.

So on the Free tier, a common pattern is: *assignment not required + the app maps
group claims to roles and refuses users with no mapped group*. On P1, you assign
groups to the app and let Entra do the gatekeeping. Details on claims in
[SSO & federation](../sso-federation/sso-federation.md#groups-in-tokens).

## Authentication and MFA

### Security defaults (the Free baseline)

A single on/off switch, free on every tenant (on by default for new tenants):

- Everyone registers for MFA (Microsoft Authenticator, or any OATH TOTP app).
- MFA is required **always** for admin roles and for Azure management (portal, CLI,
  PowerShell, ARM API); for other users "when necessary" — Microsoft decides.
- **Legacy authentication** (IMAP/POP/SMTP basic auth…) is blocked.
- **Device code flow** is blocked.
- **No exclusions** and no per-app targeting: it's all or nothing.

### Conditional Access (P1)

The policy engine that replaces security defaults once you need nuance. A policy reads
as an **if → then**:

```
IF   user/group ∈ {…}  AND app ∈ {…}  AND conditions (location/IP, device, platform, risk…)
THEN block | grant, requiring {MFA, compliant device, terms of use…}
```

Examples: require MFA only for admin portals; allow SSH sign-in only from known IPs;
exclude emergency-access accounts from a policy. Risk-based conditions (risky user /
risky sign-in) need **P2**. Requiring a *compliant* device also needs an Intune licence.

> If CA licences lapse, existing policies stay in force but can no longer be edited
> (only viewed or deleted).

### Passwords

| Capability | Free | P1 |
|---|---|---|
| User **changes** their own password (knows it) | ✅ | ✅ |
| Self-service password **reset** (SSPR, forgot it) | ❌ admin must reset | ✅ (also in M365 Business Standard/Premium) |

## Privileged access and governance (P2 and up)

- **PIM — Privileged Identity Management**: roles are *eligible* rather than permanent;
  the admin *activates* a role for a limited time (just-in-time), optionally with
  approval and justification. Works for Entra roles, Azure roles, and group membership.
- **Access reviews**: periodic "does this person still need this?" campaigns.
- **Entitlement management**: *access packages* bundling groups/apps/roles, with
  request/approval flows and **expiry** — the built-in answer to "temporary,
  revocable access for contractors."
- **ID Protection**: detects leaked credentials and risky sign-ins, feeds risk-based CA.
- Full joiner/mover/leaver automation (*Lifecycle Workflows*) needs **ID Governance**.

## Which tier unlocks what

Tiers: **Free** (included with any Microsoft cloud subscription), **P1**, **P2**; plus
add-ons (**ID Governance**, **Entra Suite**). P1 is bundled in Microsoft 365 Business
Premium and E3; P2 in E5. List prices at the time of writing (Sept 2026, USD, annual
commitment): P1 $7, P2 $10 per user/month — always recheck the pricing page.

| Feature | Free | P1 | P2 |
|---|:-:|:-:|:-:|
| SSO to any SaaS app (SAML / OIDC), unlimited apps | ✅ | ✅ | ✅ |
| Assign **users** to an app | ✅ | ✅ | ✅ |
| Assign **groups** to an app | ❌ | ✅ | ✅ |
| Automatic **user** provisioning (SCIM) | ✅ | ✅ | ✅ |
| Automatic **group** provisioning, provisioning logs | ❌ | ✅ | ✅ |
| Dynamic groups | ❌ | ✅ | ✅ |
| MFA via security defaults | ✅ | — | — |
| Conditional Access | ❌ | ✅ | ✅ |
| Risk-based Conditional Access, ID Protection | ❌ | ❌ | ✅ |
| Self-service password reset | ❌ | ✅ | ✅ |
| Custom directory roles | ❌ | ✅ | ✅ |
| PIM, access reviews, entitlement management | ❌ | ❌ | ✅ (full set: ID Governance) |
| Sign-in & audit log retention in the portal | 7 days | 30 days | 30 days |

Mnemonic: **Free = SSO + security defaults · P1 = Groups + Conditions · P2 = Privilege + Protection.**

Things the docs leave unclear (verify before relying on them):
- Whether **group claims** in tokens work on a Free-only tenant: the group-claims page
  sets no licence requirement, but doesn't say "Free" explicitly either.
- Whether **per-user MFA** (the legacy per-account toggle) exists on a Free-only
  tenant: the docs tie it to "Office 365 licensing."
- Whether *every* user benefiting from a premium feature needs a licence: only
  per-feature rules are written down (e.g. dynamic groups, SSPR, PIM, custom roles).

### Guests

B2B guests are billed per **monthly active user (MAU)**: the first 50,000 MAU are free
and can use P1/P2 features (MFA, CA…) without per-seat licences. The tenant must be
linked to an Azure subscription for MAU billing. Governance features for guests are a
separate paid MAU add-on.

## Logs

Sign-in logs and audit logs are the primary trail of "who signed in where, who changed
what." The portal keeps them **7 days on Free, 30 days on P1/P2**, and upgrading does
not bring back expired data. For longer retention, export them (diagnostic settings) to
a storage account, Log Analytics, or a SIEM — which are paid Azure resources.

## Quiz

Self-test on this material: **[Identity & Access Quiz](https://claude.ai/artifact/31uVhRCfgxsVFmxL2jngyj)**
— draws 10 random questions from a pool of 30 each time, so it's reusable. Shared with the sibling notes: covers Entra building blocks, tiers and Conditional Access, SAML vs OIDC, claims and SCIM, RBAC models, and group naming and nesting.
Supports skip / previous / jump-to-question navigation, and keeps your progress in the browser if you refresh.
