---
name: create-github-release
description: >-
  Tag the changelog commit on the default branch, push the tag, and publish
  the matching
  GitHub release using the changelog section as its body. Run right after
  update-changelog. Not for feature branches or individual PRs.
allowed-tools: Bash(git branch:*), Bash(git status:*), Bash(git log:*), Bash(git fetch:*), Bash(git symbolic-ref:*), Bash(git remote:*), Bash(gh release view:*), Bash(git tag:*), Bash(git push:*), Bash(gh release create:*), Read, Write
user-invocable: true
---

# About
This skill finishes the release workflow that `update-changelog` starts. It
tags the changelog commit, pushes the tag, and publishes a GitHub release whose
body is the changelog section for that version.

Run it only after the changelog entry has been committed and pushed to the
default branch.

# Conventions
These are fixed by the repository's release history. Do not deviate from them.

 - the git tag is bare semver with no `v` prefix — `6.6.9`, not `v6.6.9`
 - the tag is lightweight — `git tag 6.6.9`, never `git tag -a`
 - the release title does carry the prefix — `v6.6.9`
 - the release body is the changelog section without its `# [x.y.z] - date`
   header, so it starts at the first `## Category` line
 - the release is neither a draft nor a prerelease

# Step 1 - gather state
Run this block with Bash before doing anything else. Every later step reads
its output.

```bash
BRANCH=$(git branch --show-current)
echo "branch: $BRANCH"
BASE=$(git symbolic-ref --quiet --short refs/remotes/origin/HEAD 2>/dev/null | sed 's@^origin/@@')
[ -z "$BASE" ] && BASE=$(git remote show origin 2>/dev/null | sed -n 's/.*HEAD branch: //p')
[ -z "$BASE" ] && BASE=master
echo "default branch: $BASE"
if [ "$BRANCH" != "$BASE" ]; then echo "NOT ON THE DEFAULT BRANCH - stop here."; fi
git fetch --tags --quiet
echo "clean tree: $([ -z "$(git status --porcelain)" ] && echo yes || echo no)"
echo "unpushed commits: $(git log "origin/$BASE..HEAD" --oneline | wc -l | tr -d ' ')"
echo "changelog commit: $(git log -1 --format='%h %s' -- CHANGELOG.md)"
echo "HEAD:             $(git log -1 --format='%h %s')"
echo "--- newest changelog section ---"
awk '/^# \[[0-9]+\.[0-9]+\.[0-9]+\]/{if(n++)exit; print; next} n{print}' CHANGELOG.md
echo "--- most recent tags ---"
git tag --sort=-v:refname | head -3
```

The `awk` expression skips the permanent `# [6.x.x] - xx-07-2026` template
block, because `x` is not a digit, and captures only the newest real versioned
entry.

# Step 2 - abort conditions
Stop, report to the user, and change nothing if any of these hold.

 - the branch is not the default branch
 - the working tree is not clean
 - there are unpushed commits — the tag must point at a commit that is already
   on origin
 - the changelog commit is not HEAD — report which commits landed after it and
   stop
 - a tag for this version already exists

Never work around an abort condition. Report it and let the user decide.

# Step 3 - derive the values
Read these from the Step 1 output.

 - version — the number in the header, so `# [6.6.9] - 28-07-2026` gives
   `6.6.9`
 - tag name — the version, unprefixed
 - release title — the version with a `v` in front
 - release body — the section with its `# [...]` header line removed, and the
   trailing blank line that separates it from the next entry stripped

Check that the date in the header is today's date. If it is older, the entry
was written on an earlier day; tell the user and confirm the release is still
the intended one before continuing.

# Step 4 - confirm before changing anything
Show the user, and wait for explicit confirmation:

 - the commit to be tagged, as sha and subject
 - the tag name
 - the release title
 - the full release body

Everything up to this point is read-only. The commands below are not — they
push a tag and publish a release, both of which are awkward to undo.

# Step 5 - execute
After confirmation, write the body to a temporary file and pass it with
`--notes-file`, so the multi-line markdown survives intact. Do not inline it
with `--notes`.

```
git tag <version> <changelog-commit-sha>
git push origin <version>
gh release create <version> --title "v<version>" --notes-file <path>
```

# Step 6 - report
Print the release URL that `gh release create` returns.
