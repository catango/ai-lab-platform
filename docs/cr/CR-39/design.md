# CR-39 Design: software template (ADR-008)

| Field | Value |
|---|---|
| CR | issue #39 (Class 3) |
| Governing records | Baseline v0.1 (sections 3, 6, 7, 8), ADR-001 (decisions 6 to 8, section 5), ADR-003 (sections 3 to 8), ADR-005 (decisions 1 and 3 to 7), ADR-011 (decision 17) |
| Record produced | [ADR-008](../../adr/ADR-008-software-template.md), status Proposed. Amended in this PR: [ADR-003](../../adr/ADR-003-pr-centric-workflow.md) (Q8) and [ADR-005](../../adr/ADR-005-toolchain.md) (Q7), each with an "Amended by" row |
| Scope | Design only. Nothing is implemented, and the `ai-software-template` repository is not touched |
| State | The approver's decisions Q1 to Q14 of the architecture discussion (review on PR #40, 2026-11-19) are applied in this file, in ADR-008, and in the other CR-39 files. Section 13 records them. Section 14 lists what stays open. Amended after the merge of PR #40 (design PR `design/CR-39-provenance-location`): the provenance of the seeded generic code is recorded in a template-only file, not in the payload manifest (sections 3, 4, and 14). Amended again (design PR `design/CR-39-tc-amendments`, the approver's decisions of 2026-11-27, Q15 and Q16): a template-sync PR may also add a new `seeded` path, and an init PR may append the preset's entries to `permissions.deny` of `.claude/settings.json` (sections 3, 4, 5, 13, and 14). Amended a third time (design PR `design/CR-39-seeded-files`, the approver's decisions of 2026-11-28, Q17 to Q20): the required status checks are the aggregate jobs, the template ships three seeded files (`dependabot.yml`, `CODEOWNERS`, `SECURITY.md`) with the checks `dependabot-config` and `codeowners`, the seeded `dependabot.yml` has no `pip` entry, and `owned-drift` allows a `uses:`-only change to the same action (sections 3, 4, 8, 11, 13, and 14) |

## 1. Scope and boundaries

In scope (ADR-008): template structure, `CLAUDE.md`, rules, Skills including `address-review-feedback`, the CR and ADR templates, template-sync and the init PR, the template's own quality gate and CI (and how it relates to ADR-009), what is specific to the ARM64 lab VM and what is not, the public-repository rules, the generic operator contract, and the Stage 5 exit.

Out of scope: ADR-009 (security matrix, Actions hardening, which checks are required) and ADR-010 (token optimisation, metrics). Section 11 lists each dependency and the assumption. Nothing in section 11 is decided here. Also out of scope: this platform's preset (section 3), which is a change PR in `ai-lab-infrastructure` outside CR-39.

**Decoupling requirement (Q14, approver).** The template must be usable with a different lab infrastructure. No payload file and no generated file names `ai-lab-infrastructure`, its ADR or CR numbers, OCI, host names, or a fixed architecture. CI needs only the template and the project repository. Every platform value comes from one configuration through `initialize-project --config` (section 3). This file and ADR-008 live in this repository and name it; the template does not.

## 2. Decisions in this design

| ID | Decision | Rationale |
|---|---|---|
| ST-1 | The repository root is the payload. Every file has one class in `.template/manifest.toml`: `owned`, `seeded`, or `template-only`. Template-only files (`template-dev/`, `.github/workflows/template-ci.yml`) are removed by `initialize-project`. The template keeps no CR or ADR instance under `docs/`; its records live in `template-dev/docs/` | The payload's workflows run for real on every template PR, and template-only content never reaches a project (Q1) |
| ST-2 | `initialize-project` takes every value from `--config` and never inherits the template's own configuration file or `README.md` | A generated repository must not carry template identity (Q1, Q14) |
| ST-3 | Template-sync PR `template-sync/v<MAJOR>.<MINOR>.<PATCH>`: tags only, approver-only tags enforced by a tag ruleset, the repository's own reviewed copy of `scripts/template-sync` does every check and copy, downgrades and moved tags refused, no schedule, event, or workflow | Approver-only tags were a convention; running the fetched script would bypass review in the application repository (Q2) |
| ST-4 | An owned file that was changed locally is overwritten by default. Divergences that must survive are declared in the seeded `.template/overrides.toml`. Gate check `owned-drift` fails a PR that changes an owned file without an entry | A skipped file shows no diff and leaves `state.toml` undefined; Dependabot pins would make conflicts routine (Q3) |
| ST-5 | `CLAUDE.md` is fully owned and platform-neutral. It ends with two imports, `docs/PLATFORM.md` then `docs/PROJECT.md`, both seeded. The zoned class does not exist | A whole-file hash is simpler and stronger than markers; a failed import loses project context, not guardrails (Q4) |
| ST-6 | Guardrails: owned core deny list `.claude/required-deny.txt`, seeded platform list `.claude/deny.platform.txt`, gate checks `deny-list` and `forbidden-api` | Deny rules match only Claude's own commands; script subprocesses need a static check (Q5) |
| ST-7 | Skills are thin; each mechanical step is a script with an offline mode. Added Skills `sync-template` and `start-cr`, with constraints | Instruction discipline for flows that would otherwise be done by hand (Q6) |
| ST-8 | The template owns its generic code (`pr_contract`, gate checks, install engine) and its own CI manifest. Generic code is identical across repositories; differences come only from data | No live link between repositories (Q7, Q14) |
| ST-9 | CI is a matrix over the configured architectures and runner labels. Where no runner exists, the job summary says "not run: <arch>" and the PR template requires a host-reported evidence line with the commit SHA | The silent skip removed the only automated ARM64 evidence (Q9) |
| ST-10 | The template supports public and private repositories. The condition reads `github.event.repository.private`, not the configuration file | Q9 |
| ST-11 | Platform-neutral `operator.md`; PC-8 follows `operator_paths`; gate check `operator-contract`; guard library `scripts/lib/operator_guard.py`; no operator scripts | A prose-only contract is weaker than the platform's enforced one (Q10) |
| ST-12 | Stage 5 exit needs both the scripted dry run in template CI (two configurations, both architectures) and one real run by the approver in a throwaway public repository | `git archive` and fixtures do not prove that a generated repository works under ADR-003 (Q11) |
| ST-13 | The template is decoupled from this platform and configurable; this platform's preset lives in `ai-lab-infrastructure` | Approver requirement (Q14) |
| ST-14 | Per-repository numbering. Cross-repository references are always qualified. Owned text cites no platform record | The same numbers exist in several repositories (Q12) |
| ST-15 | New PR type `init/v<MAJOR>.<MINOR>.<PATCH>` for the first change after generation, with rule TC-2 | ADR-003 decision 1 requires a PR for every change; no type covered the first one (Q13) |

## 3. What is generated and what stays in the template

Class: O = owned (template-owned, synced), S = seeded (written once by `initialize-project`, then project-owned, never synced except as new seeded files, section 4), T = template-only (removed by `initialize-project`).

**One configuration.** `initialize-project --config <file>` is the only input. Keys (names are proposals; the schema is fixed in the implementation PR): identities (`approver_login`, `approver_id`, `machine_login`, owner), the template source repository, `design_repos`, `architectures` and runner labels, `operator_paths`, `visibility`, `post_init_actions`, and the paths of the seeded `docs/PLATFORM.md` and `.claude/deny.platform.txt` supplied by the preset. A **preset** is such a configuration plus those two files. This platform's preset is kept in `ai-lab-infrastructure` and is a change PR outside CR-39 (section 14).

| Path | Class | Content |
|---|---|---|
| `CLAUDE.md` | O | Section 5. Platform-neutral. Ends with the imports of `docs/PLATFORM.md` and `docs/PROJECT.md` |
| `docs/PLATFORM.md` | S | From the preset: where the project runs, the platform's records and constraints, platform-specific Never rules. Contains no import |
| `docs/PROJECT.md` | S | The project's own conventions. Contains no import |
| `README.md`, `docs/README.md` | S | Placeholder readme and documentation index, written from `--config` |
| `config/project.yaml` | S | Written from `--config`. Never copied from the template |
| `.template/manifest.toml` | O | File classes and the template version. It holds no provenance and names no other repository |
| `.template/state.toml` | S, then written by sync and init only | Template version and tag commit last synced, SHA-256 of each owned file as synced, entries recorded as `override` |
| `.template/overrides.toml` | S | Declared divergences from owned files: path, reason, PR reference. Written by the project; never by sync |
| `.template/budgets.toml` | O | Size budgets (values from ADR-010, section 11) |
| `.claude/settings.json` | O | Guardrails (section 5). A project addition is a divergence and needs an override entry. Its `permissions.deny` is the core list plus the seeded `deny.platform.txt`: `initialize-project` appends the preset's entries (TC-2), and a sync keeps them (section 4, step 3) |
| `.claude/required-deny.txt` | O | Generic core deny list |
| `.claude/deny.platform.txt` | S | From the preset |
| `.claude/rules/*.md`, `.claude/skills/<name>/SKILL.md` | O | Sections 5 and 6 |
| `.github/ISSUE_TEMPLATE/cr.md` | O | CR intake |
| `.github/PULL_REQUEST_TEMPLATE/{design,implementation,change,template-sync,init}.md` | O | The change PR template carries the host-reported evidence line for architectures with no runner (Q9) |
| `.github/workflows/{ci,pr-contract}.yml` | O | Section 8 |
| `.github/dependabot.yml` | S | Version updates: one entry, `github-actions` on `/`. It has no `pip` entry for `toolchain/python`: that directory and the checksums of `toolchain/ci-manifest.toml` are owned gate tooling, changed together by template releases (Q20). The project adds its own ecosystems. Gate check `dependabot-config` (section 8) |
| `.github/CODEOWNERS` | S | The one line `* @<approver_login>`, written by `initialize-project` from `github.approver_login` of the configuration; a sync that adds it writes it from the project's own configuration, never from the tag's file (section 4). Gate check `codeowners` (section 8) |
| `SECURITY.md` | S | Generic text that relies on GitHub private vulnerability reporting, which `initialize-project` lists among the post-init actions it prints (section 4) |
| `scripts/quality-gate.sh`, `scripts/checks/`, `scripts/lib/` | O | Section 8; `scripts/lib/operator_guard.py` is among them |
| `scripts/checks.d/` | S | Project checks, same interface as `scripts/checks/` |
| `scripts/pr-contract`, `scripts/pr_contract/`, `scripts/pr-body` | O | Data-driven by `config/project.yaml` (section 4 and ADR-003 section 6) |
| `scripts/pr-checkpoint`, `scripts/review-threads`, `scripts/cr-context`, `scripts/cr-criteria`, `scripts/new-cr`, `scripts/initialize-project`, `scripts/template-sync` | O | Section 6 |
| `toolchain/` (engine and the template's CI manifest) | O | Gate tools, exact versions, both architectures, checksums (section 8) |
| `docs/adr/_template.md`, `docs/cr/_template/` | O | Section 7 |
| `docs/adr/`, `docs/cr/` instances | S | Created by the project; never touched by sync. The template has none |
| `template-dev/**` and `.github/workflows/template-ci.yml` | T | The generation test and its fixtures, the sync test, the dry-run script, release checklist, `template-dev/provenance.toml` (the repository and commit the generic code was seeded from, history only; the one file that may name another repository), and the template's own records under `template-dev/docs/` (CR-39 stays in `ai-lab-infrastructure` as the one transitional exception) |

**Never in the payload:** operator scripts or an operator profile, anything that names this platform, OCI, a host, a CR or ADR number of another repository, a fixed architecture, an identifier, or a credential. Gate check `leak` and criterion T-21 scan for it.

## 4. Template-sync and the init PR

**Template-sync** (`scripts/template-sync --to v<MAJOR>.<MINOR>.<PATCH> [--dry-run]`, run by Claude Code in a session the user started, as the machine account):

1. Preconditions: clean tree, new branch `template-sync/<tag>` from the default branch, the tag exists in the template source repository, the tag's commit is contained in its default branch, the tag is newer than the version in `.template/state.toml` (a **downgrade is refused**), and a tag whose commit differs from the commit recorded in `state.toml` for that version (a **moved tag**) is refused. Tags are created by the approver only, enforced by a **tag ruleset** on the template repository (user-only, before the first tag).
2. Clone the template at the tag into a temporary directory, with **credential helpers disabled**. The script that runs is **the repository's own reviewed copy** of `scripts/template-sync`, never the fetched one. A changed sync script therefore takes effect on the next sync; a release that needs the new script first is a MAJOR release with a "sync twice" note.
3. Compare `.template/state.toml` with the tag's manifest and files:
   - an `owned` file is **overwritten**, added, or removed, **including a locally changed one**, unless `.template/overrides.toml` lists it; listed entries are honoured, listed in the PR body, and recorded as `override` in `state.toml`;
   - drift limited to `uses:` lines in owned workflows (Dependabot pin bumps) is overwritten and listed;
   - a `seeded` file is never changed or removed. A seeded path of the tag that the project lacks is added once, and listed in the PR body (section "Changed files by class"), only when it is new to the project: the project's manifest has no seeded pattern for the path, or lists it by its exact name and no commit of the checked-out history (`git log HEAD`) ever changed it. Other branches and fetched remotes do not count. A file the project deleted is never added back. In a shallow clone the history cannot tell a deleted file from one that never existed, so nothing is added and the plan says so;
   - `.github/CODEOWNERS` is written as `* @<approver_login>` from the project's own `config/project.yaml`, never from the tag's file (which names the template's approver); when the key is missing or not a valid account name, the file is not added and the plan says why;
   - the owned `.claude/settings.json` is overwritten with the tag's version, and every entry of the project's seeded `.claude/deny.platform.txt` that the tag's list lacks is appended to `permissions.deny`, so the file is a function of the core list plus the seeded list;
   - `.template/overrides.toml` is never changed by sync.
4. Update `.template/state.toml` (version, tag commit, hashes), commit, push, open a **draft** PR with `gh pr create --draft --body-file` from `scripts/pr-body template-sync`. The body lists the versions, the changed files by class, the overrides honoured, the `uses:`-only drift, a link to the release notes, and, when `.github/workflows/` changed, the section "Workflow change justification" (PC-6).
5. The approver reviews and merges like any PR.

`--dry-run` prints the plan and writes nothing. There is no schedule, event trigger, or workflow that runs the script. The lag between the repository's version and the newest tag is reported read-only by `resume-work` and `prepare-cr-context` (`template-sync --check`) and by the template's release checklist; nothing more.

**First sync of a project whose `scripts/template-sync` predates the seeded-file rule (Q19).** The script that runs is the project's own copy (step 2), and the older copy skips every path that the project's manifest already classes seeded. The first sync therefore brings the new script and the new checks `dependabot-config` and `codeowners` (both owned) but not `.github/dependabot.yml`, `.github/CODEOWNERS`, and `SECURITY.md`, and both checks fail ("missing") until the files exist. Two routes, both allowed by TC-1: add the three files by hand in a change PR **before the first sync** (the sync then finds them and leaves them alone), or add them to the `template-sync/<tag>` branch in a commit of their own and list each under "Changed files by class" in the PR body (`CODEOWNERS` as the one line for the approver of the project's own configuration). A second sync is not a route: it would add the files only after the failing first sync PR is merged.

**Rule TC-1 (template-sync paths).** A template-sync PR may change or remove only paths that are `owned` in the **base** manifest, may add only paths that are new in the **head** manifest and either `owned` or `seeded`, updates `.template/state.toml` with a version equal to the branch version, and never changes `.template/overrides.toml`. A new `seeded` path is only added (a sync never modifies or removes a seeded path), must be absent from the base commit, and is listed in the PR body (section "Changed files by class"). A seeded file that exists in the base is never changed.

**Init PR** (`init/v<MAJOR>.<MINOR>.<PATCH>`, the first change after "Use this template", ST-15). **Rule TC-2 (init).** An init PR may only delete `template-only` paths and write `seeded` files, the configuration file, and `.template/state.toml`. One exception, for `.claude/settings.json` only: an init PR may append entries to `permissions.deny`, and every appended entry must be listed in the head's `.claude/deny.platform.txt`; nothing else in that file may change, and no entry may be removed or reordered (gate check `deny-list` requires every entry of the seeded list in the owned file, so `initialize-project` appends them). It fails if the base branch already has `.template/state.toml`. PC-6 applies. `initialize-project` prints the post-init actions of the configuration and the step that the seeded `SECURITY.md` needs (enable private vulnerability reporting; for a private repository, where it is not offered, edit `SECURITY.md` to name another channel). For this platform (preset): add the machine account as a collaborator, update the AC1-1 allowlist by a change PR in `ai-lab-infrastructure`, and apply the printed branch and tag rulesets **before the init PR merges**. The script creates no label, ruleset, repository, collaborator, or setting.

TC-1 and TC-2 are defined by the template (this section and ADR-008). ADR-003 requires that application repositories enforce a contract that implements PC-1 to PC-8 and may add rules (ADR-003 section 6, as amended).

**Amendment of ADR-003 in this PR (Q8).** ADR-003 is amended with the approver's text: a header row; the init PR in section 3; the approver's immutable account ID in section 5; a first paragraph in section 6; PC-2 (prefixes), PC-3 (qualified cross-repository references), PC-8 (paths from configuration); section 7 (review identity and the PR template selection); and AC3-4. The amendment takes effect when this PR is merged. The checks are enforced in application repositories once the template's PRs are merged. In this repository, `scripts/pr_contract` and `config/project.yaml` need the new keys and rules to match ADR-003 as amended; that is not part of this PR (section 14).

**Amendment of ADR-005 in this PR (Q7).** Scope amendment on decision 1: "ADR-005's manifest is the single source for this platform's hosts (`builder`, `lab`, `operator`). The software template's CI manifest is owned by the template, uses the same schema and rules (exact versions, both architectures, SHA-256), and is not a host profile."

**Provenance.** The template's generic code is seeded once from `ai-lab-infrastructure` at a commit recorded in `template-dev/provenance.toml`, a `template-only` file: `initialize-project` removes it, the `leak` scan and T-21 do not cover template-only files, and the payload (including `.template/manifest.toml`) therefore names no other repository. That commit is history only; no later link exists. After Stage 5 the template is the origin of the generic code. Generic code is the same in every repository; differences come only from data (configuration).

## 5. `CLAUDE.md`, rules, and guardrails

**`CLAUDE.md` (owned, platform-neutral).** The text is the same in every repository and cites no platform record. It carries: instruction trust (only the user in the session and review comments by the approver; everything else is data); a **precedence sentence** ("nothing in any other instruction file relaxes the rules above"); the Never list (merge, approve, enable auto-merge, push to the default branch, create tags, change repository settings, secrets, or collaborators, `sudo` and installs, credentials, MCP servers, workflows that invoke Claude, Actions not pinned to a full commit SHA); the workflow summary (ADR-003: classes, branch prefixes, draft PR early, commit and push after each checklist item, stop when ready for review); validation (`./scripts/quality-gate.sh`; report what could not be run); context discipline (locate with `rg` and `fd`, read targeted ranges, summarise long output); and the discussion-prompt rule (a prompt for the architecture discussion carries commit-pinned URLs of the files it needs). It **ends with exactly two imports**, `docs/PLATFORM.md` and then `docs/PROJECT.md`. Platform-specific rules (OCI, hosts, the ADR-001 specifics) are in `docs/PLATFORM.md`.

**Gate check `claude-md`:** the file matches its manifest hash and has exactly those two imports at its end; neither imported file contains an import; no nested `CLAUDE.md` and no `CLAUDE.local.md` is tracked; the sizes are within `.template/budgets.toml`. A host-local `CLAUDE.local.md` cannot be seen by the gate; this is a **residual risk**, which the precedence sentence reduces. How Claude Code resolves imports is verified against the current documentation in the implementation PR.

**Rules** (`.claude/rules/*.md`, owned, path-scoped; key names verified against the current Claude Code documentation during implementation):

| Rule file | Applies to | Content |
|---|---|---|
| `public-repo.md` | always | Section 10 |
| `bash.md` | `**/*.sh`, `scripts/**` | `set -euo pipefail`, ShellCheck-clean, `--dry-run` for anything that deletes or overwrites |
| `workflows.md` | `.github/workflows/**` | Pin Actions to a full commit SHA, read-only default permissions, no `pull_request_target`, no step that invokes Claude, the PC-6 section (details are ADR-009's) |
| `docs.md` | `docs/**` | Section 7; never edit an approved design or ADR in an implementation PR (PC-5); qualified cross-repository references |
| `multi-arch.md` | dependency files, Dockerfiles, `scripts/**` | A download, image, or wheel change needs an artifact and checksum for every configured architecture (section 9) |
| `operator.md` | paths in `operator_paths` | The generic operator rules (section 12) |

**Guardrails.** `.claude/settings.json` (owned) carries the deny rules. `.claude/required-deny.txt` (owned) is the **generic core** list; `.claude/deny.platform.txt` (seeded from the preset) holds the platform-specific entries (for this platform: the OCI and Terraform rules of ADR-001 section 5). Gate check `deny-list` verifies that `settings.json` contains every entry of both; `initialize-project` appends the seeded entries to the owned file (the one exception of TC-2) and a sync keeps them (section 4, step 3). A project addition to `settings.json` is a divergence and goes through `overrides.toml`.

Deny rules match only Claude's own commands, not what a script runs. Gate check **`forbidden-api`** therefore rejects, in every script, the REST calls and GraphQL mutations for merge, review submission, auto-merge, rulesets, collaborators, and repository settings. Replying on a review thread and marking a PR ready are not in that list. That the core list plus this platform's list equals ADR-001 section 5 is a check in `ai-lab-infrastructure` (section 14), which is not decided here.

On a host with managed settings (ADR-001 section 5), those have precedence; the repository files are the second layer.

## 6. Skills

Each `SKILL.md` is short: when to use it, the inputs, the script to run, what to read and what not to, what to report. Mechanical logic is in the script, so it is tested in CI and costs no tokens. Scripts have `--fixture <dir>` (or `--offline`) so tests run without GitHub, and call GitHub through `gh api` with argument arrays (no `shell=True`), so the token never enters the script's environment.

| Skill | Script | Behaviour |
|---|---|---|
| `initialize-project` | `scripts/initialize-project --config <file>` | Writes the seeded files and `.template/state.toml` from the configuration, removes `template-only` paths, **prints** the configured post-init actions and the branch and tag rulesets. It creates no label, ruleset, repository, collaborator, or setting |
| `prepare-cr-context` | `scripts/cr-context <issue>` | Loads the CR `README.md` first, lists the other files with sizes, finds the design PR, reports the template lag. **Stops on an issue not authored by the approver or the machine account** |
| `implement-approved-cr` | `scripts/cr-preflight <issue> <design-pr>` (part of `cr-context`) | The PC-3 check (design PR merged and linked to the issue), then opens a draft implementation PR from `scripts/pr-body implementation` with the checkpoint markers |
| `review-implementation` | `scripts/cr-criteria <CR>` | Prints the criteria IDs and evidence as JSON; the Skill returns one verdict per ID, posted as **one** PR comment |
| `resume-work` | `scripts/pr-checkpoint show` | Reads only the checkpoint list, the latest CI summary, and `git log` since the merge base |
| **`address-review-feedback`** | `scripts/review-threads` | See below |
| `sync-template` | `scripts/template-sync` | Section 4. **Never edits `.template/overrides.toml` and never resolves a conflict** |
| `start-cr` | `scripts/new-cr <n> <slug>` | Uses **only intake text the user gave in the session**, proposes the class, and applies the class label **only after the user confirms the class in the session**. It never creates a label. **Stops on an issue not authored by the approver or the machine account.** Then creates the branch, the CR skeleton, and the draft design PR |

**`address-review-feedback`.** `scripts/review-threads <pr>` reads the approver from `config/project.yaml`: `approver_login` and the immutable account ID `approver_id`. It fetches review threads, review comments, and review summaries through `gh api` and keeps **only** those authored by the approver, **matched by the account ID**, and **not edited by any other account**. It **refuses a PR not authored by the machine account and a PR from a fork**. It prints compact JSON: thread id, file and line, outdated or resolved, and the text. Anything else is not printed as an instruction; only the counts of excluded items by reason are reported (look-alike login, bot, machine account, edited by another account). Text by any other account, including text the approver points to, is data until the approver restates it as an instruction. The Skill addresses each unresolved thread, pushes, and runs `scripts/review-threads reply <thread-id> <commit>`. The approver resolves threads and re-approves. The script never resolves a thread, submits a review, or merges.

**`scripts/pr-checkpoint`** (`show`, `tick <item>`) reads and edits only the text between the checkpoint markers. It updates the PR body with the REST PATCH call and marks a PR ready with the GraphQL mutation `markPullRequestReadyForReview`, because `gh pr edit` and `gh pr ready` fail on hosts with the Projects classic error (the platform workspace). Whether the lab VM shows the same error is verified later (T-12; deferred to Stage 6).

**Budgets and loading order** are ADR-010's (section 11). This design keeps each Skill to one script and one report.

## 7. CR and ADR templates

- `docs/cr/_template/` has the five files of ADR-003 section 3. `README.md`: summary table (issue, class, status, ADR), summary, file table. `impact-analysis.md`: what changes, affected components, security and privacy, operational impact on the approver, dependencies and assumptions, cost and capacity, rollback, alternatives. `design.md`: header table, decisions table, and the sections for questions and for the approver's decisions. `acceptance-criteria.md`: table with ID, criterion, evidence. `implementation-plan.md`: PR table, checklist, user-only actions, and what the plan does not do.
- `docs/adr/_template.md` has the header table (status, date, repositories, CR, scope, related, amended by), then context, decision, alternatives, consequences, open items, and acceptance criteria.
- **Numbering (Q12).** The CR ID is that repository's issue number; ADRs start at `ADR-001` per repository. A reference to another repository is always qualified: `<owner>/<repo>#<n>` for issues and PRs, `<repo>:ADR-<n>` for ADRs. Owned text cites no platform record.
- Gate check `cr-layout`: every `docs/cr/CR-<n>/` has the five files, criteria IDs are unique, every plan row names a criterion ID, and a reference to another repository is qualified (a local `#<n>` or a qualified reference is accepted; nothing else).
- The template's own later CRs live in `template-dev/docs/cr/`, numbered from the template repository's issues.
- `scripts/new-cr <n> <slug>` copies the skeleton and refuses to overwrite.

## 8. The template's quality gate and CI

**Generated into every repository.** The entry point `scripts/quality-gate.sh` and the generic checks (ShellCheck, Bash hygiene, YAML, Markdown, links, actionlint, workflow policy, no Claude in workflows, public-repo hygiene, Python, gate tests) plus the template checks `claude-md`, `deny-list`, `forbidden-api`, `owned-drift`, `operator-contract`, `cr-layout`, and `size-budget`. Two checks cover the seeded files of section 3: **`dependabot-config`** (generic) requires `.github/dependabot.yml` to have an update entry for `github-actions` that covers `/`, no entry with `open-pull-requests-limit: 0`, and no `ignore` rule that switches every update off; **`codeowners`** (template class) requires `.github/CODEOWNERS` to be exactly the one line `* @<approver_login>` for the approver of `config/project.yaml`, so that the approver's review counts where a ruleset requires code-owner review. Project checks go in `scripts/checks.d/`. The operator-contract and Ansible checks of this platform are not copied as they are: `operator-contract` is rewritten as the generic check on `operator_paths` (section 12).

**Gate check `owned-drift`** fails a PR that changes an owned file (against the hashes in `.template/state.toml`) without a matching entry in `.template/overrides.toml`, and passes with one. A template-sync or init PR changes `state.toml` together with the files, so it passes. **`uses:`-only changes (Q18).** A change limited to the `uses:` lines of an owned workflow (a Dependabot pin bump) is allowed by `owned-drift` without an override entry, consistent with sync overwriting and listing such drift. It must keep the same action: only the pin and the trailing comment change; the action name, the indent, the `- ` marker, the number of lines, and every other line stay. PC-7 lets a Dependabot PR change only dependency manifests, lock files, and workflow `uses:` pins, and such a PR cannot touch `overrides.toml`, so any other reading would fail every Dependabot Actions PR.

**CI.** `ci.yml` runs the gate as a **matrix over the configured architectures and runner labels**, named `quality-gate-<arch>`; `pr-contract.yml` runs `pr-contract` from the base checkout on `pull_request` with a read-only token and no secrets (ADR-003 section 6). The checks whose policy input comes from the base commit run from the base checkout against the head: `workflow-policy`, `no-claude-in-workflows`, `forbidden-api`, `operator-contract`, `cr-layout`, `owned-drift`, and `dependabot-config` (the base's copy reads the head's `.github/dependabot.yml`, so a PR that deletes or empties it is found). `codeowners` is not on the list: it compares the head's file with the head's configuration and has no policy input from the base. **Required status checks (Q17)** are the aggregate jobs `pr-contract` and `quality-gate` (section 10 says where a required check can be enforced), and `template-ci` in the template only; the matrix legs (`quality-gate-<arch>`, `generate-<arch>-<config>`, `skills-<arch>`, `sync-<arch>`) are not required names. Each aggregate needs its other jobs, runs with `if: always()`, and fails unless every needed job succeeded (a skipped or cancelled job is not a pass), and the leg names follow `architectures` of `config/project.yaml`, so a fixed list of them would go stale. Where no runner exists for an architecture, the job summary says `not run: <arch>`, and the change PR template requires a host-reported evidence line with the commit SHA. The condition reads `github.event.repository.private`.

**Template-only, in `template-dev/` (run by `.github/workflows/template-ci.yml`):**

| Test | What it proves |
|---|---|
| `manifest` | Every tracked file has exactly one class; `initialize-project` removes every template-only path |
| `generate` | The dry run of section 12 (steps 1 to 4 and 8) with two configurations, on both architectures |
| `sync` | Section 12, step 6 |
| `skills` | Skill and script fixtures (steps 5 and 7); every script has `--fixture` or `--offline` |
| `leak` | No identifier pattern and no platform reference in the payload (every file not classed `template-only`) or in a generated tree |

**Tools in CI.** The template carries the install engine and its **own CI manifest** (the gate tools, exact versions, both architectures, SHA-256) under the schema and rules of ADR-005 (decisions 1, 3, 4). It is not a host profile, and ADR-005 is amended to say so (section 4). CI fetches no other repository.

**Relation to ADR-009.** ADR-009 decides the repository-wide security and CI baseline: the settings matrix, Actions hardening, which checks are required status checks, and code scanning. ADR-008 decides what the template carries and how it is tested. This design **assumes** the practice of this repository (required checks on a public repository; Actions pinned by SHA; read-only default permissions; no `pull_request_target`; no workflow that invokes Claude; Dependabot for Actions pins). If ADR-009 differs, the template changes by a release and a sync.

## 9. Architectures: what is specific to the ARM64 lab VM and what is not

The template is architecture-neutral and names no architecture. Everything below that is specific to this platform comes from this platform's preset.

| In the template (neutral) | In this platform's preset (`ai-lab-infrastructure`) |
|---|---|
| Template content, `CLAUDE.md`, rules, Skills, CR and ADR templates, `pr-contract`, scripts (Bash, standard-library Python) | `architectures`: amd64 and arm64; runner labels `ubuntu-24.04` and `ubuntu-24.04-arm`; the lab VM is ARM64 Ubuntu 24.04 and the workspace is x86-64 Debian 13 (ADR-001 O-1, ADR-005) |
| Rule `multi-arch.md`: a dependency, download, image, or wheel change needs an artifact and checksum for every configured architecture; `uname -m` branching only in the install engine | `docs/PLATFORM.md`: where Claude Code runs (workspace, lab VM through Desktop SSH), the lab VM clone under `/srv/work`, the backup protection of uncommitted work, and the rule to commit and push after each checklist item |
| CI matrix over configured architectures and runner labels; `not run: <arch>` with a host-reported evidence line where no runner exists (Q9) | Hosted ARM64 runners are free for public repositories only (ADR-005 context), so a private repository has no runner for arm64 and relies on the evidence line (D9-6) |
| The template's generation test on both architectures in its own CI | Tools on the lab VM come from the `lab` profile (ADR-005 O5-5, Stage 3); the template assumes tools are on `PATH` and installs none |

Nothing in the template needs the lab VM before Stage 6. The dry run uses the workspace and GitHub's ARM64 runner as the proxy.

## 10. Public-repository rules

Apply to the template and, by this platform's preset, to every application repository (public by default, Q9):

- No tenancy or resource OCIDs, IP addresses, usernames, emails, keys, fingerprints, or tokens in files, commits, PR text, or logs. Use variables and placeholders. Gate check `public-repo-hygiene` runs on every repository, public or private, because a repository can become public.
- Pin every version and Action to a full commit SHA; verify checksums or signatures; no `curl | sh`.
- **Public and private repositories are supported.** `pr-contract` is a required status check on a public repository and advisory on a private one on GitHub Free (ADR-003 section 6). The deny rules cover only Claude's own commands: they do not cover a script's subprocesses, a command typed by a person, or a push from a compromised host. On a private repository without rulesets, branch protection is therefore procedural (ADR-001 RR-10). The `forbidden-api` check reduces the script part only.
- **This platform's policy** (preset): public by default; private only with the per-repository accepted-risk note of ADR-001 RR-10.
- The identities in the configuration are public logins and an account ID. No other personal data is stored.
- Which repository features (secret scanning, push protection, code scanning) are on is ADR-009's.

## 11. Dependencies on ADR-009, ADR-010, and ADR-005, and the assumptions

Listed, not decided.

| ID | Depends on | Assumption in this design |
|---|---|---|
| D9-1 | ADR-009: required status checks and ruleset content for application repositories | As in this repository: required on a public repository. **Changed (Q17):** the required names are the aggregate jobs `pr-contract`, `quality-gate`, and (in the template only) `template-ci`; the matrix legs are not required names (section 8) |
| D9-2 | ADR-009: Actions hardening rules | The current rules of `scripts/checks/workflow-policy.sh`, seeded into the template; ADR-009 may change them |
| D9-3 | ADR-009: repository security features, code scanning, `SECURITY.md` and community files | ADR-008 ships three seeded files in v1: `.github/dependabot.yml`, `.github/CODEOWNERS`, and `SECURITY.md` (Q19). Code scanning and other community files: none |
| D9-4 | ADR-009: fork PRs and third-party Actions | `pr-contract` runs on `pull_request` with a read-only token; no Action that is not pinned |
| D9-5 | ADR-009: cross-repository checkout in CI | **Raised and withdrawn:** not needed after Q7 |
| D9-6 | ADR-009: host-reported evidence for architectures with no hosted runner | Informational evidence that the approver accepts per PR |
| D9-7 | ADR-009: content of the branch and tag rulesets that `initialize-project` prints | As in this repository and ADR-011 decision 3 |
| D9-8 | ADR-009: the gate checks `forbidden-api`, `operator-contract`, and `owned-drift` shipped by the template | ADR-009 may extend them and does not remove them |
| D10-1 | ADR-010: size budgets and loading order for `CLAUDE.md` plus the imported `docs/PLATFORM.md` and `docs/PROJECT.md` (loaded at start), rules, and the Skills including the two added | Placeholder values in `.template/budgets.toml` (`CLAUDE.md` 120 lines, a rule 40, a `SKILL.md` 60); ADR-010 sets them without a code change |
| D10-2 | ADR-010: checkpoint format beyond the markers, and metrics | Only the ADR-003 markers. Scripts print one summary line per run |
| D5-1 | ADR-005: profile names for the template's CI tools | **Closed by Q7:** the template's CI manifest is not a host profile; ADR-005 is amended in this PR |
| D3-1 | ADR-003: template-sync and init PR rules | **Closed by Q8:** ADR-003 is amended in this PR; TC-1 and TC-2 are defined by the template |

## 12. The Stage 5 exit and the generic operator contract

**Stage 5 exit needs both (Q11).**

**(a) The scripted dry run**, entirely in template CI, on both architectures, with two different configurations (`template-dev/dry-run.sh --template <path> --config <file> --out <empty dir>`; it refuses a non-empty directory and writes only inside it; no GitHub repository, credential, or OCI resource):

1. Export the template commit without history, like "Use this template" (`git archive`).
2. Run `initialize-project --config` non-interactively; every `template-only` path is removed; the tree is classified.
3. `git init`, commit, run `./scripts/quality-gate.sh --strict` in the generated repository: green.
4. Run `scripts/new-cr 1 sample`; `cr-layout` passes; a CR with a missing file or an unqualified cross-repository reference fails it.
5. Run `pr-contract` against fixture PRs (design, implementation, change, template-sync, init; passing and failing bodies and paths, including TC-1 and TC-2).
6. Sync: generate from version A, change one owned file, one seeded file, and one workflow `uses:` line in the template (version B); change one owned file locally, and one locally with an override entry; run `template-sync --from-dir`: the locally changed owned file is overwritten, the overridden one is kept and listed, the `uses:` drift is overwritten and listed, the seeded file is untouched, `state.toml` is updated, and a second run changes nothing. A downgrade and a moved tag are refused.
7. Run `review-threads --fixture` on threads by the approver, the machine account, a bot, a look-alike login, and an approver comment edited by another account: only the approver's unedited items are output, and the counts of the others are reported.
8. Leak scan of the generated tree: no unfilled placeholder, no identifier pattern, no platform reference (T-21).
9. Run steps 1 to 8 on x86-64 and on ARM64 (template CI), with a neutral configuration and a second, different one.
10. Write a report with one line per step; it goes in the last implementation PR.

**(b) One real run by the approver** in a throwaway public repository with this platform's preset (user-only): "Use this template"; the init PR passes `pr-contract`; a change PR with a known violation fails and then passes; one review round with an approver comment and a machine-account comment, where only the approver's is acted on; one template-sync PR from a second tag. Afterwards the repository is deleted, the machine account is removed as collaborator, and the AC1-1 allowlist is restored. **Prerequisite:** the preset change PR in `ai-lab-infrastructure`. Claude Code never creates the repository.

**Generic operator contract** (ADR-011 decision 17, Q10). The template ships a platform-neutral rule `operator.md`, the PC-8 PR section "Operator script review", and no operator scripts or profile. PC-8 is active when `operator_paths` in the configuration is not empty. The rule states the generic part of what ADR-011 and CR-18 fixed: every operator script has a dry-run mode and an apply mode with dry run as default; credentials are prompted for or short-lived, never stored, and handled inside the script process; fact collection is read-only, enforced in code; output is safe to paste; scripts never change infrastructure; Claude Code never runs them. It cites no platform record; the ADR-011 specifics are in `docs/PLATFORM.md` of this platform's preset. Enforcement:

- Gate check `operator-contract`, on `operator_paths`: rejects `shell=True`, `os.system`, `eval`, `exec`, a token read from `argv` or the environment, a subprocess outside the configured allowlist, and a missing dry-run default. Inactive when `operator_paths` is empty.
- `scripts/lib/operator_guard.py` refuses an apply run unless `HEAD` equals a release tag contained in the default branch.

## 13. Decisions of the approver (2026-11-19, 2026-11-27, and 2026-11-28)

Q1 to Q14 are recorded from the approver's review on PR #40 (architecture discussion, 2026-11-19); Q15 and Q16 from the approver's decisions of 2026-11-27; Q17 to Q20 from the approver's decisions of 2026-11-28. Wording is condensed; each decision is applied in the section named.

| # | Decision | Applied in |
|---|---|---|
| Q1 | Root payload. New file class `template-only` (`template-dev/`, `.github/workflows/template-ci.yml`), removed by `initialize-project`. No CR or ADR instances under `docs/` in the template; its records live in `template-dev/docs/` (CR-39 stays here as the one transitional exception). `initialize-project` takes every value from `--config` and never inherits the template's config file or `README.md` | ST-1, ST-2, section 3, T-1, T-2 |
| Q2 | Tags `v<MAJOR>.<MINOR>.<PATCH>`, approver only, enforced by a tag ruleset on the template; sync targets tags only; the repository's own reviewed copy of `template-sync` does all checks and copying; a MAJOR release with a "sync twice" note when a release needs a new script first; downgrades and moved tags refused (tag commit recorded in `state.toml`); template clone with credential helpers disabled; no schedule, event, or workflow; lag reported read-only | ST-3, section 4, plan section 3, T-10 |
| Q3 | Locally changed owned files are overwritten by default. Divergences are declared in the seeded `.template/overrides.toml` (path, reason, PR reference); sync honours and lists them and records them as `override`. Drift limited to `uses:` lines in owned workflows is overwritten and listed. New gate check `owned-drift` | ST-4, sections 3, 4, 8, T-9, T-17 |
| Q4 | `CLAUDE.md` fully owned and platform-neutral; zoned class dropped; precedence sentence; ends with the imports of `docs/PLATFORM.md` and `docs/PROJECT.md` (both seeded); the gate rejects imports in those files, a tracked nested `CLAUDE.md`, and a tracked `CLAUDE.local.md`; host-local `CLAUDE.local.md` is a residual risk; import behaviour verified in implementation | ST-5, sections 3 and 5, T-1, T-4, T-9 |
| Q5 | Owned generic core `.claude/required-deny.txt`; seeded `.claude/deny.platform.txt` from the preset; `deny-list` verifies both; project additions through `overrides.toml`; new check `forbidden-api`. That core plus this platform's list equals ADR-001 section 5 is checked in `ai-lab-infrastructure` (open) | ST-6, sections 5 and 8, T-5, T-19 |
| Q6 | Yes to `sync-template` and `start-cr`, with the constraints of section 6 | ST-7, section 6, T-13 |
| Q7 | The template is the origin of its generic code (seeded once at a recorded commit, history only); it carries the engine and its own CI manifest; generic code identical across repositories; ADR-005 amended in scope | ST-8, sections 4 and 8, ADR-005, T-14 |
| Q8 | Amend ADR-003 in this PR, reframed as this platform's minimum requirement on the template; TC-1 (template-sync paths) and TC-2 (init) are defined by the template | Section 4, ADR-003, ADR-008, T-8 |
| Q9 | Public and private supported; matrix from configured architectures and runner labels; `not run: <arch>` plus a host-reported evidence line; the condition reads `github.event.repository.private`; the "only enforcement" claim corrected; the platform's policy in its preset | ST-9, ST-10, sections 8 to 10, T-3 |
| Q10 | Platform-neutral `operator.md`; PC-8 active when `operator_paths` is not empty; generic `operator-contract` check; `operator_guard.py`; no operator scripts or profile; ADR-011 specifics in `docs/PLATFORM.md` | ST-11, sections 8 and 12, T-6, T-22 |
| Q11 | Stage 5 exit needs both the scripted dry run in CI and one real run by the approver in a throwaway public repository | ST-12, section 12, plan, T-16 |
| Q12 | Per-repository numbering; qualified cross-repository references; owned text cites no platform record; `cr-layout` accepts a local `#<n>` or a qualified reference only; the template's later CRs in `template-dev/docs/cr/` | ST-14, section 7, T-7 |
| Q13 | New init PR type `init/v<MAJOR>.<MINOR>.<PATCH>` with TC-2; `initialize-project` prints the configured post-init actions | ST-15, sections 3 and 4, T-20 |
| Q14 | The template is decoupled from `ai-lab-infrastructure` and configurable (approver requirement); the preset lives in `ai-lab-infrastructure` (change PR outside CR-39); `host: workspace` becomes a constraint of this platform's preset | ST-13, sections 1, 3, 9, 11, T-21 |
| Q15 | Decision of 2026-11-27, option B (TC-1). A template-sync PR may also add paths that are new and `seeded` in the head manifest, in addition to new `owned` paths. Such a path is only ever added, must be absent from the base commit, and is listed in the PR body (section "Changed files by class"). Seeded files that exist in the base are never changed. The rest of TC-1 is unchanged. Section 4 step 3 and the rule now agree | Section 4 (step 3, TC-1), ADR-008 decision 4, T-9 |
| Q16 | Decision of 2026-11-27, option 1 (TC-2). An init PR may append entries to `permissions.deny` of `.claude/settings.json`, and only there; every appended entry must be listed in the head's `.claude/deny.platform.txt`; nothing else in the file may change and no entry may be removed or reordered. A sync that overwrites the owned `settings.json` produces the tag's version plus every entry of the project's seeded list that the tag's list lacks, appended to `permissions.deny`; TC-1 already allows it because the file is owned in the base manifest. The gate check `deny-list` is unchanged | Section 3 (`settings.json`), section 4 (step 3, TC-2), section 5, ADR-008 decision 4, T-20, plan row 8 |
| Q17 | Decision of 2026-11-28 (D9-1). The required status checks of the template and of application repositories are the aggregate jobs `pr-contract`, `quality-gate`, and `template-ci` (the last in the template only). The matrix legs (`quality-gate-<arch>`, `generate-<arch>-<config>`, `skills-<arch>`, `sync-<arch>`) are not required names. Reason: the `quality-gate` aggregate needs `plan` and every leg, runs with `if: always()`, and fails unless every needed job succeeded (a skipped or cancelled job is not a pass), and the leg names depend on `architectures` of `config/project.yaml`, so a fixed ruleset would go stale. The earlier D9-1 wording (the list covers the matrix job names) is replaced | Section 8 (CI), section 11 (D9-1) |
| Q18 | Decision of 2026-11-28 (`owned-drift` and `uses:`-only changes). The reading of section 8 is confirmed: a change limited to the `uses:` lines of an owned workflow is allowed by `owned-drift` without an override entry, for the same action only (only the pin and the trailing comment change; the action name, the indent, the `- ` marker, the number of lines, and every other line stay). Reason: PC-7 lets a Dependabot PR change only dependency manifests, lock files, and workflow `uses:` pins, and such a PR cannot touch `overrides.toml`, so the other reading would make every Dependabot Actions PR fail. In the template the switches are `OWNED_DRIFT_ALLOWS_USES_ONLY` and `SAME_ACTION_ONLY` in `scripts/lib/template_sync/uses_drift.py` | Section 8 (`owned-drift`), section 14 |
| Q19 | Decision of 2026-11-28 (seeded files). The template ships three seeded files: `.github/dependabot.yml` (version updates, `github-actions` on `/`), `.github/CODEOWNERS` (the one line `* @<approver_login>`, written by `initialize-project` from `github.approver_login`), and `SECURITY.md` (generic text that relies on private vulnerability reporting, which `initialize-project` lists as a printed post-init action). Code scanning and other community files are not shipped. Gate checks `dependabot-config` (generic) and `codeowners` (template class); `dependabot-config` is on the base-run list of `pr-contract.yml`, `codeowners` is not (it compares the head's file with the head's configuration and has no policy input from the base). Facts of the implementation recorded with it: when a sync adds a seeded path and what it does in a shallow clone; `CODEOWNERS` is written from the project's own `approver_login`; a project whose own sync script is older cannot get the files with its first sync (TC-1 routes) | Section 3 (file table), section 4 (step 3, first-sync paragraph, init PR), section 8 (checks, base-run list), section 11 (D9-3) |
| Q20 | Decision of 2026-11-28 (no `pip` entry). The seeded `dependabot.yml` has no `pip` entry for `toolchain/python`. Its files (`ci.in`, `ci.lock`) and the checksums of `toolchain/ci-manifest.toml` are owned gate tooling, identical in every project and changed together by template releases; a Dependabot lock bump would be an owned-file change (`owned-drift`) and would break the checksum-verified install. Projects add their own ecosystems to the seeded file | Section 3 (`dependabot.yml`) |

Q17 to Q20 are applied by the design PR `design/CR-39-seeded-files`; Q15 and Q16 by the design PR `design/CR-39-tc-amendments`.

Two consequences I derived and the approver may correct: the rule `arm64.md` of the first draft is `multi-arch.md` because the payload must not name an architecture (Q14), and the OCI and Terraform entries of the Never list moved to `deny.platform.txt` and `docs/PLATFORM.md` for the same reason.

## 14. Still open

| Item | Who decides | When |
|---|---|---|
| This platform's preset in `ai-lab-infrastructure` (identities, `deny.platform.txt`, `docs/PLATFORM.md` with the ADR-001 and ADR-011 specifics, architectures and runner labels, post-init actions, the RR-10 note for private repositories). Claude Code proposes it as a change PR | Approver | Before the real run of section 12 (b) |
| The check that the core deny list plus this platform's `deny.platform.txt` equals ADR-001 section 5, in `ai-lab-infrastructure` (form) | Approver | With the preset PR |
| Whether `ai-lab-infrastructure` later adopts the template's generic code | Approver (a later CR) | After Stage 5 |
| **This repository does not yet implement ADR-003 as amended:** `config/project.yaml` has no `approver_id`, `design_repos`, or `operator_paths`, and `scripts/pr_contract` hard-codes the PC-8 paths, accepts only same-repository PC-3 references, and has no `template-sync` or `init` type. The amendment text states these rules; this repository follows through the preset or adoption PR | Approver (form of the change) | With the preset PR |
| Provenance location (found while preparing implementation PR 1): the first design put the provenance of the seeded generic code in the payload manifest, which T-21 forbids from naming another repository. Resolved by the approver on 2026-11-19, following the recommendation: `template-dev/provenance.toml`, a template-only file | Closed by this amendment | This PR |
| TC-1 and TC-2 against the template's own design (found while preparing implementation PR 3): TC-1 allowed only new `owned` paths while section 4 step 3 adds a new seeded file; TC-2 rejected the append of the preset's entries to the owned `settings.json` that `deny-list` requires. Resolved by the approver on 2026-11-27 (Q15, Q16): TC-1 also allows a new `seeded` path; TC-2 has one exception for `permissions.deny` of `.claude/settings.json` | Closed by this amendment | This PR |
| ADR-003 header state: the amendment row of CR-21 said "Proposed" at `b7257f7`. I checked: the CR-21 design PR (#22) and implementation PR (#24) are merged, so PC-8 is in force and enforced. The header row now says so | Closed by the amendment | This PR |
| `owned-drift` and `uses:`-only changes (Dependabot pin bumps): the reading of section 8 was open. Resolved by the approver on 2026-11-28 (Q18): allowed without an override entry, for the same action only | Closed by this amendment | This PR |
| Found while implementing the template: D9-1 listed the matrix job names as required checks (a fixed list goes stale and a skipped leg is not a pass); D9-3 said that no file but `dependabot.yml` is shipped; the seeded `dependabot.yml` needed a decision on `pip`; a project with an older sync script cannot get the new seeded files with its first sync. Resolved by the approver on 2026-11-28 (Q17, Q19, Q20) | Closed by this amendment | This PR |
| Verification items for implementation, recorded in the PRs: `CLAUDE.md` import behaviour (two imports, no nesting); GitHub semantics for editing other users' comments and the editor fields; current availability of hosted ARM64 runners for private repositories; T-12 on the lab VM, deferred to Stage 6 | Claude Code | Implementation PRs |
| Stage 5 entry: ADR-009 and ADR-010 approved | Approver | Before implementation PR 1 |
