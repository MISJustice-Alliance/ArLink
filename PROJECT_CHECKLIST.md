# PROJECT_CHECKLIST.md - Complete Development Roadmap
**Arweave + Witnet Authenticity Platform - Phase 1, 2, and 3**

---

## Pre-Launch: Team & Repository Setup

### Week 0 (Before Day 1)
- [ ] **Team Assignment**
  - [ ] Assign lead architect (review AGENTS.md, ARCHITECTURE.md)
  - [ ] Assign builder (Phase 1 CLI development)
  - [ ] Assign validator (testing, QA)
  - [ ] Assign ops (deployment, CI/CD)
  - [ ] Roles documented in CONTRIBUTING.md

- [ ] **Repository Setup**
  - [ ] GitHub repository created
  - [ ] Collaborators added
  - [ ] Branch protection rules enabled (main: require PR review)
  - [ ] .gitignore configured (.env, node_modules, dist)

- [ ] **Development Environment**
  - [ ] `.env.example` created with all required variables:
    - `ARWEAVE_ENDPOINT`
    - `ARWEAVE_WALLET_PATH`
    - `CLAUDE_API_KEY`
    - `WITNET_ENDPOINT`
    - `ETH_RPC_URL`, `POLYGON_RPC_URL`, `AVAX_RPC_URL`
  - [ ] Node.js 20.x LTS installed
  - [ ] npm 10.x installed
  - [ ] Docker (for Phase 3) optional but recommended

- [ ] **Documentation**
  - [ ] README.md committed (quick start + features)
  - [ ] DEVELOPMENT_PLAN.md committed
  - [ ] AGENTS.md committed
  - [ ] ARCHITECTURE.md committed
  - [ ] CLAUDE.md committed
  - [ ] SECURITY.md committed
  - [ ] CONTRIBUTING.md created

- [ ] **CI/CD Pipeline**
  - [ ] GitHub Actions workflow created (`.github/workflows/test.yml`)
    - [ ] Run `npm install`
    - [ ] Run `npm run lint`
    - [ ] Run `npm test`
    - [ ] Report coverage
  - [ ] Passing build required for PR merge

---

## Phase 1: CLI Tool Development (Weeks 1-16)

### Weeks 1-2: Foundation & Setup ✅

**Goal**: Repository initialized, CI/CD running, team onboarded

**Deliverables**:
- [ ] GitHub repository created ✅
- [ ] TypeScript + Jest configured
  - [ ] `tsconfig.json` with strict mode
  - [ ] `jest.config.js` configured
  - [ ] `.eslintrc.json` configured
  - [ ] `.prettierrc.json` configured
- [ ] Package.json with all Phase 1 dependencies:
  ```json
  {
    "dependencies": {
      "arweave": "^1.15.0",
      "@anthropic-ai/sdk": "^0.16.0",
      "yargs": "^17.7.0",
      "axios": "^1.6.0"
    },
    "devDependencies": {
      "typescript": "^5.3.0",
      "jest": "^29.7.0",
      "ts-jest": "^29.1.0",
      "eslint": "^8.50.0",
      "prettier": "^3.0.0"
    }
  }
  ```
- [ ] npm scripts configured:
  ```json
  "scripts": {
    "build": "tsc",
    "start": "node dist/cli/index.js",
    "test": "jest",
    "test:coverage": "jest --coverage",
    "lint": "eslint src --ext .ts",
    "format": "prettier --write src",
    "cli": "ts-node src/cli/index.ts"
  }
  ```
- [ ] GitHub Actions CI workflow active
  - [ ] Build passes on clean install
  - [ ] All tests passing
  - [ ] Coverage reports generated

**Acceptance**: 
- `npm install` succeeds
- `npm test` passes
- `npm run lint` passes
- CI workflow green

---

### Weeks 3-4: Storage Layer (Arweave) ⏳

**Goal**: File upload to Arweave working end-to-end

**Deliverables**:
- [ ] `src/storage/wallet.ts` – Arweave wallet initialization
  - [ ] Load private key from file path
  - [ ] Validate wallet format
  - [ ] Initialize Arweave client
  - [ ] Unit tests (mocked Arweave)

- [ ] `src/storage/uploader.ts` – Upload with retry logic
  - [ ] Read file from filesystem
  - [ ] Validate file size (<100MB recommended)
  - [ ] Upload to Arweave with progress tracking
  - [ ] Handle retry on failure (exponential backoff)
  - [ ] Return URL + transaction ID
  - [ ] Unit tests with mocked uploads

- [ ] `src/storage/validator.ts` – MIME type validation
  - [ ] Validate file type (PDF, images, text)
  - [ ] Check file size limits
  - [ ] Validate file integrity
  - [ ] Unit tests for validators

- [ ] `src/storage/types.ts` – TypeScript interfaces
  - [ ] `ArweaveConfig` interface
  - [ ] `ArweaveWallet` type
  - [ ] `UploadResult` interface
  - [ ] Error types: `FileTooLargeError`, `InvalidMimeTypeError`

- [ ] Integration tests (optional, on testnet)
  - [ ] Real Arweave testnet upload
  - [ ] Verify URL accessibility
  - [ ] Cleanup test files

- [ ] Documentation
  - [ ] Comment all exported functions with JSDoc
  - [ ] Add to DEVELOPMENT_PLAN.md

**Acceptance**:
- `npm test -- storage.test.ts` passes
- Coverage >90% for storage module
- File uploads to Arweave testnet successfully
- `stamp storage-test <file>` command works (manual)

---

### Weeks 5-6: Analysis Layer (Claude) ⏳

**Goal**: Document analysis producing structured metadata

**Deliverables**:
- [ ] `src/analysis/client.ts` – Claude SDK setup
  - [ ] Initialize Anthropic client
  - [ ] Handle API key from environment
  - [ ] Implement rate limiting (exponential backoff)
  - [ ] Error handling for API failures
  - [ ] Unit tests with mocked Claude

- [ ] `src/analysis/prompts.ts` – Prompt templates
  - [ ] Classification prompt (document type, category, confidence)
  - [ ] Entity extraction prompt (persons, organizations, dates, amounts)
  - [ ] Summarization prompt (2-3 sentence abstract)
  - [ ] Vision prompt for PDF/image analysis
  - [ ] Helper functions to build prompts

- [ ] `src/analysis/extractors.ts` – Parse Claude responses
  - [ ] JSON parsing from Claude output
  - [ ] Validation of extracted data format
  - [ ] Type conversion + normalization
  - [ ] Error handling for malformed responses
  - [ ] Unit tests

- [ ] `src/analysis/vision.ts` – PDF/image handling
  - [ ] Base64 encoding for images
  - [ ] PDF page extraction (use pdf-parse or similar)
  - [ ] Send to Claude with vision capability
  - [ ] Handle multi-page documents
  - [ ] Unit tests

- [ ] `src/analysis/types.ts` – Response interfaces
  - [ ] `AnalysisResult` type
  - [ ] `ClassificationResult` type
  - [ ] `EntityResult` type
  - [ ] `SummaryResult` type
  - [ ] Error types

- [ ] Rate limit handling
  - [ ] Exponential backoff on 429 errors
  - [ ] Max retry attempts (3 default)
  - [ ] Logging of rate limit events

- [ ] Documentation
  - [ ] CLAUDE.md complete with prompt templates
  - [ ] PII handling documented
  - [ ] Cost estimation included

**Acceptance**:
- `npm test -- analysis.test.ts` passes
- Coverage >80% for analysis module
- Claude API calls successful (integration test on real API)
- Sample documents analyzed correctly
- CLAUDE.md matches actual implementation

---

### Weeks 7: Integrity Layer (Hashing) ⏳

**Goal**: Deterministic hashing for proof of authenticity

**Deliverables**:
- [ ] `src/integrity/hasher.ts` – SHA-256 hashing
  - [ ] Hash file content with SHA-256
  - [ ] Hash structured data (JSON)
  - [ ] Deterministic output (same input → same hash)
  - [ ] Handle large files efficiently
  - [ ] Unit tests (100% coverage required)

- [ ] `src/integrity/assembler.ts` – Combine hashes
  - [ ] Combine fileHash + metadataHash
  - [ ] Generate documentId (SHA-256 of combined hashes)
  - [ ] Verification of assembly
  - [ ] Unit tests

- [ ] `src/integrity/canonicalize.ts` – JSON canonicalization
  - [ ] Sort JSON keys alphabetically
  - [ ] Remove whitespace deterministically
  - [ ] Handle special types (dates, numbers)
  - [ ] Unit tests (100% coverage)

- [ ] `src/integrity/types.ts` – Hash types
  - [ ] `HashResult` type
  - [ ] `IntegrityProof` type

- [ ] Determinism verification
  - [ ] Tests: hash("content") === hash("content") on repeated calls
  - [ ] Tests: different content → different hashes
  - [ ] Cross-platform verification (Linux, macOS, Windows)

**Acceptance**:
- `npm test -- integrity.test.ts` passes with 100% coverage
- Hashes are deterministic
- Hash collision probability negligible (SHA-256: 2^256 operations)

---

### Weeks 8-10: Oracle Layer (Witnet) ⏳

**Goal**: Witnet oracle requests + multichain relay working

**Deliverables**:
- [ ] `src/oracle/client.ts` – Witnet HTTP API client
  - [ ] Initialize Witnet HTTP client
  - [ ] Handle connection + retries
  - [ ] Error handling for API failures
  - [ ] Rate limiting

- [ ] `src/oracle/requests.ts` – HTTP GET/POST task construction
  - [ ] Create HTTP task for Witnet witness network
  - [ ] Build request payload (GET Arweave URL)
  - [ ] Encode response expected (JSON)
  - [ ] Handle Witnet request submission
  - [ ] Track request ID

- [ ] `src/oracle/multichain.ts` – L1 tracking (Ethereum, Polygon, Avax)
  - [ ] Track Witnet report relay to L1 chains
  - [ ] Query Ethereum RPC for transaction confirmation
  - [ ] Query Polygon RPC for transaction confirmation
  - [ ] Query Avalanche RPC for transaction confirmation
  - [ ] Support 3+ L1 chains minimum
  - [ ] Wait for block confirmations (12+ blocks typical)

- [ ] `src/oracle/verification.ts` – Report validation
  - [ ] Verify Witnet report format
  - [ ] Validate report signature
  - [ ] Check report timestamp
  - [ ] Verify L1 confirmations

- [ ] `src/oracle/types.ts` – Request/response types
  - [ ] `WitnetRequest` type
  - [ ] `WitnetReport` type
  - [ ] `L1Confirmation` type
  - [ ] Error types

- [ ] Integration tests (on Witnet testnet)
  - [ ] Submit real request to Witnet testnet
  - [ ] Wait for finalization
  - [ ] Verify L1 relays
  - [ ] Cleanup test requests

**Acceptance**:
- `npm test -- oracle.test.ts` passes
- Integration test: Request → Finalization → L1 Relay successful
- Coverage >80% for oracle module
- Supports 3+ L1 chains

---

### Week 11: Proof Package & Verification ⏳

**Goal**: Assemble proofs from all layers; verify independently

**Deliverables**:
- [ ] `src/proofs/generator.ts` – Proof assembly
  - [ ] Combine storage + analysis + integrity + oracle data
  - [ ] Create ProofPackage JSON
  - [ ] Validate all required fields present
  - [ ] Unit tests

- [ ] `src/proofs/store.ts` – Persist proofs locally
  - [ ] Store ProofPackage to filesystem (JSON)
  - [ ] Store in database (sqlite for Phase 1)
  - [ ] Index by documentId for fast lookup
  - [ ] Unit tests

- [ ] `src/proofs/serializer.ts` – Export formats
  - [ ] JSON export
  - [ ] PDF export with human-readable format
  - [ ] Plain text export
  - [ ] Unit tests for each format

- [ ] `src/proofs/types.ts` – ProofPackage TypeScript
  - [ ] `ProofPackage` type (matches schema)
  - [ ] Validation functions

- [ ] Verification logic
  - [ ] Re-compute hashes against stored file
  - [ ] Verify Witnet finalization
  - [ ] Verify L1 confirmations
  - [ ] Integration tests

**Acceptance**:
- Proof created → Stored → Exported successfully
- Independent verification works without CLI access

---

### Weeks 12-13: CLI Commands ⏳

**Goal**: All CLI commands working with full workflows

**Deliverables**:
- [ ] `src/cli/index.ts` – Command router (yargs)
  - [ ] Parse command line arguments
  - [ ] Route to appropriate command handler
  - [ ] Global options (--config, --verbose)
  - [ ] Help text

- [ ] `src/cli/commands/init.ts` – Setup credentials
  - [ ] Prompt for Arweave wallet path
  - [ ] Prompt for Claude API key
  - [ ] Prompt for Witnet endpoint
  - [ ] Prompt for L1 RPC URLs
  - [ ] Validate credentials
  - [ ] Save to .env file
  - [ ] Integration test

- [ ] `src/cli/commands/stamp.ts` – Single file stamping
  - [ ] Accept file path argument
  - [ ] Upload to Arweave
  - [ ] Analyze with Claude
  - [ ] Compute hashes
  - [ ] Submit to Witnet
  - [ ] Wait for finalization
  - [ ] Generate ProofPackage
  - [ ] Save proof
  - [ ] Display results
  - [ ] Integration test

- [ ] `src/cli/commands/batch.ts` – Batch processing
  - [ ] Accept directory argument
  - [ ] Process all files in directory
  - [ ] Show progress bar
  - [ ] Handle errors per-file
  - [ ] Generate batch report
  - [ ] Integration test

- [ ] `src/cli/commands/verify.ts` – Proof verification
  - [ ] Accept proof JSON path
  - [ ] Accept file path to verify
  - [ ] Re-compute hashes
  - [ ] Check Witnet finalization
  - [ ] Check L1 confirmations
  - [ ] Display verification result
  - [ ] Integration test

- [ ] `src/cli/commands/list.ts` – List proofs
  - [ ] List all stored proofs
  - [ ] Show summary (ID, filename, date, status)
  - [ ] Filter options (--type, --since)
  - [ ] Integration test

- [ ] `src/cli/commands/export.ts` – Export proof
  - [ ] Accept proof ID + format
  - [ ] Export to JSON/PDF/TXT
  - [ ] Save to specified location
  - [ ] Integration test

- [ ] Help text + examples documented
- [ ] All commands tested manually

**Acceptance**:
- All commands functional with correct exit codes
- Help text accessible (`stamp --help`, `stamp stamp --help`)
- Integration tests for each command

---

### Weeks 14-15: Testing & Hardening ⏳

**Goal**: >80% coverage, error recovery, performance validated

**Deliverables**:
- [ ] Coverage targets
  - [ ] Generate coverage report: `npm run test:coverage`
  - [ ] Overall coverage: >80%
  - [ ] `integrity/` module: 100%
  - [ ] `storage/` module: >90%
  - [ ] `analysis/` module: >85%
  - [ ] `oracle/` module: >85%

- [ ] Error path testing
  - [ ] Network failure handling
  - [ ] Invalid file formats
  - [ ] Malformed API responses
  - [ ] Missing environment variables
  - [ ] Timeout scenarios
  - [ ] Each error path has test case

- [ ] Batch processing tests
  - [ ] Process 100+ files successfully
  - [ ] Handle mixed file types
  - [ ] Proper error reporting per-file
  - [ ] Performance acceptable

- [ ] Performance benchmarks
  - [ ] Single file: <60s end-to-end
  - [ ] Hash computation: <500ms (1GB)
  - [ ] Upload: <30s (100MB)
  - [ ] Claude analysis: <10s
  - [ ] Witnet finalization: <5 min average

- [ ] Security audit
  - [ ] No hardcoded secrets (grep for passwords, keys)
  - [ ] All input validated
  - [ ] No SQL injection risks (N/A for Phase 1 CLI)
  - [ ] PII handling correct (masked in logs)
  - [ ] Dependencies audited: `npm audit`
  - [ ] Zero critical vulnerabilities

**Acceptance**:
- Coverage report >80%
- All tests passing
- Security audit clean
- Performance benchmarks met

---

### Weeks 15-16: Documentation & Release ⏳

**Goal**: Ready for Phase 2 hand-off

**Deliverables**:
- [ ] README.md complete
  - [ ] Project overview
  - [ ] Quick start guide
  - [ ] Features list
  - [ ] Architecture diagram
  - [ ] Installation instructions
  - [ ] Usage examples (each CLI command)
  - [ ] Troubleshooting section

- [ ] ARCHITECTURE.md updated
  - [ ] Actual implementation details
  - [ ] Data flow diagrams
  - [ ] Database schema (Phase 1: SQLite prep)
  - [ ] API specifications (Phase 2 prep)

- [ ] SECURITY.md threats verified against code
  - [ ] Threat model documented
  - [ ] Mitigations implemented
  - [ ] No critical gaps
  - [ ] Incident response plan drafted

- [ ] CONTRIBUTING.md for Phase 2 developers
  - [ ] Development setup guide
  - [ ] Code style guidelines
  - [ ] Testing requirements
  - [ ] PR review process
  - [ ] Deployment process

- [ ] API_REFERENCE.md with all functions
  - [ ] Export every public function
  - [ ] JSDoc for all exports
  - [ ] Parameter types + descriptions
  - [ ] Return types + examples
  - [ ] Error types documented

- [ ] Troubleshooting guide
  - [ ] Common errors + solutions
  - [ ] FAQ
  - [ ] Debug mode (`VERBOSE=1`)
  - [ ] Log file locations

- [ ] Example proof packages
  - [ ] Real proofs generated
  - [ ] Verification walkthrough
  - [ ] Export examples (JSON, PDF, TXT)

- [ ] Release
  - [ ] Version 1.0.0 tagged in git
  - [ ] Changelog created (git log summary)
  - [ ] GitHub Release created
  - [ ] Release notes published

**Acceptance**:
- Phase 2 developers can onboard in <1 hour
- All documentation current + accurate
- Code comments match actual behavior
- Version 1.0.0 released

---

## Phase 2: Web Dashboard (Weeks 4-28)

### Weeks 4-6: Backend API Design ⏳
- [ ] Express.js setup + middleware
- [ ] REST endpoint design (GET /proofs, etc.)
- [ ] Authentication (JWT)
- [ ] Database schema (SQLite)
- [ ] API tests

### Weeks 7-12: React Dashboard ⏳
- [ ] Component library (React + Tailwind)
- [ ] Proof list page + search
- [ ] Verification portal (public)
- [ ] Compliance reports
- [ ] Unit + integration tests

### Weeks 13-14: Testing & Deployment ⏳
- [ ] End-to-end tests
- [ ] Docker image creation
- [ ] Deploy to staging
- [ ] Load testing

### Weeks 15-16: Documentation & Phase 2 Release ⏳
- [ ] API documentation (OpenAPI/Swagger)
- [ ] Deployment guide
- [ ] Version 2.0.0 released

---

## Phase 3: File Integrity Monitoring & Audit Trail (Weeks 17-24) ⭐ NEW

### Weeks 17-18: Watcher Daemon Core ⏳
- [ ] Chokidar integration (cross-platform file monitoring)
- [ ] FileEvent type + detection
- [ ] Hash comparison against ProofPackage
- [ ] Event queue + processing
- [ ] Performance: <500ms detection latency
- [ ] CPU overhead: <1% idle
- [ ] Unit tests + integration tests on filesystem
- [ ] Documentation: WATCHER.md

**Acceptance Criteria**:
- File modifications detected within 500ms
- Hash computed correctly
- Comparison against blockchain-anchored hash works
- Cross-platform support (Linux/macOS/Windows)

---

### Weeks 19-20: Smart Policy Engine ⏳
- [ ] Policy execution engine + predicate evaluation
- [ ] Built-in policy actions:
  - [ ] Witnet re-attestor (auto-reattestify on modification)
  - [ ] Email notifier (send diffs to admins)
  - [ ] Snapshot archiver (create Arweave backups)
  - [ ] Quarantine mode (move suspicious files)
- [ ] Plugin system for custom policies (sandboxed execution)
- [ ] Policy CRUD CLI commands
- [ ] Unit tests for each policy action
- [ ] Integration tests (policy trigger accuracy)
- [ ] Documentation: POLICIES.md with examples

**Acceptance Criteria**:
- Policies trigger correctly on file events
- Actions execute in correct order + complete
- Re-attestation happens within 2 seconds
- Email alerts sent with accurate diffs
- Snapshot archives created correctly
- Custom policies cannot escape sandbox

---

### Weeks 21-22: Tamper-Proof Audit Trail ⏳
- [ ] Append-only audit log writer
- [ ] Merkle tree construction + verification
- [ ] Periodic anchoring to Witnet/Arweave (15 min intervals)
- [ ] Independent audit log verification (no app access)
- [ ] Audit log search/filter/export
- [ ] CLI: `verify-integrity <file>` command
- [ ] CLI: `audit search` + `audit export` commands
- [ ] Unit tests for Merkle proofs (100% coverage)
- [ ] Integration tests for Witnet anchoring
- [ ] Documentation: AUDIT.md with verification examples

**Acceptance Criteria**:
- Audit logs immutable (append-only verified)
- Merkle roots anchor every 15 min consistently
- Independent verification works without app
- Search <100ms for 10K entries
- Export formats working (JSON/CSV/PDF)

---

### Weeks 23-24: Multi-Server Deployment & Hardening ⏳
- [ ] Lightweight agent daemon for distributed monitoring
- [ ] mTLS communication with central vault
- [ ] Docker image for agent deployment
- [ ] Ansible playbook for multi-server deployment
- [ ] Central configuration vault
- [ ] Secure agent-to-vault communication
- [ ] HA setup (multiple agents, failover)
- [ ] Security audit + threat model
- [ ] Performance testing: <2% CPU, <100MB RAM
- [ ] Documentation: AGENT_DEPLOYMENT.md

**Acceptance Criteria**:
- Agents communicate securely with vault
- Multi-server deployment working
- CPU <2%, RAM <100MB baseline
- Audit events propagate <5 seconds
- Failover works (agent loss tolerance)
- Security audit passed
- Load tested with 1000+ monitored files per server

---

### Phase 3 Success Metrics
- [ ] File modification detection: <5 seconds
- [ ] Smart policies execute correctly: <2s latency
- [ ] Audit logs anchor to Witnet: every 15 minutes
- [ ] Independent verification works
- [ ] Multi-server deployment functioning
- [ ] Zero unauthorized modifications undetected
- [ ] Performance <2% CPU overhead
- [ ] Security audit passed

---

## CI/CD Pipeline (All Phases)

### GitHub Actions Setup
- [ ] `.github/workflows/test.yml`
  - [ ] Run on: push to main/develop, PR
  - [ ] Build: `npm install`
  - [ ] Lint: `npm run lint`
  - [ ] Test: `npm test`
  - [ ] Coverage: `npm run test:coverage`
  - [ ] Report to Codecov (optional)

- [ ] `.github/workflows/security.yml` (Phase 3+)
  - [ ] Dependency audit: `npm audit`
  - [ ] SAST scanning (optional: Snyk)
  - [ ] Fail on critical vulnerabilities

- [ ] `.github/workflows/deploy.yml` (Phase 2+)
  - [ ] Build Docker image
  - [ ] Push to Docker Hub/ECR
  - [ ] Deploy to staging/production

---

## Team Synchronization

### Weekly Sync (30 min)
- [ ] Status updates (Phase 1/2/3 progress)
- [ ] Blockers + risks
- [ ] Next week priorities
- [ ] Decision items requiring team input

### Code Review Process
- [ ] All PRs require 1 approval before merge
- [ ] Tests must pass (CI green)
- [ ] Coverage >80% maintained
- [ ] Security: no hardcoded secrets, input validated

### Definition of Done
- [ ] Code written + committed
- [ ] Tests added (>80% coverage)
- [ ] Code review approved
- [ ] CI passing (build + test + lint)
- [ ] Documentation updated
- [ ] Performance benchmarks met (if applicable)
- [ ] Security audit passed (if applicable)

---

## Success Gates Between Phases

### Phase 1 → Phase 2 Gate
- [x] All CLI commands pass integration tests
- [x] Arweave uploads consistently work
- [x] Claude analysis produces correct metadata
- [x] Witnet requests finalize + L1 relay confirmations
- [x] Proof verification works end-to-end
- [x] >80% test coverage
- [x] Security audit passed
- [x] Documentation complete

### Phase 2 → Phase 3 Gate
- [ ] Web dashboard deployed + accessible
- [ ] Document search/filtering working
- [ ] Verification portal public
- [ ] Phase 1 & 2 integration tests passing
- [ ] Performance acceptable (<2s page load)
- [ ] Ready to monitor documents continuously

---

## Post-Launch Goals (Week 25+)

- [ ] Support 5+ L1 chains
- [ ] Batch process 100+ documents
- [ ] 99.5% uptime (Phase 2+)
- [ ] <100ms document lookup (Phase 2+)
- [ ] Community contributions (GitHub issues/PRs)
- [ ] Additional features (encryption, advanced policies)

---

**End PROJECT_CHECKLIST.md**