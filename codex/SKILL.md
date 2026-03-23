---
name: codex
version: 3.0.0
description: "Invoke Codex CLI as a peer AI agent — reviews, implementation, fixes, analysis, or any task. Full project access by default; sandboxed for reviews/analysis. Use for '/codex', 'ask codex', 'codex review', 'codex implement', 'codex fix', or any second opinion."
argument-hint: "<task> | review <target> | implement <scope> | fix <issues> | resume <follow-up> | models | help"
allowed-tools: Read, Bash, Grep, Glob
context: session
---

Arguments: **"$ARGUMENTS"**

## Special Commands

If empty or "help": `bash ~/.claude/skills/codex/scripts/help.sh` → output verbatim, stop.
If "models": Read `~/.claude/skills/shared/models.yaml` (codex section) → formatted table with current config highlighted, stop.

If "resume <follow-up>":
```bash
codex exec resume --last "[FOLLOW-UP]" --skip-git-repo-check 2>./codex/stderr.log
```
Present response, stop.

## Execution

1. **Show current model:** `grep '^model' ~/.codex/config.toml`
2. **Compile conversation context** — recent exchanges, code discussed, decisions made (keep concise)
3. **Detect intent** from arguments and select settings from the table below
4. **Compose prompt** using the building blocks
5. **Assemble command** and run

### Intent Detection

| Intent | Trigger words | Permission | Reasoning |
|--------|--------------|------------|-----------|
| **Review** | "review" | Sandbox | high |
| **Implement** | "implement", "build", "write", "create", "add" | Unrestricted | high |
| **Fix** | "fix", "patch", "resolve", "address" | Unrestricted | high |
| **Analyze** | "analyze", "investigate", "audit", "check", "compare" | Sandbox | high |
| **General** | anything else | Unrestricted | high (medium for simple questions) |

**Key principle:** Default is **unrestricted**. Sandbox only when the task doesn't need source edits (reviews, analysis). The orchestrator can override — e.g., `/codex implement [scope]` always gets unrestricted access.

### Compose the Prompt

**Permission block** (pick one based on intent):
- **Unrestricted:** `You have full read and write access to this project.`
- **Sandbox:** `You have full read access to this project. Do not modify existing project source files.`

**Output block** (independent of permission — controls where NEW artifacts go):
- **You (the calling Claude Code instance) decide the output path.** Consider your context: Are you in a worktree? An orchestrator review round? A standalone invocation? Choose a path that makes sense.
- Examples: `Save your review to ./codex/auth-review.md`, `Write findings to ./docs/reviews/api-audit.md`
- **Be explicit when it matters** — for reviews, orchestrated builds, or worktree sessions, always specify the path. For casual questions or general tasks, omit (Codex writes wherever it sees fit).
- This only controls where *new files* are created, not which *existing files* can be edited (that's the permission block's job).

**Role block** (optional — add only when intent benefits from framing):
- **Review:** `You are reviewing code changes for correctness, performance, security, and maintainability. Flag actionable issues with file:line citations. Categorize findings as P0 (critical), P1 (major), or P2 (minor).`
- **Implement/Fix:** Include specific scope, files, and what "done" looks like. Add `Run the project's test suite after implementation and report pass/fail.` when appropriate.
- **General/Analyze:** No special role needed — the task description is sufficient.

**Assemble:**
```
<instructions>
[PERMISSION BLOCK]
[OUTPUT BLOCK — if caller specified a path]
[ROLE BLOCK — if applicable]
</instructions>

<context>
[CONVERSATION CONTEXT]
</context>

[TASK FROM ARGUMENTS]
```

### Command Template

```bash
mkdir -p ./codex && codex exec --skip-git-repo-check \
  "[ASSEMBLED PROMPT]" \
  2>./codex/stderr.log; CODEX_EXIT=$?; if [ $CODEX_EXIT -ne 0 ]; then echo "CODEX FAILED (exit $CODEX_EXIT)"; { echo "=== Exit $CODEX_EXIT ==="; grep -i 'error\|fatal\|fail\|denied\|unauthorized\|refused\|timeout\|invalid\|limit\|quota' ./codex/stderr.log | tail -20; echo ""; tail -30 ./codex/stderr.log; } > ./codex/errors.log; fi
```

**Why not `--full-auto`?** It overrides global config's `sandbox_mode = "danger-full-access"` to `workspace-write`, blocking `~/.wrangler` writes and network — breaks `vitest-pool-workers` (workerd) and `npm install`. Without it, global config applies: `approval_policy = "never"` + `sandbox_mode = "danger-full-access"`. Matches Gemini's `yolo` mode.

For lower reasoning effort, add `-c model_reasoning_effort="medium"` after `--skip-git-repo-check`.

Run in **background** — use `run_in_background: true` on the Bash tool.

### Concrete Examples

**Review (sandbox):**
```bash
mkdir -p ./codex && codex exec --skip-git-repo-check \
  "<instructions>
You have full read access to this project. Do not modify existing project source files.
Save your review report as ./docs/reviews/batch-delete-review.md
You are reviewing code changes for correctness, performance, security, and maintainability. Flag actionable issues with file:line citations. Categorize findings as P0 (critical), P1 (major), or P2 (minor).
</instructions>

<context>
We just implemented a batch delete feature. Changes span src/services/batch/ and tests/batch.test.ts.
</context>

Review the batch delete implementation for correctness and security." \
  2>./codex/stderr.log; CODEX_EXIT=$?; if [ $CODEX_EXIT -ne 0 ]; then echo "CODEX FAILED (exit $CODEX_EXIT)"; { echo "=== Exit $CODEX_EXIT ==="; grep -i 'error\|fatal\|fail\|denied\|unauthorized\|refused\|timeout\|invalid\|limit\|quota' ./codex/stderr.log | tail -20; echo ""; tail -30 ./codex/stderr.log; } > ./codex/errors.log; fi
```

**Implement (unrestricted):**
```bash
mkdir -p ./codex && codex exec --skip-git-repo-check \
  "<instructions>
You have full read and write access to this project.
Implement the changes described below. Run npm test after implementation and report pass/fail.
</instructions>

<context>
We need to add a tag merging feature. Design: docs/design/tag-merge.md. Existing tag code: src/models/tag.ts
</context>

Implement the merge_tags function per the design doc. Add it to the exports in index.ts." \
  2>./codex/stderr.log; CODEX_EXIT=$?; if [ $CODEX_EXIT -ne 0 ]; then echo "CODEX FAILED (exit $CODEX_EXIT)"; { echo "=== Exit $CODEX_EXIT ==="; grep -i 'error\|fatal\|fail\|denied\|unauthorized\|refused\|timeout\|invalid\|limit\|quota' ./codex/stderr.log | tail -20; echo ""; tail -30 ./codex/stderr.log; } > ./codex/errors.log; fi
```

**General question (unrestricted, medium reasoning):**
```bash
mkdir -p ./codex && codex exec --skip-git-repo-check \
  -c model_reasoning_effort="medium" \
  "<instructions>
You have full read and write access to this project.
</instructions>

<context>
Working on a Cloudflare Workers app with Durable Objects for per-user storage.
</context>

What are the tradeoffs between using DO SQLite vs D1 for user settings?" \
  2>./codex/stderr.log; CODEX_EXIT=$?; if [ $CODEX_EXIT -ne 0 ]; then echo "CODEX FAILED (exit $CODEX_EXIT)"; { echo "=== Exit $CODEX_EXIT ==="; grep -i 'error\|fatal\|fail\|denied\|unauthorized\|refused\|timeout\|invalid\|limit\|quota' ./codex/stderr.log | tail -20; echo ""; tail -30 ./codex/stderr.log; } > ./codex/errors.log; fi
```

### After Completion

- Present Codex's response to the user
- Note any differences from your own analysis
- Check for artifacts at the output path you specified (if any)
- If failed, read `./codex/errors.log` for diagnostics

## Error Handling

- **Stale logs:** `errors.log`/`stderr.log` persist across runs. Check `ps aux | grep "codex exec"` before assuming failure — a running process means logs are stale.
- **Common failures:** Rate limit / quota exhaustion (most common), auth/API key issues, model unavailable, prompt too long.

## Session Resume & Recovery

### Finding Session IDs

Codex stores sessions in SQLite at `~/.codex/state_5.sqlite`. The `threads` table has all session data.

**Find recent sessions:**
```bash
sqlite3 ~/.codex/state_5.sqlite "SELECT id, title, source, cwd FROM threads ORDER BY created_at DESC LIMIT 10;"
```

**Find sessions for a specific worktree/directory:**
```bash
sqlite3 ~/.codex/state_5.sqlite "SELECT id, title FROM threads WHERE cwd LIKE '%my-project%' ORDER BY created_at DESC LIMIT 5;"
```

**Get session details (first message, model, tokens):**
```bash
sqlite3 ~/.codex/state_5.sqlite "SELECT id, substr(first_user_message, 1, 200), model_provider, tokens_used FROM threads WHERE id = 'SESSION_ID';"
```

### Resuming a Session

```bash
codex --resume SESSION_ID
```

This continues from the session's last checkpoint with full context preserved. Useful when:
- Quota runs out mid-implementation (switch accounts, resume)
- Session needs follow-up work
- Want to inspect what Codex did and ask it to continue

### Thread Table Schema (key columns)

- `id` — UUID thread identifier
- `cwd` — working directory (useful for filtering by worktree)
- `title` — first line of the prompt (truncated)
- `first_user_message` — full initial prompt
- `source` — "exec" for `codex exec` invocations
- `created_at` / `updated_at` — Unix timestamps
- `tokens_used` — total token consumption
- `model_provider` — which model was used

### Implementation Session Pattern

When running multi-session builds (like `/orchestrate auto-build`), record the Codex session ID after launching each implementation session. Format:

```
Session 1: 019ccf22-907c-7843-8ce6-eb27f6da350c (Core write architecture)
Session 2: [pending]
```

This enables:
1. Resuming if quota exhausts mid-session
2. Reviewing what Codex read/wrote after completion
3. Following up on caveats (e.g., "fix the z.any() workaround")

### `--resume` vs `/codex resume`

The `/codex` skill's `resume` command (`/codex resume "follow-up"`) uses a **different mechanism** — it resumes the LAST codex session via `codex exec resume`. The SQLite `--resume SESSION_ID` flag is for resuming **any** session by ID, not just the most recent one.
