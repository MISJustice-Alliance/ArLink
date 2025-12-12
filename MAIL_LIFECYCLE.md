# MAIL_LIFECYCLE.md - Phase 4: Complete Mail Job Lifecycle & Delivery Tracking

**End-to-End State Machine, Receipt Generation, and Delivery Automation for ArweaveStamp v3**

---

## Overview

This document describes the complete lifecycle of a mail job from creation through delivery, including receipt generation, blockchain attestation, and delivery confirmation.

---

## Complete State Machine

### Job States

```
┌───────────────────────────────────────────────────────┐
│                   INITIAL STATE                       │
├───────────────────────────────────────────────────────┤
│ created                                               │
│ ├─ Job created in database                           │
│ ├─ Document references set (Arweave + Witnet)        │
│ ├─ Recipient/sender data entered                     │
│ ├─ Cost estimate calculated                          │
│ └─ Ready for: address verification                   │
└────────┬────────────────────────────────────────────┘
         │ [OCR extract address]
         │ [PostGrid verify address]
         ▼
┌───────────────────────────────────────────────────────┐
│                CONFIRMATION STATE                     │
├───────────────────────────────────────────────────────┤
│ awaiting_confirmation                                │
│ ├─ Address verification complete                    │
│ ├─ QR code generated (Arweave + Witnet refs)       │
│ ├─ Cost breakdown finalized                         │
│ ├─ Confirmation summary displayed to user           │
│ ├─ Payment form ready                               │
│ └─ User must confirm or cancel                      │
│    └─ Expires after 1 hour                          │
└────┬─────────────────────────────────────────────────┘
     │ [user confirms]
     │ [payment method: stripe OR crypto]
     ▼
┌───────────────────────────────────────────────────────┐
│                 PAYMENT STATE                         │
├───────────────────────────────────────────────────────┤
│ payment_processing                                   │
│ ├─ Stripe: charge created (immediate)               │
│ │  └─ Webhook: charge.succeeded → settled           │
│ ├─ BTCPay: invoice created (60 min expiry)          │
│ │  └─ Webhook: invoice.confirmed → settled          │
│ └─ Waiting for payment confirmation                 │
└────┬─────────────────────────────────────────────────┘
     │ [payment settled]
     │ [OR payment failed → error state]
     ▼
┌───────────────────────────────────────────────────────┐
│             POSTGRID SUBMISSION STATE                 │
├───────────────────────────────────────────────────────┤
│ submitted_to_postgrid                                │
│ ├─ Mailing receipt PDF generated                    │
│ ├─ Mailing receipt uploaded to Arweave             │
│ ├─ Mailing receipt attested on Witnet              │
│ ├─ QR codes embedded in print-ready PDF            │
│ ├─ Job submitted to PostGrid API                   │
│ ├─ PostGrid job ID received                        │
│ ├─ Carrier + tracking numbers received             │
│ └─ Awaiting PostGrid confirmation                  │
└────┬─────────────────────────────────────────────────┘
     │ [postgrid_acknowledged]
     ▼
┌───────────────────────────────────────────────────────┐
│              IN-PRODUCTION STATE                      │
├───────────────────────────────────────────────────────┤
│ in_production                                        │
│ ├─ PostGrid confirmed job receipt                  │
│ ├─ Document(s) in print queue                      │
│ ├─ Waiting for shipment to carrier                 │
│ └─ Background service polling carrier tracking     │
└────┬─────────────────────────────────────────────────┘
     │ [carrier_received]
     ▼
┌───────────────────────────────────────────────────────┐
│              IN-TRANSIT STATE                         │
├───────────────────────────────────────────────────────┤
│ in_transit                                           │
│ ├─ Carrier has package                             │
│ ├─ Tracking updates available                      │
│ ├─ User can view real-time tracking               │
│ ├─ Background service polling every 12 hours       │
│ └─ Waiting for delivery confirmation               │
└────┬─────────────────────────────────────────────────┘
     │ [carrier_delivered]
     ▼
┌───────────────────────────────────────────────────────┐
│            DELIVERED STATE (FINAL)                    │
├───────────────────────────────────────────────────────┤
│ delivered                                            │
│ ├─ Package delivered to recipient                  │
│ ├─ Delivery receipt generated                      │
│ ├─ Delivery receipt uploaded to Arweave           │
│ ├─ Delivery receipt attested on Witnet            │
│ ├─ Full cryptographic chain complete              │
│ ├─ Job closed (no further polling)                │
│ └─ Complete audit trail available                 │
└───────────────────────────────────────────────────────┘

[ERROR STATES]
├─ address_verification_failed
│  └─ No valid address found; user must correct
├─ payment_failed
│  └─ Card declined or crypto payment expired; user can retry
├─ postgrid_submission_failed
│  └─ Document format error; automatic retry with backoff
├─ carrier_delivery_failed
│  └─ Package undeliverable; tracking shows reason
├─ cancelled_by_user
│  └─ User requested cancellation before PostGrid submission
└─ expired
   └─ Confirmation or payment window expired
```

---

## Mailing Receipt Generation

### Receipt Data Structure

```typescript
interface MailingReceipt {
  // Receipt Metadata
  receiptId: string;                           // UUID
  createdAt: Date;
  
  // Job Reference
  jobId: string;
  jobTitle: string;
  jobDescription?: string;
  
  // Organization
  organizationId: string;
  organizationName: string;
  organizationLogo?: string;
  
  // Sender Information
  sender: {
    name: string;
    address: string;
    phone?: string;
  };
  
  // Recipient Information
  recipient: {
    name: string;
    address: string;                          // Verified by PostGrid
  };
  
  // Document List (with Arweave + Witnet references)
  documents: Array<{
    documentArweaveId: string;
    documentArweaveUrl: string;
    documentWitnetId: string;
    documentWitnetUrl: string;
    
    title: string;
    fileHash: string;
    fileSize: number;
    uploadedAt: Date;
    
    // QR code for this document
    qrCode: {
      content: Buffer;                        // PNG image
      data: {
        arweaveUrl: string;
        witnetUrl: string;
        jobReference: string;
      };
    };
  }>;
  
  // Cost Breakdown
  costBreakdown: {
    printCost: number;
    postageCost: number;
    verificationFee: number;
    claudeMetadataFee: number;
    witnetAttestationFee: number;
    receiptStorageFee: number;
    paymentProcessorFee: number;
    totalCost: number;
    currency: string;
  };
  
  // Payment Details (masked for security)
  payment: {
    method: 'stripe' | 'crypto';
    status: string;
    
    // Stripe: masked card info
    stripeCardBrand?: string;                  // "Visa", "Mastercard", etc.
    stripeCardLastFour?: string;               // "1234"
    stripeTransactionId?: string;
    
    // Crypto: transaction hash
    btcpayInvoiceId?: string;
    btcpayTxHash?: string;
    btcpayCryptoCode?: string;                 // "BTC", "ETH", "USDC", etc.
    btcpayConfirmations?: number;
  };
  
  // PostGrid Integration
  postgrid: {
    jobId: string;
    status: string;                            // "created", "submitted", etc.
    carriers: Array<{
      name: string;                            // "USPS", "UPS", "FedEx"
      trackingNumber: string;
      trackingUrl: string;
      estimatedDeliveryDate: Date;
    }>;
  };
  
  // Attestations
  attestations: {
    mailingReceiptArweaveId: string;
    mailingReceiptArweaveUrl: string;
    mailingReceiptWitnetId: string;
    mailingReceiptWitnetUrl: string;
  };
  
  // AI Metadata (Claude)
  aiMetadata?: {
    jobSummary: string;
    riskFlags: string[];
    suggestedRetention: string;
    entities: Array<{ type: string; value: string }>;
  };
  
  // Verification Portal
  verificationPortalUrl: string;
}
```

### Receipt PDF Generation

```typescript
import PDFDocument from 'pdfkit';
import QRCode from 'qrcode';

async function generateMailingReceiptPDF(receipt: MailingReceipt): Promise<Buffer> {
  const doc = new PDFDocument({ margin: 50 });
  const buffers: Buffer[] = [];
  
  // Collect output
  doc.on('data', (chunk: Buffer) => buffers.push(chunk));
  
  // ===== HEADER =====
  
  // Organization logo (if available)
  if (receipt.organizationLogo) {
    doc.image(receipt.organizationLogo, 50, 50, { width: 100 });
  }
  
  // Header text
  doc.fontSize(18).font('Helvetica-Bold').text('Mailing Receipt', 200, 60);
  doc.fontSize(10).font('Helvetica').text(`Receipt ID: ${receipt.receiptId}`, 200, 85);
  doc.fontSize(10).text(`Created: ${receipt.createdAt.toISOString().split('T')[0]}`, 200, 100);
  
  doc.moveTo(50, 130).lineTo(550, 130).stroke();
  
  // ===== SENDER / RECIPIENT =====
  
  doc.fontSize(12).font('Helvetica-Bold').text('Sender', 50, 150);
  doc.fontSize(10).font('Helvetica');
  doc.text(receipt.sender.name, 50, 170);
  doc.text(receipt.sender.address, 50, 185);
  
  doc.fontSize(12).font('Helvetica-Bold').text('Recipient', 300, 150);
  doc.fontSize(10).font('Helvetica');
  doc.text(receipt.recipient.name, 300, 170);
  doc.text(receipt.recipient.address, 300, 185);
  
  doc.moveTo(50, 220).lineTo(550, 220).stroke();
  
  // ===== DOCUMENTS TABLE =====
  
  let y = 240;
  
  doc.fontSize(12).font('Helvetica-Bold').text('Documents', 50, y);
  y += 25;
  
  // Table header
  doc.fontSize(9).font('Helvetica-Bold');
  doc.text('Document', 50, y, { width: 150 });
  doc.text('Arweave ID', 210, y, { width: 100 });
  doc.text('Witnet ID', 320, y, { width: 100 });
  doc.text('QR Code', 430, y, { width: 100 });
  y += 20;
  
  // Table rows
  doc.font('Helvetica').fontSize(8);
  for (const doc_ref of receipt.documents) {
    // Document title
    doc.text(doc_ref.title.substring(0, 20), 50, y, { width: 150 });
    
    // Arweave link
    doc.fillColor('blue').text(doc_ref.documentArweaveId.substring(0, 20), 210, y, {
      width: 100,
      link: doc_ref.documentArweaveUrl,
      underline: true,
    });
    doc.fillColor('black');
    
    // Witnet link
    doc.fillColor('blue').text(doc_ref.documentWitnetId.substring(0, 20), 320, y, {
      width: 100,
      link: doc_ref.documentWitnetUrl,
      underline: true,
    });
    doc.fillColor('black');
    
    // QR code (embedded image)
    doc.image(doc_ref.qrCode.content, 430, y, { width: 60 });
    
    y += 70;
  }
  
  doc.moveTo(50, y).lineTo(550, y).stroke();
  y += 15;
  
  // ===== COST BREAKDOWN =====
  
  doc.fontSize(12).font('Helvetica-Bold').text('Cost Breakdown', 50, y);
  y += 20;
  
  doc.fontSize(9).font('Helvetica');
  const breakdown = receipt.costBreakdown;
  
  const items = [
    ['Print', breakdown.printCost.toFixed(2)],
    ['Postage', breakdown.postageCost.toFixed(2)],
    ['Address Verification', breakdown.verificationFee.toFixed(2)],
    ['AI Metadata (Claude)', breakdown.claudeMetadataFee.toFixed(2)],
    ['Witnet Attestation', breakdown.witnetAttestationFee.toFixed(2)],
    ['Receipt Storage (Arweave)', breakdown.receiptStorageFee.toFixed(2)],
    ['Payment Processing', breakdown.paymentProcessorFee.toFixed(2)],
  ];
  
  for (const [item, amount] of items) {
    doc.text(item, 50, y);
    doc.text(`$${amount}`, 450, y, { align: 'right' });
    y += 15;
  }
  
  doc.moveTo(50, y).lineTo(550, y).stroke();
  y += 10;
  
  doc.fontSize(11).font('Helvetica-Bold');
  doc.text('Total Cost', 50, y);
  doc.text(`$${breakdown.totalCost.toFixed(2)}`, 450, y, { align: 'right' });
  y += 20;
  
  doc.moveTo(50, y).lineTo(550, y).stroke();
  y += 15;
  
  // ===== PAYMENT DETAILS =====
  
  doc.fontSize(10).font('Helvetica-Bold').text('Payment Details', 50, y);
  y += 15;
  
  doc.fontSize(9).font('Helvetica');
  if (receipt.payment.method === 'stripe') {
    doc.text(`Payment Method: Credit Card (${receipt.payment.stripeCardBrand} ****${receipt.payment.stripeCardLastFour})`);
    doc.text(`Transaction ID: ${receipt.payment.stripeTransactionId}`);
  } else if (receipt.payment.method === 'crypto') {
    doc.text(`Payment Method: Cryptocurrency (${receipt.payment.btcpayInvoiceId})`);
    doc.text(`Crypto Code: ${receipt.payment.btcpayInvoiceId}`);
    doc.text(`Confirmations: ${receipt.payment.btcpayConfirmations}`);
    doc.text(`Transaction Hash: ${receipt.payment.btcpayTxHash}`, {
      link: `https://blockchair.com/${receipt.payment.btcpayTxHash}`,
      underline: true,
    });
  }
  y += 60;
  
  // ===== CARRIER INFORMATION =====
  
  doc.fontSize(10).font('Helvetica-Bold').text('Carrier Information', 50, y);
  y += 15;
  
  doc.fontSize(9).font('Helvetica');
  for (const carrier of receipt.postgrid.carriers) {
    doc.text(`Carrier: ${carrier.name}`);
    doc.text(`Tracking Number: ${carrier.trackingNumber}`);
    doc.fillColor('blue').text(`View Tracking`, 50, y, {
      link: carrier.trackingUrl,
      underline: true,
    });
    doc.fillColor('black');
    y += 40;
  }
  
  // ===== BLOCKCHAIN ATTESTATIONS =====
  
  doc.addPage();
  doc.fontSize(12).font('Helvetica-Bold').text('Blockchain Attestations', 50, 50);
  y = 80;
  
  doc.fontSize(9).font('Helvetica');
  doc.text('This receipt and all associated documents have been permanently recorded on Arweave');
  doc.text('and attested on Witnet, creating an immutable cryptographic chain of custody.');
  y += 30;
  
  doc.fontSize(10).font('Helvetica-Bold').text('Mailing Receipt Attestation', 50, y);
  y += 15;
  
  doc.fontSize(9).font('Helvetica');
  doc.fillColor('blue').text(`Arweave ID: ${receipt.attestations.mailingReceiptArweaveId}`, {
    link: receipt.attestations.mailingReceiptArweaveUrl,
    underline: true,
  });
  doc.fillColor('black');
  y += 15;
  
  doc.fillColor('blue').text(`Witnet ID: ${receipt.attestations.mailingReceiptWitnetId}`, {
    link: receipt.attestations.mailingReceiptWitnetUrl,
    underline: true,
  });
  doc.fillColor('black');
  y += 30;
  
  // Verification Portal QR
  const verificationQR = await QRCode.toDataURL(receipt.verificationPortalUrl);
  doc.text('Verify this receipt at:', 50, y);
  doc.image(verificationQR, 50, y + 15, { width: 150 });
  
  doc.end();
  
  return Buffer.concat(buffers);
}
```

---

## Delivery Receipt Generation

### Delivery Receipt Data Structure

```typescript
interface DeliveryReceipt extends MailingReceipt {
  // Additional delivery-specific fields
  deliveryStatus: 'delivered' | 'undeliverable' | 'return_to_sender';
  deliveryDate: Date;
  deliveryTime?: string;
  
  carrierDeliveryNotes?: string;
  signedBy?: string;                         // Name of recipient (if available)
  signatureRequired: boolean;
  
  // Updated tracking info
  carriers: Array<{
    ...CarrierInfo,
    finalStatus: string;
    deliveryProof?: {
      location: string;
      timestamp: Date;
      notes?: string;
    };
  }>;
  
  // Delivery attestations (separate from mailing receipt)
  deliveryAttestations: {
    deliveryReceiptArweaveId: string;
    deliveryReceiptArweaveUrl: string;
    deliveryReceiptWitnetId: string;
    deliveryReceiptWitnetUrl: string;
  };
}
```

---

## Background Tasks: Carrier Tracking Polling

### Polling Service

```typescript
class CarrierTrackingPoller {
  private pollInterval: number = 12 * 60 * 60 * 1000;  // 12 hours
  private maxPollAttempts: number = 30;                // ~15 days max
  
  async startPolling() {
    setInterval(async () => {
      await this.pollActiveJobs();
    }, this.pollInterval);
  }
  
  private async pollActiveJobs() {
    // Get all jobs that are in-transit
    const activeJobs = await db.getMailJobsByStatus('in_transit');
    
    logger.info('Polling carrier tracking', {
      activeJobCount: activeJobs.length,
    });
    
    for (const job of activeJobs) {
      await this.pollJobTracking(job);
    }
  }
  
  private async pollJobTracking(job: MailJob) {
    try {
      // Poll each carrier
      for (let i = 0; i < job.carriers.length; i++) {
        const carrier = job.carriers[i];
        
        const tracking = await this.getCarrierTracking(
          carrier.trackingNumber,
          carrier.name
        );
        
        // Update job with latest tracking
        job.carriers[i] = {
          ...carrier,
          currentStatus: tracking.status,
          lastUpdatedAt: new Date(),
          scanEvents: tracking.scanEvents,
        };
        
        logger.info('Carrier tracking updated', {
          jobId: job.id,
          carrier: carrier.name,
          status: tracking.status,
          scanCount: tracking.scanEvents.length,
        });
        
        // If delivered, generate delivery receipt
        if (tracking.status === 'delivered') {
          await this.handleDeliveryConfirmed(job, i, tracking);
        }
        
        // If undeliverable, mark as failed
        if (tracking.status === 'undeliverable' || tracking.status === 'return_to_sender') {
          await this.handleDeliveryFailed(job, i, tracking);
        }
      }
      
      // Save updated job
      await db.saveMailJob(job);
      
    } catch (error) {
      logger.error('Carrier tracking poll failed', {
        jobId: job.id,
        error: error.message,
      });
      
      // Graceful degradation: don't fail the job, retry next cycle
    }
  }
  
  private async getCarrierTracking(
    trackingNumber: string,
    carrierName: string
  ): Promise<CarrierTrackingResult> {
    // Use PostGrid API to get tracking (they aggregate all carriers)
    return await postgridClient.getTracking(trackingNumber, carrierName);
  }
  
  private async handleDeliveryConfirmed(
    job: MailJob,
    carrierIndex: number,
    tracking: CarrierTrackingResult
  ) {
    logger.info('Delivery confirmed', {
      jobId: job.id,
      carrier: job.carriers[carrierIndex].name,
      deliveredAt: tracking.deliveryTimestamp,
    });
    
    // Update job status
    job.deliveryConfirmedAt = new Date();
    job.postgridStatus = 'delivered';
    
    // Generate delivery receipt
    const deliveryReceipt = await this.generateDeliveryReceipt(job, tracking);
    
    // Upload receipt to Arweave
    const arweaveId = await uploadToArweave(deliveryReceipt);
    job.deliveryReceiptArweaveId = arweaveId;
    
    // Attest on Witnet
    const witnetId = await attestOnWitnet({
      data: deliveryReceipt,
      arweaveId,
      jobId: job.id,
      references: [
        job.sourceDocumentArweaveId,
        job.sourceDocumentWitnetId,
        job.mailingReceiptArweaveId,
        job.mailingReceiptWitnetId,
      ],
    });
    job.deliveryReceiptWitnetId = witnetId;
    
    // Send notification to user
    await sendEmailNotification(
      job.submittedBy,
      'delivery_confirmed',
      { jobId: job.id, deliveryDate: job.deliveryConfirmedAt }
    );
  }
  
  private async handleDeliveryFailed(
    job: MailJob,
    carrierIndex: number,
    tracking: CarrierTrackingResult
  ) {
    logger.warn('Delivery failed', {
      jobId: job.id,
      carrier: job.carriers[carrierIndex].name,
      reason: tracking.failureReason,
    });
    
    job.postgridStatus = 'failed';
    job.carriers[carrierIndex].currentStatus = tracking.status;
    
    // Send notification to user
    await sendEmailNotification(
      job.submittedBy,
      'delivery_failed',
      {
        jobId: job.id,
        reason: tracking.failureReason,
        carrierNotes: tracking.notes,
      }
    );
  }
  
  private async generateDeliveryReceipt(
    job: MailJob,
    tracking: CarrierTrackingResult
  ): Promise<DeliveryReceipt> {
    return {
      ...job,  // Include all mailing receipt data
      
      // Delivery-specific data
      deliveryStatus: 'delivered',
      deliveryDate: tracking.deliveryTimestamp,
      deliveryTime: tracking.deliveryTime,
      carrierDeliveryNotes: tracking.notes,
      
      deliveryAttestations: {
        // Will be populated after Arweave + Witnet upload
        deliveryReceiptArweaveId: '',
        deliveryReceiptArweaveUrl: '',
        deliveryReceiptWitnetId: '',
        deliveryReceiptWitnetUrl: '',
      },
    };
  }
}
```

---

## Email Notifications

### Notification Types

```typescript
enum NotificationType {
  JOB_CREATED = 'job_created',
  ADDRESS_VERIFIED = 'address_verified',
  PAYMENT_REQUIRED = 'payment_required',
  PAYMENT_CONFIRMED = 'payment_confirmed',
  SUBMITTED_TO_POSTGRID = 'submitted_to_postgrid',
  CARRIER_PICKED_UP = 'carrier_picked_up',
  IN_TRANSIT = 'in_transit',
  DELIVERY_CONFIRMED = 'delivery_confirmed',
  DELIVERY_FAILED = 'delivery_failed',
  JOB_CANCELLED = 'job_cancelled',
}

async function sendEmailNotification(
  recipientEmail: string,
  notificationType: NotificationType,
  data: MailJob
) {
  const templates: Record<NotificationType, string> = {
    job_created: 'Email body: Your mail job has been created. Please confirm details.',
    address_verified: 'Email body: Address verified successfully.',
    payment_required: 'Email body: Please complete payment to proceed.',
    payment_confirmed: 'Email body: Payment confirmed! Preparing to mail.',
    submitted_to_postgrid: 'Email body: Job submitted to PostGrid. Tracking info: ...',
    carrier_picked_up: 'Email body: Package picked up by carrier.',
    in_transit: 'Email body: Package in transit. Track here: ...',
    delivery_confirmed: 'Email body: Package delivered successfully!',
    delivery_failed: 'Email body: Delivery failed. Reason: ...',
    job_cancelled: 'Email body: Mail job cancelled.',
  };
  
  await emailService.send({
    to: recipientEmail,
    subject: getEmailSubject(notificationType),
    html: templates[notificationType],
    data,
  });
}
```

---

## Error Handling & Recovery

### Transient Errors (Retry)

- Carrier API timeout
- Network connectivity issue
- PostGrid API rate limit

### Terminal Errors (Escalate)

- Address verification failed
- Payment method declined
- PostGrid document format error
- Carrier undeliverable

### Recovery Workflow

```typescript
async function handleError(job: MailJob, error: Error) {
  if (isTransient(error)) {
    // Retry with backoff
    const retryCount = (job.retryCount || 0) + 1;
    if (retryCount <= MAX_RETRIES) {
      job.retryCount = retryCount;
      job.nextRetryAt = new Date(Date.now() + exponentialBackoff(retryCount));
      await db.saveMailJob(job);
    } else {
      // Max retries exceeded
      job.status = 'failed';
      await notifyUser(job, 'Max retries exceeded');
    }
  } else {
    // Terminal error
    job.status = 'failed';
    job.errorMessage = error.message;
    await notifyUser(job, `Error: ${error.message}`);
  }
}
```

---

## Testing Strategy

### Unit Tests

```bash
npm run test -- mail/lifecycle/
```

- State transitions
- Receipt generation
- Email notifications
- Cost calculations

### Integration Tests

```bash
npm run test:integration -- --suite mail-lifecycle
```

- Complete job flow: creation → confirmation → payment → submission
- Carrier tracking polling simulation
- Delivery receipt generation

### End-to-End Tests

```bash
npm run test:e2e -- --suite mail-lifecycle
```

- Real PostGrid sandbox: create job → get confirmation → submit
- Mock carrier: receive tracking data → generate delivery receipt
- Full blockchain attestation chain

---

**Last Updated**: December 2025  
**Status**: Phase 4 Specification  
**Next**: Implement receipt generators + background polling service
