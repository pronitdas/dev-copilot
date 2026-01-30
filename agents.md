# Agent Coordination Directives

## Intent Gate

### Before EVERY Response: Classify Request Type

| Type | Signal | Action |
|------|--------|--------|
| **Trivial** | Single file, known location, direct answer | Direct tools only |
| **Explicit** | Specific file/line, clear command | Execute directly |
| **Exploratory** | "How does X work?", "Find Y" | Fire explore agents |
| **Open-ended** | "Improve", "Refactor", "Add feature" | Assess codebase first |
| **Ambiguous** | Unclear scope, multiple interpretations | Ask clarifying question |

### Key Triggers (Check Before Classification)

- External library mentioned → fire `librarian` background agent
- 2+ modules involved → fire `explore` background agent
- "Look into" + "create PR" → Full implementation cycle expected

## Tool & Agent Selection

### When to Use Direct Tools

- Known file location
- Single keyword/pattern search
- Specific file/line edits
- Simple refactoring within one file

### When to Use Explore Agent

- Multiple search angles needed
- Unfamiliar module structure
- Cross-layer pattern discovery
- "How does X work?"

### When to Use Librarian Agent

- Unfamiliar libraries involved
- Need official documentation
- Looking for external examples
- "How do I use [library]?"

### When to Use Oracle Agent

- Complex architecture decisions
- Multi-system tradeoffs
- After 2+ failed fix attempts
- High-difficulty design patterns

## Delegation Protocol

### Category Selection

| Category | Domain / Best For |
|----------|-------------------|
| `visual-engineering` | Frontend, UI/UX, design, styling, animation |
| `ultrabrain` | Deep logical reasoning, complex architecture |
| `artistry` | Creative/artistic tasks, novel ideas |
| `quick` | Trivial tasks, single file changes |
| `unspecified-low` | Low effort tasks |
| `unspecified-high` | High effort tasks |
| `writing` | Documentation, prose, technical writing |

### Skill Evaluation Protocol

For EVERY skill, ask: "Does this skill's expertise domain overlap with my task?"

**Include skill if:** Task domain overlaps with skill domain

**Omit skill if:** Provide explicit justification

### Delegation Pattern

```typescript
delegate_task(
  category="[selected-category]",
  load_skills=["skill-1", "skill-2"],
  prompt="..."
)
```

**MUST INCLUDE** session_id for multi-turn tasks:

```typescript
delegate_task(
  session_id="{session_id}",
  prompt="Fix: {specific error}"
)
```

## Background Execution

### Parallel Execution (DEFAULT)

```typescript
// CORRECT: Always background, always parallel
delegate_task(subagent_type="explore", run_in_background=true, prompt="Find auth patterns...")
delegate_task(subagent_type="librarian", run_in_background=true, prompt="Lookup JWT docs...")

// WRONG: Blocking/synchronous
delegate_task(..., run_in_background=false)
```

### Background Result Collection

1. Launch parallel agents → receive task_ids
2. Continue immediate work
3. When results needed: `background_output(task_id="...")`
4. Before final answer: `background_cancel(all=true)`

## Search Stop Conditions

STOP searching when:

- Enough context to proceed confidently
- Same information appearing across sources
- 2 iterations yielded no new useful data
- Direct answer found

**DO NOT over-explore.**

## Pre-Implementation

1. If task has 2+ steps → Create todo list IMMEDIATELY
2. Mark current task `in_progress` before starting
3. Mark `completed` IMMEDIATELY after done
4. Use todos to track ALL work

## Verification Requirements

| Action | Required Evidence |
|--------|-------------------|
| File edit | `lsp_diagnostics` clean on changed files |
| Build command | Exit code 0 |
| Test run | Pass (or note pre-existing failures) |
| Delegation | Agent result received and verified |

**NO EVIDENCE = NOT COMPLETE.**

## Failure Recovery

### After 3 Consecutive Fix Failures

1. STOP all further edits immediately
2. REVERT to last known working state
3. DOCUMENT what was attempted and failed
4. CONSULT Oracle with full failure context
5. If Oracle cannot resolve → ASK USER before proceeding

### Never

- Leave code in broken state
- Continue hoping it'll work
- Delete failing tests to "pass"
- Use `as any`, `@ts-ignore`, `@ts-expect-error`

## Session Continuity

### When to Use session_id

| Scenario | Action |
|----------|--------|
| Task failed/incomplete | session_id with "Fix: {error}" |
| Follow-up question | session_id with "Also: {question}" |
| Multi-turn with same agent | Always use session_id |

**Why session_id is critical:**
- Subagent has FULL conversation context
- No repeated file reads or exploration
- Saves 70%+ tokens on follow-ups

## Prompt Structure (MANDATORY)

Every delegation prompt MUST include:

```
1. TASK: Atomic, specific goal
2. EXPECTED OUTCOME: Concrete deliverables with success criteria
3. REQUIRED TOOLS: Explicit tool whitelist
4. MUST DO: Exhaustive requirements
5. MUST NOT DO: Forbidden actions
6. CONTEXT: File paths, existing patterns, constraints
```

## Ambiguity Protocol

| Situation | Action |
|-----------|--------|
| Single valid interpretation | Proceed |
| Multiple interpretations, similar effort | Proceed with reasonable default, note assumption |
| Multiple interpretations, 2x+ effort difference | **MUST ask** |
| Missing critical info | **MUST ask** |
| User's design seems flawed | **MUST raise concern** |

### When Challenging User

```
I notice [observation]. This might cause [problem] because [reason].
Alternative: [suggestion].
Should I proceed with your original request, or try the alternative?
```

## Codebase Assessment (Open-ended Tasks)

### Quick Assessment

1. Check config files: linter, formatter, type config
2. Sample 2-3 similar files for consistency
3. Note project age signals

### State Classification

| State | Signals | Behavior |
|-------|---------|----------|
| **Disciplined** | Consistent patterns, configs present, tests | Follow existing style strictly |
| **Transitional** | Mixed patterns, some structure | Ask: "I see X and Y patterns. Which to follow?" |
| **Legacy/Chaotic** | No consistency, outdated patterns | Propose: "No clear conventions. I suggest [X]. OK?" |
| **Greenfield** | New/empty project | Apply modern best practices |

## Completion Criteria

A task is complete when:

- [ ] All planned todo items marked done
- [ ] Diagnostics clean on changed files
- [ ] Build passes (if applicable)
- [ ] User's original request fully addressed
- [ ] All background tasks cancelled

## Available Skills

| Skill | When to Use |
|-------|-------------|
| `playwright` | Browser-related tasks |
| `frontend-ui-ux` | UI/UX design without mockups |
| `git-master` | ANY git operations |
| `dev-browser` | Browser automation with persistent state |
| `frontend-design-system` | React + Tailwind + shadcn/ui |
| `rust-tauri` | Rust code, Tauri apps |
| `fullstack-gis` | Geospatial data, maps, spatial analysis |
| `typescript-best-practices` | TypeScript, generics, API types |
