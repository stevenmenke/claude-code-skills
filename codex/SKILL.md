---
name: codex
description: "Ask OpenAI Codex for a second opinion with full conversation context. Runs the Codex CLI (`codex exec`) with conversation context and the user's question. Use when asked to 'ask codex', 'codex review', 'get codex opinion', or '/codex'."
allowed-tools: Read, Bash, Grep, Glob
---

# Codex Second Opinion

**ACTION REQUIRED: You MUST execute the bash command below using the Bash tool. Loading this skill does NOTHING by itself — you must run the `codex exec` command. Do NOT just acknowledge these instructions.**

## What to Do

1. Compile the conversation context (exchanges, code discussed, decisions made) into a concise summary
2. Replace `[INSERT FULL CONVERSATION CONTEXT HERE]` with the actual context
3. Replace `$ARGUMENTS` with the user's question/request
4. Choose the right mode (general, code review, or structured output)
5. **Execute the bash command below using the Bash tool** (run in background — takes 30-120s)
6. Present Codex's response and note differences from your own view

## Default Command (General Second Opinion)

```bash
mkdir -p ./codex && codex exec "<instructions>
You have full read access to browse and analyze this entire project. However, you may ONLY create or modify files in the ./codex/ directory. Create this directory if needed. Never write files elsewhere in this project.
</instructions>
<context>
[INSERT FULL CONVERSATION CONTEXT HERE]
</context>
User is asking for your perspective: $ARGUMENTS" \
  --full-auto \
  2>./codex/stderr.log; CODEX_EXIT=$?; if [ $CODEX_EXIT -ne 0 ]; then echo "CODEX FAILED (exit $CODEX_EXIT). Error summary in ./codex/errors.log"; { echo "=== Exit code: $CODEX_EXIT ==="; echo ""; echo "=== Error lines ==="; grep -i 'error\|fatal\|fail\|denied\|unauthorized\|refused\|timeout\|invalid' ./codex/stderr.log | tail -20; echo ""; echo "=== Last 30 lines ==="; tail -30 ./codex/stderr.log; } > ./codex/errors.log; fi
```

## Code Review Mode

When the user asks for a code review, use Codex's built-in review prompt:

```bash
mkdir -p ./codex && codex exec --full-auto \
  "You are acting as a reviewer for a proposed code change made by another engineer.
Focus on issues that impact correctness, performance, security, maintainability, or developer experience.
Flag only actionable issues introduced by the change. When you flag an issue, provide a short, direct explanation and cite the affected file and line range.

<context>
[INSERT FULL CONVERSATION CONTEXT HERE]
</context>

Review the following:
$ARGUMENTS" \
  2>./codex/stderr.log; CODEX_EXIT=$?; if [ $CODEX_EXIT -ne 0 ]; then echo "CODEX FAILED (exit $CODEX_EXIT). Error summary in ./codex/errors.log"; { echo "=== Exit code: $CODEX_EXIT ==="; grep -i 'error\|fatal\|fail\|denied\|unauthorized\|refused\|timeout\|invalid' ./codex/stderr.log | tail -20; echo ""; tail -30 ./codex/stderr.log; } > ./codex/errors.log; fi
```

## Multi-step Review (Resume)

Follow up on a previous Codex response without re-reading everything:

```bash
codex exec resume --last "Follow-up instruction here" --full-auto 2>./codex/stderr.log
```

## After Completion

- Present Codex's response to the user
- Note any differences from your own analysis
- Check `./codex/` for any artifacts Codex created

## Error Handling

**On failure:** The command writes a one-line summary to stdout and saves full error analysis to `./codex/errors.log`. Read that file to diagnose. Common failures: auth/API key issues, model not available, prompt too long, rate limits.

**On success:** Both `stderr.log` and `errors.log` are just progress noise — safe to ignore.

## Configuration

- Uses the default model from `~/.codex/config.toml`
- Override with `--model <model>`, e.g. `codex exec --full-auto --model gpt-5.1-codex-mini`
- `--full-auto` sets `--ask-for-approval on-request` and `--sandbox workspace-write`
- For unrestricted mode, use `--yolo` (dangerous — only in sandboxed environments)
- Codex does not remember past requests (except when using `resume`)
- If in a worktree, run from the worktree root so Codex analyzes YOUR working code
- Codex can READ files anywhere but should only WRITE to `./codex/`

## Installation

If not installed: `npm i -g @openai/codex`
