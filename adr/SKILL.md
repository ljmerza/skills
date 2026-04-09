---
name: adr
description: Create Architecture Decision Records (ADRs) with interactive option exploration. Use this skill whenever the user invokes /adr, discusses architecture decisions, evaluates technology choices, asks "should we use X or Y", compares technical approaches, or needs to document a design decision. Also triggers when the user mentions ADRs, architecture records, or wants to weigh pros and cons of technical options before implementing.
---

# ADR Skill

Create Architecture Decision Records through an interactive workflow: research the project, propose options with pros/cons, walk through trade-offs with the user, record the decision, then optionally implement.

The user's argument is available as `$ARGUMENTS`.

## Workflow

### Step 1: Parse Arguments & Setup

**Determine the decision topic:**
- If `$ARGUMENTS` contains a topic (e.g., `/adr "what database to use"`), use it.
- If `$ARGUMENTS` is empty, use `AskUserQuestion` to ask: "What architectural decision do you need to make?"

**Find the ADR directory.** Run these checks:
1. Glob for existing ADR directories: `**/adr/` and `**/ADR/` within 3 levels of the project root
2. If found, use the existing directory
3. If not found, create `docs/adr/` at the project root

Use `git rev-parse --show-toplevel` to find the project root. If not a git repo, use the current working directory.

**Determine the next ADR number.** List `*.md` files in the ADR directory, extract the highest `NNNN` prefix, and increment by 1. If the directory is empty or only has an index, start at `0001`.

### Step 2: Research Project Context

Understand the project before proposing options. Read these in parallel where possible:
- `CLAUDE.md` (if it exists) for project conventions and architecture notes
- `README.md` for project overview and tech stack
- Package manifests: `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `composer.json`, `Gemfile`, `pom.xml` (whichever exists)
- Top-level directory structure (`ls` the project root)
- Existing ADRs in the ADR directory (read at least the titles and decisions of recent ones to understand prior choices and constraints)

If the decision topic relates to existing code (e.g., "what testing framework" in a project that already has tests), examine the relevant source files.

Keep this research targeted. Read manifests and READMEs first, only dig deeper if the topic warrants it. The goal is to propose alternatives that fit the project's actual tech stack and constraints, not generic options.

### Step 3: Present Options

Based on your research, propose 3-5 alternatives. For each option provide:
- **Name** (concise label)
- **Description** (1-2 sentences on what it entails)
- **Pros** (bulleted list)
- **Cons** (bulleted list)
- **Fit with existing architecture** (how well it integrates with what the project already uses)

Use `AskUserQuestion` to present all options. Frame it as: "Here are the alternatives I've identified for [topic]. Which would you like to explore further, or should we discuss specific trade-offs?"

Present the options using the `preview` field on `AskUserQuestion` options so the user can see structured pros/cons for each alternative side by side. Each preview should show the pros, cons, and architecture fit in a readable format.

### Step 4: Interactive Discussion

Based on the user's response, iterate:
- If they want to explore a specific option deeper, provide more technical detail, implementation complexity, and real-world considerations
- If they want to compare two options head-to-head, do a focused comparison
- If they raise concerns or new constraints, revise the analysis
- If they suggest an option you didn't list, add it to the analysis with pros/cons

Continue using `AskUserQuestion` until the user indicates they've made a choice or are leaning toward one option. Watch for signals like "I think X", "let's go with X", or "X makes the most sense".

### Step 5: Confirm Decision

Use `AskUserQuestion` to confirm the final decision:
- State the chosen option clearly
- Summarize the key reasons (2-3 bullet points)
- Ask if there's any additional context or rationale to record

### Step 6: Create the ADR File

1. Read `references/adr-template.md` from this skill's directory
2. Fill in all sections based on the discussion:
   - **Title**: `ADR-NNNN: [Decision Topic]` using the next available number
   - **Status**: `Proposed`
   - **Date**: today's date (YYYY-MM-DD)
   - **Author**: from `git config user.name` (or leave blank if not in a git repo)
   - **Context**: synthesize from the research and discussion — background, current state, requirements, constraints
   - **Options Considered**: ALL options that were discussed (not just the chosen one), each with description, pros, and cons as discussed
   - **Decision**: the chosen option with rationale referencing the specific trade-offs that drove the choice
   - **Consequences**: positive outcomes, negative trade-offs, and risks with mitigations
   - **Implementation Plan**: if discussed, include phased checklist; otherwise provide a reasonable starting plan
   - **Related ADRs**: link any existing ADRs that relate to this decision
   - **References**: include any relevant documentation links discussed
3. Generate the filename slug from the title: lowercase, hyphens for spaces, strip special characters (e.g., `0003-database-selection.md`)
4. Write the file to the ADR directory

### Step 7: Update the Index

Check if an index file exists (`0000-adr-index.md` or `0000-index.md`).

**If an index exists:** Read it, append a new row to the ADR Index table:
```
| [NNNN](./NNNN-slug.md) | Title | Proposed | YYYY-MM-DD |
```

**If no index exists:** Create `0000-adr-index.md` with this structure:

```markdown
# Architecture Decision Records

## Status Definitions

| Status | Description |
|--------|-------------|
| Proposed | Under discussion, not yet accepted |
| Accepted | Decision accepted, not yet implemented |
| Implemented | Decision accepted and fully implemented |
| Deprecated | No longer relevant |
| Superseded | Replaced by a newer ADR |

## ADR Index

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [NNNN](./NNNN-slug.md) | Title | Proposed | YYYY-MM-DD |
```

### Step 8: Offer Implementation

Tell the user the ADR has been created and its file path. Then use `AskUserQuestion` to ask: "Would you like to begin implementing this decision now?"

- If yes, use the ADR's Implementation Plan section as your guide and proceed with implementation
- If no, confirm the ADR is saved and ready for future reference

## Rules

- Always use `0NNN-slug-title.md` naming (4-digit zero-padded number, hyphenated slug)
- Default ADR directory: `docs/adr/`
- Date format: YYYY-MM-DD
- Valid statuses: Proposed, Accepted, Implemented, Deprecated, Superseded
- Always present at least 3 options before asking for a decision
- Record ALL discussed options in the ADR, including ones not chosen — document "Why Not" for rejected options in the Consequences or Options sections
- Tailor options to the project's actual tech stack — never propose options that are incompatible with the existing architecture without calling it out
- Keep the ADR focused and scannable. Use bullet points over long paragraphs. Code examples only when they clarify the decision.
- If the project has existing ADRs, maintain consistency with their level of detail and style
- The ADR captures the decision, not the full implementation spec. Save detailed implementation for the code itself.
