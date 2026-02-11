---
name: gemini
description: "Ask Gemini for a second opinion with full conversation context. Runs the Gemini CLI (`gemini -p`) with conversation context and the user's question. Use when asked to 'ask gemini', 'gemini review', 'get gemini opinion', or '/gemini'."
allowed-tools: Read, Bash, Grep, Glob
---

# Gemini Second Opinion

**ACTION REQUIRED: You MUST execute the bash command below using the Bash tool. Loading this skill does NOTHING by itself — you must run the `gemini -p` command. Do NOT just acknowledge these instructions.**

## What to Do

1. Compile the conversation context (exchanges, code discussed, decisions made) into a concise summary
2. Replace `[INSERT FULL CONVERSATION CONTEXT HERE]` with the actual context
3. Replace `$ARGUMENTS` with the user's question/request
4. **Execute the bash command below using the Bash tool** (run in background — takes 30-120s)
5. Present Gemini's response and note differences from your own view

## Command Template

```bash
mkdir -p ./gemini && gemini -p "<instructions>
You have full read access to browse and analyze this entire project. However, you may ONLY create or modify files in the ./gemini/ directory. Create this directory if needed. Never write files elsewhere in this project.
</instructions>

<context>
[INSERT FULL CONVERSATION CONTEXT HERE]
</context>

User is asking for your perspective: $ARGUMENTS" \
  --approval-mode auto_edit \
  2>./gemini/stderr.log; GEMINI_EXIT=$?; if [ $GEMINI_EXIT -ne 0 ]; then echo "GEMINI FAILED (exit $GEMINI_EXIT). Error summary in ./gemini/errors.log"; { echo "=== Exit code: $GEMINI_EXIT ==="; echo ""; echo "=== Error lines ==="; grep -i 'error\|fatal\|fail\|denied\|unauthorized\|refused\|timeout\|invalid' ./gemini/stderr.log | tail -20; echo ""; echo "=== Last 30 lines ==="; tail -30 ./gemini/stderr.log; } > ./gemini/errors.log; fi
```

## After Completion

- Present Gemini's response to the user
- Note any differences from your own analysis
- Check `./gemini/` for any artifacts Gemini created

## Error Handling

**On failure:** The command writes a one-line summary to stdout and saves full error analysis to `./gemini/errors.log`. Read that file to diagnose. Common failures: auth/API key issues, model not available, prompt too long, rate limits.

**On success:** Both `stderr.log` and `errors.log` are just progress noise — safe to ignore.

## Configuration

- Uses the default model from `~/.gemini/settings.json`
- **For web search research**: always use `-m gemini-3-flash-preview` (faster, grounded search built-in)
- **For code reviews**: use default model (higher reasoning capability)
- Override with `-m <model>`, e.g. `gemini -m gemini-3-flash-preview`
- `--approval-mode auto_edit` allows Gemini to write files without interactive approval
- Gemini does not remember past requests — include all relevant context every time
- If in a worktree, run from the worktree root so Gemini analyzes YOUR working code
- Gemini can READ files anywhere but should only WRITE to `./gemini/`
