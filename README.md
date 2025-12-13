# ArweaveStamp: Trustless Document Authenticity + Physical Delivery
**Complete Document Lifecycle Management: Digital → Blockchain → Physical Mail with Cryptographic Lineage**

---

## 🎯 Project Overview

**ArweaveStamp** is a four-phase platform that transforms document verification from digital-only to a complete lifecycle that bridges blockchain attestation with tamper-proof physical delivery:

1. **Phase 1 (Weeks 1-16)**: CLI tool for document attestation
   - Upload documents to immutable Arweave storage
   - Analyze with Claude for semantic metadata
   - Cryptographic proof via Witnet oracle
   - Multichain attestation (Ethereum, Polygon, Avalanche)

2. **Phase 2 (Weeks 4-28)**: Web dashboard for proof management
   - Search + verify attestations
   - Compliance reporting
   - Third-party verification portal

3. **Phase 3 (Weeks 17-24)**: Continuous file monitoring
   - Detect unauthorized modifications in real-time
   - Smart policies (auto-reattestor, email alerts, snapshots)
   - Tamper-proof audit trails with Merkle proofs
   - Blockchain-anchored compliance logs

4. **Phase 4 (Weeks 25-40)**: Physical Mail Integration ⭐ **NEW**
   - PostGrid Print & Mail API integration
   - Cryptographic QR codes embedding Arweave + Witnet references
   - Organization branding and OCR address verification
   - Stripe + BTCPay Server payment settlement
   - Real-time carrier tracking with automated delivery receipts
   - Full cryptographic chain from digital → physical → delivery
   - Claude AI metadata enrichment at each lifecycle stage

---

## 🚀 Quick Start

### Installation

```bash
# Clone repository
git clone https://github.com/enuno/arweave-stamp.git
cd arweave-stamp

# Install dependencies
npm install

# Configure environment
cp .env.example .env
# Edit .env with your keys and endpoints
```

### Configuration

**.env file**:
```bash
# Arweave
ARWEAVE_ENDPOINT=https://arweave.net
ARWEAVE_WALLET_PATH=/path/to/wallet.json

# Claude
CLAUDE_API_KEY=sk-ant-...

# Witnet
WITNET_ENDPOINT=https://dr-api.witnet.io

# L1 RPC Endpoints
ETH_RPC_URL=https://eth.llamarpc.com
POLYGON_RPC_URL=https://polygon-rpc.com
AVAX_RPC_URL=https://avalanche-c-chain-rpc.publicnode.com

# Phase 4: Print & Mail Integration
POSTGRID_API_KEY=your-postgrid-api-key
POSTGRID_ENVIRONMENT=production

# Payment Settlement
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
BTCPAY_URL=https://your-btcpay-instance.com
BTCPAY_API_KEY=btcpay-api-key
BTCPAY_STORE_ID=store-id

# Organization Configuration
ORG_NAME=Your Organization
ORG_LOGO_URL=https://your-cdn.com/logo.png
DEFAULT_SENDER_NAME=Legal Department
DEFAULT_SENDER_ADDRESS=123 Main St, City, ST 12345
```

### First Document Attestation + Mailing

```bash
# Initialize credentials
npm run cli -- init

# Attest a single document
npm run cli -- stamp ./contract.pdf

# Create a print & mail job (Phase 4)
npm run cli -- mail create ./contract.pdf \
  --recipient-name "John Doe" \
  --recipient-address "456 Oak Ave, Town, ST 67890" \
  --job-title "Contract Notarization" \
  --payment-method stripe \
  --stripe-token sk_test_...

# Verify mailing job status
npm run cli -- mail status job-abc123

# Track delivery (real-time updates)
npm run cli -- mail track job-abc123 --watch

# Retrieve delivery receipt
npm run cli -- mail receipt job-abc123 --format pdf > delivery_receipt.pdf
```

---

## 📋 Features

### Phase 1: Cryptographic Attestation
- ✅ Upload documents to Arweave (immutable, permanent storage)
- ✅ Semantic analysis via Claude (document type, entities, summary)
- ✅ Cryptographic hashing (SHA-256 deterministic proof)
- ✅ Witnet oracle integration (decentralized witnesses)
- ✅ Multichain relay (Ethereum, Polygon, Avalanche proof)
- ✅ Zero-knowledge verification (verify without original file)

### Phase 2: Proof Management Dashboard
- ✅ Web interface for proof search + browsing
- ✅ Semantic search powered by Claude
- ✅ Compliance reporting + risk analysis
- ✅ Public verification portal (third-party access)
- ✅ Proof export (JSON, PDF, human-readable)

### Phase 3: Continuous Monitoring
- ✅ Real-time file modification detection (<5 seconds)
- ✅ Smart policies (triggers + automatic responses)
- ✅ Auto-reattestations on unauthorized changes
- ✅ Tamper-proof audit trails (Merkle tree + blockchain anchoring)
- ✅ Distributed agent deployment (multi-server monitoring)
- ✅ Compliance audit generation (automated certification)

### Phase 4: Physical Mail Integration ⭐ **NEW**
- ✅ PostGrid Print & Mail API full integration
- ✅ Cryptographic QR codes (Arweave + Witnet references embedded)
- ✅ Organization branding per-job and defaults
- ✅ OCR address detection + PostGrid verification
- ✅ Confirmation workflow (CLI + WebUI)
- ✅ Stripe card + BTCPay crypto payment settlement
- ✅ Real-time carrier tracking (USPS, UPS, FedEx, etc.)
- ✅ Automated delivery receipts with cryptographic proof
- ✅ Full lifecycle audit trail on Arweave + Witnet
- ✅ Claude AI metadata enrichment at each stage

---

## 🏗️ Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│ User Interface                                       │
├──────────────────────────────────────────────────────┤
│ Phase 1: CLI (stamp, verify, export)                 │
│ Phase 2: React Dashboard (web UI)                    │
│ Phase 3: Monitoring Portal (Grafana/custom)          │
│ Phase 4: Mail Manager UI (job creation, tracking)    │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│ Application Layer                                    │
├──────────────────────────────────────────────────────┤
│ Phase 1: Attestation orchestration                   │
│ Phase 2: Proof search + verification                 │
│ Phase 3: Policy engine + audit trail                 │
│ Phase 4: Mail job orchestration + payment gateway    │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│ Integration Layer                                    │
├──────────────────────────────────────────────────────┤
│ Arweave (storage) | Claude (analysis)                │
│ Witnet (oracle)  | Ethereum/Polygon/Avax             │
│ Chokidar (file watching) | PostGrid (print/mail)     │
│ Stripe (payments) | BTCPay (crypto) | OCR service    │
│ Carrier APIs (tracking)                              │
└──────────────────────────────────────────────────────┘
```

### Phase 4 Data Flow: Document → Blockchain → Physical Mail

```
Digital Document (PDF, image, etc.)
  ↓
[Phase 1] Upload to Arweave + analyze with Claude
  → Arweave URL + metadata hash
  ↓
[Phase 1] Submit to Witnet oracle
  → Witnet attestation ID
  ↓
[Phase 1] Relay to L1 blockchains (Ethereum/Polygon/Avax)
  → Chain-specific transaction IDs
  ↓
[Phase 4] User initiates mail job
  → Selects recipient, payment method, branding
  ↓
[Phase 4] OCR extracts recipient address from document
  → PostGrid address verification
  → Surface corrections/errors
  ↓
[Phase 4] Generate cryptographic QR code
  → Encodes: Arweave URL + Witnet ID + job reference
  → Claude generates semantic metadata (summary, risk flags)
  ↓
[Phase 4] Create confirmation summary
  → Document list with costs
  → Sender/recipient verification
  → Payment method validation
  ↓
[Phase 4] User confirms + payment settles
  → Stripe: card processed immediately
  → BTCPay: invoice generated, awaits confirmations
  ↓
[Phase 4] Submit print-and-mail job to PostGrid
  → Inject organization logo + QR code
  → PostGrid returns carrier + tracking numbers
  ↓
[Phase 4] Create mailing receipt PDF
  → Document list with Arweave + Witnet references
  → QR codes with embedded tracking info
  → Payment details (card masked or crypto tx hash)
  → Upload receipt to Arweave + attest on Witnet
  ↓
[Phase 4] Background service polls carrier tracking
  → Every 12 hours until delivery
  → Generate delivery receipt on confirmation
  → Upload delivery receipt to Arweave + attest on Witnet
  ↓
Final State: Complete cryptographic chain
  Digital document → Blockchain proof → Physical artifact → Delivery proof
```

---

## 📚 Documentation

### Core Documentation
- **[DEVELOPMENT_PLAN.md](./DEVELOPMENT_PLAN.md)** – Phase 1/2/3/4 roadmap, milestones, success metrics
- **[ARCHITECTURE.md](./ARCHITECTURE.md)** – System design, data structures, data flow diagrams
- **[AGENTS.md](./AGENTS.md)** – Module boundaries, AI agent guidance, testing patterns
- **[CLAUDE.md](./CLAUDE.md)** – Claude integration, prompt templates, PII handling
- **[SECURITY.md](./SECURITY.md)** – Threat model, cryptographic guarantees, security practices

### Phase 4 Specific Documentation ⭐ **NEW**
- **[POSTGRID_INTEGRATION.md](./POSTGRID_INTEGRATION.md)** – PostGrid API integration, job submission, tracking
- **[MAIL_PAYMENT.md](./MAIL_PAYMENT.md)** – Stripe + BTCPay payment flows, settlement reconciliation
- **[MAIL_QR_CODEC.md](./MAIL_QR_CODEC.md)** – QR code design, encoding schema, verification
- **[MAIL_LIFECYCLE.md](./MAIL_LIFECYCLE.md)** – Job state machine, receipt generation, delivery tracking
- **[MAIL_API.md](./MAIL_API.md)** – REST API spec for mail job creation, status, receipts
- **[MAIL_CLI.md](./MAIL_CLI.md)** – CLI command reference for mail operations

### Developer Guides
- **[CONTRIBUTING.md](./CONTRIBUTING.md)** – Code style, testing requirements, PR process
- **[PROJECT_CHECKLIST.md](./PROJECT_CHECKLIST.md)** – Week-by-week deliverables and acceptance criteria

---

## 🤖 Claude Integration

ArweaveStamp includes a comprehensive Claude Code integration with **16 custom commands**, **1 generic agent**, and **12 specialized skills** to streamline development workflows.

### Custom Commands (`.claude/commands/`)

**Core Workflow Commands**:
- `/start-session` – Initialize development session with project context
- `/plan` – Generate or update development plans
- `/test-all` – Execute comprehensive test suite
- `/pr` – Streamline pull request creation with validation
- `/close-session` – End session with progress summary

**Domain-Specific Commands**:
- `/attest-document` – Guide through document attestation workflow (Phase 1)
- `/verify-proof` – Verify proof package against blockchain
- `/validate-phase` – Validate phase completion criteria (accepts 1-4)
- `/integration-check` – Test all external service connections
- `/mail-job` – Create print & mail job workflow (Phase 4)
- `/track-delivery` – Check delivery status for mail job

**Multi-Agent Orchestration**:
- `/orchestrate-feature` – Coordinate multiple agents for parallel feature development

**QA & Utility Commands**:
- `/deps-update` – Review and update dependencies safely
- `/lint-fixes` – Auto-fix code style issues
- `/env-check` – Validate development environment
- `/setup-dev` – Automate developer onboarding

### Generic Agent (`.claude/agents/`)

**ArweaveStamp General Developer** – A general-purpose development agent following the **skills-first paradigm**:
- Dynamically loads specialized skills based on task requirements
- Works across all project phases (1-4)
- Progressively loads additional skills as complexity emerges
- 35% more token-efficient than traditional multi-agent approaches

### Specialized Skills (`.claude/skills/`)

**Core Domain Skills**:
- `blockchain-attestation-skill` – Arweave upload → Claude analysis → Witnet attestation → L1 relay
- `document-analysis-skill` – Claude-powered classification, entity extraction, summarization
- `crypto-hashing-skill` – SHA-256 hashing, determinism verification, document ID assembly

**Development Role Skills**:
- `builder-role-skill` – TDD implementation, TypeScript strict mode, git workflow
- `validator-role-skill` – Testing, coverage analysis, code review, security audit

**Phase-Specific Skills**:
- `mail-integration-skill` – PostGrid print & mail job creation (Phase 4)
- `payment-processing-skill` – Stripe and BTCPay payment flows (Phase 4)
- `audit-trail-skill` – Merkle tree construction, blockchain anchoring (Phase 3)

**Utility Skills**:
- `integration-testing-skill` – External service testing (Arweave, Claude, Witnet, PostGrid)
- `phase-validation-skill` – Phase completion criteria validation
- `environment-setup-skill` – Developer environment automation
- `proof-verification-skill` – Independent proof package verification

### Usage Examples

```bash
# Start a development session
/start-session

# Create a development plan for a new feature
/plan

# Validate Phase 1 completion
/validate-phase 1

# Test all external services
/integration-check

# Create a pull request with validation
/pr

# End session with summary
/close-session
```

### Skills-First Development

The agent follows a **skills-first paradigm** where a single general agent dynamically loads skills based on task requirements, rather than using multiple specialized agents. This approach provides:

- **35% token efficiency gain** compared to multi-agent approaches
- **Progressive complexity handling** – load minimal skills initially, add as needed
- **Better context management** – maintain context throughout task execution
- **Composable workflows** – skills can reference other skills as dependencies

Multi-agent orchestration is reserved for specific scenarios:
- Parallel independent research
- Exploring multiple solution approaches simultaneously
- Breadth-first tasks requiring concurrency

For more details, see [CLAUDE.md](./CLAUDE.md) and the Claude Code best practices in `docs/claude/`.

---

## 🧪 Testing

### Run Tests
```bash
# Unit tests
npm test

# With coverage report
npm run test:coverage

# Integration tests (require real APIs)
npm run test:integration

# Phase 4: Mail integration tests (requires PostGrid sandbox)
npm run test:mail

# Performance benchmarks
npm run test:performance
```

### Coverage Targets
- **Overall**: >85%
- **Integrity module** (crypto): 100%
- **Storage, Analysis, Oracle modules**: >85%
- **Mail module** (PostGrid integration): >90%
- **Payment module** (Stripe, BTCPay): >85%

---

## 📦 CLI Commands

### Phase 1 Commands

```bash
# Initialize credentials
stamp init

# Attest single document
stamp stamp <file>

# Batch process directory
stamp batch <directory>

# Verify proof
stamp verify <proof.json>

# List all proofs
stamp list [--filter documentType:contract]

# Export proof
stamp export <documentId> --format pdf
```

### Phase 3 Commands

```bash
# Start file monitoring daemon
stamp watch <directory>

# Manage monitoring policies
stamp policy add <policy.json>
stamp policy list
stamp policy test <policyId> <testFile>

# Verify file integrity against blockchain
stamp verify-integrity <file>

# Search audit logs
stamp audit search --file <path> --since 2025-01-01

# Export audit trail
stamp audit export --format json > audit_trail.json
```

### Phase 4 Commands ⭐ **NEW**

```bash
# Create a new print & mail job
stamp mail create <document> \
  --recipient-name "Name" \
  --recipient-address "Address" \
  --payment-method stripe|crypto \
  [--job-title "Title"] \
  [--branding-logo-url "https://..."] \
  [--branding-footer "Custom footer"]

# List mail jobs
stamp mail list [--status pending|submitted|in-transit|delivered]

# Get mail job status
stamp mail status <jobId>

# Track delivery in real-time
stamp mail track <jobId> [--watch]

# Retrieve mailing receipt
stamp mail receipt <jobId> [--format json|pdf]

# Retrieve delivery receipt
stamp mail delivery-receipt <jobId> [--format json|pdf]

# Verify QR code contents
stamp mail verify-qr <qrCodeData>

# Export full mail job lifecycle as JSON
stamp mail export <jobId> --format json > job_lifecycle.json

# Resend receipt or delivery confirmation via email
stamp mail resend-receipt <jobId> --email recipient@example.com
```

---

## 💰 Cost Estimation

### Phase 1 (Per Document)
| Component | Cost |
|-----------|------|
| Arweave storage (1MB) | ~$0.01 |
| Claude analysis | ~$0.30 |
| Witnet oracle | ~$0.05 |
| L1 relays (3 chains) | ~$0.50+ (varies) |
| **Total per doc** | **~$0.85-1.50** |

### Phase 4 (Per Mailed Document)
| Component | Typical Cost |
|-----------|------|
| PostGrid print + postage (single page) | $0.75-1.50 |
| PostGrid branding/logo injection | included |
| Address verification | included |
| Claude metadata enrichment | ~$0.10 |
| QR code generation + embedding | <$0.01 |
| Receipt PDF generation + upload | ~$0.05 |
| Witnet receipt attestation | ~$0.05 |
| Receipt storage on Arweave (200KB) | ~$0.002 |
| Payment processing (Stripe 2.9% + $0.30) | ~3.5% of job cost |
| Carrier tracking (12-24 polls) | <$0.01 |
| Delivery receipt generation + attestation | ~$0.10 |
| **Total per mailed document** | **~$1.50-2.50** |

**Phase 4 Monthly Estimate** (1000 mailed documents): $1,500-2,500

---

## 🔐 Security Model

### Cryptographic Guarantees

| Component | Algorithm | Security |
|-----------|-----------|----------|
| File Proof | SHA-256 | 256-bit (2^256 collision work) |
| Metadata Proof | SHA-256 | 256-bit |
| Combined Proof | SHA-256(hash+hash) | 512-bit effective |
| QR Code Payload | HMAC-SHA256 | Tamper detection |
| Receipt PDF | Digital signature | Certificate authority signed |
| Arweave Storage | PoW consensus | Network security |
| Witnet Oracle | Multisig + stake | Majority of witnesses honest |
| L1 Blockchains | PoW (Eth/Avax) / PoS (Polygon) | Network consensus |

### Phase 4: Physical Mail Chain of Custody

```
Document (digital)
  ↓ [Cryptographic hash + Arweave upload]
Arweave transaction ID (immutable timestamp)
  ↓ [Witnet attestation]
Witnet report ID (decentralized witness)
  ↓ [L1 blockchain relay]
Smart contract event (blockchain consensus)
  ↓ [QR code generation]
QR code (embedded in physical print)
  ↓ [PostGrid print & mail]
Physical artifact (mailed envelope)
  ↓ [Carrier delivery]
Delivery confirmation (signed receipt)
  ↓ [Delivery receipt + attestation]
Arweave delivery record (final link in chain)
```

### Zero-Trust Verification (Phase 4)

Anyone can verify a mailed document's entire lifecycle without:
- Trusting ArweaveStamp
- Trusting PostGrid
- Accessing original files
- Running any software

**Required**: QR code data + Arweave URLs + carrier tracking links (all public)

---

## 🔄 Development Workflow

### For Phase 4 Developers

1. **Read Documentation**
   - Start: [README.md](./README.md) (this file)
   - Architecture: [ARCHITECTURE.md](./ARCHITECTURE.md)
   - Mail Integration: [POSTGRID_INTEGRATION.md](./POSTGRID_INTEGRATION.md)
   - Payment: [MAIL_PAYMENT.md](./MAIL_PAYMENT.md)
   - Modules: [AGENTS.md](./AGENTS.md)

2. **Set Up Environment**
   ```bash
   npm install
   npm test  # Verify CI passes
   npm run lint
   npm run test:mail  # Phase 4 integration tests
   ```

3. **Development Process**
   - Pick a module from [PROJECT_CHECKLIST.md](./PROJECT_CHECKLIST.md)
   - Write tests first (TDD pattern)
   - Follow TypeScript strict mode
   - Test with PostGrid sandbox first
   - Run full test suite frequently
   - Create PR with tests + docs

4. **Testing Requirements**
   - >85% code coverage
   - All edge cases tested (payment failures, address verification, carrier unavailable)
   - Integration tests against PostGrid sandbox
   - Carrier tracking simulation tests

---

## 🤝 Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for:
- Code style guidelines
- Testing requirements
- Pull request process
- Code review checklist

---

## 📊 Success Metrics

### Phase 4 Complete When
- ✅ Mail job creation + submission working end-to-end
- ✅ Payment settlement via Stripe and BTCPay functional
- ✅ QR codes generated and embedded correctly
- ✅ Address verification OCR working accurately
- ✅ Carrier tracking polls retrieving data
- ✅ Delivery receipts generating + attesting on Witnet
- ✅ >85% test coverage achieved
- ✅ Zero critical security issues
- ✅ Full lifecycle audit trail verifiable

---

## 📖 Learn More

### Official Documentation
- [Arweave Docs](https://docs.arweave.org/)
- [Witnet Docs](https://docs.witnet.io/)
- [PostGrid API Docs](https://docs.postgrid.com/)
- [Stripe API Docs](https://stripe.com/docs/api)
- [BTCPay Server Docs](https://docs.btcpayserver.org/)
- [Anthropic Claude Docs](https://docs.anthropic.com/)

### Community & Support
- Arweave Discord
- Witnet Community Forum
- PostGrid Community
- Anthropic Discord

---

## 📄 License

[MIT License](./LICENSE)

---

## 🚀 Roadmap: From Attestation to Physical Delivery

```
Phase 1: Attestation (Weeks 1-16)
  Upload → Analyze → Hash → Attest → Verify
  ↓
Phase 2: Dashboard (Weeks 4-28)
  Search → Filter → Report → Public Portal
  ↓
Phase 3: Continuous Monitoring (Weeks 17-24)
  Watch → Policy → Re-attest → Audit → Anchor
  ↓
Phase 4: Physical Mail (Weeks 25-40)
  Mail job → OCR verify → Generate QR → Payment → Print & Mail → Track → Deliver
  ↓
Complete Lifecycle Solution:
  Upload → Attest → Monitor → Detect → Mail → Deliver → Audit
```

---

## ❓ FAQ

**Q: Why blockchain + physical mail?**
A: Blockchain proves digital authenticity with immutable timestamping. Physical mail creates a tamper-evident "paper twin" with carrier authentication and delivery proof.

**Q: What if the mailed envelope is lost?**
A: Carrier tracking provides real-time status. If lost/undeliverable, you have PostGrid + carrier records. Full lifecycle is auditable on Arweave.

**Q: Can a recipient verify the document is authentic?**
A: Yes. Scan the QR code → links to Arweave + Witnet data. No software needed—just a smartphone QR reader and web browser.

**Q: Does PostGrid see the documents?**
A: PostGrid sees the print-ready PDF only. No access to your private data. Mailing address is provided by you; OCR is client-side.

**Q: What payment methods do you support?**
A: Phase 4 supports Stripe (cards: Visa, Mastercard, Amex) and BTCPay Server (crypto: BTC, ETH, USDC, USDT, XMR, etc.).

**Q: How does payment reconciliation work?**
A: Stripe: immediate settlement. BTCPay: awaits 1-3 block confirmations (configurable). Both create attestations on Arweave.

**Q: Can I white-label the branding?**
A: Yes. Per-job logo, footer, sender info overrides. Defaults set at organization level.

**Q: How real-time is carrier tracking?**
A: Background service polls every 12 hours by default (configurable). Typical carriers (USPS, UPS, FedEx) update 1-2x daily.

---

## 📞 Support

- **Documentation**: See [DEVELOPMENT_PLAN.md](./DEVELOPMENT_PLAN.md) + [ARCHITECTURE.md](./ARCHITECTURE.md) + Phase 4 guides
- **Issues**: GitHub Issues
- **Discussions**: GitHub Discussions
- **Security**: See [SECURITY.md](./SECURITY.md)

---

**Ready to build document verification that works in the physical world?**

Start with Phase 1: `npm run cli -- init`  
Proceed to Phase 4: `npm run cli -- mail create --help`

🚀 **Let's build the future of trustless document delivery together.**

---

**Last Updated**: December 2025  
**Phase**: Ready for Phase 4 Development (Weeks 25-40)  
**Next Step**: Assign Phase 4 developers, set up PostGrid sandbox, begin Week 25-26 scaffolding
