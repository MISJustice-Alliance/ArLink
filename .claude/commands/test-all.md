---
description: "Execute comprehensive test suite for ArweaveStamp"
allowed-tools: ["Bash(npm:*)"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Test All - Comprehensive Testing

## Purpose
Run all test suites (unit, integration, performance, security) and generate consolidated report.

## Workflow

1. **Run unit tests**
   ```bash
   !npm test
   ```

2. **Generate coverage report**
   ```bash
   !npm run test:coverage
   ```

3. **Run integration tests** (if implemented)
   ```bash
   !npm run test:integration
   ```

4. **Run performance tests** (if implemented)
   ```bash
   !npm run test:performance
   ```

5. **Run security tests** (if implemented)
   ```bash
   !npm run test:security
   ```

6. **Consolidate results**
   Display summary:
   - Total tests: [count]
   - Passing: [count] ✓
   - Failing: [count] ✗
   - Coverage: [percentage]
   - Performance: [Pass/Fail against benchmarks]
   - Security: [Issues found]

7. **Check coverage thresholds**
   - Phase 1: >80% overall, 100% integrity module
   - Phase 2: >80% overall
   - Phase 3: >80% overall
   - Phase 4: >85% overall, >90% mail/payment

8. **Report failures**
   If any tests fail:
   - List failing tests with error messages
   - Suggest next steps (check logs, run individual test)

## Success Criteria
- All test suites executed
- Coverage meets phase requirements
- Clear pass/fail status
- Actionable failure information

## Version History
- 1.0: Initial release
