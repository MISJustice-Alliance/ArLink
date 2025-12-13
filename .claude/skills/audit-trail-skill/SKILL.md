---
name: audit-trail-skill
version: 1.0.0
triggers:
  - "create audit trail"
  - "merkle tree"
  - "tamper detection"
dependencies: []
tools:
  - Read
  - Write
  - Bash(npm:*)
---

# Skill: Audit Trail Management (Phase 3)

## Objective
Create tamper-proof audit trails using Merkle trees and blockchain anchoring.

## Prerequisites
- Audit events exist (file modifications, attestations, etc.)
- WITNET_ENDPOINT configured for anchoring

## Workflow

### Phase 1: Event Collection
1. Gather all audit events for time period
2. Serialize to canonical JSON format
3. Sort by timestamp

### Phase 2: Merkle Tree Construction
1. Hash each event with SHA-256
2. Build Merkle tree bottom-up
3. Compute root hash
4. Store tree structure

### Phase 3: Blockchain Anchoring
1. Submit Merkle root to Witnet
2. Wait for finalization
3. Relay to L1 chains (Ethereum, Polygon, Avalanche)
4. Store anchor proof

### Phase 4: Audit Trail Storage
1. Save Merkle tree to local database
2. Save anchor proof with blockchain tx IDs
3. Generate audit trail manifest

### Phase 5: Verification
1. Recompute Merkle root from events
2. Verify against anchored root
3. Check blockchain confirmations
4. Return verification status

## Error Handling
- **Event hash collision**: CRITICAL ERROR (should never happen)
- **Anchoring failure**: Retry with backoff
- **Verification mismatch**: Alert user, investigate tampering

## Security
- Use canonical JSON serialization
- Never modify events after hashing
- Store Merkle proofs securely
- Verify blockchain confirmations

## Success Criteria
- Merkle tree constructed correctly
- Root anchored on blockchain
- Verification succeeds
- Audit trail immutable

## Version History
- 1.0.0: Initial release
