---
name: document-analysis-skill
version: 1.0.0
triggers:
  - "analyze document"
  - "extract metadata"
  - "classify document"
dependencies: []
tools:
  - Read
  - Bash(npm:*)
---

# Skill: Document Analysis with Claude

## Objective
Extract semantic metadata from documents using Claude AI (classification, entity extraction, summarization).

## Prerequisites
- CLAUDE_API_KEY configured
- Document content available (text or base64-encoded image)

## Workflow

### Phase 1: Preparation
1. Read document content
2. Determine document type (text, PDF, image)
3. If PDF/image: Convert to base64 encoding
4. Mask PII in content (SSNs, credit cards)

### Phase 2: Classification
1. Send classification prompt to Claude
2. Request: document_type, category, sub_category, confidence_score
3. Parse JSON response
4. Validate confidence_score >0.8

### Phase 3: Entity Extraction
1. Send entity extraction prompt to Claude
2. Request: persons, organizations, dates, amounts, locations, key_terms
3. Parse JSON response
4. Validate extraction_confidence >0.8

### Phase 4: Summarization
1. Send summarization prompt to Claude
2. Request: summary (2-3 sentences), key_risks, key_obligations
3. Parse JSON response

### Phase 5: Vision Analysis (if image/PDF)
1. Send image with vision prompt
2. Extract: extracted_text, visual_elements, layout_notes
3. Parse JSON response

### Phase 6: Result Assembly
1. Combine all analysis results
2. Return structured AnalysisResult:
   - classification
   - entities
   - summary
   - visionData (if applicable)

## Error Handling
- **Rate limit (429)**: Exponential backoff (2^attempt seconds, max 3 retries)
- **Invalid API key (401)**: Alert user to check CLAUDE_API_KEY
- **Malformed JSON response**: Log raw response, request retry
- **Low confidence (<0.8)**: Flag for manual review

## Prompt Templates

### Classification Prompt
```
System: You are a document classifier specializing in legal, financial, and compliance documents.

User Input:
---
Document Content:
{document_text}
---

Classify this document and provide output in JSON format:
{
  "document_type": "contract|invoice|certificate|email|report|other",
  "category": "legal|financial|hr|compliance|technical|other",
  "sub_category": "e.g., employment_contract, tax_invoice, etc.",
  "confidence_score": 0.0-1.0,
  "reasoning": "Why this classification?"
}

Respond ONLY with valid JSON.
```

### Entity Extraction Prompt
```
System: You are an entity extraction specialist.

User Input:
---
Document Content:
{document_text}
---

Extract and return JSON:
{
  "persons": [{ "name": string, "role": string, "context": string }],
  "organizations": [{ "name": string, "relationship": string }],
  "dates": [{ "date": "YYYY-MM-DD", "event": string }],
  "amounts": [{ "value": number, "currency": string, "description": string }],
  "locations": [{ "name": string, "context": string }],
  "key_terms": [string],
  "extraction_confidence": 0.0-1.0
}

Respond ONLY with valid JSON.
```

## PII Masking
```typescript
function maskPII(text: string): string {
  return text
    .replace(/\d{3}-\d{2}-\d{4}/g, "XXX-XX-****")  // SSN
    .replace(/\b\d{16}\b/g, "****-****-****-****")  // Credit card
    .replace(/\d{10,19}/g, (match) =>
      match.slice(0, 4) + "*".repeat(match.length - 8) + match.slice(-4)
    );
}
```

## Success Criteria
- All prompts executed successfully
- Valid JSON responses received
- Confidence scores meet thresholds
- PII masked before sending to Claude
- Structured AnalysisResult returned

## Example Usage
```bash
# User request: "Analyze this employment contract"
Agent loads: document-analysis-skill
Agent executes: Classification → Entity extraction → Summarization
Agent returns: AnalysisResult with structured metadata
```

## Version History
- 1.0.0: Initial release
