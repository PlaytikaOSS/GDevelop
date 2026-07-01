---
name: release-beta-editor
description: >-
  Release a beta version of the GDevelop editor by rebasing the
  release/beta-editor branch onto a chosen upstream 4ian/GDevelop release and
  merging a chosen PlaytikaOSS/GDevelop feature branch on top, then opening a
  PR. Use when the user asks to cut/prepare/release a beta editor build or
  update release/beta-editor to a new GDevelop version.
disable-model-invocation: true
---

# Release Beta Editor

Prepare a beta editor release for `PlaytikaOSS/GDevelop`: reset `release/beta-editor`
to a chosen upstream release, re-apply a chosen feature branch, and open a PR back
into `release/beta-editor`.

## Result

Each run produces `release/beta-editor` content = **one chosen release tag + one
chosen feature branch** (plus the preserved paths - `build-editor.yml` and this
skill itself, see Environment). Feature branches do not stack up across runs:
Step 3 hard-resets the tree to the release tag (`git read-tree --reset`), so any
previously merged feature is dropped from the content before the new one is
merged. To combine several feature branches in a single beta, merge each of them
in Step 4 (the skill otherwise assumes one).

Only the git *history* of `release/beta-editor` grows (each run branches off it and
the PR merges back), so past release/merge commits stay in the log. That does not
affect the delivered files - the built editor is always release tag + current feature.

## Environment

- Works on Windows and macOS: run each git/gh command separately rather than
  chaining with shell-specific operators (`&&`, `;`).
- Repos:
  - Upstream (releases): `4ian/GDevelop`
  - Fork (branches, target): `PlaytikaOSS/GDevelop`
- Target branch: `release/beta-editor`
- Default source branch: `feature/breakpoints`
- Protected paths (must keep the `release/beta-editor` version, never overwrite from
  the release - the release tag has neither, since both are fork-only):
  - `.github/workflows/build-editor.yml`
  - `.cursor/skills/release-beta-editor/SKILL.md` (this skill - Step 3's tree reset
    would otherwise delete it on every run)

## GitHub access

For any GitHub read/write (releases, branches, PRs), prefer whatever GitHub MCP
server is available: discover its tools first, then call the one that fits. Fall
back to the `gh` CLI only if no GitHub MCP is available or it lacks the needed tool.

## Workflow

```
- [ ] Step 1: Pick base upstream release
- [ ] Step 2: Pick source branch
- [ ] Step 3: Reset release/beta-editor content to the release (keep protected paths)
- [ ] Step 4: Merge source branch, resolve conflicts
- [ ] Step 5: Push working branch and open PR into release/beta-editor
```

### Step 1: Pick base upstream release

List the latest 10 releases of `4ian/GDevelop` and let the user choose one.

- GitHub MCP: call its releases-listing tool for `owner=4ian, repo=GDevelop`, limit 10.
- Fallback: `gh api repos/4ian/GDevelop/releases --jq ".[0:10][].tag_name"`

Present the 10 versions with `AskQuestion` (newest first). Record the chosen
`<version>` (e.g. `5.6.273`); the git tag is `v<version>`.

### Step 2: Pick source branch

Default is `feature/breakpoints`. Offer it as the recommended option, and also
offer the 5 most recently updated fork branches.

Fetch and compute the top 5 recent branches:

```
git fetch origin --prune --tags
git for-each-ref --sort=-committerdate --count=5 --format="%(refname:short)  %(committerdate:relative)" refs/remotes/origin
```

Present via `AskQuestion` with `feature/breakpoints` first (recommended) followed by
the 5 recent branches. Record the chosen `<source-branch>`.

### Step 3: Reset content to the release (keep protected paths)

Ensure the upstream release tag is available locally, then create a working branch
off `release/beta-editor` whose tree equals the release tag, restoring the protected
paths listed in Environment.

Make sure a remote named `upstream` points at `https://github.com/4ian/GDevelop.git`.
This is a one-time setup; skip it if the remote already exists:

```
git remote add upstream https://github.com/4ian/GDevelop.git
```

```
git fetch upstream --tags
git checkout -B release/beta-editor-<version> origin/release/beta-editor
git read-tree -u --reset refs/tags/v<version>
git checkout origin/release/beta-editor -- .github/workflows/build-editor.yml
git checkout origin/release/beta-editor -- .cursor/skills/release-beta-editor/SKILL.md
git add -A
git commit -m "Align beta editor with GDevelop <version> (keep protected paths)"
```

`read-tree -u --reset` sets the working tree/index to the release tag while keeping
`HEAD` on `release/beta-editor`, so the commit stays a child of `release/beta-editor`
(the resulting PR is a clean change against it).

### Step 4: Merge source branch, resolve conflicts

```
git merge --no-ff --no-edit origin/<source-branch>
```

If it stops on conflicts:

- Resolve each conflicted file, keeping the intended beta behavior.
- Never resolve a protected path (see Environment) toward the release/upstream version — keep the fork's version.
- `git add <resolved-files>` then `git commit --no-edit`.
- Verify no markers remain: `git grep -n "^<<<<<<< "` returns nothing.

### Step 5: Push and open PR

```
git push -u origin release/beta-editor-<version>
```

Open a PR with base `release/beta-editor`, head `release/beta-editor-<version>`.

- GitHub MCP: call its create-PR tool (`owner=PlaytikaOSS, repo=GDevelop, base=release/beta-editor, head=release/beta-editor-<version>`).
- Fallback: `gh pr create --repo PlaytikaOSS/GDevelop --base release/beta-editor --head release/beta-editor-<version> --title "Beta editor: GDevelop <version> + <source-branch>" --body "..."`

PR title: `Beta editor: GDevelop <version> + <source-branch>`.
PR body: base release version, merged source branch, and a note that the protected
paths (`.github/workflows/build-editor.yml`, this skill) were preserved from
`release/beta-editor`.

## Constraints

- Do not modify any protected path (see Environment) at any step.
- Do not force-push or overwrite `release/beta-editor` directly; always go through the PR.
- Confirm the chosen version and source branch with the user before Step 3.
