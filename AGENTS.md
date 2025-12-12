# AGENTS.md - Module Architecture & AI Agent Guidance
**Arweave + Witnet Authenticity Platform - Phase 1, 2, and 3**

---

## Overview

This document defines module boundaries, responsibilities, and AI agent guidance for all three phases of the ArweaveStamp platform. It serves as the single source of truth for architectural decisions and developer guidance.

---

## Part 1: High-Level Architecture (All Phases)

### Three-Layer Architecture

```
┌─────────────────────────────────────────────────┐
│ User Interface Layer                            │
├─────────────────────────────────────────────────┤
│ Phase 1: CLI (Node.js yargs)                    │
│ Phase 2: Web Dashboard (React + Express.js)    │
│ Phase 3: Monitoring UI (Grafana/custom portal) │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│ Application/Business Logic Layer                │
├─────────────────────────────────────────────────┤
│ Phase 1: Attestation orchestration              │
│ Phase 2: Proof search/verification              │
│ Phase 3: Policy engine, audit trail             │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│ Integration Layer                               │
├─────────────────────────────────────────────────┤
│ • Storage: Arweave                              │
│ • Analysis: Claude                              │
│ • Oracle: Witnet                                │
│ • Chains: Ethereum, Polygon, Avalanche         │
│ • Filesystem: Chokidar (Phase 3)               │
└─────────────────────────────────────────────────┘
```

### Data Flow Diagram

```
Document Upload (Phase 1)
  ↓
[Storage Layer] → Arweave storage
  ↓
[Analysis Layer] → Claude classification + entity extraction
  ↓
[Integrity Layer] → SHA-256 hash + documentId
  ↓
[Oracle Layer] → Witnet request + multichain relay
  ↓
[Proof Generation] → ProofPackage JSON
  ↓
Web Dashboard (Phase 2)
  ↓
File Monitoring (Phase 3)
  ↓
[Watcher] detects modification
  ↓
[Policies] trigger actions (reattestor, email, snapshot, quarantine)
  ↓
[Audit Trail] logs all events → Merkle tree → Witnet anchor
```

---

## Part 2: Module Boundaries (Phase 1)

### Phase 1 Modules (16 Weeks)

#### `storage/` – Persistent File Storage on Arweave

**Responsibility**: Handle Arweave wallet initialization, file uploads, and transactions.

**Files**:
- `wallet.ts` – Arweave wallet initialization, key management
- `uploader.ts` – File upload with retry logic, URL retrieval
- `validator.ts` – MIME type validation, file size checks
- `types.ts` – TypeScript interfaces

**Key Exports**:
```typescript
async function initializeWallet(privateKeyPath: string): Promise<ArweaveWallet>
async function uploadFile(filePath: string, wallet: ArweaveWallet): Promise<{ url: string; txId: string }>
function validateFileType(filePath: string): boolean
interface ArweaveConfig { endpoint: string; network: 'mainnet' | 'testnet' }
```

**Dependencies**: `arweave`, `fs`, `path`

**Testing**: Unit tests with mocked Arweave API, optional integration tests on testnet

**Performance Target**: Upload 100MB file in <30 seconds

---

#### `analysis/` – Document Analysis via Claude

**Responsibility**: Send documents to Claude, extract classification, entities, summary.

**Files**:
- `client.ts` – Anthropic SDK initialization, rate limiting
- `prompts.ts` – Prompt templates (classification, entity, summary, vision)
- `extractors.ts` – Parse Claude JSON responses, validation
- `vision.ts` – PDF/image encoding, base64 handling
- `types.ts` – Response interfaces

**Key Exports**:
```typescript
async function analyzeDocument(content: string, base64Image?: string): Promise<AnalysisResult>
interface AnalysisResult {
  classification: ClassificationResult;
  entities: EntityResult;
  summary: SummaryResult;
  visionData?: VisionResult;
}
```

**Dependencies**: `@anthropic-ai/sdk`, `pdf-parse`

**Testing**: Unit tests with mocked Claude API, integration tests with real Claude (uses quota)

**Performance Target**: Full analysis <10 seconds per document

**PII Handling**: Mask SSNs, credit cards before sending to Claude

---

#### `integrity/` – Cryptographic Hashing & Proof of Authenticity

**Responsibility**: Generate deterministic hashes, assemble documentId from hashes.

**Files**:
- `hasher.ts` – SHA-256 hashing, canonical form
- `assembler.ts` – Combine hashes to create documentId
- `canonicalize.ts` – JSON canonicalization (sort keys, remove whitespace)
- `types.ts` – Hash types

**Key Exports**:
```typescript
function hashDocument(content: Buffer): string
function hashMetadata(analysis: AnalysisResult): string
function assembleDocumentId(docHash: string, metaHash: string): string
async function verifyDocumentHash(documentId: string, content: Buffer): Promise<boolean>
```

**Dependencies**: `crypto` (Node.js built-in)

**Testing**: 100% unit test coverage, determinism verification (same input → same hash)

**Performance Target**: Hash 1GB file in <500ms

**Cryptographic Guarantees**: SHA-256 provides 256-bit security; collision probability negligible

---

#### `oracle/` – Witnet Oracle Integration

**Responsibility**: Submit HTTP GET/POST tasks to Witnet, track request lifecycle, relay to L1 chains.

**Files**:
- `client.ts` – Witnet HTTP API client
- `requests.ts` – Construct HTTP GET/POST tasks
- `multichain.ts` – Track L1 confirmations (Ethereum, Polygon, Avalanche)
- `verification.ts` – Validate Witnet reports
- `types.ts` – Request/response types

**Key Exports**:
```typescript
async function submitWitnetRequest(
  documentId: string,
  arweaveUrl: string
): Promise<{ requestId: string; taskId: string }>

async function waitForFinalization(
  requestId: string,
  maxWaitMs: number
): Promise<WitnetReport>

async function verifyL1Confirmations(
  report: WitnetReport,
  chains: string[]
): Promise<Record<string, { txHash: string; confirmed: boolean }>>
```

**Dependencies**: `axios` (HTTP), `ethers.js` (L1 verification)

**Testing**: Unit tests with mocked Witnet API, integration tests on Witnet testnet

**Performance Target**: Request submission <2 seconds, finalization <5 minutes

**Supported Chains** (Phase 1): Ethereum, Polygon, Avalanche minimum; roadmap for more

---

#### `proofs/` – Proof Package Assembly & Serialization

**Responsibility**: Assemble ProofPackage from all layers, verify, store, export.

**Files**:
- `generator.ts` – Assemble ProofPackage from attestation layers
- `store.ts` – Store proofs locally (file-based + optional DB)
- `serializer.ts` – Export to JSON, PDF, human-readable format
- `types.ts` – ProofPackage interface

**Key Exports**:
```typescript
interface ProofPackage {
  id: string;
  documentId: string;
  timestamp: ISO8601;
  arweave: { url: string; txId: string };
  claude: AnalysisResult;
  integrity: { docHash: string; metaHash: string };
  witnet: { requestId: string; report: WitnetReport };
  multichain: Record<string, L1Confirmation>;
}

async function generateProof(
  documentId: string,
  allLayers: ProofLayers
): Promise<ProofPackage>

async function verifyProof(proof: ProofPackage): Promise<boolean>

async function exportProof(
  proof: ProofPackage,
  format: 'json' | 'pdf' | 'txt'
): Promise<string>
```

**Dependencies**: `pdfkit` (PDF export), `json-stringify-deterministic`

**Testing**: Unit tests for assembly + verification, integration tests for export formats

---

#### `cli/` – Command-Line Interface

**Responsibility**: Parse user commands, orchestrate modules, provide feedback.

**Files**:
- `index.ts` – Command router (yargs setup)
- `commands/init.ts` – Setup credentials
- `commands/stamp.ts` – Single file stamping
- `commands/batch.ts` – Batch file processing
- `commands/verify.ts` – Proof verification
- `commands/list.ts` – List proofs
- `commands/export.ts` – Export proof
- `utils/` – Logging, config management

**Key Commands** (Phase 1):
```bash
stamp init                              # Setup Arweave wallet + Claude key
stamp stamp <file>                      # Attest single document
stamp batch <directory>                 # Process all files in directory
stamp verify <proof.json>               # Verify proof
stamp list                              # Show all proofs
stamp export <documentId> --format pdf  # Export proof
```

**Dependencies**: `yargs`, `chalk` (colored output), `ora` (spinner)

**Testing**: Integration tests for each command with real file I/O

---

### Phase 1 Module Dependencies

```
Dependency Graph:
  storage/ ← (standalone, Arweave SDK)
  analysis/ ← (standalone, Claude SDK)
  integrity/ ← (standalone, crypto)
  oracle/ ← (standalone, Witnet + RPC)
  proofs/ ← depends on: storage, analysis, integrity, oracle
  cli/ ← depends on: proofs (and transitively all)
```

**No circular dependencies** ✅

---

## Part 3: Module Boundaries (Phase 2 - Web Dashboard)

### Phase 2 New Modules (Weeks 4-28)

#### `backend/` – Express.js REST API

**Responsibility**: Expose CLI functions via HTTP, database queries, authentication.

**Files**:
- `app.ts` – Express setup, middleware
- `routes/proofs.ts` – `/api/proofs/*` endpoints
- `routes/documents.ts` – `/api/documents/*` endpoints
- `middleware/auth.ts` – JWT/API key authentication
- `middleware/error.ts` – Error handling
- `db/` – Database models (Phase 2 uses SQLite, Phase 3+ PostgreSQL)
- `types.ts` – API request/response types

**Key Endpoints**:
```
GET /api/proofs                         # List all proofs
GET /api/proofs/:id                     # Get proof details
POST /api/proofs/search                 # Semantic search (Claude-powered)
GET /api/documents/:id/verify           # Verify document integrity
POST /api/documents/batch                # Batch upload + attest
```

**Dependencies**: `express`, `sqlite`, `jsonwebtoken`

---

#### `frontend/` – React Dashboard

**Responsibility**: Display proofs, search, verification portal, compliance reports.

**Files**:
- `components/ProofCard.tsx` – Proof display component
- `pages/ProofList.tsx` – Proof listing + search
- `pages/VerificationPortal.tsx` – Third-party verification
- `hooks/useProofSearch.ts` – Search logic
- `api/client.ts` – Backend communication
- `styles/` – Tailwind + CSS modules

**Key Features** (Phase 2):
- Proof search with filters (document type, date range, entity)
- Visual proof timeline (upload → analysis → oracle → confirmation)
- Verification portal for third parties (read-only)
- Compliance reports (aggregated summaries)

**Dependencies**: `react`, `axios`, `tailwindcss`

---

## Part 4: Module Boundaries (Phase 3 - File Monitoring)

### Phase 3 New Modules (Weeks 17-24)

#### `watcher/` – File System Monitoring Daemon

**Responsibility**: Watch files for modifications, compute hashes, detect tampering.

**Files**:
- `daemon.ts` – Main service loop, lifecycle management
- `monitor.ts` – Chokidar integration (cross-platform FS watching)
- `events.ts` – File event types, event queue
- `types.ts` – TypeScript interfaces
- `config.ts` – Configuration loader

**Key Exports**:
```typescript
async function startWatcher(dirPath: string, config: WatcherConfig): Promise<WatcherService>

interface FileEvent {
  type: 'create' | 'modify' | 'delete' | 'chmod' | 'chown';
  path: string;
  timestamp: ISO8601;
  previousHash?: string;
  currentHash?: string;
  sizeChange?: number;
  permissionChange?: boolean;
}

async function validateFileIntegrity(
  filePath: string,
  proofPackage: ProofPackage
): Promise<{ intact: boolean; reason: string }>
```

**Dependencies**: `chokidar`, `crypto`

**Performance Target**: Detection latency <500ms, <1% CPU overhead during idle

**Cross-Platform**: Linux (inotify via chokidar), macOS (FSEvents), Windows (polling)

**Testing**: Unit tests for event detection, integration tests on real filesystem

---

#### `policies/` – Smart Policy Engine

**Responsibility**: Execute policies in response to file events.

**Files**:
- `engine.ts` – Policy execution orchestrator
- `predicates.ts` – Condition evaluators (file:modified, file:deleted, etc.)
- `actions.ts` – Action implementations
  - `witnet-reattestor.ts` – Auto-reattestify via Witnet
  - `email-notifier.ts` – Send alerts
  - `snapshot-archiver.ts` – Create version snapshots
  - `quarantine.ts` – Secure isolation
- `registry.ts` – Plugin system for custom policies
- `types.ts` – Policy schema

**Key Exports**:
```typescript
interface Policy {
  id: string;
  name: string;
  enabled: boolean;
  trigger: { event: string; filePattern: string; timescale?: number };
  conditions?: { hashMismatch?: boolean; sizeChange?: string };
  actions: Array<{ type: string; config: Record<string, any> }>;
  escalation?: { afterAttempts: number; action: string };
}

async function executePolicies(
  fileEvent: FileEvent,
  applicablePolicies: Policy[]
): Promise<PolicyExecution[]>

async function registerCustomPolicy(
  policy: CustomPolicyDefinition
): Promise<void>
```

**Dependencies**: `nodemailer` (email), existing storage/oracle modules

**Sandboxing**: Custom policies run in isolated VM context, no filesystem access by default

**Testing**: Unit tests for each policy action, integration tests for trigger accuracy

**Acceptance Criteria**: Policies trigger correctly, actions execute in order, no side effects

---

#### `audit/` – Tamper-Proof Audit Trail

**Responsibility**: Log all events, create Merkle proofs, anchor to Witnet/Arweave.

**Files**:
- `logger.ts` – Append-only log writer
- `merkle.ts` – Merkle tree construction + verification
- `anchorer.ts` – Periodic anchoring to Witnet/Arweave
- `verifier.ts` – Independent log verification (no app access needed)
- `search.ts` – Audit log search/filter/export
- `types.ts` – Audit types

**Key Exports**:
```typescript
interface AuditEntry {
  timestamp: ISO8601;
  eventId: string;
  fileHash: string;
  event: { type: string; details: Record<string, any> };
  policy?: { id: string; actions: string[] };
  witnetProof?: WitnetProof;
  merkleRoot?: string;
  previousEntryHash: string; // Chain link
}

async function logAuditEvent(event: AuditEntry): Promise<void>

async function getMerkleProof(entryId: string): Promise<MerkleProof>

async function verifyAuditIntegrity(
  startEntry: AuditEntry,
  endEntry: AuditEntry
): Promise<boolean> // Without app access

async function searchAuditLog(
  query: AuditSearchQuery
): Promise<AuditEntry[]>
```

**Dependencies**: `merkletreejs`, existing oracle module for anchoring

**Anchoring Interval**: Every 15 minutes or 10K entries

**Immutability Guarantee**: Entries are append-only, Merkle chain prevents tampering

**Verification**: Anyone can verify using Witnet report + blockchain data

---

#### `agent/` – Distributed Monitoring Client

**Responsibility**: Lightweight daemon for multi-server deployments with central vault communication.

**Files**:
- `client.ts` – Agent main loop
- `comms.ts` – mTLS communication with central vault
- `vault.ts` – Configuration/credentials vault integration
- `types.ts` – Agent types

**Key Exports**:
```typescript
async function startAgent(config: AgentConfig): Promise<AgentService>

interface AgentConfig {
  vaultUrl: string;
  vaultCert: string;
  monitoredPaths: string[];
  agentId: string;
  policies: Policy[];
}

async function reportAuditEvent(event: AuditEntry): Promise<void>
```

**Dependencies**: `node-forge` (mTLS), existing watcher + audit modules

**Communication**: mTLS 1.3 to central vault, event streaming

**Deployment**: Docker container, Ansible playbook

**HA Setup**: Multiple agents per vault, failover on agent loss

---

### Phase 3 Module Dependencies

```
Dependency Graph:
  watcher/ ← (standalone)
  policies/ ← depends on: watcher, storage, oracle
  audit/ ← depends on: oracle
  agent/ ← depends on: watcher, policies, audit
  cli/ (extended) ← depends on: watch, policy, verify-integrity, audit commands
```

**No circular dependencies** ✅

---

## Part 5: AI Agent Development Guidance

### For GitHub Copilot / Claude AI Assistants

When building modules, AI agents should follow these conventions:

#### 1. TypeScript Strict Mode Mandatory
```typescript
// ✅ Required
interface ProofPackage {
  id: string;
  timestamp: ISO8601;
  arweave: { url: string; txId: string };
}

// ❌ Avoid
const proofPackage: any = {};
```

#### 2. Error Handling Pattern
```typescript
// ✅ Required
try {
  const result = await uploadToArweave(file);
} catch (error: any) {
  if (error.status === 413) {
    throw new FileTooLargeError("Max 100MB");
  }
  throw new ArweaveError(error.message);
}

// ❌ Avoid
try {
  const result = await uploadToArweave(file);
} catch (error) {
  console.log(error); // Silent failure
}
```

#### 3. Async/Await Pattern
```typescript
// ✅ Required
async function processFile(path: string): Promise<ProofPackage> {
  const content = await fs.promises.readFile(path);
  const hash = hashDocument(content);
  return { ...hash };
}

// ❌ Avoid
function processFile(path: string): void {
  fs.readFile(path, (err, data) => {
    // Callback hell
  });
}
```

#### 4. Testing Pattern (TDD)
```typescript
// ✅ Required: Write tests first
describe("hashDocument", () => {
  test("should return same hash for same input", () => {
    const hash1 = hashDocument(sampleContent);
    const hash2 = hashDocument(sampleContent);
    expect(hash1).toBe(hash2);
  });
});

// Then implement
function hashDocument(content: Buffer): string {
  return crypto.createHash("sha256").update(content).digest("hex");
}
```

#### 5. Documentation Pattern
```typescript
/**
 * Upload file to Arweave and return transaction details
 * @param filePath - Absolute path to file
 * @param wallet - Initialized Arweave wallet
 * @returns {Promise<{url, txId}>} - File URL and transaction ID
 * @throws {FileTooLargeError} - If file > 100MB
 * @throws {ArweaveError} - On network failure
 * @example
 *   const result = await uploadFile("/path/to/doc.pdf", wallet);
 *   console.log(result.url); // https://arweave.net/abc123...
 */
async function uploadFile(
  filePath: string,
  wallet: ArweaveWallet
): Promise<{ url: string; txId: string }>
```

#### 6. Environment Configuration Pattern
```typescript
// ✅ Required: Use environment variables
const config = {
  arweaveEndpoint: process.env.ARWEAVE_ENDPOINT || "https://arweave.net",
  claudeApiKey: process.env.CLAUDE_API_KEY,
  witnetEndpoint: process.env.WITNET_ENDPOINT || "https://dr-api.witnet.io",
};

if (!config.claudeApiKey) {
  throw new Error("CLAUDE_API_KEY environment variable required");
}
```

#### 7. Logging Pattern
```typescript
// ✅ Required: Use structured logging
logger.info("Document uploaded successfully", {
  documentId,
  arweaveUrl,
  fileSize,
  uploadTime: endTime - startTime,
});

logger.error("Upload failed", {
  documentId,
  error: error.message,
  attempt,
});
```

#### 8. Security Pattern
```typescript
// ✅ Required: No secrets in logs
logger.info("API call made", { endpoint: "/api/proofs" }); // ✅ Safe

// ❌ Never log secrets
logger.info("Arweave wallet", { privateKey }); // ❌ SECURITY RISK

// ✅ Mask sensitive data
const maskedKey = privateKey.slice(0, 8) + "...";
logger.info("Using wallet", { maskedKey });
```

#### 9. Validation Pattern
```typescript
// ✅ Required: Input validation
async function analyzeDocument(content: string): Promise<AnalysisResult> {
  if (!content || content.trim().length === 0) {
    throw new ValidationError("Document content cannot be empty");
  }
  if (content.length > 100_000) {
    throw new ValidationError("Document exceeds 100KB limit");
  }
  // Process
}
```

#### 10. Performance Pattern
```typescript
// ✅ Required: Performance monitoring
const startTime = performance.now();
const result = await process();
const duration = performance.now() - startTime;

logger.info("Process completed", {
  duration,
  warning: duration > 10_000 ? "Slow operation" : undefined,
});
```

---

## Part 6: Testing Guidelines for AI Agents

### Test Coverage Targets
- **Phase 1**: >80% overall, 100% for `integrity/` (crypto module)
- **Phase 2**: >80% overall, including integration tests
- **Phase 3**: >80% overall, including performance tests

### Test Organization
```
tests/
├── unit/
│   ├── storage.test.ts
│   ├── analysis.test.ts
│   ├── integrity.test.ts
│   ├── oracle.test.ts
│   └── proofs.test.ts
├── integration/
│   ├── storage-arweave.test.ts
│   ├── analysis-claude.test.ts
│   ├── oracle-witnet.test.ts
│   └── watcher-filesystem.test.ts
├── performance/
│   ├── hashing.perf.test.ts
│   ├── upload.perf.test.ts
│   └── watcher.perf.test.ts
└── security/
    ├── pii-masking.test.ts
    ├── api-auth.test.ts
    └── audit-integrity.test.ts
```

### Example Unit Test (TDD)
```typescript
describe("hashDocument", () => {
  it("should hash PDF content correctly", () => {
    const pdfBuffer = fs.readFileSync("./test-files/sample.pdf");
    const hash = hashDocument(pdfBuffer);
    expect(hash).toHaveLength(64); // SHA-256 hex
    expect(hash).toMatch(/^[a-f0-9]{64}$/);
  });

  it("should return same hash for identical content", () => {
    const content = Buffer.from("test data");
    const hash1 = hashDocument(content);
    const hash2 = hashDocument(content);
    expect(hash1).toBe(hash2);
  });

  it("should return different hash for different content", () => {
    const hash1 = hashDocument(Buffer.from("data1"));
    const hash2 = hashDocument(Buffer.from("data2"));
    expect(hash1).not.toBe(hash2);
  });
});
```

---

## Part 7: Code Review Checklist for AI Agents

When AI agents create PRs, human reviewers should verify:

- [ ] **TypeScript Strict Mode**: `noImplicitAny: true`, all types explicit
- [ ] **Error Handling**: All promises have `.catch()` or `try/catch`
- [ ] **Testing**: >80% coverage, TDD pattern followed
- [ ] **Documentation**: JSDoc comments on all exported functions
- [ ] **Security**: No hardcoded secrets, PII masked in logs
- [ ] **Performance**: Latency targets met (see each module section)
- [ ] **API Compatibility**: Doesn't break Phase 1/2/3 contract
- [ ] **Dependencies**: Only approved packages (see Part 5)
- [ ] **Circular Dependencies**: None (use `npm ls` to verify)

---

## Part 8: Approved Technologies & Versions

### Runtime
- **Node.js**: 20.x LTS minimum
- **TypeScript**: 5.3+, strict mode

### Core Dependencies (Phase 1)
- `arweave`: ^1.15.0
- `@anthropic-ai/sdk`: ^0.16.0
- `crypto`: Node.js built-in
- `yargs`: ^17.7.0

### Phase 2 Add-ons
- `express`: ^4.18.0
- `sqlite`: ^5.0.0
- `jsonwebtoken`: ^9.1.0
- `react`: ^18.2.0

### Phase 3 Add-ons
- `chokidar`: ^3.5.3
- `merkletreejs`: ^0.3.10
- `nodemailer`: ^6.9.0
- `node-forge`: ^1.3.0

### Development
- `jest`: ^29.7.0
- `ts-jest`: ^29.1.0
- `eslint`: ^8.50.0
- `prettier`: ^3.0.0

---

**End AGENTS.md**