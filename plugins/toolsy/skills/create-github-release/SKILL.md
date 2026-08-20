---
name: create-github-release
description: >-
  Tag the changelog commit on the default branch, push the tag, and publish
  the matching
  GitHub release using the changelog section as its body. Run right after
  update-changelog. Not for feature branches or individual PRs.
allowed-tools: Bash(git branch:*), Bash(git status:*), Bash(git log:*), Bash(git fetch:*), Bash(git symbolic-ref:*), Bash(git remote:*), Bash(git rev-parse:*), Bash(gh release view:*), Bash(git tag:*), Bash(git push:*), Bash(gh release create:*), Bash(sed:*), Bash(wc:*), Bash(tr:*), Bash(awk:*), Bash(head:*), Write
user-invocable: true
---

# About
This skill finishes the release workflow that `update-changelog` starts. It
tags the changelog commit, pushes the tag, and publishes a GitHub release whose
body is the changelog section for that version.

Run it only after the changelog entry has been committed and pushed to the
default branch.

# Conventions
These are the defaults. Check the most recent tags first (Step 1 lists the
last three) — if the repo already releases under a different scheme, notably
a `v` prefix on tags, match that instead of the defaults below.

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
if [ -z "$BASE" ]; then BASE=main; echo "could not detect default branch, falling back to: $BASE"; fi
echo "default branch: $BASE"
if [ "$BRANCH" != "$BASE" ]; then echo "NOT ON THE DEFAULT BRANCH - stop here."; fi
git fetch --tags --quiet
echo "clean tree: $([ -z "$(git status --porcelain)" ] && echo yes || echo no)"
if git rev-parse --verify --quiet "origin/$BASE" >/dev/null; then
  echo "unpushed commits: $(git log "origin/$BASE..HEAD" --oneline | wc -l | tr -d ' ')"
else
  echo "unpushed commits: UNKNOWN - origin/$BASE not found, fetch or check the remote"
fi
echo "changelog commit: $(git log -1 --format='%h %s' -- CHANGELOG.md)"
echo "HEAD:             $(git log -1 --format='%h %s')"
if [ -f CHANGELOG.md ]; then
  echo "--- newest changelog section ---"
  awk '/^# \[[0-9]+\.[0-9]+\.[0-9]+\]/{if(n++)exit; print; next} n{print}' CHANGELOG.md
else
  echo "CHANGELOG.md: NOT FOUND - stop here"
fi
echo "--- most recent tags ---"
git tag --sort=-v:refname | head -3
```

Some CHANGELOG.md files keep a permanent, unfilled template block at the top
(e.g. `# [6.x.x] - xx-07-2026`) as a fixture for the next entry. The `awk`
expression skips it, if present, because `x` is not a digit, and captures
only the newest real versioned entry. If the file has no such block, the
first entry it finds is already the one you want.

# Step 2 - abort conditions
Stop, report to the user, and change nothing if any of these hold.

 - the branch is not the default branch
 - CHANGELOG.md does not exist, or `origin/$BASE` could not be found
 - the working tree is not clean
 - there are unpushed commits — the tag must point at a commit that is already
   on origin
 - the changelog commit is not HEAD — report which commits landed after it and
   stop
 - a tag or release for this version already exists (checked in Step 3)

Never work around an abort condition. Report it and let the user decide.

# Step 3 - derive the values
Read these from the Step 1 output.

 - version — the number in the header, so `# [6.6.9] - 28-07-2026` gives
   `6.6.9`
 - tag name — apply the Conventions above. By default the tag is bare
   (`6.6.9`). If the "most recent tags" list in Step 1 shows the repo already
   uses a `v` prefix on tags, the tag name is `v6.6.9` instead.
 - release title — `v6.6.9`. If the tag name you just derived already carries
   the `v` (because the repo uses that convention), the title equals the tag
   name exactly — do not add a second `v`.
 - release body — the section with its `# [...]` header line removed, and the
   trailing blank line that separates it from the next entry stripped

Check that the date in the header is today's date. If it is older, the entry
was written on an earlier day; tell the user and confirm the release is still
the intended one before continuing.

Once you have the tag name, confirm neither the tag nor the release already
exists — do not rely on the "most recent tags" list alone, since the version
you need may not be recent.

```bash
git rev-parse --verify --quiet "refs/tags/<tag>" >/dev/null && echo "TAG EXISTS - stop here"
gh release view "<tag>" >/dev/null 2>&1 && echo "RELEASE EXISTS - stop here"
```

# Step 4 - confirm before changing anything
Everything above is read-only. The commands in Step 5 are not — they push a
tag and publish a release, both of which are awkward to undo. Show the user,
and wait for explicit confirmation:

 - the commit to be tagged, as sha and subject
 - the tag name derived in Step 3
 - the release title derived in Step 3
 - the full release body

# Step 5 - execute
After confirmation, write the body to a temporary file and pass it with
`--notes-file`, so the multi-line markdown survives intact. Do not inline it
with `--notes`. Use the exact tag name and title derived in Step 3 — do not
reconstruct them here.

```
git tag <tag> <changelog-commit-sha>
git push origin <tag>
gh release create <tag> --title "<title>" --notes-file <path>
```

# Step 6 - report
Print the release URL that `gh release create` returns.
