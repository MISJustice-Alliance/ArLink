---
description: "Create print & mail job workflow (Phase 4)"
allowed-tools: ["Read", "Bash(npm:*)"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Mail Job - Physical Mail Creation

## Purpose
Guide through creating a PostGrid print & mail job with payment and tracking.

## Workflow

1. **Prerequisites check**
   - Document already attested (has proof package)
   - PostGrid API key configured
   - Payment method configured (Stripe or BTCPay)

2. **Extract recipient address** (OCR)
   ```bash
   !npm run cli -- mail create $DOCUMENT \
     --extract-address \
     --confirm
   ```

3. **Verify address with PostGrid**
   Display verification results:
   - Original address
   - Verified address (PostGrid standardized)
   - Corrections made
   - Ask user to confirm

4. **Generate QR code**
   Encode:
   - Arweave URL
   - Witnet attestation ID
   - Job reference

5. **Calculate cost**
   Display:
   - Print cost: $X.XX
   - Postage: $X.XX
   - Processing fee: $X.XX
   - Total: $X.XX

6. **Process payment**
   For Stripe:
   ```bash
   !npm run cli -- mail create ... --payment-method stripe --token $TOKEN
   ```

   For BTCPay:
   ```bash
   !npm run cli -- mail create ... --payment-method crypto
   ```
   (Generates invoice, user pays, waits for confirmations)

7. **Submit to PostGrid**
   After payment settles:
   - Create print job
   - Get job ID and tracking number
   - Display confirmation

8. **Generate initial receipt**
   - Upload receipt to Arweave
   - Attest receipt on Witnet
   - Display Arweave URL

## Success Criteria
- Address verified
- Payment processed
- Job submitted to PostGrid
- Tracking number obtained
- Receipt generated

## Version History
- 1.0: Initial release
