---
name: crypto-hashing-skill
version: 1.0.0
triggers:
  - "hash document"
  - "compute sha256"
  - "verify hash"
dependencies: []
tools:
  - Read
---

# Skill: Cryptographic Hashing

## Objective
Compute deterministic SHA-256 hashes for document integrity verification.

## Prerequisites
- Document content available as Buffer

## Workflow

### Phase 1: Hash Computation
1. Read document content into Buffer
2. Create SHA-256 hash instance
3. Update with content
4. Digest to hexadecimal string

### Phase 2: Determinism Verification
1. Recompute hash from same content
2. Compare hashes (must match)
3. If mismatch: CRITICAL ERROR (hash function broken)

### Phase 3: Metadata Hashing
1. Canonicalize metadata JSON (sorted keys, no whitespace)
2. Compute SHA-256 of canonical form
3. Verify determinism

### Phase 4: Document ID Assembly
1. Concatenate file hash + metadata hash
2. Compute SHA-256 of combined string
3. Return as document ID (final identifier)

### Phase 5: Verification (Optional)
1. Given document ID and original content
2. Recompute file hash
3. Recompute metadata hash (if metadata provided)
4. Recompute document ID
5. Compare: matches → VERIFIED, mismatch → FAILED

## Implementation

### Hash Function
```typescript
function hashDocument(content: Buffer): string {
  return crypto
    .createHash("sha256")
    .update(content)
    .digest("hex");
}
```

### Metadata Hashing
```typescript
function hashMetadata(analysis: AnalysisResult): string {
  const canonical = JSON.stringify(analysis, Object.keys(analysis).sort());
  return crypto
    .createHash("sha256")
    .update(canonical)
    .digest("hex");
}
```

### Document ID Assembly
```typescript
function assembleDocumentId(docHash: string, metaHash: string): string {
  const combined = docHash + metaHash;
  return crypto
    .createHash("sha256")
    .update(combined)
    .digest("hex");
}
```

### Verification
```typescript
async function verifyDocumentHash(
  documentId: string,
  content: Buffer,
  metadata?: AnalysisResult
): Promise<boolean> {
  const recomputedDocHash = hashDocument(content);

  if (metadata) {
    const recomputedMetaHash = hashMetadata(metadata);
    const recomputedId = assembleDocumentId(recomputedDocHash, recomputedMetaHash);
    return recomputedId === documentId;
  }

  // If no metadata, just verify file hash matches first part
  return documentId.startsWith(recomputedDocHash);
}
```

## Error Handling
- **Buffer creation fails**: Check file exists and is readable
- **Hash mismatch on verification**: Document has been modified
- **Invalid hex format**: Hash function error (should never happen)

## Performance Targets
- Hash 1GB file in <500ms
- Hash 100MB file in <50ms
- Hash 1MB file in <5ms

## Security Guarantees
- SHA-256 provides 256-bit security
- Collision probability: 2^256 (negligible)
- Deterministic: Same input always produces same hash
- One-way: Cannot reverse hash to original content

## Success Criteria
- Hash computed successfully
- Determinism verified (recompute matches)
- Performance targets met
- Verification function works correctly

## Example Usage
```bash
# User request: "Hash this contract document"
Agent loads: crypto-hashing-skill
Agent executes: Hash computation → Determinism check → Return hash
Agent returns: SHA-256 hash (64 hex characters)
```

## Version History
- 1.0.0: Initial release with SHA-256 implementation
