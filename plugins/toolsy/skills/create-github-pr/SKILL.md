---
name: create-github-pr
description: >-
  Title format and description structure for pull requests. Use when writing
  a PR, whether with gh pr create or by hand.
user-invocable: true
---

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
 - security (vulnerability patches, security-driven dependency upgrades)
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
