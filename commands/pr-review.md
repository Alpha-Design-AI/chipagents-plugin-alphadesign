---
description: Review a PR — auto-detects current branch or accepts a PR number
argument-hint: [pr_number]
---

Review the pull request for this repository.

If `$ARGUMENTS` is provided, review that PR number. Otherwise, detect the current branch's open PR:

```bash
gh pr view --json number,title,url
```

Steps:

1. Get the PR number (from `$ARGUMENTS` or `gh pr view` above)
2. Fetch the diff: `gh pr diff <number>`
3. List changed files: `gh pr view <number> --json files`
4. Read the PR description: `gh pr view <number> --json number,title,body,author`

Then produce a structured review with these sections:

**Summary** — What this PR does in 2-3 sentences.

**Issues** — Bugs, logic errors, security concerns, or missing error handling. Cite `file:line` for each. If none, write "None found."

**Suggestions** — Non-blocking improvements: naming, clarity, style, test coverage. If none, omit this section.

**Verdict** — One of: `Approve`, `Request Changes`, or `Comment`. Follow with a one-line rationale.
