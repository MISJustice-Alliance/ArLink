---
description: "Test all external service connections"
allowed-tools: ["Bash(npm:*)"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Integration Check - Service Health

## Purpose
Test connectivity and health of all external services (Arweave, Claude, Witnet, PostGrid, Stripe, BTCPay).

## Workflow

1. **Check environment variables**
   Verify all required API keys and endpoints exist

2. **Arweave health check**
   ```bash
   !curl -s https://arweave.net/info | jq .height
   ```

3. **Claude API check**
   ```bash
   !npm run test:integration -- claude-ping.test.ts
   ```

4. **Witnet health check**
   ```bash
   !curl -s https://dr-api.witnet.io/health
   ```

5. **L1 RPC health checks**
   ```bash
   !curl -s $ETH_RPC_URL -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
   !curl -s $POLYGON_RPC_URL -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
   !curl -s $AVAX_RPC_URL -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
   ```

6. **PostGrid health check** (Phase 4)
   ```bash
   !npm run test:integration -- postgrid-ping.test.ts
   ```

7. **Payment gateway checks** (Phase 4)
   ```bash
   !npm run test:integration -- stripe-ping.test.ts
   !npm run test:integration -- btcpay-ping.test.ts
   ```

8. **Generate health report**
   Display:
   - ✓/✗ Arweave (block height: X)
   - ✓/✗ Claude API
   - ✓/✗ Witnet
   - ✓/✗ Ethereum RPC (block: X)
   - ✓/✗ Polygon RPC (block: X)
   - ✓/✗ Avalanche RPC (block: X)
   - ✓/✗ PostGrid API
   - ✓/✗ Stripe API
   - ✓/✗ BTCPay Server
   - Overall: HEALTHY / DEGRADED / DOWN

## Success Criteria
- All services reachable
- API keys valid
- Endpoints responding
- Clear health status

## Version History
- 1.0: Initial release
