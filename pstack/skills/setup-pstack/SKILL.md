---
name: setup-pstack
description: Configure which model or subagent pstack uses per role, including Codex CLI and OpenRouter models. Writes ~/.claude/pstack-models.md, which every pstack skill reads before spawning subagents. Use for /setup-pstack, "configure pstack models", "pstack budget", or changing pstack's model choices.
---

# Setup pstack

Write `~/.claude/pstack-models.md`, the per-role model config that pstack skills read before they spawn subagents.

## Value grammar

Every role value is one of these. A panel role takes a comma-separated list, and one subagent runs per entry.

| Value | Agent tool call |
|---|---|
| `opus`, `sonnet`, `haiku`, `fable` | `subagent_type` as the skill prescribes, `model` set to the value. |
| `inherit` | `model` omitted. The role runs on the parent session's model. |
| `agent:<name>` | `subagent_type: "<name>"`, `model` omitted. For custom agents you defined in `~/.claude/agents/` or another plugin. |
| `codex:<model>` | `subagent_type: "pstack:codex-bridge"`. The brief starts with `Codex model: <model>`, `Mode: review` or `Mode: write`, and `Working directory: <path>`. Runs on the ChatGPT subscription through the Codex CLI. Codex can read files and run commands, so it fits every seat, including arena runners in `write` mode inside their own worktree. |
| `openrouter:<model-id>` | `subagent_type: "pstack:openrouter-bridge"`. The brief starts with `OpenRouter model: <model-id>`. Text-in, text-out, so it fits review, judge, and design-sketch seats. For a code-writing seat, the parent applies the returned patch. |
| `@<alias>` | Look up the `@<alias>:` line in the same file and use its value. Aliases let one line change a model everywhere, for example a free OpenRouter model that rotates often. An alias with an empty value (`@free:`) is disabled: panel seats that use it are skipped, and a single-value role that uses it falls back to the skill default. |

**Spawning rules for every skill.** Read the file once per task. Use the role's line, or the skill's default when the file or the line is missing. Expand aliases first. When a spawn fails, or a bridge replies `FAILED`, rerun that seat on the skill's default and say so in the reply. A bridge returns another model's words. Judge them like any reviewer's, and never cite them as your own verification. A `codex:` seat that edits files is spawned with `isolation: "worktree"` and gets `Mode: write` with that worktree as its `Working directory`. Never point it at the user's checkout. Review its diff before taking any of it.

## Steps

### 1. Detect what is available

Build the detected set:

- The `model` values the Agent tool accepts in this session.
- The custom agent types listed for the Agent tool (user, project, and plugin agents).
- Codex: `codex --version` succeeds and `codex login status` reports a login. Then `codex:<any model>` is valid. Codex rejects unknown models at run time, so name the model the user asked for.
- OpenRouter: `OPENROUTER_API_KEY` is set (check with `[ -n "$OPENROUTER_API_KEY" ]`, never print it). Then `openrouter:<id>` is valid for any id in `openrouter-ask --list-free` or the full list at `https://openrouter.ai/api/v1/models`.

`inherit` is always valid.

### 2. Load current state

The default mapping is the file shape in step 5. If `~/.claude/pstack-models.md` exists, read it and treat its `# budget` line, alias lines, and role values as the current choices. Otherwise start from the defaults. A line whose role is not in step 5 is from a retired role. Drop it.

### 3. Budget, map, and confirm

**(a) Ask for a budget** with AskUserQuestion. Offer these labels and name the current budget when the file records one.

- `max`: every `sonnet` role becomes `opus`.
- `balanced`: the step 5 defaults.
- `lean`: every `opus` role becomes `sonnet`, and swarm workers become `haiku`.

**(b) Apply it.** Build the working table from the defaults with the budget applied. On a re-run, keep any role the user set to a value the budget does not touch (`inherit`, `fable`, `agent:`, `codex:`, `openrouter:`, an alias, or a customized list).

**(c) External models.** When Codex or OpenRouter is detected, ask which external models to use and define each as an alias (for example `@luna: codex:gpt-6-luna`, `@free: openrouter:<id>`). For OpenRouter free models, show the current `openrouter-ask --list-free` output as the options. For each external model, ask which kind of work it may take. **Judgment seats** are the panel roles (`arena runners`, `arena cross-judge pool`, `architect runners`, `interrogate reviewers`). One external seat per panel restores the multi-vendor diversity those skills were designed around, and `arena runners` also writes code. **Bulk work** is `swarm workers` and `mechanical edits`: many simple, tightly scoped tasks. A model the user does not trust with judgment goes only in bulk work. Tell the user that external seats send code and diffs to that provider, and that free and stealth models may log prompts.

**(d) Show the roles and confirm.** Show every alias and role with its value, and list each line step 2 dropped. Ask with AskUserQuestion whether to accept as-is or change specific roles.

### 4. Validate

Every value must parse per the grammar, every alias must be defined (an empty value counts as defined and disabled), and every value must be in the detected set. If one is not, stop and ask again.

### 5. Write the file

Overwrite `~/.claude/pstack-models.md` whole, so re-runs stay idempotent. Alias lines go first. Shape:

```
# pstack model configuration. One line per role. Delete a line to fall back to the skill default.
# Values: opus | sonnet | haiku | fable | inherit | agent:<subagent_type> | codex:<model> | openrouter:<model-id> | @<alias>
# Panel roles take a comma-separated list; one subagent per entry.
# budget: balanced
# Aliases. Change a model everywhere by editing one line here.
# @luna: codex:gpt-6-luna
# @free: openrouter:openrouter/free
feature, refactoring: sonnet
bug-fix: sonnet
perf-issue: sonnet
hillclimb: sonnet
judgment and prose: opus
hardest tasks: opus
how explorer: sonnet
how explainer: opus
why investigators: sonnet
why synthesizer: opus
reflect tooling: sonnet
reflect judgment, divergent, synthesizer: opus
arena runners: opus, opus, sonnet
arena cross-judge pool: opus, sonnet
swarm workers: sonnet
mechanical edits: sonnet
architect runners: opus, opus, sonnet
interrogate reviewers: opus, opus, sonnet
```

Write active aliases without the leading `# `.

### 6. Confirm

Tell the user the file was written. Skills read it at spawn time, so it applies immediately. To swap a free model later, edit its alias line or re-run this skill. To drop it while no good free model exists, leave the value empty (`@free:`).

### 7. Offer a verification skill (optional)

Check whether the project has a way to drive the real app for proof (a `verify-*` skill, or an existing harness). If not, offer once: "want a project-local verification skill, so agents can drive the app the way a user does and prove changes work? I can generate one with /create-verification-skill." On yes, run the **create-verification-skill** skill. On no, move on without pushing.

## pstack on Claude Code

`<pstack>` is `${CLAUDE_PLUGIN_ROOT}`. Other pstack skills live at `<pstack>/skills/<name>/SKILL.md`. Read them with the Read tool.
