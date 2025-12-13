---
description: "Guide through document attestation workflow (Phase 1)"
allowed-tools: ["Read", "Bash(npm:*)"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Attest Document - Blockchain Attestation Workflow

## Purpose
Guide through the complete document attestation workflow: Upload → Analyze → Hash → Attest → Verify.

## Workflow

1. **Prerequisites check**
   Verify environment variables:
   - ARWEAVE_WALLET_PATH
   - CLAUDE_API_KEY
   - WITNET_ENDPOINT
   - ETH_RPC_URL, POLYGON_RPC_URL, AVAX_RPC_URL

2. **File validation**
   - Check file exists at provided path
   - Validate MIME type (PDF, image, text)
   - Check file size (<100MB)

3. **Run attestation command**
   ```bash
   !npm run cli -- stamp $FILE_PATH
   ```

4. **Monitor progress**
   Display stages:
   - [1/5] Uploading to Arweave...
   - [2/5] Analyzing with Claude...
   - [3/5] Computing hash...
   - [4/5] Submitting to Witnet...
   - [5/5] Relaying to L1 chains...

5. **Display results**
   Show:
   - Arweave URL
   - Document ID
   - Witnet request ID
   - L1 transaction IDs (Ethereum, Polygon, Avalanche)
   - Proof package location

6. **Verify attestation** (optional)
   ```bash
   !npm run cli -- verify $PROOF_PACKAGE_PATH
   ```

## Success Criteria
- File uploaded to Arweave
- Claude analysis completed
- Witnet attestation confirmed
- L1 chains relayed
- Proof package generated

## Version History
- 1.0: Initial release
