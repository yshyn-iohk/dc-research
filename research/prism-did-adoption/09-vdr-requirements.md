# VDR requirements and PRISM alignment

## ACA-Py expectations for a VDR

ACA-Py treats Verifiable Data Registries as pluggable destinations for AnonCreds objects. The maintainer guidance explicitly asks method implementers to ship both a registrar (write) and resolver (read) that plug into the AnonCreds framework, typically via an ACA-Py plugin (`docs/features/AnonCredsMethods.md:1-79`). Key implications:

- **Plugin contract** – A method registers identifier patterns during plugin initialization so ACA-Py can route AnonCreds operations to the right registrar/resolver pair (`docs/features/AnonCredsMethods.md:25-64`).
- **Write & read symmetry** – Each AnonCreds method must implement `BaseAnonCredsRegistrar` and `BaseAnonCredsResolver`, ensuring that schemas, credential definitions, revocation registries, and status lists can be stored and later resolved from the same VDR (`docs/features/AnonCredsMethods.md:48-64`).
- **Event plumbing** – Registrars must trigger `finish_*` events (e.g., revocation registry automation) so ACA-Py’s built-in workflows run regardless of the registry backend (`docs/features/AnonCredsMethods.md:68-79`).

**VDR requirement summary:** ACA-Py needs a driver that can persist AnonCreds artefacts, expose deterministic identifiers, support revocation artefacts (tails files, status lists), and provide read/verification APIs that can be wrapped by `BaseAnonCredsResolver`. The driver also needs to surface metadata (network, driver family/version) so plugins can serialize/deserialize identifiers consistently.

## Credo-TS expectations for a VDR

Credo abstracts VDR interactions through the `AnonCredsRegistry` interface. Registries announce a method name and regex for supported identifiers plus CRUD methods for schemas, credential definitions, revocation registries, and status lists (`credo-ts/packages/anoncreds/src/services/registry/AnonCredsRegistry.ts:22-64`). Notable requirements:

- **Consistent identifiers** – `supportedIdentifier` ensures drivers only claim IDs they can resolve; `methodName` is recorded alongside stored artefacts so agents can query by registry.
- **Full lifecycle operations** – Every registry must provide `register` and `get` APIs for each artefact type plus revocation status list updates, matching the expectations of `AnonCredsRegistryService` and higher-layer credential workflows.
- **Agent-context aware** – All methods accept an `AgentContext`, meaning drivers must respect per-tenant storage, caching, and logging policies when they interact with the underlying VDR.

**VDR requirement summary:** Credo expects a driver package that implements the entire `AnonCredsRegistry` contract, exposes deterministic identifiers, and handles revocation data at scale. It must integrate with existing caching and multi-tenant patterns through the provided agent context.

## What the PRISM VDR specification adds

The PRISM VDR spec formalizes a multi-layer architecture consisting of a VDR interface, Drivers, and URL Managers to decouple storage from URL construction (`vdr/README.md:51-120`). Important requirements to consider when wiring ACA-Py or Credo to PRISM’s VDR:

- **Driver obligations** – Drivers are responsible for storage, retrieval, update/delete, and proof generation. They expose a `store` API that returns an `OperationResult` (paths, query params, fragments) and a `verify` API that returns a cryptographic `Proof` (hashes for immutable data, signatures for mutable data) (`vdr/README.md:190-217`).
- **Driver families and metadata** – Drivers publish family (`drf`), identifier (`drid`), and version (`drv`) metadata so any compatible VDR instance can interpret URLs consistently. Mutable data must carry signatures and proof-type metadata, while immutable data uses SHA-256 hashes (`vdr/README.md:223-510`).
- **URL Manager contract** – After a store operation, the URL Manager takes the `OperationResult` plus driver metadata to construct a verifiable URL (which can use traditional schemes or the `vdr://` scheme) (`vdr/README.md:321-337`). Consumers resolve URLs back into driver instructions via the same component before calling `read`, `update`, `delete`, or `verify` (`vdr/README.md:339-420`).
- **Query & metadata registries** – The spec standardizes query parameters (`drf`, `drid`, `drv`, `h`, `s`, `pt`, `m`, etc.) and metadata keys to guarantee that any VDR-aware client can pick the correct driver and independently validate data integrity (`vdr/README.md:430-520`).

## Alignment and actionable requirements

| Need | ACA-Py | Credo-TS | VDR specification implications |
| --- | --- | --- | --- |
| **Identifier + driver metadata** | Plugins register identifier prefixes for routing (`docs/features/AnonCredsMethods.md:37-44`). | `methodName` & `supportedIdentifier` control routing (`AnonCredsRegistry.ts:22-33`). | PRISM VDR URLs must embed `drf/drid/drv`, so both agents need parsers/serializers that translate between AnonCreds identifiers and VDR URLs. |
| **Registrar/resolver contract** | `BaseAnonCredsRegistrar/Resolver` require deterministic store/read APIs for schemas, cred defs, revocation registries, and tails files. | `register*/get*` methods expect synchronous responses with identifiers and optional metadata. | PRISM drivers must support structured payloads (JSON/protobuf) and return `OperationResult` data that agents can map onto AnonCreds IDs; if storage is async, drivers must expose `storeResultState` so registrars can poll. |
| **Revocation artefacts** | ACA-Py automation needs to publish revocation registries + status lists and retrieve tails files later (`docs/features/AnonCredsMethods.md:68-79`). | `registerRevocationRegistryDefinition` and `registerRevocationStatusList` are first-class APIs (`AnonCredsRegistry.ts:44-63`). | PRISM VDR URLs and metadata must capture mutable artefacts (e.g., status lists). Drivers must surface signature-based proofs so verifiers can detect stale data when `m=1`. |
| **Verification** | ACA-Py resolvers need to verify proofs before trusting downloaded artefacts. | Credo registries may cache results but must re-validate when fetching from the VDR. | The VDR `verify` pathway (with `Proof` objects) gives agents a standard way to request hashes/signatures; integration layers should expose this as part of their resolvers to detect tampering. |
| **Multi-tenant / environment metadata** | ACA-Py stores ledger metadata alongside wallet entries; controllers need to know which registry a DID/AnonCreds object came from. | Credo tags records with `methodName` and network; registries can coexist. | VDR metadata registries should be persisted with AnonCreds records (e.g., driver family, version, content/media type) so agents can rehydrate the correct driver configuration per tenant. |

### Recommended next steps

1. **Define a PRISM-VDR registry package** for each agent that wraps a PRISM driver and URL manager. In ACA-Py this would be an AnonCreds plugin; in Credo it could live in a new `@credo-ts/prism-vdr` package implementing `AnonCredsRegistry`.
2. **Map AnonCreds identifiers to VDR URLs** by establishing naming conventions (e.g., embed `drf`, `drid`, and object-specific query params) so that registrars can return VDR URLs while resolvers can parse them and feed the metadata back into the PRISM driver.
3. **Leverage the VDR proof API** for verification hooks. Both agents should expose admin/SDK options to call `verify` (with `returnData=true` when needed) as part of their resolver flows so VDR proofs are enforced uniformly.
4. **Capture driver configuration per tenant/environment** by storing the PRISM metadata registry fields (driver family/id/version, mutable flag, proof type) inside each agent’s records. This keeps Cardano/PRISM deployment details decoupled from higher-level credential workflows.
