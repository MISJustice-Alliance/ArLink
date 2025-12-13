---
name: integration-testing-skill
version: 1.0.0
triggers:
  - "run integration tests"
  - "test external services"
  - "api integration test"
dependencies:
  - validator-role-skill
tools:
  - Read
  - Bash(npm:*)
  - Test
---

# Skill: Integration Testing

## Objective
Test interactions with external services (Arweave, Claude, Witnet, PostGrid, payment gateways).

## Prerequisites
- Test environment configured
- API keys for sandbox/testnet environments
- Test fixtures available

## Workflow

### Phase 1: Environment Setup
1. Verify test API keys exist
2. Configure sandbox endpoints
3. Prepare test data and fixtures

### Phase 2: Service Health Checks
1. Test Arweave connectivity
2. Test Claude API access
3. Test Witnet endpoint
4. Test L1 RPC endpoints
5. Test PostGrid API (Phase 4)
6. Test payment gateways (Phase 4)

### Phase 3: End-to-End Workflows
1. **Document Attestation Flow**:
   - Upload test file to Arweave testnet
   - Analyze with Claude
   - Submit to Witnet testnet
   - Verify proof package

2. **Mail Creation Flow** (Phase 4):
   - Create test mail job
   - Process test payment
   - Verify job submitted
   - Check tracking created

### Phase 4: Error Handling Tests
1. Test network timeouts
2. Test rate limiting
3. Test invalid credentials
4. Test malformed data
5. Test service unavailability

### Phase 5: Performance Testing
1. Measure API response times
2. Test concurrent requests
3. Verify retry logic
4. Check resource cleanup

## Test Organization
```
tests/integration/
├── arweave/
│   ├── upload.test.ts
│   └── download.test.ts
├── claude/
│   ├── analysis.test.ts
│   └── vision.test.ts
├── witnet/
│   ├── attestation.test.ts
│   └── verification.test.ts
├── postgrid/
│   └── mail-job.test.ts
└── payments/
    ├── stripe.test.ts
    └── btcpay.test.ts
```

## Success Criteria
- All health checks pass
- E2E workflows complete successfully
- Error handling works correctly
- Performance meets targets
- No resource leaks

## Version History
- 1.0.0: Initial release
