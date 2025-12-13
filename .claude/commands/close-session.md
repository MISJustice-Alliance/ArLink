---
description: "End development session with progress summary"
allowed-tools: ["Read", "Edit", "Bash(git:*)"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Close Session - Session Summary

## Purpose
Gracefully close development session with summary of progress and next steps.

## Workflow

1. **Git status check**
   ```bash
   !git status --porcelain
   !git log --oneline -5
   ```

2. **Summarize session**
   Display:
   - Session duration (if SESSION_LOG.md exists)
   - Goals stated at start
   - Commits made during session
   - Files modified
   - Tests run (if any)

3. **Check for uncommitted work**
   If uncommitted changes exist:
   - List uncommitted files
   - Ask if user wants to commit or stash
   - Suggest next steps

4. **Update session log** (if exists)
   Add to SESSION_LOG.md:
   - End time
   - Summary of work completed
   - Next session goals

5. **Suggest next steps**
   Based on current state:
   - If tests failing: "Run /test-all to diagnose failures"
   - If uncommitted: "Commit work or run /pr to create pull request"
   - If phase incomplete: "Continue with next task from DEVELOPMENT_PLAN.md"

## Success Criteria
- Session summary displayed
- Uncommitted work handled
- Next steps clear
- Clean state for next session

## Version History
- 1.0: Initial release
