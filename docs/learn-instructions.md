# /learn Command Instructions

Analyze code changes and/or session learnings, then save as skills for future sessions.

## Philosophy: Compounding Knowledge

**The goal:** Build a knowledge base that makes Claude smarter over time.
- Each session adds patterns, techniques, and insights
- Knowledge compounds - today's learnings build on yesterday's
- The more Claude learns, the better it gets at solving new problems
- Balance: Save valuable knowledge, but avoid noise that dilutes the signal

## What this does

1. Summarizes the current session (what was discussed, tried, worked/failed)
2. Analyzes code changes (uncommitted, commits, or both)
3. Extracts patterns, techniques, and knowledge
4. Categorizes by tags (security, performance, php, etc.)
5. Prompts user to choose storage location (Global or Project)
6. Updates or creates skills in chosen location
7. Skills are automatically loaded in future sessions - **Claude gets smarter over time**

## Scope Options

Three sources can be analyzed:
- **`changes`** - Uncommitted changes (git diff)
- **`commit [hash]`** - Specific commit (default: HEAD)
- **`session`** - Session conversation and learnings

**Default:** `changes session` (if no arguments given)
**When specified:** Only analyze what you specify

## Storage Location

Every time you run /learn, you'll be asked where to save the learnings:

- **Global** - Save in ~/.claude/skills/learned-{tag}/ (available in all projects)
- **Project** - Save in {current-project}/.claude/skills/learned-{tag}/ (project-specific)

Choose based on whether the learnings are universally applicable or specific to the current project.

## Usage

```bash
/learn                          # Default: changes + session (destination prompted)
/learn session                  # Only session learnings (destination prompted)
/learn changes                  # Only uncommitted changes (destination prompted)
/learn commit                   # Only last commit (HEAD) (destination prompted)
/learn commit abc123            # Only specific commit (destination prompted)
/learn changes session          # Uncommitted + session (destination prompted)
/learn commit session           # Last commit + session (destination prompted)
/learn changes commit           # Uncommitted + last commit (destination prompted)
/learn changes commit session   # All three sources (destination prompted)
/learn changes commit abc123    # Uncommitted + specific commit (destination prompted)
```

**Note:** Every run will prompt you to choose between Global or Project storage.

---

**INSTRUCTIONS BELOW ARE FOR CLAUDE IN THE MAIN SESSION**

---

## Step 1: Ask User for Storage Location

FIRST, ask the user where to save the learnings using AskUserQuestion (this affects what's worth saving):

```json
{
  "question": "Var vill du spara dessa learnings?",
  "header": "Destination",
  "multiSelect": false,
  "options": [
    {
      "label": "Global",
      "description": "Spara i ~/.claude/skills/ - tillgängligt i alla projekt"
    },
    {
      "label": "Project",
      "description": "Spara i detta projekts .claude/skills/ - projektspecifikt"
    }
  ]
}
```

Store the user's answer - you'll need it for the session summary and agent prompt.

## Step 2: Session Summary (CRITICAL)

Before spawning the Task agent, YOU (Claude in the main session) must create a focused session summary.
You have access to the full conversation history - the spawned agent does NOT.

**Philosophy:** Capture valuable knowledge to build expertise over time. Focus on patterns that compound.

**Important:** Consider the storage location from Step 1 when evaluating what's worth saving:
- **Global storage:** Focus on patterns applicable across different codebases
- **Project storage:** Can include project-specific patterns and conventions

### Summary Template

Create a CONCISE summary using this format:

```
SESSION SUMMARY
===============

## Problem
[One sentence: What core problem did we solve?]

## Final Approach
[What pattern/technique ended up working? Why this over alternatives?]

## Key Insight
[The ONE lesson worth remembering - what would you do differently next time?]

## Anti-Pattern (if discovered)
[What looked promising but failed? Why? Under what conditions should this be avoided?]
```

### Quality Checklist - IS THIS WORTH SAVING?

Before spawning the agent, apply the SELECTIVITY FILTER:
- [ ] **Specific**: Contains concrete pattern/technique, not vague "it depends"
- [ ] **Actionable**: Can be applied to future similar problems
- [ ] **Generalizable**: Useful beyond this exact implementation (Global) OR useful for this project (Project)
- [ ] **Valuable**: Teaches something that makes future Claude sessions better

**DO SAVE - Green flags:**
- Patterns/techniques discovered through trial and error
- Anti-patterns found through actual failure (not hypothetical)
- Non-obvious solutions to common problems
- Domain-specific best practices learned from experience
- Insights about "why X works better than Y in context Z"
- **For Project storage:** Project-specific conventions, architecture decisions, domain knowledge

**DON'T SAVE - Red flags:**
- Raw implementation details without extractable pattern
- Standard framework usage documented in official docs
- "We tried X then Y then Z" logs without insights about WHY

**IMPORTANT:** If nothing meets the selectivity criteria, DON'T spawn the agent. Instead, inform the user:
"No valuable learnings found in this session that meet the quality criteria. This is normal for routine tasks - not every session produces reusable knowledge."

## Step 3: Spawn Task Agent

Use the Task tool to spawn a general-purpose agent with this prompt:

---

**PROMPT FOR THE SPAWNED AGENT STARTS BELOW:**

---

```
You are an agent building a compounding knowledge base that makes future Claude sessions smarter.

MISSION: Extract valuable patterns and insights from this session. Each learning should:
- Make future Claude sessions better at solving similar problems
- Build on existing knowledge in the skills database
- Be worth remembering and reusing across sessions

Balance quality with growth - save learnings that teach valuable lessons, but avoid noise.

SCOPE: {Parse arguments and specify what to analyze:}
- If no arguments: "changes + session"
- If arguments given: only what's specified
- Possible: "changes", "commit [hash]", "session", or combinations

EXAMPLES:
- /learn → analyze: changes + session
- /learn session → analyze: session only
- /learn changes commit abc123 → analyze: uncommitted changes + commit abc123

STORAGE LOCATION:
{Insert user's choice from Step 1:}
- If "Global": Save skills in ~/.claude/skills/learned-{tag}/
- If "Project": Save skills in {current working directory}/.claude/skills/learned-{tag}/

SESSION SUMMARY (from main session):
{Insert the session summary you created in Step 2 here - or skip this section if "session" not in scope}

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
   - Extract the key insight and anti-pattern (if present)
   - This is your source of truth for session context

   **If session NOT in scope:** Skip session summary analysis entirely

2. IDENTIFY VALUABLE LEARNINGS:

   Look for knowledge worth adding to your knowledge base:

   **DO SAVE - Green flags:**
   - Patterns/techniques discovered through trial and error
   - Anti-patterns found through actual failure (these are VALUABLE)
   - Non-obvious solutions to common problems
   - Domain-specific best practices learned from experience
   - Insights about "why X works better than Y in context Z"
   - Debugging techniques that solved real issues
   - Performance optimizations with measurable impact

   **DON'T SAVE - Red flags:**
   - Raw data dumps or implementation details without patterns
   - Standard framework usage documented in official docs
   - Chronological logs without insights about WHY
   - Hypothetical "this might work" without validation

3. APPLY QUALITY FILTER:

   For EACH potential learning, verify:
   - [ ] **Specific**: Contains concrete pattern/technique, not vague advice
   - [ ] **Actionable**: Can be applied to future similar problems
   - [ ] **Generalizable**: Useful beyond this exact implementation (or this project if Project storage)
   - [ ] **Valuable**: Teaches something that makes future Claude sessions better

   If learning meets 3+ criteria, save it. If only 1-2, consider discarding unless exceptionally valuable.

   **CRITICAL:** If NO learnings pass the quality filter, STOP and report:
   "No valuable learnings found that meet quality criteria. Analyzed [sources], but nothing reached the threshold for saving. This is normal for routine implementation tasks."

4. EXTRACT PATTERNS (for learnings worth saving):

   For each learning, distill to:
   - **Pattern/Technique** (one sentence - what's the reusable approach?)
   - **When to Apply** (one sentence - under what conditions?)
   - **Why It Works** (one sentence - the core insight)
   - **Anti-Pattern** (if discovered - what to avoid and why)
   - **Tags** (1-2 primary: security, performance, php, javascript, etc.)

   **Prioritize principles over details. Examples should illustrate patterns, not be the learning itself.**

5. ESTIMATE VALUE:
   Count learnings that passed the quality filter:
   - 1-3 learnings: Common for focused sessions
   - 4-6 learnings: Good for complex/exploratory sessions
   - 7-10 learnings: Acceptable if session covered multiple domains
   - 10+ learnings: Review - might be saving implementation details instead of patterns

6. UPDATE OR CREATE SKILLS:

   ⚠️ **CRITICAL: NEVER UPDATE SPECIALIST SKILLS** ⚠️

   Skills WITHOUT "learned-" prefix are MANUAL specialist skills (filemaker, lasso, ageraehandel, svg-animation, etc.)
   - These are NEVER updated by /learn command
   - They contain curated domain expertise, not session learnings
   - If tag matches a specialist skill name, create "learned-{tag}" instead

   **Example:** If tag is "filemaker", create "learned-filemaker", NOT "filemaker"

   **IMPORTANT: Use the storage location specified in STORAGE LOCATION section above.**
   - Global → base path: ~/.claude/skills/learned-{tag}/
   - Project → base path: {working directory}/.claude/skills/learned-{tag}/

   For each primary tag:

   A. UPDATE/CREATE SKILL.MD (Lightweight index):
      - Path: {base path from STORAGE LOCATION}/SKILL.md (note "learned-" prefix!)
      - If exists: Read existing, check for duplicates/overlaps
      - If not: create with frontmatter and structure
      - Add ONE-LINER for this learning under "## Available Learnings"
      - Format: ### {Date} - {Title}\n- **Use when:** {one sentence}\n- **Tags:** {tags}
      - Keep it MINIMAL! Just a trigger to know when to read reference.md

   B. UPDATE/CREATE REFERENCE.MD (Focused details):
      - Path: {base path from STORAGE LOCATION}/reference.md (note "learned-" prefix!)
      - **If file exists:**
        1. Read entire file first
        2. Check for duplicate/overlapping topics
        3. If similar exists: UPDATE/MERGE instead of creating duplicate
        4. If truly new: append with separator
      - **If new file:** create with header
      - Add FOCUSED learning entry (pattern + minimal example)
      - **Focus on WHY and WHEN, not exhaustive HOW**

7. SKILL.MD FORMAT (LIGHTWEIGHT INDEX):

IMPORTANT: Keep SKILL.md lightweight to avoid context bloat!

NOTE: This file will be read by future Claude sessions. Use "you" to refer to that future Claude.

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
`{base path from STORAGE LOCATION}/reference.md`

Only read reference.md when you need the details for the current task.

8. REFERENCE.MD FORMAT (FOCUSED):

# {Tag} Learnings

## {Date} - {Title}

**Pattern:** {One sentence - the reusable technique/approach}

**When to Apply:** {One sentence - conditions where this pattern applies}

**Why It Works:** {One sentence - the core insight/principle}

**Anti-Pattern (if applicable):** {What to avoid and why - ONLY if teaches important lesson}

**Supersedes (if applicable):** {Brief note if this replaces earlier advice}

**Minimal Example:**
```{language}
{Shortest code snippet that illustrates the pattern - 5-10 lines max}
```

**Tags:** {1-2 primary tags}

---

**QUALITY CHECK before saving:**
- **For Global:** Can this be applied to different codebases? (If no → don't save)
- **For Project:** Will this be useful for future work in THIS project? (If no → don't save)
- Does it teach a principle/pattern, not just document an implementation? (If no → don't save)
- Would future Claude sessions find this useful in 3 months? (If no → don't save)

9. CHECK FOR DUPLICATES AND CONFLICTS:
   - Before adding a new learning, check if similar entry exists in reference.md
   - If similar topic exists: UPDATE/MERGE instead of creating duplicate
   - Avoid accumulating redundant or overlapping learnings

   **CONFLICT RESOLUTION - New knowledge takes precedence:**
   - If new learning CONTRADICTS existing: REPLACE the old with new
   - If new learning EXTENDS existing: MERGE and update
   - If new learning is MORE SPECIFIC: Keep both (general + specific case)

   **How to identify conflicts:**
   - Same problem, different recommended solution
   - "Always do X" vs "Avoid X in context Y"
   - Performance/security advice that contradicts earlier advice

   **When replacing:**
   - Remove outdated pattern completely (don't keep "historical" versions)
   - Update the date to reflect when knowledge was revised
   - Optionally note in "Supersedes" field what was replaced

10. REPORT:

   **If learnings were saved:**
   - Number of learnings saved
   - Which skills were created/updated
   - Primary tags used
   - Sources analyzed (changes/commit/session)
   - Confirm skills will auto-load next session

   **If NO learnings were saved:**
   - State clearly: "No valuable learnings found that meet quality criteria"
   - Mention what was analyzed (changes/commit/session)
   - Reassure: "This is normal for routine tasks - not every session produces reusable knowledge"

FINAL REMINDER - COMPOUNDING KNOWLEDGE:
- Each valuable learning makes future Claude sessions smarter
- Anti-patterns from actual failures are WORTH saving - they prevent repeating mistakes
- Focus on PRINCIPLES that transfer across problems
- Code examples should ILLUSTRATE patterns, not BE the learning itself
- Balance: Save learnings that build expertise, but avoid noise that dilutes signal
- Knowledge compounds - the more Claude learns, the better it gets
```

---

**END OF AGENT PROMPT**

---

When the agent completes, skills will automatically load in future Claude sessions!
