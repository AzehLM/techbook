# SSO & Identity Federation (SAML, OIDC, OAuth 2.0, SCIM)

Notes from:
- Doc: [What is single sign-on in Microsoft Entra ID?](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/what-is-single-sign-on) — Microsoft Learn
- Doc: [SAML authentication with Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/architecture/auth-saml) — Microsoft Learn
- Doc: [OAuth 2.0 and OpenID Connect protocols](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols) — Microsoft identity platform
- Article: [How OpenID Connect works](https://openid.net/developers/how-connect-works/) — OpenID Foundation
- Spec: [RFC 6749 — The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749) — IETF
- Spec: [RFC 7519 — JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519) — IETF
- Spec: [RFC 7644 — SCIM 2.0 Protocol](https://datatracker.ietf.org/doc/html/rfc7644) — IETF
- Doc: [Configure group claims for applications](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-fed-group-claims) — Microsoft Learn
- Doc: [How application provisioning works](https://learn.microsoft.com/en-us/entra/identity/app-provisioning/how-provisioning-works) — Microsoft Learn
- Doc: [Scoping filters for provisioning](https://learn.microsoft.com/en-us/entra/identity/app-provisioning/define-conditional-rules-for-provisioning-user-accounts) — Microsoft Learn

Related notes: [Microsoft Entra ID](../entra-id/entra-id.md) ·
[RBAC](../../access-control/rbac/rbac.md) ·
[Mesh VPN (IdP login for network access)](../../../networking/vpn/mesh-vpn/mesh-vpn.md)

## What it is

**Single sign-on (SSO)**: the user authenticates **once**, to one central
**identity provider (IdP)**, and is then let into many applications without a separate
password for each. **Federation** is the trust relationship that makes it work: each
app (the *service provider*, SP, or *relying party*, RP) agrees to trust signed
statements — **tokens** / **assertions** — issued by the IdP.

Why bother:
- **One place to enforce MFA** and sign-in policy, instead of per-app settings.
- **One place to disable a leaver** — no orphan accounts in 15 SaaS tools.
- Apps never see or store the user's password.

## The vocabulary

| Term | Meaning | Example |
|---|---|---|
| **IdP** (identity provider) / OpenID Provider / authorization server | Authenticates users and issues tokens | Microsoft Entra ID, Okta, Keycloak, Google |
| **SP** (service provider) / **RP** (relying party) / client | The app that trusts the IdP | A SaaS tool, a self-hosted dashboard |
| **Principal** / resource owner | The user being authenticated | Alice |
| **Assertion / token** | Signed statement from the IdP about the user | SAML assertion (XML), ID token (JWT) |
| **Claim** / attribute | One fact inside the token | `email`, `name`, `groups`, `roles` |
| **Metadata** | Config describing the other side: endpoints, signing certs | SAML metadata XML; OIDC `/.well-known/openid-configuration` |

**Authentication vs authorization**: *AuthN* = who are you (IdP's job). *AuthZ* = what
may you do (usually the app's job, fed by claims). See [RBAC](../../access-control/rbac/rbac.md).

## The protocols

| | **SAML 2.0** | **OAuth 2.0** | **OpenID Connect (OIDC)** | **LDAP** |
|---|---|---|---|---|
| Purpose | Authentication / SSO for web apps | **Authorization** (delegated access to APIs) | **Authentication** layer on top of OAuth 2.0 | Directory lookup + bind (username/password check) |
| Year / owner | 2005, OASIS | 2012, IETF (RFC 6749) | 2014, OpenID Foundation | 1990s, IETF |
| Format | XML assertions, signed (XML-DSig) | Access tokens (format unspecified; often JWT) | **ID token = JWT** + OAuth tokens | Binary protocol (BER) |
| Transport | Browser redirects / HTTP POST | HTTPS redirects + back-channel token call | Same as OAuth | TCP 389 / 636 (LDAPS) |
| Typical use | Enterprise SaaS, older apps | App calling an API on the user's behalf | Modern web, mobile, SPAs, CLIs | On-prem apps, Linux, network gear |
| Does the app see the password? | No | No | No | **Yes** (the app forwards it to the directory) |

Key points:
- **OAuth 2.0 alone is not a login protocol.** An access token says "this client may
  call this API," not "this is Alice." Using raw OAuth for login was a classic mistake;
  **OIDC** fixes it by adding the **ID token** — a JWT with at least a subject (`sub`),
  issuer (`iss`), audience (`aud`), and when/how the user authenticated.
- **SAML and OIDC solve the same SSO problem** with different eras of technology. When
  an app supports both, OIDC is usually easier (JSON, discovery document, works for
  mobile/CLI); SAML is still the enterprise SaaS lingua franca.
- **LDAP is not SSO**: the app collects the password and checks it against the
  directory. It centralises accounts, not sign-in. (Entra ID itself doesn't speak LDAP
  — that would need Entra Domain Services.)

Mnemonic: **SAML = XML for the enterprise browser · OAuth = Access (a valet key) ·
OIDC = OAuth + ID token (who you are).**

## How a sign-in flows

### SAML (SP-initiated)

```
1. User → App (SP):           GET /dashboard (no session)
2. App → Browser:             redirect to IdP with a SAML AuthnRequest
3. Browser → IdP:             user signs in (password + MFA)
4. IdP → Browser:             auto-submitting form with a signed SAML Response (assertion)
5. Browser → App (ACS URL):   POST the assertion
6. App:                       verifies signature with IdP cert, reads claims → session
```

**IdP-initiated** SSO skips steps 1–2: the user clicks the app tile in the IdP portal
and the IdP POSTs an unsolicited assertion to the app.

### OIDC (authorization code flow)

```
1. App → Browser:   redirect to IdP /authorize?client_id=…&redirect_uri=…&scope=openid profile&response_type=code
2. Browser ↔ IdP:   user signs in
3. IdP → Browser:   redirect to redirect_uri?code=XYZ
4. App → IdP:       back-channel POST /token (code + client secret or PKCE verifier)
5. IdP → App:       ID token (JWT) + access token (+ refresh token)
6. App:             validates JWT signature via the IdP's JWKS keys → session
```

The three OAuth/OIDC tokens:

| Token | For whom | Says |
|---|---|---|
| **ID token** | The client app | Who the user is (authentication result) |
| **Access token** | The API / resource server | What the client may do there |
| **Refresh token** | The IdP only | "Give me new tokens" without re-prompting the user — treat as a secret |

## Groups in tokens

Apps often derive roles from the user's **group memberships** carried in the token
(`groups` claim). Things to know (Entra ID as the example, but the problems are generic):

- **Identifier vs name**: Entra emits cloud groups as **object IDs (GUIDs)** by
  default, so the app maps `3f2a…-…` → `admin`. Readable names are only emitted with the
  "groups assigned to the application" option, which requires assigning groups to the
  app (P1 in Entra).
- **Size limits / overage**: if the user is in too many groups (Entra: 150 in SAML,
  200 in JWT), the claim is replaced by an "overage" pointer and the app must query the
  Graph API. Filter the claim (security groups only, prefix filters, assigned groups)
  to stay under the limit.
- **Nesting**: Entra group claims include transitive (nested) memberships — except
  with the "assigned groups" option.
- **App roles** are an alternative: the app declares roles (`Admin`, `Reader`), you
  assign users/groups to them, and the token carries a `roles` claim with readable names.

## Provisioning: SCIM

SSO answers *"can this person sign in?"* but many apps also need an **account to exist**
beforehand (to attach licences, data, permissions), and — crucially — to be
**disabled when the person leaves**. That's provisioning.

| Approach | How | Downside |
|---|---|---|
| Manual | Admin creates accounts in each app | Forgotten leavers |
| **JIT (just-in-time) provisioning** | Account auto-created at first SSO login from token claims | Never *deprovisions*: a leaver's account lingers (though they can't sign in via SSO) |
| **SCIM** | The IdP pushes create/update/disable to the app's SCIM API | App must support SCIM; often a paid tier |

**SCIM 2.0** (System for Cross-domain Identity Management, RFC 7643/7644) is a REST
API with a standard JSON schema: `POST/GET/PUT/PATCH/DELETE` on `/Users` and
`/Groups`. The IdP runs a sync cycle and calls those endpoints.

In Entra: user provisioning works on Free; provisioning **groups** and scoping
provisioning by group need P1; nested groups are never expanded by provisioning.

## Keep a way in when SSO breaks

Federation makes the IdP a single point of failure for every app behind it. Standard
practice for critical systems:
- Keep at least one **local admin account** in each app/tool, not dependent on the IdP,
  with credentials stored securely and use logged.
- Check the app allows **local login and SSO side by side** (some "SSO-enforced" plans
  disable local login entirely).
- Know which apps can be reached if the IdP, the Internet, or the VPN is down — part of
  the failure-mode analysis in an [HLD](../../../system-design/hld/hld.md).

> SSO replaces *authentication*, not *encryption*. A password manager with SSO login
> typically still needs a master password (or equivalent key) to decrypt the vault —
> the IdP never holds the key.

## Checklist: evaluating an app for SSO integration

For each app, answer from the vendor's docs **for your exact version and plan**:

| Question | Why |
|---|---|
| SSO supported natively? Which protocol (SAML / OIDC / LDAP only)? | LDAP-only means no real SSO |
| Which plan/edition unlocks SSO? Extra cost? | The "SSO tax": often Enterprise-only |
| Can it read **group claims** and map groups → its own roles? | Otherwise roles are managed by hand in the app |
| **SCIM** support? Which plan? Users only or groups too? | Deprovisioning of leavers |
| Does a **local admin** keep working with SSO on? | Break-glass path |
| Refuses users who have no mapped group? | Otherwise "assignment not required" lets everyone in |
| Tested in a PoC? | Docs often skip edge cases (nested groups, fallback login, claim format) |

## Quiz

Self-test on this material: **[Identity & Access Quiz](https://claude.ai/artifact/31uVhRCfgxsVFmxL2jngyj)**
— draws 10 random questions from a pool of 30 each time, so it's reusable. Shared with the sibling notes: covers Entra building blocks, tiers and Conditional Access, SAML vs OIDC, claims and SCIM, RBAC models, and group naming and nesting.
Supports skip / previous / jump-to-question navigation, and keeps your progress in the browser if you refresh.
