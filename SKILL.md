---
name: vespa-review
description: Review a GitHub PR or local branch by splitting the diff into focused chapters internally, dispatching a sub-agent per chapter in parallel, then filtering false positives and emitting a single consolidated feedback report.
---

# Vespa Review

## Purpose

Produce one high-signal review of a code change by:

1. Splitting the diff into intent-grouped chapters (internal — never shown to the user).
2. Dispatching one sub-agent per chapter in parallel with a focused rubric.
3. Re-verifying every returned concern against the actual code.
4. Emitting one consolidated feedback report.

Chapters exist only to give each sub-agent a tight focus area. They are scaffolding, not output.

## Triggers

Use this skill when the user asks for a non-interactive review of:

- A GitHub PR.
- A local branch with an identifiable base and head ref.

Do **not** use this skill for:

- Interactive section-by-section walkthroughs (use a guided/interactive review skill instead).
- Single-file or single-function reviews — the multi-agent overhead exceeds the benefit.
- Quick spot-checks where the user wants conversation, not a report.

## Prerequisites

- A `Task` tool (or equivalent sub-agent dispatcher) must be available. Without it, abort and tell the user this skill needs sub-agent dispatch.
- A working tree where `git diff <base>..<head>` returns a sensible diff.

## Workflow

### 1. Resolve refs

- **PR**: `gh pr view <n> --json baseRefName,headRefName,headRefOid,baseRefOid`. Fetch both refs locally if missing (`git fetch origin <ref>`).
- **Branch**: `<base>` defaults to `origin/main` (fall back to `origin/master`); `<head>` is the current branch tip.

Record `BASE` and `HEAD` for the rest of the run.

### 2. Plan chapters (silent)

Inspect the changed surface:

```sh
git diff --stat $BASE..$HEAD
git diff --name-only $BASE..$HEAD
```

Group files into chapters by **intent**, not filename. Good boundaries:

- API surface / public boundary
- Data flow / persistence
- Behavior changes
- Migrations
- Tests
- Cleanup / refactors
- User-facing changes

Aim for 3–7 chapters. Combine trivially small changes into a single "cleanup" chapter. Each chapter is `{ id, intent, files: [...] }`. Do **not** show this plan to the user.

### 3. Dispatch one sub-agent per chapter, in parallel

Issue one `Task` call per chapter in a single turn so they run concurrently. Use the prompt template below, filling the placeholders.

**Sub-agent prompt template:**

```
You are reviewing one chapter of a larger code change. Other agents are
reviewing the other chapters in parallel; a parent agent will consolidate
everything afterwards.

Chapter: <chapter_id> — <intent>
Files: <files>
Base ref: <BASE>
Head ref: <HEAD>

Read the diff for these files with `git diff <BASE>..<HEAD> -- <files>`. Read
the post-change files as needed for context.

Analyse along these seven pillars — every concern you report must belong to one:

- Correctness — bugs, logical errors, off-by-one, wrong branch taken.
- Maintainability — structure, modularity, adherence to existing repo patterns.
- Readability — naming, non-obvious comments, formatting consistency.
- Efficiency — perf or resource regressions introduced by this change.
- Security — injection, auth bypass, secret handling, unsafe deserialization.
- Edge cases & error handling — null / empty / overflow / concurrency / failure paths.
- Testability — missing or weak coverage for the new or modified code.

Each concern must:
1. Follow from the actual code, not a guess.
2. Not already be neutralised by surrounding code (caller, callee, sibling branches).
3. Be worth the reviewer's attention. Drop nitpicks, style preferences, and restatements of the diff.

Label each `high`, `medium`, or `low`. Use language a 10-year-old could follow.

Return one JSON object and nothing else — no prose, no fenced code blocks:

{
  "concerns": [
    { "text": "...", "severity": "medium", "file_path": "src/...", "line": 24 }
  ]
}

If you find nothing worth reporting, return `{ "concerns": [] }`.
```

### 4. Verify every returned concern (parent-side false-positive pass)

For **every** concern from every sub-agent, before it reaches the report:

1. Open the cited `file_path` at the cited `line`. Confirm the code there actually exhibits the issue described.
2. Inspect surrounding code (caller, callee, sibling branches in the same function). Confirm the concern isn't already handled.
3. Confirm it's worth the reviewer's attention. Drop nitpicks, style nits, and items that just restate what the diff shows.

Silently drop any concern that fails the check. Do **not** surface a "removed" list — silent filtering keeps the report focused.

If a sub-agent returned malformed JSON or failed outright, redo that chapter's analysis yourself in the same turn under the same rubric and same false-positive criteria.

### 5. Emit the consolidated report

Group surviving concerns by **severity**, high first. Within a severity group, one concern per line:

```
[high] src/api/handlers.rs:24 — Empty body returns 200; should it be 400?
[medium] src/api/validation.rs:88 — Length check off-by-one against MAX_INPUT.
[low] src/api/handlers.rs:142 — No test for the rate-limited path.
```

End with a one-line tally: `Total: 1 high, 1 medium, 1 low.`

If everything was filtered out, emit a single line: `No actionable concerns.`

Do **not** mention chapters in the output. Do **not** describe what the PR does. Do **not** add a TL;DR. The output is feedback only.

## Non-negotiables

1. Chapters are internal scaffolding. Never shown to the user.
2. Output is consolidated feedback only — no walkthrough, no diff summary, no chapter headers.
3. Every concern is verified by the parent before it ships, regardless of which sub-agent produced it.
4. One severity label per concern: `high`, `medium`, `low`.
5. The seven-pillar rubric is the only frame. Do not invent new categories.
6. Dispatch sub-agents in parallel. Sequential dispatch defeats the design.
