---
description: "Streamline pull request creation with validation"
allowed-tools: ["Bash(git:*)", "Bash(npm:*)", "Bash(gh:*)"]
author: "ArweaveStamp Team"
version: "1.0"
---

# PR - Pull Request Automation

## Purpose
Run all pre-PR quality checks and create pull request with comprehensive description.

## Workflow

1. **Run linter**
   ```bash
   !npm run lint
   ```

2. **Run tests**
   ```bash
   !npm test
   ```

3. **Check coverage**
   ```bash
   !npm run test:coverage
   ```

4. **Git status check**
   ```bash
   !git status --porcelain
   ```

5. **Stage changes**
   ```bash
   !git add .
   ```

6. **Create commit**
   Ask user for commit message or use $1 parameter
   ```bash
   !git commit -m "$COMMIT_MSG"
   ```

7. **Push to remote**
   ```bash
   !git push -u origin $(git branch --show-current)
   ```

8. **Generate PR description**
   Analyze changes and create description:
   - Summary of changes
   - Modules affected
   - Testing done
   - Phase alignment

9. **Create PR**
   ```bash
   !gh pr create --title "$TITLE" --body "$BODY"
   ```

10. **Display PR link**
    Show user the PR URL for review

## Success Criteria
- Lint passing
- Tests passing
- Coverage adequate
- PR created successfully
- PR link displayed

## Version History
- 1.0: Initial release
