---
name: environment-setup-skill
version: 1.0.0
triggers:
  - "setup development environment"
  - "configure environment"
  - "developer onboarding"
dependencies: []
tools:
  - Read
  - Write
  - Bash
---

# Skill: Development Environment Setup

## Objective
Automate developer environment setup and validation.

## Prerequisites
- Git installed
- Node.js 20.x or higher
- npm or yarn

## Workflow

### Phase 1: System Requirements Check
1. Check Node.js version (>=20.x)
2. Check npm version
3. Check TypeScript globally available
4. Check git configuration

### Phase 2: Repository Setup
1. Clone repository (if not already cloned)
2. Install dependencies: `npm install`
3. Verify package.json scripts

### Phase 3: Environment Configuration
1. Copy .env.example to .env
2. Prompt user for required values:
   - ARWEAVE_ENDPOINT (default: https://arweave.net)
   - ARWEAVE_WALLET_PATH
   - CLAUDE_API_KEY
   - WITNET_ENDPOINT
   - ETH_RPC_URL, POLYGON_RPC_URL, AVAX_RPC_URL
3. Phase 4 specific (optional):
   - POSTGRID_API_KEY
   - STRIPE_SECRET_KEY
   - BTCPAY_URL, BTCPAY_API_KEY

### Phase 4: Arweave Wallet Setup
1. Check if wallet exists at ARWEAVE_WALLET_PATH
2. If not, guide user to create wallet:
   - Option 1: Use existing wallet file
   - Option 2: Generate new wallet (testnet)
3. Display wallet address

### Phase 5: Validation
1. Run environment check: `npm run env:check`
2. Run initial test suite: `npm test`
3. Verify build works: `npm run build`

### Phase 6: Documentation
1. Display onboarding checklist
2. Link to README.md
3. Link to DEVELOPMENT_PLAN.md
4. Suggest running /start-session

## Environment Variables

### Required (All Phases)
```bash
ARWEAVE_ENDPOINT=https://arweave.net
ARWEAVE_WALLET_PATH=/path/to/wallet.json
CLAUDE_API_KEY=sk-ant-...
WITNET_ENDPOINT=https://dr-api.witnet.io
ETH_RPC_URL=https://eth-mainnet.g.alchemy.com/v2/...
POLYGON_RPC_URL=https://polygon-mainnet.g.alchemy.com/v2/...
AVAX_RPC_URL=https://api.avax.network/ext/bc/C/rpc
```

### Phase 4 Only
```bash
POSTGRID_API_KEY=...
STRIPE_SECRET_KEY=sk_test_...
BTCPAY_URL=https://btcpay.example.com
BTCPAY_API_KEY=...
```

## Error Handling
- **Node.js version too old**: Provide download link
- **Missing dependencies**: Run npm install
- **Invalid API key**: Guide user to obtain valid key
- **Wallet not found**: Guide wallet creation process

## Success Criteria
- All system requirements met
- Dependencies installed
- Environment configured
- Wallet setup complete
- Initial tests passing

## Version History
- 1.0.0: Initial release
