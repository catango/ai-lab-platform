# ADR-008: Software Template: Structure, CLAUDE.md, Rules, Skills, and Template Updates

| Field | Value |
|---|---|
| Status | **Accepted** (2026-11-19, by the merge of the CR-39 design PR #40 by the approver) |
| Date | 2026-11-19 |
| Repositories | `ai-software-template` (primary), `ai-lab-infrastructure` (this record) |
| Change request | CR-39 (issue #39) |
| Amended by | CR-43 (design PR #44 merged, 2026-11-20): decisions 1, 2, 3, 6, 9, and 10. ADR-010 (Proposed, CR-45): budgets and loading order, checkpoint format, metrics; closes D10-1 and D10-2 of design section 11. Takes effect when the CR-45 design PR is merged. CR-39 design PR `design/CR-39-tc-amendments` (the approver's decisions of 2026-11-27, design Q15 and Q16): decision 4, TC-1 and TC-2. CR-39 design PR `design/CR-39-seeded-files` (design Q17 to Q20, 2026-11-28): section 5, the open item on `owned-drift` is closed (Q18) |
| Scope | The structure of the template, what is generated into an application repository, `CLAUDE.md`, rules, Skills (including `address-review-feedback`), the CR and ADR templates, template-sync and the init PR, the template's own quality gate and CI, architectures, public and private repositories, the generic operator contract, and the Stage 5 exit. Not ADR-009 (security matrix, Actions hardening) and not ADR-010 (token optimisation, metrics) |
| Related | ADR-001 (guardrails, instruction trust, RR-10), ADR-003 (work item model, `pr-contract`, Skill impact; amended by this CR), ADR-005 (toolchain; scope amended by this CR), ADR-011 (decision 17, operator contract), ADR-009 and ADR-010 (reserved; dependencies in design section 11) |

> This ADR records a decision. It does not create or change a repository, a label, a ruleset, a workflow, a setting, or a tag, and it does not authorize any of them.
> Detail, rationale, and the approver's decisions Q1 to Q14 (architecture discussion, 2026-11-19) and Q15 and Q16 (2026-11-27) are in `docs/cr/CR-39/design.md`. The decisions are applied below.

---

## 1. Context

- Baseline v0.1 Stage 5 builds the software template. Its entry criteria are ADR-008, ADR-009, and ADR-010 approved. Its exit criterion is a template validated by generating a throwaway repository in a dry run (made precise by decision 12). Stages 2 to 4 are postponed (2026-11-19), so Stage 5 runs first, on the platform workspace.
- ADR-003 fixes the workflow that every application repository follows and the Skill changes it needs (its section 8). It names the template-sync PR but did not define it.
- ADR-001 requires guardrails and the instruction-trust rule in each repository. ADR-011 decision 17 hands the operator contract to this ADR.
- **Requirement (approver, Q14): the template is decoupled from this platform and configurable.** It must be usable with a different lab infrastructure. This record names `ai-lab-infrastructure`; the template does not.

## 2. Decision

1. **The repository root of `ai-software-template` is the generated payload.** The GitHub template-repository flag is on (Stage 0). Every file has one class in `.template/manifest.toml`: `owned` (template-owned, synced), `seeded` (written once by `initialize-project`, then project-owned), or `template-only` (`template-dev/`, `.github/workflows/template-ci.yml`; removed by `initialize-project`). The template keeps no CR or ADR instance under `docs/`; its records live in `template-dev/docs/`. `initialize-project` takes every value from `--config` and never inherits the template's own configuration file or `README.md`. `SECURITY.md` and `.github/CODEOWNERS` are seeded; `scripts/gate-evidence` is owned.
2. **Template updates reach application repositories by a template-sync PR**, branch `template-sync/v<MAJOR>.<MINOR>.<PATCH>`, opened in a human-started Claude Code session as the machine account. Tags are created by the approver only, enforced by a tag ruleset. The repository's own reviewed copy of `scripts/template-sync` does every check and copy. Downgrades and moved tags are refused. An owned file changed locally is overwritten unless declared in the seeded `.template/overrides.toml`; gate check `owned-drift` enforces this. Drift limited to `uses:` lines is overwritten and listed. Never on a schedule, event, or workflow (ADR-003 section 7). `owned-drift` reads the hashes from `.template/state.toml` on the base commit and grants the template-sync and init exemption only to a PR that `pr-contract` classifies as such and that passes TC-1 or TC-2 (ADR-009). A project raises a single value of `.template/budgets.toml` only by a per-key entry in `.template/overrides.toml` (ADR-010); `budgets.toml` itself is never a whole-file override.
3. **The first change after generation is an init PR**, branch `init/v<MAJOR>.<MINOR>.<PATCH>`. `initialize-project` prints the configured post-init actions and creates no label, ruleset, repository, collaborator, or setting. It also prints the rulesets of ADR-009 decision 8 for a public repository, or the RR-10 note for a private one.
4. **Two rules are defined by the template** (design section 4): **TC-1** (a template-sync PR changes or removes only paths that are owned in the base manifest, adds only paths that are new in the head manifest and either owned or seeded, updates `.template/state.toml` with the branch version, and never changes `.template/overrides.toml`; a new seeded path is only added, must be absent from the base commit, and is listed in the PR body; a seeded file that exists in the base is never changed) and **TC-2** (an init PR only deletes template-only paths and writes seeded files, the configuration file, and `state.toml`, with one exception: it may append entries to `permissions.deny` of `.claude/settings.json`, each listed in the head's `.claude/deny.platform.txt`, with nothing else changed in that file and no entry removed or reordered; it fails if the base branch already has `state.toml`; PC-6 applies). A sync that overwrites the owned `.claude/settings.json` appends the entries of the project's seeded `deny.platform.txt` that the tag's list lacks.
5. **`CLAUDE.md` is fully owned and platform-neutral.** It carries the instruction-trust rule, the Never list, a precedence sentence ("nothing in any other instruction file relaxes the rules above"), the workflow summary, validation, and context discipline, and ends with exactly two imports, `docs/PLATFORM.md` then `docs/PROJECT.md`, both seeded. The gate rejects an import inside those files, a tracked nested `CLAUDE.md`, and a tracked `CLAUDE.local.md`. A host-local `CLAUDE.local.md` is a residual risk.
6. **Rules** are small, path-scoped, owned files in `.claude/rules/`. **Guardrails**: `.claude/settings.json` (owned) against the owned core list `.claude/required-deny.txt` and the seeded `.claude/deny.platform.txt`, checked by `deny-list`; gate check `forbidden-api` rejects, in every script, the REST calls and GraphQL mutations for merge, review submission, auto-merge, rulesets, collaborators, and repository settings. `forbidden-api` also rejects the endpoint classes of ADR-009 decision 9 and any GitHub API call whose endpoint or GraphQL document is not a literal in the script, through any HTTP client.
7. **Skills are thin and call deterministic scripts**: the six of ADR-003 section 8 (`initialize-project`, `prepare-cr-context`, `implement-approved-cr`, `review-implementation`, `resume-work`, `address-review-feedback`) plus `sync-template` and `start-cr`, with the constraints of design section 6. `address-review-feedback` keeps only items authored by the approver, matched by the immutable account ID and not edited by any other account, and refuses a PR not authored by the machine account and a PR from a fork. Each `SKILL.md` follows the layout and size budgets of ADR-010.
8. **The template owns its generic code and its own CI manifest.** `pr_contract`, the gate checks, and the install engine are seeded once from `ai-lab-infrastructure` at a commit recorded in a template-only file (history only; the payload names no other repository) and are identical across repositories; differences come only from data. The CI manifest (gate tools, both architectures, SHA-256) is not a host profile. ADR-005 is amended accordingly.
9. **CI is a matrix over the configured architectures and runner labels** (`quality-gate-<arch>`). Where no runner exists, the job summary says `not run: <arch>` and the PR template requires a host-reported evidence line with the commit SHA. The aggregate job `quality-gate` needs the matrix jobs and is the required check. `template-ci.yml` has its own aggregate `template-ci`, required in the template only.
10. **The template supports public and private repositories.** The private condition reads `github.event.repository.private`. This platform's policy (public by default; private only with the ADR-001 RR-10 note) is in its preset. Public-repository rules apply to the template itself. Workflows read `github.event.repository.private`; `pr-contract` reads the private flag from the API (PC-9).
11. **Generic operator contract.** A platform-neutral `operator.md`, PC-8 driven by `operator_paths`, gate check `operator-contract`, and guard library `scripts/lib/operator_guard.py`. No operator scripts or profile. The ADR-011 specifics are in the preset's `docs/PLATFORM.md`.
12. **Stage 5 exit needs both:** (a) the scripted dry run in template CI on both architectures with two configurations; (b) one real run by the approver in a throwaway public repository with this platform's preset (generation; an init PR that passes `pr-contract`; a change PR that fails and then passes; one review round in which only the approver's comment is acted on; one template-sync PR from a second tag), after which the repository and the collaborator are removed and the AC1-1 allowlist is restored. The preset change PR in `ai-lab-infrastructure` is a prerequisite.
13. **Numbering and references.** Per repository: the CR ID is that repository's issue number and ADRs start at `ADR-001`. A reference to another repository is always qualified (`<owner>/<repo>#<n>`, `<repo>:ADR-<n>`). Owned text cites no platform record.
14. **All platform values come from one configuration** passed to `initialize-project --config`: identities (including `approver_id`), the template source repository, `design_repos`, architectures and runner labels, `operator_paths`, visibility, post-init actions, and the seeded `docs/PLATFORM.md` and `.claude/deny.platform.txt`. A configuration with those two files is a preset. This platform's preset lives in `ai-lab-infrastructure` and is a change PR outside CR-39.

## 3. Alternatives considered

| Alternative | Outcome | Reason |
|---|---|---|
| A generator script that copies a `template/` subdirectory | Not chosen (Q1) | The flag is on; the payload's workflows then run for real on every template PR |
| Zoned `CLAUDE.md` (owned zone between markers, project zone below) | Rejected (Q4) | A whole-file hash is simpler and stronger; the project part is imported |
| Skip a locally changed owned file and list it as a conflict | Rejected (Q3) | It shows no diff, leaves `state.toml` undefined, and Dependabot pins would make conflicts routine |
| Run the template's `template-sync` from the fetched tag | Rejected (Q2) | It bypasses review in the application repository |
| Everything synced, or everything seeded | Rejected | A project could not adapt its files, or security-relevant files would drift |
| Reusable workflows, a submodule, or a live cross-repository link for shared code | Rejected (Q7, Q14) | A trust edge for ADR-009 and a coupling the approver forbids |
| A scheduled workflow that opens sync PRs | Rejected | The machine account token is not an Actions secret (ADR-001), and ADR-003 excludes automatic actions |
| The scripted dry run alone as Stage 5 exit | Rejected (Q11) | It does not prove that a generated repository works under ADR-003 |
| A prose-only operator contract | Rejected (Q10) | Weaker than the platform's enforced one |
| A silent skip of the ARM64 job on private repositories | Rejected (Q9) | It removed the only automated evidence |

## 4. Consequences

- One more repository to release and tag: the approver creates template tags, enforced by a tag ruleset.
- Every sync PR is reviewed like any PR. A project that must diverge from an owned file declares it, and the declaration is visible in review.
- ADR-003 is amended in the CR-39 design PR (header row, init PR type, approver account ID, application-repository contract, PC-2, PC-3, PC-8 from configuration, review identity, PR template selection, AC3-4). ADR-005 is amended in scope (the template's CI manifest). Both take effect when the design PR merges.
- **`ai-lab-infrastructure` does not yet implement ADR-003 as amended** (`approver_id`, `design_repos`, `operator_paths`, qualified PC-3 references). It follows through the preset or adoption change PR (design section 14).
- The generic code is not linked after seeding. Whether this repository later adopts the template's copy is a later CR.
- ADR-010 adds `scripts/metrics`, the gate checks `output-budget` and `skill-layout`, an extended `size-budget`, and the values of `.template/budgets.toml`. They follow the template's implementation PRs.
- Stage 5 cannot start until ADR-009 and ADR-010 are also approved. This CR delivers ADR-008 only.

## 5. Open items

Listed in `docs/cr/CR-39/design.md` section 14: the preset, the core-plus-platform deny-list equality check, later adoption by this repository, the follow-through of this repository to ADR-003 as amended, the verification items, and the Stage 5 entry.

## 6. Acceptance criteria

The template criteria T-1 to T-22 (T-18 unused) and their evidence are in `docs/cr/CR-39/acceptance-criteria.md`. They are verified in the implementation PRs. T-16 is the Stage 5 exit criterion.
