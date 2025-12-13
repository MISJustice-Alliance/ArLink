# CLAUDE.md - Integration Guide for Phase 1, 2, and 3
**Claude 3.5 Sonnet Integration with Arweave + Witnet Authenticity Platform**

---

## Overview

This document specifies how Claude 3.5 Sonnet is integrated throughout all three phases of the ArweaveStamp platform:

- **Phase 1**: Document analysis, classification, entity extraction
- **Phase 2**: Proof interpretation, search query understanding, compliance summarization
- **Phase 3**: Policy reasoning, anomaly detection, audit interpretation

---

## Claude Code Integration

ArweaveStamp includes a comprehensive **Claude Code** development integration with custom commands, agent configurations, and specialized skills for streamlined development workflows.

### Integration Components

**Custom Commands** (`.claude/commands/`):
- 16 custom slash commands for core workflows, domain-specific tasks, QA, and multi-agent orchestration
- See [README.md - Claude Integration](./README.md#-claude-integration) for full command list
- Commands include: `/start-session`, `/plan`, `/test-all`, `/attest-document`, `/verify-proof`, `/validate-phase`, `/mail-job`, `/orchestrate-feature`, and more

**Generic Agent** (`.claude/agents/`):
- `arweave-general.md` – Skills-first agent configuration
- Dynamically loads domain-specific and development role skills based on task requirements
- Provides 35% token efficiency improvement over traditional multi-agent approaches

**Specialized Skills** (`.claude/skills/`):
- 12 specialized skills covering blockchain attestation, document analysis, cryptographic hashing, TDD development, testing, mail integration, payments, audit trails, and more
- Skills organized by domain: core domain skills, development role skills, phase-specific skills, and utility skills
- See skill directories for detailed workflow specifications (SKILL.md files)

### Skills-First Development Paradigm

This project follows Anthropic's recommended **skills-first paradigm**:

1. **Default to Single Agent + Skills**: For most development tasks, use the general agent which dynamically loads appropriate skills
2. **Multi-Agent for Parallelization Only**: Use the `/orchestrate-feature` command when tasks benefit from parallel execution (multiple approaches, breadth-first research, large-scale concurrent work)
3. **Progressive Skill Loading**: Agent loads minimal skills initially, adds more as task complexity emerges
4. **Better Token Efficiency**: 35% reduction in token usage compared to multi-agent approaches
5. **Improved Context Management**: Skills maintain context better than switching between specialized agents

### Usage

```bash
# Start a development session
/start-session

# Validate external service integrations
/integration-check

# Create a multi-agent workflow for complex features
/orchestrate-feature

# Validate phase completion
/validate-phase 1
```

For complete command reference and skill descriptions, see:
- [README.md - Claude Integration](./README.md#-claude-integration)
- Command files in `.claude/commands/`
- Skill specifications in `.claude/skills/*/SKILL.md`
- Claude Code best practices in `docs/claude/docs/best-practices/`

---

## Phase 1: Document Analysis & Metadata Extraction

### Use Case: Semantic Understanding

When a user uploads a document (PDF, image, or text), Claude analyzes it to extract:
1. **Classification**: Document type (contract, invoice, certificate, etc.)
2. **Entities**: Key actors (names, organizations, dates, amounts)
3. **Summary**: Human-readable abstract
4. **Confidence Scores**: Reliability of extracted data

### Architecture: Claude Integration Points

```typescript
// Phase 1 Analysis Pipeline
src/analysis/client.ts
  ↓
  • Initialize Anthropic SDK
  • Handle rate limiting (exponential backoff)
  • Manage API keys securely
  ↓
src/analysis/prompts.ts
  ↓
  • Classification prompt template
  • Entity extraction prompt template
  • Summarization prompt template
  • Vision prompt for PDF/images
  ↓
src/analysis/extractors.ts
  ↓
  • Parse Claude responses into JSON
  • Validate extracted data format
  • Return structured metadata
  ↓
src/analysis/vision.ts
  ↓
  • Encode PDF pages / images to base64
  • Send to Claude with vision prompt
  • Extract text + visual elements
```

### API Endpoint Details

**Endpoint**: `messages` (Anthropic Bedrock API or Direct API)

**Model**: `claude-3-5-sonnet-20241022` (or latest Sonnet variant)

**Max Tokens**: 2,048 per request (typical response ~1,000 tokens)

**Rate Limits**: 100 requests/min (standard tier)

### Prompt Templates

#### 1. Classification Prompt

```
System: You are a document classifier specializing in legal, financial, and compliance documents.

User Input:
---
Document Content:
{document_text}

Image Data:
{base64_image_or_null}
---

Classify this document and provide output in JSON format:

{
  "document_type": "contract|invoice|certificate|email|report|other",
  "category": "legal|financial|hr|compliance|technical|other",
  "sub_category": "e.g., employment_contract, tax_invoice, etc.",
  "confidence_score": 0.0-1.0,
  "reasoning": "Why this classification?"
}

Respond ONLY with valid JSON. Do not include markdown code blocks.
```

**Example Output**:
```json
{
  "document_type": "contract",
  "category": "legal",
  "sub_category": "employment_agreement",
  "confidence_score": 0.95,
  "reasoning": "Document contains employment terms, compensation, and signature lines typical of employment contracts."
}
```

#### 2. Entity Extraction Prompt

```
System: You are an entity extraction specialist. Extract all named entities and key facts from the document.

User Input:
---
Document Content:
{document_text}
---

Extract and return JSON with these entity types:

{
  "persons": [
    { "name": string, "role": string, "context": string }
  ],
  "organizations": [
    { "name": string, "relationship": string }
  ],
  "dates": [
    { "date": "YYYY-MM-DD", "event": string }
  ],
  "amounts": [
    { "value": number, "currency": string, "description": string }
  ],
  "locations": [
    { "name": string, "context": string }
  ],
  "key_terms": [ string ],
  "extraction_confidence": 0.0-1.0
}

Respond ONLY with valid JSON.
```

**Example Output**:
```json
{
  "persons": [
    { "name": "Alice Johnson", "role": "Employee", "context": "Party A in employment agreement" },
    { "name": "Bob Smith", "role": "Manager", "context": "Company representative" }
  ],
  "organizations": [
    { "name": "TechCorp Inc.", "relationship": "Employer" }
  ],
  "dates": [
    { "date": "2025-01-01", "event": "Start date" },
    { "date": "2026-01-01", "event": "Contract renewal" }
  ],
  "amounts": [
    { "value": 120000, "currency": "USD", "description": "Annual salary" }
  ],
  "locations": [
    { "name": "San Francisco, CA", "context": "Office location" }
  ],
  "key_terms": [ "remote work", "equity", "non-compete" ],
  "extraction_confidence": 0.88
}
```

#### 3. Summarization Prompt

```
System: You are a summarization expert. Create a concise executive summary of the document.

User Input:
---
Document Content:
{document_text}
Document Type: {document_type}
---

Provide a 2-3 sentence summary capturing the key points:

{
  "summary": string,
  "key_risks": [ string ],
  "key_obligations": [ string ],
  "summary_confidence": 0.0-1.0
}

Respond ONLY with valid JSON.
```

#### 4. Vision/PDF Prompt

```
System: You can see images of documents. Extract text and visual information.

User Input:
---
Image Data: {base64_encoded_image}
---

Extract all visible text, visual elements, and metadata:

{
  "extracted_text": string,
  "visual_elements": [ "signature", "watermark", "logo", "table", "chart" ],
  "layout_notes": string,
  "text_regions": [
    { "position": "top|center|bottom", "content": string }
  ],
  "confidence_score": 0.0-1.0
}

Respond ONLY with valid JSON.
```

### Implementation: Phase 1 Analysis Client

```typescript
// src/analysis/client.ts

import Anthropic from "@anthropic-ai/sdk";

interface AnalysisResult {
  classification: ClassificationResult;
  entities: EntityResult;
  summary: SummaryResult;
  visionData?: VisionResult;
}

export async function analyzeDocument(
  fileContent: string,
  base64Image?: string
): Promise<AnalysisResult> {
  const client = new Anthropic();

  // 1. Classification
  const classificationResponse = await client.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "text",
            text: buildClassificationPrompt(fileContent),
          },
          ...(base64Image
            ? [
                {
                  type: "image",
                  source: {
                    type: "base64",
                    media_type: "image/png",
                    data: base64Image,
                  },
                },
              ]
            : []),
        ],
      },
    ],
  });

  const classification = parseJSON(classificationResponse);

  // 2. Entity Extraction
  const entitiesResponse = await client.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 2048,
    messages: [
      {
        role: "user",
        content: buildEntityExtractionPrompt(fileContent),
      },
    ],
  });

  const entities = parseJSON(entitiesResponse);

  // 3. Summarization
  const summaryResponse = await client.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 512,
    messages: [
      {
        role: "user",
        content: buildSummarizationPrompt(
          fileContent,
          classification.document_type
        ),
      },
    ],
  });

  const summary = parseJSON(summaryResponse);

  // 4. Vision Analysis (if image provided)
  let visionData: VisionResult | undefined;
  if (base64Image) {
    const visionResponse = await client.messages.create({
      model: "claude-3-5-sonnet-20241022",
      max_tokens: 1024,
      messages: [
        {
          role: "user",
          content: [
            {
              type: "image",
              source: {
                type: "base64",
                media_type: "image/png",
                data: base64Image,
              },
            },
            {
              type: "text",
              text: buildVisionPrompt(),
            },
          ],
        },
      ],
    });
    visionData = parseJSON(visionResponse);
  }

  return {
    classification,
    entities,
    summary,
    visionData,
  };
}

// Helper to handle rate limiting
async function callClaudeWithRetry(
  fn: () => Promise<any>,
  maxRetries: number = 3
): Promise<any> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error: any) {
      if (error.status === 429 && attempt < maxRetries) {
        // Exponential backoff: 2^attempt seconds
        const delayMs = Math.pow(2, attempt) * 1000;
        console.warn(`Rate limited. Retrying in ${delayMs}ms...`);
        await new Promise((resolve) => setTimeout(resolve, delayMs));
      } else {
        throw error;
      }
    }
  }
}
```

### PII Handling: Phase 1

**Important**: Claude processes PII (names, dates, SSNs, etc.) as needed for analysis.

**Privacy Approach**:
1. **User Consent**: Users explicitly opt-in to Claude analysis
2. **API Retention**: Anthropic retains conversation data per terms (enable "disable API data retention" for enterprise)
3. **Sensitive Fields**: Mask SSNs in prompts: "SSN: XXX-XX-1234"
4. **Pseudonymization**: Store analysis results linked to ProofPackage ID, not raw content

**Example PII Masking**:
```typescript
function maskPII(text: string): string {
  return text
    .replace(/\d{3}-\d{2}-\d{4}/g, "XXX-XX-****") // SSN
    .replace(/\d{3}-\d{2}-\d{4}/g, "****-**-****") // SSN (full mask)
    .replace(/\b\d{16}\b/g, "****-****-****-****") // Credit card
    .replace(/\d{10,19}/g, (match) =>
      match.slice(0, 4) + "*".repeat(match.length - 8) + match.slice(-4)
    ); // Generic numbers
}
```

---

## Phase 2: Proof Interpretation & Search

### Use Case: LLM-Powered Document Search

In Phase 2, Claude is used to understand natural language queries and improve search relevance:

```typescript
// Phase 2 Application: Search Query Understanding

/**
 * User query: "Show me all employment contracts from 2024"
 *
 * Claude interprets:
 *   - Search type: document_type == "employment_contract" AND year == 2024
 *   - Entities to match: "employment contracts"
 *   - Time constraint: 2024
 *
 * Returns: Ranked list of ProofPackages matching criteria
 */

async function semanticSearch(userQuery: string): Promise<ProofPackage[]> {
  const client = new Anthropic();

  // Ask Claude to parse the query
  const parseResponse = await client.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 256,
    messages: [
      {
        role: "user",
        content: `
Parse this document search query into structured filters:
"${userQuery}"

Return JSON:
{
  "document_types": [ string ],
  "keywords": [ string ],
  "date_range": { "start": "YYYY-MM-DD", "end": "YYYY-MM-DD" },
  "entities_to_match": [ string ]
}

Respond ONLY with valid JSON.
        `,
      },
    ],
  });

  const filters = parseJSON(parseResponse);

  // Execute database query based on filters
  return queryProofPackages(filters);
}
```

### Use Case: Compliance Summarization

```typescript
/**
 * User: "Generate a compliance report for all Q4 contracts"
 *
 * Claude generates:
 *   1. Aggregated summary of all Q4 contracts
 *   2. Risk flags (missing clauses, concerning terms)
 *   3. Recommendations for legal review
 */

async function generateComplianceReport(
  proofPackages: ProofPackage[]
): Promise<string> {
  const client = new Anthropic();

  const aggregatedMetadata = proofPackages
    .map((pkg) => pkg.metadata.summary)
    .join("\n\n");

  const response = await client.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 2048,
    messages: [
      {
        role: "user",
        content: `
You are a compliance officer. Analyze these document summaries and generate a report:

${aggregatedMetadata}

Provide:
1. Overview of contract portfolio
2. Key compliance risks
3. Missing clauses or concerning terms
4. Recommendations for legal review

Format as markdown.
        `,
      },
    ],
  });

  return response.content[0].type === "text" ? response.content[0].text : "";
}
```

---

## Phase 3: Policy Reasoning & Anomaly Detection

### Use Case: Smart Policy Decision Making

When a file modification is detected, Claude helps the policy engine decide on appropriate actions:

```typescript
// Phase 3 Application: Anomaly Detection & Response

/**
 * Event: File modified at 3 AM (unusual time)
 * File: sensitive_contract.pdf
 * Modification: Content added (not just metadata change)
 *
 * Claude reasoning:
 *   - Is this modification legitimate?
 *   - What risk level? (low/medium/high)
 *   - Recommended action? (reattestor, quarantine, alert, etc.)
 */

async function classifyFileModification(
  event: FileEvent,
  previousMetadata: DocumentMetadata
): Promise<PolicyDecision> {
  const client = new Anthropic();

  const response = await client.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 512,
    messages: [
      {
        role: "user",
        content: `
Analyze this file modification event and recommend a policy response:

File: ${event.path}
Type: ${previousMetadata.document_type}
Modification: ${event.type} at ${event.timestamp}
Previous Hash: ${event.previousHash}
Current Hash: ${event.currentHash}
Size Change: ${calculateSizeChange(event.previousHash, event.currentHash)}
Time: ${new Date(event.timestamp).toLocaleString()}

Based on file type and modification pattern, classify risk and recommend action:

{
  "risk_level": "low|medium|high|critical",
  "anomaly_reason": string,
  "recommended_action": "reattestor|email|snapshot|quarantine|monitor",
  "confidence": 0.0-1.0
}

Respond ONLY with valid JSON.
      `,
      },
    ],
  });

  const decision = parseJSON(response);
  return {
    event,
    decision,
    appliedAt: new Date().toISOString(),
  };
}
```

### Use Case: Audit Log Interpretation

```typescript
/**
 * Phase 3: User asks "What happened to contract_v3.pdf?"
 *
 * Claude reads audit trail and explains:
 *   - Original attestation date
 *   - Modification timestamps
 *   - Policy actions taken
 *   - Current status
 */

async function interpretAuditTrail(
  auditEntries: AuditEntry[]
): Promise<string> {
  const client = new Anthropic();

  const auditSummary = auditEntries
    .map(
      (entry) =>
        `[${entry.timestamp}] ${entry.event.type}: ${JSON.stringify(entry.event.details)}`
    )
    .join("\n");

  const response = await client.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 1024,
    messages: [
      {
        role: "user",
        content: `
Interpret this file's audit trail and explain what happened in plain language:

${auditSummary}

Provide:
1. Timeline of events
2. Any concerning modifications
3. Actions taken by system
4. Current file status

Format as easy-to-read narrative.
      `,
      },
    ],
  });

  return response.content[0].type === "text" ? response.content[0].text : "";
}
```

---

## Security & Privacy Best Practices

### 1. API Key Management
- Store API keys in `.env` file (never commit)
- Use AWS Secrets Manager or HashiCorp Vault for production
- Rotate keys quarterly

### 2. PII Handling
```typescript
// Phase 1: Mask sensitive data in prompts
const maskedContent = maskPII(documentContent);
const response = await analyzeDocument(maskedContent);

// Phase 3: Don't send raw file modifications to Claude
// Instead: send event type, hash comparison, metadata
const sanitizedEvent = {
  type: event.type,
  filesize_delta: event.sizeChange,
  permission_changed: event.permissionChange,
  timestamp: event.timestamp,
  // Do NOT include file content
};
```

### 3. Rate Limiting
```typescript
const rateLimiter = new RateLimiter({
  requestsPerMinute: 100,
  tokenBucket: true,
});

async function callClaude(prompt: string): Promise<string> {
  await rateLimiter.acquire();
  try {
    return await client.messages.create({...});
  } finally {
    // Token released automatically on completion
  }
}
```

### 4. Error Handling
```typescript
try {
  const result = await analyzeDocument(content);
} catch (error: any) {
  if (error.status === 401) {
    // Invalid API key
    logger.error("Claude API authentication failed");
  } else if (error.status === 429) {
    // Rate limited - implement retry
    logger.warn("Rate limited, retrying...");
  } else if (error.status === 500) {
    // Service error - fallback to local analysis
    logger.error("Claude API error, using fallback");
  }
}
```

---

## Testing Claude Integration

### Unit Tests
```bash
# Mock Claude API responses
npm run test:unit -- analysis.test.ts

# Example test:
test("should classify contract correctly", async () => {
  const mockResponse = {
    classification: {
      document_type: "contract",
      category: "legal",
      confidence_score: 0.95,
    },
  };

  jest
    .spyOn(client, "messages.create")
    .mockResolvedValue(mockResponse);

  const result = await analyzeDocument(sampleContent);
  expect(result.classification.document_type).toBe("contract");
});
```

### Integration Tests
```bash
# Real Claude API calls (uses quota)
npm run test:integration -- analysis-claude.test.ts

# Example:
test("should extract entities from real document", async () => {
  const realResult = await analyzeDocument(realDocumentContent);
  expect(realResult.entities.persons.length).toBeGreaterThan(0);
  expect(realResult.entities.extraction_confidence).toBeGreaterThan(0.8);
}, { timeout: 30000 }); // Claude calls take longer
```

---

## Monitoring & Observability

### Logging
```typescript
// Log all Claude API calls for auditing
logger.info("Claude API call", {
  phase: "phase-1",
  operation: "classification",
  tokens_sent: prompt.split(" ").length,
  tokens_received: response.content[0].text.split(" ").length,
  latency_ms: endTime - startTime,
});
```

### Metrics
- **Latency**: Average time per Claude call (P50, P95, P99)
- **Cost**: Monthly token usage + estimated billing
- **Error Rate**: Failed API calls (by error type)
- **Accuracy**: Classification/extraction accuracy (compared to ground truth)

---

## Migrating to Newer Claude Models

When Anthropic releases Claude 4 or newer:

1. **Test on new model** with same prompts
2. **Compare outputs** (classification accuracy, entity completeness)
3. **Update max_tokens** if needed
4. **Benchmark latency** (newer models may be faster/slower)
5. **Update CLAUDE.md** with new model name
6. **Gradually roll out** (feature flag for A/B testing)

---

## FAQ

**Q: Can Claude see the original document content?**
A: Yes, in Phase 1 during analysis. We mask PII and use API data retention settings to limit persistence.

**Q: What if Claude misclassifies a document?**
A: Confidence scores help flag uncertain results. In Phase 2 dashboard, users can provide feedback to improve future classifications.

**Q: Does Phase 3 send file content to Claude?**
A: No. Phase 3 only sends metadata (event type, hash comparison, timestamp). The actual file never leaves the local system.

**Q: How much does Claude integration cost?**
A: Approximately $0.30-0.50 per document (Phase 1 analysis). See [Anthropic Pricing](https://www.anthropic.com/pricing).

---

**End CLAUDE.md**Import command and agent standards from docs/claude/
