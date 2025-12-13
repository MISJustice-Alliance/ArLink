---
description: "Review and update dependencies safely"
allowed-tools: ["Bash(npm:*)"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Deps Update - Dependency Management

## Purpose
Review outdated packages, security vulnerabilities, and safely update dependencies.

## Workflow

1. **Check for outdated packages**
   ```bash
   !npm outdated
   ```

2. **Run security audit**
   ```bash
   !npm audit
   ```

3. **Review findings**
   Display:
   - Packages with available updates
   - Security vulnerabilities (severity)
   - Breaking vs non-breaking updates

4. **Update non-breaking changes**
   ```bash
   !npm update
   ```

5. **Run tests after update**
   ```bash
   !npm test
   ```

6. **Generate update report**
   Display:
   - Packages updated
   - Security fixes applied
   - Test results
   - Breaking changes flagged for manual review

7. **Suggest next steps**
   If breaking changes exist:
   - List packages needing manual update
   - Link to migration guides

## Success Criteria
- Outdated packages identified
- Security fixes applied
- Tests passing after updates
- Breaking changes documented

## Version History
- 1.0: Initial release
