---
name: blockchain-attestation-skill
version: 1.0.0
triggers:
  - "attest document"
  - "upload to arweave"
  - "submit to witnet"
  - "relay to blockchain"
dependencies: []
tools:
  - Read
  - Bash(npm:*)
---

# Skill: Blockchain Attestation

## Objective
Automate the complete blockchain attestation workflow: Arweave upload → Claude analysis → Hash computation → Witnet attestation → L1 relay.

## Prerequisites
- Document file exists locally
- ARWEAVE_WALLET_PATH configured
- CLAUDE_API_KEY configured
- WITNET_ENDPOINT configured
- L1 RPC URLs configured

## Workflow

### Phase 1: Pre-flight Checks
1. Verify environment variables exist
2. Validate file exists and size <100MB
3. Check MIME type (PDF, image, text)

### Phase 2: Arweave Upload
1. Initialize Arweave wallet
2. Upload file with retry logic (exponential backoff)
3. Wait for transaction confirmation
4. Retrieve Arweave URL and transaction ID

### Phase 3: Claude Analysis
1. Read file content
2. Mask PII (SSNs, credit cards)
3. Send to Claude for analysis
4. Parse response (classification, entities, summary)
5. Validate confidence scores (>0.8 threshold)

### Phase 4: Hash Computation
1. Compute SHA-256 hash of file content
2. Compute SHA-256 hash of metadata
3. Assemble document ID (combined hash)
4. Verify determinism (recompute and compare)

### Phase 5: Witnet Attestation
1. Construct Witnet request payload
2. Submit to Witnet oracle
3. Poll for finalization (max 5 minutes)
4. Verify report signature

### Phase 6: L1 Blockchain Relay
1. Relay to Ethereum (submit transaction, wait for confirmations)
2. Relay to Polygon (submit transaction, wait for confirmations)
3. Relay to Avalanche (submit transaction, wait for confirmations)
4. Verify all L1 confirmations (>12 blocks)

### Phase 7: Proof Package Generation
1. Assemble ProofPackage with all data
2. Serialize to JSON
3. Store locally
4. Return proof package ID to user

## Error Handling
- **File too large**: Alert user, suggest splitting
- **Arweave network error**: Retry with exponential backoff (3 attempts)
- **Claude rate limit**: Wait and retry
- **Witnet timeout**: Alert user, provide request ID for manual checking
- **L1 relay failure**: Continue with successful chains, log failed ones

## Success Criteria
- File uploaded to Arweave
- Claude analysis completed with confidence >0.8
- Witnet attestation finalized
- All L1 chains confirmed (or failures logged)
- Proof package generated and stored

## Example Usage
```bash
# User request: "Attest this contract document"
Agent loads: blockchain-attestation-skill
Agent executes: Full workflow from upload to proof generation
Agent returns: Proof package with Arweave URL, Witnet ID, L1 tx hashes
```

## Version History
- 1.0.0: Initial release
