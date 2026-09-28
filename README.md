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
| `readonly: true` | `pstack:reviewer`, an agent without edit or Agent tools |
| `environment: "cloud"` | `isolation: "worktree"` background agents, or `isolation: "remote"` when wanted |
| `~/.cursor/rules/pstack-models.mdc` (always-applied rule) | `~/.claude/pstack-models.md`, read at spawn time |
| Model slugs (`grok-4.7-xhigh-fast`, `gpt-5.6-sol-max`, `claude-opus-5-5-max`) | `opus`, `sonnet`, `haiku`, `fable`, `<model>:<effort>`, `inherit`, `agent:<subagent_type>`, `codex:<model>[:<effort>]`, `openrouter:<model-id>`, or `nvidia:<model-id>` |
| Reasoning-effort budgets | `max` / `balanced` / `lean` model-tier budgets, plus a fixed effort per role with `<model>:<effort>` |
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
| `codex:<model>[:<effort>]` | `pstack:codex-bridge` runs `bin/codex-seat` | [Codex CLI](https://github.com/openai/codex) logged in (`codex login`), so it uses your ChatGPT plan | Review and write seats. Codex reads files and runs commands itself. |
| `openrouter:<model-id>` | `pstack:chat-bridge` runs `bin/chat-ask` | `OPENROUTER_API_KEY` in the environment | Review, judge, and design seats. Text in, text out. |
| `nvidia:<model-id>` | `pstack:chat-bridge` runs `bin/chat-ask` | `NVIDIA_API_KEY` (an `nvapi-` key from [build.nvidia.com](https://build.nvidia.com/models)) in the environment | Same as `openrouter:`. `chat-ask --list nvidia` prints the model ids. |

Aliases keep a model in one place, and `/pstack:setup-pstack` asks whether each external model may take judgment seats or only bulk work:

```
@luna: codex:gpt-6-luna:max
swarm workers: @luna
mechanical edits: @luna
```

Here `@luna` only gets bulk work: parallel `swarm` slices and `mechanical edits` (bulk renames, boilerplate rewrites). Put an alias in a panel line (`interrogate reviewers: opus, opus, @luna`) to use it for judgment instead. Leave an alias empty (`@luna:`) to drop its seats. A failed external seat falls back to the skill's default model, and the reply says so.

### How seats run

- **Review and write seats.** Each skill marks a spawn as one or the other. Claude review seats run as `pstack:reviewer`, which keeps Bash and MCP tools but has no edit or Agent tools. Bridges get `Mode: review` or `Mode: write`.
- **The parent writes the prompt once.** It goes into a private file, and the bridge gets a five-line brief (model, effort, mode, repository, prompt file). The bridge returns the answer file's path, so the small relay model never retypes the task or the answer.
- **Write seats stay in their worktree.** `codex-seat` gives Codex a writable sandbox only inside a linked git worktree, the kind the Agent tool creates with `isolation: "worktree"`. Anywhere else it downgrades the seat to review. Changes stay uncommitted for the parent to review and apply.
- **Codex runs without your Codex extras.** `--ignore-user-config` keeps your Codex MCP servers, hooks, and notify program out of the run, because they execute outside Codex's sandbox. On Windows that also drops `[windows] sandbox`, and the default Windows sandbox cannot start a shell, so `codex-seat` passes `windows.sandbox="elevated"` (override with `PSTACK_CODEX_WINDOWS_SANDBOX`).
- **The chat bridge never attaches secrets**: `.env` files, keys, credential files, and anything git ignores.

### Claude effort per role

A plain `opus` seat runs at your session's effort. Write `<model>:<effort>` for a fixed level, for example `hardest tasks: opus:max`. The Agent tool has no effort parameter, so `/pstack:setup-pstack` writes an agent pair to `~/.claude/agents/` for each such value in your config: `pstack-<model>-<effort>-review` for review seats and `pstack-<model>-<effort>` for write seats. Files it generated carry a marker, and a re-run rewrites or deletes only those.

Fixed high effort costs most on frequent roles. `architect runners` and `how explorer` fire on most feature work, while `hardest tasks` and `arena runners` run rarely.

### Settings worth adding

Bridges run under the Bash timeout, 10 minutes by default. Write seats start from your default branch. To change both, add to `~/.claude/settings.json`:

```json
{
  "env": { "BASH_MAX_TIMEOUT_MS": "1800000" },
  "worktree": { "baseRef": "head" }
}
```

External seats send code and diffs to that provider, and `reflect` sends the session transcript. Keep panels Claude-only for code you cannot share.

## Not ported

- `make-bot-ui`. It targets Cursor Automations webhooks.
- The `benny` automation pack and the `docs/guide`. Both are Cursor-specific.
- The `scripts/` tooling (`watch-pr`, `orch`) is included unchanged. It needs [Bun](https://bun.sh).

## License

MIT. The original work is Copyright (c) 2026 Lauren Tan, see [LICENSE](LICENSE). This port keeps that notice and is distributed under the same license. It is not affiliated with or endorsed by Cursor or Anysphere.
