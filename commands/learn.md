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
5. Prompts you to choose storage location (Global or Project)
6. Updates or creates skills in chosen location
7. Skills are automatically loaded in future sessions

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

## Step 2: Ask User for Storage Location

Before spawning the agent, ask the user where to save the learnings using AskUserQuestion:

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

Store the user's answer - you'll need to pass it to the agent in the next step.

## Step 3: Spawn Task Agent

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

STORAGE LOCATION:
{Insert user's choice from Step 2:}
- If "Global": Save skills in ~/.claude/skills/learned-{tag}/
- If "Project": Save skills in {current working directory}/.claude/skills/learned-{tag}/

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

   **IMPORTANT: Use the storage location specified in STORAGE LOCATION section above.**
   - Global → base path: ~/.claude/skills/learned-{tag}/
   - Project → base path: {working directory}/.claude/skills/learned-{tag}/

   For each primary tag:

   A. UPDATE/CREATE SKILL.MD (Lightweight):
      - Path: {base path from STORAGE LOCATION}/SKILL.md (note "learned-" prefix!)
      - If exists: Read existing, analyze for conflicts/overlaps, integrate intelligently
      - If not: create with frontmatter and structure
      - Add ONE-LINER summary for this learning under "## Available Learnings"
      - Format: ### {Date} - {Title}\n- **Use when:** {one sentence}\n- **Tags:** {tags}
      - Keep it minimal! Just enough to know when to read reference.md

   B. UPDATE/CREATE REFERENCE.MD (Full details):
      - Path: {base path from STORAGE LOCATION}/reference.md (note "learned-" prefix!)
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

Only read reference.md when you actually need the details for current task.

7. REFERENCE.MD FORMAT (WITH CONTEXT):

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
```{language}
{relevant code snippet from final solution}
```

**Tags:** {comma-separated tags}

---

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
