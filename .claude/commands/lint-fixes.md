---
description: "Auto-fix code style issues"
allowed-tools: ["Bash(npm:*)"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Lint Fixes - Code Style Automation

## Purpose
Automatically fix linting and formatting issues across codebase.

## Workflow

1. **Run ESLint auto-fix**
   ```bash
   !npm run lint:fix
   ```

2. **Run Prettier formatting**
   ```bash
   !npm run format
   ```

3. **Type check**
   ```bash
   !npm run type-check
   ```

4. **Display results**
   - Files modified: [count]
   - Issues fixed: [count]
   - Remaining issues: [count]

5. **Git diff** (optional)
   Show changes made:
   ```bash
   !git diff
   ```

6. **Suggest commit**
   If changes made:
   ```bash
   !git add .
   !git commit -m "style: auto-fix linting issues"
   ```

## Success Criteria
- Linting issues fixed
- Code formatted consistently
- Type errors resolved (if any)
- Changes ready to commit

## Version History
- 1.0: Initial release
