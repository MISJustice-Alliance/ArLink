---
name: mail-integration-skill
version: 1.0.0
triggers:
  - "create mail job"
  - "send physical mail"
  - "postgrid integration"
dependencies:
  - payment-processing-skill
tools:
  - Read
  - Bash(npm:*)
---

# Skill: Physical Mail Integration (Phase 4)

## Objective
Automate PostGrid print & mail job creation with address verification, QR code generation, and tracking.

## Prerequisites
- POSTGRID_API_KEY configured
- Document already attested (has proof package)
- Payment method configured

## Workflow

### Phase 1: Address Extraction & Verification
1. Extract recipient address using OCR (Tesseract.js)
2. Verify address with PostGrid API
3. Display corrections and ask user to confirm
4. Handle invalid addresses gracefully

### Phase 2: QR Code Generation
1. Encode Arweave URL + Witnet ID + Job reference
2. Generate QR code image (qrcode library)
3. Embed in mail template

### Phase 3: Cost Calculation
1. Query PostGrid for pricing
2. Display itemized costs (print + postage + fees)
3. Get user confirmation

### Phase 4: Job Submission
1. Create PostGrid print job
2. Wait for job acceptance
3. Retrieve job ID and tracking number
4. Generate initial receipt

### Phase 5: Receipt Attestation
1. Upload receipt to Arweave
2. Attest receipt on Witnet
3. Return receipt URL to user

## Error Handling
- **Invalid address**: Request manual entry
- **PostGrid API error**: Retry with backoff
- **Payment failure**: Alert user and rollback
- **Job rejection**: Notify user with reason

## Success Criteria
- Address verified
- Job submitted to PostGrid
- Tracking number obtained
- Receipt attested on blockchain

## Version History
- 1.0.0: Initial release
