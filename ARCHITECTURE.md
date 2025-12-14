# ARCHITECTURE.md - System Design & Data Structures
**Arweave + Chainlink Authenticity Platform - Complete System**

---

## Executive Summary

ArweaveStamp is a three-phase document authenticity platform that evolves from point-in-time attestation (Phase 1-2) to continuous lifecycle verification (Phase 3).

- **Phase 1 (Weeks 1-16)**: Upload documents → Arweave → Claude analysis → Chainlink oracle → Proof
- **Phase 2 (Weeks 4-28)**: Web dashboard for proof management and verification
- **Phase 3 (Weeks 17-24)**: File monitoring daemon with smart policies and audit trails

---

## Part 1: Phase 1 Architecture (Attestation)

### Sequence Diagram: Single Document Attestation

```
User                 CLI            Storage         Analysis        Oracle          Proof
  │                   │               │                │               │              │
  │ stamp <file>      │               │                │               │              │
  ├────────────>│               │                │               │              │
  │                   │ read file     │                │               │              │
  │                   ├─────────>│                │               │              │
  │                   │<─────────│ (hash+content) │               │              │
  │                   │               │                │               │              │
  │                   │ upload + attest│               │               │              │
  │                   ├──────────────────>│               │              │
  │                   │<──────────────────│ (classification)│              │
  │                   │               │                │               │              │
  │                   │ compute SHA-256               │               │              │
  │                   ├─────────>│               │               │              │
  │                   │ + documentId  │               │               │              │
  │                   │<─────────│               │               │              │
  │                   │               │ submit to Chainlink│               │
  │                   │               │                ├──────────>│              │
  │                   │               │                │<────────── attestationId   │
  │                   │               │                │               ├─────>│
  │                   │               │                │               │ generate proof│
  │                   │               │                │               │<─────│
  │ ProofPackage      │               │                │               │              │
  │<────────────│               │                │               │              │
  │ (JSON + stores)   │               │                │               │              │
```

### Data Structure: ProofPackage (Phase 1)

```typescript
interface ProofPackage {
  // Metadata
  id: string;                               // UUID
  documentId: string;                       // SHA-256(content) + SHA-256(metadata)
  uploadedAt: ISO8601;
  fileName: string;
  fileSize: number;

  // Phase 1: Storage Layer
  storage: {
    provider: 'arweave';
    url: string;                            // https://arweave.net/{txId}
    txId: string;                           // Arweave transaction ID
    contentType: string;                    // e.g., application/pdf
    fileHash: string;                       // SHA-256(file content)
  };

  // Phase 1: Analysis Layer
  analysis: {
    classification: {
      documentType: 'contract' | 'invoice' | 'certificate' | /* ... */;
      category: 'legal' | 'financial' | 'hr' | /* ... */;
      subCategory: string;
      confidenceScore: number;              // 0.0-1.0
    };
    entities: {
      persons: Array<{ name: string; role: string }>;
      organizations: Array<{ name: string }>;
      dates: Array<{ date: ISO8601; event: string }>;
      amounts: Array<{ value: number; currency: string }>;
      keyTerms: string[];
    };
    summary: string;                        // Human-readable abstract
    extractionConfidence: number;
  };

  // Phase 1: Integrity Layer
  integrity: {
    documentHash: string;                   // SHA-256(file)
    metadataHash: string;                   // SHA-256(analysis JSON)
    combinedHash: string;                   // SHA-256(documentHash + metadataHash)
    hashAlgorithm: 'SHA-256';
    canonicalization: 'deterministic-json'; // Ensures reproducibility
  };

  // Phase 1: Oracle Layer (Chainlink)
  chainlink: {
    attestationId: string;                  // Unique attestation identifier
    sourceChain: 'ethereum';                // Primary chain
    ccipMessageId: string;                  // Cross-chain message ID
    multichain: {
      [chain: string]: {                    // e.g., 'ethereum', 'polygon', 'avalanche'
        blockNumber: number;
        transactionHash: string;
        blockTimestamp: ISO8601;
        confirmations: number;
        confirmed: boolean;
      };
    };
  };

  // Cryptographic Proof
  proof: {
    merkleRoot: string;                     // Root of hash chain
    proofChain: string[];                   // Hash chain for verification
  };

  // Verification Status
  verification: {
    arweaveVerified: boolean;
    hashesVerified: boolean;
    chainlinkFinalized: boolean;
    multiChainConfirmed: Record<string, boolean>;
    overallStatus: 'verified' | 'pending' | 'failed';
  };
}
```

### Data Flow: File → Hash → Chainlink → Blockchains

```
Original File (PDF, Image, etc.)
  │
  ├─> SHA-256(content) = fileHash
  │
  ├─> Upload to Arweave (immutable storage)
  │   └─> Arweave URL + txId
  │
  ├─> Send to Claude for analysis
  │   └─> Classification, entities, summary
  │   └─> SHA-256(analysis JSON) = metadataHash
  │
  ├─> Combine: SHA-256(fileHash + metadataHash) = documentId
  │
  ├─> Submit to Chainlink:
  │   └─> Send arweaveUrl + documentId to Ethereum router
  │   └─> Chainlink processes attestation
  │   └─> CCIP relays to Polygon + Avalanche automatically
  │
  └─> Generate ProofPackage
      └─> Verifiable: Arweave + Claude + Chainlink + 3 L1 chains
```

### Cryptographic Guarantees (Phase 1)

| Layer | Hash Algorithm | Security Level | Immutability |
|-------|----------------|----------------|--------------|n| File Integrity | SHA-256 | 256-bit (2^256 work to collision) | Once hashed, any change detected |
| Metadata | SHA-256 | 256-bit | Claude analysis snapshot |
| DocumentId | SHA-256(hash1 + hash2) | 512-bit effective | Combines file + metadata |
| Arweave Storage | Bundler fees (PoW) | Requires re-mining blocks | Transaction finalized in ~30 min |
| Chainlink Oracle | Multiple witness nodes | Chainlink DON consensus | Report finalized after CCIP relay |
| L1 Blockchains | PoW (Eth/Avax) / PoS (Polygon) | Network consensus | After 12+ block confirmations |

---

## Part 2: Phase 2 Architecture (Web Dashboard)

### System Diagram: Phase 1 + Phase 2

```
┌──────────────────────┐
│ Phase 1: CLI (Still Running)      │
│ ├─ stamp command                │
│ ├─ verify command               │
│ └─ batch processing             │
└────────────┬──────────┐
                     │ (Both write to same database)
                     ▼
        ┌────────────┐
        │ Proof Database       │
        │ (SQLite Phase 2)     │
        │ (PostgreSQL Phase 3) │
        └────────────┘
                     │
    ┌────────────────┐
    │              │              │
    ▼              ▼              ▼
┌─────┐  ┌───────┐  ┌───┐
│Express│  │React      │  │CLI  │
│Backend│  │Dashboard  │  │    │
└─────┘  └───────┘  └───┘
```

### API Endpoints (Phase 2)

```
GET    /api/proofs
       Returns: [ ProofPackage, ... ]

GET    /api/proofs/:id
       Returns: ProofPackage (single)

POST   /api/proofs/search
       Body: { query: string }
       Returns: [ ProofPackage, ... ]
       (Claude-powered semantic search)

GET    /api/documents/:id/verify
       Returns: { verified: boolean, chain: BlockchainConfirmations }

POST   /api/documents/batch
       Body: FormData { files: File[] }
       Returns: { uploadJobId: string }

GET    /api/compliance/report
       Query: { startDate, endDate }
       Returns: { summary, risks, recommendations }

POST   /api/auth/login
       Body: { email, password }
       Returns: { token: JWT }

GET    /api/proofs/export/:id
       Query: { format: 'json' | 'pdf' | 'txt' }
       Returns: File download
```

### React Component Hierarchy

```
App
├─ AuthProvider (JWT context)
├─ ProofListPage
│  ├─ SearchBar (semantic search)
│  ├─ FilterPanel
│  │  ├─ DocumentTypeFilter
│  │  ├─ DateRangeFilter
│  │  └─ EntityFilter
│  └─ ProofList
│     └─ ProofCard (repeating)
│        ├─ ProofHeader (title, date)
│        ├─ Classification badge
│        ├─ VerificationStatus
│        └─ ExportButton
├─ VerificationPortal (public, third-party)
│  └─ VerifyProofForm
│     ├─ FileUpload
│     ├─ ProofJsonUpload
│     └─ VerificationResult
└─ ComplianceReportPage
   └─ AggregatedSummary
      ├─ RiskMatrix
      └─ RecommendationsList
```

### Database Schema (Phase 2 - SQLite)

```sql
-- Proofs table
CREATE TABLE proofs (
  id TEXT PRIMARY KEY,
  documentId TEXT NOT NULL,
  fileName TEXT NOT NULL,
  fileSize INTEGER,
  uploadedAt DATETIME,
  fileHash TEXT,
  arweaveUrl TEXT,
  arweaveTxId TEXT,
  claudeSummary TEXT,
  documentType TEXT,
  confidence REAL,
  chainlinkAttestationId TEXT,
  chainlinkSourceChain TEXT,
  chainlinkFinalized BOOLEAN,
  ethereumTxHash TEXT,
  ethereumConfirmed BOOLEAN,
  polygonTxHash TEXT,
  polygonConfirmed BOOLEAN,
  avalancheTxHash TEXT,
  avalancheConfirmed BOOLEAN,
  createdAt DATETIME DEFAULT CURRENT_TIMESTAMP,
  updatedAt DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Entities table (for search indexing)
CREATE TABLE entities (
  id TEXT PRIMARY KEY,
  proofId TEXT NOT NULL,
  type TEXT NOT NULL, -- 'person', 'org', 'date', 'amount'
  value TEXT NOT NULL,
  context TEXT,
  FOREIGN KEY(proofId) REFERENCES proofs(id)
);

-- Search index
CREATE INDEX idx_proofs_documentType ON proofs(documentType);
CREATE INDEX idx_entities_value ON entities(value);
```

---

## Part 3: Phase 3 Architecture (File Monitoring & Audit)

### System Diagram: Phase 1 + 2 + 3

```
User's File System
  │
  ├─ /documents/contract.pdf (monitored by watcher daemon)
  │
  ├─> Watcher daemon detects modification
  │
  ├─> Policy engine evaluates:
  │   ├─ File modified (hash changed)?
  │   ├─ Policy: "CriticalDocs_Reattestor"?
  │   ├─ Time of modification (unusual?)
  │   └─ Size change?
  │
  ├─> Actions triggered:
  │   ├─ Chainlink re-attestation (automatic)
  │   ├─ Email alert to admin
  │   ├─ Snapshot to Arweave
  │   └─ Log to audit trail
  │
  └─> Audit trail entry created
      ├─ Timestamp
      ├─ File hash (before + after)
      ├─ Policy action
      ├─ Merkle root
      └─ Chainlink anchor (every 15 min)
```

### FileEvent Type (Phase 3)

```typescript
interface FileEvent {
  id: string;                               // UUID for this event
  type: 'create' | 'modify' | 'delete' | 'chmod' | 'chown';
  path: string;                             // Absolute file path
  timestamp: ISO8601;                       // When event occurred
  
  // Hash comparison
  previousHash?: string;                    // SHA-256(file before change)
  currentHash?: string;                     // SHA-256(file after change)
  
  // Metadata changes
  sizeChange?: number;                      // Bytes added/removed
  permissionChange?: boolean;               // chmod detected?
  ownerChange?: boolean;                    // chown detected?
  
  // Context
  detector: 'chokidar' | 'inotify' | 'fsevents';
  detectionLatency: number;                 // ms from event to detection
}
```

### Policy Type (Phase 3)

```typescript
interface Policy {
  id: string;                               // UUID
  name: string;                             // e.g., "CriticalDocs_Reattestor"
  enabled: boolean;
  
  // Trigger conditions
  trigger: {
    event: 'file:modified' | 'file:deleted' | 'file:accessed';
    filePattern: string;                    // Glob: **/*.pdf, /data/contracts/**
    timescale?: number;                     // ms debounce (coalesce events)
  };
  
  // Additional conditions
  conditions?: {
    hashMismatch: boolean;                  // Original hash changed?
    sizeChange?: 'increase' | 'decrease' | 'any';
    permissionChange?: boolean;
    timeWindow?: { from: string; to: string }; // e.g., "00:00"-"06:00" (unusual hours)
  };
  
  // Actions to execute
  actions: Array<{
    type: 'chainlink-reattestor' | 'email' | 'snapshot' | 'quarantine' | 'custom';
    enabled: boolean;
    config: Record<string, any>;            // Action-specific config
    timeout?: number;                       // Max execution time (ms)
  }>;
  
  // Escalation (if action fails)
  escalation?: {
    afterAttempts: number;                  // Retry N times
    action: string;                         // Fallback action
  };
  
  // Audit
  createdAt: ISO8601;
  createdBy: string;
  lastTriggered?: ISO8601;
  triggerCount: number;
}
```

### AuditEntry Type (Phase 3)

```typescript
interface AuditEntry {
  // Identity
  id: string;                               // UUID for this log entry
  timestamp: ISO8601;                       // When event logged
  eventId: string;                          // Links to FileEvent
  
  // What happened
  event: {
    type: 'file:modified' | 'policy:triggered' | 'reattested' | 'verified';
    details: Record<string, any>;           // Event-specific data
  };
  
  // File state
  fileHash: string;                         // SHA-256(file at this point)
  proofDocumentId: string;                  // Reference to original ProofPackage
  proofDocumentHash?: string;               // Original hash from ProofPackage
  
  // Policy execution
  policy?: {
    id: string;
    name: string;
    actions: Array<{
      type: string;
      status: 'success' | 'failed' | 'timeout';
      executionTime: number;                // ms
      error?: string;
    }>;
  };
  
  // Chainlink re-attestation (if triggered)
  chainlinkProof?: {
    attestationId: string;
    sourceChain: 'ethereum';
    finalized: boolean;
    l1Chains: { ethereum: tx, polygon: tx, avalanche: tx };
  };
  
  // Merkle chain
  merkleRoot?: string;                      // Merkle tree root at this entry
  previousEntryHash: string;                // SHA-256(previous entry) - forms chain
  
  // Verification status
  verification: {
    integrityVerified: boolean;             // Hash chain unbroken
    chainlinkVerified: boolean;             // Chainlink attestation confirmed
    verifiedAt?: ISO8601;
  };
}
```

### Merkle Tree Construction (Phase 3)

```
Audit Entries (chronological):
[Entry1] → [Entry2] → [Entry3] → ... → [EntryN]
   │         │         │         │        │
   ▼         ▼         ▼         ▼        ▼
 Hash1    Hash2    Hash3    Hash4 ...  HashN
   │         │         │         │        │
   └─────┬─────      └──┬──────        │
        │               │                  │
       H12             H34 ...                │
        │               │                  │
        └─────┬─────                  │
               H1234 ...                   │
                │                          │
                └──────┬──────────│
                          MerkleRoot
                    (Anchored to Chainlink
                     every 15 min / 10K entries)
```

**Properties**:
- Append-only: New entries always appended, never modified
- Tamper-proof: Changing any entry changes all downstream hashes
- Efficient verification: O(log N) to verify entry position
- Independent: Verifiable using only Merkle root + Chainlink data

### Audit Log Anchoring Strategy

```
Timeline:
├─ 00:00: Start collecting audit entries
├─ 15:15: 10,000 entries collected
│         ├─ Construct Merkle tree
│         ├─ Calculate root: abc123def456...
│         ├─ Submit to Chainlink: "Anchor: abc123def456..."
│         └─ Chainlink finalizes (CCIP relay to 3 chains)
│
├─ 15:16: Chainlink relay to L1 chains
│         ├─ Ethereum: tx 0xabc...
│         ├─ Polygon:  tx 0xdef...
│         └─ Avalanche: tx 0x123...
│
├─ 15:30: Anchor #2 collected (next 10K entries)
│         └─ Merkle root: xyz789...
│
└─ 16:00: Anchor #3 collected
          └─ Merkle root: ghi234...

Verification:
Entry N in interval 2:
  1. Find Entry N in audit log
  2. Calculate Merkle path to root (xyz789...)
  3. Verify root anchored to Chainlink in Ethereum/Polygon/Avalanche
  4. ✅ Entry verified as authentic
```

### Multi-Server Agent Architecture (Phase 3)

```
┌────────────────────────────────┐
│ Central Vault (Secure)                  │
│ ├─ Configuration storage                 │
│ ├─ Credential storage (encrypted)         │
│ ├─ Audit log collection                  │
│ └─ Merkle tree anchor coordination        │
└────────────────────┬────────────┘
         │                │                │
    (mTLS)           (mTLS)           (mTLS)
         │                │                │
    ┌───▼───┐     ┌───▼───┐    ┌───▼───┐
    │Agent 1   │     │Agent 2   │    │Agent 3   │
    │Server A  │     │Server B  │    │Server C  │
    │/data    │     │/data    │    │/data    │
    │watched  │     │watched  │    │watched  │
    └───┬───┘     └───┬───┘    └───┬───┘
         │                │               │
    File events     File events      File events
    audit events    audit events     audit events
         │                │               │
         └─────────┬──────────┬─────────┘
                          │
                  [Central Collection]
                          │
                  [Merkle Tree Construction]
                          │
                  [Chainlink/L1 Anchoring]
```

**Communication**:
- Agents → Vault: mTLS 1.3 (certificate-based auth)
- Vault → Agents: Configuration push (encrypted channels)
- Events: Real-time streaming with backpressure handling

---

## Part 4: Cross-Phase Data Consistency

### ProofPackage Evolution

```typescript
// Phase 1: ProofPackage created
{
  id: "proof-1",
  documentId: "abc123...",
  storage: { url, txId },
  analysis: { classification, entities },
  integrity: { fileHash, metadataHash },
  chainlink: { attestationId, sourceChain, multichain }
}

// Phase 3: Same ProofPackage enhanced with monitoring
{
  ...previousPhases,
  
  // NEW: Monitoring references
  monitoringStatus: {
    enabled: true,
    policies: ["CriticalDocs_Reattestor", "SecurityAlert"],
    lastChecked: ISO8601,
    integrityStatus: "verified"
  },
  
  // NEW: Re-attestations
  reattestations: [
    {
      timestamp: ISO8601,
      trigger: "file:modified",
      newHash: "xyz789...",
      chainlinkAttestationId: "attest-456..."
    }
  ],
  
  // NEW: Audit entries
  auditTrail: {
    totalEvents: 42,
    lastEventAt: ISO8601,
    merkleRoot: "def456...",
    anchors: [
      { timestamp, chainlinkAttestationId, l1: { ethereum: tx, polygon: tx } }
    ]
  }
}
```

---

## Part 5: Performance Targets & Scaling

### Phase 1 Benchmarks
| Operation | Target | Actual |
|-----------|--------|--------|
| File upload to Arweave | <30s (100MB) | TBD |
| Claude analysis | <10s per doc | TBD |
| Hash computation | <500ms (1GB) | TBD |
| Chainlink attestation | <5 min | TBD |
| L1 confirmation | <15 min | TBD |

### Phase 3 Benchmarks
| Operation | Target | Actual |
|-----------|--------|--------|
| File modification detection | <500ms | TBD |
| Policy trigger latency | <2s | TBD |
| Policy execution | <5s per action | TBD |
| Merkle tree construction | <2s (10K entries) | TBD |
| Audit log search | <100ms (10K entries) | TBD |
| CPU overhead (idle monitoring) | <2% | TBD |
| RAM overhead (baseline) | <100MB | TBD |

### Scalability Roadmap
- **Phase 1**: Single user, <100 documents
- **Phase 2**: Multiple users, <10K documents, SQLite
- **Phase 3**: Distributed agents, 100K+ documents, PostgreSQL
- **Post-Launch**: 1M+ documents, sharded database, multiple Chainlink nodes

---

## Part 6: Security & Trust Model

### Trust Assumptions
1. **Arweave Network**: Majority of nodes honest (PoW consensus)
2. **Chainlink DON**: Majority of witnesses honest (stake-based)
3. **L1 Blockchains**: Network consensus secure (Eth PoW, Polygon PoS, Avax PoW)
4. **Local System**: Filesystem watcher process not compromised
5. **User**: Provides correct private keys, doesn't share credentials

### Threat Model (Phase 3)

| Threat | Severity | Mitigation |
|--------|----------|------------|
| File modified locally (post-attestation) | Medium | Phase 3 monitoring detects + re-attests |
| Audit log tampering | High | Merkle chain prevents undetected changes |
| Policy engine compromise | High | Sandboxing + code review |
| mTLS certificate expiration | Medium | Automated rotation + monitoring |
| Chainlink witness collusion | Low | Majority assumption + L1 anchoring |
| Arweave storage loss | Low | Redundancy across multiple nodes |

---

**End ARCHITECTURE.md**
