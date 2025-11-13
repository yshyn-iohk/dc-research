# Gap analysis, networks, and library work

## Capability gaps between existing DID methods and `did:prism`

| Requirement from PRISM | ACA-Py status | Credo-TS status | Impact |
| --- | --- | --- | --- |
| **Cardano anchoring & block confirmation** – PRISM only recognises Cardano mainnet operations with 112-block finality (`prism-did-method-spec/w3c-spec/PRISM-method.md:16-41`). | No Cardano stack awareness; Indy ledger clients assume Indy transactions (`acapy/acapy_agent/resolver/default/indy.py:1-210`). | No Cardano integration in default modules; current ledgers are Indy/Cheqd/Hedera per README (`credo-ts/README.md:41-58`). | Need entirely new VDR connector (NeoPRISM or equivalent) plus config surfaces for Cardano nodes/DBSync/Blockfrost. |
| **secp256k1 master keys with Ed25519/X25519 companions** (`PRISM-method.md:20-124`). | `DIDMethod` registry only exposes Ed25519/X25519/P256/BLS (`acapy/acapy_agent/wallet/did_method.py:62-104`); no secp256k1 key type definitions or wallet crypto helper. | Credo’s key resolvers/registrars focus on Ed25519/X25519 (peer) and whichever type the method demands; secp256k1 appears only via optional packages, not in the DID core. | Need new key abstractions, encoding/decoding, and verkey storage for secp256k1 and PRISM usage-specific tags (master, authentication, issuance). |
| **Protobuf Atala operations & batching** (`PRISM-method.md:142-210`). | No serializer for PRISM messages; DID creation flows expect synchronous ledger writes (e.g., Indy NYM). | Registrar interfaces support asynchronous operations but rely on existing SDKs; no Atala object builder. | Must import or port Castor logic (TS) to Python/TypeScript agent contexts. |
| **Long-form DID support** – encode full initial state for offline use (`PRISM-method.md:66-90`). | DID Exchange now defaults to `did:peer:4` for offline contexts (`acapy/docs/features/QualifiedDIDs.md:5-38`), but there is no parser for PRISM long-form syntax. | Credo lacks a `did:prism` parser/resolver; `DidResolverService` will mark PRISM DIDs as unsupported (`packages/core/src/modules/dids/services/DidResolverService.ts:54-96`). | Implement parser + fallback to Castor’s `LongFormPrismDIDResolver` (`sdk-ts/src/castor/resolver/LongFormPrismDIDResolver.ts:23-178`). |
| **Rich DIDComm & LinkedDomains services embedded in ledger-backed docs** (`PRISM-method.md:83-140`). | Services for Indy DIDs depend on ledger endpoint attributes and are limited to DIDComm v1 contexts (`acapy/acapy_agent/resolver/default/indy.py:121-209`). | Peer and web DIDs can express DIDComm V2 services but not anchored to a ledger. | Need new mapping layer from PRISM services (DIDComm V2, LinkedDomains) into each agent’s DID document models and Aries protocol assumptions.

## Blockchain networks and configuration impact

| Agent | Current DID backends | Networks today | Configuration touch points | Changes once PRISM is added |
| --- | --- | --- | --- | --- |
| ACA-Py | Indy ledgers for `did:sov`/`did:indy` (`acapy/acapy_agent/resolver/default/indy.py:1-210`); HTTP for `did:web`/`did:webvh`; self-certifying (`did:key`, `did:jwk`); peer DID wallets (`docs/features/QualifiedDIDs.md:5-40`). | Sovrin, BCovrin, IDUnion, etc. plus arbitrary HTTPS hosts. | `ledger.*` settings, did-exchange flags, resolver config. | Add Cardano-specific config: Blockfrost project IDs (`sdk-ts/docs/prism/publishing-did.md:6-90`), Cardano-node endpoints or a NeoPRISM URL, Cardano wallet credentials, DBSync/oura options. Need new admin APIs for publishing PRISM transactions and referencing NeoPRISM/universal resolver endpoints. |
| Credo-TS | Multi-ledger: Indy (`did:sov`, `did:indy`), Cheqd, Hedera, HTTP methods, and peer/key/jwk (`credo-ts/README.md:41-58`). | Indy networks chosen by Indy VDR config; Cosmos-based Cheqd; Hedera DID service; HTTP(s). | `DidsModuleConfig` registrars/resolvers and method-specific module options. | Introduce Cardano environment config (Blockfrost key, Cardano node/gateway, NeoPRISM URL). Tenants will need to pick Cardano network per DID (mainnet/preprod) similar to how Indy namespaces are configured. |

Adding PRISM also links both agents to additional infrastructure: a Cardano node (or hosted provider), DBSync database, Cardano wallet for transaction submission, and a NeoPRISM deployment (possibly behind a gateway with Blockfrost, per `neoprism/docker/blockfrost-neoprism-demo/README.md:1-155`).

## Potential library and SDK work

| Language / stack | Proposed addition | Source to build from | Rationale |
| --- | --- | --- | --- |
| Python (ACA-Py) | `prism-did-client` module: secp256k1 key utilities, Atala protobuf serializers, and NeoPRISM REST client. | Port from SDK-TS Castor (`sdk-ts/src/castor/Castor.ts:42-220`) and NeoPRISM OpenAPI. | Avoid re-implementing Cardano logic inside ACA-Py services; keep operations testable outside the agent. |
| TypeScript (Credo) | `@credo-ts/prism` package exposing a registrar/resolver pair built atop Castor + NeoPRISM HTTP client. | Reuse Castor directly (already TS) plus DID resolver proxies (`sdk-ts/src/castor/resolver/PrismDIDResolver.ts:12-40`). | Separates PRISM dependencies (Mesh SDK, CIP-30 wallets) from Credo core for optional adoption. |
| Rust | Thin client inside NeoPRISM or a Rust crate for DID resolution/publishing consumed by both agents when running in standalone services. | Existing NeoPRISM crates implement full node functionality (`neoprism/README.md:18-180`). | Could expose high-performance PRISM operations to other ecosystems (e.g., Python via FFI) later on.

These reusable libraries reduce duplication, centralise Cardano-specific signing logic, and make it easier to keep up with future revisions of the PRISM spec.
