# DEVELOPMENT_PLAN.md - Phase 1, 2, 3, and 4 (PostGrid Integration)
**Arweave + Witnet Authenticity Platform with Physical Mail Integration**

---

## Executive Summary: Four-Phase Evolution

The platform evolves from digital attestation to hybrid physical-digital verification:

```
PHASE 1: Document Attestation (Weeks 1-16)
  Upload → Analyze → Hash → Attest → Verify (Digital Only)
  
PHASE 2: Proof Management (Weeks 4-28)
  Web dashboard → Search/filter → Reports → Verification portal
  
PHASE 3: Lifecycle Monitoring (Weeks 17-24)
  Watch → Policy → Re-attest → Audit → Anchor (Continuous)
  
PHASE 4: Physical Mail Integration (Weeks 25-36) ⭐ **NEW**
  Embed QR → Extract address → Verify → Print → Mail → Track → Deliver
  ↓
COMPLETE SOLUTION:
  Digital attestation + Physical delivery + Cryptographic lineage + Audit trail
```

---

## 1. Phases 1-3 Summary (Existing)

*[Weeks 1-24 remain unchanged - see DEVELOPMENT_PLAN_v2.md for full details]*

**Completed by Week 24**:
- ✅ Phase 1: CLI tool with core Arweave + Claude + Witnet integrations
- ✅ Phase 2: Web dashboard for proof management and search
- ✅ Phase 3: File monitoring daemon with smart policies and audit trails

---

## 2. Phase 4: PostGrid Print & Mail Integration (Weeks 25-36) ⭐ NEW

### High-Level Goal

Transform ArweaveStamp into a hybrid digital-physical authenticity platform:

1. **Digitally attest** documents (Phases 1-3)
2. **Embed cryptographic references** as QR codes
3. **Extract and verify** mailing addresses
4. **Organize branding** (logos, templates)
5. **Process payments** (Stripe or BTCPay)
6. **Print and mail** via PostGrid
7. **Track delivery** with carrier updates
8. **Attest outcomes** (receipts, delivery) back to blockchain

### Architecture: Phase 4 Integration

```
┌─────────────────────────────────────────┐
│ Phase 1-3: Digital Attestation          │
│ (Arweave + Claude + Witnet)            │
└──────────────┬──────────────────────────┘
               │ (ProofPackage)
               ▼
┌──────────────────────────────────────────┐
│ Phase 4: Physical Mail Integration      │
├──────────────────────────────────────────┤
│ • QR code generation                     │
│ • Address OCR + verification             │
│ • Branding injection                     │
│ • Payment processing (Stripe/BTCPay)    │
│ • PostGrid job submission                │
│ • Carrier tracking polling               │
│ • Receipt generation + attestation       │
└──────────────────────────────────────────┘
               │
               ▼
    ┌─────────────────────────┐
    │ Arweave (receipts)      │
    │ Witnet (attestations)   │
    │ Blockchain (payments)   │
    └─────────────────────────┘
```

### New Modules (Phase 4)

```
src/
├── postal/                          # PostGrid integration (NEW)
│   ├── client.ts                    # PostGrid API client
│   ├── jobs.ts                      # Print job creation
│   ├── tracking.ts                  # Carrier tracking polling
│   ├── types.ts                     # PostGrid types
│   └── config.ts                    # PostGrid configuration
│
├── mail/                            # Mail job orchestration (NEW)
│   ├── composer.ts                  # Build mail payload
│   ├── address.ts                   # OCR + verification
│   ├── qrcodes.ts                   # QR code generation
│   ├── branding.ts                  # Logo/template injection
│   ├── receipts.ts                  # Receipt PDF generation
│   └── types.ts                     # Mail types
│
├── payment/                         # Payment processing (NEW)
│   ├── stripe.ts                    # Stripe integration
│   ├── btcpay.ts                    # BTCPay integration
│   ├── settlement.ts                # Payment settlement logic
│   ├── callbacks.ts                 # Webhook handlers
│   └── types.ts                     # Payment types
│
├── tracking/                        # Delivery tracking service (NEW)
│   ├── poller.ts                    # Background polling service
│   ├── updates.ts                   # Update processing
│   ├── notifications.ts             # User notifications
│   └── types.ts                     # Tracking types
│
├── cli/
│   └── commands/
│       ├── mail.ts                  # NEW: mail command
│       ├── mail-status.ts           # NEW: check mail status
│       └── mail-receipt.ts          # NEW: retrieve receipt
│
└── frontend/
    └── pages/
        ├── MailComposer.tsx         # NEW: UI for mail jobs
        ├── MailTracking.tsx         # NEW: tracking dashboard
        └── PaymentFlow.tsx          # NEW: payment UI
```

### New Data Structures (Phase 4)

#### MailJob Type

```typescript
interface MailJob {
  // Identity
  id: string;                         // UUID
  organizationId: string;
  createdAt: ISO8601;
  
  // Document references
  documents: Array<{
    proofDocumentId: string;          // Reference to ProofPackage
    arweaveUrl: string;
    witnetAttestationId: string;
    title: string;
  }>;
  
  // Addressing
  sender: {
    name: string;
    address: PostalAddress;
    verified: boolean;
  };
  recipients: Array<{
    name: string;
    address: PostalAddress;
    verified: boolean;
    status: 'pending' | 'submitted' | 'printing' | 'in_transit' | 'delivered' | 'failed';
  }>;
  
  // Branding
  branding: {
    organizationLogo?: string;        // Base64 or URL
    organizationName: string;
    returnAddress?: PostalAddress;
  };
  
  // QR codes embedded
  qrCodes: Array<{
    recipientIndex: number;
    arweaveUrl: string;
    witnetId: string;
    receiptArweaveUrl?: string;       // After printing
    encodedData: string;
  }>;
  
  // Payment
  payment: {
    method: 'stripe' | 'btcpay';
    currency: 'USD' | 'BTC' | 'ETH';
    estimatedCost: number;
    actualCost?: number;
    status: 'pending' | 'processing' | 'confirmed' | 'failed';
    transactionId?: string;           // Stripe charge ID or BTCPay invoice ID
    transactionHash?: string;         // Blockchain tx hash (crypto)
    confirmations?: number;
    settledAt?: ISO8601;
  };
  
  // PostGrid integration
  postgrid: {
    jobId?: string;                   // PostGrid job ID
    submitted: boolean;
    submittedAt?: ISO8601;
    trackingNumbers?: string[];
    carrier?: 'USPS' | 'UPS' | 'FedEx';
  };
  
  // AI metadata
  metadata: {
    summary: string;                  // Claude-generated
    tags: string[];
    classification: string;
    riskFlags: string[];
  };
  
  // Receipts
  receipts: {
    initialReceiptArweaveUrl?: string;
    initialReceiptWitnetId?: string;
    deliveryReceiptArweaveUrl?: string;
    deliveryReceiptWitnetId?: string;
  };
  
  // Delivery tracking
  tracking: {
    status: 'pending' | 'in_transit' | 'delivered' | 'failed';
    lastUpdate: ISO8601;
    deliveredAt?: ISO8601;
    events: Array<{
      timestamp: ISO8601;
      status: string;
      location?: string;
      details?: string;
    }>;
  };
}

interface PostalAddress {
  streetLine1: string;
  streetLine2?: string;
  city: string;
  stateProvince: string;
  postalCode: string;
  country: string;
  verified: boolean;
  verificationErrors?: string[];
}

interface MailReceipt {
  id: string;
  mailJobId: string;
  createdAt: ISO8601;
  
  // Job summary
  jobSummary: {
    senderAddress: PostalAddress;
    recipientCount: number;
    documentCount: number;
  };
  
  // Document list
  documents: Array<{
    title: string;
    arweaveUrl: string;
    witnetId: string;
    qrCode: string;                   // Embedded image
  }>;
  
  // Mailing details
  mailingDetails: {
    submittedAt: ISO8601;
    carrier: string;
    trackingNumbers: string[];
    estimatedDelivery?: ISO8601;
  };
  
  // Payment record
  payment: {
    method: string;
    amount: number;
    currency: string;
    transactionId: string;
    timestamp: ISO8601;
  };
  
  // Attestation
  attestation: {
    arweaveUrl: string;
    arweaveTxId: string;
    witnetRequestId: string;
    witnetReportId: string;
    attestedAt: ISO8601;
  };
}

interface DeliveryReceipt {
  id: string;
  mailJobId: string;
  initialReceiptArweaveUrl: string;   // Link to original receipt
  
  deliveryDetails: {
    status: 'delivered' | 'failed';
    deliveredAt?: ISO8601;
    failureReason?: string;
    signedBy?: string;
  };
  
  trackingData: {
    carrier: string;
    trackingNumbers: string[];
    events: Array<{
      timestamp: ISO8601;
      status: string;
      location?: string;
    }>;
  };
  
  // Attestation (references original + initial receipt)
  attestation: {
    referencesDocuments: string[];    // Original doc Arweave URLs
    referencesInitialReceipt: string; // Initial receipt Arweave URL
    arweaveUrl: string;
    arweaveTxId: string;
    witnetRequestId: string;
    witnetReportId: string;
    attestedAt: ISO8601;
  };
}
```

---

## 3. Phase 4: Detailed Milestone Breakdown

### Weeks 25-26: PostGrid Integration & API Client

**Goal**: Connect to PostGrid API, handle print job submission

**Deliverables**:
- [ ] `src/postal/client.ts` – PostGrid HTTP client
  - [ ] Initialize PostGrid API with auth
  - [ ] Implement exponential backoff + retry logic
  - [ ] Error handling (rate limits, API errors)
  - [ ] Unit tests with mocked PostGrid API

- [ ] `src/postal/jobs.ts` – Print job creation
  - [ ] Construct PostGrid job payload
  - [ ] Support PDF uploads
  - [ ] Handle recipient lists
  - [ ] Submit to PostGrid
  - [ ] Retrieve job ID + initial tracking
  - [ ] Integration tests

- [ ] `src/postal/types.ts` – PostGrid types
  - [ ] PostGrid job schema
  - [ ] Recipient type
  - [ ] Tracking information
  - [ ] Response types

- [ ] Documentation
  - [ ] PostGrid integration guide
  - [ ] API credential setup
  - [ ] Supported parameters

**Acceptance Criteria**:
- ✅ Can submit print job to PostGrid
- ✅ Job ID retrieved successfully
- ✅ Tracking numbers available
- ✅ Error handling tested

---

### Weeks 27-28: Address Handling & QR Code Generation

**Goal**: Extract, verify addresses; generate QR codes with attestation references

**Deliverables**:
- [ ] `src/mail/address.ts` – OCR + verification
  - [ ] Extract address blocks from documents (Tesseract.js or similar)
  - [ ] Call PostGrid address verification API
  - [ ] Handle verification errors + corrections
  - [ ] Return verified `PostalAddress` type
  - [ ] Unit tests

- [ ] `src/mail/qrcodes.ts` – QR code generation
  - [ ] Encode Arweave URL in QR code
  - [ ] Encode Witnet attestation ID in QR code
  - [ ] Support data URL or file output
  - [ ] Handle large payloads (nested JSON)
  - [ ] Unit tests

- [ ] `src/mail/composer.ts` – Mail payload assembly
  - [ ] Combine document PDF + QR codes
  - [ ] Inject recipient address
  - [ ] Generate final print-ready PDF
  - [ ] Support custom templates
  - [ ] Integration tests

- [ ] Documentation
  - [ ] OCR setup guide
  - [ ] Address verification errors
  - [ ] QR code encoding format

**Acceptance Criteria**:
- ✅ Addresses extracted and verified
- ✅ QR codes generate correctly
- ✅ Print-ready PDFs created
- ✅ Error handling for invalid addresses

---

### Weeks 29-30: Branding & Organization Support

**Goal**: Support per-organization logos, templates, and overrides

**Deliverables**:
- [ ] `src/mail/branding.ts` – Logo and template injection
  - [ ] Store organization logos (base64, URL, or Arweave)
  - [ ] Inject logo into document header/footer
  - [ ] Support custom templates (HTML + CSS)
  - [ ] Per-job branding overrides
  - [ ] Unit tests

- [ ] Organization settings
  - [ ] Store default branding per org
  - [ ] Organization-level return address
  - [ ] Default document templates
  - [ ] Database schema updates

- [ ] CLI support
  - [ ] `stamp mail compose --org <id> --branding-override`
  - [ ] Upload org logo: `stamp org upload-logo --org <id>`

- [ ] Web UI
  - [ ] Organization settings page
  - [ ] Logo upload
  - [ ] Template preview

**Acceptance Criteria**:
- ✅ Logos injected correctly
- ✅ Templates applied
- ✅ Branding overrides work
- ✅ Per-org customization functional

---

### Weeks 31-32: Payment Processing (Stripe + BTCPay)

**Goal**: Support card and crypto payments with settlement tracking

**Deliverables**:
- [ ] `src/payment/stripe.ts` – Stripe integration
  - [ ] Initialize Stripe API
  - [ ] Create charge for mail job cost
  - [ ] Handle 3D Secure / card verification
  - [ ] Webhook handling for charge events
  - [ ] Refund support
  - [ ] Unit tests

- [ ] `src/payment/btcpay.ts` – BTCPay integration
  - [ ] Initialize BTCPay API
  - [ ] Create invoice for mail job cost (BTC, ETH, USDT)
  - [ ] Generate payment QR code
  - [ ] Webhook handling for payment confirmations
  - [ ] Monitor blockchain confirmations
  - [ ] Unit tests

- [ ] `src/payment/settlement.ts` – Settlement logic
  - [ ] Determine when payment is "settled"
  - [ ] For Stripe: immediate on charge confirmation
  - [ ] For BTCPay: configurable confirmations (12+ typical)
  - [ ] Only submit to PostGrid when settled
  - [ ] Refund policy on failed mailings

- [ ] `src/payment/callbacks.ts` – Webhook handlers
  - [ ] Stripe webhook endpoint
  - [ ] BTCPay webhook endpoint
  - [ ] Update payment status in DB
  - [ ] Trigger PostGrid submission if ready
  - [ ] Security: signature verification

- [ ] Documentation
  - [ ] Stripe setup + API keys
  - [ ] BTCPay setup + pairing
  - [ ] Payment flow diagram
  - [ ] Refund policy

**Acceptance Criteria**:
- ✅ Stripe charges work
- ✅ BTCPay invoices work
- ✅ Webhooks process correctly
- ✅ Payment status tracked
- ✅ Settlement gates PostGrid submission

---

### Weeks 33-34: Confirmation Flow & Receipt Generation

**Goal**: Present user confirmation, generate comprehensive receipts

**Deliverables**:
- [ ] CLI confirmation step
  - [ ] Display summary: sender, recipients, documents, cost
  - [ ] Show verified addresses (with any corrections)
  - [ ] Payment method selection
  - [ ] User confirmation prompt before payment
  - [ ] Abort on payment failure

- [ ] Web UI confirmation page
  - [ ] React component for `<MailConfirmationFlow />`
  - [ ] Address summary with verification status
  - [ ] Document list with Arweave links
  - [ ] Cost breakdown (print, postage, fees)
  - [ ] Payment method selector
  - [ ] Confirmation button + error handling

- [ ] `src/mail/receipts.ts` – Receipt PDF generation
  - [ ] Generate initial receipt (post-printing, pre-delivery)
    - [ ] Sender + recipient addresses
    - [ ] Document list with Arweave/Witnet links
    - [ ] QR codes embedding attestation references
    - [ ] Carrier + tracking numbers
    - [ ] Payment details (card brand + last 4 or tx hash)
    - [ ] Job metadata (ID, timestamp, org)
  - [ ] Generate delivery receipt (post-delivery)
    - [ ] Delivery date/time
    - [ ] Final status + carrier notes
    - [ ] Link to initial receipt
    - [ ] Updated metadata

- [ ] Receipt upload to Arweave + attestation
  - [ ] Upload initial receipt to Arweave
  - [ ] Create Witnet attestation for receipt
  - [ ] Link attestation to original documents
  - [ ] Store Arweave URL + Witnet ID in DB

- [ ] Documentation
  - [ ] Receipt format specification
  - [ ] Attestation payload structure
  - [ ] PDF generation examples

**Acceptance Criteria**:
- ✅ Confirmation flow works CLI + Web
- ✅ Receipts generated with all details
- ✅ Receipts uploaded to Arweave
- ✅ Witnet attestations created
- ✅ User cannot submit without confirmation

---

### Weeks 35-36: Delivery Tracking & Automation

**Goal**: Background service polls carrier tracking, updates on delivery

**Deliverables**:
- [ ] `src/tracking/poller.ts` – Background polling service
  - [ ] Scheduled job runs every 12 hours
  - [ ] Query PostGrid for updated tracking
  - [ ] Filter for non-final statuses (not delivered/failed yet)
  - [ ] Retry logic + error handling
  - [ ] Graceful shutdown

- [ ] `src/tracking/updates.ts` – Update processing
  - [ ] Process new tracking events
  - [ ] Detect delivery confirmation (final status)
  - [ ] Generate delivery receipt PDF
  - [ ] Upload delivery receipt to Arweave
  - [ ] Create Witnet attestation for delivery
  - [ ] Update DB with final status

- [ ] `src/tracking/notifications.ts` – User notifications
  - [ ] Send email on delivery confirmation
  - [ ] Include link to delivery receipt
  - [ ] Optional SMS/in-app notifications
  - [ ] Notification preferences per org

- [ ] Delivery receipt Arweave + Witnet
  - [ ] Reference original documents
  - [ ] Reference initial receipt
  - [ ] Include carrier tracking payload
  - [ ] Full audit trail on blockchain

- [ ] CLI/Web status commands
  - [ ] `stamp mail-status <jobId>` – Get current status
  - [ ] Web UI: `<MailTrackingDashboard />`
  - [ ] Real-time updates (WebSocket or polling)

- [ ] Integration tests
  - [ ] Mock PostGrid tracking API
  - [ ] Verify polling + update flow
  - [ ] Verify Arweave + Witnet uploads
  - [ ] Verify notifications sent

- [ ] Documentation
  - [ ] Tracking lifecycle diagram
  - [ ] Polling service setup
  - [ ] Webhook configuration
  - [ ] Notification templates

**Acceptance Criteria**:
- ✅ Polling service runs on schedule
- ✅ Delivery updates processed
- ✅ Delivery receipts created + attested
- ✅ Users notified on delivery
- ✅ Full audit trail on blockchain

---

### Phase 4: AI Metadata Integration (Throughout)

At each major stage, generate Claude metadata:

1. **Initial registration**: Document classification, entity extraction
2. **Job creation**: Mail job summary, risk assessment
3. **Job submission**: Confirmation of mailing parameters
4. **Delivery confirmation**: Delivery status summary

Metadata included in:
- QR code payload (if space allows)
- Arweave JSON metadata
- Witnet attestation payloads

---

## 4. Phase 4: CLI Commands

### New Commands

```bash
# Compose mail job (with confirmation)
stamp mail compose \
  --documents <proof1.json> <proof2.json> \
  --recipient-address "123 Main St, City, State 12345" \
  --sender-name "Acme Corp" \
  --organization-id <org-id>

# Show confirmation summary
stamp mail confirm --job-id <jobId>

# Submit mail job (after payment)
stamp mail submit --job-id <jobId> --payment-method stripe

# Check mail status
stamp mail-status --job-id <jobId>

# Retrieve receipt
stamp mail-receipt --job-id <jobId> --format pdf

# List mail jobs
stamp mail list --organization-id <org-id>

# Organization branding setup
stamp org upload-logo --org <org-id> --file logo.png
stamp org set-return-address --org <org-id> --address "..."
stamp org set-default-template --org <org-id> --template <template.html>
```

---

## 5. Phase 4: Web UI Components

### New Pages

```typescript
// pages/MailComposer.tsx
// - Document selection
// - Address entry + verification preview
// - Branding selection
// - Cost summary
// - Confirmation button

// pages/MailConfirmation.tsx
// - Address verification status
// - Document list with QR preview
// - Payment method selection
// - Payment processing UI
// - Confirmation + abort options

// pages/MailTracking.tsx
// - List of mail jobs
// - Status per job (pending, in_transit, delivered)
// - Tracking number + carrier link
// - Receipt download
// - Delivery date + notes

// pages/OrganizationSettings.tsx
// - Logo upload
// - Return address setup
// - Template selection
// - Default branding
```

---

## 6. Phase 4: Database Schema Updates

### New Tables (SQLite Phase 4, PostgreSQL Phase 5+)

```sql
-- Mail jobs
CREATE TABLE mail_jobs (
  id TEXT PRIMARY KEY,
  organization_id TEXT NOT NULL,
  created_at DATETIME,
  status TEXT,
  sender_name TEXT,
  sender_address_json TEXT,
  document_count INTEGER,
  recipient_count INTEGER,
  estimated_cost REAL,
  actual_cost REAL,
  postgrid_job_id TEXT,
  submitted_at DATETIME,
  delivered_at DATETIME,
  initial_receipt_arweave_url TEXT,
  initial_receipt_witnet_id TEXT,
  delivery_receipt_arweave_url TEXT,
  delivery_receipt_witnet_id TEXT,
  FOREIGN KEY(organization_id) REFERENCES organizations(id)
);

-- Mail recipients
CREATE TABLE mail_recipients (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL,
  recipient_name TEXT,
  address_json TEXT,
  verified BOOLEAN,
  status TEXT,
  tracking_number TEXT,
  FOREIGN KEY(job_id) REFERENCES mail_jobs(id)
);

-- Mail documents
CREATE TABLE mail_documents (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL,
  proof_document_id TEXT,
  arweave_url TEXT,
  witnet_id TEXT,
  title TEXT,
  FOREIGN KEY(job_id) REFERENCES mail_jobs(id)
);

-- Payments
CREATE TABLE payments (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL,
  method TEXT,
  currency TEXT,
  amount REAL,
  status TEXT,
  transaction_id TEXT,
  transaction_hash TEXT,
  confirmations INTEGER,
  settled_at DATETIME,
  FOREIGN KEY(job_id) REFERENCES mail_jobs(id)
);

-- Tracking events
CREATE TABLE tracking_events (
  id TEXT PRIMARY KEY,
  job_id TEXT NOT NULL,
  timestamp DATETIME,
  status TEXT,
  location TEXT,
  details TEXT,
  FOREIGN KEY(job_id) REFERENCES mail_jobs(id)
);

-- Organization branding
CREATE TABLE organization_branding (
  org_id TEXT PRIMARY KEY,
  logo_url TEXT,
  return_address_json TEXT,
  default_template_id TEXT,
  FOREIGN KEY(org_id) REFERENCES organizations(id)
);
```

---

## 7. Phase 4: Integration Architecture

### System Flow: End-to-End

```
User uploads documents (Phase 1-3)
  ↓
ProofPackages generated + attested
  ↓
User initiates mail job (Phase 4)
  ↓
Extract recipient address + verify
  ↓
Generate QR codes with Arweave/Witnet refs
  ↓
Inject branding + create print-ready PDF
  ↓
Calculate cost (print + postage + fees)
  ↓
User confirms summary + selects payment
  ↓
Process payment (Stripe or BTCPay)
  ↓
(IF settled) Submit to PostGrid
  ↓
PostGrid prints + mails
  ↓
Initial receipt generated + attested
  ↓
Background service polls tracking (12h intervals)
  ↓
(ON delivery) Generate delivery receipt + attest
  ↓
User notified (email + in-app)
  ↓
Full audit trail on Arweave + Witnet
```

---

## 8. Phase 4: Success Metrics

### Phase 4 Complete When
- ✅ Can compose mail job with CLI/Web
- ✅ Address verification working
- ✅ QR codes embed correctly
- ✅ Stripe + BTCPay payments work
- ✅ Jobs submit to PostGrid after settlement
- ✅ Initial receipts generated + attested
- ✅ Tracking polls successfully
- ✅ Delivery receipts created + attested
- ✅ Full lifecycle tracked on blockchain

### Post-Phase 4 Goals
- ✅ Support international mail
- ✅ Multiple document formats
- ✅ Bulk mail job templates
- ✅ Compliance certifications (USPS, SOC 2)
- ✅ Advanced analytics (delivery rates, cost optimization)

---

## 9. Phases 1-4: Complete Success Criteria

### By End of Phase 4 (Week 36)
- ✅ Complete digital-to-physical pipeline
- ✅ Cryptographic lineage for all artifacts
- ✅ Payment settlement tracked
- ✅ Delivery confirmed on blockchain
- ✅ Audit trail for full lifecycle
- ✅ Organization branding support
- ✅ Multi-payment support (card + crypto)
- ✅ Tracking automation

### Platform Maturity: Full Lifecycle Solution
```
Attestation (Phase 1)
  + Management (Phase 2)
  + Monitoring (Phase 3)
  + Physicalization (Phase 4)
  = Complete Document Lifecycle Platform
```

---

**End DEVELOPMENT_PLAN.md (v3 with Phase 4)**