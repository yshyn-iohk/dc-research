# Implementation plan – Credo-TS

## Phase breakdown

1. **Module scaffolding** – Introduce `PrismDidResolver` and `PrismDidRegistrar` types plus configuration knobs in `DidsModuleConfig`.
2. **Resolution path** – Integrate Castor’s resolver logic and NeoPRISM HTTP proxying so `did:prism` can be consumed anywhere `DidResolverService` is used.
3. **Publishing + Cardano connectivity** – Allow agents to create long-form DIDs, build Atala objects, and submit Cardano transactions through wallet connectors (e.g., Mesh SDK, Blockfrost, or server-side signers).
4. **Protocol & storage wiring** – Expose PRISM options in DID selection APIs, connection flows, and DIDComm document services; persist DID docs and tx metadata in `DidRepository` when helpful.
5. **Validation & rollout** – Extend the automation suite, documentation, and feature flags to cover PRISM-specific behaviour.

## Detailed tasks

### 1. Module scaffolding

- Define `PrismDidResolver`/`PrismDidRegistrar` in a new package (e.g., `@credo-ts/prism`) that implements Credo’s `DidResolver`/`DidRegistrar` interfaces (`credo-ts/packages/core/src/modules/dids/domain`).
- Extend `DidsModuleConfig` so integrators can append the PRISM resolver/registrar via module options while keeping `did:key`/`did:peer` always-on defaults (`packages/core/src/modules/dids/DidsModuleConfig.ts:1-110`).
- Add configuration schema for Cardano environment (network id, NeoPRISM URL, Blockfrost key, CIP-30 wallet picker) exposed through the Node/React Native transports.

### 2. Resolution path

- Wrap `sdk-ts` Castor’s resolver stack (which already supports short/long forms and HTTP proxying, `sdk-ts/src/castor/resolver/PrismDIDResolver.ts:12-40`) behind the Credo resolver interface. Castor already clones docs via long-form fallback, so reuse that logic directly.
- Inject the resolver via `DidResolverService`, ensuring caching semantics honour NeoPRISM’s TTL while bypassing cache for long-form-only DIDs (`packages/core/src/modules/dids/services/DidResolverService.ts:31-145`).
- Provide configuration to point Castor’s HTTP proxy at the deployed NeoPRISM endpoint or a gateway such as the Blockfrost demo (`neoprism/docker/blockfrost-neoprism-demo/README.md:83-155`).

### 3. Publishing + Cardano connectivity

- Build a registrar that:
  - Uses Castor’s `createPrismDID` and `createPrismDIDAtalaObject` helpers to produce long-form DIDs and serialized Atala objects (`sdk-ts/src/castor/Castor.ts:85-220`).
  - Accepts signing strategies (Mesh SDK + CIP30 wallet for browser, cardano-serialization-lib for Node, hardware wallet connectors) similar to the publishing example (`sdk-ts/docs/prism/publishing-did.md:6-190`).
  - Submits transactions via Blockfrost or a Cardano node/wallet backend, returning tx hash + DID state so controllers can poll for confirmation.
- Persist DID docs + metadata inside `DidRepository` when `persistDidDocument` is requested, enabling offline resolution for long-form DIDs and cache invalidation hooks (`packages/core/src/modules/dids/repository`).

### 4. Protocol & storage wiring

- Update `DidsApi` to surface `createPrismDid` helpers and to allow `did:prism` to be selected in DIDComm/OOB flows where controllers currently choose between `did:key`, `did:peer`, or `did:web`.
- Ensure `DidCommDocumentService` can map PRISM DIDComm services (which include DIDComm V2 service endpoints) into agent routing tables.
- Record txn hashes and Cardano network names in DID tags so multi-tenant deployments know which network a DID belongs to when reusing or rotating it.
- Support DID rotation/update once NeoPRISM exposes update/deactivate endpoints by invalidating caches through `DidResolverService.invalidateCacheForDid` (`packages/core/src/modules/dids/services/DidResolverService.ts:164-167`).

### 5. Validation & rollout

- Add unit tests for the resolver (mocking NeoPRISM responses and long-form decoding) and registrar (ensuring Atala object creation matches spec limits).
- Extend end-to-end demos (Node + React Native) to show PRISM DID issuance using the Mesh SDK wallet flow described in the docs.
- Document configuration and operational recipes alongside other method guides (Supported Features) describing infrastructure requirements (Blockfrost API key, NeoPRISM deployment, Cardano wallet) referencing the Blockfrost-NeoPRISM architecture for clarity (`neoprism/docker/blockfrost-neoprism-demo/README.md:1-155`).
- Ship with feature flags so integrators can pilot the capability in preprod Cardano networks before hitting mainnet.
