---
name: release
description: Cut a new release (patch or minor). Usage: /release vX.Y.Z [pr1 pr2 ...] — PR numbers are optional; if omitted, commits since the last release are surfaced for selection.
disable-model-invocation: true
---

## Current State

- Current branch: !`git branch --show-current`
- Existing release branches: !`git branch -a | grep "release/v" | sed 's/.*release\//release\//' | sort -V | tail -10`
- Latest tags: !`git tag --sort=-v:refname | head -5`
- Commits since last tag: !`git log $(git describe --tags --abbrev=0)..origin/main --oneline --no-merges 2>/dev/null || git log origin/main --oneline --no-merges -30`
- cli version: !`node -e "process.stdout.write(require('./cli/package.json').version)"`
- backend version: !`node -e "process.stdout.write(require('./backend/package.json').version)"`

## Arguments

$ARGUMENTS

Parse the arguments as: `<version> [pr-number ...]`

- If the version patch component is > 0 (e.g. v0.14.1), this is a **patch release**.
- If the patch component is 0 (e.g. v0.15.0), this is a **minor release**.
- Any remaining tokens are explicit PR numbers whose commits must be cherry-picked.
- If no PR numbers are provided, follow **Step 0** below to discover and confirm which commits to include.

---

## Hard Rules (enforce throughout)

- Version bumps (`package.json`) only happen in release PRs — never in feature/fix commits.
- All changes to `release/vX.Y.x` must go through a PR. Never push directly to the long-lived branch.
- `CHANGELOG.md` entries marked `[INTERNAL]` are NOT included in `docs/src/content/docs/changelog.mdx`.
- Ticket links (e.g. `[CHI-123](...)`) go in `CHANGELOG.md` only — not in `changelog.mdx`.
- Back-merges into `main` must use a **merge commit**, not squash.
- Never credit AI tools in commit messages.
- Use `git commit -F - <<'EOF'` for multi-line commit messages (not command substitution).

---

## Step 0 — Discover commits (only when no PRs provided)

Get all commits on `main` since the last release tag. This repo uses squash merges, so use
`--no-merges` and anchor explicitly to `origin/main` so discovery works from any branch:

```bash
git log $(git describe --tags --abbrev=0)..origin/main --oneline --no-merges
```

For a **minor release** (vX.Y.0), find the previous minor tag (e.g. for v0.15.0, use v0.14.0)
rather than the latest tag, to capture all commits since the prior minor:

```bash
# Find previous minor tag: strip patch component and search backwards
git tag --sort=-v:refname | grep -E "^v[0-9]+\.[0-9]+\.0$" | sed -n '2p'
# Then use: git log <prev-minor-tag>..origin/main --oneline --no-merges
```

For each commit, extract the PR number from the title (e.g. `(#1234)`). Fetch each PR's
title, body, and labels for context:

```bash
gh pr view <pr-number> --json number,title,body,labels
```

Group by type (fix, feat, chore, docs, etc.) and present a summary. Then use **AskUserQuestion**:

> "Here are the PRs merged since vX.Y.W. Which should be included in vX.Y.Z?
>
> **Fixes**
>
> - #NNNN short title
>
> **Features / Changes**
>
> - #NNNN short title
>
> **Chores / Internal**
>
> - #NNNN short title
>
> Reply with the PR numbers to include, or 'all' to include everything listed."

Use the confirmed list as the set of PRs for the rest of the steps.

---

## Step 1 — Documentation review

Before touching any files, read the diff for each included PR:

```bash
gh pr diff <pr-number>
```

For each PR, look for:

- New CLI flags, commands, config options, or environment variables
- Changed behavior for existing user-facing features
- New or removed API endpoints
- Changed user-facing error messages or UI text
- New setup or configuration requirements

Then scan the docs directory to check for existing coverage:

```bash
ls docs/src/content/docs/
```

Compile a list of anything that appears undocumented or out of date. If there is anything to
flag, use **AskUserQuestion**:

> "Before cutting vX.Y.Z, I noticed these changes may need documentation updates:
>
> - **#NNNN** adds `--foo` flag — not currently covered in docs/
> - **#NNNN** changes behavior of `/command` — docs/commands.mdx may be stale
>
> Would you like to update docs now, open follow-up issues, or skip and proceed?"

If there is nothing to flag, or the user says skip, proceed immediately without commenting on it.
Do NOT flag purely internal changes (no user-visible surface area).

---

## Patch Release Steps (vX.Y.Z where Z > 0)

Patch releases use **two separate PRs** into `release/vX.Y.x`:

1. **`cherry-picks/vX.Y.Z`** — all code changes (cherry-picks from main). Merge this first.
2. **`release/vX.Y.Z`** — version bump + changelog only. Open after the cherry-picks PR merges.

Both PRs must be merged with a **merge commit** (not squash) so histories stay linked.

### 2. Resolve PR commits

For each confirmed PR, find its squash-merge commit SHA on `origin/main`. This repo uses
squash merges, so each PR lands as a single non-merge commit with `(#NNNN)` in the title.
Use an exact-match pattern to avoid false matches (e.g. `#12` matching `#123`):

```bash
git log origin/main --oneline --no-merges | grep -E "\(#<pr-number>\)$"
```

Collect SHAs in chronological order (oldest first — reverse the log output).

### 3. Create the cherry-picks branch

```bash
git checkout release/vX.Y.x && git pull
git checkout -b cherry-picks/vX.Y.Z
```

### 4. Cherry-pick commits

Apply each commit in chronological order:

```bash
git cherry-pick <sha1> [sha2 ...]
```

Resolve any conflicts before proceeding.

### 5. Push and open cherry-picks PR

```bash
git push -u origin cherry-picks/vX.Y.Z
gh pr create --base release/vX.Y.x --head cherry-picks/vX.Y.Z \
  --title "cherry-picks: vX.Y.Z" \
  --body "Cherry-picks from main for vX.Y.Z:

- #NNNN short title
- #NNNN short title"
```

Note the PR number from the output.

### 6. Poll for cherry-picks PR merge — PAUSE POINT A

Poll the PR status every 30 seconds. At each iteration run:

```bash
gh pr view <pr-number> --json state,reviews,comments
```

Handle each outcome:

- **`state` is `MERGED`**: proceed immediately to step 7.
- **New review comment appears**: surface the comment text to the user in full, then use
  **AskUserQuestion** to ask how they'd like to respond (address the feedback, push a fix, etc.).
  Resume polling after the user replies.
- **20 polls pass (~10 minutes) with no merge**: stop polling and use **AskUserQuestion**:

  > "The cherry-picks PR `cherry-picks/vX.Y.Z -> release/vX.Y.x` hasn't merged yet. Let me
  > know when it's merged, or if there are any issues to address."

  Wait for the user's reply before continuing. If they say it's merged, proceed to step 7.
  If they provide feedback, address it, push a new commit, and resume polling.

### 7. Create the release branch

After the cherry-picks PR is merged, pull the updated release line and create the release branch:

```bash
git checkout release/vX.Y.x && git pull
git checkout -b release/vX.Y.Z
```

### 8. Bump versions

Edit `cli/package.json` and `backend/package.json`: change `"version"` to `"X.Y.Z"`.

Then run:

```bash
bun install
```

### 9. Update CHANGELOG.md

Add a new section immediately after `## [Unreleased]`:

```markdown
## [vX.Y.Z] - YYYY-MM-DD

### Fixed

- Description of fix ([CHI-NNN](https://linear.app/chipagents/issue/CHI-NNN))
- [INTERNAL] Internal fix description (#pr-number)

### Changed

- [INTERNAL] Internal change description (#pr-number)
```

Rules:

- Use today's date.
- Include `[INTERNAL]` prefix for anything not directly user-facing.
- Link ticket numbers as markdown links in this file.
- Only include change types that have entries (omit empty sections).

### 10. Update docs/src/content/docs/changelog.mdx

Insert a new `<div>` block at the very top (before the existing first `<div>`), containing only
**user-facing** entries (no `[INTERNAL]`, no ticket links):

```mdx
<div className="bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-700 rounded-lg px-6 pt-6 pb-6 shadow-sm">

## [vX.Y.Z] - YYYY-MM-DD

### Fixed

- Description of fix

</div>
```

If all entries are `[INTERNAL]`, add the block with a single generic line:

```markdown
### Fixed

- Internal bug fixes and stability improvements
```

### 11. Commit

Stage all changes and commit:

```
release: vX.Y.Z
```

(Single-line commit message, no AI attribution.)

### 12. Push and open release PR

```bash
git push -u origin release/vX.Y.Z
gh pr create --base release/vX.Y.x --head release/vX.Y.Z --title "release: vX.Y.Z" \
  --body "Version bump, changelog, and docs update for vX.Y.Z."
```

Note the PR number from the output.

### 13. Poll for release PR merge — PAUSE POINT B

Poll the PR status every 30 seconds. At each iteration run:

```bash
gh pr view <pr-number> --json state,reviews,comments
```

Handle each outcome:

- **`state` is `MERGED`**: proceed immediately to step 14.
- **New review comment appears**: surface the comment text to the user in full, then use
  **AskUserQuestion** to ask how they'd like to respond (address the feedback, push a fix, etc.).
  Resume polling after the user replies.
- **20 polls pass (~10 minutes) with no merge**: stop polling and use **AskUserQuestion**:

  > "The release PR `release/vX.Y.Z -> release/vX.Y.x` hasn't merged yet. Let me know when
  > it's merged, or if there are any issues to address."

  Wait for the user's reply before continuing. If they say it's merged, proceed to step 14.
  If they provide feedback, address it, push a new commit, and resume polling.

### 14. Tag and push

```bash
git checkout release/vX.Y.x && git pull
git tag vX.Y.Z
git push origin vX.Y.Z
```

### 15. Open backmerge PR — PAUSE POINT C

Create a branch from `main`, cherry-pick only the release commit (version bump + changelog + docs):

```bash
git checkout main && git pull
git checkout -b backmerge/vX.Y.Z
git cherry-pick <release-commit-sha>
git push -u origin backmerge/vX.Y.Z
gh pr create --base main --head backmerge/vX.Y.Z \
  --title "docs: back-merge vX.Y.Z release notes into main" \
  --body "Back-merges the vX.Y.Z version bump and changelog into main."
```

Then use **AskUserQuestion**:

> "Backmerge PR is open at <pr-url>. Please review and merge it when ready (use a merge commit,
> not squash), then let me know to continue with deploy and AE notification."

Wait for the user's reply. Also poll `gh pr view <backmerge-pr-number> --json state` every 30
seconds (up to 20 times) — if it merges before the user replies, proceed automatically.

### 16. Deploy

Before deploying, wait for CI/CD to finish building and publishing the tag image — `deploy:vpc`
checks that the version tag exists in the registry before retagging and will fail if run too
early. Poll until the tag workflow is green:

```bash
gh run list --limit 10 | grep vX.Y.Z
gh run watch <run-id>
```

Once CI is green, prompt the user to deploy:

```bash
bun run deploy:prod vX.Y.Z
bun run deploy:vpc vX.Y.Z
```

See `deploy/README.md` for details.

### 17. Notify AE team

Once the user has confirmed the release is deployed in the appropriate environments, post in the AE channel using the following format. Prepare the message in a copy pasteable format for them (e.g. prepare a temp markdown file):

```
Hi @appeng. Just released vX.Y.Z of chipagents CLI. Please distribute the following to
stakeholders

External:
[paste public changelog.mdx entry]

Internal:
[paste CHANGELOG.md entry including [INTERNAL] lines]
```

---

## Minor Release Steps (vX.Y.0 where Y is new)

Steps 0 and 1 (commit discovery and docs review) apply identically. For minor releases, Step 0
should surface all commits since the previous minor tag (e.g. since v0.13.0 for v0.14.0).

### 2. Cut the long-lived release branch from main

```bash
git checkout main && git pull
git checkout -b release/vX.Y.x main
git push -u origin release/vX.Y.x
```

### 3. Prepare the release commit

Create a short-lived branch off the new long-lived branch:

```bash
git checkout -b release/vX.Y.0
```

Then:

- Bump `"version"` to `"X.Y.0"` in `cli/package.json` and `backend/package.json`
- Run `bun install`
- Update `CHANGELOG.md`: convert `## [Unreleased]` into `## [vX.Y.0] - YYYY-MM-DD`
- Update `docs/src/content/docs/changelog.mdx`: add new `<div>` block at top

Commit as `release: vX.Y.0`, then:

```bash
git push -u origin release/vX.Y.0
gh pr create --base release/vX.Y.x --head release/vX.Y.0 --title "release: vX.Y.0" \
  --body "Version bump, changelog, and docs update for vX.Y.0."
```

**PAUSE POINT A** — Poll and wait exactly as described in patch step 10. Once merged, proceed.

### 4. Stabilize

Cherry-pick critical fixes from `main` via PRs into `release/vX.Y.x` as needed.

### 5. Tag and push (branch + tag together)

```bash
git checkout release/vX.Y.x && git pull
git tag vX.Y.0
git push origin release/vX.Y.x vX.Y.0
```

### 6. Open backmerge PR — PAUSE POINT B

For minor releases, cut the backmerge branch from `release/vX.Y.x` (not `main`) so the PR
carries full release-line ancestry, which makes it easier to audit stabilization commits:

```bash
git checkout release/vX.Y.x && git pull
git checkout -b backmerge/vX.Y.0
git push -u origin backmerge/vX.Y.0
gh pr create --base main --head backmerge/vX.Y.0 \
  --title "docs: back-merge vX.Y.0 release notes into main" \
  --body "Back-merges the vX.Y.0 version bump and changelog into main."
```

Wait exactly as described in patch step 12 — use **AskUserQuestion** and/or poll for merge.
Back-merge must use a **merge commit**, not squash.

### 7. Deploy and notify

Same as patch steps 13-14 above.

> **SDK note**: CI/CD automatically builds Python SDK wheels on every tag. No separate SDK release step needed.
