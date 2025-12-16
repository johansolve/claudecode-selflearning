# Implementation Guide: Claude Code Self-Learning System

Detta dokument innehåller steg-för-steg instruktioner för att implementera det självlärande systemet.

## Översikt

Implementation är uppdelad i 4 faser:
1. **Core Infrastructure** - Database, vector store, analysis engine
2. **Integration** - MCP server, hooks, slash commands
3. **AI Analysis Pipeline** - Deep analyzer, knowledge extraction
4. **Skill Generator** - Auto-generation av skills

**Estimerad tid:** ~7 dagars arbete

---

## Förberedelser

### Systemkrav

- Python 3.9+
- Node.js 18+
- Git
- ~500MB diskutrymme
- ANTHROPIC_API_KEY

### Installera systempaket

```bash
# macOS
brew install python@3.11 node

# Verifiera
python3 --version
node --version
```

---

## Fas 1: Core Infrastructure

### Steg 1.1: Skapa projektstruktur

```bash
# Skapa huvudmapp
mkdir -p ~/.claude/learning-system

# Skapa undermappar
cd ~/.claude/learning-system
mkdir -p analysis-engine mcp-server skill-generator hooks vectors logs
```

### Steg 1.2: Setup Python Analysis Engine

```bash
cd ~/.claude/learning-system/analysis-engine

# Skapa virtual environment
python3 -m venv venv

# Aktivera
source venv/bin/activate

# Skapa requirements.txt
cat > requirements.txt << 'EOF'
anthropic>=0.40.0
sentence-transformers>=3.0.0
faiss-cpu>=1.8.0
numpy>=1.26.0
EOF

# Installera dependencies
pip install -r requirements.txt
```

### Steg 1.3: Skapa database setup script

**Fil:** `~/.claude/learning-system/setup-db.py`

```python
#!/usr/bin/env python3
import sqlite3
import sys
from pathlib import Path

DB_PATH = Path.home() / ".claude" / "learning-system" / "knowledge.db"

SCHEMA = """
-- Main knowledge entries table
CREATE TABLE IF NOT EXISTS knowledge_entries (
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

CREATE TABLE IF NOT EXISTS code_examples (
    id TEXT PRIMARY KEY,
    knowledge_id TEXT NOT NULL,
    language TEXT NOT NULL,
    code TEXT NOT NULL,
    context TEXT,
    FOREIGN KEY(knowledge_id) REFERENCES knowledge_entries(id) ON DELETE CASCADE
);

CREATE TABLE IF NOT EXISTS tags (
    knowledge_id TEXT NOT NULL,
    tag TEXT NOT NULL,
    PRIMARY KEY(knowledge_id, tag),
    FOREIGN KEY(knowledge_id) REFERENCES knowledge_entries(id) ON DELETE CASCADE
);

CREATE TABLE IF NOT EXISTS relations (
    from_id TEXT NOT NULL,
    to_id TEXT NOT NULL,
    relation_type TEXT CHECK(relation_type IN ('related', 'alternative', 'prerequisite', 'supersedes')) NOT NULL,
    PRIMARY KEY(from_id, to_id),
    FOREIGN KEY(from_id) REFERENCES knowledge_entries(id) ON DELETE CASCADE,
    FOREIGN KEY(to_id) REFERENCES knowledge_entries(id) ON DELETE CASCADE
);

CREATE TABLE IF NOT EXISTS trade_offs (
    id TEXT PRIMARY KEY,
    knowledge_id TEXT NOT NULL,
    type TEXT CHECK(type IN ('pro', 'con')) NOT NULL,
    description TEXT NOT NULL,
    FOREIGN KEY(knowledge_id) REFERENCES knowledge_entries(id) ON DELETE CASCADE
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_type ON knowledge_entries(type);
CREATE INDEX IF NOT EXISTS idx_project ON knowledge_entries(project_path);
CREATE INDEX IF NOT EXISTS idx_confidence ON knowledge_entries(confidence);
CREATE INDEX IF NOT EXISTS idx_usage ON knowledge_entries(usage_count DESC);
CREATE INDEX IF NOT EXISTS idx_tags_tag ON tags(tag);
CREATE INDEX IF NOT EXISTS idx_timestamp ON knowledge_entries(timestamp DESC);

-- Full-text search
CREATE VIRTUAL TABLE IF NOT EXISTS knowledge_fts USING fts5(
    title,
    summary,
    detailed_explanation,
    problem_solved,
    content=knowledge_entries,
    content_rowid=rowid
);

-- Triggers for FTS sync
DROP TRIGGER IF EXISTS knowledge_fts_insert;
CREATE TRIGGER knowledge_fts_insert AFTER INSERT ON knowledge_entries BEGIN
    INSERT INTO knowledge_fts(rowid, title, summary, detailed_explanation, problem_solved)
    VALUES (new.rowid, new.title, new.summary, new.detailed_explanation, new.problem_solved);
END;

DROP TRIGGER IF EXISTS knowledge_fts_update;
CREATE TRIGGER knowledge_fts_update AFTER UPDATE ON knowledge_entries BEGIN
    UPDATE knowledge_fts
    SET title = new.title,
        summary = new.summary,
        detailed_explanation = new.detailed_explanation,
        problem_solved = new.problem_solved
    WHERE rowid = new.rowid;
END;

DROP TRIGGER IF EXISTS knowledge_fts_delete;
CREATE TRIGGER knowledge_fts_delete AFTER DELETE ON knowledge_entries BEGIN
    DELETE FROM knowledge_fts WHERE rowid = old.rowid;
END;
"""

def setup_database():
    print(f"Creating database at {DB_PATH}...")

    # Ensure parent directory exists
    DB_PATH.parent.mkdir(parents=True, exist_ok=True)

    # Connect and create schema
    conn = sqlite3.connect(DB_PATH)
    conn.executescript(SCHEMA)
    conn.commit()
    conn.close()

    print("✓ Database created successfully")
    print(f"  Location: {DB_PATH}")

    # Set permissions (user only)
    DB_PATH.chmod(0o600)
    print("✓ Permissions set (600)")

if __name__ == '__main__':
    setup_database()
```

**Kör setup:**
```bash
python3 ~/.claude/learning-system/setup-db.py
```

### Steg 1.4: Implementera Vector Store

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

        # Load existing index if available
        self.load()

    def add_knowledge(self, knowledge_id: str, text: str):
        """Add a knowledge entry to the vector store"""
        embedding = self.model.encode(text, normalize_embeddings=True)
        self.index.add(np.array([embedding], dtype='float32'))
        self.id_map.append(knowledge_id)

    def search(self, query: str, k: int = 10):
        """Search for similar entries"""
        if self.index.ntotal == 0:
            return []

        query_embedding = self.model.encode(query, normalize_embeddings=True)
        distances, indices = self.index.search(
            np.array([query_embedding], dtype='float32'),
            min(k, self.index.ntotal)
        )

        results = []
        for distance, idx in zip(distances[0], indices[0]):
            if 0 <= idx < len(self.id_map):
                results.append({
                    'knowledge_id': self.id_map[idx],
                    'similarity': float(distance)
                })
        return results

    def save(self):
        """Persist index to disk"""
        self.storage_path.mkdir(parents=True, exist_ok=True)

        # Save FAISS index
        faiss.write_index(self.index, str(self.storage_path / 'faiss.index'))

        # Save ID map
        with open(self.storage_path / 'id_map.pkl', 'wb') as f:
            pickle.dump(self.id_map, f)

    def load(self):
        """Load index from disk"""
        index_path = self.storage_path / 'faiss.index'
        map_path = self.storage_path / 'id_map.pkl'

        if index_path.exists() and map_path.exists():
            self.index = faiss.read_index(str(index_path))
            with open(map_path, 'rb') as f:
                self.id_map = pickle.load(f)
            print(f"✓ Loaded {len(self.id_map)} entries from vector store")
```

### Steg 1.5: Implementera Significance Detector

**Fil:** `~/.claude/learning-system/analysis-engine/significance_detector.py`

```python
import re
from typing import Dict, List

class SignificanceDetector:
    THRESHOLDS = {
        'files_changed': 5,
        'lines_changed': 150,
    }

    KEYWORDS = [
        'refactor', 'fix', 'bug', 'optimize', 'improve',
        'security', 'performance', 'vulnerability'
    ]

    ARCHITECTURAL_PATTERNS = [
        'controller', 'service', 'repository', 'model',
        'middleware', 'interface', 'abstract', 'factory'
    ]

    def is_significant(self, git_diff: str, commit_messages: List[str],
                      session_context: Dict = None) -> int:
        """
        Calculate significance score for changes
        Returns score (threshold default: 5)
        """
        score = 0

        # Parse git diff
        files = self._parse_git_diff(git_diff)

        # File count
        if len(files) >= self.THRESHOLDS['files_changed']:
            score += 3

        # Line count
        total_lines = sum(f['additions'] + f['deletions'] for f in files)
        if total_lines >= self.THRESHOLDS['lines_changed']:
            score += 2

        # Keyword analysis
        all_messages = ' '.join(commit_messages).lower()
        if any(kw in all_messages for kw in self.KEYWORDS):
            score += 2

        # Architectural signals
        if self._detect_architectural_changes(files):
            score += 4

        # New patterns detected
        if self._detect_new_patterns(files):
            score += 5

        # User corrections
        if session_context and session_context.get('user_corrections', 0) >= 2:
            score += 1

        return score

    def _parse_git_diff(self, git_diff: str) -> List[Dict]:
        """Parse git diff into file changes"""
        files = []
        current_file = None

        for line in git_diff.split('\n'):
            if line.startswith('diff --git'):
                if current_file:
                    files.append(current_file)
                match = re.search(r'b/(.+)$', line)
                current_file = {
                    'path': match.group(1) if match else 'unknown',
                    'additions': 0,
                    'deletions': 0
                }
            elif current_file:
                if line.startswith('+') and not line.startswith('+++'):
                    current_file['additions'] += 1
                elif line.startswith('-') and not line.startswith('---'):
                    current_file['deletions'] += 1

        if current_file:
            files.append(current_file)

        return files

    def _detect_architectural_changes(self, files: List[Dict]) -> bool:
        """Detect if changes involve architectural components"""
        for file in files:
            path = file['path'].lower()
            if any(pattern in path for pattern in self.ARCHITECTURAL_PATTERNS):
                return True
        return False

    def _detect_new_patterns(self, files: List[Dict]) -> bool:
        """Detect if changes introduce new patterns"""
        # Simple heuristic: new files with significant content
        for file in files:
            if file['additions'] > 50 and file['deletions'] < 10:
                return True
        return False
```

### Steg 1.6: Skapa config file

**Fil:** `~/.claude/learning-system/config.json`

```json
{
  "analysis": {
    "auto_trigger": true,
    "significance_threshold": 5,
    "min_files_changed": 5,
    "min_lines_changed": 150,
    "ignored_paths": ["node_modules", ".git", "vendor", "*.lock", "dist", "build"]
  },
  "ai": {
    "model": "claude-sonnet-4.5",
    "api_key_env": "ANTHROPIC_API_KEY",
    "max_tokens": 4096,
    "temperature": 0.3
  },
  "knowledge": {
    "min_confidence_for_skill": 4,
    "skill_generation_threshold": 10,
    "max_entries_per_session": 20,
    "retention_days": 365
  },
  "search": {
    "default_limit": 10,
    "min_similarity": 0.6,
    "vector_weight": 0.6,
    "fts_weight": 0.3,
    "confidence_bonus": 0.05,
    "usage_bonus": 0.05
  },
  "skills": {
    "auto_generate": true,
    "min_patterns": 5,
    "min_total_usage": 10,
    "output_dir": "~/.claude/skills/"
  },
  "logging": {
    "level": "INFO",
    "file": "~/.claude/learning-system/logs/system.log"
  }
}
```

---

## Fas 2: Integration

### Steg 2.1: Setup MCP Server

```bash
cd ~/.claude/learning-system/mcp-server

# Skapa package.json
cat > package.json << 'EOF'
{
  "name": "claude-learning-system-mcp",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "watch": "tsc --watch"
  },
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

# Skapa tsconfig.json
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ES2020",
    "moduleResolution": "node",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
EOF

# Installera dependencies
npm install

# Skapa src directory
mkdir src
```

### Steg 2.2: Implementera MCP Server

**Fil:** `~/.claude/learning-system/mcp-server/src/index.ts`

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import sqlite3 from "sqlite3";
import { promisify } from "util";
import { readFile } from "fs/promises";
import { join } from "path";
import { homedir } from "os";

const DB_PATH = join(homedir(), ".claude", "learning-system", "knowledge.db");

// Database helper
class Database {
  private db: sqlite3.Database;

  constructor() {
    this.db = new sqlite3.Database(DB_PATH);
  }

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

// Create server
const server = new Server({
  name: "claude-learning-system",
  version: "1.0.0",
}, {
  capabilities: {
    tools: {},
  },
});

// Register tools
server.setRequestHandler("tools/list", async () => {
  return {
    tools: [
      {
        name: "learn_search",
        description: "Search the knowledge base for patterns, solutions, and learnings",
        inputSchema: {
          type: "object",
          properties: {
            query: { type: "string" },
            type: { type: "string", enum: ["code_pattern", "bug_fix", "architecture", "project_rule"] },
            project: { type: "string" },
            min_confidence: { type: "integer", default: 3, minimum: 1, maximum: 5 },
            limit: { type: "integer", default: 10 }
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
            id: { type: "string" }
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
            project: { type: "string" }
          }
        }
      }
    ]
  };
});

// Implement tools
server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "learn_search") {
    const results = await learnSearch(args as any);
    return {
      content: [{
        type: "text",
        text: JSON.stringify(results, null, 2)
      }]
    };
  }

  if (name === "learn_get") {
    const result = await learnGet(args as any);
    return {
      content: [{
        type: "text",
        text: JSON.stringify(result, null, 2)
      }]
    };
  }

  if (name === "learn_status") {
    const status = await learnStatus(args as any);
    return {
      content: [{
        type: "text",
        text: JSON.stringify(status, null, 2)
      }]
    };
  }

  throw new Error(`Unknown tool: ${name}`);
});

async function learnSearch(args: any) {
  const { query, type, project, min_confidence = 3, limit = 10 } = args;

  const sql = `
    SELECT ke.*, GROUP_CONCAT(t.tag) as tags
    FROM knowledge_entries ke
    LEFT JOIN tags t ON t.knowledge_id = ke.id
    WHERE (? IS NULL OR ke.type = ?)
      AND (? IS NULL OR ke.project_path LIKE ?)
      AND ke.confidence >= ?
    GROUP BY ke.id
    ORDER BY ke.usage_count DESC, ke.confidence DESC
    LIMIT ?
  `;

  const results = await db.all(sql, [
    type, type,
    project, project ? `%${project}%` : null,
    min_confidence,
    limit
  ]);

  // Update usage stats
  for (const result of results.slice(0, 5)) {
    await db.run(
      "UPDATE knowledge_entries SET usage_count = usage_count + 1, last_used = strftime('%s', 'now') WHERE id = ?",
      [result.id]
    );
  }

  return { results, query, total: results.length };
}

async function learnGet(args: any) {
  const { id } = args;

  const entry = await db.get(
    "SELECT * FROM knowledge_entries WHERE id = ?",
    [id]
  );

  if (!entry) {
    return { error: "Entry not found" };
  }

  // Get related data
  const tags = await db.all("SELECT tag FROM tags WHERE knowledge_id = ?", [id]);
  const examples = await db.all("SELECT * FROM code_examples WHERE knowledge_id = ?", [id]);
  const tradeOffs = await db.all("SELECT * FROM trade_offs WHERE knowledge_id = ?", [id]);

  return {
    ...entry,
    tags: tags.map(t => t.tag),
    code_examples: examples,
    trade_offs: tradeOffs
  };
}

async function learnStatus(args: any) {
  const { project } = args;

  const stats = await db.get(`
    SELECT
      COUNT(*) as total_entries,
      COUNT(CASE WHEN type = 'code_pattern' THEN 1 END) as patterns,
      COUNT(CASE WHEN type = 'bug_fix' THEN 1 END) as bug_fixes,
      COUNT(CASE WHEN type = 'architecture' THEN 1 END) as architecture,
      COUNT(CASE WHEN type = 'project_rule' THEN 1 END) as project_rules,
      AVG(confidence) as avg_confidence,
      SUM(usage_count) as total_usage
    FROM knowledge_entries
    WHERE (? IS NULL OR project_path LIKE ?)
  `, [project, project ? `%${project}%` : null]);

  const recent = await db.all(`
    SELECT title, type, timestamp, confidence
    FROM knowledge_entries
    ORDER BY timestamp DESC
    LIMIT 10
  `);

  return { stats, recent };
}

// Start server
const transport = new StdioServerTransport();
await server.connect(transport);
console.error("Claude Learning System MCP server running");
```

**Bygg:**
```bash
cd ~/.claude/learning-system/mcp-server
npm run build
```

### Steg 2.3: Konfigurera Claude Code

**Modifiera:** `~/.claude/settings.json`

```json
{
  "mcpServers": {
    "claude-learning-system": {
      "command": "node",
      "args": ["/Users/johan/.claude/learning-system/mcp-server/dist/index.js"]
    }
  },
  "permissions": {
    "allow": [
      "... existing permissions ...",
      "Read(/Users/johan/.claude/**)"
    ]
  }
}
```

### Steg 2.4: Skapa Slash Commands

**Fil:** `~/.claude/commands/learn.md`

```markdown
---
description: Manuellt trigga learning analys
argument-hint:
---

Trigger learning system att analysera och extrahera kunskap från recent work.

Usage:
- `/learn` - Analysera current session
- `/learn commit` - Analysera senaste commit
- `/learn files <paths>` - Analysera specifika filer

Systemet kör deep AI-analys för att extrahera:
- Code patterns och best practices
- Bug fixes och lösningar
- Arkitektoniska insights
- Projektspecifika regler
```

**Fil:** `~/.claude/commands/learn-status.md`

```markdown
---
description: Visa learning system statistik
argument-hint: [project]
---

Visa statistik om learned knowledge.

Usage:
- `/learn-status` - Global statistik
- `/learn-status project` - Current project statistik

Visar:
- Total knowledge entries per typ
- Recent learnings
- Mest använda patterns
- Confidence distribution
```

**Fil:** `~/.claude/commands/learn-search.md`

```markdown
---
description: Sök i kunskapsbasen
argument-hint: <query>
---

Sök i learned knowledge för relevanta patterns och lösningar.

Usage:
- `/learn-search <query>`
- `/learn-search <query> --type code_pattern`
- `/learn-search <query> --project /path/to/project`

Returnerar relevanta knowledge entries rankade efter similarity och usage.
```

---

## Fas 3: AI Analysis Pipeline

*OBS: Denna fas kräver ANTHROPIC_API_KEY*

### Steg 3.1: Sätt API Key

```bash
# Lägg till i ~/.zshrc eller ~/.bashrc
export ANTHROPIC_API_KEY="your-api-key-here"

# Ladda om
source ~/.zshrc
```

### Steg 3.2: Implementera Deep Analyzer

**Fil:** `~/.claude/learning-system/analysis-engine/deep_analyzer.py`

```python
import anthropic
import json
import os

class DeepAnalyzer:
    STAGE_PROMPTS = {
        'what': """Analyze these code changes and describe WHAT was done:

Git Diff:
{git_diff}

Commit Messages:
{commit_messages}

Provide:
1. Summary of changes (2-3 sentences)
2. Files affected and their roles
3. Type of change (feature, bug fix, refactoring, etc.)

Output as JSON with keys: summary, files, change_type""",

        'why': """Now analyze WHY these decisions were made:

Changes: {what_analysis}
Session Context: {session_context}

Extract:
1. Problem that was being solved
2. Trade-offs considered
3. Why this approach over alternatives
4. Architectural principles applied

Output as JSON with keys: problem, trade_offs, alternatives, principles""",

        'patterns': """Extract reusable patterns and principles:

Full Context: {full_context}

Identify:
1. Code patterns that could be reused
2. Architectural principles demonstrated
3. Best practices shown
4. Anti-patterns avoided and why
5. Project-specific conventions followed

Output as JSON with keys: patterns, best_practices, anti_patterns""",

        'knowledge': """Create knowledge base entries:

Analysis: {pattern_analysis}

For each significant pattern/insight, generate:
1. Title (5-10 words)
2. Category (code_pattern | bug_fix | architecture | project_rule)
3. Summary (2-3 sentences)
4. Detailed explanation with code examples
5. When to apply
6. When NOT to apply
7. Tags for search
8. Confidence level (1-5)

Output as JSON array of knowledge entries."""
    }

    def __init__(self, api_key=None):
        self.client = anthropic.Anthropic(
            api_key=api_key or os.environ.get('ANTHROPIC_API_KEY')
        )
        self.model = "claude-sonnet-4.5-20251022"

    def analyze(self, session_data):
        """Run full 4-stage analysis"""
        # Stage 1: WHAT
        what_analysis = self._call_claude(
            self.STAGE_PROMPTS['what'],
            {
                'git_diff': session_data['git_diff'][:10000],  # Limit size
                'commit_messages': '\n'.join(session_data['commit_messages'])
            }
        )

        # Stage 2: WHY
        why_analysis = self._call_claude(
            self.STAGE_PROMPTS['why'],
            {
                'what_analysis': what_analysis,
                'session_context': session_data.get('session_context', '')[:5000]
            }
        )

        # Stage 3: PATTERNS
        patterns = self._call_claude(
            self.STAGE_PROMPTS['patterns'],
            {
                'full_context': what_analysis + '\n\n' + why_analysis
            }
        )

        # Stage 4: KNOWLEDGE
        knowledge = self._call_claude(
            self.STAGE_PROMPTS['knowledge'],
            {
                'pattern_analysis': patterns
            }
        )

        return {
            'what': json.loads(what_analysis),
            'why': json.loads(why_analysis),
            'patterns': json.loads(patterns),
            'knowledge': json.loads(knowledge)
        }

    def _call_claude(self, prompt_template, variables):
        """Call Claude API"""
        prompt = prompt_template.format(**variables)

        message = self.client.messages.create(
            model=self.model,
            max_tokens=4096,
            temperature=0.3,
            messages=[{
                "role": "user",
                "content": prompt
            }]
        )

        return message.content[0].text
```

### Steg 3.3: Implementera Knowledge Extractor

**Fil:** `~/.claude/learning-system/analysis-engine/knowledge_extractor.py`

```python
import json
import uuid
import time
from typing import Dict, List

class KnowledgeExtractor:
    def extract(self, analysis_result: Dict, session_data: Dict) -> List[Dict]:
        """Extract structured knowledge entries from analysis"""
        entries = []

        knowledge_list = analysis_result['knowledge']
        if not isinstance(knowledge_list, list):
            knowledge_list = [knowledge_list]

        for item in knowledge_list:
            entry = {
                'id': str(uuid.uuid4()),
                'timestamp': int(time.time()),
                'session_id': session_data.get('session_id', 'unknown'),
                'type': item.get('category', 'code_pattern'),
                'title': item['title'],
                'summary': item['summary'],
                'detailed_explanation': item['detailed_explanation'],
                'problem_solved': item.get('problem_solved', ''),
                'when_to_apply': item.get('when_to_apply', ''),
                'when_not_to_apply': item.get('when_not_to_apply', ''),
                'confidence': item.get('confidence', 3),
                'project_path': session_data.get('project_path', ''),
                'tags': item.get('tags', []),
                'code_examples': self._extract_code_examples(item),
                'trade_offs': item.get('trade_offs', []),
                'metadata': json.dumps({
                    'lines_changed': session_data.get('lines_changed', 0),
                    'commits': session_data.get('commits', []),
                    'user_corrections': session_data.get('user_corrections', 0)
                })
            }
            entries.append(entry)

        return entries

    def _extract_code_examples(self, item: Dict) -> List[Dict]:
        """Extract code examples from item"""
        examples = []
        if 'code_examples' in item:
            for ex in item['code_examples']:
                examples.append({
                    'id': str(uuid.uuid4()),
                    'language': ex.get('language', 'text'),
                    'code': ex.get('code', ''),
                    'context': ex.get('context', '')
                })
        return examples
```

### Steg 3.4: Implementera Main Analysis Script

**Fil:** `~/.claude/learning-system/analysis-engine/main.py`

```python
#!/usr/bin/env python3
import sys
import json
import argparse
from pathlib import Path
import sqlite3

from significance_detector import SignificanceDetector
from deep_analyzer import DeepAnalyzer
from knowledge_extractor import KnowledgeExtractor
from vector_store import VectorStore

class LearningSystem:
    def __init__(self, base_dir):
        self.base_dir = Path(base_dir)
        self.db_path = self.base_dir / "knowledge.db"
        self.vector_path = self.base_dir / "vectors"

        self.detector = SignificanceDetector()
        self.analyzer = DeepAnalyzer()
        self.extractor = KnowledgeExtractor()
        self.vectors = VectorStore(str(self.vector_path))

        self.db = sqlite3.connect(str(self.db_path))

    def analyze_session(self, session_data, threshold=5):
        """Main analysis pipeline"""
        print(f"Analyzing session {session_data['session_id']}...")

        # Step 1: Check significance
        score = self.detector.is_significant(
            session_data['git_diff'],
            session_data['commit_messages'],
            session_data.get('session_context')
        )

        if score < threshold:
            print(f"  Significance score {score} below threshold {threshold}, skipping")
            return

        print(f"  Significant changes detected (score: {score})")

        # Step 2: Deep analysis
        print("  Running deep analysis...")
        analysis = self.analyzer.analyze(session_data)

        # Step 3: Extract knowledge
        print("  Extracting knowledge entries...")
        entries = self.extractor.extract(analysis, session_data)

        # Step 4: Store
        print(f"  Storing {len(entries)} knowledge entries...")
        for entry in entries:
            self.store_knowledge(entry)

        print(f"✓ Learning complete: {len(entries)} entries added")

    def store_knowledge(self, entry):
        """Store knowledge entry in DB and vector store"""
        cursor = self.db.cursor()

        # Insert main entry
        cursor.execute("""
            INSERT OR REPLACE INTO knowledge_entries
            (id, timestamp, session_id, type, title, summary,
             detailed_explanation, problem_solved, when_to_apply,
             when_not_to_apply, confidence, project_path, metadata)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            entry['id'], entry['timestamp'], entry['session_id'],
            entry['type'], entry['title'], entry['summary'],
            entry['detailed_explanation'], entry['problem_solved'],
            entry['when_to_apply'], entry['when_not_to_apply'],
            entry['confidence'], entry['project_path'], entry['metadata']
        ))

        # Insert tags
        for tag in entry['tags']:
            cursor.execute(
                "INSERT OR IGNORE INTO tags (knowledge_id, tag) VALUES (?, ?)",
                (entry['id'], tag)
            )

        # Insert code examples
        for example in entry.get('code_examples', []):
            cursor.execute("""
                INSERT INTO code_examples
                (id, knowledge_id, language, code, context)
                VALUES (?, ?, ?, ?, ?)
            """, (
                example['id'], entry['id'], example['language'],
                example['code'], example['context']
            ))

        self.db.commit()

        # Add to vector store
        text = f"{entry['title']} {entry['summary']} {entry['detailed_explanation']}"
        self.vectors.add_knowledge(entry['id'], text)
        self.vectors.save()

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--session-id', required=True)
    parser.add_argument('--project-path', required=True)
    parser.add_argument('--git-diff', required=True)
    parser.add_argument('--commits', required=True)
    parser.add_argument('--threshold', type=int, default=5)

    args = parser.parse_args()

    base_dir = Path.home() / ".claude" / "learning-system"
    system = LearningSystem(base_dir)

    session_data = {
        'session_id': args.session_id,
        'project_path': args.project_path,
        'git_diff': args.git_diff,
        'commit_messages': args.commits.split('\n'),
        'lines_changed': len(args.git_diff.split('\n'))
    }

    system.analyze_session(session_data, args.threshold)

if __name__ == '__main__':
    main()
```

**Gör executable:**
```bash
chmod +x ~/.claude/learning-system/analysis-engine/main.py
```

---

## Fas 4: Skill Generator

*Se SPECIFICATION.md för detaljer om skill generation.*

Implementation av skill generator är optional för MVP. Kan adderas senare när kunskapsbasen har vuxit.

---

## Testing & Validation

### Test 1: Database Setup

```bash
python3 ~/.claude/learning-system/setup-db.py

# Verifiera
sqlite3 ~/.claude/learning-system/knowledge.db "SELECT name FROM sqlite_master WHERE type='table';"
```

**Förväntat output:**
```
knowledge_entries
code_examples
tags
relations
trade_offs
knowledge_fts
```

### Test 2: Vector Store

```python
from vector_store import VectorStore

vs = VectorStore("~/.claude/learning-system/vectors")
vs.add_knowledge("test-1", "XSS protection using DOMPurify")
vs.add_knowledge("test-2", "SQL injection prevention with prepared statements")
results = vs.search("security vulnerability", k=2)
print(results)
```

### Test 3: MCP Server

```bash
cd ~/.claude/learning-system/mcp-server
npm run build
node dist/index.js
# Should output: "Claude Learning System MCP server running"
# Press Ctrl+C to stop
```

### Test 4: End-to-End

1. Starta Claude Code i ett git repo
2. Gör ändringar (≥5 filer)
3. Commit med message innehållande "fix" eller "refactor"
4. Avsluta session
5. Kontrollera logs: `tail ~/.claude/learning-system/logs/system.log`
6. Kontrollera database: `sqlite3 ~/.claude/learning-system/knowledge.db "SELECT COUNT(*) FROM knowledge_entries;"`

---

## Troubleshooting

### Problem: API Key Error

```bash
# Verifiera att key är satt
echo $ANTHROPIC_API_KEY

# Om inte, sätt den:
export ANTHROPIC_API_KEY="your-key"
```

### Problem: ModuleNotFoundError

```bash
# Aktivera virtual environment
cd ~/.claude/learning-system/analysis-engine
source venv/bin/activate

# Installera om dependencies
pip install -r requirements.txt
```

### Problem: MCP Server Not Found

```bash
# Verifiera att dist/ finns
ls ~/.claude/learning-system/mcp-server/dist/

# Om inte, rebuild:
cd ~/.claude/learning-system/mcp-server
npm run build
```

### Problem: Database Lock

```bash
# Stäng alla connections
pkill -f "claude-learning-system"

# Verifiera att ingen process har låst DB
lsof ~/.claude/learning-system/knowledge.db
```

---

## Maintenance

### Backup

```bash
# Backup database
cp ~/.claude/learning-system/knowledge.db ~/.claude/learning-system/knowledge.db.backup

# Backup vectors
tar -czf ~/.claude/learning-system/vectors-backup.tar.gz ~/.claude/learning-system/vectors/
```

### Cleanup Old Entries

```bash
sqlite3 ~/.claude/learning-system/knowledge.db << 'EOF'
DELETE FROM knowledge_entries
WHERE timestamp < strftime('%s', 'now', '-365 days');
EOF
```

### Optimize Database

```bash
sqlite3 ~/.claude/learning-system/knowledge.db "VACUUM;"
```

---

## Next Steps

Efter implementation:

1. **Monitorera första veckan** - Kontrollera logs och API costs
2. **Justera thresholds** - Ändra significance_threshold baserat på behavior
3. **Implementera skill generator** - När ≥50 patterns i databasen
4. **Utöka prompts** - Finjustera AI-prompts baserat på output quality
5. **Add metrics dashboard** - Bygg enkelt dashboard för insights

## Support

För problem eller frågor:
- Check logs in `~/.claude/learning-system/logs/`
- Review SPECIFICATION.md för tekniska detaljer
- Kontrollera API usage på anthropic.com/account
