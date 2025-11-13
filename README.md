# PRISM DID Adoption Research

This repository snapshot captures a cross-project research effort aimed at adding `did:prism` support to the two flagship Aries agents maintained here:

1. **Aries Cloud Agent Python (ACA-Py)** – the reference Python cloud agent widely used in the Aries ecosystem.
2. **Credo-TS** (formerly Aries Framework JavaScript) – the extensible TypeScript agent framework used for edge and server deployments.

The research focuses on what it will take to bring Cardano-backed PRISM DIDs (and their Verifiable Data Registry) into those stacks alongside the DID methods and VDRs they already support.

## Goals

- Catalogue the DID capabilities ACA-Py and Credo-TS ship today and map them against the PRISM DID method specification.
- Inventory the components (registries, resolvers, registrars, SDK helpers, infrastructure) involved in DID/VDR interactions for both agents.
- Identify capability gaps, infrastructure prerequisites, and reusable library work needed to close the delta.
- Produce phased implementation plans, decision/risk registers, architecture diagrams, and testing/documentation backlogs so engineering teams can execute with minimal discovery.

## Key outcomes

All findings live in `research/prism-did-adoption/`. Highlights:

- [`01-did-methods.md`](research/prism-did-adoption/01-did-methods.md) – tabular comparison of ACA-Py and Credo-TS DID methods vs. PRISM traits (crypto, lineage, services, operations).
- [`02-component-analysis.md`](research/prism-did-adoption/02-component-analysis.md) – deep dive on resolver/registrar stacks, SDK dependencies, and how PRISM building blocks (Castor, NeoPRISM) fit in.
- [`03-gap-analysis.md`](research/prism-did-adoption/03-gap-analysis.md) – detailed capability gaps, required blockchain networks/config surfaces, and proposed shared libraries per language.
- [`04-implementation-plan-acapy.md`](research/prism-did-adoption/04-implementation-plan-acapy.md) / [`05-implementation-plan-credo.md`](research/prism-did-adoption/05-implementation-plan-credo.md) – phased roadmaps for ACA-Py and Credo-TS covering modelling, resolution, publishing, protocol wiring, and rollout.
- [`06-infrastructure-and-architecture.md`](research/prism-did-adoption/06-infrastructure-and-architecture.md) – current vs. future-state diagrams plus sequence diagrams for publishing/resolving PRISM DIDs.
- [`07-recommendations-risks.md`](research/prism-did-adoption/07-recommendations-risks.md) – guidance on phased rollout, feature flags, reuse of existing assets, and a risk register with mitigations.
- [`08-decisions-testing-docs.md`](research/prism-did-adoption/08-decisions-testing-docs.md) – open decisions, documentation backlog, and multi-layer test strategy.
- [`09-vdr-requirements.md`](research/prism-did-adoption/09-vdr-requirements.md) – alignment between ACA-Py/Credo VDR interfaces and the PRISM VDR technical specification (drivers, URL managers, proofs, metadata registries) with actionable next steps.
- [`10-prism-backlog.md`](research/prism-did-adoption/10-prism-backlog.md) – consolidated phase/task backlog (per agent + shared infra) distilled from the research so engineering teams can plan executions.

Each document cites the exact files/lines across ACA-Py, Credo-TS, SDK-TS, NeoPRISM, and the PRISM spec that informed the findings.

## How to use this research

1. **Architecture & planning** – Start with `01`–`03` to understand the current landscape and gaps, then dive into the implementation plan for your target agent.
2. **Infrastructure teams** – Use `06` for deployment blueprints (Cardano node, Blockfrost, db-sync, NeoPRISM) and `07`/`08` for risk + doc requirements.
3. **VDR/AnonCreds contributors** – [`09-vdr-requirements.md`](research/prism-did-adoption/09-vdr-requirements.md) spells out how PRISM’s VDR spec maps onto ACA-Py plugins and Credo’s `AnonCredsRegistry` abstraction.
4. **Execution** – Follow the phase-by-phase tasks in `04`/`05`, keeping the decision log, test plan, and documentation backlog in `08` up to date as work lands.

The intent is to keep this directory as the single source of truth for bringing PRISM DID + VDR capabilities into ACA-Py and Credo-TS. Contributions should extend the documents rather than scatter context across issue trackers or ad-hoc notes.
