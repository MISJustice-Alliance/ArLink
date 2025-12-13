---
name: proof-verification-skill
version: 1.0.0
triggers:
  - "verify proof package"
  - "validate attestation"
  - "check proof integrity"
dependencies:
  - crypto-hashing-skill
tools:
  - Read
  - Bash(npm:*)
---

# Skill: Proof Package Verification

## Objective
Independently verify proof package integrity against blockchain data.

## Prerequisites
- Proof package file exists
- Network access to Arweave, Witnet, and L1 chains
- RPC endpoints configured

## Workflow

### Phase 1: Proof Package Loading
1. Read proof package JSON file
2. Validate structure and required fields
3. Extract verification parameters:
   - Arweave URL
   - Document hash
   - Witnet request ID
   - L1 transaction IDs

### Phase 2: File Hash Verification
1. Download file from Arweave URL
2. Recompute SHA-256 hash
3. Compare to proof package hash
4. Status: MATCH / MISMATCH

### Phase 3: Witnet Attestation Verification
1. Query Witnet for request by ID
2. Verify report exists
3. Check report status (finalized)
4. Verify report signature
5. Compare attested hash to file hash
6. Status: VERIFIED / FAILED

### Phase 4: L1 Blockchain Verification
For each chain (Ethereum, Polygon, Avalanche):
1. Query transaction by ID
2. Verify transaction exists
3. Check confirmation count (>12 blocks)
4. Parse event logs for attested hash
5. Verify hash matches
6. Status: CONFIRMED / PENDING / FAILED

### Phase 5: Metadata Verification (Optional)
If proof package includes metadata:
1. Recompute metadata hash
2. Verify against proof package
3. Check Claude analysis confidence scores

### Phase 6: Verification Report
Generate comprehensive report:
```
Proof Verification Report
========================

Proof Package ID: [id]
Arweave URL: [url]
Verification Date: [timestamp]

File Hash Verification:
- Expected: [hash]
- Computed: [hash]
- Status: ✓ MATCH / ✗ MISMATCH

Witnet Attestation:
- Request ID: [id]
- Status: ✓ FINALIZED / ✗ FAILED
- Signature: ✓ VALID / ✗ INVALID
- Hash Match: ✓ YES / ✗ NO

Ethereum:
- TX ID: [tx]
- Confirmations: [count]
- Status: ✓ CONFIRMED / ⚠ PENDING / ✗ FAILED

Polygon:
- TX ID: [tx]
- Confirmations: [count]
- Status: ✓ CONFIRMED / ⚠ PENDING / ✗ FAILED

Avalanche:
- TX ID: [tx]
- Confirmations: [count]
- Status: ✓ CONFIRMED / ⚠ PENDING / ✗ FAILED

Overall Verification: ✓ VERIFIED / ✗ FAILED
```

### Phase 7: Export Report (Optional)
1. Generate PDF report
2. Save to file
3. Return file path

## Error Handling
- **Proof package not found**: Alert user
- **Network error**: Retry with backoff
- **Arweave file not found**: Report as FAILED
- **Witnet request not found**: Report as FAILED
- **L1 transaction not found**: Report as FAILED
- **Confirmation timeout**: Report as PENDING

## Security
- Verify cryptographic signatures
- Check all blockchain confirmations
- Don't trust single data source
- Cross-verify across multiple chains

## Success Criteria
- All verifications complete
- Report generated
- Status clear (VERIFIED/FAILED/PENDING)
- User informed of result

## Version History
- 1.0.0: Initial release
