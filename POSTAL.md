# POSTAL.md - Phase 4 PostGrid Integration Guide
**Print & Mail with Embedded Cryptographic Lineage**

---

## Overview

This document specifies the complete PostGrid integration for ArweaveStamp Phase 4, enabling "paper twin" creation of digitally attested documents with full cryptographic lineage, address verification, payment processing, and delivery tracking.

---

## Part 1: PostGrid API Integration

### PostGrid Account Setup

1. **Create PostGrid Account**: [https://www.postgrid.io](https://www.postgrid.io)
2. **Generate API Key**: Dashboard → Settings → API Keys
3. **Set Up Organization**: Dashboard → Organizations
4. **Configure Defaults**: Default mail class, paper type, finish

### PostGrid API Endpoints

**Base URL**: `https://api.postgrid.io/v1`

**Authentication**: Bearer token in Authorization header

**Key Endpoints**:
```
POST   /mailings               # Submit a mailing job
GET    /mailings/{id}          # Get mailing status
GET    /mailings/{id}/tracking # Get tracking info
POST   /addressVerifications   # Verify address format
GET    /events                 # Get webhook events
```

### PostGrid Client Implementation

```typescript
// src/postal/client.ts

import axios, { AxiosInstance } from 'axios';

export class PostGridClient {
  private client: AxiosInstance;

  constructor(apiKey: string) {
    this.client = axios.create({
      baseURL: 'https://api.postgrid.io/v1',
      headers: {
        Authorization: `Bearer ${apiKey}`,
        'Content-Type': 'application/json',
      },
    });
  }

  /**
   * Verify a postal address
   * @param address - Address to verify
   * @returns Verified address with corrections (if any)
   */
  async verifyAddress(address: PostalAddress): Promise<VerifyAddressResponse> {
    try {
      const response = await this.client.post('/addressVerifications', {
        streetLine1: address.streetLine1,
        streetLine2: address.streetLine2,
        city: address.city,
        stateProvince: address.stateProvince,
        postalCode: address.postalCode,
        country: address.country,
      });

      return {
        verified: response.data.verificationStatus === 'verified',
        corrected: response.data.correctedAddress,
        errors: response.data.addressErrors || [],
      };
    } catch (error: any) {
      if (error.response?.status === 400) {
        throw new AddressVerificationError(
          'Invalid address format',
          error.response.data.errors
        );
      }
      throw error;
    }
  }

  /**
   * Submit a mailing job to PostGrid
   * @param job - Mail job configuration
   * @returns PostGrid job ID + tracking info
   */
  async submitMailing(job: PostGridMailingRequest): Promise<PostGridMailingResponse> {
    try {
      const response = await this.client.post('/mailings', {
        recipients: job.recipients.map((r) => ({
          addressLine1: r.address.streetLine1,
          addressLine2: r.address.streetLine2,
          city: r.address.city,
          state: r.address.stateProvince,
          zipCode: r.address.postalCode,
          country: r.address.country,
          name: r.name,
        })),
        front: job.frontPdfUrl,  // URL to front PDF (with QR codes)
        back: job.backPdfUrl,    // Optional back side
        mailClass: 'usps_first_class_mail',
        mailType: 'letter',
      });

      return {
        postgridJobId: response.data.id,
        status: response.data.status,
        trackingNumbers: response.data.trackingNumbers || [],
        carrier: response.data.carrier,
        costCents: response.data.costCents,
        expectedDeliveryDate: response.data.expectedDeliveryDate,
      };
    } catch (error: any) {
      logger.error('PostGrid submission failed', { error });
      throw new PostGridError('Failed to submit mailing', error);
    }
  }

  /**
   * Get mailing status and tracking
   */
  async getMailingStatus(postgridJobId: string): Promise<MailingStatusResponse> {
    try {
      const response = await this.client.get(`/mailings/${postgridJobId}`);
      return {
        status: response.data.status,
        trackingNumbers: response.data.trackingNumbers,
        carrier: response.data.carrier,
        expectedDelivery: response.data.expectedDeliveryDate,
        events: response.data.events?.map((e: any) => ({
          timestamp: e.timestamp,
          status: e.status,
          location: e.location,
          details: e.details,
        })),
      };
    } catch (error: any) {
      throw new PostGridError('Failed to get mailing status', error);
    }
  }
}
```

---

## Part 2: Address Extraction and Verification

### OCR Address Extraction

```typescript
// src/mail/address.ts

import Tesseract from 'tesseract.js';

/**
 * Extract address block from document using OCR
 * @param pdfPath - Path to document
 * @returns Extracted address with OCR confidence
 */
export async function extractAddressFromDocument(
  pdfPath: string
): Promise<ExtractedAddress> {
  // Convert PDF to images
  const images = await convertPdfToImages(pdfPath);

  // Look for address pattern (runs OCR on full page)
  let bestAddress: ExtractedAddress | null = null;
  let highestConfidence = 0;

  for (const image of images) {
    const result = await Tesseract.recognize(image, 'eng');
    const text = result.data.text;

    // Extract address using regex patterns
    const address = parseAddressFromText(text);
    if (address && result.data.confidence > highestConfidence) {
      bestAddress = address;
      highestConfidence = result.data.confidence;
    }
  }

  if (!bestAddress) {
    throw new Error('No address found in document');
  }

  return {
    ...bestAddress,
    ocrConfidence: highestConfidence,
    extractedAt: new Date().toISOString(),
  };
}

/**
 * Parse USPS-formatted address from OCR text
 * Supports formats like:
 *   "123 Main Street\nCity, State 12345"
 *   "Jane Doe\n123 Main St\nCity, ST 12345"
 */
function parseAddressFromText(text: string): PostalAddress | null {
  const lines = text.split('\n').map((l) => l.trim());

  // Look for ZIP code (5 or 9 digits)
  const zipMatch = text.match(/\b(\d{5}(?:-\d{4})?)\b/);
  if (!zipMatch) return null;

  const zipCode = zipMatch[1];

  // Extract state (2 letters before ZIP)
  const stateMatch = text.match(/\b([A-Z]{2})\s+\d{5}/);
  const stateProvince = stateMatch ? stateMatch[1] : '';

  // Extract city (word before state)
  const cityMatch = text.match(/(.+?),\s*[A-Z]{2}/);
  const city = cityMatch ? cityMatch[1].trim() : '';

  // Extract street (first line or line before city)
  const streetLine1 = lines[0];

  return {
    streetLine1,
    city,
    stateProvince,
    postalCode: zipCode,
    country: 'US',
    verified: false,
  };
}

/**
 * Call PostGrid address verification API
 * @param address - Address to verify
 * @returns Verification result with corrections
 */
export async function verifyAddressWithPostGrid(
  address: PostalAddress,
  postgridClient: PostGridClient
): Promise<AddressVerificationResult> {
  try {
    const verified = await postgridClient.verifyAddress(address);

    return {
      originalAddress: address,
      verifiedAddress: verified.corrected || address,
      isVerified: verified.verified,
      errors: verified.errors,
      corrections: verified.corrected
        ? compareAddresses(address, verified.corrected)
        : [],
    };
  } catch (error: any) {
    if (error instanceof AddressVerificationError) {
      return {
        originalAddress: address,
        verifiedAddress: address,
        isVerified: false,
        errors: error.errors,
        corrections: [],
      };
    }
    throw error;
  }
}

/**
 * Compare original vs corrected address
 */
function compareAddresses(original: PostalAddress, corrected: PostalAddress): string[] {
  const changes: string[] = [];

  if (original.streetLine1 !== corrected.streetLine1) {
    changes.push(`Street: ${original.streetLine1} → ${corrected.streetLine1}`);
  }
  if (original.city !== corrected.city) {
    changes.push(`City: ${original.city} → ${corrected.city}`);
  }
  if (original.stateProvince !== corrected.stateProvince) {
    changes.push(`State: ${original.stateProvince} → ${corrected.stateProvince}`);
  }
  if (original.postalCode !== corrected.postalCode) {
    changes.push(`ZIP: ${original.postalCode} → ${corrected.postalCode}`);
  }

  return changes;
}
```

---

## Part 3: QR Code Generation with Cryptographic Lineage

### QR Code Data Structure

```typescript
// src/mail/qrcodes.ts

interface QRCodePayload {
  // Document references
  documentId: string;                 // From ProofPackage
  arweaveUrl: string;                 // https://arweave.net/{txId}
  arweaveTxId: string;
  
  // Attestation references
  witnetRequestId: string;
  witnetReportId: string;
  
  // Optional: Receipt reference (added after printing)
  receiptArweaveUrl?: string;
  receiptWitnetId?: string;
  
  // Verification metadata
  attestedAt: ISO8601;
  verifyUrl: string;                  // URL to verification portal
}

/**
 * Generate QR code with embedded Arweave/Witnet references
 * @param payload - Data to encode
 * @returns QR code as data URL
 */
export async function generateQRCode(payload: QRCodePayload): Promise<string> {
  const json = JSON.stringify(payload);

  // Check size (QR code can hold ~4,296 characters for alphanumeric)
  if (json.length > 4000) {
    logger.warn('QR payload exceeds recommended size', {
      size: json.length,
      payload: payload,
    });
  }

  const qr = qrcode.toDataURL(json, {
    errorCorrectionLevel: 'H',  // High error correction
    type: 'image/png',
    width: 300,                 // 300x300 pixels
    margin: 1,
    color: {
      dark: '#000000',
      light: '#FFFFFF',
    },
  });

  return qr;
}

/**
 * Embed QR code into PDF document
 * @param pdfPath - Path to document PDF
 * @param qrDataUrl - QR code as data URL
 * @param position - Where to place QR (top-left, bottom-right, etc.)
 * @returns Modified PDF with embedded QR
 */
export async function embedQRCodeInPDF(
  pdfPath: string,
  qrDataUrl: string,
  position: 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right'
): Promise<Buffer> {
  const PDFDocument = require('pdfkit');
  const fs = require('fs');

  const doc = new PDFDocument();
  const output = Buffer.alloc(0);

  // Read existing PDF
  const existingPdf = fs.readFileSync(pdfPath);
  const pdfParser = require('pdf-parse');
  await pdfParser(existingPdf);

  // Merge existing PDF + QR code overlay
  const merger = new (require('pdf-merge'))();
  merger.add(pdfPath);

  // Create overlay PDF with QR
  const overlayDoc = new PDFDocument({
    size: 'letter',
    margin: 0,
  });

  // Position QR code based on parameter
  const qrSize = 100; // 100x100 pixels
  let x = 20, y = 20; // Default: top-left

  if (position === 'top-right') {
    x = 595 - 20 - qrSize; // Letter width - margin - QR size
  } else if (position === 'bottom-left') {
    y = 792 - 20 - qrSize; // Letter height - margin - QR size
  } else if (position === 'bottom-right') {
    x = 595 - 20 - qrSize;
    y = 792 - 20 - qrSize;
  }

  // Convert data URL to buffer
  const qrBuffer = Buffer.from(
    qrDataUrl.replace(/^data:image\/png;base64,/, ''),
    'base64'
  );

  overlayDoc.image(qrBuffer, x, y, { width: qrSize, height: qrSize });
  overlayDoc.end();

  // Merge PDFs
  const mergedPdf = await merger.asBuffer();
  return mergedPdf;
}
```

---

## Part 4: Receipt Generation and Attestation

### Receipt Generation

```typescript
// src/mail/receipts.ts

/**
 * Generate initial receipt PDF
 * Created after printing, before delivery
 */
export async function generateInitialReceipt(
  mailJob: MailJob
): Promise<Buffer> {
  const PDFDocument = require('pdfkit');
  const doc = new PDFDocument();

  // Header
  doc.fontSize(20).text('Mail Receipt', { align: 'center' });
  doc.fontSize(10).text(`Job ID: ${mailJob.id}`, { align: 'center' });
  doc.fontSize(10).text(`Date: ${new Date(mailJob.createdAt).toLocaleDateString()}`, {
    align: 'center',
  });

  doc.moveTo(50, 100).lineTo(550, 100).stroke();

  // Sender info
  doc.fontSize(12).text('From:');
  doc.fontSize(10).text(mailJob.sender.name);
  doc.text(formatAddress(mailJob.sender.address));

  doc.moveTo(50, 180).lineTo(550, 180).stroke();

  // Recipients
  doc.fontSize(12).text('Mailing To:');
  mailJob.recipients.forEach((recipient, idx) => {
    doc.fontSize(10).text(`${idx + 1}. ${recipient.name}`);
    doc.fontSize(9).text(formatAddress(recipient.address));
  });

  doc.moveTo(50, 280).lineTo(550, 280).stroke();

  // Documents
  doc.fontSize(12).text('Documents:');
  mailJob.documents.forEach((doc_ref, idx) => {
    doc.fontSize(10).text(`${idx + 1}. ${doc_ref.title}`);
    doc.fontSize(8).text(`Arweave: ${doc_ref.arweaveUrl}`, { color: 'blue' });
    doc.fontSize(8).text(`Witnet: ${doc_ref.witnetAttestationId}`);
  });

  doc.moveTo(50, 400).lineTo(550, 400).stroke();

  // Mailing details
  doc.fontSize(12).text('Mailing Details:');
  doc.fontSize(10).text(`Carrier: ${mailJob.postgrid.carrier}`);
  doc.fontSize(10).text(`Tracking: ${mailJob.postgrid.trackingNumbers?.join(', ')}`);
  doc.fontSize(10).text(
    `Estimated Delivery: ${formatDate(mailJob.tracking.estimatedDelivery)}`
  );

  doc.moveTo(50, 480).lineTo(550, 480).stroke();

  // Payment
  doc.fontSize(12).text('Payment:');
  doc.fontSize(10).text(`Amount: $${(mailJob.payment.actualCost / 100).toFixed(2)}`);
  doc.fontSize(10).text(`Method: ${mailJob.payment.method}`);
  doc.fontSize(10).text(`Status: ${mailJob.payment.status}`);

  if (mailJob.payment.method === 'stripe') {
    doc.fontSize(9).text(`Charge ID: ${mailJob.payment.transactionId}`);
  } else if (mailJob.payment.method === 'btcpay') {
    doc.fontSize(9).text(`Invoice ID: ${mailJob.payment.transactionId}`);
    doc.fontSize(9).text(`TX Hash: ${mailJob.payment.transactionHash}`);
  }

  doc.moveTo(50, 580).lineTo(550, 580).stroke();

  // Attestation QR
  doc.fontSize(12).text('Blockchain Attestation:');
  if (mailJob.receipts.initialReceiptArweaveUrl) {
    doc.fontSize(9).text(
      `Receipt: ${mailJob.receipts.initialReceiptArweaveUrl}`,
      { color: 'blue' }
    );
    doc.fontSize(9).text(`Witnet: ${mailJob.receipts.initialReceiptWitnetId}`);
  } else {
    doc.fontSize(9).text('(Pending attestation after printing)');
  }

  return doc.end();
}

/**
 * Generate delivery receipt PDF
 * Created after delivery confirmation
 */
export async function generateDeliveryReceipt(
  mailJob: MailJob,
  initialReceiptUrl: string
): Promise<Buffer> {
  const PDFDocument = require('pdfkit');
  const doc = new PDFDocument();

  doc.fontSize(20).text('Delivery Receipt', { align: 'center' });
  doc.fontSize(10).text(`Job ID: ${mailJob.id}`, { align: 'center' });

  doc.moveTo(50, 100).lineTo(550, 100).stroke();

  // Delivery status
  doc.fontSize(12).text('Delivery Status:');
  doc.fontSize(10).text(`Status: ${mailJob.tracking.status}`);
  doc.fontSize(10).text(`Delivered: ${formatDate(mailJob.tracking.deliveredAt)}`);

  // Tracking events
  doc.fontSize(12).text('Tracking Events:');
  mailJob.tracking.events.forEach((event) => {
    doc.fontSize(9).text(`${formatDate(event.timestamp)}: ${event.status}`);
    if (event.location) doc.fontSize(8).text(`Location: ${event.location}`);
  });

  doc.moveTo(50, 300).lineTo(550, 300).stroke();

  // References
  doc.fontSize(12).text('References:');
  doc.fontSize(9).text(`Initial Receipt: ${initialReceiptUrl}`, { color: 'blue' });
  doc.fontSize(9).text(`Original Documents:`);
  mailJob.documents.forEach((doc_ref) => {
    doc.fontSize(8).text(`  ${doc_ref.arweaveUrl}`, { color: 'blue' });
  });

  doc.moveTo(50, 450).lineTo(550, 450).stroke();

  // Blockchain attestation
  doc.fontSize(12).text('Blockchain Attestation:');
  if (mailJob.receipts.deliveryReceiptArweaveUrl) {
    doc.fontSize(9).text(
      `Receipt: ${mailJob.receipts.deliveryReceiptArweaveUrl}`,
      { color: 'blue' }
    );
    doc.fontSize(9).text(`Witnet: ${mailJob.receipts.deliveryReceiptWitnetId}`);
  }

  return doc.end();
}
```

### Receipt Upload and Attestation

```typescript
/**
 * Upload receipt to Arweave and create Witnet attestation
 */
export async function attestReceipt(
  receipt: MailReceipt,
  arweaveClient: ArweaveClient,
  witnetClient: WitnetClient,
  claudeClient: AnthropicClient
): Promise<ReceiptAttestation> {
  // Generate AI metadata for receipt
  const metadata = await claudeClient.generateMetadata({
    type: 'mail_receipt',
    summary: `Mailed ${receipt.jobSummary.documentCount} documents to ${receipt.jobSummary.recipientCount} recipients`,
    tags: ['physical-mail', 'receipt', `payment-${receipt.payment.method}`],
  });

  // Create JSON payload
  const payload = {
    receiptId: receipt.id,
    mailJobId: receipt.mailJobId,
    createdAt: receipt.createdAt,
    jobSummary: receipt.jobSummary,
    documents: receipt.documents,
    payment: receipt.payment,
    metadata,
  };

  // Upload to Arweave
  const arweaveResult = await arweaveClient.uploadJSON(payload, {
    tags: [
      { name: 'Content-Type', value: 'application/json' },
      { name: 'receipt-type', value: 'mail-initial' },
      { name: 'mail-job-id', value: receipt.mailJobId },
    ],
  });

  // Create Witnet attestation
  const witnetResult = await witnetClient.submitRequest({
    documentId: receipt.id,
    arweaveUrl: arweaveResult.url,
    payload: payload,
  });

  return {
    arweaveUrl: arweaveResult.url,
    arweaveTxId: arweaveResult.txId,
    witnetRequestId: witnetResult.requestId,
    witnetReportId: witnetResult.reportId,
    attestedAt: new Date().toISOString(),
  };
}
```

---

## Part 5: Tracking Service

### Polling and Update Logic

```typescript
// src/tracking/poller.ts

import cron from 'node-cron';

/**
 * Start background tracking polling service
 * Runs every 12 hours to check carrier status
 */
export function startTrackingPoller(
  postgridClient: PostGridClient,
  mailJobService: MailJobService
): NodeJS.Timer {
  // Run every 12 hours at midnight and noon
  const job = cron.schedule('0 0,12 * * *', async () => {
    logger.info('Starting tracking poll cycle');

    try {
      // Get all non-final mail jobs
      const pendingJobs = await mailJobService.findPending();

      for (const mailJob of pendingJobs) {
        if (!mailJob.postgrid.jobId) continue;

        try {
          // Get updated tracking from PostGrid
          const status = await postgridClient.getMailingStatus(mailJob.postgrid.jobId);

          // Process updates
          await processTrackingUpdate(mailJob, status, mailJobService);
        } catch (error) {
          logger.error('Failed to update tracking for job', {
            jobId: mailJob.id,
            error,
          });
        }
      }

      logger.info('Tracking poll cycle complete');
    } catch (error) {
      logger.error('Tracking poll cycle failed', { error });
    }
  });

  return job;
}

/**
 * Process tracking update from PostGrid
 */
async function processTrackingUpdate(
  mailJob: MailJob,
  status: MailingStatusResponse,
  mailJobService: MailJobService
): Promise<void> {
  // Update tracking events
  const newEvents = (status.events || []).filter(
    (event) =>
      !mailJob.tracking.events.some(
        (e) => e.timestamp === event.timestamp && e.status === event.status
      )
  );

  if (newEvents.length > 0) {
    mailJob.tracking.events.push(...newEvents);
    mailJob.tracking.lastUpdate = new Date().toISOString();

    // Check if delivered
    const isDelivered = status.status === 'delivered' || status.status === 'failed';

    if (isDelivered && mailJob.tracking.status !== 'delivered') {
      mailJob.tracking.status = status.status;
      mailJob.tracking.deliveredAt = new Date().toISOString();

      // Generate delivery receipt
      await generateAndAttestDeliveryReceipt(mailJob);

      // Notify user
      await notifyDelivery(mailJob);
    }

    // Persist updates
    await mailJobService.update(mailJob);
  }
}

/**
 * Generate and attest delivery receipt
 */
async function generateAndAttestDeliveryReceipt(
  mailJob: MailJob
): Promise<void> {
  const deliveryReceiptPdf = await generateDeliveryReceipt(
    mailJob,
    mailJob.receipts.initialReceiptArweaveUrl || ''
  );

  // Create DeliveryReceipt object
  const deliveryReceipt: DeliveryReceipt = {
    id: `delivery-${mailJob.id}`,
    mailJobId: mailJob.id,
    initialReceiptArweaveUrl: mailJob.receipts.initialReceiptArweaveUrl || '',

    deliveryDetails: {
      status: mailJob.tracking.status as 'delivered' | 'failed',
      deliveredAt: mailJob.tracking.deliveredAt,
    },

    trackingData: {
      carrier: mailJob.postgrid.carrier || 'unknown',
      trackingNumbers: mailJob.postgrid.trackingNumbers || [],
      events: mailJob.tracking.events,
    },

    attestation: {
      referencesDocuments: mailJob.documents.map((d) => d.arweaveUrl),
      referencesInitialReceipt: mailJob.receipts.initialReceiptArweaveUrl || '',
      arweaveUrl: '',
      arweaveTxId: '',
      witnetRequestId: '',
      witnetReportId: '',
      attestedAt: new Date().toISOString(),
    },
  };

  // Attest to blockchain
  const attestation = await attestDeliveryReceipt(deliveryReceipt);
  mailJob.receipts.deliveryReceiptArweaveUrl = attestation.arweaveUrl;
  mailJob.receipts.deliveryReceiptWitnetId = attestation.witnetReportId;
}
```

---

## Part 6: CLI and Web UI Integration

### CLI Commands

```bash
# Compose mail job (interactive)
npm run cli -- mail compose \
  --documents ./proofs/doc1.json ./proofs/doc2.json \
  --recipient-name "John Doe" \
  --recipient-address "123 Main St, City, State 12345"

# Show cost and details before submitting
npm run cli -- mail confirm --job-id <jobId>

# Submit and process payment
npm run cli -- mail submit --job-id <jobId> --payment-method stripe

# Check status
npm run cli -- mail-status --job-id <jobId>

# Get receipt
npm run cli -- mail-receipt --job-id <jobId> --format pdf > receipt.pdf
```

### React Web UI Components

```typescript
// pages/MailComposer.tsx
export function MailComposer() {
  return (
    <div className="mail-composer">
      <h1>Compose Mail Job</h1>
      
      {/* Document selection */}
      <DocumentSelector onSelect={handleDocumentSelect} />
      
      {/* Address input with OCR option */}
      <AddressInput
        onExtract={handleOCRExtract}
        onManualEntry={handleAddressEntry}
      />
      
      {/* Address verification status */}
      {addressVerification && (
        <AddressVerificationResult result={addressVerification} />
      )}
      
      {/* Cost calculation */}
      <CostBreakdown cost={mailJob.estimatedCost} />
      
      {/* Next step button */}
      <button onClick={handleProceedToConfirmation}>
        Review & Confirm
      </button>
    </div>
  );
}

// pages/MailConfirmation.tsx
export function MailConfirmation({ jobId }: { jobId: string }) {
  return (
    <div className="mail-confirmation">
      <h1>Confirm Mail Job</h1>
      
      {/* Summary of all details */}
      <MailJobSummary job={mailJob} />
      
      {/* QR code preview */}
      <QRCodePreview qrCode={mailJob.qrCodes[0]} />
      
      {/* Payment method selection */}
      <PaymentMethodSelector
        onSelect={handlePaymentMethodSelect}
        methods={['stripe', 'btcpay']}
      />
      
      {/* Process payment */}
      <PaymentProcessor
        method={selectedPaymentMethod}
        amount={mailJob.actualCost}
        onSuccess={handlePaymentSuccess}
        onError={handlePaymentError}
      />
    </div>
  );
}

// pages/MailTracking.tsx
export function MailTracking() {
  return (
    <div className="mail-tracking">
      <h1>Mail Tracking</h1>
      
      {/* List of mail jobs */}
      <MailJobList
        jobs={mailJobs}
        onSelectJob={handleSelectJob}
      />
      
      {/* Tracking status for selected job */}
      {selectedJob && (
        <TrackingTimeline
          job={selectedJob}
          events={selectedJob.tracking.events}
        />
      )}
      
      {/* Receipt download */}
      {selectedJob && (
        <ReceiptDownload
          jobId={selectedJob.id}
          receiptUrl={selectedJob.receipts.initialReceiptArweaveUrl}
        />
      )}
    </div>
  );
}
```

---

## Part 7: Testing Strategy

### Unit Tests

```typescript
// tests/unit/postal.test.ts

describe('PostGrid Integration', () => {
  describe('Address Verification', () => {
    test('should extract address from OCR text', async () => {
      const ocrText = `
        Jane Doe
        123 Main Street
        San Francisco, CA 94103
      `;
      const address = parseAddressFromText(ocrText);
      expect(address.streetLine1).toBe('123 Main Street');
      expect(address.city).toBe('San Francisco');
      expect(address.stateProvince).toBe('CA');
    });

    test('should handle PostGrid verification', async () => {
      const mockResponse = {
        verificationStatus: 'verified',
        correctedAddress: { /* ... */ },
        addressErrors: [],
      };
      
      const result = await postgridClient.verifyAddress(testAddress);
      expect(result.verified).toBe(true);
    });
  });

  describe('QR Code Generation', () => {
    test('should generate valid QR code', async () => {
      const payload = {
        documentId: 'doc-123',
        arweaveUrl: 'https://arweave.net/abc123',
        witnetRequestId: 'req-456',
      };
      
      const qrCode = await generateQRCode(payload);
      expect(qrCode).toMatch(/^data:image\/png;base64,/);
    });

    test('should warn on large payload', async () => {
      // Create payload > 4000 chars
      const largePayload = { /* ... */ };
      
      await expect(generateQRCode(largePayload)).rejects.toThrow();
    });
  });

  describe('Receipt Generation', () => {
    test('should generate initial receipt PDF', async () => {
      const receipt = await generateInitialReceipt(mockMailJob);
      expect(receipt).toBeInstanceOf(Buffer);
      expect(receipt.length).toBeGreaterThan(1000);
    });

    test('should attest receipt to blockchain', async () => {
      const attestation = await attestReceipt(mockReceipt, arweaveClient, witnetClient);
      expect(attestation.arweaveUrl).toMatch(/https:\/\/arweave.net/);
      expect(attestation.witnetRequestId).toBeDefined();
    });
  });
});
```

### Integration Tests

```bash
# Test with real PostGrid API (testnet)
npm run test:integration -- postal --api-mode testnet

# Test payment processing (Stripe testnet)
npm run test:integration -- payment --stripe-mode test

# Test full workflow
npm run test:integration -- postal:full-workflow
```

---

## Part 8: Error Handling

### Custom Error Types

```typescript
export class PostalError extends Error {
  constructor(message: string, public readonly details?: any) {
    super(message);
    this.name = 'PostalError';
  }
}

export class AddressVerificationError extends PostalError {
  name = 'AddressVerificationError';
}

export class PostGridError extends PostalError {
  name = 'PostGridError';
}

export class PaymentError extends PostalError {
  name = 'PaymentError';
}

export class TrackingError extends PostalError {
  name = 'TrackingError';
}
```

### Error Recovery Strategies

| Error | Strategy |
|-------|----------|
| Address undeliverable | Suggest correction or allow manual override |
| Payment failed | Retry with exponential backoff; allow payment method change |
| PostGrid API down | Queue job; retry every 5 minutes for 24 hours |
| Tracking unavailable | Continue polling; notify after 30 days as failed |
| Arweave/Witnet error | Retry attestation; store locally until successful |

---

**End POSTAL.md**