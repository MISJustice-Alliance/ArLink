---
name: phase-validation-skill
version: 1.0.0
triggers:
  - "validate phase completion"
  - "phase readiness check"
  - "milestone validation"
dependencies:
  - validator-role-skill
tools:
  - Read
  - Bash(npm:*)
---

# Skill: Phase Validation

## Objective
Validate completion criteria for each development phase (1-4).

## Prerequisites
- DEVELOPMENT_PLAN.md with phase requirements
- PROJECT_CHECKLIST.md with weekly deliverables
- Test suite configured

## Workflow

### Phase 1: Requirements Gathering
1. Read DEVELOPMENT_PLAN.md for selected phase
2. Extract module list and acceptance criteria
3. Read PROJECT_CHECKLIST.md for weekly deliverables

### Phase 2: Module Completeness Check
1. Verify all planned modules exist
2. Check for missing source files
3. Validate module exports

### Phase 3: Test Validation
1. Run phase-specific test suites
2. Check coverage meets thresholds:
   - Phase 1: >80% overall, 100% integrity module
   - Phase 2: >80% overall
   - Phase 3: >80% overall
   - Phase 4: >85% overall, >90% mail/payment

### Phase 4: Integration Validation
1. Run integration tests for phase
2. Verify external service connectivity
3. Check API endpoints functional

### Phase 5: Security Audit
1. Run npm audit for vulnerabilities
2. Check for hardcoded secrets
3. Verify PII masking in logs
4. Review authentication/authorization

### Phase 6: Documentation Check
1. Verify README.md updated
2. Check ARCHITECTURE.md reflects phase
3. Validate API documentation exists
4. Review usage examples

### Phase 7: Validation Report
Generate comprehensive report:
```
Phase [1-4] Validation Report
=============================

Module Completeness:
- Expected: [X] modules
- Found: [Y] modules
- Missing: [list]

Test Results:
- Total Tests: [count]
- Passing: [count]
- Failing: [count]
- Coverage: [percentage]

Integration:
- Service Health: [status]
- API Endpoints: [status]

Security:
- Vulnerabilities: [count]
- Secrets: [status]
- PII Handling: [status]

Documentation:
- README: [status]
- ARCHITECTURE: [status]
- API Docs: [status]

Overall Status: READY / NOT READY
```

## Success Criteria
- All modules present
- Tests passing with adequate coverage
- Integration tests successful
- No critical security issues
- Documentation complete

## Version History
- 1.0.0: Initial release
