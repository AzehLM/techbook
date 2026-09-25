# Mesh / Overlay VPNs (NetBird as the example)

Notes from:
- Site: [WireGuard: fast, modern, secure VPN tunnel](https://www.wireguard.com/) — WireGuard project
- Paper: [WireGuard: Next Generation Kernel Network Tunnel](https://www.wireguard.com/papers/wireguard.pdf) — J. A. Donenfeld (NDSS 2017)
- Doc: [How NetBird works](https://docs.netbird.io/about-netbird/how-netbird-works) — NetBird Docs
- Doc: [Managing access with NetBird: groups and access policies](https://docs.netbird.io/manage/access-control/manage-network-access) — NetBird Docs
- Doc: [Provision users and groups from your identity provider](https://docs.netbird.io/manage/team/idp-sync) — NetBird Docs
- Doc: [Provision users and groups from Microsoft Entra ID via SCIM](https://docs.netbird.io/manage/team/idp-sync/microsoft-entra-id-scim-sync) — NetBird Docs
- Doc: [Peer session expiration](https://docs.netbird.io/manage/settings/enforce-periodic-user-authentication) — NetBird Docs
- Spec: [RFC 8445 — Interactive Connectivity Establishment (ICE)](https://datatracker.ietf.org/doc/html/rfc8445) — IETF
- Spec: [RFC 6598 — IANA-Reserved IPv4 Prefix for Shared Address Space (100.64.0.0/10)](https://datatracker.ietf.org/doc/html/rfc6598) — IETF

Related notes: [OSI model](../../osi/osi-model.md) ·
[SSO & federation](../../../security/identity/sso-federation/sso-federation.md) ·
[RBAC](../../../security/access-control/rbac/rbac.md) ·
[Azure Arc (SSH over a relay vs direct)](../../../cloud/azure/azure-arc/azure-arc.md)

## What it is

A **mesh (overlay) VPN** builds a private network *on top of* existing networks
(home Wi-Fi, office LAN, cloud VPCs, 4G), where **every device connects directly to
every other device it's allowed to reach** — no central VPN concentrator carrying all
the traffic.

| | Classic hub-and-spoke VPN | Mesh / overlay VPN |
|---|---|---|
| Topology | All clients tunnel into one gateway | Peer-to-peer tunnels between devices |
| Data path | Everything hairpins through the hub | Direct when possible, relay as fallback |
| Access granularity | Usually "you're on the network" (subnet-level) | Per-device, per-port policies |
| Identity | VPN account / certificate | Usually SSO with the company IdP |
| Firewall needs | Open an inbound port on the gateway | Typically outbound only (NAT traversal) |
| Examples | OpenVPN server, IPsec site-to-site, WireGuard hub | NetBird, Tailscale, ZeroTier, Nebula |

It's the network half of a **zero-trust** approach: network location grants nothing by
itself; each connection is allowed because *this identity, on this device, matches a
policy for this destination*.

## The building block: WireGuard

Most modern mesh VPNs (NetBird, Tailscale) use **WireGuard** tunnels underneath.

- A small kernel-level VPN protocol that **encapsulates IP packets in UDP** — a layer-3
  tunnel in [OSI](../../osi/osi-model.md) terms, carried over layer 4 (UDP).
- Each peer has a key pair (**Curve25519**); config is "my private key + list of peers'
  public keys," similar to SSH keys.
- Fixed modern crypto, no negotiation: Noise protocol framework, ChaCha20-Poly1305,
  BLAKE2s, SipHash24, HKDF.
- **Cryptokey routing**: each peer's public key is bound to its *allowed IPs*. Outbound,
  that's the routing table (which peer gets this packet); inbound, it's an ACL (this
  packet claims source X — was it encrypted by X's key?).
- **Silent** when idle and roams transparently when a peer's IP changes.

What WireGuard does **not** do: discover peers, distribute keys, traverse NAT, or know
about users. That's what the mesh VPN's control plane adds.

## How NetBird works

```
                 ┌──────────────────── control plane ────────────────────┐
                 │  Management service   Signal service   Relay (TURN)   │
                 └──────▲──────────────────────▲──────────────────▲─────┘
                        │ network map,         │ exchange         │ fallback when
                        │ policies, keys       │ ICE candidates   │ direct fails
             ┌──────────┴──┐              ┌────┴────────┐         │
             │  Peer A     │◀══ WireGuard (direct, P2P) ══▶ Peer B │
             │  laptop     │              │  server     │─────────┘
             └─────────────┘              └─────────────┘
```

| Component | Role | Sees your traffic? |
|---|---|---|
| **Client (agent)** | Runs on each peer; generates the WireGuard key pair ("the private key … never leaves the machine"), authenticates, applies firewall/DNS rules locally | It's the endpoint |
| **Management service** | Source of truth: peers, public keys, groups, policies, DNS; assigns overlay IPs from **100.64.0.0/10** (the CGNAT shared range); pushes each peer its *network map* | No |
| **Signal service** | Lets peers find each other and exchange connection candidates (end-to-end encrypted); stores nothing | No |
| **Relay (TURN)** | Forwards packets when a direct path is impossible (symmetric NAT, strict firewalls) | Encrypted packets only — it can't decrypt WireGuard |

Peers use **ICE** (with STUN to discover their public address) to punch through NAT and
connect directly; only if that fails does traffic go through the relay.

NetBird comes as a managed **cloud** service or **self-hosted** (open source).

## Identity: SSO login and IdP sync

- **SSO login**: users add a device by signing in through the IdP (Entra ID, Okta,
  Google, Keycloak…) via OIDC — see [SSO & federation](../../../security/identity/sso-federation/sso-federation.md).
  Servers are usually enrolled with a **setup key** instead (no human login).
- **IdP sync**: users and groups are provisioned from the IdP — for Entra ID either by
  polling the Graph API (~every 5 min) or by **SCIM** push. When a user is removed in
  the IdP, their network access is revoked. On NetBird Cloud this needs the *Team* plan
  or above.
- **Session expiration**: SSO-added peers must re-authenticate periodically (default
  24 h, configurable 1 h – 180 days). Optionally also after 10 minutes disconnected
  from the management service. Setup-key peers are not affected.
- **User approval**: new users joining via domain match can be held as "pending" until
  an admin approves them.

## Access control: groups and policies

- **Groups** contain peers; a peer can be in several groups. A built-in **`All`** group
  contains every peer. Groups synced from the IdP (e.g. `PRD-VPN-USERS`) can drive
  policies directly — see the naming conventions in [RBAC](../../../security/access-control/rbac/rbac.md#group-naming-conventions).
- **Policies**: *source group → destination group*, with protocol, ports/ranges, and
  direction (bidirectional or source-initiates-only).
- **Default policy**: a new account starts with `All → All` — a full mesh where
  everything talks to everything. **Delete it** and write explicit policies (default
  deny).
- **Posture checks** add device conditions (client version, OS, location, running
  process…) before a policy applies.
- Policies are enforced **on the peers** (the management service distributes the
  resulting network map), not on a central firewall.

Example:

| Source group | Destination group | Protocol/port | Meaning |
|---|---|---|---|
| `LAB-LINUX-ADMINS` | `lab-servers` | TCP 22 | Admins can SSH to lab servers |
| `PRD-MONITORING` | `prod-servers` | TCP 9100 | Monitoring scrapes node exporters |
| `CONTRACTORS` | `staging-web` | TCP 443 | Contractors reach only the staging web app |

## Mesh VPN + SSO SSH: layered access

Mesh VPNs pair well with identity-based SSH (e.g. [Entra SSH via Azure Arc](../../../cloud/azure/azure-arc/azure-arc.md)):

```
User ──SSO──▶ IdP ──▶ mesh VPN (network policy: may reach server:22)
                              ──▶ SSH to server (IdP-issued certificate + RBAC: may log in / sudo)
```

Two independent checks, both driven by IdP groups: *can reach* (network) and *can log
in* (host). Offboarding the user in the IdP closes both.

## Failure modes

Inferred from the architecture above. NetBird's docs don't describe outage behaviour,
so verify each row in a test.

| What fails | Likely effect |
|---|---|
| IdP down | No new SSO logins or re-authentications; peers already authenticated keep working until their session expires |
| Management service down | No new peers, no policy updates. **Whether established tunnels keep working is not clearly documented** — test it |
| Signal service down | New peer-to-peer connections can't be negotiated |
| Relay down | Only peers that needed the relay lose connectivity |
| Internet lost at a site | Peers on the same LAN may still connect directly if already established; don't count on it |

Keep an out-of-band path to critical infrastructure (local console, a separate
management network) that doesn't depend on the VPN's control plane — part of the
failure-mode analysis in an [HLD](../../../system-design/hld/hld.md).

## Quiz

Self-test on this material: **[Infra Access Quiz](https://claude.ai/artifact/XUcm5e6T6dHWsueoa9Txbp)**
— draws 10 random questions from a pool of 30 each time, so it's reusable. Shared with the sibling note: covers the Arc agent, outbound 443 flows, Entra SSH roles, outage behaviour, WireGuard, NetBird components, and groups and policies.
Supports skip / previous / jump-to-question navigation, and keeps your progress in the browser if you refresh.
