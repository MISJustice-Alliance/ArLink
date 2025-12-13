---
description: "Generate or update development plans for ArweaveStamp features"
allowed-tools: ["Read", "Edit", "Write"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Plan - Development Planning

## Purpose
Create or update development plans following ArweaveStamp's phase-based architecture.

## Workflow

1. **Read current planning documents**
   - @DEVELOPMENT_PLAN.md (overall roadmap)
   - @PROJECT_CHECKLIST.md (weekly deliverables)
   - @AGENTS.md (module boundaries)

2. **Identify planning scope**
   Ask user:
   - Are you planning a new feature or updating existing plan?
   - Which phase does this relate to? (1-4)
   - Which modules are involved?

3. **Generate plan structure**
   Create plan with sections:
   - **Objective**: What needs to be accomplished
   - **Modules Affected**: List from AGENTS.md
   - **Implementation Steps**: Numbered, actionable tasks
   - **Testing Requirements**: Unit, integration, performance
   - **Dependencies**: Arweave, Claude, Witnet, PostGrid, etc.
   - **Acceptance Criteria**: Measurable outcomes
   - **Estimated Complexity**: Simple/Moderate/Complex

4. **Write or update plan file**
   - For features: Create `plans/feature-name.md`
   - For phases: Update `DEVELOPMENT_PLAN.md`
   - Follow markdown structure with clear sections

5. **Validate plan completeness**
   Check that plan includes:
   - TypeScript strict mode considerations
   - TDD approach (tests first)
   - Security requirements (PII masking, secrets)
   - Integration testing strategy
   - Coverage targets (>80% Phase 1, >85% Phase 4)

## Success Criteria
- Clear, actionable plan created
- Aligns with phase architecture
- Includes testing and security
- Ready for implementation

## Version History
- 1.0: Initial release
