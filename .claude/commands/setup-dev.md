---
description: "Developer onboarding automation"
allowed-tools: ["Read", "Write", "Bash"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Setup Dev - Developer Onboarding

## Purpose
Automate developer environment setup for ArweaveStamp.

## Workflow

1. **Clone repository** (if needed)
   ```bash
   !git clone https://github.com/enuno/arweave-stamp.git
   !cd arweave-stamp
   ```

2. **Install dependencies**
   ```bash
   !npm install
   ```

3. **Create .env file**
   ```bash
   !cp .env.example .env
   ```

4. **Prompt for API keys**
   Ask user to provide:
   - Arweave wallet path
   - Claude API key
   - Witnet endpoint (or use default)
   - L1 RPC URLs
   - PostGrid API key (Phase 4)
   - Stripe/BTCPay credentials (Phase 4)

5. **Write .env file**
   Populate with user-provided values

6. **Run initial tests**
   ```bash
   !npm test
   ```

7. **Generate onboarding checklist**
   Display:
   - ✓ Repository cloned
   - ✓ Dependencies installed
   - ✓ Environment configured
   - ✓ Tests passing
   - Next: Read README.md and DEVELOPMENT_PLAN.md
   - Next: Run /start-session to begin work

## Success Criteria
- Repository cloned
- Dependencies installed
- Environment configured
- Initial tests passing
- Developer ready to contribute

## Version History
- 1.0: Initial release
