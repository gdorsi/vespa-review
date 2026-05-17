# vespa-review

A single-skill repository for the `vespa-review` agent skill.

Vespa Review produces one high-signal review of a GitHub PR or local branch by
splitting the diff into intent-grouped chapters internally, dispatching one
sub-agent per chapter in parallel, then filtering false positives in the parent
agent before emitting a single consolidated feedback report.

Chapters are scaffolding for sub-agent focus — they are never shown to the user.
The output is feedback only: no walkthrough, no diff summary, no chapter
headers.

## Install

After pushing this repository to GitHub, install it with:

```bash
npx skills add <owner>/vespa-review
```

Or by URL:

```bash
npx skills add https://github.com/<owner>/vespa-review
```

## Contents

- `SKILL.md`: the skill definition consumed by agents and the `skills` CLI
- `agents/openai.yaml`: UI metadata for compatible agents
