# Role-Based Access Control (RBAC)

Notes from:
- Project: [Role Based Access Control](https://csrc.nist.gov/projects/role-based-access-control) — NIST CSRC
- Doc: [What is Azure role-based access control (Azure RBAC)?](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview) — Microsoft Learn
- Doc: [Steps to assign an Azure role](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-steps) — Microsoft Learn
- Doc: [Azure built-in roles for Compute](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/compute) — Microsoft Learn
- Doc: [Microsoft Entra service limits and restrictions (nested groups)](https://learn.microsoft.com/en-us/entra/identity/users/directory-service-limits-restrictions) — Microsoft Learn
- Doc: [Manage users and groups assignment to an application](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/assign-user-or-group-access-portal) — Microsoft Learn
- Doc: [Microsoft Entra licensing (PIM, access reviews)](https://learn.microsoft.com/en-us/entra/fundamentals/licensing) — Microsoft Learn

Related notes: [Microsoft Entra ID](../../identity/entra-id/entra-id.md) ·
[SSO & federation](../../identity/sso-federation/sso-federation.md) ·
[Azure Arc (RBAC for SSH)](../../../cloud/azure/azure-arc/azure-arc.md) ·
[Mesh VPN (group-based network policies)](../../../networking/vpn/mesh-vpn/mesh-vpn.md)

## What it is

**RBAC** grants permissions to **roles**, not to people. People get permissions only by
being assigned a role. When someone changes job, you change their roles; when a role's
needs change, you change the role once instead of editing every user.

```
User ──member of──▶ Group ──assigned──▶ Role ──contains──▶ Permissions ──on──▶ Scope/Resource
```

Formalised by NIST (Ferraiolo & Kuhn, 1992; ANSI/INCITS 359 standard in 2004).

Compared with other models:

| Model | Access decided by | Example |
|---|---|---|
| **DAC** (discretionary) | The owner of the resource | Unix file permissions, sharing a doc |
| **MAC** (mandatory) | Central labels / clearance levels | SELinux, military classification |
| **RBAC** | The role(s) the subject holds | "DB admins can restart the DB" |
| **ABAC** (attribute-based) | Rules over attributes (user, resource, environment) | "Engineering users, on managed devices, during work hours" |

In practice, systems mix them: RBAC for the bulk, ABAC-style conditions on top (e.g.
Conditional Access in [Entra ID](../../identity/entra-id/entra-id.md)).

## Principles

- **Least privilege** — each role gets only what the job needs; default is *deny*.
- **Separation of duties** — no single role can do a sensitive thing end to end (e.g.
  the person who requests a payment isn't the one who approves it; the person who
  manages a VM isn't automatically allowed to log in to it).
- **Assign to groups, not individuals** — a user-by-user assignment is invisible
  technical debt; a group assignment is auditable ("who is in `PRD-DB-ADMINS`?").
- **Separate environments** — lab/non-prod and production get different groups, so
  broad lab rights never leak into production.
- **Temporary where possible** — privileged roles are activated just-in-time and
  expire; contractor access has an end date.
- **Review periodically** — memberships drift; someone must re-confirm them.

## Where roles live: the IdP vs the app

A common, pragmatic split in SSO setups:

| Layer | Holds | Example |
|---|---|---|
| **Identity provider** | *Who you are* and *which groups you're in* | Entra ID / Okta groups |
| **Each application** | *What each group may do there* (fine-grained permissions) | The app maps group `PRD-GITLAB-DEVELOPERS` → its own "Developer" role |

The IdP sends group membership in the token (see
[groups in tokens](../../identity/sso-federation/sso-federation.md#groups-in-tokens));
the app keeps its native role model. This avoids trying to express every app's
permission model in the IdP, at the cost of configuring the mapping in each app.

## Group naming conventions

A predictable name tells you, at a glance, **what** a group grants **where**. A widely
used pattern:

```
<ENV>-<SERVICE>-<ROLE>
```

| Part | Values (examples) | Purpose |
|---|---|---|
| `ENV` | `LAB`, `DEV`, `STG`, `PRD` | Environment separation |
| `SERVICE` | `LINUX`, `GITLAB`, `VAULT`, `VPN`, `K8S` | The system the group grants access to |
| `ROLE` | `ADMINS`, `DEVELOPERS`, `USERS`, `READERS` | The level of access within that system |

Examples: `LAB-LINUX-ADMINS`, `PRD-GITLAB-DEVELOPERS`, `PRD-VAULT-USERS`.

Tips:
- Pick singular or plural once and never mix (`ADMIN` vs `ADMINS`).
- Upper case and hyphens avoid case-sensitivity and escaping issues across tools.
- Add a prefix per *kind* of group if you have several (`APP-`, `ROLE-`, `LIC-` for
  licensing groups) so filters like "all groups starting with `PRD-`" work in token
  claim filters and scripts.

### Profile groups vs technical groups

With many services, each person ends up in dozens of `<ENV>-<SERVICE>-<ROLE>` groups.
A second layer of **profile groups** (by job) helps:

```
ROLE-DEVELOPERS ──nested into──▶ PRD-GITLAB-DEVELOPERS
                               ▶ LAB-LINUX-USERS
                               ▶ PRD-VPN-USERS
```

Onboard a developer = add them to one group. **But nesting only works if every
consumer expands nested memberships** — and they often don't. In Entra ID:

| Consumer | Follows nesting? |
|---|---|
| Group assigned to an enterprise app | ❌ direct members only |
| Group claims in tokens | ✅ |
| Group-scoped provisioning (SCIM) | ❌ |
| Azure RBAC role assignment | ✅ transitive |

So: check each target before relying on nesting, or use a tool that *expands* profile
groups into direct memberships.

## The access matrix

The design artefact for RBAC is a matrix, reviewed with the business:

| Profile | Group | System | Role in the system |
|---|---|---|---|
| Developer | `PRD-GITLAB-DEVELOPERS` | GitLab | Developer |
| Developer | `LAB-LINUX-USERS` | Lab Linux servers | Login, no sudo |
| Sysadmin | `LAB-LINUX-ADMINS` | Lab Linux servers | Login + sudo |
| Contractor | `PRD-VPN-CONTRACTORS` | VPN | Access to one subnet, time-limited |

Read it both ways: rows answer "what can this group do?", filtering by profile answers
"what does a new hire in this job need?". It goes naturally into an architecture
document ([HLD](../../../system-design/hld/hld.md)).

## Worked example: Azure RBAC

Azure's authorization system for Azure resources (separate from Entra *directory*
roles, which govern the identity tenant itself).

A **role assignment** = **who** (security principal) + **what** (role definition) +
**where** (scope).

- **Security principal**: user, group, service principal, or managed identity.
- **Role definition**: a list of allowed operations.
  - `actions` — control plane (manage the resource: create, delete, configure).
  - `dataActions` — data plane (use what's inside: read blobs, log into a VM).
- **Scope** — inherited downwards:

```
Management group
└── Subscription
    └── Resource group
        └── Resource
```

Assign at the highest scope that is still correct (e.g. a resource group per
environment), not per resource: there's a limit of **5,000 role assignments per
subscription**, and per-resource assignments are hard to audit.

Separation of duties in action — logging into a VM (or an Arc-enabled server) is a
**data action**, so:

| Role | Can manage the VM | Can log in | sudo |
|---|:-:|:-:|:-:|
| Owner / Contributor | ✅ | ❌ | — |
| Virtual Machine Administrator Login | ❌ | ✅ | ✅ |
| Virtual Machine User Login | ❌ | ✅ | ❌ |

Group role assignments are **transitive** (nested members inherit), and Azure RBAC
itself is free with the subscription. More in [Azure Arc](../../../cloud/azure/azure-arc/azure-arc.md).

## Just-in-time and governance

Standing admin rights are the biggest RBAC risk. Tools to reduce them (Entra names;
equivalents exist elsewhere):

| Mechanism | What it does | Entra tier |
|---|---|---|
| **PIM** (Privileged Identity Management) | Role is *eligible*; user activates it for N hours with justification/approval | P2 / ID Governance |
| **PIM for Groups** | Same, for membership of a privileged group | P2 |
| **Access reviews** | Periodic re-certification of memberships | P2 (advanced: ID Governance) |
| **Entitlement management** | Requestable access packages with approval and **expiry** | P2 (advanced: ID Governance) |

Without these (e.g. on a free tier), the same principles hold but are manual: a
calendar reminder to review groups, and removing contractors by hand at the end of
their engagement.

## Quiz

Self-test on this material: **[Identity & Access Quiz](https://claude.ai/artifact/31uVhRCfgxsVFmxL2jngyj)**
— draws 10 random questions from a pool of 30 each time, so it's reusable. Shared with the sibling notes: covers Entra building blocks, tiers and Conditional Access, SAML vs OIDC, claims and SCIM, RBAC models, and group naming and nesting.
Supports skip / previous / jump-to-question navigation, and keeps your progress in the browser if you refresh.
