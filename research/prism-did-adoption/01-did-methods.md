# DID methods and capabilities

## ACA-Py support snapshot

ACA-Py keeps its DID portfolio in the `DIDMethod` registry where each method lists supported key types, whether rotation is allowed, and if controllers can supply their own DID (`acapy/acapy_agent/wallet/did_method.py:62-120`). Native resolvers cover `did:web`, `did:webvh`, `did:jwk`, `did:key`, `did:indy`, and all `did:peer` versions with optional fallback to a universal resolver (`acapy/acapy_agent/resolver/default/*.py`, `acapy/acapy_agent/resolver/did_resolver.py:30-176`).

| Method | Key types / crypto | Rotation | Holder-defined DID | Backing registry | Notable traits vs `did:prism` |
| --- | --- | --- | --- | --- | --- |
| `sov` | `ed25519` (`acapy/acapy_agent/wallet/did_method.py:62-67`) | ✅ | Allowed | Hyperledger Indy ledgers | Legacy Indy NYM semantics; no JSON-LD doc on-ledger; single verification key. |
| `indy` | `ed25519` (`acapy/acapy_agent/wallet/did_method.py:68-73`) | ✅ | Allowed | Indy (public or private) | Similar to `sov`, but qualified DID strings; relies on ledger read/write stack (`acapy/acapy_agent/did/indy/indy_manager.py:1-78`). |
| `did:key` | `ed25519`, `p256`, `bls12381g2` (`acapy/acapy_agent/wallet/did_method.py:74-78`) | ❌ | Derived | Self-certifying | No persistence, limited services, no update/deactivate semantics. |
| `did:web` | `ed25519`, `bls12381g2` (`acapy/acapy_agent/wallet/did_method.py:79-84`) | ✅ | Controller supplied | HTTPS hosted JSON-LD (`acapy/acapy_agent/resolver/default/web.py:1-82`) | Depends on HTTPS hosting; no ledger finality or batching. |
| `did:peer:2/4` | `ed25519`, `x25519` (`acapy/acapy_agent/wallet/did_method.py:85-104`) | ❌ (pairwise only) | Derived | Stored per-connection | High privacy, but no verifiable history; unanchored updates. |
| `did:webvh` | `ed25519`, `x25519` (`acapy/acapy_agent/wallet/did_method.py:99-104`) | ❌ | Derived | DID Web + verifiable history (`acapy/acapy_agent/resolver/default/webvh.py:1-45`) | Adds tamper-evident timeline but still HTTP hosted. |
| `did:jwk` | Any JWK (resolver builds doc) (`acapy/acapy_agent/resolver/default/jwk.py:1-52`) | ❌ | Derived | None | Single-key DID; enc/sig flagged via `use` claim. |

Compared to `did:prism`, ACA-Py’s existing methods either rely on Indy-style ledgers (ED25519 only) or self-contained DIDs that cannot express a DID history anchored to a public blockchain.

## Credo-TS support snapshot

Credo exposes DID support through the `DidsModule`. Defaults register resolvers/registrars for `did:key`, `did:peer`, `did:jwk`, and `did:web`, while optional packages add `did:sov`, `did:indy`, `did:cheqd`, and `did:hedera` (`credo-ts/packages/core/src/modules/dids/DidsModuleConfig.ts:1-110`, `credo-ts/README.md:41-58`). The resolver and registrar services dynamically route to the correct implementation via registry lookups (`credo-ts/packages/core/src/modules/dids/services/DidResolverService.ts:31-181`, `credo-ts/packages/core/src/modules/dids/services/DidRegistrarService.ts:35-175`).

| Method | Key types / crypto | Rotation | Backing registry | Notes |
| --- | --- | --- | --- | --- |
| `did:key` | Multicodec fingerprints for Ed25519, X25519, BLS, P-256 (dynamic) (`packages/core/src/modules/dids/methods/key/KeyDidResolver.ts:1-41`) | ❌ | Self-certifying | Useful for bootstrap; no anchoring. |
| `did:jwk` | Any JWK via `DidJwk` (`packages/core/src/modules/dids/methods/jwk/JwkDidResolver.ts:1-41`) | ❌ | Self-certifying | Single verification relationship, encryption or signing via `use`. |
| `did:peer` | Key material managed by KMS; supports numalgo 0/1/2/4 with short/long forms (`packages/core/src/modules/dids/methods/peer/PeerDidRegistrar.ts:1-164`, `PeerDidResolver.ts:1-101`) | Limited | Pairwise store / DidRepository | Local-first but no public registry or history. |
| `did:web` | Resolves via `web-did-resolver`, caches results (`packages/core/src/modules/dids/methods/web/WebDidResolver.ts:1-55`) | Controller managed | HTTPS hosting | Similar limitations as ACA-Py. |
| `did:sov` / `did:indy` | Provided by optional Indy modules; ledger-backed (per README) | ✅ | Indy ledgers | Requires Indy VDR packages and ledger endpoints. |
| `did:cheqd` | Cosmos SDK ledger (per README feature list) | ✅ | Cheqd network | Additional plugin. |
| `did:hedera` | Hedera network (per README) | ✅ | Hedera DID service | Additional plugin. |

Credo already multiplexes between heterogeneous ledgers (Indy, cheqd, Hedera) but still lacks Cardano-specific tooling and secp256k1-first DID docs.

## PRISM DID traits

The PRISM method is defined for Cardano mainnet and constrains crypto, encoding, and DID document shape (`prism-did-method-spec/w3c-spec/PRISM-method.md:20-125`). Key characteristics:

- Anchored via Cardano transactions and `PRISM nodes` that monitor, validate, and index DID operations (`PRISM-method.md:10-125`).
- Supports both short-form (on-chain) and long-form (self-contained) identifiers; long-form avoids waiting for finality but must eventually be anchored for global resolution (`PRISM-method.md:66-75`).
- DID documents can express all W3C verification relationships with JsonWebKey2020 verification methods using `secp256k1`, `ed25519`, and `x25519` keys plus up to 50 verification methods/services per DID (`PRISM-method.md:80-140`).
- Operations (create, update, deactivate) are batched as protobuf `AtalaOperations` and published as Cardano metadata, then consumed by PRISM nodes (`PRISM-method.md:142-210`).

## Capability comparison highlights

| Trait | ACA-Py DIDs | Credo-TS DIDs | `did:prism` |
| --- | --- | --- | --- |
| Public verifiable history | Indy-based methods only, tied to specific ledger pools; `did:webvh` offers limited HTTP-based history (`acapy/docs/features/QualifiedDIDs.md:5-40`, `acapy_agent/resolver/default/webvh.py:1-45`). | Indy/Cheqd/Hedera plugins provide ledger anchoring; `did:peer`/`did:key` do not. | Blockchain-backed with NeoPRISM indexers observing Cardano (`neoprism/README.md:18-68`). |
| Cryptography | Mostly Ed25519/Curve25519; P-256 for `did:key`; BLS for anoncreds contexts. | Similar mix; no first-class secp256k1 except optional modules. | Primary master keys on secp256k1 plus Ed25519/X25519 for issuance and key agreement (`PRISM-method.md:20-124`). |
| DID operations | Indy manager handles registration and writes NYM/ATTRIB transactions (`acapy_agent/did/indy/indy_manager.py:1-78`); other methods are either static or controller hosted. | Registrars abstract method-specific create/update/deactivate but rely on per-ledger SDKs (e.g., Indy VDR). | Explicit protobuf operations with batching, signature requirements, and Cardano metadata publishing (`PRISM-method.md:142-210`). |
| Service payloads | `did:web`/`did:peer` limited by hosting medium; Indy DIDs rely on endpoint attributes. | Similar constraints; `did:web` and `did:peer` require manual service injection. | Supports DIDComm V2, Linked Domains, and arbitrary services with Cardano-backed anchoring (`PRISM-method.md:83-140`). |
| Offline/long-form use | `did:peer` covers pairwise offline use. | `did:peer` & `did:key` provide offline fallback. | Long-form `did:prism` encodes full initial state for offline-first flows (`PRISM-method.md:70-90`). |

Takeaway: ACA-Py and Credo-TS already manage multiple DID styles, but none combine Cardano anchoring, secp256k1 master keys, and PRISM’s batched operation semantics. Adding `did:prism` requires both new crypto tooling and new infrastructure for publishing and resolving operations.
