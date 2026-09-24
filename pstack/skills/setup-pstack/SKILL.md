---
name: setup-pstack
description: Configure which model or subagent pstack uses per role. Writes ~/.claude/pstack-models.md, which every pstack skill reads before spawning subagents. Use for /setup-pstack, "configure pstack models", "pstack budget", or changing pstack's model choices.
---

# Setup pstack

Write `~/.claude/pstack-models.md`, the per-role model config that pstack skills read before they spawn subagents.

## Value grammar

Every role value is one of these. A panel role takes a comma-separated list, and one subagent runs per entry.

| Value | Agent tool call |
|---|---|
| `opus`, `sonnet`, `haiku`, `fable` | `subagent_type` as the skill prescribes, `model` set to the value. |
| `inherit` | `model` omitted. The role runs on the parent session's model. |
| `agent:<name>` | `subagent_type: "<name>"`, `model` omitted. Use this for custom or proxy-routed agents (for example `agent:ocx-gpt-5-6-luna`), which pin their own model. |

## Steps

### 1. Detect what is available

List the `model` values the Agent tool accepts in this session, and the custom agent types listed for the Agent tool (user, project, and plugin agents). Those two lists are the detected set. `inherit` is always valid. Never write an `agent:<name>` whose agent is not in the detected set.

### 2. Load current state

The default mapping is the file shape in step 5. If `~/.claude/pstack-models.md` exists, read it and treat its `# budget` line and role values as the current choices. Otherwise start from the defaults. A line whose role is not in step 5 is from a retired role. Drop it.

### 3. Budget, map, and confirm

**(a) Ask for a budget** with AskUserQuestion. Offer these labels and name the current budget when the file records one.

- `max`: every `sonnet` role becomes `opus`.
- `balanced`: the step 5 defaults.
- `lean`: every `opus` role becomes `sonnet`, and swarm workers become `haiku`.

**(b) Apply it.** Build the working table from the defaults with the budget applied. On a re-run, keep any role the user set to a value the budget does not touch (`inherit`, `fable`, `agent:<name>`, or a customized list).

**(c) Show the roles and confirm.** Show every role with its value, and list each line step 2 dropped. Ask with AskUserQuestion whether to accept as-is or change specific roles. When the detected set includes custom agents backed by non-Claude models, point out that adding them to the panel roles (`arena runners`, `arena cross-judge pool`, `architect runners`, `interrogate reviewers`) restores the multi-model diversity those skills were designed around.

### 4. Validate

Every value must parse per the grammar and be in the detected set. If one is not, stop and ask again.

### 5. Write the file

Overwrite `~/.claude/pstack-models.md` whole, so re-runs stay idempotent. Shape:

```
# pstack model configuration. One line per role. Delete a line to fall back to the skill default.
# Values: opus | sonnet | haiku | fable | inherit | agent:<subagent_type>. Panel roles take a comma-separated list; one subagent per entry.
# budget: balanced
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
architect runners: opus, opus, sonnet
interrogate reviewers: opus, opus, sonnet
```

### 6. Confirm

Tell the user the file was written. Skills read it at spawn time, so it applies immediately. Re-running this skill updates it.

### 7. Offer a verification skill (optional)

Check whether the project has a way to drive the real app for proof (a `verify-*` skill, or an existing harness). If not, offer once: "want a project-local verification skill, so agents can drive the app the way a user does and prove changes work? I can generate one with /create-verification-skill." On yes, run the **create-verification-skill** skill. On no, move on without pushing.

## pstack on Claude Code

`<pstack>` is `${CLAUDE_PLUGIN_ROOT}`. Other pstack skills live at `<pstack>/skills/<name>/SKILL.md`. Read them with the Read tool.
