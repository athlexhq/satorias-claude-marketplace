---
name: update-changelog
description: >-
  Write a new versioned CHANGELOG.md entry on the default branch from the
  commits made
  since the last release tag. Use when cutting a release. Not for feature
  branches or individual PRs.
allowed-tools: Bash(git branch:*), Bash(git status:*), Bash(git log:*), Bash(git describe:*), Bash(git show:*), Bash(git fetch:*), Bash(git rev-list:*), Bash(git symbolic-ref:*), Bash(git remote:*), Bash(gh pr view:*), Bash(sed:*), Bash(grep:*), Read, Edit
user-invocable: true
---

# About
The format is based on [Keep a Changelog](http://keepachangelog.com/),
and the project adheres to [Semantic Versioning](http://semver.org/).

The changelog is only ever updated on the repo's default branch, never on a
feature branch, and only from commits that have already landed there — run
this after the relevant pull requests are merged, not before.

# Step 1 - gather state
Run this block with Bash before doing anything else.

```bash
BASE=$(git symbolic-ref --quiet --short refs/remotes/origin/HEAD 2>/dev/null | sed 's@^origin/@@')
[ -z "$BASE" ] && BASE=$(git remote show origin 2>/dev/null | sed -n 's/.*HEAD branch: //p')
if [ -z "$BASE" ]; then BASE=main; echo "could not detect default branch, falling back to: $BASE"; fi
echo "default branch: $BASE"
BRANCH=$(git branch --show-current)
echo "branch: $BRANCH"
if [ "$BRANCH" != "$BASE" ]; then
  echo "NOT ON THE DEFAULT BRANCH - stop here."
else
  echo "clean tree: $([ -z "$(git status --porcelain)" ] && echo yes || echo no)"
  git fetch --tags --quiet
  echo "behind origin/$BASE: $(git rev-list --count "HEAD..origin/$BASE")"
  TAG=$(git describe --tags --abbrev=0 2>/dev/null)
  if [ -z "$TAG" ]; then
    echo "last tag: none (no prior release)"
    echo "--- all commits (no prior tag) ---"
    git log --oneline --no-merges
  else
    echo "last tag: $TAG"
    echo "--- commits since $TAG ---"
    git log "$TAG"..HEAD --oneline --no-merges
  fi
  if [ -f CHANGELOG.md ]; then
    echo "--- versions already in CHANGELOG.md ---"
    grep -m 5 -E '^# \[[0-9]+\.[0-9]+\.[0-9]+\]' CHANGELOG.md
  else
    echo "CHANGELOG.md: NOT FOUND - stop here, ask the user how to proceed"
  fi
fi
```

# Step 2 - abort conditions
Stop, report to the user, and change nothing if any of these hold. The block
above only reads state — it will not fix them for you.

 - the branch is not the default branch reported in Step 1
 - CHANGELOG.md does not exist
 - the tree is not clean — the uncommitted work would land in the changelog
   commit
 - the branch is behind its origin counterpart — ask the user to pull first,
   otherwise the entry misses commits that are already released

# Source of the entries
Write the entries from the commit messages listed above — the commits made
since the last release tag. Each commit is one semantical task; map it to a
category below.

If the repo's commits follow the `create-github-pr` skill's `action/tag/task`
title format, the action tag maps directly to a category — e.g.
`system/update/bump-postcss-from-8.5.15-to-8.5.24` is a System entry, and
`security/patch/bump-rack-cve-2026-1234` is a Security entry. Rewrite them as
readable prose, do not paste the raw branch name.

If the repo does not follow that convention, categorize from the commit
content instead, falling back to the PR/diff as described below.

Commit subjects are branch names and are often too terse to describe the change
well. When a subject does not carry enough meaning, read the pull request
instead. This assumes PRs are squash-merged, so a subject ending in a PR
number — `add/integration/webhook-retries (#1481)` — is read with
`gh pr view 1481`. If the repo merges a different way and no PR number is
present, fall back to `git show --stat <sha>` for that commit.

# Categories
There are six main categories. Every PR can contain one, or more, semantical
tasks that can be put in one of the categories.
Each task needs to be expressed succinctly and must fit on one line.

 - Security (patching vulnerabilities, upgrading security packages, etc.)
 - System (dependency upgrades, or changes in deployment, devops, etc.)
 - Fix (patches, hotfixes)
 - Add
 - Remove
 - Change

# Release Header
Every run of this skill writes exactly one new versioned header, since the
skill only runs when a release is being cut. Header must contain:
 - write version inside the square brackets
 - add a date to the version in the format "dd-mm-yyyy"

Abstract example: [4.X.X] - XX-XX-2024

Concrete example: [6.6.5] - 13-07-2026

The version must be greater than the highest version already present in
CHANGELOG.md. Read the file first and increment from that — never reuse an
existing version number.

Default to a patch bump, whatever the categories contain — an Add-only
release still takes a patch bump unless the user says otherwise.

A minor bump marks a deliberate milestone that accumulates a lot of work. A
major bump is never implied by the commits. Propose either only when the user
has said a release is one of those, and confirm before writing the header.

# The Format
The new entry goes above the most recent released version and below the
template block at the top of the file.

The template block is a permanent fixture of the file, not an entry to fill in.
Leave it in place, untouched, and write the new versioned entry below it. Never
delete it.

This is the template, showing all categories in the order they must always
appear: Security, System, Fix, Add, Remove, Change. Only include the
categories that the commits since the last tag actually produced, omitting
the rest, but never reorder the ones you do include.
> # [6.x.x] - xx-07-2026
> ## Security
>  - ...
> 
> ## System
>  - ...
> 
> ## Fix
>  - ...
> 
> ## Add
>  - ...
> 
> ## Remove
>  - ...
> 
> ## Change
>  - ...

And this is an example of the actual entry. The entry lines are deliberately
left unwrapped, because each task must fit on a single line.
> # [6.6.0] - 15-06-2026
> ## Security
>  - bump rack from 3.1.2 to 3.1.5 to patch CVE-2026-1234
> 
> ## System
>  - fix `bin/bundle exec rails console` pry issue
>  - locally install extensions for stringio, io-console, and date gems
>  - bump binding_of_caller gem
> 
> ## Add
>  - expand the export feature:
>   - support CSV in addition to JSON
>   - let the user schedule a recurring export
>   - cap exports older than 60 days from being re-downloaded
>   - option to share an export link with teammates
>   - styling and UI improvements
> 
> ## Change
>  - billing: use the invoice date instead of the payment date when computing usage totals

If you are unsure about the task category read the CHANGELOG.md and make a
decision from how they were placed historically in that file.

# Confirm before changing anything
Everything above is read-only. Show the user the full entry as it will be
written — header, categories, and tasks — and wait for explicit confirmation
before editing CHANGELOG.md.

# Next step
This skill only edits CHANGELOG.md — it does not commit or push. Before
running `create-github-release`:

 1. review the diff
 2. commit it on the default branch
 3. push it to origin

`create-github-release` aborts unless the changelog commit is HEAD and
already on origin, so tell the user these steps are required before handing
off to it.
