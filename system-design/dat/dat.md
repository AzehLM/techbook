# DAT — Document d'Architecture Technique

Notes from:
- Template: [arc42 overview](https://arc42.org/overview) — Gernot Starke & Peter Hruschka (CC BY-SA 4.0)
- Standard: [ISO/IEC/IEEE 42010:2022 — Software, systems and enterprise — Architecture description](https://www.iso.org/standard/74393.html), and its [conceptual model](http://www.iso-architecture.org/42010/cm/)
- Model: [The C4 model](https://c4model.com/) — Simon Brown
- Docs: [NVIDIA Multi-Instance GPU User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/) — cited as an example of sourcing a hardware constraint

Related: [HLD](../hld/hld.md)

## What it is

A **DAT** (*Document d'Architecture Technique*, "technical architecture document") is the
French-IT-world name for the document that describes **how a system is technically built
and deployed**: its components and their versions, where it runs, how data flows between
systems, and which constraints it must satisfy. It's typically produced for an
architecture review board, IT operations and security before a project goes live, and
kept up to date as the system evolves.

It sits between the other design documents:

| Document | Question it answers | Audience |
|----------|---------------------|----------|
| Functional spec | *What* must the system do? | business, product |
| [HLD](../hld/hld.md) (High-Level Design) | What are the major components and how do they interact? | architects, tech leads |
| **DAT** | With *which products/versions*, on *which infrastructure*, with *which interfaces, data and constraints*? | architecture board, ops, security, integrators |
| LLD (Low-Level Design) | How is each component implemented internally? | developers |

In practice a DAT overlaps heavily with the HLD but is more **concrete and operational**:
product names, version numbers, environments, network flows, sizing, data classification.

There is no single official DAT template — each organization has its own. The public,
vendor-neutral references below give the vocabulary and a sound skeleton.

## The public references

### ISO/IEC/IEEE 42010 — the vocabulary

The international standard for **architecture descriptions** (2nd edition, 2022). It doesn't
impose sections; it defines what a good description must contain:

| Concept | Meaning | In a DAT |
|---------|---------|----------|
| **Stakeholder** | Anyone with an interest in the system | ops, security, users, vendor, project team |
| **Concern** | A stakeholder's question or interest | "Will it fit on our servers?", "Where does the data live?" |
| **Viewpoint** | Conventions for building a kind of view (which concerns, which notation) | "deployment viewpoint", "data viewpoint" |
| **View** | The actual description of the system from one viewpoint | the per-environment diagram, the interface table |

Mnemonic: **stakeholders have concerns; viewpoints frame concerns; views apply viewpoints.**
The practical lesson: every DAT section should exist because some stakeholder has a concern
it answers.

### arc42 — a ready-made skeleton

A free (CC BY-SA 4.0) template with 12 sections:

| # | Section | Contents |
|---|---------|----------|
| 1 | Introduction & Goals | Requirements, top quality goals, stakeholders |
| 2 | Constraints | Regulations, mandated technologies, external constraints |
| 3 | Context & Scope | External systems and interfaces |
| 4 | Solution Strategy | Core ideas and technology choices |
| 5 | Building Block View | Static decomposition into components |
| 6 | Runtime View | Important runtime scenarios |
| 7 | Deployment View | Hardware, infrastructure, environments |
| 8 | Crosscutting Concepts | Security, logging, persistence patterns… |
| 9 | Architecture Decisions | Important decisions and their rationale |
| 10 | Quality Requirements | Quality tree, scenarios |
| 11 | Risks & Technical Debt | Known problems and risks |
| 12 | Glossary | Domain and technical terms |

### C4 — the diagrams

A notation-independent way to draw the architecture at four zoom levels:

| Level | Shows |
|-------|-------|
| 1. System Context | The system, its users and the external systems it talks to |
| 2. Container | Deployable/runnable units (apps, databases, services) and how they communicate |
| 3. Component | Main components inside one container |
| 4. Code | Classes/functions (rarely drawn by hand) |

Plus supplementary diagrams: **System Landscape**, **Dynamic** (a scenario), and
**Deployment** (containers mapped onto infrastructure/environments) — the last one is exactly
the "architecture per environment" diagram a DAT needs.

Mnemonic: **C4 = Context, Containers, Components, Code** — zooming in like a map.

## A typical DAT structure

A common shape for a French-style DAT, mapped to arc42/C4:

| DAT section | Contents | arc42 / C4 |
|-------------|----------|------------|
| **Functional description** | Business need, scope of the document, role of each module, options retained | arc42 §1, §4 |
| **Macro software architecture** | Simplified functional flow (source → processing → storage → consumption) | arc42 §5, C4 context/container |
| **IT compatibility matrix** | Component × version constraint × source × status | arc42 §2 |
| **Application reference** | Product name, version, edition, description | — |
| **Front-end / back-end / client / other components** | Tables per tier: product, version, options | arc42 §5, C4 container |
| **Architecture per environment** | Dev / test / prod diagrams, where each component runs | arc42 §7, C4 deployment |
| **Sizing** | Estimation method: CPU, RAM, disk, GPU; what is isolated and what is shared | arc42 §7, §10 |
| **Data: nature, location, classification** | Each data type, where it's stored, its sensitivity | arc42 §8 |
| **Application interfaces** | Every flow: from → to, protocol (JDBC, HTTPS, in-process…), auth | arc42 §3, C4 dynamic |
| **Derogations / open points** | Deviations from standards, decisions still pending, blockers | arc42 §9, §11 |
| **Appendices** | Deep dives (e.g. model choice, hardware options), sources | — |

## Good practices

### Source every version constraint

A compatibility matrix is only trustworthy if every line says **where the constraint comes
from** and **how sure you are**:

| Component | Constraint | Source | Status |
|-----------|------------|--------|--------|
| Platform feature X | ≥ version 12.3 | vendor docs (link) | confirmed |
| Partitioned GPU (MIG) | NVIDIA Ampere or newer | NVIDIA MIG User Guide | to verify on the actual hardware |
| Installed platform version | unknown | — | **blocking**: ask ops |

The **highest** minimum version across all features is the real floor — list it explicitly.

### Track the vendor lifecycle

Record **GA** (general availability), **EOS** (end of support) and **EOL** (end of life) dates
for the product versions you depend on, so the DAT shows how long the chosen version is
supported. These roadmaps are often **not public**: vendors share them with customers
through the account team or support portal — note that as an action to take rather than
leaving the cell empty.

### Data inherits its classification

Derived data (indexes, caches, embeddings, logs) **inherits the classification of its source
data**: vectors computed from confidential documents are confidential. Public artefacts
(e.g. an open-source model downloaded from a public hub) can be marked non-sensitive.

### Describe versioned targets

When the architecture will evolve, show the stages explicitly — e.g. a **V0 / PoC** (proof of
concept: fastest to prove value, fewest components) and a **V1 / target** (robust,
reusable). Keep the common trunk identical and show **only what differs** between versions
(often one component, like where the data is stored), with a table column per version.

### Be explicit about the unknowns

A "derogations / points to decide" section is a feature, not a weakness: listing what is
unconfirmed (versions, hardware models, pending choices) and what it blocks keeps the
document honest and turns it into an action list.

### Every section answers a concern

(42010's lesson.) If nobody would miss a section, cut it; if a stakeholder has a question the
DAT can't answer, add a view.

## Quiz

Self-test on this material: **[DAT Review Quiz](https://claude.ai/artifact/1f6iLUmHp3JfTFCGXhkUqB)**
— draws 10 random questions from a pool of 28 each time. Covers DAT vs HLD/LLD, the
42010 vocabulary, arc42 sections, C4 levels, the typical DAT structure and good practices.
Supports skip / previous / jump-to-question navigation, and keeps your progress in the
browser if you refresh.

## Takeaways

- DAT = concrete technical architecture: products, versions, environments, flows, data, sizing.
- Vocabulary from ISO/IEC/IEEE 42010, skeleton from arc42, diagrams from C4.
- Source every constraint, track GA/EOS/EOL, classify derived data, show versioned targets, list open points.
