# Components and responsibilities

## ACA-Py building blocks

- **DID method registry** – `DIDMethod` instances carry crypto, rotation, and holder-defined policies with a central registry for lookups (`acapy/acapy_agent/wallet/did_method.py:18-148`). This registry is injected into wallet sessions, DID validation, and DID creation routes.
- **Wallet + routes** – Admin HTTP routes such as `POST /wallet/did/create` and DID listing rely on injected `DIDMethods`, `KeyTypes`, and the current wallet implementation (`acapy/acapy_agent/wallet/routes.py:508-760`). They are the touch points for adding a new DID method type, validations, and metadata.
- **Ledger-aware managers** – The Indy DID manager derives seeds, writes NYM records, and persists metadata in the wallet storage (`acapy/acapy_agent/did/indy/indy_manager.py:1-78`). Equivalent managers do not yet exist for other ledger-backed methods.
- **Resolver framework** – `DIDResolver` routes resolution requests to registered resolvers, prioritising native implementations but permitting non-native (e.g., universal resolver) fallbacks (`acapy/acapy_agent/resolver/did_resolver.py:30-176`).
- **Concrete resolvers** – Each DID method has its own resolver module: web (`resolver/default/web.py:1-82`), webvh (`resolver/default/webvh.py:1-45`), JWK (`resolver/default/jwk.py:1-52`), Indy (`resolver/default/indy.py:1-210`), and peer variants. They encapsulate networking, ledger lookups, and DIDComm service derivation.
- **Universal resolver adapter** – The HTTP-based adapter can proxy methods the agent does not natively support by querying any universal resolver endpoint configured via settings (`acapy/acapy_agent/resolver/default/universal.py:1-126`).

### ACA-Py ↔ PRISM implications

- ACA-Py needs a new `DIDMethod` entry for `prism`, including secp256k1/Ed25519 key support plus rotation/policy flags. That definition drives wallet validations and DID creation options.
- A `BaseDIDResolver` implementation is required to talk to a PRISM resolver (NeoPRISM or universal). Because ACA-Py already supports HTTP resolvers, the universal resolver adapter can serve as a starting point by pointing it at NeoPRISM’s DID endpoint (`neoprism/docker/blockfrost-neoprism-demo/README.md:83-155`).
- Publishing and key management mirror what `DidIndyManager` does but using PRISM’s protobuf signing rules and Cardano plumbing instead of Indy NYM/ATTRIB calls.

## Credo-TS building blocks

- **Module registration** – The `DidsModule` wires a module config, resolver service, registrar service, and repository into the dependency manager (`credo-ts/packages/core/src/modules/dids/DidsModule.ts:1-27`).
- **Registrar routing** – `DidsModuleConfig` seeds default registrars/resolvers and keeps peer/key implementations always available so connection protocols remain functional (`packages/core/src/modules/dids/DidsModuleConfig.ts:1-110`).
- **Resolver service** – `DidResolverService` parses any DID, picks the right resolver, enforces caching/persistence semantics, and returns typed DIDDocuments (`packages/core/src/modules/dids/services/DidResolverService.ts:31-181`).
- **Registrar service** – Handles create/update/deactivate dispatching and cache invalidation, ensuring state transitions are serialized per DID (`packages/core/src/modules/dids/services/DidRegistrarService.ts:35-175`).
- **Peer DID stack** – The registrar and resolver implementations for `did:peer` show how create flows persist KMS keys and DID docs in the repository, and how resolvers reconstruct docs for each numalgo (`packages/core/src/modules/dids/methods/peer/*.ts`).

### Credo ↔ PRISM implications

- Adding `did:prism` is conceptually a new pair of registrar/resolver classes registered via `DidsModuleConfig`. Because resolvers can opt into caching and storing DID docs locally, the PRISM resolver can leverage NeoPRISM’s HTTP API yet still use long-form fallback logic borrowed from SDK-TS.
- Registrar support currently relies on method-specific SDKs (e.g., Indy VDR). For PRISM, Castor from SDK-TS already exposes helpers for DID creation, long-form decoding, and Atala object generation (`sdk-ts/src/castor/Castor.ts:42-220`). Wrapping Castor behind a Credo registrar would minimise duplicated Cardano logic.

## PRISM-specific components

- **PRISM DID specification** – Defines crypto primitives, service limits, and the protobuf structures for PRISM operations (`prism-did-method-spec/w3c-spec/PRISM-method.md:20-210`). Any implementation must obey these constraints.
- **SDK-TS Castor** – Provides programmatic DID creation, Atala object signing, long-form decoding, and resolver logic (including HTTP proxying and offline fallbacks) (`sdk-ts/src/castor/Castor.ts:42-220`, `sdk-ts/src/castor/resolver/PrismDIDResolver.ts:12-40`, `LongFormPrismDIDResolver.ts:23-178`).
- **NeoPRISM** – Runs as an indexer/submitter/standalone node, watching Cardano via DBSync or Oura, persisting DID state in PostgreSQL, and exposing universal-resolver compatible APIs (`neoprism/README.md:18-68`). The Blockfrost demo shows how it can sit behind a gateway alongside existing Cardano stacks (`neoprism/docker/blockfrost-neoprism-demo/README.md:1-155`).

## Component alignment

| Responsibility | ACA-Py today | Credo-TS today | PRISM counterpart | Integration notes |
| --- | --- | --- | --- | --- |
| DID method metadata | `DIDMethods` registry (`acapy/acapy_agent/wallet/did_method.py:107-148`) | `DidsModuleConfig` registrars/resolvers (`packages/core/src/modules/dids/DidsModuleConfig.ts:1-110`) | PRISM spec defines crypto + limits (`PRISM-method.md:20-89`) | Need to add `prism` entries with secp256k1 and service caps.
| DID creation | Indy manager + wallet insertions (`acapy/acapy_agent/did/indy/indy_manager.py:1-78`) | Registrar classes instantiating DID docs (`packages/core/src/modules/dids/methods/peer/PeerDidRegistrar.ts:1-164`) | Castor’s `createPrismDID` + Atala operations (`sdk-ts/src/castor/Castor.ts:165-220`) | Wrap Castor (TS) or reimplement proto logic (Python/Rust) to generate operations.
| DID resolution | Resolver interface + explicit method resolvers (`acapy/acapy_agent/resolver/did_resolver.py:30-176`, `resolver/default/*.py`) | Resolver service + method-specific resolvers (`packages/core/src/modules/dids/services/DidResolverService.ts:31-181`, `methods/*`) | NeoPRISM HTTP resolver + Castor long-form resolver (`sdk-ts/src/castor/resolver/*.ts`, `neoprism/README.md:18-68`) | Add HTTP resolver calling NeoPRISM, with long-form fallback for unpublished DIDs.
| Transport to ledger | Indy ledger executors and HTTP resolvers | Plugin-specific (Indy VDR, cheqd, etc.) | Cardano stack (Blockfrost, Cardano node, wallet, db-sync) (`sdk-ts/docs/prism/publishing-did.md:6-190`, `neoprism/docker/blockfrost-neoprism-demo/README.md:83-155`) | Both agents must wire new infrastructure credentials + services into their config surfaces.
