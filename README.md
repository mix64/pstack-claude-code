# pstack for Claude Code

A Claude Code port of [pstack](https://github.com/cursor/plugins/tree/main/pstack) v0.15.5 by [poteto](https://x.com/poteto) (Lauren Tan), MIT licensed. The upstream README explains the philosophy. This file covers what differs on Claude Code.

## Install

From GitHub:

```
/plugin marketplace add mix64/pstack-claude-code
/plugin install pstack@pstack-claude-code
```

From a local clone:

```
/plugin marketplace add /path/to/pstack-claude-code
/plugin install pstack@pstack-claude-code
```

Then run `/pstack:setup-pstack` once to choose models per role.

## Use

```
/pstack:poteto-mode this pr has a subtle bug where the scroll drifts every 750ms even when idle. repro first, then fix and verify.
/pstack:how do we cancel runs?
/pstack:interrogate review this pr.
```

Every skill except `setup-pstack` is user-invocable only (`disable-model-invocation: true`, as upstream). That keeps 45 skill descriptions out of every session's context. `poteto-mode` reaches the others by reading `${CLAUDE_PLUGIN_ROOT}/skills/<name>/SKILL.md` directly.

## What changed from the Cursor version

| Cursor | Claude Code port |
|---|---|
| `.cursor-plugin/plugin.json` | `.claude-plugin/plugin.json` |
| `Task` tool, `generalPurpose` | Agent tool, `general-purpose` |
| `readonly: true` | Prompt-level "do not edit files" (no flag exists) |
| `environment: "cloud"` | `isolation: "worktree"` background agents, or `isolation: "remote"` when wanted |
| `~/.cursor/rules/pstack-models.mdc` (always-applied rule) | `~/.claude/pstack-models.md`, read at spawn time |
| Model slugs (`grok-4.7-xhigh-fast`, `gpt-5.6-sol-max`, `claude-opus-5-5-max`) | `opus`, `sonnet`, `haiku`, `fable`, `inherit`, or `agent:<subagent_type>` |
| Reasoning-effort budgets | `max` / `balanced` / `lean` model-tier budgets |
| `~/.cursor/projects/<slug>/agent-transcripts/` | `~/.claude/projects/<slug>/<session-id>.jsonl` |
| `subagent_type: "poteto-agent"`, `"Comment Sicko"` | `pstack:poteto-agent`, `pstack:comment-sicko` |
| `mode: true` sticky mode with `reminder` | A sticky instruction at the top of `poteto-mode` (invoked skill content stays in context) |
| `cursor-team-kit` (`deslop`, `control-ui`, `control-cli`) | built-in `simplify` skill, built-in browser tools, Bash, `run` skill |
| `create-skill` (Cursor built-in) | `skill-creator` skill |

The "Platform mapping" table in `skills/poteto-mode/SKILL.md` tells the agent how to translate any Cursor term still left in the playbooks.

## Multi-model panels

Upstream runs `arena`, `architect`, and `interrogate` across Claude, GPT, and Grok on purpose. The default here is Claude only (`opus, opus, sonnet`). Two bridge agents bring other vendors back without a proxy:

| Value | Bridge | Needs | Fits |
|---|---|---|---|
| `codex:<model>` | `pstack:codex-bridge` runs `codex exec` | [Codex CLI](https://github.com/openai/codex) logged in (`codex login`), so it uses your ChatGPT plan | Every seat. Codex reads files and runs commands itself, and edits files in `write` mode inside the seat's worktree. |
| `openrouter:<model-id>` | `pstack:openrouter-bridge` runs `bin/openrouter-ask` | `OPENROUTER_API_KEY` in the environment | Review, judge, and design seats. Text-in, text-out. |

Aliases keep a rotating model in one place. `/pstack:setup-pstack` writes them, and you can edit them by hand:

```
@luna: codex:gpt-6-luna
@free: openrouter:stealth/space-bunny-alpha
interrogate reviewers: opus, @luna, @free
arena cross-judge pool: opus, @luna, @free
```

`openrouter-ask --list-free` prints the current free OpenRouter models. A failed or rate-limited external seat falls back to the skill's default model, and the reply says so.

External seats send code and diffs to that provider. Free and stealth OpenRouter models may log prompts. Keep panels Claude-only for code you cannot share.

## Not ported

- `make-bot-ui`. It targets Cursor Automations webhooks.
- The `benny` automation pack and the `docs/guide`. Both are Cursor-specific.
- The `scripts/` tooling (`watch-pr`, `orch`) is included unchanged. It needs [Bun](https://bun.sh).

## License

MIT. The original work is Copyright (c) 2026 Lauren Tan, see [LICENSE](LICENSE). This port keeps that notice and is distributed under the same license. It is not affiliated with or endorsed by Cursor or Anysphere.
