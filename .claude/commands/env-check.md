---
description: "Validate development environment"
allowed-tools: ["Read", "Bash"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Env Check - Environment Validation

## Purpose
Validate development environment setup for ArweaveStamp.

## Workflow

1. **Check Node.js version**
   ```bash
   !node --version
   ```
   Required: >=20.x

2. **Check npm version**
   ```bash
   !npm --version
   ```

3. **Check TypeScript installation**
   ```bash
   !npx tsc --version
   ```

4. **Check environment variables**
   Verify existence (not values):
   - ARWEAVE_ENDPOINT
   - ARWEAVE_WALLET_PATH
   - CLAUDE_API_KEY
   - WITNET_ENDPOINT
   - ETH_RPC_URL
   - POLYGON_RPC_URL
   - AVAX_RPC_URL

5. **Phase 4 environment** (if applicable):
   - POSTGRID_API_KEY
   - STRIPE_SECRET_KEY
   - BTCPAY_URL
   - BTCPAY_API_KEY

6. **Check git configuration**
   ```bash
   !git config user.name
   !git config user.email
   ```

7. **Generate health report**
   Display:
   - ✓/✗ Node.js (version X)
   - ✓/✗ npm (version X)
   - ✓/✗ TypeScript (version X)
   - ✓/✗ Environment variables (X/Y configured)
   - ✓/✗ Git configuration
   - Overall: READY / ISSUES FOUND

## Success Criteria
- All tools installed
- Environment variables configured
- Git setup complete
- Ready for development

## Version History
- 1.0: Initial release
