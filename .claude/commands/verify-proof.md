---
description: "Verify proof package against blockchain"
allowed-tools: ["Read", "Bash(npm:*)"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Verify Proof - Independent Verification

## Purpose
Independently verify a proof package's integrity against blockchain data.

## Workflow

1. **Load proof package**
   ```bash
   !npm run cli -- verify $PROOF_PACKAGE_PATH
   ```

2. **Re-compute file hash**
   - Download file from Arweave URL
   - Compute SHA-256 hash
   - Compare to proof package hash

3. **Verify Witnet attestation**
   - Query Witnet for report
   - Verify report signature
   - Check finalization status

4. **Verify L1 confirmations**
   For each chain (Ethereum, Polygon, Avalanche):
   - Query transaction
   - Verify confirmation count (>12)
   - Check event logs

5. **Generate verification report**
   Display:
   - ✓ File hash matches
   - ✓ Witnet attestation valid
   - ✓ Ethereum confirmed (X blocks)
   - ✓ Polygon confirmed (X blocks)
   - ✓ Avalanche confirmed (X blocks)
   - Overall: VERIFIED / FAILED

6. **Export report** (optional)
   ```bash
   !npm run cli -- verify $PROOF --format pdf > verification_report.pdf
   ```

## Success Criteria
- All verification checks pass
- Report generated
- User informed of status

## Version History
- 1.0: Initial release
