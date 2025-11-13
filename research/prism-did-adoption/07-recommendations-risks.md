# Recommendations, rollout phases, and risks

## Recommended implementation approach (phases)

1. **Prototype with long-form only** – Reuse Castor’s decoding logic (TS) and port it to Python so both agents can at least consume long-form DIDs without Cardano infrastructure. This keeps initial work local and testable (`sdk-ts/src/castor/resolver/LongFormPrismDIDResolver.ts:23-178`).
2. **Introduce NeoPRISM-backed resolution** – Stand up a NeoPRISM indexer (or reuse the Blockfrost demo topology) per environment and point both agents’ resolvers at it (`neoprism/docker/blockfrost-neoprism-demo/README.md:83-155`). Ship read-only support first so dependencies can stabilise.
3. **Add publishing & transaction handling** – Once resolution is reliable, expose create/publish APIs leveraging Castor or its ports. Use the Mesh SDK guidance to integrate with wallets/Blockfrost (`sdk-ts/docs/prism/publishing-did.md:6-190`).
4. **Wire to connection protocols** – Allow DID Exchange / Out-of-Band to select `did:prism` and ensure DID Rotation works with the new resolver (`acapy/docs/features/QualifiedDIDs.md:15-49`, Credo’s `DidResolverService`).
5. **Harden + document** – Deliver doc updates, monitoring (tx hash logs, NeoPRISM latency), and enable A/B testing via feature flags before defaulting to PRISM in any flows.

This phased approach lets both projects land read support (low risk), then add write support once infrastructure and governance decisions are settled.

## Opinions & guidance

- **Lean on existing assets** – Avoid creating ad-hoc Cardano integrations; Castor (TS) and NeoPRISM already implement the spec. Wrapping them keeps both agents aligned with the evolving PRISM protocol and reduces cryptography risk.
- **Favour gateway deployments** – The Blockfrost + NeoPRISM demo shows how to expose DID resolution and Cardano APIs behind a single endpoint. Reusing that pattern simplifies SaaS offerings and minimises firewall changes.
- **Feature flag everything** – Both ACA-Py and Credo serve production workloads. Ship PRISM support disabled by default with explicit capability negotiation so existing ecosystems (Indy, peer) continue to interoperate.
- **Document end-to-end flows** – Publishing a DID now involves Cardano wallets, transaction fees, and confirmation polling. The SDK-TS guide is an excellent reference—mirror that level of detail in ACA-Py/Credo docs so adopters know what infrastructure is mandatory.

## Risk register

| Risk | Description | Severity | Mitigation |
| --- | --- | --- | --- |
| **Cardano infrastructure readiness** | Running a Cardano node + db-sync is heavier than Indy; Blockfrost quotas or network outages could block DID publishing/resolution. | High | Provide official guidance for hosted options (Blockfrost, managed NeoPRISM) and allow multiple endpoints with failover. Monitor latency and add retries. |
| **Crypto/key management gaps** | ACA-Py lacks secp256k1 support today; mishandling ECDSA keys could leak secrets or produce invalid signatures. | High | Port Castor’s key utilities, add extensive unit tests, and consider using audited libraries (e.g., `secp256k1` bindings). Keep keys in KMS where possible. |
| **Ledger finality delays** | PRISM requires 112 confirmations (~2 hours) for short-form DIDs. Agents might mis-handle the pending state. | Medium | Expose transaction status fields via APIs/webhooks so controllers know when a short-form DID is live. Continue to use long-form DID in the interim (per spec). |
| **NeoPRISM operational burden** | Indexers rely on DBSync and PostgreSQL; misconfiguration can yield stale DID docs. | Medium | Ship health checks, dashboards, and guidance on scaling/resilience. Lean on NeoPRISM’s OpenAPI health endpoint (`neoprism/docker/blockfrost-neoprism-demo/README.md:41-71`). |
| **Interoperability regressions** | Adding `did:prism` as a default might break ecosystems still on `did:peer` or Indy. | Medium | Keep DID selection explicit (similar to `use_did_method` today) until PRISM is widely supported; add feature detection to Out-of-Band invites. |
| **Spec evolution** | PRISM spec is still marked “first version” and may change (`prism-did-method-spec/w3c-spec/PRISM-method.md:1-44`). | Medium | Track spec revisions in the new libraries, centralise serialization code, and run conformance tests against NeoPRISM releases. |
