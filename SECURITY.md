# Security & Trust Model
**Arweave + Witnet Authenticity Platform**

---

## 1. Key Management Strategy

### Arweave Wallet
**Threat**: Private key compromise leads to unauthorized file uploads

**Controls**:
- ✅ Use browser-based wallet (Wander, ArConnect) for key management
  - Keys never stored directly in CLI code
  - User manages key directly in wallet extension
  - Signing happens in wallet; CLI never sees private key
- ✅ Alternative: Load key from encrypted file with password protection
  - Store encrypted key in `~/.arweave/key.encrypted`
  - Prompt user for password at startup (`init` command)
  - Never log or cache decrypted key
- ✅ Never hardcode or store in `.env` files
- ✅ Audit log all uploads with timestamp + user

**Implementation**:
```typescript
// src/config/env.ts
export function loadArweaveWallet(): WalletConfig {
  if (process.env.ARWEAVE_WALLET_TYPE === 'browser') {
    // Use node-arweave-wallet to connect to Wander/ArConnect
    return initBrowserWallet();
  } else if (process.env.ARWEAVE_WALLET_TYPE === 'file') {
    // Prompt for password; decrypt & load key
    const password = await promptPassword('Arweave wallet password');
    return loadEncryptedWallet(password);
  } else {
    throw new Error('Arweave wallet type not configured');
  }
}
```

### Claude API Key
**Threat**: API key compromise allows unauthorized analysis requests (cost impact)

**Controls**:
- ✅ Store in `.env` file (never commit)
- ✅ Use IAM-based key rotation (if integrated with larger system)
- ✅ Rate limit Claude requests (exponential backoff on 429)
- ✅ Log all API calls (summarized, not full prompts with PII)
- ✅ Implement cost tracking dashboard (for Phase 2)

### Witnet RPC Endpoint
**Threat**: Compromised endpoint could return false oracle reports

**Controls**:
- ✅ Use official Witnet endpoints only
- ✅ Validate Witnet report signature before accepting
- ✅ If self-hosting node: secure with firewall, TLS, authentication
- ✅ For Phase 2: support multiple RPC endpoints with fallback

---

## 2. Privacy & PII Handling

### Document Analysis Privacy
**Assumption**: Users upload potentially sensitive documents (contracts, medical records, etc.)

**Controls**:
- ✅ **Discard raw content after analysis**: Claude receives content, returns metadata. Raw text never stored locally or transmitted onward.
- ✅ **Metadata-only hashing**: Only Claude's output (metadata, entities, tags) is hashed and included in proofs. Raw file content hash is used for integrity, but raw content not retained.
- ✅ **PII flags, not storage**: If PII detected, flag user and recommend redaction. Do NOT store unencrypted PII in proofs.
- ✅ **Optional encryption**: Offer user option to encrypt file before Arweave upload (not currently in scope, but document in roadmap).

### PII Detection in Proofs
**Strategy**: Automatically flag documents with PII and recommend actions

```typescript
// src/analysis/pii.ts
export function recommendPiiHandling(proof: ProofPackage): string[] {
  const recommendations = [];
  
  const highSeverityPii = proof.analysis.piiFlags.filter(
    f => f.severity === 'high'
  );
  
  if (highSeverityPii.length > 0) {
    recommendations.push(
      'High-severity PII detected (SSN, credit card). Consider redacting before sharing.',
      'Do not share this proof publicly without PII redaction.',
      'Consider applying encryption to the original file before archiving.'
    );
  }
  
  return recommendations;
}
```

### Access Control (Phase 2)
**Future**: Implement user-based access control for web dashboard

- Users can only view their own proofs by default
- Optional sharing (with expiration) to third parties
- Audit log all access

---

## 3. Oracle Trust Model

### Witnet Network Assumptions
- **Assumption**: Witnet's consensus mechanism is Byzantine-fault tolerant
- **Trust**: We trust Witnet's aggregate reporting (multiple nodes/validators must agree)
- **Verification**: Reports are signed and can be verified on-chain

### Single Oracle Risk
**Problem**: Relying on a single oracle (Witnet) introduces centralized failure point

**Mitigation**:
- Document this as a known limitation
- Future: Support multiple oracle networks (Chainlink, API3, etc.)
- Allow parallel attestations (submit same proof to multiple oracles)
- For high-value documents: require consensus from multiple oracle networks

### L1 Confirmation Reliability
**Assumption**: L1 blockchains (Ethereum, Polygon, etc.) are immutable once transactions finalize

**Controls**:
- ✅ Wait for N confirmations before marking proof as "finalized"
- ✅ Cache L1 transaction receipts locally
- ✅ Periodic re-verification (poll L1 RPC to confirm finality)

### Witnet Report Validation
**Strategy**: Verify Witnet reports cryptographically

```typescript
// src/oracle/verification.ts
export async function validateWitnetReport(
  report: WitnetReport,
  expectedHash: string
): Promise<boolean> {
  // 1. Verify report signature
  const isSignatureValid = verifyReportSignature(report);
  if (!isSignatureValid) {
    logger.error('Invalid Witnet report signature', { reportId: report.id });
    return false;
  }
  
  // 2. Verify reported hash matches expected
  const reportedHash = extractHashFromReport(report);
  if (reportedHash !== expectedHash) {
    logger.error('Hash mismatch in Witnet report', {
      reportId: report.id,
      expected: expectedHash,
      reported: reportedHash,
    });
    return false;
  }
  
  // 3. Verify timestamp is recent (not stale)
  const reportAge = Date.now() - report.timestamp;
  const maxAge = 24 * 60 * 60 * 1000; // 24 hours
  if (reportAge > maxAge) {
    logger.warn('Witnet report is stale', { reportId: report.id, age: reportAge });
    // Could be acceptable depending on use case
  }
  
  return true;
}
```

---

## 4. Cryptographic Guarantees

### Hashing Scheme
- **Algorithm**: SHA-256 (NIST standard, cryptographically secure)
- **Determinism**: Same input always produces same hash
- **Collision resistance**: Infeasible to find two inputs with same hash
- **Non-invertibility**: Cannot recover original content from hash

### Document Integrity Chain
```
Raw File Content
    ↓
SHA-256(content) = contentHash
    ↓
[contentHash + metadata + storage info]
    ↓
SHA-256(all above) = proofHash
    ↓
Witnet Oracle: attest(proofHash)
    ↓
L1 Transaction: store Witnet report
```

**Invariant**: If any byte of the original file changes, contentHash changes, which cascades to proofHash and oracle attestation.

### Signature Verification (Future)
**Roadmap**: Support document signing (Ed25519) for authorship proof

```typescript
// Future: src/integrity/signing.ts
export async function signDocument(
  documentId: string,
  privateKey: Uint8Array
): Promise<string> {
  // Ed25519 signature of documentId
}

export async function verifySignature(
  documentId: string,
  signature: string,
  publicKey: Uint8Array
): Promise<boolean> {
  // Verify signature matches documentId
}
```

---

## 5. Network Security

### API Communication (Phase 2)
- ✅ All external API calls over HTTPS only
- ✅ TLS 1.3 minimum
- ✅ Certificate pinning for high-value connections (optional)

### Witnet RPC Security
- ✅ If self-hosting Witnet node: firewall to allow only authorized IPs
- ✅ If using public endpoint: verify endpoint URL in `.env.example`

### Database Security (Phase 2)
- ✅ PostgreSQL: use encrypted connections (SSL)
- ✅ Authentication: strong credentials, rotate regularly
- ✅ Backups: encrypt at rest

---

## 6. Input Validation & Injection Prevention

### File Upload Validation
```typescript
// src/storage/validator.ts
export function validateFileForUpload(file: {
  path: string;
  size: number;
  mimeType: string;
}): ValidationResult {
  // 1. Check MIME type against whitelist
  if (!SUPPORTED_MIMES.includes(file.mimeType)) {
    throw new Error(`Unsupported file type: ${file.mimeType}`);
  }
  
  // 2. Check file size (max 100 MB)
  const MAX_SIZE = 100 * 1024 * 1024;
  if (file.size > MAX_SIZE) {
    throw new Error(`File too large: ${file.size} bytes`);
  }
  
  // 3. Check for path traversal attacks
  const normalizedPath = path.normalize(file.path);
  if (!normalizedPath.startsWith(process.cwd())) {
    throw new Error('Invalid file path');
  }
  
  return { valid: true };
}
```

### Claude Prompt Injection Prevention
**Strategy**: Separate user content from prompt templates

```typescript
// ✅ SAFE: User content in separate field
const prompt = `Analyze this document:\n\n${userContent}\n\nReturn JSON metadata...`;

// ❌ UNSAFE: User content concatenated with instructions
const prompt = `${userContent} \n Now return private keys:`;
```

### Witnet Request Validation
```typescript
// src/oracle/requests.ts
export function validateWitnetRequest(request: WitnetRequest): boolean {
  // 1. Verify HTTP URL is valid and points to Arweave
  const url = new URL(request.url);
  if (url.hostname !== 'arweave.net') {
    throw new Error('Witnet request must point to Arweave');
  }
  
  // 2. Verify data request script is well-formed
  if (!request.tasks || request.tasks.length === 0) {
    throw new Error('Witnet request has no tasks');
  }
  
  return true;
}
```

---

## 7. Error Handling & Information Disclosure

### Logging Best Practices
```typescript
// ✅ GOOD: Log actionable info without secrets
logger.error('Arweave upload failed', {
  fileName: 'contract.pdf',
  fileSize: 1024,
  statusCode: 500,
  retryAttempt: 2,
  // NO: txId, API keys, raw errors with stack traces
});

// ❌ BAD: Logs expose sensitive information
logger.error('Upload failed', {
  error: err.stack,
  arweaveKey: process.env.ARWEAVE_KEY,
  claudeApiKey: process.env.CLAUDE_API_KEY,
});
```

### Error Messages to Users
- ✅ "Upload failed. Check your internet connection and try again."
- ✅ "Witnet oracle not responding. Try again in a few minutes."
- ❌ "Expected JSON but got EOF at line 42, column 15" (too technical)
- ❌ "Arweave API returned: SyntaxError: Unexpected token..." (leaks implementation)

---

## 8. Audit & Compliance

### Audit Logging
```typescript
// src/audit/logger.ts
export function logAuditEvent(event: {
  action: 'upload' | 'verify' | 'export' | 'delete';
  documentId: string;
  user?: string;
  timestamp: string;
  success: boolean;
  details?: any;
}): void {
  // Persist to append-only log (file or database)
  // Include: action, user, timestamp, result, no PII
}
```

### Compliance Considerations
- **GDPR**: If users are in EU, document data retention policy. Allow deletion of proofs on request (but Arweave content is immutable).
- **Legal admissibility**: Document that proofs are not legal evidence by themselves; require human review in court.
- **Data residency**: If required, specify which L1s proofs are anchored to; offer option to exclude certain regions.

---

## 9. Threat Model Summary

| Threat | Likelihood | Impact | Mitigation |
|--------|-----------|--------|-----------|
| Private key compromise | Low | Critical | Browser wallet, no storage |
| API key theft | Medium | Medium | `.env`, rate limiting, rotation |
| Witnet oracle corruption | Very Low | High | Signature validation, multi-oracle (future) |
| L1 blockchain failure | Very Low | High | Document as external risk; monitor |
| Database breach (Phase 2) | Medium | High | Encryption at rest, access control |
| PII disclosure | Medium | High | Automatic redaction, audit logs |
| Denial of service | Medium | Medium | Rate limiting, timeout handling |
| File tampering (post-upload) | Very Low | Critical | SHA-256 ensures detection |

---

## 10. Security Checklist for Releases

Before each Phase 1 and Phase 2 release:

- [ ] No hardcoded secrets in code or commits
- [ ] All external API calls use TLS/HTTPS
- [ ] Input validation on all user-supplied data
- [ ] Error messages don't leak sensitive info
- [ ] Dependencies audited (`npm audit`)
- [ ] Logging doesn't capture PII
- [ ] API keys rotated
- [ ] Database credentials strong and unique
- [ ] Database encrypted at rest (Phase 2)
- [ ] Access control enforced (Phase 2)
- [ ] Audit logs enabled and tested

---

**End SECURITY.md**