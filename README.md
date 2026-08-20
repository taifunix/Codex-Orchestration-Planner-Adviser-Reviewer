# Codex Orchestration — Planner / Advisor / Reviewer

Bring specialized models into a Codex workflow, give each model a bounded role, and keep the current Codex task model in charge from planning through implementation review.

This repository is a derivative of [Cjbuilds/Codex-Orchestration](https://github.com/Cjbuilds/Codex-Orchestration). It keeps the upstream Planner, Advisor, Designer, Executor, External Model, routing, repair, and safety machinery, and adds a first-class **Reviewer** route backed by **Claude Sonnet 5** through the existing hardened Claude Code MCP bridge.

## What this fork adds

The main addition is a post-implementation Reviewer stage:

- **Reviewer** runs only after Executor integration and required verification.
- The bundled Reviewer route uses **Claude Sonnet 5** through the official Claude Code CLI and a compatible first-party Claude login.
- The persisted Reviewer default is **Claude Sonnet 5 / Medium**.
- `review_code` returns one of two structured decisions:
  - `CODE_REVIEW_PASS`
  - `CODE_REVIEW_FINDINGS`
- Findings are adjudicated by the root Codex model before any fix is accepted.
- Accepted findings go back to Executor, verification is rerun, and Reviewer may review fresh results again.
- The workflow allows **at most three total Reviewer reviews**.
- A third material `CODE_REVIEW_FINDINGS` result halts with a non-approval artifact. There is no silent fourth review and no false "done".
- A single `review_code` call may supply task-local `model` and `effort` overrides. They must be bridge-qualified/supported and **must not persist task-local Reviewer overrides** into routing state.
- Omitted task-local override fields fall back to the persisted Reviewer route.

Catalog-selected `claude-opus-*` and `claude-sonnet-*` model IDs are accepted by family without a source-file update; the bridge still verifies the exact persisted ID and observed runtime metadata fail closed.

## Roles

Codex Orchestration gives a task several explicit roles:

- **Planner** creates and revises the plan. It is optional; when omitted, the current Codex model plans.
- **Advisor** reviews the plan and either approves it or returns actionable planning findings. It is optional.
- **Designer** creates a bounded design handoff when the task needs one. It is optional.
- **Executor** implements the approved work. It is required for persistent setup.
- **Reviewer** performs read-only implementation review after integration and verification. It is optional.

The model selected for the Codex task remains the **root orchestrator and final authority**. Planner, Advisor, Designer, Executor, and Reviewer do not take ownership of the task away from Codex.

## Workflow

```text
                           YOUR TASK
                               |
                               v
                    CODEX COORDINATES WORK
                               |
                               v
                     PLANNER CREATES PLAN
                      optional / root may plan
                               |
                               v
                       ADVISOR REVIEWS
                               |
                   PLAN_REVISE? ---- yes ----+
                               |              |
                              no              v
                               |       PLANNER REVISES
                               |              |
                               +<-------------+
                               |
                         PLAN APPROVED
                               |
                               v
                       DESIGNER HANDOFF
                            optional
                               |
                               v
                     EXECUTOR IMPLEMENTS
                               |
                               v
                      TESTS / VERIFICATION
                               |
                               v
                     REVIEWER REVIEWS CODE
                       Claude Sonnet 5
                               |
                 +-------------+-------------+
                 |                           |
          CODE_REVIEW_PASS           CODE_REVIEW_FINDINGS
                 |                           |
                 v                           v
              DELIVER             ROOT ADJUDICATES FINDINGS
                                             |
                                             v
                                      EXECUTOR FIXES
                                             |
                                             v
                                      TESTS RERUN
                                             |
                                             v
                                    FINAL REVIEWER PASS
                                             |
                          +------------------+------------------+
                          |                                     |
                   CODE_REVIEW_PASS                    MATERIAL FINDINGS
                          |                                     |
                          v                                     v
                       DELIVER                         HALT NON-APPROVED
```

Planner and Advisor may iterate until the plan is ready. Codex stops as soon as Advisor returns `PLAN_APPROVED`, with a **safety limit of eight reviews**. If plan approval is not reached within that bound, execution stops and Codex reports the latest plan and unresolved findings.

Reviewer uses a separate, tighter bound: **at most three total Reviewer reviews** after implementation. Reviewer failure or unavailability is never interpreted as approval.

## Why use it?

- Keep one Codex model in charge while delegating bounded specialist work.
- Use different models for planning, plan review, design, implementation, and code review.
- Bring **Claude Fable 5**, Claude Opus 5, and Claude Sonnet 5 into supported subscription-backed roles without putting Anthropic API keys into Codex.
- Get independent plan review before code changes begin.
- Add an independent implementation review gate after tests.
- Keep review findings structured and root-adjudicated rather than allowing one model to rewrite another model's work directly.
- Preserve fail-closed routing and exact runtime model identity checks.
- Run suitable Executor work in parallel while keeping integration and verification under Codex control.

Results depend on the models, task, context, available parallel work, retries, and local tooling. Model use is not a guarantee of correctness; tests and root adjudication remain part of the workflow.

## Requirements

- Codex with plugin support.
- Python 3.11 or newer.
- For bundled Claude subscription routes: the official Claude Code CLI and a compatible first-party Claude login.
- For Claude Opus 5, retain the upstream Claude Code minimum documented by the project.
- For External Model providers such as OpenRouter, follow the provider-specific secure enrollment and qualification flow documented under `references/`.

The bundled Claude routes use first-party Claude authentication through the official CLI. They do **not** require you to paste an Anthropic API key into Codex.

## Install this fork

Add this repository as a Codex plugin marketplace and install the existing plugin package:

```bash
codex plugin marketplace add taifunix/Codex-Orchestration-Planner-Adviser-Reviewer --ref main
codex plugin add codex-orchestration@codex-orchestration-reviewer
```

The fork uses the marketplace ID `codex-orchestration-reviewer`, so it can coexist with the upstream `codex-orchestration` marketplace without a source-name collision.

Then fully restart Codex Desktop and start a new task.

Setup prompts use the literal skill label:

```text
$codex-orchestration:codex-orchestration
```

These are prompts for Codex chat, not shell commands.

## Quick start

A simple setup with the new Reviewer route:

```text
$codex-orchestration:codex-orchestration setup reviewer: Claude Sonnet 5 Medium, executor: GPT-5.6 Luna Extra High
```

A fuller native workflow with GPT planning/review and Claude Sonnet implementation review:

```text
$codex-orchestration:codex-orchestration setup planner: GPT-5.6 Sol High, advisor: GPT-5.6 Terra Medium, executor: GPT-5.6 Luna High, reviewer: Claude Sonnet 5 Medium
```

After setup completes, fully restart Codex Desktop if instructed, start a new task, and use Codex normally. The saved routing policy applies automatically.

Check the installed state with:

```text
$codex-orchestration:codex-orchestration status
```

Repair only plugin-owned routing drift with:

```text
$codex-orchestration:codex-orchestration repair
```

Disable the saved routing policy with:

```text
$codex-orchestration:codex-orchestration disable
```

## Choose your roles

Persistent setup accepts explicit role labels:

```text
$codex-orchestration:codex-orchestration setup planner: <model and effort>, advisor: <model and effort>, designer: <model and effort>, executor: <model and effort>, reviewer: <model and effort>
```

Role labels are literal:

- `planner:` configures only Planner.
- `advisor:` configures only Advisor.
- `designer:` configures only Designer.
- `executor:` configures only Executor.
- `reviewer:` configures only Reviewer.

Omitting a role does not silently move another model into that seat.

Important defaults:

- Omitted Planner means the current Codex root model plans.
- Omitted Advisor means no separate plan review.
- Omitted Designer means no separate design handoff.
- Executor is required for persistent setup.
- Omitted Reviewer means no separate post-implementation Reviewer.
- The persisted Claude Sonnet Reviewer default effort is `medium`.

## Bundled Claude subscription routes

The project uses a sealed bridge for audited Claude subscription routes.

Current bundled role families include:

- **Claude Fable 5** — Planner or Advisor.
- **Claude Opus 5** — Planner or Advisor.
- **Claude Sonnet 5** — Reviewer.

The current Sonnet Reviewer model identity is qualified as `claude-sonnet-5`.

Planner and Advisor may configure at most one bundled Claude planning seat between them. Reviewer is an independent bundled Claude seat, so Opus Advisor plus Sonnet/Opus Reviewer is valid; all bundled seats must use the same managed launcher.

The bridge:

- uses the official Claude Code CLI;
- relies on the user's compatible first-party Claude login;
- does not extract or persist Claude credentials;
- uses a minimal process environment;
- disables tools for sealed review calls;
- disables session persistence;
- mechanically validates runtime model metadata;
- fails closed on an unqualified or unexpected primary model identity.

Accounting metadata emitted by a first-party CLI is not treated as proof that an API-billed route was used.

## Reviewer behavior

Reviewer is a root-directed, read-only implementation review role.

The normal sequence is:

1. Executor completes the implementation.
2. Required tests/verification run.
3. Root builds a self-contained implementation review packet.
4. Root calls `review_code`.
5. Reviewer returns `CODE_REVIEW_PASS` or `CODE_REVIEW_FINDINGS`.
6. Root validates and adjudicates any findings.
7. Accepted findings go to Executor.
8. Executor fixes the accepted issues and reruns verification.
9. Root may send fresh packets for the second and third Reviewer passes.
10. A third material findings result halts without approval.

Reviewer does not contact Executor directly and does not modify implementation files itself.

## Task-local Reviewer overrides

The persisted route is a default, not a per-task invariant.

For a specific review, `review_code` may receive optional task-local:

```text
model
effort
```

Rules:

- omitted fields use the persisted Reviewer route;
- the model must be a bridge-qualified Reviewer model;
- the effort must be supported by the qualified Sonnet route;
- the override applies only to that call;
- task-local overrides do not mutate saved routing state;
- arbitrary model strings outside the `claude-opus-*` and `claude-sonnet-*` catalog families are rejected.

The default model is Claude Sonnet 5. Future catalog-selected Sonnet or Opus releases can be selected without changing the interface; exact runtime identity checks remain required.

## Advisor behavior

Advisor is a planning gate, not an implementation reviewer.

It returns structured plan decisions such as:

- `PLAN_APPROVED`
- `PLAN_REVISE`

Planner and Advisor may iterate through a bounded approval loop. Advisor reports to the root Codex model; it does not contact Executors directly. Failure to call Advisor or obtain a valid response never counts as approval.

## External Model roles

External Models remain roles rather than Desktop picker entries.

Codex stays signed in with ChatGPT, the selected GPT model remains root, and validated external providers use provider-pinned bounded roles. The plugin does not replace top-level root model/provider state merely to invoke an External Model.

For external providers, setup is deliberately staged. Authentication is performed through the documented hidden local enrollment flow rather than by pasting secrets into chat.

A read-only availability question performs status inspection only; it **never authorizes configuration, credentials, or spend**.

Never paste API keys, bearer tokens, private keys, or other credentials into Codex chat, repository files, provider manifests, tests, or issue reports.

See:

- `plugins/codex-orchestration/skills/codex-orchestration/references/external-models.md`
- `plugins/codex-orchestration/skills/codex-orchestration/references/providers-and-models.md`

## Codex Goals

You can use the saved workflow with ordinary Codex Goals.

Codex still owns:

- Goal lifecycle and state;
- permissions and approvals;
- task integration;
- verification;
- final delivery.

The orchestration plugin guides role routing; it does not silently create, pause, resume, clear, or replace a Goal.

## Useful prompts

```text
$codex-orchestration:codex-orchestration status
$codex-orchestration:codex-orchestration status --require-effective
$codex-orchestration:codex-orchestration repair
$codex-orchestration:codex-orchestration disable
$codex-orchestration:codex-orchestration --update
```

Reviewer setup:

```text
$codex-orchestration:codex-orchestration setup reviewer: Claude Sonnet 5 Medium, executor: GPT-5.6 Luna Extra High
```

Planner + Advisor + Executor + Reviewer:

```text
$codex-orchestration:codex-orchestration setup planner: GPT-5.6 Sol High, advisor: GPT-5.6 Terra Medium, executor: GPT-5.6 Luna High, reviewer: Claude Sonnet 5 Medium
```

## Important limits

- Codex remains the root orchestrator and final authority.
- Planner, Advisor, Designer, and Reviewer are bounded specialist roles.
- Advisor approval is a planning gate, not a guarantee that implementation will succeed.
- Reviewer approval is an implementation review result, not a replacement for tests or root verification.
- Reviewer is read-only and root-directed.
- Reviewer has a hard three-review bound.
- Task-local Reviewer overrides do not persist.
- Direct model routes remain constrained by provider/routing policy.
- External provider setup does not bypass authentication, billing consent, permissions, or approvals.
- If you explicitly say `no subagents`, Codex must not delegate through ordinary native subagents.
- The project fails closed when bundled external runtime identity cannot be established mechanically.

## Updating this fork

To update the installed plugin from this repository, use the plugin's native update flow when the installed source is recognized:

```text
$codex-orchestration:codex-orchestration --update
```

For development, this repository also keeps the original project as `upstream`, so upstream changes can be reviewed and integrated deliberately rather than overwriting fork-specific Reviewer behavior.

When integrating upstream changes, rerun the relevant routing, bridge, subscription, native-policy, and skill-contract tests before publishing.

## Development

Install development dependencies:

```bash
python3 -m pip install -r requirements-dev.txt
```

Core checks:

```bash
python3 -m compileall -q plugins tests scripts
python3 -m ruff check plugins tests scripts
```

Reviewer-related targeted suites:

```bash
python3 -m unittest discover -s tests -p test_external_providers.py
python3 -m unittest discover -s tests -p test_external_subscription.py
python3 -m unittest discover -s tests -p test_routing_state.py
python3 -m unittest discover -s tests -p test_fable_advisor_mcp.py
python3 -m unittest discover -s tests -p test_native_routing.py
python3 -m unittest discover -s tests -p test_skill_contract.py
```

The upstream full suite may exercise platform-specific filesystem behavior. Treat targeted Reviewer gates and any known baseline platform failures separately; do not convert pre-existing unrelated failures into false Reviewer regressions.

Before committing:

```bash
git diff --check
git status --short
```

## Security

Do not commit:

- API keys or bearer tokens;
- Claude authentication material;
- local Codex routing state;
- `.codex` user state;
- credential-store exports;
- private keys;
- local debug dumps containing secrets.

The bundled Claude bridge is designed to use first-party CLI authentication without copying credentials into project state.

For vulnerability reporting and the inherited security model, see `SECURITY.md`.

## Attribution

This repository is based on **[Cjbuilds/Codex-Orchestration](https://github.com/Cjbuilds/Codex-Orchestration)** and preserves the upstream MIT-licensed work.

The Claude Sonnet Reviewer workflow in this fork extends the upstream orchestration model with a bounded post-implementation review stage while retaining Codex as root orchestrator.

Upstream project:

```text
https://github.com/Cjbuilds/Codex-Orchestration
```

This fork:

```text
https://github.com/taifunix/Codex-Orchestration-Planner-Adviser-Reviewer
```

## Disclaimer

This is an independent open-source project.

It is **not an official product of, endorsed by, or affiliated with OpenAI, Anthropic, or Cjbuilds**.

"Codex", "OpenAI", "Claude", "Anthropic", and other product or company names are used only to describe interoperability with their respective tools and services. Their trademarks remain the property of their respective owners.

Users are responsible for complying with the terms, account rules, usage limits, and billing rules of any service they connect.

## License

MIT. See `LICENSE`.

Copyright notices and the MIT license text from the project must remain with copies or substantial portions of the software.
