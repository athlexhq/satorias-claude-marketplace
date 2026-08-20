---
name: create-github-pr
description: >-
  Title format and description structure required for pull requests in the
  inbound-sync repo. Use when writing a PR there, whether with gh pr create or
  by hand. These conventions are specific to that repo — do not apply them to
  pull requests in any other repo.
user-invocable: true
---

# Scope
These rules describe the inbound-sync repo's conventions, not general PR
practice. If the current repo is not inbound-sync, stop and write the pull
request however that repo does it.

# Title
Each title consists of three parts:
 - action
 - tag
 - task

## Action
When writing a title start with one of the following tags:
 - system
 - fix
 - add
 - remove
 - change
Then use forward slash as a separator ('/').

## Tag
After that use any of the other tags:
 - integration (when work is done specifically for a PMS integration)
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
When writing new PMS integration - 'add/integration/greystar'.
When writing a hotfix - 'fix/patch/strip-email'.


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
