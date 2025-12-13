---
name: payment-processing-skill
version: 1.0.0
triggers:
  - "process payment"
  - "stripe payment"
  - "crypto payment"
dependencies: []
tools:
  - Read
  - Bash(npm:*)
---

# Skill: Payment Processing (Phase 4)

## Objective
Handle payment processing for mail jobs via Stripe (fiat) or BTCPay (crypto).

## Prerequisites
- STRIPE_SECRET_KEY configured (for fiat)
- BTCPAY_URL and BTCPAY_API_KEY configured (for crypto)
- Payment amount calculated

## Workflow

### Stripe Payment Flow
1. Create Stripe payment intent
2. Process card payment
3. Wait for confirmation
4. Handle webhooks for async updates
5. Return payment confirmation

### BTCPay Crypto Payment Flow
1. Create BTCPay invoice
2. Display payment QR code and address
3. Wait for blockchain confirmations
4. Poll invoice status
5. Return payment confirmation

### Phase 1: Payment Method Selection
1. Display available methods (Stripe/BTCPay)
2. User selects payment method
3. Route to appropriate flow

### Phase 2: Payment Execution
1. Execute payment based on method
2. Monitor payment status
3. Handle timeouts gracefully

### Phase 3: Confirmation & Receipt
1. Verify payment settled
2. Generate payment receipt
3. Update mail job status
4. Return confirmation to caller

## Error Handling
- **Payment declined**: Alert user, allow retry
- **Network timeout**: Implement retry logic
- **Webhook failure**: Poll payment status manually
- **Crypto underpayment**: Request additional payment

## Security
- Never log full card numbers
- Use Stripe test keys in development
- Validate webhook signatures
- Secure BTCPay API keys

## Success Criteria
- Payment processed successfully
- Receipt generated
- Status updated
- User notified

## Version History
- 1.0.0: Initial release
