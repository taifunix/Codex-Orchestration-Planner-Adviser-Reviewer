# Codex Orchestration - Planner / Advisor / Reviewer

Give Codex bounded specialist roles for planning, implementation, and review while the current Codex task remains in charge.

## What is it?

This fork of [Cjbuilds/Codex-Orchestration](https://github.com/Cjbuilds/Codex-Orchestration) adds a first-class, read-only Reviewer role. It uses Claude Code for subscription-backed Claude routes without placing an Anthropic API key in Codex.

The model selected for the Codex task remains in charge. Codex remains the root orchestrator: it owns integration, verification, approvals, and delivery.

## How it works

```text
YOUR TASK
    |
    v
CODEX COORDINATES THE WORK
    |
    +--> PLANNER CREATES THE FIRST PLAN (optional)
    |         |
    |         v
    |     ADVISOR REVIEWS IT (optional)
    |
    v
EXECUTOR IMPLEMENTS IT
    |
    v
CODEX TESTS & DELIVERS
    |
    v
REVIEWER REVIEWS THE INTEGRATED CHANGE (optional)
```

Advisor returns `PLAN_APPROVED` or `PLAN_REVISE`. Reviewer returns `CODE_REVIEW_PASS` or `CODE_REVIEW_FINDINGS`. Codex decides when delegation or parallel work is useful; the plugin never forces a worker count.

## Why use it?

- Keep planning, implementation, and code review separate without replacing Codex as the decision maker.
- Use Claude Opus for plan critique and Claude Sonnet for a bounded post-implementation review.
- Keep Claude subscription login in the official Claude Code CLI with fail-closed runtime model checks.

## Install and Update

Requirements: Codex with plugin support, Python 3.11+, and, for Claude seats, Claude Code with a compatible first-party login. You do not need to add an Anthropic API key to Codex.

Install this fork once:

```powershell
codex plugin marketplace add taifunix/Codex-Orchestration-Planner-Adviser-Reviewer --ref main
codex plugin add codex-orchestration@codex-orchestration-reviewer
```

Update an existing installation:

```powershell
codex plugin marketplace upgrade codex-orchestration-reviewer
codex plugin add codex-orchestration@codex-orchestration-reviewer
```

Fully restart Codex Desktop and start a new task after installing or updating. The commands above use this fork's marketplace ID and can coexist with the upstream marketplace.

## Recommended Setup

Paste this into a new Codex chat. It is the default recommended orchestration: Sol plans, Opus 5 critiques the plan, Luna implements, and Sonnet 5 reviews the integrated change.

```text
$codex-orchestration:codex-orchestration setup planner: GPT-5.6 Sol High, advisor: Claude Opus 5 High, executor: GPT-5.6 Luna Extra High, reviewer: Claude Sonnet 5 Medium
```

Restart Codex Desktop if setup requests it, then start a new task. The saved policy applies automatically.

## Common Presets

**Fast implementation with review**

```text
$codex-orchestration:codex-orchestration setup executor: GPT-5.6 Luna Extra High, reviewer: Claude Sonnet 5 Medium
```

The current Codex task plans; Luna implements; Sonnet reviews.

**GPT plan review with Claude code review**

```text
$codex-orchestration:codex-orchestration setup planner: GPT-5.6 Sol High, advisor: GPT-5.6 Terra Medium, executor: GPT-5.6 Luna Extra High, reviewer: Claude Sonnet 5 Medium
```

**Claude planning with GPT implementation**

```text
$codex-orchestration:codex-orchestration setup planner: Claude Opus 5 High, executor: GPT-5.6 Luna Extra High, reviewer: Claude Sonnet 5 Medium
```

**Use exact newer Claude model IDs**

Catalog-selected `claude-opus-*` and `claude-sonnet-*` IDs are accepted without a source-file update. The bridge verifies the exact saved ID from runtime metadata and fails closed when it does not match. Use the plugin setup flow to inspect and apply an available exact ID; do not guess unavailable models.

## Manage the Saved Policy

```text
$codex-orchestration:codex-orchestration status
$codex-orchestration:codex-orchestration status --require-effective
$codex-orchestration:codex-orchestration repair
$codex-orchestration:codex-orchestration disable
```

`repair` restores only plugin-owned routing drift. `disable` restores the routing values saved before setup; it does not delete user-owned custom roles. Review and remove any user-owned custom roles separately.

## Role Rules

- Planner is optional. If omitted, the current Codex task model plans.
- Advisor, Designer, and Reviewer are optional. Executor is required for persistent setup.
- Planner and Advisor may use at most one bundled Claude planning seat between them. Reviewer is an independent bundled Claude seat.
- Claude Fable 5 and Claude Opus 5 are available for Planner or Advisor; Claude Sonnet 5 is the standard Reviewer route.
- Advisor has a safety limit of eight reviews. Reviewer has a hard limit of three reviews.
- Reviewer is read-only and root-directed. It never edits files or directs Executor work.
- Models already available through Codex may be used directly. Other unbundled providers must already be configured and authenticated through an existing authenticated, compatible provider.
- External-model discovery never authorizes configuration, credentials, or spend. The plugin never creates credentials or bypasses permissions.

## Development

Run the repository checks from the project root:

```powershell
py -3 scripts/preflight.py quick
py -3 scripts/preflight.py full
git diff --check
```

For the detailed provider and security boundaries, see [providers-and-models.md](plugins/codex-orchestration/skills/codex-orchestration/references/providers-and-models.md), [external-models.md](plugins/codex-orchestration/skills/codex-orchestration/references/external-models.md), and [SECURITY.md](SECURITY.md).

## Attribution and License

This project is based on [Cjbuilds/Codex-Orchestration](https://github.com/Cjbuilds/Codex-Orchestration) and is released under the MIT license. It is an independent open-source project, not an official product of or affiliated with OpenAI, Anthropic, or Cjbuilds.
