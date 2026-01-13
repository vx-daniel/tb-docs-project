---
name: brainstorming
description: >-
  Use when request is vague, complex, or needs design exploration BEFORE
  implementation. Ask questions one at a time. Do NOT use for clear, mechanical
  tasks. Output: design ready for OpenSpec proposal.
---

# Brainstorming

Refine rough ideas into concrete designs through iterative questioning.

> **Announce:** "I'm using brainstorming to explore and clarify this idea before implementation."

## Iron Law

```
ONE QUESTION AT A TIME - NO ASSUMPTIONS
```

If you're unsure about something, ASK. Do not guess or assume.

## Process

### Phase 1: Understand Context

1. Check project state: relevant files, recent commits, existing patterns
2. Read `openspec/project.md` for conventions
3. Run `openspec list --specs` to see current capabilities
4. Identify what already exists vs what's new

### Phase 2: Clarify Requirements

Ask questions ONE AT A TIME:

- Prefer multiple choice when possible
- Open-ended for exploration
- Wait for answer before next question

Focus on:
- What is the actual goal?
- What are the constraints?
- What does success look like?
- What should NOT happen?

### Phase 3: Explore Approaches

Present 2-3 approaches with tradeoffs:

```markdown
## Approach A: [Name]
- How it works: ...
- Pros: ...
- Cons: ...

## Approach B: [Name]
- How it works: ...
- Pros: ...
- Cons: ...

**My recommendation:** Approach A because [reason].
```

ASK which approach the user prefers.

### Phase 4: Validate Design

Present design in sections of 200-300 words:
1. Present section
2. Ask: "Does this look right?"
3. Adjust based on feedback
4. Move to next section

Cover:
- Architecture / structure
- Key components
- Data flow
- Error handling
- Testing approach

## Output

When design is validated, summarize:

```markdown
## Design Summary

**Goal:** [One sentence]

**Approach:** [Which approach was chosen]

**Key decisions:**
- [Decision 1]
- [Decision 2]

**Next step:** Create OpenSpec proposal
```

## REQUIRED SUB-SKILL

After design is approved:
→ Load `openspec-propose` skill to create the change proposal

## Red Flags - STOP and Reset

If you catch yourself:
- Asking multiple questions at once
- Assuming answers without asking
- Jumping to implementation
- Presenting only one approach
- Writing code during brainstorming

STOP. Return to Phase 2.

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "This is obvious, no need to ask" | Obvious to you ≠ obvious to user. Ask anyway. |
| "I'll figure it out as I code" | That's how you build the wrong thing. Design first. |
| "Just one small assumption" | Small assumptions compound into wrong designs. |
| "User is busy, I'll minimize questions" | Wrong implementation wastes more time than questions. |
