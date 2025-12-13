---
name: validator-role-skill
version: 1.0.0
triggers:
  - "validate code"
  - "review code"
  - "run tests"
  - "check coverage"
dependencies: []
tools:
  - Read
  - Bash(npm:*)
  - Test
---

# Skill: Validator Role (Testing & Review)

## Objective
Ensure code quality through comprehensive testing, coverage analysis, and code review.

## Prerequisites
- Code implementation exists
- Test framework configured (Jest)
- Coverage tools configured

## Workflow

### Phase 1: Test Execution

1. **Run unit tests**
   ```bash
   npm test
   ```

2. **Run integration tests**
   ```bash
   npm run test:integration
   ```

3. **Run performance tests**
   ```bash
   npm run test:performance
   ```

4. **Run security tests**
   ```bash
   npm run test:security
   ```

### Phase 2: Coverage Analysis

1. **Generate coverage report**
   ```bash
   npm run test:coverage
   ```

2. **Check coverage thresholds**
   - Phase 1: >80% overall, 100% integrity module
   - Phase 2: >80% overall
   - Phase 3: >80% overall
   - Phase 4: >85% overall, >90% mail/payment modules

3. **Identify uncovered lines**
   - Review coverage report HTML
   - List files below threshold
   - Prioritize critical paths

### Phase 3: Code Review

1. **TypeScript compliance**
   - Strict mode enabled: `noImplicitAny: true`
   - All types explicit
   - No `any` types except in error handling

2. **Error handling**
   - All promises have `.catch()` or `try/catch`
   - Error types specific (not generic Error)
   - User-friendly error messages

3. **Security checks**
   - No hardcoded secrets
   - PII masked in logs
   - Input validation present
   - SQL injection prevention (if database)

4. **Performance checks**
   - No unnecessary loops
   - Efficient algorithms
   - Proper caching where appropriate

5. **Code style**
   - Consistent formatting (Prettier)
   - Meaningful variable names
   - Functions <50 lines
   - Cyclomatic complexity <10

### Phase 4: Security Audit

1. **Dependency vulnerabilities**
   ```bash
   npm audit
   ```

2. **Secret detection**
   - Scan for API keys in code
   - Check .env is gitignored
   - Verify no credentials in logs

3. **PII handling**
   - SSNs masked before logging
   - Credit cards masked
   - Personal data encrypted

### Phase 5: Validation Report

Generate comprehensive report:
```
Validation Report - [Module Name]
================================

Test Results:
- Unit Tests: X/Y passing (Z%)
- Integration Tests: X/Y passing (Z%)
- Performance Tests: X/Y passing (Z%)

Coverage:
- Overall: X% (Threshold: 80%)
- integrity/: 100% ✓
- storage/: X%
- analysis/: X%

Code Quality:
- TypeScript Strict: ✓ Passing
- Linting: ✓ No issues
- Security Audit: X vulnerabilities found

Recommendations:
- Increase coverage in storage/ module
- Fix security vulnerability in dependency X
- Refactor function Y for better performance
```

### Phase 6: Approval or Rejection

**Approve if:**
- All tests passing
- Coverage meets thresholds
- No security vulnerabilities
- Code quality acceptable

**Reject if:**
- Tests failing
- Coverage below threshold
- Security issues present
- Code quality concerns

## Testing Checklist

### Unit Tests
- [ ] All public functions tested
- [ ] Edge cases covered
- [ ] Error conditions tested
- [ ] Mocks used appropriately
- [ ] Deterministic (no flaky tests)

### Integration Tests
- [ ] Real API calls tested (sandbox/testnet)
- [ ] Network error handling tested
- [ ] Rate limiting tested
- [ ] Authentication tested
- [ ] Timeout handling tested

### Performance Tests
- [ ] Benchmarks meet targets
- [ ] No performance regressions
- [ ] Memory usage acceptable
- [ ] CPU usage acceptable

### Security Tests
- [ ] PII masking validated
- [ ] Authentication tested
- [ ] Authorization tested
- [ ] Input validation tested
- [ ] SQL injection prevented (if applicable)

## Code Review Checklist

### Architecture
- [ ] Follows module boundaries (AGENTS.md)
- [ ] Single responsibility principle
- [ ] Appropriate abstraction level
- [ ] Minimal coupling

### Implementation
- [ ] TypeScript strict mode compliant
- [ ] Error handling comprehensive
- [ ] Async/await used correctly
- [ ] No code duplication
- [ ] Functions well-named

### Documentation
- [ ] JSDoc comments on public functions
- [ ] README updated if needed
- [ ] Examples provided
- [ ] Complex logic explained

### Testing
- [ ] Tests written before code (TDD)
- [ ] Coverage meets thresholds
- [ ] Integration tests included
- [ ] Performance tests included (if applicable)

## Success Criteria
- All tests passing
- Coverage meets phase requirements
- Code review checklist complete
- Security audit clean
- Approval given or actionable feedback provided

## Example Usage
```bash
# User request: "Validate the Arweave storage module"
Agent loads: validator-role-skill
Agent executes:
  1. Run all tests
  2. Generate coverage report
  3. Review code quality
  4. Run security audit
  5. Generate validation report
Agent returns: Approval or list of issues to fix
```

## Version History
- 1.0.0: Initial release with comprehensive validation workflow
