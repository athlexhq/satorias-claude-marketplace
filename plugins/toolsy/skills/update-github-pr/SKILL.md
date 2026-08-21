---
name: update-github-pr
description: >-
  Rewrite the description of the pull request open for the current branch, so
  it matches the code that has been written since the PR was created. Use after
  pushing new commits to a branch that already has a PR.
allowed-tools: Bash(git branch:*), Bash(git status:*), Bash(git log:*), Bash(git diff:*), Bash(git rev-parse:*), Bash(git symbolic-ref:*), Bash(git remote:*), Bash(git ls-remote:*), Bash(gh pr view:*), Bash(gh pr edit:*), Bash(sed:*), Bash(wc:*), Bash(tr:*), Read, Write
user-invocable: true
---

# About
The workflow in this repo opens a pull request first and writes the code
afterwards, so the description drifts out of date as commits land. This skill
regenerates that description from the branch's full diff against the repo's
default branch.

It edits an existing pull request. To create one, follow the `create-github-pr`
skill in this plugin.

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
elif git ls-remote --exit-code --heads origin "$BRANCH" >/dev/null 2>&1; then
  echo "unpushed commits: UNKNOWN - branch exists on origin but has no local upstream tracking; compare manually or set it with git push -u"
else
  echo "unpushed commits: NO UPSTREAM - branch was never pushed"
fi
if git rev-parse --verify --quiet "$BASE" >/dev/null; then
  BASE_REF="$BASE"
elif git rev-parse --verify --quiet "origin/$BASE" >/dev/null; then
  BASE_REF="origin/$BASE"
else
  BASE_REF=""
  echo "base ref for diff/log: NOT FOUND - neither $BASE nor origin/$BASE resolves, stop here"
fi
[ -n "$BASE_REF" ] && echo "base ref for diff/log: $BASE_REF"
echo "--- pull request ---"
gh pr view --json number,title,state,isDraft,url \
  -q '"#\(.number) [\(.state)\(if .isDraft then ", draft" else "" end)] \(.title)\n\(.url)"' \
  2>&1
if [ -n "$BASE_REF" ]; then
  echo "--- commits on this branch ---"
  git log "$BASE_REF"..HEAD --oneline --no-merges
  echo "--- files changed vs $BASE_REF ---"
  git diff "$BASE_REF"...HEAD --stat
fi
```

# Step 2 - abort conditions
Stop, report to the user, and change nothing if any of these hold.

 - the branch is detached HEAD (no current branch) or is the default branch
   reported in Step 1
 - neither the local default branch nor `origin/<default-branch>` resolves
   (Step 1 reports "base ref for diff/log: NOT FOUND")
 - no pull request exists for the branch — tell the user to create one first
 - the pull request is closed or merged
 - the branch was never pushed (Step 1 reports "NO UPSTREAM") — nothing on it
   has reached the pull request
 - there are unpushed commits — push them first, otherwise the description
   would describe code that is not on the pull request yet
 - Step 1 reports "UNKNOWN" (branch exists on origin, but has no local
   upstream tracking) — do not guess; ask the user to confirm the branch is
   fully pushed, or have them run `git push -u origin <branch>` so the next
   run can check this itself
 - there are uncommitted changes — warn the user, since that work will not be
   reflected either, and let them decide whether to continue

# Step 3 - read the current description
Fetch the body that is on the pull request now.

```
gh pr view --json body -q .body
```

Read it before writing the replacement. It shows what was already claimed, and
the difference between it and the real diff is the thing being corrected.

# Step 4 - work out what the branch actually does
Base the description on the code, not on the commit subjects. Commit subjects
are often branch names and too terse to describe intent on their own.

 - read the full diff with `git diff <base-ref>...HEAD`
 - for a large branch, work from `git diff <base-ref>...HEAD --stat` first,
   then read the files that carry the real change
 - group the work into semantical tasks, one numbered item per task, in the
   order a reviewer would want to read them

Substitute `<base-ref>` with the base ref reported in Step 1 (`base ref for
diff/log`) — usually the default branch name, but `origin/<default-branch>`
if no local branch for it exists. Do not confuse this with the *current*
branch reported in the same Step 1 block. Each Bash call starts a fresh
shell, so the `$BASE_REF` variable from that block is not available here —
write it out.

Use three dots. `<base-ref>...HEAD` diffs against the merge base, so it shows
only this branch's work rather than everything that landed on the default
branch meanwhile.

# Step 5 - write the new description
Follow the `create-github-pr` skill for the structure. Every numbered item in
HOW? must correspond to the same numbered item in WHAT was done and WHY?.

Rebuild the whole description from the diff rather than patching the old text.
Anything in the old body that is no longer true must not survive.

If the old body carried sections that are not part of the format, such as
testing notes, screenshots, or links to tickets, keep them and place them after
the two required sections.

# Step 6 - check the title
Compare the current title against the `create-github-pr` rules and against what
the branch now does.

Leave the title alone if it still describes the work. Propose a new one only
when the scope has genuinely moved, and let the user decide before changing it.

# Step 7 - confirm before changing anything
Everything above is read-only. Show the user, and wait for explicit
confirmation:

 - the pull request number and URL
 - the new description in full
 - the proposed title, if a change is being suggested

Say plainly that the current description will be replaced, not merged.

# Step 8 - execute
Write the body to a temporary file and pass it with `--body-file`, so the
multi-line markdown survives intact. Do not inline it with `--body`.

```
gh pr edit <number> --body-file <path>
gh pr edit <number> --title "<new title>"
```

Run the second command only if the user approved a title change.

# Step 9 - report
Print the pull request URL.

# Next step
Once the pull request is reviewed and merged into the default branch — outside
the scope of this skill — continue the release workflow with the
`update-changelog` skill.
