# CONTRIBUTING.md: Contributing to ArweaveStamp

**Version**: 1.0  
**Last Updated**: December 11, 2025

Thank you for your interest in contributing to ArweaveStamp! This document outlines our development process, code standards, and how to get involved.

---

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Workflow](#development-workflow)
- [Coding Standards](#coding-standards)
- [Testing Requirements](#testing-requirements)
- [Pull Request Process](#pull-request-process)
- [Commit Message Format](#commit-message-format)
- [Issue Templates](#issue-templates)
- [Documentation](#documentation)

---

## Code of Conduct

### Our Pledge

We are committed to providing a welcoming and inclusive environment for all contributors, regardless of age, body size, disability, ethnicity, sex characteristics, gender identity and expression, level of experience, education, socio-economic status, nationality, personal appearance, race, religion, or sexual identity and orientation.

### Expected Behavior

- Use welcoming and inclusive language
- Be respectful of differing opinions and experiences
- Accept constructive criticism gracefully
- Focus on what is best for the community
- Show empathy towards other community members

### Unacceptable Behavior

- Harassment or discrimination of any kind
- Insulting/derogatory comments
- Personal or political attacks
- Public or private harassment
- Publishing private information without consent

**Enforcement**: Violations may result in removal from the project.

---

## Getting Started

### Prerequisites

- **Node.js**: 18.x or later
- **npm**: 9.x or later (or yarn/pnpm)
- **Git**: Latest version
- **GitHub Account**: For pull requests

### Initial Setup

```bash
# 1. Fork repository on GitHub
# 2. Clone your fork
git clone https://github.com/your-username/arweavestamp.git
cd arweavestamp

# 3. Add upstream remote
git remote add upstream https://github.com/yourorg/arweavestamp.git

# 4. Install dependencies
npm install

# 5. Create feature branch
git checkout -b feature/your-feature-name

# 6. Set up environment variables
cp .env.example .env
# Edit .env with your API keys (test/sandbox only)

# 7. Verify setup
npm run build
npm run test
npm run lint
```

### Development Environment

**Recommended Tools**:
- **Editor**: VSCode with TypeScript support
- **Extensions**: ESLint, Prettier, Thunder Client
- **Terminal**: Zsh or Bash with git aliases
- **Version Manager**: nvm for Node version management

**VSCode Settings** (`.vscode/settings.json`):
```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "eslint.validate": ["typescript", "typescriptreact"],
  "typescript.tsdk": "node_modules/typescript/lib"
}
```

---

## Development Workflow

### Branch Strategy

```
main (production) ← release branches ← develop ← feature/bugfix branches
```

### Creating a Feature Branch

```bash
# Update develop branch
git checkout develop
git pull upstream develop

# Create feature branch from develop
git checkout -b feature/your-feature-name

# Naming convention:
# feature/phase1-arweave-upload-123
# bugfix/claude-rate-limit-145
# docs/update-readme-456
```

### Making Changes

```bash
# 1. Make your changes
# 2. Format code
npm run format:fix

# 3. Run linter
npm run lint

# 4. Run tests
npm run test

# 5. Commit with meaningful message
git commit -m "feat(arweave): implement document upload pipeline"

# 6. Keep branch updated
git fetch upstream
git rebase upstream/develop
# OR git merge upstream/develop (if public branch)

# 7. Push to your fork
git push origin feature/your-feature-name
```

---

## Coding Standards

### TypeScript

**Strict Mode**: Always enabled
```typescript
// ✅ GOOD
async function uploadDocument(path: string): Promise<UploadResult> {
  if (!path) throw new ValidationError('Path required');
  const file = await readFile(path);
  return uploadToArweave(file);
}

// ❌ BAD
async function uploadDocument(path: any): any {
  const file = await readFile(path); // No error handling
  return uploadToArweave(file);
}
```

**Type Definitions**:
- All function parameters must have explicit types
- Return types must be specified (no implicit any)
- Use const assertions for readonly objects
- Generic types should be bounded where possible

### Naming Conventions

```typescript
// Constants
const DEFAULT_TIMEOUT = 30000;
const MAX_FILE_SIZE = 100 * 1024 * 1024;

// Variables & Functions
const documentHash = calculateHash(file);
const isValid = validateDocument(file);
async function uploadToArweave(file: FileBuffer): Promise<string>

// Classes & Interfaces
class DocumentProcessor { }
interface IDocumentMetadata { } // Optional I prefix
type DocumentType = 'contract' | 'invoice';

// Booleans
const hasError = true;
const isLoading = false;
const shouldRetry = true;
```

### Code Organization

**File Structure**:
```
src/
├── cli/              # CLI commands
├── core/             # Business logic
├── services/         # External API integration
├── utils/            # Utilities
├── types/            # Shared types
└── index.ts          # Main entry point
```

**Import Organization**:
```typescript
// 1. External dependencies
import Arweave from 'arweave';
import { z } from 'zod';

// 2. Internal absolute imports
import { DocumentProcessor } from '@/core/document/processor';
import { logger } from '@/utils/logger';

// 3. Relative imports
import { DocumentMetadata } from './types';

// 4. Side effects (last)
import './styles.css';
```

---

## Testing Requirements

### Test Coverage

- **Minimum**: 85% unit, 70% integration
- **Target**: 90% unit, 80% integration
- **Critical paths**: 100% coverage required

### Writing Tests

**Unit Test Example**:
```typescript
// src/core/document/__tests__/processor.test.ts
import { DocumentProcessor } from '../processor';
import { ValidationError } from '@/utils/errors';

describe('DocumentProcessor', () => {
  let processor: DocumentProcessor;

  beforeEach(() => {
    processor = new DocumentProcessor();
  });

  describe('validateFile', () => {
    it('should accept valid PDF files', async () => {
      const file = new File(['test'], 'test.pdf', { type: 'application/pdf' });
      const result = await processor.validateFile(file);
      expect(result.isValid).toBe(true);
    });

    it('should reject unsupported file types', async () => {
      const file = new File(['test'], 'test.exe', { type: 'application/x-msdownload' });
      await expect(processor.validateFile(file)).rejects.toThrow(ValidationError);
    });

    it('should enforce maximum file size', async () => {
      const large = new ArrayBuffer(101 * 1024 * 1024);
      const file = new File([large], 'large.pdf');
      await expect(processor.validateFile(file)).rejects.toThrow('File exceeds 100MB');
    });
  });
});
```

### Running Tests

```bash
# Run all tests
npm run test

# Run with coverage
npm run test:coverage

# Run in watch mode (development)
npm run test:watch

# Run specific test file
npm run test -- src/core/document.test.ts

# Update snapshots
npm run test -- -u
```

---

## Pull Request Process

### Before Creating PR

- [ ] Branch is updated with latest `develop`
- [ ] Code formatted: `npm run format:fix`
- [ ] Linter passes: `npm run lint`
- [ ] Tests pass: `npm run test`
- [ ] Coverage > 85% unit: `npm run test:coverage`
- [ ] No console.log statements (use logger)
- [ ] No hardcoded secrets or API keys
- [ ] JSDoc comments on public functions
- [ ] Git history is clean (consider squashing if needed)

### Creating PR

1. **Push to your fork**:
```bash
git push origin feature/your-feature-name
```

2. **Create PR on GitHub**:
   - Title: Clear, descriptive, follows convention
   - Description: Comprehensive, references issues
   - Link related issues: "Fixes #123"
   - Select reviewers: 2 code review minimum

3. **PR Template**:
```markdown
## Description
Brief description of changes.

## Type of Change
- [ ] Bug fix (non-breaking)
- [ ] New feature (non-breaking)
- [ ] Breaking change
- [ ] Documentation update

## Related Issues
Fixes #123
Related to #456

## Testing
- [ ] Unit tests added
- [ ] Integration tests added
- [ ] Manual testing completed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated
- [ ] No new warnings generated
- [ ] Tests pass locally
```

### Review Process

- **Code Review**: Check for style, logic, security
- **Automated Tests**: CI/CD pipeline runs all tests
- **Approval**: Requires 2 approvals
- **Merge**: Squash commits, delete branch

---

## Commit Message Format

### Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Rules

1. **Type**: feat, fix, docs, style, refactor, perf, test, chore, ci
2. **Scope**: arweave, claude, stampery, cli, web, core, etc.
3. **Subject**: Imperative mood, present tense, lowercase, no period
4. **Body**: Explain what and why (not how)
5. **Footer**: Reference issues (Fixes #123) or breaking changes

### Examples

```
feat(arweave): implement document upload pipeline
- Add DocumentProcessor class
- Support batch uploads with parallel processing
- Implement retry logic for network failures

Fixes #123

fix(claude): handle image processing errors
- Add try/catch for Vision API timeout
- Implement exponential backoff retry
- Add detailed error logging

Refs #456

docs(readme): update installation instructions
- Add prerequisites section
- Include environment setup
- Add troubleshooting FAQ

BREAKING CHANGE: Removed support for Node 16
```

---

## Issue Templates

### Bug Report

```markdown
## Description
Clear description of the bug.

## Steps to Reproduce
1. Run command: `npm run cli -- stamp file.pdf`
2. Observe output
3. Error occurs

## Expected Behavior
What should happen.

## Actual Behavior
What actually happened.

## Environment
- OS: macOS 14.1
- Node.js: 18.17.0
- npm version: 9.6.0

## Logs
[Paste relevant log output]

## Possible Solution
Optional: suggest a fix.
```

### Feature Request

```markdown
## Feature Description
Clear description of desired feature.

## Use Case
Why this feature would be useful.

## Proposed Solution
How you imagine it working.

## Alternatives
Other approaches considered.

## Additional Context
Links, sketches, additional information.
```

---

## Documentation

### When to Document

- [ ] New public functions/classes
- [ ] Complex algorithms
- [ ] Non-obvious design decisions
- [ ] Setup/configuration changes
- [ ] Breaking changes

### Documentation Standards

**JSDoc Comments**:
```typescript
/**
 * Upload document to Arweave permanent storage.
 * 
 * Accepts various file formats and uploads to Arweave network,
 * generating permanent transaction ID for verification.
 * 
 * @param filePath - Absolute path to document file
 * @param metadata - Optional metadata to attach
 * @param options - Upload configuration options
 * 
 * @returns Promise resolving to upload result with transaction ID
 * 
 * @throws {FileNotFoundError} If file path does not exist
 * @throws {ValidationError} If file fails validation
 * @throws {ArweaveNetworkError} If upload fails
 * 
 * @example
 * ```typescript
 * const result = await uploadToArweave('./document.pdf', {
 *   documentType: 'contract',
 *   author: 'John Doe'
 * });
 * ```
 * 
 * @remarks
 * - Upload typically completes in 5-30 seconds
 * - Requires ~0.005 AR per file
 * - Transaction is immutable once confirmed
 * 
 * @since 1.0.0
 */
async function uploadToArweave(
  filePath: string,
  metadata?: DocumentMetadata,
  options?: UploadOptions
): Promise<UploadResult>
```

### Updating Documentation

- **README.md**: User-facing overview and quick start
- **ARCHITECTURE.md**: System design and component details
- **Code comments**: Explain "why" not "what"
- **CHANGELOG.md**: Track changes per release

---

## Community Guidelines

### Getting Help

- **Questions**: GitHub Discussions
- **Bugs**: GitHub Issues with bug template
- **Features**: GitHub Issues with feature template
- **Security**: See SECURITY.md for vulnerability reporting

### Community Channels

- **Discussions**: https://github.com/yourorg/arweavestamp/discussions
- **Issues**: https://github.com/yourorg/arweavestamp/issues
- **Email**: support@arweavestamp.app

---

## Recognition

### Contributors

We maintain a list of contributors in:
- CONTRIBUTORS.md (Hall of fame)
- GitHub commit history
- Release notes (acknowledgements section)

### Levels of Contribution

- **Code**: Pull requests and commits
- **Documentation**: Guides, examples, translations
- **Community**: Answering questions, reporting issues
- **Testing**: Bug reports, test coverage improvements
- **Design**: UI/UX suggestions, architecture discussions

---

## Additional Resources

- [AGENTS.md](AGENTS.md) - AI coding agent guide
- [CLAUDE.md](CLAUDE.md) - Claude-specific development
- [DEVELOPMENT_PLAN.md](DEVELOPMENT_PLAN.md) - Project roadmap
- [ARCHITECTURE.md](ARCHITECTURE.md) - System design
- [SECURITY.md](SECURITY.md) - Security policies

---

## Summary

1. **Fork** and clone repository
2. **Create** feature branch from `develop`
3. **Code** with TypeScript strict mode
4. **Test** with >85% coverage
5. **Format** with Prettier
6. **Commit** with conventional format
7. **Push** and create pull request
8. **Iterate** on feedback
9. **Merge** after approvals

**Thank you for contributing to ArweaveStamp!** 🚀

---

**Last Updated**: December 11, 2025  
**Maintained by**: ArweaveStamp Community
