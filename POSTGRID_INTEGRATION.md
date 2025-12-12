# POSTGRID_INTEGRATION.md - Phase 4: Print & Mail API Integration

**PostGrid API Integration Guide for ArweaveStamp v3**

---

## Overview

This document specifies the complete integration between ArweaveStamp and the PostGrid Print & Mail API, enabling trustless physical delivery of blockchain-attested documents.

### Key Design Principles

1. **Separation of Concerns**: PostGrid handles print & mail logistics; ArweaveStamp handles cryptographic proof
2. **Zero PostGrid Trust**: PostGrid job ID + tracking data are attestable facts on Arweave, not security-critical
3. **Client-Side OCR**: Address extraction happens locally; PostGrid only receives verified data
4. **Deterministic QR Codes**: Same Arweave + Witnet references always produce same QR code
5. **Full Audit Trail**: Every state transition (job creation → submission → delivery) recorded on Arweave + Witnet

---

## Architecture

### Service Components

```
┌─────────────────┐
│   Mail Manager  │ (orchestrates job lifecycle)
├─────────────────┤
│  - Job creation │
│  - Payment flow │
│  - PostGrid API │
│  - Tracking     │
└────────┬────────┘
         │
    ┌────┴──────┬──────────┬──────────┐
    │            │          │          │
┌───▼──┐  ┌─────▼─┐  ┌────▼───┐  ┌──▼─────┐
│ OCR  │  │ QR    │  │Payment │  │Carrier │
│Svc   │  │ Codec │  │Gateway │  │ Track  │
└──────┘  └───────┘  └────────┘  └────────┘
    │            │          │          │
    └────┬───────┴──────────┴──────────┘
         │
    ┌────▼─────────────┐
    │  PostGrid API    │
    │  + Webhooks      │
    └──────────────────┘
```

### Data Model: Mail Job

```typescript
interface MailJob {
  jobId: string;                           // UUID
  organizationId: string;
  
  // Document Reference
  sourceDocumentArweaveId: string;         // Original document on Arweave
  sourceDocumentWitnetId: string;          // Original document attestation
  
  // Recipient Information
  recipientName: string;
  recipientAddress: string;                // Verified by PostGrid
  recipientAddressRaw: string;             // Before verification
  recipientAddressVerified: boolean;
  recipientAddressErrors?: string[];       // Corrections/flags from PostGrid
  
  // Sender Information
  senderName: string;
  senderAddress: string;
  senderLogoUrl?: string;                  // Organization logo
  senderFooter?: string;                   // Custom branding footer
  
  // Job Metadata
  jobTitle: string;
  jobDescription?: string;
  jobCreatedAt: Date;
  jobSubmittedAt?: Date;
  jobSubmittedBy: string;                  // User email/ID
  
  // Cryptographic Identifiers
  qrCodeData: {
    arweaveId: string;
    witnetId: string;
    jobReference: string;                  // jobId + checksum
  };
  
  // Cost Breakdown
  costBreakdown: {
    printCost: number;                     // USD
    postageCost: number;
    verificationFee: number;
    claudeMetadataFee: number;
    qrCodeFee: number;
    receiptStorageFee: number;
    witnetAttestationFee: number;
    totalCost: number;
  };
  
  // Payment
  paymentMethod: 'stripe' | 'crypto';
  paymentStatus: 'pending' | 'processing' | 'settled' | 'failed';
  paymentDetails: {
    stripeTransactionId?: string;
    stripeMaskedCard?: string;              // "Visa ****1234"
    btcpayInvoiceId?: string;
    btcpayTxHash?: string;
    btcpayConfirmations?: number;
  };
  
  // PostGrid Integration
  postgridJobId: string;                   // PostGrid internal job ID
  postgridStatus: PostGridJobStatus;
  postgridResponse: object;                // Full PostGrid response
  
  // Tracking
  carriers: Array<{
    name: string;                          // USPS, UPS, FedEx, etc.
    trackingNumber: string;
    trackingUrl: string;
    lastUpdatedAt: Date;
    currentStatus: string;                 // "in-transit", "delivered", etc.
  }>;
  
  // Delivery
  deliveryConfirmedAt?: Date;
  deliverySignedFor?: boolean;
  deliveryNotes?: string;
  
  // Receipts & Attestations
  mailingReceiptArweaveId?: string;        // Mailing receipt on Arweave
  mailingReceiptWitnetId?: string;         // Mailing receipt attestation
  deliveryReceiptArweaveId?: string;       // Delivery receipt on Arweave
  deliveryReceiptWitnetId?: string;        // Delivery receipt attestation
  
  // AI Metadata (Claude)
  aiMetadata: {
    jobSummary: string;
    riskFlags: string[];                   // e.g., "international", "signature_required"
    entities: Array<{ type: string; value: string }>;
    suggestedRetention: string;             // e.g., "permanent" or "90-days"
  };
}

type PostGridJobStatus = 
  | 'created'                              // Job created in PostGrid
  | 'submitted'                            // Sent to print facility
  | 'in_production'                        // Being printed
  | 'in_transit'                           // With carrier
  | 'delivered'                            // Delivered
  | 'failed'                               // Failed/undeliverable
  | 'cancelled';                           // Cancelled by user
```

---

## API Specification

### 1. Create Mail Job (POST /mail/jobs)

**Request**:
```typescript
interface CreateMailJobRequest {
  documentArweaveId: string;               // From Phase 1 attestation
  documentWitnetId: string;
  
  recipientName: string;
  recipientAddress: string;                // Will be OCR-verified
  
  jobTitle: string;
  jobDescription?: string;
  
  paymentMethod: 'stripe' | 'crypto';
  stripeToken?: string;                    // Required if paymentMethod='stripe'
  
  senderLogoUrl?: string;                  // Override organization default
  senderFooter?: string;
  
  metadata?: Record<string, unknown>;      // Custom metadata
}
```

**Response**:
```typescript
interface CreateMailJobResponse {
  jobId: string;
  status: 'confirmation_required';         // User must confirm
  
  // Address Verification Results
  addressVerification: {
    rawAddress: string;
    verifiedAddress: string;
    errors: string[];                      // e.g., "ZIP code invalid"
    corrections: string[];                 // e.g., "State corrected to TX"
    confidence: number;                    // 0-100 (PostGrid confidence score)
  };
  
  // Cost Breakdown
  costBreakdown: {
    printCost: number;
    postageCost: number;
    totalCost: number;
    estimatedDelivery: string;            // e.g., "3-5 business days"
  };
  
  // QR Code Preview
  qrCode: {
    content: string;                       // Base64-encoded PNG
    data: {
      arweaveUrl: string;
      witnetUrl: string;
      jobReference: string;
    };
  };
  
  confirmationUrl: string;                // Link to confirm job (web UI)
  expiresAt: Date;                        // Confirmation expires in 1 hour
}
```

**Error Handling**:
```typescript
// Address verification failed
{
  status: 400,
  error: 'ADDRESS_VERIFICATION_FAILED',
  message: 'Address could not be verified. Please correct and retry.',
  suggestion: 'Try: 456 Oak Avenue, Town, ST 67890'
}

// Payment method invalid
{
  status: 402,
  error: 'PAYMENT_FAILED',
  message: 'Stripe token expired or invalid.',
  retryable: true
}

// Document not found on Arweave
{
  status: 404,
  error: 'DOCUMENT_NOT_FOUND',
  message: 'Arweave document with ID ... not found.',
  suggestion: 'Verify documentArweaveId and attestation status.'
}
```

### 2. Confirm Mail Job (POST /mail/jobs/{jobId}/confirm)

**Request**:
```typescript
interface ConfirmMailJobRequest {
  confirmed: boolean;                      // true = proceed, false = cancel
  
  // Optional: override costs/branding
  senderLogoUrl?: string;
  senderFooter?: string;
  metadata?: Record<string, unknown>;
}
```

**Response**:
```typescript
interface ConfirmMailJobResponse {
  jobId: string;
  status: 'submitted_to_postgrid';
  
  postgridJobId: string;
  postgridResponse: {
    id: string;
    status: 'created';
    estimatedDeliveryDate: string;
  };
  
  // Tracking Info
  carriers: Array<{
    name: string;
    trackingNumber: string;
    trackingUrl: string;
  }>;
  
  // Receipts
  mailingReceiptUrl: string;              // Can download immediately
  estimatedDeliveryDate: string;
  
  // Next Steps
  trackingDashboardUrl: string;
  estimatedDeliveryDate: string;
}
```

### 3. Get Mail Job Status (GET /mail/jobs/{jobId})

**Response**:
```typescript
interface GetMailJobResponse extends MailJob {
  // All job fields + current tracking
  carriers: Array<{
    name: string;
    trackingNumber: string;
    trackingUrl: string;
    lastUpdatedAt: Date;
    currentStatus: string;
    scanEvents: Array<{
      timestamp: Date;
      location: string;
      status: string;
      details?: string;
    }>;
  }>;
  
  deliveryEstimate: {
    originalEstimate: Date;
    currentEstimate?: Date;
    delivered?: Date;
  };
}
```

### 4. Get Mailing Receipt (GET /mail/jobs/{jobId}/receipt)

**Query Params**:
```
format=json|pdf (default: pdf)
includeQrCodes=true|false (default: true)
```

**Response (JSON)**:
```typescript
interface MailingReceiptJSON {
  jobId: string;
  createdAt: Date;
  
  // Document List
  documents: Array<{
    arweaveId: string;
    arweaveUrl: string;
    witnetId: string;
    witnetUrl: string;
    title: string;
    fileHash: string;
  }>;
  
  // Sender/Recipient
  senderName: string;
  senderAddress: string;
  recipientName: string;
  recipientAddress: string;
  
  // Costs
  costBreakdown: {
    printCost: number;
    postageCost: number;
    cloudeFee: number;
    witnetAttestationFee: number;
    totalCost: number;
  };
  
  // Payment
  paymentMethod: string;
  paymentStatus: string;
  paymentDetails: {
    // Masked for security
    lastFourDigits?: string;
    btcpayTxHash?: string;
  };
  
  // PostGrid
  postgridJobId: string;
  carriers: Array<{
    name: string;
    trackingNumber: string;
    trackingUrl: string;
  }>;
  
  // Attestations
  mailingReceiptArweaveId: string;
  mailingReceiptWitnetId: string;
  
  // QR Codes (embedded as data URIs in JSON, full images in PDF)
  qrCodes: Array<{
    documentArweaveId: string;
    qrCodeDataUri: string;
    encodedData: string;
  }>;
}
```

**Response (PDF)**:
- Professional receipt document
- Organization logo at top
- Sender/recipient address sections
- Document list with embedded QR codes
- Carrier tracking info
- Itemized cost breakdown
- Payment method (masked)
- Attestation IDs + links
- QR code linking to verification portal

### 5. Get Delivery Receipt (GET /mail/jobs/{jobId}/delivery-receipt)

**Query Params**: Same as mailing receipt (format, includeQrCodes)

**Response**:
- Similar to mailing receipt
- Plus: Delivery date/time
- Plus: Carrier delivery notes
- Plus: Signature confirmation (if available)
- Plus: Delivery receipt Arweave ID + Witnet ID

---

## OCR Address Verification

### Client-Side Address Extraction

```typescript
interface OCRAddressExtractionRequest {
  documentPath: string;                    // Local file path or data URI
  documentFormat: 'pdf' | 'image';
  addressBlockRegion?: {                   // Optional: hint where address is
    x1: number;
    y1: number;
    x2: number;
    y2: number;
  };
}

interface OCRAddressExtractionResponse {
  success: boolean;
  extractedAddress?: string;               // e.g., "456 Oak Ave, Town, ST 67890"
  confidence: number;                      // 0-100
  alternatives?: string[];                 // Other potential addresses found
  errors?: string[];
}
```

### OCR Implementation Strategy

1. **Local-first**: Use Tesseract.js (open-source) for client-side OCR
2. **No transmission**: Address extracted locally, never sent to ArweaveStamp servers
3. **User verification**: Always display extracted address; allow manual override
4. **PostGrid verification**: Only verified address sent to PostGrid

```typescript
// Example implementation
import Tesseract from 'tesseract.js';

async function extractAddress(pdfPath: string): Promise<string> {
  // Step 1: Extract text from PDF
  const text = await Tesseract.recognize(pdfPath, 'eng');
  
  // Step 2: Pattern match for address block
  // Look for: street, city, state, zip patterns
  const addressRegex = /(\d+[\w\s]+(?:St|Ave|Blvd|Rd|Dr|Ln).*?)\n([\w\s]+),\s([A-Z]{2})\s(\d{5})/;
  const match = text.data.text.match(addressRegex);
  
  // Step 3: Return extracted address
  return match ? match[0] : null;
}
```

### PostGrid Address Verification API

```typescript
interface PostGridVerifyAddressRequest {
  address: string;
  country?: string;                        // Default: 'USA'
}

interface PostGridVerifyAddressResponse {
  address: string;                         // Corrected if necessary
  confidence: number;                      // 0-100
  errors: string[];                        // List of issues found
  corrections: string[];                   // Applied corrections
  deliverable: boolean;                    // PostGrid can deliver to this
  suggestions: string[];                   // Alternative formats
}
```

---

## QR Code Design & Encoding

### QR Code Content Schema

```typescript
interface QRCodePayload {
  // Protocol version
  version: 'arweave-stamp/v1';
  
  // Document References
  documentArweaveId: string;
  documentWitnetId: string;
  
  // Mail Job Reference
  jobId: string;
  jobChecksum: string;                     // SHA256(jobId + secret)
  
  // Verification Portal
  verificationPortalUrl: string;
  
  // Optional: Short links (if size constrained)
  shortArweaveUrl?: string;
  shortWitnetUrl?: string;
}

// Encoded as JSON, then as QR code
// Example QR size: ~300x300px at 15-20% error correction
```

### QR Code Embedding

```typescript
interface EmbedQRCodeRequest {
  pdfPath: string;                         // PostGrid print-ready PDF
  qrCode: Buffer;                          // PNG/SVG QR image
  position: 'bottom-right' | 'bottom-center' | 'footer';
  scale: number;                           // 1.0 = 1-inch, 0.5 = 0.5-inch
}

// Implementation: Use PDFKit or similar to inject QR before sending to PostGrid
```

---

## Payment Settlement

### Stripe Flow

```
1. User provides Stripe token via web UI
   ↓
2. Create Stripe charge
   - Amount: totalCost (in cents)
   - Metadata: jobId, organizationId, documentArweaveId
   ↓
3. Charge succeeds
   ↓
4. Update job status: paymentStatus='settled'
   ↓
5. Submit to PostGrid
```

### BTCPay Flow

```
1. User selects "crypto" payment
   ↓
2. Create BTCPay invoice
   - Amount: totalCost (in USD)
   - Webhook: callback on payment received
   - ExpirationMinutes: 60
   ↓
3. User sees QR + address
   ↓
4. User sends crypto
   ↓
5. BTCPay webhook fires (payment received)
   - confirmations: 1+ (configurable)
   ↓
6. Update job status: paymentStatus='settled'
   ↓
7. Submit to PostGrid
```

---

## Webhook Handling

### PostGrid Webhooks

Subscribe to PostGrid events via webhook:

```
POST /webhooks/postgrid
```

**Payload**:
```typescript
interface PostGridWebhookPayload {
  event: 'job.created' | 'job.submitted' | 'job.failed' | 'tracking.updated' | 'tracking.delivered';
  jobId: string;
  timestamp: Date;
  data: {
    status: string;
    trackingInfo?: {
      carrier: string;
      trackingNumber: string;
      lastScan?: Date;
    };
    errorMessage?: string;
  };
}
```

**Handler**:
```typescript
async function handlePostGridWebhook(payload: PostGridWebhookPayload) {
  const job = await db.getMailJob(payload.jobId);
  
  // Update job status
  job.postgridStatus = payload.data.status;
  
  if (payload.event === 'tracking.delivered') {
    job.deliveryConfirmedAt = new Date();
    await generateDeliveryReceipt(job);
    await attestDeliveryOnWitnet(job);
  }
  
  await db.saveMailJob(job);
}
```

### BTCPay Webhooks

```
POST /webhooks/btcpay
```

**Payload**:
```typescript
interface BTCPayWebhookPayload {
  event: 'invoice_received' | 'invoice_confirmed' | 'invoice_complete' | 'invoice_failed';
  invoiceId: string;
  status: string;
  amount: number;
  currency: string;
  txnHash?: string;
  confirmations?: number;
  timestamp: Date;
}
```

---

## State Machine

### Job State Transitions

```
┌────────────┐
│ created    │ (user initiates)
└─────┬──────┘
      │ [address_verified]
      ▼
┌────────────────┐
│ awaiting_      │
│ confirmation   │
└─────┬──────────┘
      │ [user_confirmed + payment_settled]
      ▼
┌────────────────┐
│ submitted_to_  │
│ postgrid       │ (waiting for PostGrid confirmation)
└─────┬──────────┘
      │ [postgrid_acknowledged]
      ▼
┌────────────────┐
│ in_production  │ (PostGrid printing)
└─────┬──────────┘
      │ [shipped_by_carrier]
      ▼
┌────────────────┐
│ in_transit     │ (carrier has package)
└─────┬──────────┘
      │ [carrier_delivered]
      ▼
┌────────────────┐
│ delivered      │ (delivered to recipient)
└────────────────┘

[Error states]
  ├─ address_verification_failed
  ├─ payment_failed
  ├─ postgrid_submission_failed
  ├─ carrier_delivery_failed
  └─ cancelled_by_user
```

---

## Background Tasks

### Carrier Tracking Polling (Every 12 Hours)

```typescript
async function pollCarrierTracking() {
  const activeJobs = await db.getMailJobsByStatus('in_transit');
  
  for (const job of activeJobs) {
    for (const carrier of job.carriers) {
      // Poll carrier API via PostGrid
      const tracking = await postgridClient.getTracking(
        carrier.trackingNumber,
        carrier.name
      );
      
      // Update job
      job.carriers[carrier.index].currentStatus = tracking.status;
      job.carriers[carrier.index].scanEvents = tracking.events;
      job.carriers[carrier.index].lastUpdatedAt = new Date();
      
      // If delivered, generate delivery receipt
      if (tracking.status === 'delivered') {
        await generateDeliveryReceipt(job);
        await attestDeliveryOnWitnet(job);
        job.deliveryConfirmedAt = new Date();
      }
    }
    
    await db.saveMailJob(job);
  }
}

// Schedule: cron job every 12 hours
```

---

## Error Handling & Retries

### Retry Policy

| Operation | Max Retries | Backoff | Terminal Errors |
|-----------|-------------|---------|-----------------|
| Address verification | 3 | exponential (1s, 2s, 4s) | No valid address found |
| Stripe charge | 1 | none | Card declined, invalid token |
| BTCPay invoice | 0 | none | N/A (user retries) |
| PostGrid submit | 3 | exponential | Invalid document format |
| Carrier tracking | unlimited | 12h intervals | Carrier API down (graceful degrade) |
| Arweave upload | 5 | exponential | Network error (retry) |
| Witnet attestation | 3 | exponential | Witness unavailable |

### Fallback Behaviors

```typescript
// If carrier tracking unavailable, show last known status
if (trackingError) {
  job.carriers[i].currentStatus = job.carriers[i].lastKnownStatus;
  job.carriers[i].note = 'Tracking unavailable; last update: ' + lastUpdateTime;
}

// If Claude metadata generation fails, skip metadata
try {
  job.aiMetadata = await generateClaudeMetadata(job);
} catch (e) {
  console.warn('Claude metadata failed:', e);
  job.aiMetadata = null;  // Continue anyway
}

// If Arweave upload fails, queue for retry
try {
  job.deliveryReceiptArweaveId = await uploadToArweave(receipt);
} catch (e) {
  queue.add('arweave_upload_retry', { jobId: job.id });
}
```

---

## Security Considerations

### Data Privacy

- **PII Handling**: Addresses stored encrypted in database
- **OCR Local**: Address extraction never leaves user's device
- **PostGrid Scope**: PostGrid knows only recipient address + print-ready PDF
- **Arweave Scope**: Arweave stores mailing receipt (address included) but cannot be modified post-delivery

### Cryptographic Verification

- **QR Code Integrity**: HMAC-SHA256 checksum in QR payload
- **Receipt Signature**: Digital signature on PDF receipts
- **Delivery Proof**: Carrier tracking linked to Witnet attestation

### PCI Compliance

- **Stripe**: PCI-DSS compliant; use Stripe tokens only (never raw card data)
- **BTCPay**: Self-hosted or managed provider; ArweaveStamp never sees private keys

---

## Testing Strategy

### Unit Tests

```bash
npm run test -- mail/
```

Coverage targets:
- Address verification: 95%
- QR code generation: 100%
- Payment gateway: 90%
- State machine: 100%

### Integration Tests

```bash
npm run test:integration -- --suite mail
```

- PostGrid sandbox integration
- Stripe test mode
- BTCPay testnet
- Carrier tracking mock

### End-to-End Tests

```bash
npm run test:e2e -- --suite mail
```

- Create job → confirm → submit → track → deliver (full flow)
- Address verification error handling
- Payment failure recovery
- Carrier tracking polling

---

## Monitoring & Observability

### Metrics

- `mail_job_creation_rate` (jobs/hour)
- `mail_job_confirmation_rate` (confirmations/submissions)
- `payment_settlement_time` (seconds, by method)
- `carrier_tracking_availability` (% of active jobs with current tracking)
- `delivery_time_accuracy` (estimated vs actual)

### Logging

```typescript
// Structured logging for each major operation
logger.info('mail_job_created', {
  jobId: job.id,
  organizationId: org.id,
  documentArweaveId: job.sourceDocumentArweaveId,
  costEstimate: job.costBreakdown.totalCost,
});

logger.info('payment_settled', {
  jobId: job.id,
  method: job.paymentMethod,
  amount: job.costBreakdown.totalCost,
  processingTime: Date.now() - job.jobCreatedAt,
});

logger.info('delivery_confirmed', {
  jobId: job.id,
  carriers: job.carriers.map(c => c.name),
  deliveryTime: job.deliveryConfirmedAt - job.jobSubmittedAt,
});
```

---

## API Response Examples

### Success: Create Job

```json
{
  "status": "confirmation_required",
  "jobId": "job_abc123xyz",
  "addressVerification": {
    "rawAddress": "456 oak ave town st 67890",
    "verifiedAddress": "456 Oak Avenue, Town, ST 67890",
    "confidence": 95,
    "corrections": ["Avenue capitalized", "State code corrected"]
  },
  "costBreakdown": {
    "printCost": 0.75,
    "postageCost": 0.55,
    "claudeMetadataFee": 0.10,
    "witnetAttestationFee": 0.05,
    "totalCost": 1.45
  },
  "qrCode": {
    "content": "iVBORw0KGgoAAAANS...",
    "data": {
      "arweaveUrl": "https://arweave.net/abc123",
      "witnetUrl": "https://explorer.witnet.io/requests/123",
      "jobReference": "job_abc123xyz_checksum"
    }
  },
  "confirmationUrl": "https://app.arweave-stamp.io/mail/confirm/job_abc123xyz",
  "expiresAt": "2025-12-12T09:02:00Z"
}
```

### Success: Confirm Job

```json
{
  "status": "submitted_to_postgrid",
  "jobId": "job_abc123xyz",
  "postgridJobId": "pg_job_456def789",
  "carriers": [
    {
      "name": "USPS",
      "trackingNumber": "9400111899223456789012",
      "trackingUrl": "https://tools.usps.com/go/TrackConfirmAction?tLabels=9400111899223456789012"
    }
  ],
  "mailingReceiptUrl": "https://app.arweave-stamp.io/mail/jobs/job_abc123xyz/receipt?format=pdf",
  "estimatedDeliveryDate": "2025-12-16",
  "trackingDashboardUrl": "https://app.arweave-stamp.io/mail/jobs/job_abc123xyz"
}
```

### Error: Address Verification Failed

```json
{
  "status": 400,
  "error": "ADDRESS_VERIFICATION_FAILED",
  "message": "Address could not be verified by PostGrid",
  "details": {
    "rawAddress": "456 oak ave town st",
    "issues": ["ZIP code missing", "State code invalid"],
    "suggestions": [
      "456 Oak Avenue, Town, ST 67890",
      "456 Oak Avenue, Townville, TX 67890"
    ]
  }
}
```

---

## Future Enhancements

- Batch mail jobs (multiple recipients in one submission)
- Template system (pre-designed layouts)
- International shipping support
- Return mail handling
- Certified delivery / signature required
- Insurance coverage options

---

**Last Updated**: December 2025  
**Status**: Phase 4 Specification  
**Next**: Implement PostGrid API wrapper + OCR service
