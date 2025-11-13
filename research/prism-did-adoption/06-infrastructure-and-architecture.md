# Infrastructure & architecture

## Required infrastructure components

| Component | Purpose | Source / reference |
| --- | --- | --- |
| **NeoPRISM node** | Indexes PRISM operations, exposes universal-resolver compatible API, optionally submits operations when run in submitter/standalone mode. Supports DBSync/Oura data sources and PostgreSQL state (`neoprism/README.md:18-68`). | Deployed per environment; can run behind a gateway as shown in the Blockfrost demo (`neoprism/docker/blockfrost-neoprism-demo/README.md:83-155`). |
| **Cardano data pipeline** | Cardano node (relay or local) + `cardano-wallet` + `db-sync` or third-party provider. Ensures DID operations can be published and later indexed. | Blockfrost + NeoPRISM demo illustrates how DBSync feeds both Blockfrost Ryo and NeoPRISM (`neoprism/docker/blockfrost-neoprism-demo/README.md:83-155`). |
| **Transaction submission API** | Either Blockfrost, Mesh SDK (client-side CIP30) or custom cardano-serialization-lib stack to post metadata with Atala objects (`sdk-ts/docs/prism/publishing-did.md:6-190`). | Needed for both ACA-Py (server-side) and Credo (edge agents). |
| **SDK-TS Castor (or port)** | Encodes/decodes long-form DIDs and generates protobuf operations (`sdk-ts/src/castor/Castor.ts:42-220`). | Serves as reference implementation for other languages. |
| **Universal resolver / HTTP proxy** | Optional front door so agents can consume PRISM DIDs before native resolvers ship. ACA-Py already ships an HTTP adapter (`acapy/acapy_agent/resolver/default/universal.py:31-126`). | Can front NeoPRISM or other hosted resolvers.

## Current-state architectures

### ACA-Py today

```mermaid
flowchart LR
    Controller[Admin client] -->|REST| ACApy[ACA-Py Admin API]
    ACApy --> Wallet[Askar wallet]
    ACApy --> DIDResolver
    DIDResolver -->|Indy NYM/ATTRIB| IndyLedger[(Indy Ledger Pools)]
    DIDResolver -->|HTTP| WebHosts[(did:web JSON)]
    DIDResolver --> PeerStore[(Peer DID Records)]
    DIDResolver --> Universal[Universal Resolver]
    Universal --> OtherMethods[(External DID methods)]
```

### Credo-TS today

```mermaid
flowchart LR
    App[Host app / agent] --> DidsModule[DidsModule]
    DidsModule --> Resolver[DidResolverService]
    DidsModule --> Registrar[DidRegistrarService]
    Resolver -->|Indy/Cheqd/Hedera SDKs| Ledgers[(Configured ledgers)]
    Resolver -->|HTTP| WebResolvers[(did:web/vh)]
    Resolver --> PeerCache[(DidRepository for peer/key)]
    Registrar --> KMS[KeyManagementApi]
    Registrar --> Ledgers
```

## Future-state with `did:prism`

### ACA-Py + PRISM

```mermaid
flowchart TB
    Controller --> ACApyPRISM[ACA-Py + PRISM feature flag]
    ACApyPRISM --> PrismManager[Prism DID Manager]
    PrismManager --> CastorPy[(Castor-derived proto helpers)]
    PrismManager -->|Atala object| CardanoTx[Cardano submission layer]
    CardanoTx --> BlockfrostAPI[(Blockfrost or Cardano wallet)]
    BlockfrostAPI --> CardanoNode[(Cardano node)]
    CardanoNode --> DBSync[(db-sync/Oura)]
    DBSync --> NeoPRISM
    NeoPRISM --> ACApyResolver[Prism DID Resolver]
    ACApyResolver --> DIDResolver
    DIDResolver --> Connections[DID Exchange / Routing]
```

### Credo-TS + PRISM

```mermaid
flowchart TB
    App --> CredoPrism[DidsModule + Prism package]
    CredoPrism --> PrismRegistrar
    CredoPrism --> PrismResolver
    PrismRegistrar --> CastorTS[(SDK-TS Castor)]
    PrismRegistrar -->|metadata tx| MeshWallet[CIP-30 wallet / Blockfrost client]
    MeshWallet --> CardanoNode
    CardanoNode --> DBSync
    DBSync --> NeoPRISM
    PrismResolver --> NeoPRISM
    PrismResolver --> CastorTS
    NeoPRISM --> CredoCache[DidRepository + cache]
```

Both diagrams explicitly include Blockfrost API, Cardano node, wallet, db-sync, and NeoPRISM per the integration expectations.

## Sequence diagrams

### Publishing a PRISM DID from ACA-Py

```mermaid
sequenceDiagram
    participant Ctrl as Controller
    participant ACA as ACA-Py
    participant PM as Prism DID Manager
    participant BF as Blockfrost/Cardano API
    participant CDN as Cardano Node/DBSync
    participant NEO as NeoPRISM

    Ctrl->>ACA: POST /did/prism/create (services, keys)
    ACA->>PM: request DID creation
    PM->>PM: build long-form state, Atala object (Castor port)
    PM->>BF: submit metadata tx (signed secp256k1)
    BF-->>PM: tx hash
    PM-->>ACA: DID (long-form) + tx hash
    CDN-->>NEO: new block with metadata
    NEO-->>ACA: DID doc available via resolver
    ACA-->>Ctrl: webhook (published + resolved short form)
```

### Resolving a PRISM DID in Credo-TS

```mermaid
sequenceDiagram
    participant App as Host application
    participant CRS as Credo DidsModule
    participant PR as PrismDidResolver
    participant NEO as NeoPRISM
    participant CAS as Castor

    App->>CRS: resolveDidDocument(did:prism:...)
    CRS->>PR: parse DID, check cache
    PR->>NEO: GET /dids/{short}
    alt anchored DID found
        NEO-->>PR: DID Document
    else fallback
        PR->>CAS: decode long-form state
        CAS-->>PR: DID Document
    end
    PR-->>CRS: didDocument + metadata (cache TTL)
    CRS-->>App: DID Document (servedFromCache flag)
```

These flows illustrate how Blockfrost, Cardano nodes, db-sync, and NeoPRISM cooperate with the agents and their new components.
