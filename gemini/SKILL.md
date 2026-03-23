---
name: gemini
version: 3.0.0
description: "Invoke Gemini CLI as a peer AI agent — reviews, implementation, fixes, research, analysis, or any task. Full project access by default; sandboxed for reviews/analysis. Use for '/gemini', 'ask gemini', 'gemini review', 'gemini implement', 'gemini fix', 'gemini research', 'gemini deep-research', or any second opinion."
argument-hint: "<task> | review <target> | implement <scope> | fix <issues> | research <topic> | deep-research <topic> | resume <topic> | -m <model> | models | help"
allowed-tools: Read, Bash, Grep, Glob
context: session
---

Arguments: **"$ARGUMENTS"**

## Special Commands

If empty or "help": Show usage summary (modes: review, implement, fix, research, deep-research, resume, analyze, general), stop.
If "models": `grep '"name"' ~/.gemini/settings.json | head -1` → show current model, stop.

## Execution

1. **Show current model:** `grep '"name"' ~/.gemini/settings.json | head -1`
2. **Extract model override** if `-m <model>` in arguments
3. **Compile conversation context** — recent exchanges, code discussed, decisions made (keep concise)
4. **Detect intent** from arguments and select settings from the table below
5. **Compose prompt** using the building blocks
6. **Assemble command** and run

### Intent Detection

| Intent | Trigger words | Permission | Approval Mode | Model | Run |
|--------|--------------|------------|---------------|-------|-----|
| **Review** | "review" | Sandbox | `auto_edit` | Flagship | bg |
| **Implement** | "implement", "build", "write", "create", "add" | Unrestricted | `auto_edit` | Flagship | bg |
| **Fix** | "fix", "patch", "resolve", "address" | Unrestricted | `auto_edit` | Flagship | bg |
| **Research** | "search", "research", "google" (not "deep") | Sandbox | `yolo` | Flash (search grounding) | bg |
| **Deep Research** | "deep research", "thorough", "comprehensive analysis" | Own folder | `yolo` | Flash | bg |
| **Resume** | "continue", "follow up", "resume", "go deeper" | Previous folder | `yolo` | Flash | bg |
| **Analyze** | "analyze", "investigate", "audit", "check", "compare" | Sandbox | `auto_edit` | Flagship | bg |
| **General** | anything else | Unrestricted | `auto_edit` | Flagship | bg |

**Key principle:** Default is **unrestricted**. Sandbox only when the task doesn't need source edits (reviews, analysis, research). The orchestrator can override — e.g., `/gemini implement [scope]` always gets unrestricted access.

**Model selection:** Flagship = default model in `~/.gemini/settings.json`. Flash = use `-m gemini-<version>-flash` for research with search grounding. Check `gemini models` for available IDs.

### Compose the Prompt

**Permission block** (pick one based on intent):
- **Unrestricted:** `You have full read and write access to this project.`
- **Sandbox:** `You have full read access to this project. Do not modify existing project source files.`

**Output block** (independent of permission — controls where NEW artifacts go):
- **You (the calling Claude Code instance) decide the output path.** Consider your context: Are you on a feature branch? A code review? A standalone question? Choose a path that makes sense.
- Examples: `Save your review to ./gemini/auth-review.md`, `Write findings to ./docs/reviews/api-audit.md`
- **Be explicit when it matters** — for reviews or multi-step builds, always specify the path. For casual questions, omit (Gemini writes wherever it sees fit).
- This only controls where *new files* are created, not which *existing files* can be edited (that's the permission block's job).

**Role block** (optional — add only when intent benefits from framing):
- **Review:** `You are reviewing code changes for correctness, performance, security, and maintainability. Flag actionable issues with file:line citations. Categorize findings as P0 (critical), P1 (major), or P2 (minor).`
- **Implement/Fix:** Include specific scope, files, and what "done" looks like. Add `Run the project's test suite after implementation and report pass/fail.` when appropriate.
- **Research:** `Search the web and provide a well-sourced answer. Cite sources.`
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
mkdir -p ./gemini && gemini [-m MODEL] -p "[ASSEMBLED PROMPT]" \
  --approval-mode [auto_edit|yolo] \
  2>./gemini/stderr.log; GEMINI_EXIT=$?; if [ $GEMINI_EXIT -ne 0 ]; then echo "GEMINI FAILED (exit $GEMINI_EXIT)"; { echo "=== Exit $GEMINI_EXIT ==="; grep -i 'error\|fatal\|fail\|denied\|unauthorized\|refused\|timeout\|invalid\|limit\|quota\|429' ./gemini/stderr.log | tail -20; echo ""; tail -30 ./gemini/stderr.log; } > ./gemini/errors.log; fi
```

Run in **background** — use `run_in_background: true` on the Bash tool.

### Concrete Examples

**Review (sandbox, flagship model):**
```bash
mkdir -p ./gemini && gemini -p "<instructions>
You have full read access to this project. Do not modify existing project source files.
Save your review report as ./docs/reviews/batch-delete-review.md
You are reviewing code changes for correctness, performance, security, and maintainability. Flag actionable issues with file:line citations. Categorize findings as P0 (critical), P1 (major), or P2 (minor).
</instructions>

<context>
We just implemented a batch delete feature. Changes span src/services/batch/ and tests/batch.test.ts.
</context>

Review the batch delete implementation for correctness and security." \
  --approval-mode auto_edit \
  2>./gemini/stderr.log; GEMINI_EXIT=$?; if [ $GEMINI_EXIT -ne 0 ]; then echo "GEMINI FAILED (exit $GEMINI_EXIT)"; { echo "=== Exit $GEMINI_EXIT ==="; grep -i 'error\|fatal\|fail\|denied\|unauthorized\|refused\|timeout\|invalid\|limit\|quota\|429' ./gemini/stderr.log | tail -20; echo ""; tail -30 ./gemini/stderr.log; } > ./gemini/errors.log; fi
```

**Implement (unrestricted, flagship model):**
```bash
mkdir -p ./gemini && gemini -p "<instructions>
You have full read and write access to this project.
Implement the changes described below. Run npm test after implementation and report pass/fail.
</instructions>

<context>
We need to add a tag merging feature. Design: docs/design/tag-merge.md. Existing tag code: src/models/tag.ts
</context>

Implement the merge_tags function per the design doc. Add it to the exports in index.ts." \
  --approval-mode auto_edit \
  2>./gemini/stderr.log; GEMINI_EXIT=$?; if [ $GEMINI_EXIT -ne 0 ]; then echo "GEMINI FAILED (exit $GEMINI_EXIT)"; { echo "=== Exit $GEMINI_EXIT ==="; grep -i 'error\|fatal\|fail\|denied\|unauthorized\|refused\|timeout\|invalid\|limit\|quota\|429' ./gemini/stderr.log | tail -20; echo ""; tail -30 ./gemini/stderr.log; } > ./gemini/errors.log; fi
```

**Research (sandbox, flash model):**
```bash
mkdir -p ./gemini && gemini -m gemini-3-flash-preview -p "<instructions>
You have full read access to this project. Do not modify existing project source files.
Search the web and provide a well-sourced answer. Cite sources.
</instructions>

<context>
Building a REST API with authentication and rate limiting.
</context>

Research current best practices for API rate limiting in distributed systems. What patterns do production APIs use?" \
  --approval-mode yolo \
  2>./gemini/stderr.log; GEMINI_EXIT=$?; if [ $GEMINI_EXIT -ne 0 ]; then echo "GEMINI FAILED (exit $GEMINI_EXIT)"; { echo "=== Exit $GEMINI_EXIT ==="; grep -i 'error\|fatal\|fail\|denied\|unauthorized\|refused\|timeout\|invalid\|limit\|quota\|429' ./gemini/stderr.log | tail -20; echo ""; tail -30 ./gemini/stderr.log; } > ./gemini/errors.log; fi
```

### Deep Research & Resume

For deep research: create a folder in `~/deep-searches/<topic>/`, run Gemini with `--approval-mode yolo` and flash model. Resume with `-r <index>`. These are longer multi-turn sessions that accumulate research in their own workspace.

### After Completion

- Present Gemini's response to the user
- Note any differences from your own analysis
- Check for artifacts at the output path you specified (if any)
- If failed, read `./gemini/errors.log` for diagnostics

## Error Handling

- **Stale logs:** `errors.log`/`stderr.log` persist across runs. Check `ps aux | grep "gemini -p"` before assuming failure — a running process means logs are stale.
- **Common failures:** 429 rate limit (most common), auth (`gemini auth login`), model unavailable, prompt too long.
- **Rate limits:** Max 2-3 concurrent sessions. 5+ cascade-fail with 429 errors.
