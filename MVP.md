# MVP/POC: Self-Learning System (v2 - Ultra Simple)

Den absolut enklaste implementationen som använder Claude Codes befintliga **skill-system** och **Task tool**.

## Koncept

**Trigger:**
- `/learn` - Manuell trigger efter betydelsefulla ändringar

**Kunskapskällor:**
- Git diff (code changes)
- Session summary (sammanfattad av Claude före Task spawn)
- User corrections (what didn't work)

**Struktur:**
- **Storage:** Skills i `~/.claude/skills/learned-{topic}/`
- **Access:** Skills laddas automatiskt vid session start
- **Analys:** Claude Code Task tool (general-purpose agent)

## Arkitektur

```
User: /learn
    ↓
Slash command expanderas → Claude (huvudsession) har full konversationshistorik
    ↓
Claude sammanfattar sessionen:
  - Vad diskuterades?
  - Vilka lösningar prövades?
  - Vad fungerade vs inte?
  - Användarens korrigeringar
    ↓
Claude spawnar Task med:
  - Session summary (från steget ovan)
  - Instruktion att läsa git diff
    ↓
Agent:
  1. Läser git diff
  2. Analyserar session summary
  3. Identifierar: what worked, what didn't, why
  4. Extraherar learnings (med context poisoning filter)
  5. Uppdaterar/skapar skills
    ↓
Skills auto-loaded nästa session
```

**Viktigt:** Session history-problemet löses genom att Claude i huvudsessionen (som har tillgång till konversationen) sammanfattar innan Task spawnas. Agenten får sammanfattningen som input.

## Fördelar

- ✅ **ZERO externa dependencies** - Bara Claude Code
- ✅ **10 minuters setup** - En slash command-fil
- ✅ **Rik kunskapsextraktion** - Session summary + git changes
- ✅ **Context poisoning prevention** - Filtrerar failed approaches
- ✅ **Context optimization** - Lazy loading, skalbar till 100+ learnings
- ✅ Använder befintligt skill-system
- ✅ Använder befintligt Task tool
- ✅ Skills (lightweight index) laddas automatiskt varje session
- ✅ Full details läses on-demand via Read tool
- ✅ Ingen MCP server, databas, eller Python script
- ✅ Extremt enkelt att debugga
- ✅ Claude har direkt access (ingen sökning behövs)

## Context Poisoning Prevention

Viktigt: Session history innehåller både bra och dåliga approaches. Agenten måste filtrera:

**Strategies:**

1. **Final Code = Ground Truth**
   - Om något inte finns i final git diff, var det troligen fel approach
   - Fokusera på vad som SLUTADE i koden

2. **User Corrections Signal**
   - "Nej inte så" → Markera som failed approach
   - "Det funkade inte" → Anti-pattern
   - User rewrites → Original var fel

3. **Explicit Separation i Learnings:**
   ```markdown
   ✅ **What Worked:** DOMPurify for XSS protection
   ❌ **What Didn't Work:** Regex sanitization - missed edge cases
   💡 **Key Insight:** Use established security libraries
   ```

4. **Agent Instructions:**
   - Analyze conversation för failed attempts
   - Extract WHY they failed
   - Store as "anti-patterns" or "pitfalls to avoid"
   - This becomes VALUABLE knowledge!

**Exempel Learning med Context:**
```markdown
## XSS Protection Implementation

**Final Solution:** DOMPurify library integration

**What Worked:**
- DOMPurify handles all edge cases
- Simple API: `DOMPurify.sanitize(input)`
- Actively maintained

**What We Tried First (Didn't Work):**
- Manual regex sanitization
- WHY IT FAILED: Missed <img onerror> and other vectors
- LESSON: Don't roll your own security

**Key Pattern:** For security-critical code, prefer battle-tested libraries
```

Failade approaches är lika värdefulla som framgångsrika - de förhindrar att samma misstag görs igen!

## Context Optimization (Lazy Loading)

**Problem:** Om alla learnings laddas fullt vid session start → context explosion

**Lösning:** Separera index från details

### SKILL.md (Lightweight - Always Loaded)
- ~40-60 tokens per learning entry
- Only: title, use-when, tags
- Think of it as a table of contents
- 50 learnings = ~2,000-3,000 tokens (manageable)

**Example:**
```markdown
### 2025-01-15 - XSS Protection with DOMPurify
- **Use when:** Handling user input that will be rendered as HTML
- **Tags:** security, javascript, xss
```

### reference.md (Heavy - Read On-Demand)
- ~500-2000 tokens per learning
- Full details, code examples, trade-offs, failed approaches
- Claude only reads when actually needed via Read tool

**Example flow:**

```
Session Start:
  ✓ Load all SKILL.md files (lightweight indexes)
  Total: ~2-3k tokens for 50 learnings

User: "Add a comment form"

Claude: *Sees in learned-security SKILL.md*
        "XSS Protection - Use when: handling user input"
        *Knows it's relevant*

Claude: Read ~/.claude/skills/learned-security/reference.md
        *Gets full details*

        "I'll implement with DOMPurify based on our XSS learning..."
```

### Benefits

**Scales to 100+ learnings:**
- 100 learnings × 50 tokens = ~5,000 tokens at session start
- vs 100 learnings × 1,500 tokens = 150k tokens (impossible!)

**Fast skill loading:**
- Lightweight indexes load instantly
- No performance impact even with many skills

**On-demand details:**
- Full context only when needed
- Pay-per-use token model

**Best of both worlds:**
- Claude knows what knowledge exists (index)
- Claude gets details only when relevant (lazy load)

## Implementation

### Steg 1: Skapa slash command

**Det är allt som behövs!**

**Fil:** `~/.claude/commands/learn.md`

### Steg 1.5: Konfigurera permissions (Lokalt per projekt) - frivilligt

**VIKTIGT:** För att Task-agenten ska kunna skapa/uppdatera skills utan att fråga om tillåtelse måste Write-permissions läggas till i **projektets** lokala `.claude/settings.local.json`

Specialist-skills och learned-skills permissions finns redan i `~/.claude/settings.json` (globalt), men Task-agenten använder projektets lokala settings när den spawnas.

**Lägg till i ditt projekts `.claude/settings.local.json`:**

```json
{
  "permissions": {
    "allow": [
      "Read(~/.claude/skills/**)",
      "Edit(~/.claude/skills/**)",
      "Write(~/.claude/skills/**)"
    ]
  }
}
```

**Om filen inte finns:**
```bash
# Skapa projektets settings-fil
mkdir -p .claude
cat > .claude/settings.local.json << 'EOF'
{
  "permissions": {
    "allow": [
      "Read(~/.claude/skills/**)",
      "Edit(~/.claude/skills/**)",
      "Write(~/.claude/skills/**)"
    ]
  }
}
EOF
```

**Varför behövs detta?**
- Task-agenten spawnas i projektets kontext
- Den ärver projektets lokala settings, inte globala
- Utan Write-permissions kan agenten inte skapa learned-skills
- Detta måste göras **en gång per projekt** där du vill använda `/learn`

```markdown
---
description: Analyze and extract learnings from session, changes or commit and save as skills. Default scope is uncommitted changes + session.
argument-hint: [changes] [commit [<hash>]] [session]
---

Analyze code changes and/or session learnings, then save as skills for future sessions.

## What this does

1. Summarizes the current session (what was discussed, tried, worked/failed)
2. Analyzes code changes (uncommitted, commits, or both)
3. Extracts patterns, techniques, and knowledge
4. Categorizes by tags (security, performance, php, etc.)
5. Updates or creates skills in ~/.claude/skills/learned-{tag}/
6. Skills are automatically loaded in future sessions

## Scope Options

Three sources can be analyzed:
- **`changes`** - Uncommitted changes (git diff)
- **`commit [hash]`** - Specific commit (default: HEAD)
- **`session`** - Session conversation and learnings

**Default:** `changes session` (if no arguments given)
**When specified:** Only analyze what you specify

## Usage

```bash
/learn                          # Default: changes + session
/learn session                  # Only session learnings
/learn changes                  # Only uncommitted changes
/learn commit                   # Only last commit (HEAD)
/learn commit abc123            # Only specific commit
/learn changes session          # Uncommitted + session
/learn commit session           # Last commit + session
/learn changes commit           # Uncommitted + last commit
/learn changes commit session   # All three sources
/learn changes commit abc123    # Uncommitted + specific commit
```

## Step 1: Session Summary (CRITICAL)

Before spawning the Task agent, YOU (Claude in the main session) must create a high-quality session summary.
You have access to the full conversation history - the spawned agent does NOT.

**Why this matters:** The quality of extracted learnings depends entirely on this summary. Be thorough.

### Summary Template

Create a structured summary using this format:

```
SESSION SUMMARY
===============

## Goal
[What problem were we trying to solve? What was the user's original request?]

## Chronological Approach Log
1. [First approach tried] → [Result: worked/failed] → [Why?]
2. [Second approach tried] → [Result: worked/failed] → [Why?]
3. ...

## Final Solution
[What ended up in the code? What was the winning approach?]

## Failed Approaches (Important for anti-patterns)
- [Approach]: [Why it failed] - LESSON: [What we learned]
- ...

## User Corrections
- [Any "no, not like that", rewrites, or redirections from user]

## Key Decisions
- [Decision]: [Why we chose X over Y]
- ...

## Techniques Worth Remembering
- [Specific patterns, libraries, or methods that proved valuable]
```

### Quality Checklist

Before spawning the agent, verify your summary:
- [ ] Covers ALL significant approaches tried, not just the final one
- [ ] Explains WHY failed approaches failed (not just "didn't work")
- [ ] Includes user corrections verbatim when possible
- [ ] Captures the reasoning behind key decisions
- [ ] Is detailed enough that someone with no context could understand

## Step 2: Spawn Task Agent

Use the Task tool to spawn a general-purpose agent with this prompt:

```
You are extracting learnings from code changes and/or session context to build a rich knowledge base.

SCOPE: {Parse arguments and specify what to analyze:}
- If no arguments: "changes + session"
- If arguments given: only what's specified
- Possible: "changes", "commit [hash]", "session", or combinations

EXAMPLES:
- /learn → analyze: changes + session
- /learn session → analyze: session only
- /learn changes commit abc123 → analyze: uncommitted changes + commit abc123

SESSION SUMMARY (from main session):
{Insert the session summary you created in Step 1 here - or skip this section if "session" not in scope}

INSTRUCTIONS:

1. READ THE DATA BASED ON SCOPE:

   A. CODE CHANGES (if "changes" in scope):
   - Run: git diff
   - This captures uncommitted changes (staged and unstaged)

   B. COMMIT (if "commit" in scope):
   - If hash provided: git show {hash}
   - If no hash: git show HEAD
   - This captures committed changes

   C. SESSION SUMMARY (if "session" in scope):
   - Review the session summary provided above
   - Note: what worked, what didn't, user corrections
   - This is your source of truth for session context

   **If session NOT in scope:** Skip session summary analysis entirely

2. CONTEXT POISONING PREVENTION:

   CRITICAL (if session in scope): Session summary contains BOTH good and bad approaches!

   - **Final code diff = ground truth** - What ended up in code is what worked
   - **User corrections** ("no not like that", "that didn't work") = failed approaches
   - **Abandoned code** - If something was written but not in final diff, it failed

   Separate into:
   - ✅ **What Worked** - In final code, no user corrections
   - ❌ **What Didn't Work** - Tried but abandoned, user corrected, failed
   - 💡 **Key Insight** - Why one approach won over others

   **If no session context:** Focus purely on code patterns and techniques visible in the diffs

3. ANALYZE & EXTRACT:

   For each learning, extract:
   - **Final Solution** (what worked)
   - **What we tried first** (what didn't work, if applicable)
   - **Why it failed** (if applicable)
   - **Key pattern/technique**
   - **When to apply**
   - **Trade-offs**
   - **Tags** (3-5: security, performance, php, javascript, api, etc.)

4. CATEGORIZE BY TAGS:
   For each primary tag (pick 1-2 most important):

5. UPDATE OR CREATE SKILLS:

   ⚠️ **CRITICAL: NEVER UPDATE SPECIALIST SKILLS** ⚠️

   Skills WITHOUT "learned-" prefix are MANUAL specialist skills (filemaker, lasso, ageraehandel, svg-animation, etc.)
   - These are NEVER updated by /learn command
   - They contain curated domain expertise, not session learnings
   - If tag matches a specialist skill name, create "learned-{tag}" instead

   **Example:** If tag is "filemaker", create "learned-filemaker", NOT "filemaker"

   For each primary tag:

   A. UPDATE/CREATE SKILL.MD (Lightweight):
      - Path MUST be: ~/.claude/skills/learned-{tag}/SKILL.md (note "learned-" prefix!)
      - If exists: Read existing, analyze for conflicts/overlaps, integrate intelligently
      - If not: create with frontmatter and structure
      - Add ONE-LINER summary for this learning under "## Available Learnings"
      - Format: ### {Date} - {Title}\n- **Use when:** {one sentence}\n- **Tags:** {tags}
      - Keep it minimal! Just enough to know when to read reference.md

   B. UPDATE/CREATE REFERENCE.MD (Full details):
      - Path MUST be: ~/.claude/skills/learned-{tag}/reference.md (note "learned-" prefix!)
      - **If file exists:**
        1. Read entire file first
        2. Analyze for conflicting or overlapping information
        3. If similar topic exists: UPDATE/MERGE that section instead of creating duplicate
        4. If new topic: append at end with separator
        5. Watch for contradictions - resolve by integrating insights
      - **If new file:** create with header
      - Add FULL learning entry (all details, code examples, etc.)
      - This is where all the heavy content goes

6. SKILL.MD FORMAT (LIGHTWEIGHT INDEX):

IMPORTANT: Keep SKILL.md lightweight to avoid context bloat!

```markdown
---
name: learned-{tag}
description: {Tag} patterns learned from real implementations
---

You have {tag} knowledge from actual implementations.

## Available Learnings

{For each learning, one-liner summary:}

### {Date} - {Title}
- **Use when:** {one sentence}
- **Tags:** {comma-separated}

{Repeat for all learnings in this skill}

---

**Full details:** To get complete information, code examples, and lessons learned, read:
`~/.claude/skills/learned-{tag}/reference.md`

Only read reference.md when you actually need the details for current task.
```

**Example:**
```markdown
---
name: learned-security
description: Security patterns learned from real implementations
---

You have security knowledge from actual implementations.

## Available Learnings

### 2025-01-15 - XSS Protection with DOMPurify
- **Use when:** Handling user input that will be rendered as HTML
- **Tags:** security, javascript, xss

### 2025-01-20 - SQL Injection Prevention with Prepared Statements
- **Use when:** Building database queries with user input
- **Tags:** security, database, php, sql

### 2025-01-25 - CSRF Token Validation Pattern
- **Use when:** Building forms that modify data
- **Tags:** security, forms, csrf, php

---

**Full details:** Read `~/.claude/skills/learned-security/reference.md` when needed.
```

This way:
- Session start: Only lightweight summaries loaded (~100 tokens per skill)
- When needed: Claude reads reference.md for full details (~2k tokens)
- Scales to 100+ learnings without context explosion
```

7. REFERENCE.MD FORMAT (WITH CONTEXT):
```markdown
# {Tag} Learnings

## {Date} - {Title}

**Final Solution:** {what ended up working}

**What Worked:**
- {successful pattern/technique}
- {why it worked}

**What We Tried First (If Applicable):**
- {failed approach}
- **WHY IT FAILED:** {reason}
- **LESSON:** {what we learned}

**When to Apply:** {conditions for using this}

**Trade-offs:** {considerations}

**Code Example:**
\`\`\`{language}
{relevant code snippet from final solution}
\`\`\`

**Tags:** {comma-separated tags}

---
```

8. CHECK FOR DUPLICATES:
   - Before adding a new learning, check if similar entry exists in reference.md
   - If similar topic exists: UPDATE existing entry rather than create duplicate
   - Look for: same technique, same problem domain, overlapping tags
   - Merge insights when appropriate

9. REPORT:
   - List which skills were created/updated
   - Show tags extracted
   - Summarize sources analyzed (changes/commit/session)
   - If session in scope: summarize what worked vs what didn't
   - Confirm skills will be loaded next session

IMPORTANT:
- Failed approaches are VALUABLE (if session in scope) - they prevent repeating mistakes
- Focus on reusable knowledge, not implementation details
- Extract WHY decisions were made
- Use actual code examples from the diffs
- Session summary (if in scope) provides rich context - use it!
- Context poisoning prevention is CRITICAL (when session in scope)
- When only analyzing code: focus on patterns, techniques, and architectural decisions visible in the changes
```

When the agent completes, you'll have skills that automatically load in future sessions!
```

### Steg 2: Testa

```bash
# Gör några ändringar i ett projekt
# Ha en session där ni löser ett problem tillsammans

# Kör learning extraction (flera alternativ):

# Default: uncommitted changes + session
/learn

# Eller bara session (utan kod)
/learn session

# Eller bara kod (utan session kontext)
/learn changes

# Eller committed kod + session
git commit -m "Your changes"
/learn commit session

# Eller specifik commit (retroaktiv learning)
/learn commit abc123

# Claude:
# 1. Sammanfattar sessionen (om session i scope)
# 2. Spawnar agent som läser kod och/eller session summary
# 3. Skapar/uppdaterar skills

# Verifiera att skills skapades
ls ~/.claude/skills/

# Läs en skill
cat ~/.claude/skills/learned-php/SKILL.md
cat ~/.claude/skills/learned-php/reference.md

# Starta ny Claude Code session
# Skills laddas automatiskt!
```

## Lazy Loading i Praktiken

### Scenario: 50 Security Learnings

**Without lazy loading:**
```
Session start:
  Loading skills...
  ✗ learned-security.md: 75,000 tokens (50 learnings × 1,500 tokens)
  ✗ Context limit reached before you even start!
```

**With lazy loading:**
```
Session start:
  Loading skills...
  ✓ learned-security/SKILL.md: ~2,500 tokens (50 × 50 tokens)
  ✓ Fast load, manageable context

User: "Need to sanitize HTML"

Claude: *Checks SKILL.md index*
        *Sees: "XSS Protection - Use when: handling user input as HTML"*
        *Decides it's relevant*

        Read ~/.claude/skills/learned-security/reference.md

        *Gets full XSS learning (1,500 tokens)*
        *Only this one learning, not all 50!*

        "I'll use DOMPurify based on our learning..."
```

**Token usage comparison:**

| Scenario | Without Lazy | With Lazy Loading |
|----------|--------------|-------------------|
| Session start (50 learnings) | 75,000 tokens | ~2,500 tokens |
| Using 1 learning | 75,000 tokens | 2,500 + 1,500 = 4,000 tokens |
| Using 3 learnings | 75,000 tokens | 2,500 + 4,500 = 7,000 tokens |

**Savings:** ~30x less context at session start!

## Exempel på usage

### Exempel 1: Efter att fixa en bug

```bash
You: "Just fixed an XSS vulnerability in the comment form"

# Make the fix and commit it

You: "/learn commit"

Claude spawns agent:
  - Analyzes the commit (no session context)
  - Extracts security pattern from code
  - Creates/updates ~/.claude/skills/learned-security/
  - Adds entry about XSS prevention with DOMPurify

# Or include session context if there was valuable discussion:
You: "/learn commit session"

Claude spawns agent:
  - Analyzes both commit AND session summary
  - Captures why this approach was chosen
  - Documents any failed attempts discussed

Next session:
  - learned-security skill loads automatically
  - Claude knows about this XSS pattern
  - Can proactively apply it to similar problems
```

### Exempel 2: Efter refactoring

```bash
You: "Refactored API controllers to use a base class"

# Commit the changes

You: "/learn commit"

Claude spawns agent:
  - Extracts the inheritance pattern
  - Tags: php, api, architecture
  - Updates learned-php and learned-api skills

Next session:
  - When building new controllers, Claude suggests the pattern
  - No need to remind Claude about the architecture
```

### Exempel 3: Komplex implementation

```bash
You: "Just implemented OAuth2 flow"

# After a long conversation where you:
# - Tried JWT first (didn't work for our use case)
# - Switched to OAuth2
# - Debugged token refresh issues

# Changes are uncommitted
You: "/learn"  # Default: changes + session

Claude:
  1. Summarizes session (JWT attempt, why it failed, OAuth2 solution)
  2. Spawns agent with summary + uncommitted changes
  3. Agent extracts:
     - OAuth2 pattern (what worked)
     - JWT pitfalls (what didn't)
     - Token refresh debugging steps
  4. Creates learned-authentication skill

# Alternative: Only extract from code without session context
You: "/learn changes"

Claude:
  - Analyzes only the code changes
  - Extracts OAuth2 implementation pattern
  - No context about why JWT didn't work (less rich learning)

# Or: Analyze old commit + current session
You: "/learn commit abc123 session"

Claude:
  - Analyzes specific commit from history
  - Includes current session discussion
  - Useful for retroactive learning extraction

Next session:
  - Claude knows OAuth2 approach
  - Won't suggest JWT for similar use cases
  - Remembers token refresh gotchas
```

## File Organization

Skills organiseras automatiskt per tag:

```
~/.claude/skills/
├── learned-security/
│   ├── SKILL.md
│   └── reference.md
│       - XSS prevention with DOMPurify (2025-01-15)
│       - SQL injection prevention (2025-01-20)
│       - CSRF token validation (2025-01-25)
│
├── learned-php/
│   ├── SKILL.md
│   └── reference.md
│       - Controller base class pattern (2025-01-16)
│       - Repository pattern for database (2025-01-22)
│
├── learned-api/
│   ├── SKILL.md
│   └── reference.md
│       - REST endpoint structure (2025-01-16)
│       - Error response format (2025-01-18)
│
└── learned-performance/
    ├── SKILL.md
    └── reference.md
        - Database query optimization (2025-01-19)
        - Caching strategy (2025-01-24)
```

## Hur Claude använder learnings

### Automatisk loading

Vid session start:
```
Loading skills...
✓ learned-security (5 entries)
✓ learned-php (3 entries)
✓ learned-api (4 entries)
✓ learned-performance (2 entries)
```

### Proaktiv användning

```
You: "Need to add a comment form"

Claude: *Har learned-security loaded*
        *Vet om XSS-sårbarheten från tidigare*

        "I'll implement the comment form with XSS protection using DOMPurify,
         similar to the pattern we learned from the previous fix..."

        *Implementerar med säkerhet från start*
```

### Ingen sökning behövs

Skillnad mot andra approaches:

**Utan MVP:**
```
You: "Add comment form"
Claude: *Implementerar*
You: "This has XSS vulnerability!"
Claude: "Oh sorry, let me fix that"
```

**Med MVP:**
```
You: "Add comment form"
Claude: *Ser learned-security skill*
        *Applicerar XSS-skydd automatiskt*
        "Implemented with DOMPurify XSS protection"
```

## Best Practices

### När ska du köra /learn?

**Kör efter:**
- ✅ Buggfixar (speciellt säkerhet/performance)
- ✅ Nya patterns implementerade
- ✅ Arkitektoniska beslut
- ✅ Refactorings
- ✅ Problemlösningar som tog tid

**Skippa efter:**
- ❌ Typos och trivial formatting
- ❌ Merge commits
- ❌ Genererad kod (build output, etc.)

### Håll skills fokuserade

Bra tags:
- `security` - Säkerhetspatterns
- `performance` - Optimeringar
- `php` - PHP-specifika patterns
- `api` - API design
- `database` - DB patterns

Undvik för breda tags:
- ~~`general`~~ - För brett
- ~~`code`~~ - Meningslöst
- ~~`fix`~~ - Inte kategoriskt

### Håll skills optimerade

**SKILL.md maintenance:**

Om en skill får många (50+) learnings:
```bash
# SKILL.md börjar bli lång → Split into sub-skills
# learned-security → learned-security-xss, learned-security-sql, etc.

# Or: Archive old entries
# Edit SKILL.md: Ta bort gamla one-liners
# Keep only most recent/relevant 20-30 entries in index
```

**reference.md maintenance:**

```bash
# Review periodically
vim ~/.claude/skills/learned-php/reference.md

# Remove:
# - Obsolete learnings (replaced by better patterns)
# - Duplicates
# - Rarely used entries (usage_count could track this in future)

# Archive instead of delete:
mv ~/.claude/skills/learned-php/reference.md \
   ~/.claude/skills/learned-php/reference-archive-2025-01.md
# Start fresh reference.md
```

**Context budget rule:**

- Each skill index (SKILL.md): Max ~1,000 tokens (50-70 learnings)
- Total skills at start: Max ~10,000 tokens (~10 skills)
- If exceeding: Archive or split skills

**This ensures:**
- Fast session start
- Plenty of context left for actual work
- Skills remain manageable

## Kostnader

**Per /learn körning:**
- Task agent analys: ~2k-5k tokens input, ~1k-2k tokens output
- Cost: ~$0.02-0.05 per learning extraction

**Månadsvis (20 learnings):**
- ~$0.40-1.00

**Nästan gratis jämfört med full system!**

## Troubleshooting

### Slash command finns inte

```bash
# Kontrollera att filen skapades
ls ~/.claude/commands/learn.md

# Om inte, skapa den (kopiera från ovan)
```

### Agent skapar inte skills

Kontrollera agent output:
- Får agenten error vid git diff?
- Har agenten Write access till ~/.claude/skills/?
- Kolla om skills skapades: `ls ~/.claude/skills/`

**Vanligaste orsaken: Missing permissions i projektets settings**

```bash
# Kontrollera om projektets .claude/settings.local.json har skills-permissions
cat .claude/settings.local.json

# Om inte, lägg till (se "Steg 1.5: Konfigurera permissions" ovan)
```

Agenten behöver explicit Write-tillåtelse i **projektets** lokala settings för att skapa skills i `~/.claude/skills/`.

### Skills laddas inte

```bash
# Kontrollera att skills finns
ls -la ~/.claude/skills/*/SKILL.md

# Kontrollera SKILL.md format (måste ha frontmatter)
cat ~/.claude/skills/learned-php/SKILL.md

# Starta om Claude Code
```

### Git diff är tom

```bash
# För uncommitted changes
git status  # Kontrollera att det finns ändringar
git add .   # Stage changes först om du vill (git diff visar både staged och unstaged)

# Alternativ:
# 1. Använd commit scope istället
git commit -m "Your changes"
/learn commit

# 2. Eller analysera bara session utan kod
/learn session

# 3. Eller analysera äldre commit
/learn commit abc123
```

## Framtida förbättringar (optional)

När MVP fungerar väl kan du:

1. **Auto-consolidering** - Script som mergar duplicerade entries
2. **Confidence scores** - Markera hur ofta pattern använts
3. **Cross-references** - Länka relaterade learnings
4. **Search command** - `/search-learnings <query>`
5. **Stats** - `/learn-stats` visar antal learnings per tag

Men börja simpelt. Det här räcker!

## Migration till Full System

Om du samlar 50+ learnings och behöver:
- Semantic search
- Automatic SessionEnd trigger
- Usage tracking
- Skill auto-generation thresholds

→ Då är det dags för full system (se IMPLEMENTATION.md)

MVP-skills kan enkelt importeras till full system's databas.

## Sammanfattning

**Setup:**
- 1 fil: `~/.claude/commands/learn.md`
- 10 minuter

**Användning:**
- Lös problem tillsammans → `/learn` → Claude sammanfattar + extraherar → Skills uppdateras
- Nästa session → Skills auto-loaded → Claude kommer ihåg

**Resultat:**
- Claude lär sig från sessionskontext + kodändringar
- Både framgångsrika och misslyckade approaches sparas
- Applicerar learnings proaktivt
- Ingen manuell sökning eller påminnelser behövs

**Detta är den enklaste vägen till self-learning Claude Code!**
