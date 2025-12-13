---
description: "Validate phase completion criteria"
allowed-tools: ["Read", "Bash(npm:*)"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Validate Phase - Completion Validation

## Purpose
Run comprehensive validation for phase completion (accepts phase number 1-4).

## Parameters
- $1: Phase number (1, 2, 3, or 4)

## Workflow

1. **Read phase requirements**
   From DEVELOPMENT_PLAN.md and PROJECT_CHECKLIST.md

2. **Phase-specific validation**

   **Phase 1 (Document Attestation)**:
   ```bash
   !npm test -- src/storage src/analysis src/integrity src/oracle src/proofs
   !npm run test:coverage -- --threshold 80
   !npm run test:integration -- phase1.test.ts
   ```

   **Phase 2 (Web Dashboard)**:
   ```bash
   !npm test -- src/backend src/frontend
   !npm run test:coverage -- --threshold 80
   !npm run build:frontend
   ```

   **Phase 3 (File Monitoring)**:
   ```bash
   !npm test -- src/watcher src/policies src/audit
   !npm run test:performance -- watcher.perf.test.ts
   !npm run test:coverage -- --threshold 80
   ```

   **Phase 4 (Mail Integration)**:
   ```bash
   !npm test -- src/postal src/mail src/payment src/tracking
   !npm run test:coverage -- --threshold 85
   !npm run test:mail
   ```

3. **Check module completeness**
   For selected phase, verify all planned modules exist

4. **Run security audit**
   ```bash
   !npm audit --production
   ```

5. **Generate validation report**
   Display:
   - Phase: [1-4]
   - Test Results: [Pass/Fail]
   - Coverage: [percentage] (Threshold: [80/85]%)
   - Modules: [Complete/Incomplete]
   - Security: [Issues found]
   - Overall: READY / NOT READY

## Success Criteria
- All tests pass
- Coverage meets threshold
- Security audit clean
- Phase completion confirmed

## Version History
- 1.0: Initial release
