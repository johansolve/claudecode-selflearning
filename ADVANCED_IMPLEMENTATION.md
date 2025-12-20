# Avancerad implementation: Self-Learning System

Detta dokument beskriver hur man bygger ut MVP:n till ett fullständigt system med semantic search, automatisk triggning och MCP-integration.

**Förutsättning:** Du har redan använt MVP (`/learn`-kommandot) och samlat 50+ learnings.

**När behövs detta?**
- Du vill ha semantic search istället för manuell navigation
- Du vill ha automatisk triggning vid SessionEnd
- Du behöver hantera tusentals learnings
- Du vill ha usage tracking och confidence scores

---

## Arkitektur

```
┌────────────────────────────────────────────────────────────────┐
│                    Claude Code Environment                      │
│                                                                 │
│  ┌──────────────┐    ┌─────────────┐    ┌──────────────────┐  │
│  │   User +     │───▶│ Git Changes │───▶│  SessionEnd      │  │
│  │   Claude     │    │  + Context  │    │     Hook         │  │
│  │ Interaction  │    │             │    │                  │  │
│  └──────────────┘    └─────────────┘    └────────┬─────────┘  │
└──────────────────────────────────────────────────┼────────────┘
                                                   │
                      ┌────────────────────────────┘
                      │
                      ▼
      ┌───────────────────────────────────────────────┐
      │      Analysis Engine (Python)                 │
      │  ┌─────────────────────────────────────────┐  │
      │  │  1. Significance Detector               │  │
      │  │     - File change analysis              │  │
      │  │     - Commit message parsing            │  │
      │  │                                         │  │
      │  │  2. Deep Analyzer (4-stage AI)          │  │
      │  │     - WHAT (summarize changes)          │  │
      │  │     - WHY (reasoning)                   │  │
      │  │     - PATTERNS (extract)                │  │
      │  │     - KNOWLEDGE (structure)             │  │
      │  │                                         │  │
      │  │  3. Knowledge Extractor                 │  │
      │  │     - Generate embeddings               │  │
      │  │     - Structure data                    │  │
      │  └─────────────────────────────────────────┘  │
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
│  • Full-text index    │◀─│  • Auto-deployment      │
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
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│   Claude Code         │
│   (Next Session)      │
└───────────────────────┘
```

---

## Systemkrav

- Python 3.9+
- Node.js 18+
- ~500MB diskutrymme
- ANTHROPIC_API_KEY

---

## Fas 1: Core infrastructure

### 1.1 Projektstruktur

```bash
mkdir -p ~/.claude/learning-system/{analysis-engine,mcp-server,vectors,logs}
```

### 1.2 Python environment

```bash
cd ~/.claude/learning-system/analysis-engine

python3 -m venv venv
source venv/bin/activate

cat > requirements.txt << 'EOF'
anthropic>=0.40.0
sentence-transformers>=3.0.0
faiss-cpu>=1.8.0
numpy>=1.26.0
EOF

pip install -r requirements.txt
```

### 1.3 SQLite-databas

**Schema:** `~/.claude/learning-system/knowledge.db`

```sql
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
    metadata TEXT,
    created_at INTEGER DEFAULT (strftime('%s', 'now')),
    updated_at INTEGER DEFAULT (strftime('%s', 'now')),
    usage_count INTEGER DEFAULT 0,
    last_used INTEGER,
    UNIQUE(title, project_path)
);

CREATE TABLE code_examples (
    id TEXT PRIMARY KEY,
    knowledge_id TEXT NOT NULL,
    language TEXT NOT NULL,
    code TEXT NOT NULL,
    context TEXT,
    FOREIGN KEY(knowledge_id) REFERENCES knowledge_entries(id) ON DELETE CASCADE
);

CREATE TABLE tags (
    knowledge_id TEXT NOT NULL,
    tag TEXT NOT NULL,
    PRIMARY KEY(knowledge_id, tag),
    FOREIGN KEY(knowledge_id) REFERENCES knowledge_entries(id) ON DELETE CASCADE
);

-- Full-text search
CREATE VIRTUAL TABLE knowledge_fts USING fts5(
    title, summary, detailed_explanation, problem_solved,
    content=knowledge_entries, content_rowid=rowid
);

-- Indexes
CREATE INDEX idx_type ON knowledge_entries(type);
CREATE INDEX idx_project ON knowledge_entries(project_path);
CREATE INDEX idx_confidence ON knowledge_entries(confidence);
CREATE INDEX idx_usage ON knowledge_entries(usage_count DESC);
CREATE INDEX idx_tags_tag ON tags(tag);
```

### 1.4 Vector store

**Fil:** `~/.claude/learning-system/analysis-engine/vector_store.py`

```python
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np
from pathlib import Path
import pickle

class VectorStore:
    def __init__(self, storage_path, dimension=384):
        self.storage_path = Path(storage_path)
        self.dimension = dimension
        self.model = SentenceTransformer('all-MiniLM-L6-v2')
        self.index = faiss.IndexFlatIP(dimension)
        self.id_map = []
        self.load()

    def add_knowledge(self, knowledge_id: str, text: str):
        embedding = self.model.encode(text, normalize_embeddings=True)
        self.index.add(np.array([embedding], dtype='float32'))
        self.id_map.append(knowledge_id)

    def search(self, query: str, k: int = 10):
        if self.index.ntotal == 0:
            return []
        query_embedding = self.model.encode(query, normalize_embeddings=True)
        distances, indices = self.index.search(
            np.array([query_embedding], dtype='float32'),
            min(k, self.index.ntotal)
        )
        return [
            {'knowledge_id': self.id_map[idx], 'similarity': float(dist)}
            for dist, idx in zip(distances[0], indices[0])
            if 0 <= idx < len(self.id_map)
        ]

    def save(self):
        self.storage_path.mkdir(parents=True, exist_ok=True)
        faiss.write_index(self.index, str(self.storage_path / 'faiss.index'))
        with open(self.storage_path / 'id_map.pkl', 'wb') as f:
            pickle.dump(self.id_map, f)

    def load(self):
        index_path = self.storage_path / 'faiss.index'
        map_path = self.storage_path / 'id_map.pkl'
        if index_path.exists() and map_path.exists():
            self.index = faiss.read_index(str(index_path))
            with open(map_path, 'rb') as f:
                self.id_map = pickle.load(f)
```

### 1.5 Konfiguration

**Fil:** `~/.claude/learning-system/config.json`

```json
{
  "analysis": {
    "auto_trigger": true,
    "significance_threshold": 5,
    "min_files_changed": 5,
    "ignored_paths": ["node_modules", ".git", "vendor", "dist"]
  },
  "ai": {
    "model": "claude-sonnet-4-20250514",
    "api_key_env": "ANTHROPIC_API_KEY",
    "max_tokens": 4096
  },
  "knowledge": {
    "min_confidence_for_skill": 4,
    "skill_generation_threshold": 10,
    "retention_days": 365
  },
  "search": {
    "default_limit": 10,
    "min_similarity": 0.6,
    "vector_weight": 0.6,
    "fts_weight": 0.3
  }
}
```

---

## Fas 2: MCP Server

### 2.1 Setup

```bash
cd ~/.claude/learning-system/mcp-server

cat > package.json << 'EOF'
{
  "name": "claude-learning-system-mcp",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/index.js",
  "scripts": { "build": "tsc" },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "sqlite3": "^5.1.7"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0"
  }
}
EOF

npm install
mkdir src
```

### 2.2 Implementation

**Fil:** `~/.claude/learning-system/mcp-server/src/index.ts`

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import sqlite3 from "sqlite3";
import { promisify } from "util";
import { join } from "path";
import { homedir } from "os";

const DB_PATH = join(homedir(), ".claude", "learning-system", "knowledge.db");

class Database {
  private db: sqlite3.Database;
  constructor() { this.db = new sqlite3.Database(DB_PATH); }
  async all(sql: string, params: any[] = []): Promise<any[]> {
    return promisify(this.db.all.bind(this.db))(sql, params);
  }
  async get(sql: string, params: any[] = []): Promise<any> {
    return promisify(this.db.get.bind(this.db))(sql, params);
  }
  async run(sql: string, params: any[] = []): Promise<void> {
    return promisify(this.db.run.bind(this.db))(sql, params);
  }
}

const db = new Database();

const server = new Server({
  name: "claude-learning-system",
  version: "1.0.0",
}, { capabilities: { tools: {} } });

server.setRequestHandler("tools/list", async () => ({
  tools: [
    {
      name: "learn_search",
      description: "Search the knowledge base",
      inputSchema: {
        type: "object",
        properties: {
          query: { type: "string" },
          type: { type: "string", enum: ["code_pattern", "bug_fix", "architecture", "project_rule"] },
          min_confidence: { type: "integer", default: 3 },
          limit: { type: "integer", default: 10 }
        },
        required: ["query"]
      }
    },
    {
      name: "learn_get",
      description: "Get full details of a knowledge entry",
      inputSchema: {
        type: "object",
        properties: { id: { type: "string" } },
        required: ["id"]
      }
    },
    {
      name: "learn_status",
      description: "Get knowledge base statistics",
      inputSchema: { type: "object", properties: {} }
    }
  ]
}));

server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;

  let result;
  if (name === "learn_search") {
    const { query, type, min_confidence = 3, limit = 10 } = args as any;
    result = await db.all(`
      SELECT ke.*, GROUP_CONCAT(t.tag) as tags
      FROM knowledge_entries ke
      LEFT JOIN tags t ON t.knowledge_id = ke.id
      WHERE (? IS NULL OR ke.type = ?) AND ke.confidence >= ?
      GROUP BY ke.id
      ORDER BY ke.usage_count DESC
      LIMIT ?
    `, [type, type, min_confidence, limit]);
  } else if (name === "learn_get") {
    const { id } = args as any;
    result = await db.get("SELECT * FROM knowledge_entries WHERE id = ?", [id]);
  } else if (name === "learn_status") {
    result = await db.get(`
      SELECT COUNT(*) as total,
        COUNT(CASE WHEN type = 'code_pattern' THEN 1 END) as patterns,
        COUNT(CASE WHEN type = 'bug_fix' THEN 1 END) as bug_fixes,
        AVG(confidence) as avg_confidence
      FROM knowledge_entries
    `);
  }

  return { content: [{ type: "text", text: JSON.stringify(result, null, 2) }] };
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

### 2.3 Bygg och konfigurera

```bash
cd ~/.claude/learning-system/mcp-server
npm run build
```

Lägg till i `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "claude-learning-system": {
      "command": "node",
      "args": ["~/.claude/learning-system/mcp-server/dist/index.js"]
    }
  }
}
```

---

## Fas 3: Analysis pipeline

### 3.1 Deep Analyzer

4-stegs AI-analys: WHAT → WHY → PATTERNS → KNOWLEDGE

**Fil:** `~/.claude/learning-system/analysis-engine/deep_analyzer.py`

```python
import anthropic
import json
import os

class DeepAnalyzer:
    def __init__(self):
        self.client = anthropic.Anthropic()
        self.model = "claude-sonnet-4-20250514"

    def analyze(self, session_data):
        # Stage 1: WHAT
        what = self._call(f"""Analyze these changes:
Git Diff: {session_data['git_diff'][:8000]}
Commits: {session_data['commit_messages']}

Output JSON: {{summary, files, change_type}}""")

        # Stage 2: WHY
        why = self._call(f"""Why were these decisions made?
Changes: {what}

Output JSON: {{problem, trade_offs, alternatives, principles}}""")

        # Stage 3: PATTERNS
        patterns = self._call(f"""Extract reusable patterns:
Context: {what}\n{why}

Output JSON: {{patterns, best_practices, anti_patterns}}""")

        # Stage 4: KNOWLEDGE
        knowledge = self._call(f"""Create knowledge entries:
Analysis: {patterns}

For each pattern, output JSON array with:
title, category, summary, detailed_explanation, when_to_apply,
when_not_to_apply, confidence (1-5), tags""")

        return json.loads(knowledge)

    def _call(self, prompt):
        msg = self.client.messages.create(
            model=self.model, max_tokens=4096, temperature=0.3,
            messages=[{"role": "user", "content": prompt}]
        )
        return msg.content[0].text
```

### 3.2 Significance detector

Avgör om ändringar är värda att analysera (sparar API-kostnader).

```python
class SignificanceDetector:
    KEYWORDS = ['refactor', 'fix', 'bug', 'optimize', 'security']

    def score(self, git_diff, commit_messages):
        score = 0
        files = git_diff.count('diff --git')
        lines = git_diff.count('\n+') + git_diff.count('\n-')

        if files >= 5: score += 3
        if lines >= 150: score += 2
        if any(kw in ' '.join(commit_messages).lower() for kw in self.KEYWORDS):
            score += 2
        return score  # Threshold default: 5
```

---

## Fas 4: Skill generator (valfritt)

När du har 50+ patterns med hög confidence kan systemet auto-generera skills.

**Trigger-kriterier:**
- ≥5 patterns med samma primära tag
- Total usage ≥10
- Average confidence ≥4

**Output:** Skapar `~/.claude/skills/learned-{tag}/` med SKILL.md och reference.md, samma format som MVP.

---

## Hybrid search (ranking)

```
Total Score = (Vector Similarity × 0.6) + (FTS Rank × 0.3) + (Confidence × 0.05) + (Usage × 0.05)
```

---

## Performance

| Metric | Target |
|--------|--------|
| Search latency | <100ms |
| Analysis time | 30-60s per session |
| Storage per entry | ~3KB |
| 10,000 entries | ~35MB |

**API-kostnader:** ~$0.10-0.50 per session-analys

---

## Migration från MVP

Dina befintliga MVP-skills kan importeras:

```python
# Läs befintliga learned-*/reference.md
# Parsera entries
# Generera embeddings
# Spara i SQLite + FAISS
```

---

## Troubleshooting

**API Key Error:**
```bash
export ANTHROPIC_API_KEY="your-key"
```

**MCP Server startar inte:**
```bash
cd ~/.claude/learning-system/mcp-server && npm run build
```

**Database locked:**
```bash
lsof ~/.claude/learning-system/knowledge.db
```

---

## Sammanfattning

| Komponent | Syfte |
|-----------|-------|
| SQLite + FTS5 | Strukturerad lagring, full-text search |
| FAISS + sentence-transformers | Semantic vector search |
| MCP Server | Claude Code-integration |
| Analysis Engine | 4-stegs AI-extraktion |
| Skill Generator | Auto-generera skills vid threshold |

Börja med MVP. Bygg ut hit när du har 50+ learnings och behöver mer.
