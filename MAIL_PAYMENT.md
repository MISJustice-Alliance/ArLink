# MAIL_PAYMENT.md - Phase 4: Payment Settlement & Reconciliation

**Stripe + BTCPay Payment Integration for ArweaveStamp Mail Jobs**

---

## Overview

ArweaveStamp v3 Phase 4 supports two payment methods for print & mail jobs:

1. **Stripe** (Card payments): Immediate settlement
2. **BTCPay Server** (Crypto payments): 1-3 block confirmation wait

Both methods create auditable transactions recorded on Arweave and attested on Witnet.

---

## Architecture

### Payment Gateway Abstraction

```typescript
interface PaymentGateway {
  createCharge(request: CreateChargeRequest): Promise<ChargeResponse>;
  getChargeStatus(chargeId: string): Promise<ChargeStatus>;
  refund(chargeId: string, amount?: number): Promise<RefundResponse>;
}

// Implementations:
// - StripePaymentGateway (extends PaymentGateway)
// - BTCPayPaymentGateway (extends PaymentGateway)
```

### Payment State Machine

```
┌──────────────┐
│   pending    │ (job created, payment not started)
└──────┬───────┘
       │ [user_initiates_payment]
       ▼
┌──────────────────┐
│   processing     │ (payment being processed)
└──────┬───────────┘
       │ [payment_confirmed]
       ▼
┌──────────────────┐
│   settled        │ (payment finalized, ready for PostGrid)
└──────────────────┘

[Error states]
├─ failed       (card declined, tx reverted, etc.)
├─ expired      (BTCPay invoice expired)
└─ refunded     (user requested refund)
```

---

## Stripe Integration

### Configuration

```typescript
// .env
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_CURRENCY=usd
```

### Implementation

```typescript
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, {
  apiVersion: '2025-01-27',
});

interface StripeCreateChargeRequest {
  amount: number;                          // Amount in USD (e.g., 1.45 → 145 cents)
  description: string;                     // e.g., "ArweaveStamp mail job abc123"
  metadata: {
    jobId: string;
    organizationId: string;
    documentArweaveId: string;
  };
  stripeToken: string;                     // Token from Stripe Elements (client-side)
}

class StripePaymentGateway {
  async createCharge(request: StripeCreateChargeRequest): Promise<ChargeResponse> {
    try {
      const charge = await stripe.charges.create({
        amount: Math.round(request.amount * 100),  // Convert to cents
        currency: 'usd',
        source: request.stripeToken,
        description: request.description,
        metadata: request.metadata,
        // Receipt email sent automatically
      });

      return {
        chargeId: charge.id,
        status: charge.status,
        amount: charge.amount / 100,
        currency: charge.currency,
        receipt_url: charge.receipt_url,
        card: {
          brand: charge.payment_method_details?.card?.brand,
          lastFour: charge.payment_method_details?.card?.last4,
        },
      };
    } catch (error) {
      if (error.type === 'StripeCardError') {
        throw new PaymentError('CARD_DECLINED', error.message);
      } else if (error.type === 'StripeInvalidRequestError') {
        throw new PaymentError('INVALID_TOKEN', error.message);
      } else {
        throw new PaymentError('STRIPE_ERROR', error.message);
      }
    }
  }

  async getChargeStatus(chargeId: string): Promise<ChargeStatus> {
    const charge = await stripe.charges.retrieve(chargeId);
    return {
      chargeId: charge.id,
      status: charge.status,
      amount: charge.amount / 100,
      paid: charge.paid,
      refunded: charge.refunded,
      failure_reason: charge.failure_reason,
    };
  }

  async refund(chargeId: string, amount?: number): Promise<RefundResponse> {
    const refund = await stripe.refunds.create({
      charge: chargeId,
      amount: amount ? Math.round(amount * 100) : undefined,
    });

    return {
      refundId: refund.id,
      chargeId: refund.charge,
      amount: refund.amount / 100,
      status: refund.status,
      reason: refund.reason,
    };
  }
}
```

### Webhook Handling

```typescript
// POST /webhooks/stripe
// Validate: Stripe signature in request header

async function handleStripeWebhook(req, res) {
  const signature = req.headers['stripe-signature'];
  
  let event;
  try {
    event = stripe.webhooks.constructEvent(
      req.body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET
    );
  } catch (err) {
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  switch (event.type) {
    case 'charge.succeeded':
      await handleChargeSucceeded(event.data.object);
      break;
      
    case 'charge.failed':
      await handleChargeFailed(event.data.object);
      break;
      
    case 'charge.refunded':
      await handleChargeRefunded(event.data.object);
      break;
      
    case 'charge.dispute.created':
      await handleChargeDispute(event.data.object);
      break;
  }

  res.json({ received: true });
}

async function handleChargeSucceeded(charge: Stripe.Charge) {
  const job = await db.getMailJobByMetadata(charge.metadata.jobId);
  
  job.paymentStatus = 'settled';
  job.paymentDetails.stripeTransactionId = charge.id;
  job.paymentDetails.stripeMaskedCard = `${charge.payment_method_details?.card?.brand} ****${charge.payment_method_details?.card?.last4}`;
  
  // Now safe to submit to PostGrid
  await submitToPostGrid(job);
  
  await db.saveMailJob(job);
}

async function handleChargeFailed(charge: Stripe.Charge) {
  const job = await db.getMailJobByMetadata(charge.metadata.jobId);
  
  job.paymentStatus = 'failed';
  job.paymentDetails.failureReason = charge.failure_reason;
  
  // Notify user
  await sendEmail(job.submittedBy, 'Payment Failed', {
    jobId: job.id,
    reason: charge.failure_reason,
    retryUrl: `https://app.arweave-stamp.io/mail/retry/${job.id}`,
  });
  
  await db.saveMailJob(job);
}
```

---

## BTCPay Server Integration

### Configuration

```typescript
// .env
BTCPAY_URL=https://your-btcpay-instance.com
BTCPAY_API_KEY=btcpay-api-key-here
BTCPAY_STORE_ID=store-id
BTCPAY_ENABLED_COINS=BTC,ETH,USDC,USDT  // Supported cryptocurrencies
BTCPAY_NOTIFICATION_URL=https://arweave-stamp.io/webhooks/btcpay
```

### Implementation

```typescript
import btcpayserver from 'btcpay-js-sdk';

const btcpay = new btcpayserver.Client({
  host: process.env.BTCPAY_URL,
  apiKey: process.env.BTCPAY_API_KEY,
});

interface BTCPayCreateInvoiceRequest {
  amount: number;                          // Amount in USD
  currency: string;                        // 'USD' (BTCPay converts to crypto)
  description: string;
  orderId: string;                         // jobId
  metadata: {
    jobId: string;
    organizationId: string;
    documentArweaveId: string;
  };
  notificationUrl: string;
  expirationMinutes: number;               // Default: 60
}

class BTCPayPaymentGateway {
  async createInvoice(request: BTCPayCreateInvoiceRequest): Promise<InvoiceResponse> {
    try {
      const invoice = await btcpay.createInvoice({
        storeId: process.env.BTCPAY_STORE_ID,
        amount: request.amount,
        currency: request.currency,
        orderId: request.orderId,
        itemDescription: request.description,
        notificationUrl: request.notificationUrl,
        expirationMinutes: request.expirationMinutes,
        metadata: request.metadata,
      });

      return {
        invoiceId: invoice.id,
        orderId: invoice.orderId,
        amount: invoice.amount,
        currency: invoice.currency,
        status: invoice.status,
        expirationTime: invoice.expirationTime,
        
        // Payment methods available
        paymentMethods: invoice.paymentMethods.map(pm => ({
          cryptoCode: pm.cryptoCode,
          displayName: pm.displayName,
          address: pm.address,
          qrCode: pm.qrCode,
          amount: pm.amount,
          
          // For BTC: show address + amount
          // For Ethereum-based: show checksum address
          paymentUrl: pm.paymentUrl,
        })),
        
        // User-facing URLs
        invoiceUrl: invoice.url,
        checkoutLink: invoice.checkoutLink,
      };
    } catch (error) {
      throw new PaymentError('BTCPAY_ERROR', error.message);
    }
  }

  async getInvoiceStatus(invoiceId: string): Promise<InvoiceStatus> {
    const invoice = await btcpay.getInvoice({
      storeId: process.env.BTCPAY_STORE_ID,
      invoiceId,
    });

    return {
      invoiceId: invoice.id,
      status: invoice.status,                  // 'new', 'received', 'confirmed', 'complete', 'expired', 'invalid'
      amount: invoice.amount,
      currency: invoice.currency,
      
      // Payment details
      payments: invoice.payments.map(p => ({
        cryptoCode: p.cryptoCode,
        amount: p.amount,
        confirmations: p.confirmations,
        txHash: p.txHash,
        blockHeight: p.blockHeight,
      })),
      
      expirationTime: invoice.expirationTime,
      creationTime: invoice.creationTime,
    };
  }
}
```

### Invoice Lifecycle

```
User creates mail job
  ↓
Select "crypto" payment
  ↓
Create BTCPay invoice
  → Status: 'new'
  → Invoice expires in 60 minutes
  ↓
User sees payment methods (BTC, ETH, USDC, etc.)
  ↓
User sends crypto to provided address
  ↓
[Webhook: payment received]
  → Status: 'received'
  → 0 confirmations
  ↓
[Background: poll for confirmations]
  ↓
[After 1-3 confirmations (configurable)]
  → Status: 'confirmed' or 'complete'
  ↓
[Webhook: invoice confirmed]
  → Update job.paymentStatus = 'settled'
  → Submit to PostGrid
  ↓
[Job continues as normal]
```

### Webhook Handling

```typescript
// POST /webhooks/btcpay
// Validate: BTCPay signature in request header

async function handleBTCPayWebhook(req, res) {
  // Verify webhook signature (if required by your BTCPay instance)
  const payload = req.body;
  
  const job = await db.getMailJobByBTCPayInvoice(payload.invoiceId);
  
  switch (payload.event) {
    case 'invoice_received':
      logger.info('BTCPay payment received', {
        jobId: job.id,
        invoiceId: payload.invoiceId,
        cryptoCode: payload.cryptoCode,
        amount: payload.amount,
      });
      break;
      
    case 'invoice_confirmed':
      job.paymentStatus = 'settled';
      job.paymentDetails.btcpayInvoiceId = payload.invoiceId;
      job.paymentDetails.btcpayTxHash = payload.txHash;
      job.paymentDetails.btcpayConfirmations = payload.confirmations;
      
      await submitToPostGrid(job);
      await db.saveMailJob(job);
      
      logger.info('BTCPay payment confirmed', {
        jobId: job.id,
        txHash: payload.txHash,
        confirmations: payload.confirmations,
      });
      break;
      
    case 'invoice_complete':
      logger.info('BTCPay invoice complete', { jobId: job.id });
      break;
      
    case 'invoice_expired':
      job.paymentStatus = 'expired';
      await db.saveMailJob(job);
      
      await sendEmail(job.submittedBy, 'Payment Invoice Expired', {
        jobId: job.id,
        retryUrl: `https://app.arweave-stamp.io/mail/retry/${job.id}`,
      });
      break;
      
    case 'invoice_failed':
      job.paymentStatus = 'failed';
      await db.saveMailJob(job);
      break;
  }

  res.json({ received: true });
}
```

---

## Payment Flow Comparison

| Aspect | Stripe | BTCPay |
|--------|--------|--------|
| **Settlement** | Immediate (seconds) | 1-3 block confirmations (~10-30 min) |
| **Currencies** | USD (cards) | BTC, ETH, USDC, USDT, XMR, etc. |
| **Fees** | 2.9% + $0.30 | 0% (if self-hosted) or ~1% (hosted) |
| **Refund Policy** | Full refund available | Crypto refunds require user wallet |
| **User Experience** | Quick, familiar | More technical, requires wallet |
| **Privacy** | Requires card info | Pseudonymous (crypto address) |
| **Compliance** | PCI-DSS required | Varies by jurisdiction |

---

## Cost Breakdown & Pricing

### Calculation

```typescript
interface CostBreakdownCalculation {
  // Base costs (from PostGrid)
  printCost: 0.75;                    // Single-page B&W print
  postageCost: 0.55;                  // USPS postage
  
  // ArweaveStamp costs
  claudeMetadataFee: 0.10;            // Claude API call for metadata
  qrCodeGenerationFee: 0.01;          // QR code generation (negligible)
  receiptStorageFee: 0.05;            // Arweave storage for receipt (~200KB)
  receiptAttestationFee: 0.05;        // Witnet attestation for receipt
  
  // Optional costs
  premiumBranding: 0.25;              // Optional: custom logo injection
  expressDelivery: 1.50;              // Optional: 1-day delivery
  
  // Payment processor fee
  paymentProcessorFee: 0.03;          // 2.9% + $0.30 for Stripe
  
  // Total before payment
  subtotal: 1.51;
  
  // After payment processor fee (if Stripe)
  totalCost: 1.51 + (1.51 * 0.029) + 0.30 = 1.85;
}
```

### Dynamic Pricing

```typescript
interface DynamicPricingEngine {
  // Volume discounts
  calculateVolumeDiscount(monthlyJobCount: number): number;
  
  // Carrier pricing varies by region
  calculatePostageCost(recipient: Address, mailingClass: string): number;
  
  // Crypto fee based on network congestion (for BTCPay)
  calculateCryptoFee(cryptoCode: string): number;
}

// Example
const discount = volumeDiscount(1000);  // 15% discount for 1000+ jobs/month
const totalCost = baseCost * (1 - discount);
```

---

## Reconciliation & Accounting

### Ledger Entry

```typescript
interface LedgerEntry {
  entryId: string;
  jobId: string;
  organizationId: string;
  
  // Transaction
  transactionType: 'charge' | 'refund' | 'adjustment';
  transactionDate: Date;
  
  // Amounts (all in USD)
  originalAmount: number;               // Job cost before fees
  feeAmount: number;                    // Payment processor fee
  totalAmount: number;                  // Total charged
  
  // Payment method
  paymentMethod: 'stripe' | 'crypto';
  paymentId: string;                    // Stripe charge ID or BTCPay invoice ID
  
  // Status
  status: 'pending' | 'settled' | 'failed' | 'refunded';
  
  // Attestation
  arweaveTransactionId?: string;        // Ledger entry on Arweave
  witnetAttestationId?: string;         // Ledger entry attested on Witnet
}
```

### Monthly Reconciliation

```typescript
async function monthlyReconciliation(organizationId: string, month: Date) {
  // Get all jobs for month
  const jobs = await db.getMailJobs({
    organizationId,
    submittedAt: { $gte: monthStart, $lt: monthEnd },
  });

  // Calculate totals
  const summary = {
    totalJobs: jobs.length,
    totalRevenue: jobs.reduce((sum, job) => sum + job.costBreakdown.totalCost, 0),
    
    // By payment method
    stripeJobs: jobs.filter(j => j.paymentMethod === 'stripe').length,
    stripeRevenue: jobs
      .filter(j => j.paymentMethod === 'stripe')
      .reduce((sum, j) => sum + j.costBreakdown.totalCost, 0),
    
    cryptoJobs: jobs.filter(j => j.paymentMethod === 'crypto').length,
    cryptoRevenue: jobs
      .filter(j => j.paymentMethod === 'crypto')
      .reduce((sum, j) => sum + j.costBreakdown.totalCost, 0),
    
    // By status
    succeededJobs: jobs.filter(j => j.paymentStatus === 'settled').length,
    failedJobs: jobs.filter(j => j.paymentStatus === 'failed').length,
  };

  // Generate reconciliation report
  const report = new ReconciliationReport(summary);
  
  // Upload to Arweave
  const arweaveId = await uploadToArweave(report);
  
  // Attest on Witnet
  const witnetId = await attestOnWitnet({
    data: report,
    arweaveId,
    organizationId,
  });

  await db.saveReconciliation({
    organizationId,
    month,
    summary,
    arweaveId,
    witnetId,
  });
}
```

---

## Error Handling

### Payment Error Types

```typescript
enum PaymentErrorType {
  // Stripe errors
  CARD_DECLINED = 'card_declined',
  INVALID_TOKEN = 'invalid_token',
  EXPIRED_TOKEN = 'expired_token',
  
  // BTCPay errors
  INVOICE_EXPIRED = 'invoice_expired',
  INSUFFICIENT_PAYMENT = 'insufficient_payment',
  FAILED_TO_CONFIRM = 'failed_to_confirm',
  
  // Network/system errors
  PAYMENT_GATEWAY_DOWN = 'payment_gateway_down',
  WEBHOOK_TIMEOUT = 'webhook_timeout',
  
  // Business logic errors
  INSUFFICIENT_FUNDS = 'insufficient_funds',
  AMOUNT_MISMATCH = 'amount_mismatch',
  DUPLICATE_PAYMENT = 'duplicate_payment',
}

class PaymentError extends Error {
  constructor(
    public type: PaymentErrorType,
    public message: string,
    public retryable: boolean = false,
    public details?: unknown
  ) {
    super(message);
  }
}
```

### Retry Strategy

```typescript
async function processPaymentWithRetry(
  job: MailJob,
  paymentMethod: 'stripe' | 'crypto',
  maxRetries: number = 3
): Promise<PaymentResult> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      if (paymentMethod === 'stripe') {
        return await stripeGateway.createCharge(job);
      } else {
        return await btcpayGateway.createInvoice(job);
      }
    } catch (error) {
      if (!error.retryable || attempt === maxRetries) {
        throw error;
      }
      
      // Exponential backoff: 1s, 2s, 4s
      await delay(Math.pow(2, attempt - 1) * 1000);
    }
  }
}
```

---

## Security Considerations

### PCI Compliance

- ✅ Use Stripe tokens only (never raw card data)
- ✅ HTTPS for all payment endpoints
- ✅ No payment info stored in logs
- ✅ PCI-DSS Level 1 compliance required

### Fraud Prevention

```typescript
interface FraudDetection {
  // Check for duplicate payments
  isDuplicatePayment(jobId: string, paymentId: string): boolean;
  
  // Check for suspicious patterns
  isSuspiciousBehavior(organizationId: string): boolean;
  
  // Verify amount matches
  verifyAmount(charged: number, expected: number): boolean;
}
```

### Secret Management

- Store API keys in environment variables
- Use AWS Secrets Manager or similar for production
- Rotate keys regularly
- Never commit secrets to Git

---

## Testing

### Unit Tests

```bash
npm run test -- payment/
```

- Stripe charge creation + error handling
- BTCPay invoice creation + error handling
- Cost breakdown calculation
- Payment state transitions

### Integration Tests

```bash
npm run test:integration -- --suite payment
```

- Stripe test mode charges
- BTCPay testnet invoices
- Webhook signature validation
- Refund flows

### End-to-End Tests

```bash
npm run test:e2e -- --suite payment
```

- Stripe: job creation → payment → PostGrid submission
- BTCPay: invoice creation → payment confirmation → PostGrid submission
- Failure scenarios: declined cards, expired invoices, network timeouts

---

## Monitoring & Observability

### Metrics

- `payment_creation_rate` ($/hour, by method)
- `payment_settlement_rate` (%)
- `payment_failure_rate` (%)
- `payment_settlement_time` (seconds, by method)
- `webhook_processing_delay` (ms)

### Logging

```typescript
logger.info('payment_initiated', {
  jobId, organizationId, method, amount,
});

logger.info('payment_settled', {
  jobId, method, amount, transactionId, processingTime,
});

logger.error('payment_failed', {
  jobId, method, error: error.type, message: error.message,
});
```

---

## API Examples

### Create Stripe Charge

```bash
POST /mail/jobs/job_abc123/payment
Content-Type: application/json

{
  "paymentMethod": "stripe",
  "stripeToken": "tok_visa",
  "billingAddress": {
    "zipCode": "12345"
  }
}
```

**Response** (success):
```json
{
  "status": "settled",
  "transactionId": "ch_1OP...Ych2",
  "amount": 1.45,
  "card": {
    "brand": "Visa",
    "lastFour": "4242"
  },
  "nextSteps": "Submitting to PostGrid..."
}
```

### Create BTCPay Invoice

```bash
POST /mail/jobs/job_abc123/payment
Content-Type: application/json

{
  "paymentMethod": "crypto",
  "preferredCrypto": "BTC"
}
```

**Response** (success):
```json
{
  "status": "processing",
  "invoiceId": "btcpay_inv_xyz",
  "expiresAt": "2025-12-12T09:02:00Z",
  "paymentMethods": [
    {
      "cryptoCode": "BTC",
      "displayName": "Bitcoin",
      "address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
      "amount": 0.0000234,
      "qrCode": "data:image/png;base64,...",
      "paymentUrl": "https://your-btcpay-instance.com/i/btcpay_inv_xyz"
    },
    {
      "cryptoCode": "ETH",
      "displayName": "Ethereum",
      "address": "0x1234...abcd",
      "amount": 0.000543,
      "qrCode": "data:image/png;base64,...",
      "paymentUrl": "https://your-btcpay-instance.com/i/btcpay_inv_xyz"
    }
  ]
}
```

---

**Last Updated**: December 2025  
**Status**: Phase 4 Specification  
**Next**: Implement payment gateway wrappers + webhook handlers
