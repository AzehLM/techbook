# Keywords

## Software Lifecycle

- **GA** — General Availability. The stage at which a product or feature is officially released to all customers as stable and production-ready, following alpha/beta/preview stages.
- **EOS** — End of Support. The date after which the vendor stops providing technical support, bug fixes, or patches for a product, though it may still be usable.
- **EOL** — End of Life. The date after which a product is fully retired: no more updates, patches, or support of any kind, including security fixes. Continued use carries risk.

## System Design Documents

- **HLD** — High-Level Design. A document describing the overall architecture of a system: major components, their interactions, data flow, and technology choices, without going into implementation detail.
- **LLD** — Low-Level Design. A document detailing the internal implementation of individual components defined in the HLD: class/module structure, algorithms, data structures, APIs, and precise logic.
- **DAT** — Document d'Architecture Technique (French). Technical architecture document: products and versions, environments, interfaces, data and sizing, typically reviewed before go-live. Roughly an HLD. See [dat.md](../system-design/dat/dat.md).
- **PoC** — Proof of Concept. A minimal first version built to prove value or feasibility quickly, before investing in the robust target architecture.
- **Architecture description** — ISO/IEC/IEEE 42010's name for a document describing an architecture through stakeholders, concerns, viewpoints and views.
- **arc42** — A free 12-section template for documenting software architecture.
- **C4 model** — Simon Brown's diagramming approach at four zoom levels: Context, Containers, Components, Code.

## Identity & Access Management

- **IAM** — Identity and Access Management. The discipline and tooling for managing who a user is (identity) and what they may access (authorization) across systems.
- **IdP** — Identity Provider. The central service that authenticates users and issues signed tokens/assertions that applications trust (e.g. Microsoft Entra ID, Okta, Keycloak).
- **SP / RP** — Service Provider / Relying Party. An application that delegates sign-in to an IdP and trusts its tokens.
- **SSO** — Single Sign-On. Authenticate once to the IdP, then access many applications without re-entering credentials.
- **SAML** — Security Assertion Markup Language. XML-based OASIS standard for exchanging signed authentication assertions between an IdP and an SP; common for enterprise SaaS SSO.
- **OAuth 2.0** — IETF authorization framework (RFC 6749) that lets a client obtain an access token to call an API on a user's behalf; not a login protocol by itself.
- **OIDC** — OpenID Connect. Authentication layer on top of OAuth 2.0 that adds a JWT "ID token" describing who the user is.
- **JWT** — JSON Web Token (RFC 7519). Compact, signed JSON token format used for OIDC ID tokens and many access tokens.
- **SCIM** — System for Cross-domain Identity Management (RFC 7643/7644). REST standard an IdP uses to create, update and disable user/group accounts in applications (provisioning/deprovisioning).
- **JIT** — Just-In-Time. (1) JIT access: privileges granted only when needed and for a limited time. (2) JIT provisioning: an app account auto-created at first SSO login.
- **PIM** — Privileged Identity Management. Microsoft Entra feature (P2/ID Governance) that makes admin roles "eligible" and activated just-in-time with approval, justification and expiry.
- **CA** — Conditional Access. Microsoft Entra policy engine (P1) that grants or blocks sign-ins based on conditions (user/group, app, location, device, risk), e.g. requiring MFA.
- **Security defaults** — Free, all-or-nothing Microsoft Entra baseline: MFA registration for all, MFA for admins and Azure management, legacy auth blocked; no exclusions.
- **MFA** — Multi-Factor Authentication. Sign-in requiring two or more factors (something you know / have / are).
- **SSPR** — Self-Service Password Reset. Lets users reset a forgotten password themselves after verifying their identity (P1 in Entra).
- **MAU** — Monthly Active Users. Billing unit for external/guest identities (e.g. Entra B2B guests: first 50,000 MAU free).
- **B2B guest** — External user invited into a tenant who signs in with their own home identity.
- **RBAC** — Role-Based Access Control. Permissions are attached to roles; users get them only through role (usually group) membership.
- **ABAC** — Attribute-Based Access Control. Access decided by rules over user, resource and environment attributes.
- **Least privilege** — Security principle: give each identity only the minimum permissions its task requires.

## Cloud / Azure

- **Tenant** — A dedicated instance of an identity directory (e.g. Microsoft Entra ID) for one organisation.
- **Subscription** — Azure billing and resource container; resources live in resource groups inside a subscription, under optional management groups.
- **Azure Arc** — Azure service that projects non-Azure machines (on-prem, other clouds) and Kubernetes clusters into Azure for management, RBAC and policy.
- **Managed identity** — An identity in Entra ID automatically managed for an Azure (or Arc-enabled) resource, so it can authenticate to services without stored secrets.
- **Service tag** — Named group of Azure IP prefixes (e.g. AzureActiveDirectory) used in firewall rules instead of raw IP lists.
- **CSP** — Cloud Solution Provider. Microsoft's reseller programme through which partners sell and bill Microsoft cloud licences.
- **PAYG** — Pay-as-you-go. Billing model with no upfront commitment: you pay only for resources consumed.

## Networking

- **Mesh VPN / overlay network** — Private network built on top of existing networks where peers connect directly to each other under central policy (e.g. NetBird, Tailscale).
- **WireGuard** — Modern, minimal VPN protocol that tunnels IP over UDP using public-key peer identities (Curve25519, ChaCha20-Poly1305).
- **NAT traversal (STUN/TURN/ICE)** — Techniques letting peers behind NAT connect directly: STUN discovers the public address, ICE picks the best path, TURN relays when direct fails.
- **Zero Trust** — Security model where network location grants no implicit trust; every access is verified by identity, device and policy.

## AI: Embeddings, Vector Search & RAG

- **Embedding** — A fixed-length vector produced by a model so that content with similar meaning lands close together in vector space.
- **Vector database** — A store that indexes embeddings plus metadata and returns the nearest vectors to a query vector.
- **kNN** — k-Nearest Neighbours. Finding the k stored vectors closest to a query vector.
- **ANN** — Approximate Nearest Neighbour. Index-based kNN search that trades a little recall for much faster queries.
- **HNSW** — Hierarchical Navigable Small World. A multi-layer proximity-graph ANN index; the usual default for a strong speed/recall trade-off.
- **IVF** — Inverted File index. An ANN index that clusters vectors into cells (k-means) and scans only the closest cells at query time.
- **PQ** — Product Quantization. Compresses vectors into short codes per sub-vector to cut memory, at some cost in accuracy.
- **Cosine similarity** — Similarity based on the angle between two vectors, ignoring their length; the default for most text embeddings.
- **Recall@k** — The share of the true top-k nearest neighbours that a search actually returns.
- **RAG** — Retrieval-Augmented Generation. Retrieving relevant passages from your own corpus and adding them to an LLM prompt so answers are grounded in them.
- **Chunking** — Splitting documents into smaller passages before embedding them for retrieval.
- **top-k** — The number of most-similar results a retrieval step returns.
- **BM25** — A classic keyword-based relevance-ranking function used by search engines.
- **Hybrid search** — Combining keyword (BM25) and vector search results, usually merged with RRF.
- **RRF** — Reciprocal Rank Fusion. Merges ranked lists by summing 1/(k + rank) for each document; needs no score tuning.
- **Reranking** — A second, more precise model that re-scores the top retrieved candidates before they are passed to the LLM.
- **MTEB** — Massive Text Embedding Benchmark. A multi-task, multilingual leaderboard for comparing embedding models.
- **MRL** — Matryoshka Representation Learning. Training embeddings so they can be truncated to fewer dimensions while keeping most of their quality.
- **Air-gapped** — An environment with no connection to the Internet, so all services (models included) must be self-hosted.

## AI: GPU Inference

- **VRAM** — The GPU's on-board memory, which holds model weights, activations and the KV cache.
- **KV cache** — Stored attention keys/values that let an LLM generate text without recomputing earlier tokens; grows with context length and concurrent requests.
- **Compute capability** — NVIDIA's version number for a GPU architecture's feature set (e.g. 8.0 Ampere, 9.0 Hopper).
- **MIG** — Multi-Instance GPU. Hardware partitioning of one NVIDIA GPU (Ampere or newer) into up to 7 isolated instances, each with its own compute and memory.
- **NVLink** — NVIDIA's high-bandwidth direct GPU-to-GPU interconnect, used to split one large model across several GPUs.
- **Tensor parallelism** — Splitting each model layer across several GPUs; needs fast interconnects like NVLink.
- **Time-slicing** — Sharing one GPU between processes by taking turns, with no memory or fault isolation.

## Dataiku (name to be updated and keywords moved depending on technical specification)

- **LLM Mesh** — Intermediary layer of Dataiku which is between the DSS and all the IA models (languages, embedding, etc.) that we could use