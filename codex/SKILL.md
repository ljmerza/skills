---
name: codex
description: Use when the user asks to run Codex CLI commands (exec, review, resume, apply) or references OpenAI Codex for code analysis, code review, refactoring, or automated editing. Also use when the user says "codex", "run codex", "ask codex", or wants a second opinion from another AI coding agent.
---

# Codex CLI Skill Guide

Codex CLI (`codex-cli`) is OpenAI's non-interactive coding agent. Since Claude Code cannot interact with a TUI, always use the `codex exec` family of subcommands (non-interactive mode). Top-level commands like `codex resume` and `codex fork` launch the interactive TUI and cannot be used here.

## Running a Task (`codex exec`)

1. Ask the user which model to use and which reasoning effort they want (`xhigh`, `high`, `medium`, or `low`) in a single `AskUserQuestion` prompt. The user's default model and reasoning effort are configured in `~/.codex/config.toml` — if they're happy with their defaults, no `-m` or `--config` override is needed. Do not hardcode model names; accept whatever the user provides.
2. Select the sandbox mode for the task. Default to `--sandbox read-only` unless the task requires writing files or network access.
3. Assemble the command:
   ```
   codex exec --skip-git-repo-check [OPTIONS] "PROMPT"
   ```
   Common options:
   - `-m, --model <MODEL>` — override the default model
   - `--config model_reasoning_effort="<level>"` — override reasoning effort
   - `-s, --sandbox <read-only|workspace-write|danger-full-access>` — sandbox policy
   - `--full-auto` — convenience alias for `-a on-request --sandbox workspace-write` (do not combine with an explicit `--sandbox` flag since `--full-auto` already sets it to `workspace-write`)
   - `-C, --cd <DIR>` — set working directory
   - `-i, --image <FILE>` — attach image(s) to the prompt
   - `-p, --profile <PROFILE>` — use a config profile from `config.toml`
4. Always include `--skip-git-repo-check` since Claude Code may operate outside git repositories.
5. Run the command and capture both stdout and stderr. Do not suppress stderr with `2>/dev/null` — it contains error messages that are important for diagnosing failures. If the output is noisy, use `--json` for structured JSONL output that separates event types cleanly.
6. Summarize the outcome for the user. Let them know they can resume the session later with `codex exec resume --last`.

### Quick Reference

| Use case | Key flags |
| --- | --- |
| Read-only analysis | `--sandbox read-only` |
| Apply local edits | `--full-auto` (implies `-a on-request --sandbox workspace-write`) |
| Network or broad access | `--sandbox danger-full-access` |
| Work in another directory | `-C <DIR>` |
| Attach an image | `-i <FILE>` |
| Structured output | `--json` |

### Permissions

Before using these flags, confirm with the user via `AskUserQuestion` unless they've already authorized them:
- `--full-auto` — auto-approves commands in a sandboxed environment
- `--sandbox danger-full-access` — removes sandbox restrictions
- `--dangerously-bypass-approvals-and-sandbox` — skips ALL prompts and removes ALL sandboxing. Extremely dangerous. Only use if the user explicitly requests it and understands the risk.

## Code Review (`codex exec review`)

Runs a non-interactive code review. Useful for getting a second opinion on changes.

```
codex exec --skip-git-repo-check review [OPTIONS] ["CUSTOM INSTRUCTIONS"]
```

Key options:
- `--uncommitted` — review staged, unstaged, and untracked changes
- `--base <BRANCH>` — review changes against a base branch
- `--commit <SHA>` — review changes introduced by a specific commit
- `--title <TITLE>` — optional commit title for the review summary
- `-m, --model <MODEL>` — override model
- `-o, --output-last-message <FILE>` — write the review to a file

**Example:** Review uncommitted changes:
```
codex exec --skip-git-repo-check review --uncommitted
```

## Resuming a Session (`codex exec resume`)

Resumes a previous non-interactive session. The session inherits the original model, reasoning effort, and sandbox mode unless overridden.

```
codex exec resume --last "FOLLOW-UP PROMPT"
```

The prompt is a positional argument — no need to pipe via stdin.

Supported flags on resume: `-c`, `-m`, `--full-auto`, `--skip-git-repo-check`, `--ephemeral`, `--json`, `-o`, `-i`, `--enable`, `--disable`.

Flags **not** available on resume: `--sandbox`, `-C/--cd`, `--add-dir`, `--output-schema`, `-p/--profile`.

To override the model or reasoning effort when resuming:
```
codex exec --skip-git-repo-check -m "model-name" resume --last "PROMPT"
```

Flags shared between `exec` and `resume` go before `resume`; flags specific to `resume` (like `--last`) go after.

## Applying Diffs (`codex apply`)

Applies the latest diff from a Codex agent session as a `git apply` to the local working tree.

```
codex apply <TASK_ID>
```

The `TASK_ID` is required — it identifies which session's diff to apply.

## Advanced Options

These flags work with `codex exec`:

| Flag | Purpose |
| --- | --- |
| `--json` | Output events as JSONL to stdout (structured, parseable) |
| `--output-schema <FILE>` | Constrain model output to a JSON schema |
| `-o, --output-last-message <FILE>` | Write the agent's final message to a file |
| `--add-dir <DIR>` | Additional writable directories beyond the workspace |
| `--ephemeral` | Run without persisting session files to disk |
| `--image <FILE>` | Attach image(s) to the prompt (repeatable) |
| `-p, --profile <PROFILE>` | Use a named config profile from `config.toml` |

## Error Handling

- If a `codex exec` command exits non-zero, stop and report the failure to the user. Include stderr output — it contains the error details. Ask for direction before retrying.
- When output includes warnings or partial results, summarize them and ask how to adjust.

## Notes

- `codex fork` and top-level `codex resume` launch the interactive TUI and cannot be used from Claude Code. Always use `codex exec resume` instead.
- `--full-auto` is a convenience alias that sets both `-a on-request` and `--sandbox workspace-write`. If you need `danger-full-access`, use `--sandbox danger-full-access` directly without `--full-auto`.
- The `--skip-git-repo-check` flag is always included because Claude Code may invoke codex in directories that are not git repositories.
