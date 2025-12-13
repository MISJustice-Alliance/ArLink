---
description: "Multi-agent parallel feature development"
allowed-tools: ["Read", "Write", "Bash(git:*)", "Task"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Orchestrate Feature - Multi-Agent Development

## Purpose
Coordinate multiple agents for parallel feature development using git worktrees.

## Workflow

1. **Read feature requirements**
   Ask user to describe feature:
   - What needs to be implemented?
   - Which modules are involved?
   - Are there multiple approaches to explore?

2. **Create MULTI_AGENT_PLAN.md**
   Generate plan with:
   - Overall goal
   - Task breakdown
   - Dependencies
   - Agent assignments
   - Success criteria

3. **Determine parallelization strategy**
   Analyze if feature benefits from:
   - Multiple implementation approaches (compare alternatives)
   - Independent parallel work (frontend + backend)
   - Large-scale work requiring concurrency

4. **Create git worktrees**
   For each parallel track:
   ```bash
   !git worktree add ../worktree-agent-1 feature/approach-1
   !git worktree add ../worktree-agent-2 feature/approach-2
   !git worktree add ../worktree-agent-3 feature/approach-3
   ```

5. **Spawn worker agents**
   Use Task tool to spawn agents in each worktree:
   - Agent 1: Load blockchain-attestation-skill + builder-role-skill
   - Agent 2: Load document-analysis-skill + builder-role-skill
   - Agent 3: Load mail-integration-skill + builder-role-skill

6. **Monitor progress**
   Poll MULTI_AGENT_PLAN.md for task status updates:
   - Agent 1: [Status]
   - Agent 2: [Status]
   - Agent 3: [Status]

7. **Synthesize results**
   When all agents complete:
   - Compare implementations
   - Select best approach or merge components
   - Resolve conflicts
   - Create unified PR

8. **Cleanup worktrees**
   ```bash
   !git worktree remove ../worktree-agent-1
   !git worktree remove ../worktree-agent-2
   !git worktree remove ../worktree-agent-3
   ```

9. **Update MULTI_AGENT_PLAN.md**
   Mark as complete, document chosen approach

## Success Criteria
- Plan created
- Agents spawned successfully
- Work completed in parallel
- Results synthesized
- Worktrees cleaned up

## Notes
- Use ONLY for features benefiting from parallelization
- Default to single agent + skills for sequential work
- Hybrid approach: Workers load skills dynamically

## Version History
- 1.0: Initial release
