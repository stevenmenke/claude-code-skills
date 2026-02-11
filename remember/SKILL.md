---
name: remember
description: Document decisions, context, and learnings so the next session can continue without re-deriving understanding. Use continuously while working, not just at session end. Use when asked to "remember this", "document what we did", "save context", or before ending a session.
---

Your memory resets completely each session. Prior Claude instances are strangers who left notes — there is no recollection of their work. Only what's documented in files persists.

**Continuously while working** (not just at session end), maintain:

### CLAUDE.md (keep lean — loaded every session)
- Project conventions and patterns
- Stable gotchas that apply across sessions

### CHANGELOG.md (session context lives here)
```
## Next Actions
- [ ] Immediate priorities with enough context to start - include full path to current plan file

## [Date] Session
- Session ID: ${CLAUDE_SESSION_ID}
- To read full transcript: `cat ~/.claude/projects/*/${CLAUDE_SESSION_ID}.jsonl`
- Decisions made and *why*
- What was accomplished (file:line references)
- Learnings, gotchas, dead ends discovered
- Blockers or open questions
```

**Mindset**: Write for a capable successor who knows nothing about today's conversation. Could they continue mid-task without re-deriving your understanding?
