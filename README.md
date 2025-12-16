# Claude Code Self-Learning System

Ett självlärande system som transformerar Claude Code från "Execute & Forget" till "Execute & Learn".

## Översikt

Detta system gör att Claude Code automatiskt kan extrahera, lagra och återanvända kunskaper från tidigare sessioner.

**Två implementation paths:**

| Feature | MVP (15 min) | Full System (7 dagar) |
|---------|--------------|----------------------|
| **Setup** | 1 slash command (+ optional git hooks) | MCP server + Python + TypeScript |
| **Trigger** | `/learn` + git commits (optional) | SessionEnd (automatisk) |
| **Data Sources** | Git diff + session history | Git diff + session history |
| **Storage** | Skills (index + details) | SQLite + FAISS vectors |
| **Loading** | Lazy (index vid start, details on-demand) | Full load + semantic search |
| **Scalability** | 100+ learnings (~1.5k tokens) | Unlimited with vector search |
| **Access** | Skills index → Read tool | MCP Server tools |
| **Dependencies** | ZERO (bara Claude Code) | Python, Node, anthropic, faiss |
| **Cost** | ~$0.02/learn | ~$0.10-0.50/session |
| **Context Poisoning** | ✅ Filtreras automatiskt | ✅ Filtreras automatiskt |

**Rekommendation:** Starta med **MVP** (`MVP.md`), samla 20-30 learnings, utvärdera värdet, bygg sedan ut till full system om det visar sig värdefullt.

Det fullständiga systemet kombinerar MCP Server (för kunskapslagring och semantic search) med auto-genererade Skills (för snabb access till vanliga patterns) och bygger en växande kunskapsbas över tid.

## Hur det fungerar

```
Session 1: Claude fixar XSS-bug
           ↓
    Analys extraherar kunskap
           ↓
    Sparas i kunskapsdatabas

Session 2: Liknande säkerhetsproblem
           ↓
    Claude söker i databasen
           ↓
    Hittar tidigare XSS-lösning
           ↓
    Applicerar proven pattern

Session N: Pattern använt 10+ gånger
           ↓
    Auto-genererar skill
           ↓
    Läses automatiskt vid session start
```

## Komponenter

### 1. Analysis Engine (Python)
- Detekterar signifikanta ändringar (≥5 filer, refactorings, buggfixar)
- Kör 4-stage AI-analys med Claude API
- Extraherar patterns, principles, och trade-offs
- Genererar strukturerade knowledge entries

### 2. Knowledge Database (SQLite + FAISS)
- SQLite för strukturerad data
- FAISS för vector embeddings (semantic search)
- FTS5 för full-text keyword search
- Hybrid ranking för bästa resultat

### 3. MCP Server (TypeScript)
Ger Claude tillgång till:
- `learn_search(query)` - Sök i kunskapsbasen
- `learn_get(id)` - Hämta specifik kunskap
- `learn_status()` - Visa statistik
- `learn_trigger()` - Manuell analys

### 4. Skill Generator (Python)
- Analyserar frequently used patterns
- Auto-genererar skills när threshold nås
- Kompilerar patterns till SKILL.md + reference.md
- Läses automatiskt vid session start

## Triggering

### Automatiskt (Smart Hybrid)
- När session slutar → SessionEnd hook triggas
- Om ≥5 filer ändrade, refactoring, eller buggfix → AI-analys körs
- Annars → Skippas (sparar API-kostnader)

### Manuellt
```bash
/learn              # Analysera current session
/learn commit       # Analysera senaste commit
/learn-status       # Visa statistik
/learn-search "XSS protection patterns"
```

## Kunskapstyper

Systemet extraherar och lagrar:

1. **Code patterns & best practices** - Återanvändbara mönster från din kodbas
2. **Bug fixes & solutions** - Hur problem löstes och varför
3. **Project-specific rules** - Projektkonventioner som upptäckts
4. **Architectural insights** - Designbeslut och trade-offs

## Installation

### MVP/POC (Rekommenderad första steg)

Se `MVP.md` för en ultra-enkel första implementation som:
- **Ett enda slash command** - Inget annat behövs!
- **Triggers:** `/learn` (manuell) + git commits (optional)
- **Rich learning:** Git diff + session history + failed approaches
- **Context poisoning prevention:** Filtrerar villovägar automatiskt
- Använder Claude Code's Task tool (ingen Python)
- Sparar learnings som auto-loaded skills
- **Setup tid: ~15-30 minuter**

```bash
# Quick start MVP
# 1. Skapa ~/.claude/commands/learn.md (se MVP.md)
# 2. Optional: Setup git post-commit hook
# 3. Kör /learn efter ändringar (eller automatiskt vid commit)
# 4. Skills laddas automatiskt nästa session
# That's it!
```

**Nyckelfördelar:**
- ✅ ZERO externa dependencies (bara Claude Code)
- ✅ Extraherar från både kod OCH session (inkl. villovägar)
- ✅ Context poisoning filter = lär sig vad som INTE fungerar också
- ✅ **Lazy loading optimization** - Skalbar till 100+ learnings utan context bloat
- ✅ Skills (lightweight index) laddas automatiskt
- ✅ Full details läses on-demand när behövs
- ✅ Optional git hooks för automatisk trigger
- ✅ Claude har direkt access till alla learnings

### Full System

Se `IMPLEMENTATION.md` för detaljerad guide till full system med:

**Kort version:**
1. Skapa directory structure i `~/.claude/learning-system/`
2. Installera Python dependencies (anthropic, sentence-transformers, faiss-cpu)
3. Installera Node dependencies (@modelcontextprotocol/sdk, sqlite3)
4. Konfigurera MCP server i `~/.claude/settings.json`
5. Sätt up SessionEnd hook
6. Skapa slash commands

**Rekommendation:** Börja med MVP, samla 20-30 learnings, utvärdera, sen bygg ut till full system om nödvändigt.

## Användningsexempel

### Exempel 1: Bug Fix Learning

```
Du: "Det här formuläret är sårbart för XSS"

Claude: *Söker i kunskapsbasen*
        *Hittar: "XSS Protection Pattern - DOMPurify"*

        "Jag hittar att vi tidigare löste liknande XSS-sårbarheter
         genom att använda DOMPurify. Låt mig applicera samma pattern här."

        *Implementerar lösning*
        *Session slutar → Analys → Uppdaterar kunskap med ny kontext*
```

### Exempel 2: Architectural Pattern

```
Session 1-5: Bygger flera API endpoints med liknande struktur
             → Analys extraherar "API Controller Pattern"
             → Sparas som code_pattern med confidence 3

Session 6-10: Fortsätter använda pattern
              → Confidence ökar till 4
              → Usage count når 10

              → Skill auto-genereras: "learned-api-patterns"
              → Nästa session: Claude läser skill automatiskt
                och applicerar pattern utan att söka
```

## Teknisk Översikt

**Arkitektur:**
```
┌─────────────────────────────────────────────────┐
│           Claude Code Session                    │
│   Git Changes + Session History                 │
└────────────────┬────────────────────────────────┘
                 ↓
┌────────────────────────────────────────────────┐
│         Analysis Engine (Python)                │
│  • Significance Detector                        │
│  • Deep Analyzer (4-stage AI)                   │
│  • Knowledge Extractor                          │
└────────────────┬────────────────────────────────┘
                 ↓
┌────────────────┴────────────────┐
│                                 │
│  ┌──────────────────┐  ┌───────┴──────────┐
│  │  Knowledge DB     │  │ Skill Generator   │
│  │  SQLite + FAISS   │  │ Auto-creates      │
│  └─────────┬─────────┘  │ .claude/skills/   │
│            │             └───────────────────┘
│            ↓
│  ┌──────────────────┐
│  │   MCP Server      │
│  │   (TypeScript)    │
│  └─────────┬─────────┘
│            ↓
│  ┌──────────────────┐
│  │  Claude Code      │
│  │  (Next Session)   │
│  └───────────────────┘
```

**Storage:**
- SQLite: ~2KB per knowledge entry
- FAISS embeddings: ~1.5KB per entry
- 10,000 entries ≈ 35MB
- Models (sentence-transformers): ~90MB

**Performance:**
- Search latency: <100ms för 10k entries
- Analysis time: 30-60 sekunder per session
- Memory overhead: ~200MB (model loading)

## Konfiguration

`~/.claude/learning-system/config.json`:

```json
{
  "analysis": {
    "auto_trigger": true,
    "significance_threshold": 5,
    "min_files_changed": 5,
    "ignored_paths": ["node_modules", ".git", "vendor"]
  },
  "ai": {
    "model": "claude-sonnet-4.5",
    "api_key_env": "ANTHROPIC_API_KEY"
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
  },
  "skills": {
    "auto_generate": true,
    "min_patterns": 5,
    "min_usage": 10,
    "output_dir": "~/.claude/skills/"
  }
}
```

## Kostnader

**API Costs (Claude):**
- Per session analys: ~$0.10-0.50 (beroende på mängd ändringar)
- Significance detection sparar kostnader (endast stora ändringar analyseras)
- Estimerat: ~$5-20/månad vid aktiv användning

**Compute:**
- Vector search: Minimal (< 1 sekund)
- Model loading: ~2 sekunder första gången per session

## Säkerhet

- All data lagras lokalt i `~/.claude/learning-system/`
- Ingen data skickas till externa servrar (förutom Claude API för analys)
- API-nycklar läses från environment variables
- SQLite-databas kan krypteras om önskat
- Git-ignored filer respekteras automatiskt

## Begränsningar

**Nuvarande:**
- Endast git-baserade projekt (kräver git diff)
- Kräver ANTHROPIC_API_KEY för analys
- Svensk/engelsk text stöds (modellen är multilingual)
- Skalat för ~100k knowledge entries

**Framtida förbättringar:**
- Remote sync mellan devices
- Web UI för knowledge browsing
- Export/import av kunskapsbas
- Team sharing capabilities

## Dependencies

**Python:**
- anthropic >= 0.40.0
- sentence-transformers >= 3.0.0
- faiss-cpu >= 1.8.0
- numpy >= 1.26.0

**Node:**
- @modelcontextprotocol/sdk ^1.0.0
- sqlite3 ^5.1.7
- typescript ^5.0.0

**System:**
- Python 3.9+
- Node.js 18+
- Git
- ~500MB diskutrymme

## Support & Documentation

- **SPECIFICATION.md** - Detaljerad teknisk specifikation
- **IMPLEMENTATION.md** - Steg-för-steg implementation guide
- Issues: [GitHub repository URL]

## License

[Specify license]

## Författare

Designat för Johan's Claude Code setup.
