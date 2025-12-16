# Technical Specification: Claude Code Self-Learning System

## System Architecture

### High-Level Components

```
┌────────────────────────────────────────────────────────────────┐
│                    Claude Code Environment                      │
│                                                                 │
│  ┌──────────────┐    ┌─────────────┐    ┌──────────────────┐ │
│  │   User +     │───▶│ Git Changes │───▶│  SessionEnd      │ │
│  │   Claude     │    │  + Context  │    │     Hook         │ │
│  │ Interaction  │    │             │    │                  │ │
│  └──────────────┘    └─────────────┘    └────────┬─────────┘ │
└──────────────────────────────────────────────────┼────────────┘
                                                     │
                        ┌────────────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────────────┐
        │      Analysis Engine (Python)                 │
        │  ┌─────────────────────────────────────────┐ │
        │  │  1. Significance Detector               │ │
        │  │     - File change analysis              │ │
        │  │     - Commit message parsing            │ │
        │  │     - Pattern detection                 │ │
        │  │                                         │ │
        │  │  2. Deep Analyzer (4-stage AI)          │ │
        │  │     - Stage 1: WHAT (summarize changes) │ │
        │  │     - Stage 2: WHY (reasoning)          │ │
        │  │     - Stage 3: PATTERNS (extract)       │ │
        │  │     - Stage 4: KNOWLEDGE (structure)    │ │
        │  │                                         │ │
        │  │  3. Knowledge Extractor                 │ │
        │  │     - Parse AI output                   │ │
        │  │     - Structure data                    │ │
        │  │     - Generate embeddings               │ │
        │  └─────────────────────────────────────────┘ │
        └───────────────┬───────────────────────────────┘
                        │
            ┌───────────┴──────────┐
            │                      │
            ▼                      ▼
┌───────────────────────┐  ┌─────────────────────────┐
│   Knowledge Database  │  │   Skill Generator       │
│   (SQLite + FAISS)    │  │   (Python)              │
│                       │  │                         │
│  • Structured data    │  │  • Pattern clustering   │
│  • Vector embeddings  │  │  • Template generation  │
│  • Full-text index    │  │  • Skill compilation    │
│  • Relationships      │◀─│  • Auto-deployment      │
└───────────┬───────────┘  └─────────────────────────┘
            │
            │ MCP Protocol
            ▼
┌───────────────────────┐
│    MCP Server         │
│    (TypeScript)       │
│                       │
│  • learn_search       │
│  • learn_get          │
│  • learn_status       │
│  • learn_trigger      │
└───────────┬───────────┘
            │
            │ Tool Calls
            ▼
┌───────────────────────┐
│   Claude Code         │
│   (Next Session)      │
│                       │
│  • MCP tools active   │
│  • Skills loaded      │
│  • Proactive learning │
└───────────────────────┘
```

## Component Specifications

### 1. Analysis Engine (Python)

#### 1.1 Significance Detector

**Purpose:** Determine if session changes warrant deep AI analysis.

**Input:**
- `git_diff`: Complete diff since session start
- `commit_messages`: List of commit messages
- `session_context`: Session history from ~/.claude/history.jsonl

**Scoring Algorithm:**
```python
def calculate_significance(git_diff, commit_messages, session_context):
    score = 0

    # File count
    files = parse_git_diff(git_diff)
    if len(files) >= 5:
        score += 3

    # Line count
    lines_changed = sum(f.additions + f.deletions for f in files)
    if lines_changed >= 150:
        score += 2

    # New patterns detected
    if detect_new_patterns(files):
        score += 5

    # Keywords in commits
    keywords = ['refactor', 'fix', 'bug', 'optimize', 'improve', 'security']
    if any(kw in msg.lower() for msg in commit_messages for kw in keywords):
        score += 2

    # Architectural changes
    if detect_architectural_changes(files):
        score += 4

    # User corrections/iterations
    if session_context.get('user_corrections', 0) >= 2:
        score += 1

    return score

# Usage
if calculate_significance(data) >= threshold:
    trigger_deep_analysis()
else:
    skip_analysis()  # Save API costs
```

**Thresholds:**
- Default: 5
- Configurable per project
- Manual trigger bypasses threshold

#### 1.2 Deep Analyzer (4-Stage Pipeline)

**Purpose:** Extract deep understanding of WHY decisions were made.

**Stage 1: WHAT Analysis**

Prompt Template:
```
Analyze these code changes and describe WHAT was done:

Git Diff:
{git_diff}

Commit Messages:
{commit_messages}

Provide:
1. Summary of changes (2-3 sentences)
2. Files affected and their roles
3. Type of change (feature, bug fix, refactoring, etc.)
```

Output Schema:
```json
{
  "summary": "string",
  "files": [
    {"path": "string", "role": "string", "change_type": "added|modified|deleted"}
  ],
  "change_type": "feature|bug_fix|refactoring|optimization|documentation"
}
```

**Stage 2: WHY Analysis**

Prompt Template:
```
Now analyze WHY these decisions were made:

Changes: {stage1_output}
Session Context: {conversation_highlights}
User Corrections: {user_corrections}

Extract:
1. Problem that was being solved
2. Trade-offs considered
3. Why this approach over alternatives
4. Architectural principles applied
```

Output Schema:
```json
{
  "problem": "string",
  "trade_offs": [
    {"consideration": "string", "decision": "string", "reasoning": "string"}
  ],
  "alternatives_considered": ["string"],
  "principles": ["string"]
}
```

**Stage 3: PATTERNS Extraction**

Prompt Template:
```
Extract reusable patterns and principles:

Full Context: {stage1_output + stage2_output}

Identify:
1. Code patterns that could be reused
2. Architectural principles demonstrated
3. Best practices shown
4. Anti-patterns avoided and why
5. Project-specific conventions followed
```

Output Schema:
```json
{
  "patterns": [
    {
      "name": "string",
      "category": "code_pattern|architecture|convention",
      "description": "string",
      "applicability": "string"
    }
  ],
  "best_practices": ["string"],
  "anti_patterns_avoided": [
    {"pattern": "string", "why_avoided": "string"}
  ]
}
```

**Stage 4: KNOWLEDGE Structuring**

Prompt Template:
```
Create knowledge base entries:

Analysis: {all_previous_stages}

For each significant pattern/insight, generate:
1. Title (5-10 words, descriptive)
2. Category (code_pattern | bug_fix | architecture | project_rule)
3. Summary (2-3 sentences)
4. Detailed explanation with code examples
5. When to apply (conditions)
6. When NOT to apply (anti-conditions)
7. Related patterns or principles
8. Confidence level (1-5, based on evidence quality)
9. Tags for search (technology, domain, pattern-type)

Output as structured JSON array.
```

Output Schema: See "Knowledge Entry Format" below.

#### 1.3 Knowledge Extractor

**Purpose:** Parse AI output and prepare for database storage.

**Responsibilities:**
- Validate JSON structure from Stage 4
- Generate UUIDs for entries
- Extract code examples into separate objects
- Generate vector embeddings
- Link related patterns
- Calculate confidence scores

**Code Example:**
```python
class KnowledgeExtractor:
    def extract(self, ai_output, session_data):
        entries = []
        raw_entries = json.loads(ai_output)

        for raw in raw_entries:
            entry = {
                'id': str(uuid4()),
                'timestamp': int(time.time()),
                'session_id': session_data['session_id'],
                'type': raw['category'],
                'title': raw['title'],
                'summary': raw['summary'],
                'detailed_explanation': raw['detailed_explanation'],
                'problem_solved': raw.get('problem', ''),
                'when_to_apply': raw['when_to_apply'],
                'when_not_to_apply': raw['when_not_to_apply'],
                'confidence': raw['confidence'],
                'project_path': session_data['project_path'],
                'tags': raw['tags'],
                'code_examples': self._extract_code_examples(raw),
                'trade_offs': raw.get('trade_offs', []),
                'related_patterns': self._find_related(raw),
                'metadata': {
                    'lines_changed': session_data['lines_changed'],
                    'commits': session_data['commits'],
                    'user_corrections': session_data.get('user_corrections', 0)
                }
            }
            entries.append(entry)

        return entries
```

#### 1.4 Vector Store

**Technology:** FAISS + sentence-transformers

**Model:** `all-MiniLM-L6-v2`
- 384 dimensions
- Multilingual (Swedish + English)
- Fast inference (~10ms per encoding)
- Model size: 90MB

**Implementation:**
```python
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np

class VectorStore:
    def __init__(self, dimension=384):
        self.model = SentenceTransformer('all-MiniLM-L6-v2')
        self.dimension = dimension
        self.index = faiss.IndexFlatIP(dimension)  # Inner Product for cosine sim
        self.id_map = []

    def add_knowledge(self, knowledge_id, text):
        """
        text = title + ' ' + summary + ' ' + detailed_explanation
        """
        embedding = self.model.encode(text, normalize_embeddings=True)
        self.index.add(np.array([embedding], dtype='float32'))
        self.id_map.append(knowledge_id)

    def search(self, query, k=10):
        query_embedding = self.model.encode(query, normalize_embeddings=True)
        distances, indices = self.index.search(
            np.array([query_embedding], dtype='float32'),
            k
        )

        results = []
        for distance, idx in zip(distances[0], indices[0]):
            if idx < len(self.id_map):
                results.append({
                    'knowledge_id': self.id_map[idx],
                    'similarity': float(distance)  # Cosine similarity [0-1]
                })
        return results
```

### 2. Knowledge Database

#### 2.1 SQLite Schema

```sql
-- Main knowledge entries table
CREATE TABLE knowledge_entries (
    id TEXT PRIMARY KEY,
    timestamp INTEGER NOT NULL,
    session_id TEXT,
    type TEXT CHECK(type IN ('code_pattern', 'bug_fix', 'architecture', 'project_rule')) NOT NULL,
    title TEXT NOT NULL,
    summary TEXT NOT NULL,
    detailed_explanation TEXT NOT NULL,
    problem_solved TEXT,
    when_to_apply TEXT,
    when_not_to_apply TEXT,
    confidence INTEGER CHECK(confidence BETWEEN 1 AND 5) NOT NULL,
    project_path TEXT,
    metadata TEXT,  -- JSON blob
    created_at INTEGER DEFAULT (strftime('%s', 'now')),
    updated_at INTEGER DEFAULT (strftime('%s', 'now')),
    usage_count INTEGER DEFAULT 0,
    last_used INTEGER,
    UNIQUE(title, project_path)
);

-- Code examples (1-to-many)
CREATE TABLE code_examples (
    id TEXT PRIMARY KEY,
    knowledge_id TEXT NOT NULL,
    language TEXT NOT NULL,
    code TEXT NOT NULL,
    context TEXT,
    FOREIGN KEY(knowledge_id) REFERENCES knowledge_entries(id) ON DELETE CASCADE
);

-- Tags (many-to-many)
CREATE TABLE tags (
    knowledge_id TEXT NOT NULL,
    tag TEXT NOT NULL,
    PRIMARY KEY(knowledge_id, tag),
    FOREIGN KEY(knowledge_id) REFERENCES knowledge_entries(id) ON DELETE CASCADE
);

-- Relations between entries (many-to-many)
CREATE TABLE relations (
    from_id TEXT NOT NULL,
    to_id TEXT NOT NULL,
    relation_type TEXT CHECK(relation_type IN ('related', 'alternative', 'prerequisite', 'supersedes')) NOT NULL,
    PRIMARY KEY(from_id, to_id),
    FOREIGN KEY(from_id) REFERENCES knowledge_entries(id) ON DELETE CASCADE,
    FOREIGN KEY(to_id) REFERENCES knowledge_entries(id) ON DELETE CASCADE
);

-- Trade-offs (1-to-many)
CREATE TABLE trade_offs (
    id TEXT PRIMARY KEY,
    knowledge_id TEXT NOT NULL,
    type TEXT CHECK(type IN ('pro', 'con')) NOT NULL,
    description TEXT NOT NULL,
    FOREIGN KEY(knowledge_id) REFERENCES knowledge_entries(id) ON DELETE CASCADE
);

-- Indexes for performance
CREATE INDEX idx_type ON knowledge_entries(type);
CREATE INDEX idx_project ON knowledge_entries(project_path);
CREATE INDEX idx_confidence ON knowledge_entries(confidence);
CREATE INDEX idx_usage ON knowledge_entries(usage_count DESC);
CREATE INDEX idx_tags_tag ON tags(tag);
CREATE INDEX idx_timestamp ON knowledge_entries(timestamp DESC);

-- Full-text search (FTS5)
CREATE VIRTUAL TABLE knowledge_fts USING fts5(
    title,
    summary,
    detailed_explanation,
    problem_solved,
    content=knowledge_entries,
    content_rowid=rowid
);

-- Triggers to keep FTS in sync
CREATE TRIGGER knowledge_fts_insert AFTER INSERT ON knowledge_entries BEGIN
    INSERT INTO knowledge_fts(rowid, title, summary, detailed_explanation, problem_solved)
    VALUES (new.rowid, new.title, new.summary, new.detailed_explanation, new.problem_solved);
END;

CREATE TRIGGER knowledge_fts_update AFTER UPDATE ON knowledge_entries BEGIN
    UPDATE knowledge_fts
    SET title = new.title,
        summary = new.summary,
        detailed_explanation = new.detailed_explanation,
        problem_solved = new.problem_solved
    WHERE rowid = new.rowid;
END;

CREATE TRIGGER knowledge_fts_delete AFTER DELETE ON knowledge_entries BEGIN
    DELETE FROM knowledge_fts WHERE rowid = old.rowid;
END;
```

#### 2.2 Knowledge Entry Format

```typescript
interface KnowledgeEntry {
  id: string;  // UUID v4
  timestamp: number;  // Unix timestamp
  session_id: string;
  type: 'code_pattern' | 'bug_fix' | 'architecture' | 'project_rule';
  title: string;  // 5-10 words, descriptive
  summary: string;  // 2-3 sentences
  detailed_explanation: string;  // Markdown format
  problem_solved: string;
  when_to_apply: string;
  when_not_to_apply: string;
  confidence: 1 | 2 | 3 | 4 | 5;
  project_path: string;
  metadata: {
    lines_changed: number;
    commits: string[];
    user_corrections: number;
    [key: string]: any;
  };
  created_at: number;
  updated_at: number;
  usage_count: number;
  last_used?: number;

  // Related data (joined)
  code_examples: CodeExample[];
  tags: string[];
  relations: Relation[];
  trade_offs: TradeOff[];
}

interface CodeExample {
  id: string;
  knowledge_id: string;
  language: string;
  code: string;
  context: string;
}

interface Relation {
  from_id: string;
  to_id: string;
  relation_type: 'related' | 'alternative' | 'prerequisite' | 'supersedes';
}

interface TradeOff {
  id: string;
  knowledge_id: string;
  type: 'pro' | 'con';
  description: string;
}
```

### 3. MCP Server

#### 3.1 Server Configuration

**File:** `~/.claude/learning-system/mcp-server/index.ts`

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new Server({
  name: "claude-learning-system",
  version: "1.0.0",
}, {
  capabilities: {
    tools: {},
  },
});

// Register tools
server.setRequestHandler('tools/list', async () => {
  return {
    tools: [
      {
        name: "learn_search",
        description: "Search the knowledge base for patterns, bug fixes, and learnings",
        inputSchema: {
          type: "object",
          properties: {
            query: { type: "string", description: "Search query" },
            type: {
              type: "string",
              enum: ["code_pattern", "bug_fix", "architecture", "project_rule"],
              description: "Filter by knowledge type"
            },
            project: { type: "string", description: "Filter by project path" },
            min_confidence: { type: "integer", default: 3, minimum: 1, maximum: 5 },
            limit: { type: "integer", default: 10, minimum: 1, maximum: 50 }
          },
          required: ["query"]
        }
      },
      {
        name: "learn_get",
        description: "Get full details of a specific knowledge entry",
        inputSchema: {
          type: "object",
          properties: {
            id: { type: "string", description: "Knowledge entry ID" }
          },
          required: ["id"]
        }
      },
      {
        name: "learn_status",
        description: "Get statistics about learned knowledge",
        inputSchema: {
          type: "object",
          properties: {
            project: { type: "string", description: "Filter by project" }
          }
        }
      },
      {
        name: "learn_trigger",
        description: "Manually trigger learning analysis",
        inputSchema: {
          type: "object",
          properties: {
            scope: {
              type: "string",
              enum: ["session", "commit", "files"],
              default: "session"
            },
            target: { type: "string", description: "Commit hash or file paths" }
          }
        }
      }
    ]
  };
});
```

#### 3.2 Tool Implementations

**learn_search Implementation:**

```typescript
async function learnSearch(args: {
  query: string;
  type?: string;
  project?: string;
  min_confidence?: number;
  limit?: number;
}) {
  const {
    query,
    type = null,
    project = null,
    min_confidence = 3,
    limit = 10
  } = args;

  // 1. Vector semantic search
  const vectorResults = await vectorStore.search(query, limit * 2);

  // 2. Full-text search
  const ftsQuery = `
    SELECT ke.*, GROUP_CONCAT(t.tag) as tags
    FROM knowledge_fts fts
    JOIN knowledge_entries ke ON ke.rowid = fts.rowid
    LEFT JOIN tags t ON t.knowledge_id = ke.id
    WHERE knowledge_fts MATCH ?
      AND (? IS NULL OR ke.type = ?)
      AND (? IS NULL OR ke.project_path LIKE ?)
      AND ke.confidence >= ?
    GROUP BY ke.id
    LIMIT ?
  `;

  const ftsResults = await db.all(ftsQuery, [
    query,
    type, type,
    project, `%${project}%`,
    min_confidence,
    limit
  ]);

  // 3. Merge and rank
  const merged = mergeAndRank(vectorResults, ftsResults);

  // 4. Update usage stats
  for (const result of merged.slice(0, 5)) {
    await db.run(`
      UPDATE knowledge_entries
      SET usage_count = usage_count + 1,
          last_used = strftime('%s', 'now')
      WHERE id = ?
    `, [result.id]);
  }

  return {
    content: [{
      type: "text",
      text: JSON.stringify({
        results: merged.slice(0, limit),
        query,
        total: merged.length
      }, null, 2)
    }]
  };
}
```

**Ranking Algorithm:**

```typescript
function mergeAndRank(
  vectorResults: Array<{knowledge_id: string, similarity: number}>,
  ftsResults: Array<any>
): Array<any> {
  const scores = new Map();

  // Vector scores (60% weight)
  vectorResults.forEach((result, index) => {
    const rankScore = (vectorResults.length - index) / vectorResults.length;
    scores.set(result.knowledge_id, {
      vector_score: rankScore * result.similarity * 0.6,
      fts_score: 0,
      meta: result
    });
  });

  // FTS scores (30% weight)
  ftsResults.forEach((result, index) => {
    const rankScore = (ftsResults.length - index) / ftsResults.length;
    const existing = scores.get(result.id);

    if (existing) {
      existing.fts_score = rankScore * 0.3;
      existing.meta = { ...existing.meta, ...result };
    } else {
      scores.set(result.id, {
        vector_score: 0,
        fts_score: rankScore * 0.3,
        meta: result
      });
    }
  });

  // Add bonus factors (10% total)
  scores.forEach((scoreData, id) => {
    const meta = scoreData.meta;

    // Confidence bonus (5%)
    const confidenceBonus = (meta.confidence / 5) * 0.05;

    // Usage bonus (5%)
    const usageBonus = Math.min(meta.usage_count / 100, 0.05);

    scoreData.total_score =
      scoreData.vector_score +
      scoreData.fts_score +
      confidenceBonus +
      usageBonus;
  });

  // Sort by total score
  return Array.from(scores.entries())
    .map(([id, data]) => ({ ...data.meta, relevance_score: data.total_score }))
    .sort((a, b) => b.relevance_score - a.relevance_score);
}
```

### 4. Skill Generator

#### 4.1 Pattern Clustering

**Purpose:** Identify groups of related patterns that warrant a skill.

**SQL Query:**
```sql
SELECT
  t.tag,
  COUNT(DISTINCT ke.id) as pattern_count,
  SUM(ke.usage_count) as total_usage,
  AVG(ke.confidence) as avg_confidence,
  GROUP_CONCAT(ke.id) as pattern_ids,
  ke.project_path
FROM tags t
JOIN knowledge_entries ke ON ke.id = t.knowledge_id
WHERE ke.type = 'code_pattern'
  AND ke.confidence >= 4
GROUP BY t.tag, ke.project_path
HAVING pattern_count >= 5 AND total_usage >= 10
ORDER BY total_usage DESC
```

**Threshold Logic:**
```python
def should_generate_skill(cluster):
    return (
        cluster['pattern_count'] >= 5 and
        cluster['total_usage'] >= 10 and
        cluster['avg_confidence'] >= 4.0
    )
```

#### 4.2 Skill Template

**SKILL.md Format:**
```markdown
---
name: learned-{tag}
description: Auto-generated skill for {tag} patterns based on {pattern_count} learned implementations (used {total_usage} times)
---

You have deep expertise in {tag}-related patterns learned from real-world implementations in this codebase.

## Expertise Summary

This skill was automatically generated from {pattern_count} patterns with an average confidence of {avg_confidence}/5.

Key areas:
{bullet_list_of_top_patterns}

## Usage Guidelines

Refer to reference.md for:
- Detailed pattern explanations
- Code examples from actual implementations
- When to apply each pattern
- Trade-offs and considerations

## Context

These patterns were extracted from {project_path} over {time_span}, demonstrating proven solutions that have been successfully applied {total_usage} times.
```

**reference.md Format:**
```markdown
# {Tag} Patterns Reference

This document contains {pattern_count} proven patterns for {tag} development.

## Table of Contents

{auto_generated_toc}

---

## Pattern 1: {title}

**Confidence:** {confidence}/5 | **Used:** {usage_count} times | **Type:** {type}

### Problem
{problem_solved}

### Solution
{detailed_explanation}

### When to Apply
{when_to_apply}

### When NOT to Apply
{when_not_to_apply}

### Trade-offs

**Pros:**
{list_of_pros}

**Cons:**
{list_of_cons}

### Code Examples

#### Example 1: {context}
```{language}
{code}
```

### Related Patterns
- {related_pattern_links}

---

{repeat for all patterns}
```

## Configuration Schema

**File:** `~/.claude/learning-system/config.json`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "analysis": {
      "type": "object",
      "properties": {
        "auto_trigger": { "type": "boolean", "default": true },
        "significance_threshold": { "type": "integer", "default": 5, "minimum": 0 },
        "min_files_changed": { "type": "integer", "default": 5 },
        "min_lines_changed": { "type": "integer", "default": 150 },
        "ignored_paths": {
          "type": "array",
          "items": { "type": "string" },
          "default": ["node_modules", ".git", "vendor", "*.lock"]
        }
      }
    },
    "ai": {
      "type": "object",
      "properties": {
        "model": { "type": "string", "default": "claude-sonnet-4.5" },
        "api_key_env": { "type": "string", "default": "ANTHROPIC_API_KEY" },
        "max_tokens": { "type": "integer", "default": 4096 },
        "temperature": { "type": "number", "default": 0.3, "minimum": 0, "maximum": 1 }
      }
    },
    "knowledge": {
      "type": "object",
      "properties": {
        "min_confidence_for_skill": { "type": "integer", "default": 4, "minimum": 1, "maximum": 5 },
        "skill_generation_threshold": { "type": "integer", "default": 10 },
        "max_entries_per_session": { "type": "integer", "default": 20 },
        "retention_days": { "type": "integer", "default": 365 }
      }
    },
    "search": {
      "type": "object",
      "properties": {
        "default_limit": { "type": "integer", "default": 10 },
        "min_similarity": { "type": "number", "default": 0.6 },
        "vector_weight": { "type": "number", "default": 0.6 },
        "fts_weight": { "type": "number", "default": 0.3 },
        "confidence_bonus": { "type": "number", "default": 0.05 },
        "usage_bonus": { "type": "number", "default": 0.05 }
      }
    },
    "skills": {
      "type": "object",
      "properties": {
        "auto_generate": { "type": "boolean", "default": true },
        "min_patterns": { "type": "integer", "default": 5 },
        "min_total_usage": { "type": "integer", "default": 10 },
        "output_dir": { "type": "string", "default": "~/.claude/skills/" }
      }
    }
  }
}
```

## Performance Specifications

### Latency Targets

- **learn_search**: < 100ms (90th percentile)
- **learn_get**: < 50ms
- **learn_status**: < 200ms
- **Analysis pipeline**: 30-60 seconds per session

### Scalability Limits

| Metric | Target | Maximum |
|--------|--------|---------|
| Knowledge entries | 10,000 | 100,000 |
| Vector index size | 35MB | 350MB |
| Database size | 50MB | 500MB |
| Search results | 10 | 50 |
| Concurrent searches | 10 | 50 |

### Memory Usage

- **Idle**: ~50MB (MCP server)
- **Model loaded**: ~250MB (sentence-transformers)
- **During analysis**: ~500MB (peak)
- **FAISS index**: In-memory (~3.5KB per entry)

### API Costs

- **Per analysis**: $0.10-0.50 (depends on change size)
- **Estimated monthly**: $5-20 (active usage)
- **Token usage**: ~5k-20k tokens per session

## Security Considerations

### Data Privacy
- All data stored locally in `~/.claude/learning-system/`
- No external services except Claude API for analysis
- API key managed via environment variable

### Access Control
- SQLite database permissions: 600 (user only)
- MCP server runs as user process
- No network exposure

### Data Sanitization
- Git-ignored files excluded from analysis
- Sensitive patterns (API keys, passwords) filtered
- Code examples sanitized before storage

## Error Handling

### Analysis Engine
- Network errors: Retry with exponential backoff (3 attempts)
- Parse errors: Log and skip entry, continue pipeline
- Database errors: Transaction rollback, alert user

### MCP Server
- Query errors: Return error message in tool response
- Database lock: Wait and retry (5 attempts)
- Missing entries: Return empty results, not error

### Skill Generator
- Template errors: Skip skill, log error
- File write errors: Alert user, continue process

## Monitoring & Logging

### Log Locations
- Analysis engine: `~/.claude/learning-system/logs/analysis.log`
- MCP server: `~/.claude/learning-system/logs/mcp-server.log`
- Skill generator: `~/.claude/learning-system/logs/skills.log`

### Metrics to Track
- Knowledge entries per day
- Search query frequency
- Skill generation events
- API costs per session
- Analysis success/failure rate
