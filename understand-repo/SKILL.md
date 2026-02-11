---
name: understand-repo
description: Deeply understand any public GitHub repo using DeepWiki MCP for AI-powered architecture analysis and GitHub CLI for reading source code. Use when asked "how does X work", "look at the source of", "investigate library internals", "debug dependency behavior", "evaluate this library", "what's the implementation of", "check how repo implements", or "extract/rip out feature from". Covers understanding implementations, tracing code paths, evaluating libraries before adoption, debugging dependency issues, and extracting self-contained functionality.
allowed-tools: Read, Write, Bash, Grep, Glob, WebFetch, Task
---

# Understand Any Public GitHub Repo

Use DeepWiki MCP for AI-powered architecture analysis and GitHub CLI for reading actual source code. Together they provide expert-level understanding of any public repo — as if having worked in the codebase for months.

## When to Use

- "How does library X implement feature Y?"
- "Look at the source of this dependency"
- "Debug weird behavior from a library"
- "Evaluate this library before adopting it"
- "What's the architecture of this project?"
- "Extract/rip out a feature into self-contained code"
- Docs are outdated but the code is the source of truth
- Need to contribute to an open-source project

## Instructions

### Phase 1: Understand

1. **Map the repo** — Call `read_wiki_structure` for `{owner}/{repo}` to get architecture overview and identify key areas.

2. **Ask targeted questions** — Use `ask_question` to understand:
   - How is `{feature}` implemented?
   - What are the key files and modules involved?
   - What are the subtle implementation details and edge cases?
   - What internal dependencies does this feature have?

3. **Read the actual source** — Use GitHub CLI to fetch specific files identified by DeepWiki:
   ```bash
   gh api repos/{owner}/{repo}/contents/{path} -q .content | base64 -d
   ```
   For listing files:
   ```bash
   gh api "repos/{owner}/{repo}/git/trees/{branch}?recursive=1" -q '.tree[].path' | grep {pattern}
   ```

4. **Trace dependencies** — Follow internal imports until reaching standard library or language primitives. Document what each piece does.

### Phase 2: Apply (varies by goal)

#### If understanding/evaluating:
5. Summarize architecture, key files, complexity, and any concerns.
6. Answer the specific question with source-level evidence.

#### If debugging:
5. Trace the code path that leads to the observed behavior.
6. Identify the root cause with file and line references.

#### If extracting:
5. **Plan the extraction** — Outline which functions/classes to extract, which utilities to inline, and the target API surface.
6. **Implement self-contained version** — Reproduce the API exactly, inline internal utilities, add comments explaining non-obvious choices.
7. **Write equivalence tests** — Prove the extracted version matches the original with identical inputs/outputs.
8. **Verify** — Run tests, confirm no imports from the original remain.

## Examples

### Understanding

```
How does vercel/next.js implement server components? Use DeepWiki and source to explain the architecture.
```

### Debugging

```
I'm getting a weird error from fast-xml-parser when parsing CDATA. Use DeepWiki and gh to trace how CDATA handling works internally.
```

### Evaluating

```
Should we adopt drizzle-orm? Use DeepWiki to understand its architecture, check the query builder implementation, and assess complexity.
```

### Extracting

```
Use DeepWiki and GitHub CLI to understand how pytorch/ao implements fp8 training.
Implement fp8.py that has identical API but is fully self-contained with no torchao dependency.
Write tests proving equivalent results.
```

```
Use DeepWiki and GitHub CLI to understand how date-fns implements format().
Extract a self-contained date-format.ts with zero dependencies.
Write tests comparing output against date-fns.
```

## Guidelines

- Start with `read_wiki_structure` to map the repo, then `ask_question` for targeted understanding
- Use `gh api` to read specific source files identified by DeepWiki
- For extraction: always write equivalence tests comparing against the original
- Comment non-obvious implementation details — making hidden knowledge explicit is the whole point
- When extracting: keep code minimal, only what the use case needs
- If a feature is too deeply entangled, report that finding rather than forcing extraction

## Common Issues

- **Private repo**: DeepWiki only indexes public repos. For private repos, use `gh api` directly to read source files and skip DeepWiki
- **Large repos**: Start broad with `read_wiki_structure`, then narrow with targeted `ask_question` calls before reading source
- **Extraction too complex**: If the dependency chain is massive, consider extracting a simplified subset or recommending keeping the dependency

## Setup

This skill requires two things: the DeepWiki MCP server and GitHub CLI.

**DeepWiki MCP** — add once, available in all sessions:
```bash
claude mcp add -s user -t http deepwiki https://mcp.deepwiki.com/mcp
```

**GitHub CLI** — install and authenticate:
```bash
brew install gh   # macOS
gh auth login
```

Verify both work: run `/mcp` in Claude Code and confirm `deepwiki` appears with tools `read_wiki_structure`, `read_wiki_contents`, `ask_question`.

**Install this skill** — save to `~/.claude/skills/understand-repo/SKILL.md` (global) or `.claude/skills/understand-repo/SKILL.md` (project-level).
