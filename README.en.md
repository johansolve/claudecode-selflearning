# Claude Code Self-Learning System

[Svenska](README.md) | **English**

A self-learning system that transforms Claude Code from "Execute & Forget" to "Execute & Learn".

## Overview

This system enables Claude Code to extract, store, and reuse knowledge from previous sessions. When you solve problems with Claude, those learnings can be saved as skills that automatically load in future sessions.

**Current implementation:** A simple, working solution based on Claude Code's built-in skill system.

## How it works

```
Session: You and Claude solve a problem
              ↓
You: /learn
              ↓
Claude summarizes the session + analyzes code changes
              ↓
Extracts patterns, techniques, and learnings
              ↓
Saves as skills (global or project-specific)
              ↓
Next session: Skills load automatically
              ↓
Claude applies learnings proactively
```

## The /learn command

### What it does

1. Summarizes the current session (discussions, what worked/didn't work)
2. Analyzes code changes (uncommitted, commits, or both)
3. Extracts patterns, techniques, and knowledge
4. Categorizes with tags (security, performance, php, etc.)
5. Prompts for storage location (Global or Project)
6. Creates/updates skills that automatically load in future sessions

### Scope options

Three sources can be analyzed:
- **`changes`** - Uncommitted changes (git diff)
- **`commit [hash]`** - Specific commit (default: HEAD)
- **`session`** - Session conversation and learnings

**Default:** `changes session` (if no arguments given)

### Usage

```bash
/learn                          # Default: changes + session
/learn session                  # Session learnings only
/learn changes                  # Uncommitted changes only
/learn commit                   # Last commit only (HEAD)
/learn commit abc123            # Specific commit only
/learn changes session          # Uncommitted + session
/learn commit session           # Last commit + session
/learn changes commit           # Uncommitted + last commit
/learn changes commit session   # All three sources
```

### Storage location

Each time you run `/learn`, you choose where to save learnings:

- **Global** - `~/.claude/skills/learned-{tag}/` - Available in all projects
- **Project** - `{project}/.claude/skills/learned-{tag}/` - Project-specific

**Important precedence note:** If the same skill exists in both locations, the project version takes priority - the global one is completely ignored (no merge). Therefore, avoid having the same skill name in both places. Choose one location per category:
- Universal patterns (security, performance) → Global
- Project-specific architecture → Project

## Installation

### Step 1: Create the slash command

Copy `commands/learn.md` to `~/.claude/commands/learn.md`

```bash
mkdir -p ~/.claude/commands
cp commands/learn.md ~/.claude/commands/learn.md
```

### Step 2: Configure permissions (per project)

Add to the project's `.claude/settings.local.json`:

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

**Done!** You can now use `/learn` in your Claude Code sessions.

## Key features

### Context poisoning prevention

Session history contains both successful and failed approaches. The system separates:

- **What Worked** - In final code, no user corrections
- **What Didn't Work** - Tried but abandoned, user corrected
- **Key Insight** - Why one approach won over others

Failed approaches are valuable - they prevent repeating the same mistakes.

### Intelligent updating

Skills are kept up-to-date over time:

- **New topics** are automatically created as new entries
- **Existing topics** are updated and enriched with new insights
- **Duplication is avoided** by merging similar learnings
- **Contradictions are resolved** by integrating and updating existing knowledge

When you run `/learn`, the system analyzes whether similar knowledge already exists. Instead of creating duplicate entries, existing ones are updated with new information, keeping the knowledge base clean and current.

### Lazy loading

Skills are organized for optimal context handling:

- **SKILL.md** (Lightweight index) - ~50 tokens per learning, loaded at session start
- **reference.md** (Full details) - ~1500 tokens per learning, read on-demand

This scales to 100+ learnings without context explosion:
- 100 learnings × 50 tokens = ~5,000 tokens at session start
- Full details are only read when needed

### Proactive application

```
You: "Add a comment form"

Claude: *Sees in learned-security skill*
        "XSS Protection - Use when: handling user input as HTML"
        *Reads reference.md for details*

        "I'm implementing with DOMPurify XSS protection based on
         previous learnings..."
```

## Examples

### After a bugfix

```bash
# Fix XSS vulnerability, commit
git commit -m "Fix XSS in comment form"

# Extract learning from commit
/learn commit

# → Creates learned-security skill with XSS pattern
```

### After problem-solving

```bash
# Have a session where you:
# - Tried JWT first (didn't work)
# - Switched to OAuth2
# - Debugged token refresh

# Extract from both code and session
/learn

# → Saves both the OAuth2 pattern AND why JWT didn't work
```

### Retroactive learning

```bash
# Extract from older commit
/learn commit abc123

# Or: combine old commit with current session
/learn commit abc123 session
```

## File organization

```
~/.claude/skills/
├── learned-security/
│   ├── SKILL.md          # Lightweight index
│   └── reference.md      # Full details
├── learned-php/
│   ├── SKILL.md
│   └── reference.md
└── learned-performance/
    ├── SKILL.md
    └── reference.md
```

## Benefits

- **No setup complexity** - Just one slash command file
- **No external dependencies** - Uses only Claude Code
- **Rich knowledge extraction** - Session context + code changes
- **Flexible storage** - Global or project-specific
- **Scalable** - Lazy loading handles 100+ learnings
- **Proactive** - Claude applies learnings automatically

## Troubleshooting

### Slash command not found

```bash
ls ~/.claude/commands/learn.md
# If missing, copy from commands/learn.md
```

### Agent doesn't create skills

Most common cause: Missing permissions in project settings.

```bash
# Check project settings
cat .claude/settings.local.json

# Make sure Write(~/.claude/skills/**) is in the allow list
```

### Skills don't load

```bash
# Check that skills exist
ls -la ~/.claude/skills/*/SKILL.md

# Check SKILL.md format (must have frontmatter)
cat ~/.claude/skills/learned-php/SKILL.md

# Restart Claude Code
```

### Git diff is empty

```bash
git status  # Check that there are changes

# Alternatives:
/learn commit        # Use last commit instead
/learn session       # Analyze session only
```

## Future development

For users with 50+ learnings who need more advanced functionality, there's the possibility to expand to a full system with:

- **Semantic search** - Vector embeddings with FAISS for smart search
- **Automatic trigger** - SessionEnd hook instead of manual `/learn`
- **Usage tracking** - Track how often patterns are used
- **MCP Server** - Dedicated tools for knowledge search

See `ADVANCED_IMPLEMENTATION.md` for technical specification and implementation guide.

## License

MIT

## Author

Johan Sölve (ideas & opinions)
Claude Code (actual work)
