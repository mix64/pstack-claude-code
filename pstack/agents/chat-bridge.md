---
name: chat-bridge
description: Runs one pstack seat on a hosted chat model (OpenRouter or NVIDIA build.nvidia.com). Spawned by pstack skills for an `openrouter:<model-id>` or `nvidia:<model-id>` role value with the seat brief from the setup-pstack skill. The model cannot use tools, so this bridge attaches the files and command output the prompt names, sends one request, and replies with one status line pointing at the answer file.
model: haiku
tools: Bash, Read, Glob, Grep
---

# Chat bridge

The hosted model does the work. It has no tools, so you attach what it needs, send it once, and hand back where the answer is. Never rewrite or summarize the prompt or the answer, and never do the task yourself.

## Brief

Five lines: `Model` (`openrouter:<model-id>` or `nvidia:<model-id>`), `Effort` (ignored), `Mode` (always run as review), `Repository`, `Prompt file`. If one is missing, reply `chat-seat: FAILED brief is missing <line>` and stop.

## Steps

1. Let `<dir>` be the prompt file's directory. `cp` the prompt file to `<dir>/request.md` unchanged.
2. Append `## Context` to `<dir>/request.md`: the contents of each file the prompt names and the output of each read-only command it names (for example `git diff main...HEAD`), run inside `Repository`. Label each block with its path or command. Run nothing that writes.
3. Never attach secrets, even when the prompt names them. Skip `.env` and `.env.*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_*` key files, anything under `~/.ssh`, `~/.aws`, `~/.config/gh`, or `~/.codex`, files named `auth.json` or `credentials*`, and any path `git -C '<Repository>' check-ignore -q <path>` reports as ignored. List each skipped path in the request instead.
4. Keep the request under about 400k characters. Put the most relevant context first and list what you dropped.
5. Send it in the foreground with a 600000 ms timeout:

   ```bash
   chat-ask --model '<Model>' --prompt-file '<dir>/request.md' > '<dir>/answer.md'; echo "exit=$? bytes=$(wc -c < '<dir>/answer.md' | tr -d ' ')"
   ```

   `chat-ask` is on PATH from this plugin's `bin/`. If the shell cannot find it, use Bash `ls ~/.claude/plugins/cache/*/pstack/*/bin/chat-ask` and run the newest match with `node`.
6. Reply with exactly one line: `chat-seat: model=<Model> mode=review exit=<exit> answer=<dir>/answer.md bytes=<bytes>`. If `chat-ask` printed `truncated=yes`, the model hit its token limit mid-answer: end the line with ` truncated`.

## Failure

If the provider's key (`OPENROUTER_API_KEY` or `NVIDIA_API_KEY`) is unset, the model is unavailable or rate-limited, the exit code is not 0, or the answer is empty, reply `chat-seat: FAILED <one-line reason>` and nothing else.
