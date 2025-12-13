---
name: builder-role-skill
version: 1.0.0
triggers:
  - "implement feature"
  - "write code for"
  - "build functionality"
dependencies:
  - validator-role-skill (recommended)
tools:
  - Read
  - Edit
  - Write
  - Bash(git:*)
  - Bash(npm:*)
  - Test
---

# Skill: Builder Role (TDD Implementation)

## Objective
Implement features following Test-Driven Development (TDD) and ArweaveStamp coding standards.

## Prerequisites
- Feature requirements documented
- Module boundaries understood (from AGENTS.md)
- TypeScript configured with strict mode

## Workflow

### Phase 1: Planning
1. Read and analyze requirements
2. Identify files to modify/create
3. Determine module placement (storage/, analysis/, integrity/, etc.)
4. Create implementation checklist

### Phase 2: Test-Driven Development

#### Red Phase (Write Failing Tests)
1. Create test file: `tests/unit/{module}.test.ts`
2. Write test cases for expected behavior
3. Run tests: `npm test` (should FAIL)
4. Verify failures are expected

#### Green Phase (Minimal Implementation)
1. Create/edit implementation file: `src/{module}/{file}.ts`
2. Write minimal code to pass tests
3. Run tests: `npm test` (should PASS)
4. Verify all tests passing

#### Refactor Phase (Improve Code Quality)
1. Improve code structure while keeping tests green
2. Add type annotations (TypeScript strict mode)
3. Extract reusable functions
4. Add error handling
5. Run tests after each refactor (must stay green)

### Phase 3: Integration

1. Import new code in appropriate modules
2. Update exports in index.ts files
3. Run full test suite: `npm test`
4. Check coverage: `npm run test:coverage`

### Phase 4: Validation

1. Lint code: `npm run lint`
2. Type check: `npm run type-check`
3. Build verification: `npm run build`
4. Verify coverage meets thresholds

### Phase 5: Documentation

1. Add JSDoc comments to public functions
2. Update README.md if public API changed
3. Update ARCHITECTURE.md if module structure changed
4. Add usage examples

### Phase 6: Git Workflow

1. Stage changes: `git add .`
2. Write descriptive commit message:
   ```
   feat(module): add feature description

   - Implemented X functionality
   - Added Y test coverage
   - Updated Z documentation
   ```
3. Commit: `git commit -m "..."`
4. Push to feature branch: `git push -u origin feature/name`

## Coding Standards

### TypeScript Strict Mode
```typescript
// ✓ Correct
interface ProofPackage {
  id: string;
  timestamp: ISO8601;
}

function generate(data: ProofPackage): string {
  return data.id;
}

// ✗ Incorrect
const proof: any = {};
function generate(data) { return data.id; }
```

### Error Handling
```typescript
// ✓ Correct
try {
  const result = await uploadToArweave(file);
  return result;
} catch (error: any) {
  if (error.status === 413) {
    throw new FileTooLargeError("Max 100MB");
  }
  throw new ArweaveError(error.message);
}

// ✗ Incorrect
try {
  return await uploadToArweave(file);
} catch (error) {
  console.log(error); // Silent failure
}
```

### Async/Await Pattern
```typescript
// ✓ Correct
async function processFile(path: string): Promise<ProofPackage> {
  const content = await fs.promises.readFile(path);
  const hash = hashDocument(content);
  return { hash, ...metadata };
}

// ✗ Incorrect (callback hell)
function processFile(path: string, callback) {
  fs.readFile(path, (err, data) => {
    if (err) callback(err);
    else callback(null, hashDocument(data));
  });
}
```

## TDD Example

### Step 1: Write Failing Test
```typescript
// tests/unit/integrity.test.ts
describe("hashDocument", () => {
  test("should return same hash for identical content", () => {
    const content = Buffer.from("test data");
    const hash1 = hashDocument(content);
    const hash2 = hashDocument(content);
    expect(hash1).toBe(hash2);
  });
});

// Run: npm test → FAILS (hashDocument not implemented)
```

### Step 2: Minimal Implementation
```typescript
// src/integrity/hasher.ts
import crypto from "crypto";

export function hashDocument(content: Buffer): string {
  return crypto
    .createHash("sha256")
    .update(content)
    .digest("hex");
}

// Run: npm test → PASSES
```

### Step 3: Refactor
```typescript
// src/integrity/hasher.ts
import crypto from "crypto";

/**
 * Compute SHA-256 hash of document content
 * @param content - Document content as Buffer
 * @returns Hexadecimal hash string (64 characters)
 */
export function hashDocument(content: Buffer): string {
  if (!Buffer.isBuffer(content)) {
    throw new TypeError("Content must be a Buffer");
  }

  return crypto
    .createHash("sha256")
    .update(content)
    .digest("hex");
}

// Run: npm test → PASSES (still green)
```

## Success Criteria
- All tests written before implementation
- Tests pass (green)
- Code coverage meets thresholds
- TypeScript strict mode compliant
- Linting passes
- Documentation complete
- Git commit with good message

## Example Usage
```bash
# User request: "Implement the Arweave upload functionality"
Agent loads: builder-role-skill
Agent executes:
  1. Write tests for upload function
  2. Implement minimal upload function
  3. Refactor with error handling
  4. Document function
  5. Commit changes
Agent returns: Feature implemented with tests
```

## Version History
- 1.0.0: Initial release with TDD workflow
