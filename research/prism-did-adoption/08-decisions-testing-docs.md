# Decisions, documentation, and testing backlog

## Decisions & tradeoffs that need alignment

1. **Cardano connectivity model** – Decide whether agents will talk directly to Blockfrost, managed Cardano nodes, or a sidecar service. The Blockfrost + NeoPRISM gateway pattern (`neoprism/docker/blockfrost-neoprism-demo/README.md:83-155`) is attractive, but some deployments may require air-gapped nodes.
2. **Where to host Castor logic** – Credo can import SDK-TS directly, but ACA-Py needs a Python equivalent. Options: port Castor, call out to a sidecar microservice, or invoke a WASM bundle. The choice affects performance, security, and upgrade cadence.
3. **Key custody** – Determine whether secp256k1 master keys live inside existing wallets (Askar, Credo KMS) or external HSMs. This impacts API design and compliance requirements.
4. **Default DID selection** – Should new invitations default to `did:prism` once support lands, or remain opt-in like `use_did_method` does now (`acapy/docs/features/QualifiedDIDs.md:15-38`)? Similar question for Credo’s DID APIs.
5. **Update/deactivate roadmap** – PRISM supports update/deactivate operations, but NeoPRISM submitter endpoints must be exposed and agents need to plumb rotation semantics. Clarify timelines so testing can cover the full lifecycle early.
6. **Universal resolver role** – Decide whether to rely on universal resolver adapters long-term or require direct NeoPRISM access. Universal resolvers add latency but simplify onboarding.

## Documentation tasks once `did:prism` ships

- New ACA-Py feature page (“Using PRISM DIDs”) covering configuration flags, required infrastructure, and admin API examples referencing the publishing flow from SDK-TS docs (`sdk-ts/docs/prism/publishing-did.md:6-190`).
- Update ACA-Py Qualified DIDs doc with `use_did_method="did:prism"` examples and DID Rotation guidance.
- Credo documentation additions under “Supported Features” describing how to enable the PRISM module, required environment variables (NeoPRISM URL, Blockfrost key), and example code for browser vs. server issuance.
- NeoPRISM operations guide (if not already published) tailored for ACA-Py/Credo consumers: backup/restore, scaling, health endpoints.
- Integration playbooks describing how to stitch Blockfrost, Cardano node, db-sync, and NeoPRISM together (diagram similar to the Blockfrost demo) so DevOps teams have a canonical reference.
- Release notes highlighting the new DID method, caveats (confirmation delays, fees), and migration advice for existing Indy/peer deployments.

## Testing strategy (unit, integration, e2e)

| Layer | ACA-Py focus | Credo-TS focus | Notes |
| --- | --- | --- | --- |
| **Unit tests** | `PrismDIDResolver` (long-form parsing, error paths), protobuf encoding helpers, secp256k1 key utilities, config parsing. | `PrismDidResolver` wrapper around Castor, registrar input validation, cache behaviour, environment config parsing. | Use fixtures derived from the PRISM spec example DID doc (`prism-did-method-spec/w3c-spec/PRISM-method.md:83-140`). |
| **Integration tests** | Admin API tests that create a PRISM DID, submit a mocked transaction (or fixture), and confirm resolver output matches NeoPRISM responses. Extend DID Exchange tests to accept `did:prism`. | Tests that spin up a mock NeoPRISM HTTP server + Castor to verify resolver fallback, plus tests that exercise registrar flows end-to-end with Mesh SDK wallet mocks. | Keep Cardano dependencies mocked in CI; real-network tests can run nightly in preprod. |
| **End-to-end (E2E)** | Run Aries Agent Test Harness scenarios where ACA-Py issues an invite using `did:prism` and another ACA-Py or Credo agent resolves it via NeoPRISM. Include DID Rotation once updates are supported. | Browser + Node demos that publish a DID through CIP30 wallets/Blockfrost, then resolve via NeoPRISM-backed resolver. Add regression tests for DIDComm flows (routing, connection reuse) using PRISM identifiers. | Reuse Blockfrost demo topology for staging to ensure all components (Cardano node, wallet, db-sync, NeoPRISM) are exercised together. |

Automated coverage should also assert observability hooks (webhooks, logs) so operators can trace Cardano transactions back to agent activities.
