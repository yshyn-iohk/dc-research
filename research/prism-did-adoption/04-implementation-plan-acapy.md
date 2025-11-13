# Implementation plan – ACA-Py

## Phase breakdown

1. **Analysis & modelling** – Extend ACA-Py’s DID metadata surfaces so `prism` is recognised with the right crypto and policy hooks.
2. **Resolution pathway** – Provide a resolver that knows how to call NeoPRISM/Blockfrost-backed infrastructure and decode long-form PRISM DIDs.
3. **Publishing & wallet integration** – Enable controllers to create/publish `did:prism` via admin APIs, including secp256k1 key management and Atala object signing.
4. **Protocol wiring & multi-tenancy** – Allow DID Exchange, Out-of-Band, and DID Rotation flows to request/use `did:prism` while respecting tenant boundaries and webhook semantics.
5. **Testing, docs, and rollout** – Add automated coverage, update docs, and gate rollout behind feature flags.

## Detailed tasks

### 1. Analysis & modelling

- Add a `PRISM = DIDMethod(...)` entry with key types `[secp256k1, ED25519, X25519]`, rotation enabled, and holder-defined DID policy so controllers can pre-provision long-form DIDs (`acapy/acapy_agent/wallet/did_method.py:18-148`).
- Extend `KeyTypes`/wallet crypto helpers with secp256k1 support (similar to how ED25519/P256 are handled) so Askar can store PRISM public/private keys.
- Update DID validation (`acapy/acapy_agent/wallet/did_parameters_validation.py:7-72`) to understand PRISM method names and enforce the JSON-form long-form syntax.

### 2. Resolution pathway

- Implement `PrismDIDResolver(BaseDIDResolver)` that:
  - Detects short vs. long form (split on the encoded state per `PRISM-method.md:66-90`).
  - Calls NeoPRISM’s DID endpoint (or any universal resolver pointing at NeoPRISM) similar to `UniversalResolver` (`acapy/acapy_agent/resolver/default/universal.py:31-126`).
  - Falls back to decoding the long-form initial state locally using Castor’s logic (port from `sdk-ts/src/castor/resolver/LongFormPrismDIDResolver.ts:23-178`).
- Register the resolver inside `DIDResolver` bootstrapping so `did:prism` is routed before the universal resolver (`acapy/acapy_agent/resolver/did_resolver.py:50-134`).
- Provide configuration keys for NeoPRISM URL, bearer token (if behind a gateway), and cache TTL matching how universal resolver settings are supplied.

### 3. Publishing & wallet integration

- Create a `PrismDidManager` mirroring `DidIndyManager` but generating Atala objects instead of NYMs: derive secp256k1 master keys, build protobuf `CreateDIDOperation`, hash+sign with SHA256/ECDSA (`acapy/acapy_agent/did/indy/indy_manager.py:1-78`, `prism-did-method-spec/w3c-spec/PRISM-method.md:20-210`).
- Wrap the manager behind admin APIs (`POST /did/prism/create`, `POST /did/prism/publish`) similar to `wallet/did/create` so controllers can specify services, additional verification keys, and Blockfrost/Cardano submission details (`acapy/acapy_agent/wallet/routes.py:508-760`).
- Integrate Cardano connectivity via one of:
  - A thin Python client that hits a Blockfrost-like API to submit metadata transactions (`sdk-ts/docs/prism/publishing-did.md:6-190`).
  - Or a sidecar command (CLI) invoked from ACA-Py that hands Atala objects to a publishing service.
- Store PRISM metadata (state hash, anchoring tx hash, long-form blob) inside wallet records so DID rotation can reference them later.

### 4. Protocol wiring & multi-tenancy

- Update Out-of-Band and DID Exchange controllers to accept `use_did_method="did:prism"` just like the existing peer DID switches (`acapy/docs/features/QualifiedDIDs.md:15-38`).
- Ensure PRISM DIDs participate in DID Rotation via the existing `/did-rotate/{conn_id}/rotate` flow; the resolver must be able to fetch updated DID docs after on-chain operations settle.
- Respect multitenant isolation by scoping NeoPRISM cache entries and Cardano credentials per tenant when `BaseMultitenantManager` is active (`acapy/acapy_agent/resolver/default/indy.py:155-206`).
- Emit webhooks when PRISM operations complete (created, published, transaction confirmed) so controllers can coordinate with Cardano wallets.

### 5. Testing, docs, rollout

- Add unit tests for the new resolver, DID validation, and proto encoding (mocking NeoPRISM responses and long-form payloads).
- Extend AATH or pytest integration suites with flows that use long-form PRISM DIDs locally while hitting a mocked NeoPRISM endpoint for short-form resolution.
- Document configuration and operational steps in ACA-Py docs (e.g., new "PRISM DID" feature page referencing infrastructure outlined in `neoprism/docker/blockfrost-neoprism-demo/README.md:83-155`).
- Ship behind a feature flag so deployments can enable PRISM in stages (lab → preprod → mainnet), and capture observability metrics (transaction IDs, NeoPRISM latency) for early adopters.
