# PRISM DID backlog (ACA-Py + Credo-TS)

This document consolidates the theoretical implementation plan into a backlog that engineering teams can track. Tasks are grouped by phase and tagged according to whether they target ACA-Py (Py), Credo-TS (TS), or are shared (Both).

## Phase 0 – Foundations & prerequisites

| ID | Work item | Owner |
| --- | --- | --- |
| P0-1 | Stand up shared Cardano infra (Blockfrost project, Cardano node/relay, cardano-wallet, db-sync, NeoPRISM indexer) in dev + preprod per `research/prism-did-adoption/06-infrastructure-and-architecture.md`. | Both |
| P0-2 | Align on Cardano connectivity patterns (direct Blockfrost vs. gateway) and document decisions (`research/prism-did-adoption/08-decisions-testing-docs.md`). | Both |
| P0-3 | Define canonical PRISM DID/VDR identifier format (naming, query params) so ACA-Py/Credo registries return consistent URLs. | Both |
| P0-4 | Port or expose Castor capabilities as shared libraries (Python port, npm package extraction) per `03-gap-analysis.md`. | Both |

## Phase 1 – Modelling & crypto enablement

| ID | Work item | Owner |
| --- | --- | --- |
| P1-Py-1 | Add `PRISM` entry to `DIDMethods` with secp256k1/Ed25519/X25519 support and holder-defined policies; expose secp256k1 in KeyTypes/Askar. | Py |
| P1-Py-2 | Extend DID validation + admin schemas to recognise `did:prism` long-form syntax. | Py |
| P1-TS-1 | Add configuration surfaces in `DidsModuleConfig` / new module options for PRISM resolver/registrar injection. | TS |
| P1-TS-2 | Define TypeScript types/config for Cardano env (network, NeoPRISM URL, wallet connectors) shared across Node/RN builds. | TS |

## Phase 2 – Resolution path

| ID | Work item | Owner |
| --- | --- | --- |
| P2-Py-1 | Implement `PrismDIDResolver` (native) with NeoPRISM HTTP client + long-form fallback using Castor logic; integrate with `DIDResolver`. | Py |
| P2-Py-2 | Add admin/config options for NeoPRISM endpoint, auth, cache TTL; update multi-tenant scoping. | Py |
| P2-TS-1 | Wrap Castor’s `PrismDIDResolver` behind Credo `DidResolver`; hook into `DidResolverService` caching & failover rules. | TS |
| P2-TS-2 | Provide resolver config (NeoPRISM URL, fallback) through module options + environment loaders. | TS |
| P2-B-1 | Create mocked NeoPRISM fixture/service for CI so both agents can test resolution without live Cardano infra. | Both |

## Phase 3 – Publishing & VDR integration

| ID | Work item | Owner |
| --- | --- | --- |
| P3-Py-1 | Implement `PrismDidManager` (proto builder + secp256k1 signing) and wallet storage for PRISM metadata. | Py |
| P3-Py-2 | Add admin APIs (`/did/prism/create`, `/did/prism/publish`) with service/key payloads and Blockfrost/Node submission. | Py |
| P3-TS-1 | Build `PrismDidRegistrar` leveraging Castor’s create/publish helpers; support CIP-30 wallets + server-side signers. | TS |
| P3-TS-2 | Persist DID docs + metadata (state hash, tx hash, driver info) in `DidRepository` when requested. | TS |
| P3-B-1 | Package a shared PRISM VDR driver (plugin for ACA-Py, `@credo-ts/prism-vdr` for Credo) implementing AnonCreds `register/get` flows per `09-vdr-requirements.md`. | Both |
| P3-B-2 | Document transaction submission pathways (Blockfrost REST, CLI sidecar, wallet adapters) for operators. | Both |

## Phase 4 – Protocol wiring & UX

| ID | Work item | Owner |
| --- | --- | --- |
| P4-Py-1 | Update Out-of-Band/DID Exchange to accept `use_did_method="did:prism"`; ensure DID Rotation recognises PRISM entries. | Py |
| P4-Py-2 | Emit new webhooks (creation, anchoring confirmed, rotation) and expose metrics (tx latency, resolver latency). | Py |
| P4-TS-1 | Extend `DidsApi`, DIDComm services, and connection flows to surface PRISM options and route DIDComm endpoints. | TS |
| P4-TS-2 | Tag records with Cardano network + tx hash for multi-tenant reuse and future updates. | TS |
| P4-B-1 | Update documentation/SDK samples to show DID selection toggles, wallet UX, and fallback behaviours. | Both |

## Phase 5 – Testing, docs, rollout

| ID | Work item | Owner |
| --- | --- | --- |
| P5-Py-1 | Unit tests: resolver parsing, proto encoding, secp256k1 utils. Integration: mocked NeoPRISM, end-to-end DID Exchange w/ PRISM. | Py |
| P5-TS-1 | Unit tests: resolver wrapper, registrar validation, env config. Integration: CIP-30 wallet simulators, Node server tests. | TS |
| P5-B-1 | Add AATH scenarios + Credo interoperability tests covering PRISM invites, rotations, and credential issuance. | Both |
| P5-B-2 | Finalise documentation backlog (feature guides, ops runbooks) and mark release criteria incl. feature flags, monitoring. | Both |
| P5-B-3 | Plan staged rollout (dev → preprod → mainnet) with health checks for NeoPRISM, Blockfrost quotas, and VDR proofs. | Both |

## Tracking notes

- Each backlog item references the detailed context in documents `04`, `05`, `06`, `07`, `08`, and `09` to keep this list concise.
- IDs can evolve into Jira/Epic identifiers; the current notation simply clarifies ownership and phase.
- Infrastructure/integration items (tagged “Both”) should be coordinated via shared workstreams so the two agents do not diverge.
