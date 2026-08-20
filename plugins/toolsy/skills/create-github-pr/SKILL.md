---
name: create-github-pr
description: >-
  Open a new pull request for the current branch, with a title and
  description built from its diff against the default branch. Use after
  pushing a branch that has no pull request yet.
allowed-tools: Bash(git branch:*), Bash(git status:*), Bash(git log:*), Bash(git diff:*), Bash(git rev-parse:*), Bash(git symbolic-ref:*), Bash(git remote:*), Bash(gh pr view:*), Bash(gh pr create:*), Bash(sed:*), Bash(wc:*), Bash(tr:*), Read, Write
user-invocable: true
---

# About
This skill creates a pull request from the current branch's diff against the
repo's default branch. To edit the description of a pull request that
already exists, use the `update-github-pr` skill in this plugin instead.

# Step 1 - gather state
Run this block with Bash before doing anything else. Every later step reads
its output.

```bash
BASE=$(git symbolic-ref --quiet --short refs/remotes/origin/HEAD 2>/dev/null | sed 's@^origin/@@')
[ -z "$BASE" ] && BASE=$(git remote show origin 2>/dev/null | sed -n 's/.*HEAD branch: //p')
if [ -z "$BASE" ]; then BASE=main; echo "could not detect default branch, falling back to: $BASE"; fi
echo "default branch: $BASE"
BRANCH=$(git branch --show-current)
echo "branch: $BRANCH"
if [ -z "$BRANCH" ]; then
  echo "DETACHED HEAD - stop here."
elif [ "$BRANCH" = "$BASE" ]; then
  echo "ON THE DEFAULT BRANCH - stop here."
fi
echo "uncommitted changes: $([ -z "$(git status --porcelain)" ] && echo no || echo yes)"
if git rev-parse --abbrev-ref @{u} >/dev/null 2>&1; then
  echo "unpushed commits: $(git log @{u}..HEAD --oneline | wc -l | tr -d ' ')"
else
  echo "unpushed commits: NO UPSTREAM - branch was never pushed"
fi
if git rev-parse --verify --quiet "$BASE" >/dev/null; then
  BASE_REF="$BASE"
else
  BASE_REF="origin/$BASE"
fi
echo "base ref for diff/log: $BASE_REF"
echo "--- existing pull request for this branch ---"
gh pr view --json number,url -q '"#\(.number) \(.url)"' 2>&1
echo "--- commits on this branch ---"
git log "$BASE_REF"..HEAD --oneline --no-merges
echo "--- files changed vs $BASE_REF ---"
git diff "$BASE_REF"...HEAD --stat
```

# Step 2 - abort conditions
Stop, report to the user, and change nothing if any of these hold.

 - the branch is detached HEAD (no current branch) or is the default branch
   reported in Step 1
 - the branch has no upstream — it was never pushed, so there is nothing to
   open a pull request from
 - there are unpushed commits — push them first, otherwise the pull request
   would not reflect the latest code
 - a pull request already exists for this branch — tell the user to use the
   `update-github-pr` skill instead
 - there are uncommitted changes — warn the user, since that work will not be
   on the pull request, and let them decide whether to continue

# Step 3 - work out what the branch does
Base the title and description on the code, not on the commit subjects.
Commit subjects are often branch names and too terse to describe intent on
their own.

 - read the full diff with `git diff <base-ref>...HEAD`
 - for a large branch, work from `git diff <base-ref>...HEAD --stat` first,
   then read the files that carry the real change
 - group the work into semantical tasks, one numbered item per task, in the
   order a reviewer would want to read them

Substitute `<base-ref>` with the base ref reported in Step 1 (`base ref for
diff/log`) — usually the default branch name, but `origin/<default-branch>`
if no local branch for it exists. This is distinct from `<base>` in Step 5,
which is always the plain default-branch name GitHub expects, and from
`<branch>` in Step 5, which is the current/head branch. Each Bash call starts
a fresh shell, so the `$BASE_REF` variable from that block is not available
here — write it out.

Use three dots. `<base-ref>...HEAD` diffs against the merge base, so it shows
only this branch's work rather than everything that landed on the default
branch meanwhile.

# Title
Each title consists of three parts:
 - action
 - tag
 - task

## Action
When writing a title start with one of the following tags:
 - security (vulnerability patches, security-driven dependency upgrades)
 - system
 - fix
 - add
 - remove
 - change
Then use forward slash as a separator ('/').

## Tag
After that use any of the other tags:
 - integration (when work is done for a specific third-party integration)
 - devops (when work is within the DevOps domain)
 - feature
 - patch
 - documentation
Also separate with the forward slash.

## Task
Task is a custom name created from the work performed.
Task can contain five words at most.
Separate each word with the dash ('-').

## Examples
When writing a new integration - 'add/integration/stripe-webhooks'.
When writing a hotfix - 'fix/patch/strip-email'.
When patching a vulnerability - 'security/patch/bump-rack-cve-2026-1234'.


# Description
When writing the PR description use this format.

> ## WHAT was done and WHY?
> 1. I added (something)...
>    - because (something)...
> 
> ## HOW?
> 1. I did it by (something)...

Ensure that each numbered bullet point in the HOW? section matches the bullet
point in the WHAT was done and WHY? section.

This is an example.

> ## WHAT was done and WHY?
> 
> 1. rename the puma callback hooks from `on_worker_boot` to
>    `before_worker_boot` in `config/puma/production.rb` and
>    `config/puma/staging.rb`
>    - because puma v7 renamed all lifecycle hooks to the before_* / after_*
>      naming convention (one of the v7 breaking changes)
> 2. upgrade the puma gem from `6.6.0` to `8.0.2`
>     - because we were two major versions behind and `8.0.2` is the current
>       stable release
> 
> ## HOW?
> 1. edit `config/puma/production.rb` and `config/puma/staging.rb`, replace
>    on_worker_boot do with before_worker_boot do, and update the nearby
>    comments that referenced the old name
> 2. change 'puma', '~> 6.6.0' to gem 'puma', '8.0.2' in the Gemfile (exact
>    pin, not pessimistic)

If the Pull Request is larger in size (10+ files added/edited) add a general
description before the two sections.

# Step 4 - confirm before changing anything
Everything above is read-only. Show the user, and wait for explicit
confirmation:

 - the base and head branches
 - the proposed title
 - the full description

# Step 5 - execute
Write the body to a temporary file and pass it with `--body-file`, so the
multi-line markdown survives intact. Do not inline it with `--body`.

```
gh pr create --base <base> --head <branch> --title "<title>" --body-file <path>
```

# Step 6 - report
Print the pull request URL that `gh pr create` returns.

# Next step
If new commits land on this branch before the pull request is merged, use the
`update-github-pr` skill to refresh the description rather than editing it by
hand.
