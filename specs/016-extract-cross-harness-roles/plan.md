# Implementation Plan: Extract Cross-Harness CLI Tools out of `claude_code`

**Branch**: `016-extract-cross-harness-roles` | **Date**: 2026-07-04

**Spec**: none (engineering refactor, not a user-facing feature — produced
via `/ralplan` consensus, not `/speckit-specify`)

**Input**: `/ralplan --interactive` consensus loop (Planner → Architect →
Critic, 3 iterations, final verdict **APPROVE**). No `spec.md` exists for
this feature; this `plan.md` is the durable record of the consensus plan
and its ADR, per Constitution Principle X (self-contained durable
artefacts).

**Status**: implemented. All five new roles (`rtk`, `beads`, `omc_cli`,
`specify_cli`, `ai_agent_workspace`) and the trimmed `claude_code` role pass
`molecule test` full lifecycle standalone; the verify-assertion count
reconciles to 21 across all six `verify.yml` files; both plays in
`playbooks/configure-profile-roles.yml` are updated. See "Implementation
findings" below for two transitive-dependency gaps discovered and fixed
during implementation.

## Summary

`roles/claude_code` currently bundles Claude-Code-specific mechanisms (binary
install, plugin marketplace/install, MCP server config, `settings.json`
hooks) together with several CLI tools/repos that are cross-harness —
usable from opencode, antigravity, codex, etc., not just Claude Code. This
is a standing Constitution Principle II (single-responsibility role
organisation) violation. This plan extracts the cross-harness pieces into
five new single-purpose roles — `rtk`, `beads`, `omc_cli`, `specify_cli`,
`ai_agent_workspace` — following the repo's existing one-role-per-tool
convention (`roles/github_cli`, `roles/opencode`, `roles/dolt_sql_server`).
Claude-Code-specific mechanisms stay in `claude_code` unchanged. Done when
`molecule test` passes the full `create→prepare→converge→idempotence→verify→destroy`
lifecycle for each of the five new roles and for the trimmed `claude_code`.

Cross-harness reuse is a supporting benefit of this change, not the primary
justification — the primary justification is restoring Principle II
conformance for code that has no actual Claude-Code coupling.

## Technical Context

**Language/Version**: Ansible YAML (roles/tasks), consistent with the
existing single-tool role convention in this repo.

**Primary Dependencies**: None new. Reuses `scripts/new-role.sh` scaffolding,
the `role-template/` skeleton, and the `molecule-testing` skill's scenario
contract (Dockerfile/molecule.yml/prepare.yml/converge.yml/verify.yml).

**Storage**: N/A (Ansible roles + playbooks only).

**Testing**: Molecule, Podman driver, full
`create→prepare→converge→idempotence→verify→destroy` lifecycle per role
(Constitution Principle II/III). This is the acceptance gate for the whole
feature — see Acceptance Criteria below.

**Target Platform**: Linux desktop hosts (`basic`/`desktop` profiles in
`playbooks/configure-profile-roles.yml`), validated locally via Podman
containers before any cloud apply (Constitution Principle III).

**Project Type**: Ansible role library (single repo, no frontend/backend
split).

**Performance Goals**: N/A.

**Constraints**: Behavior-preserving extraction only (Constitution Principle
IV/YAGNI) — task bodies and verify assertions move verbatim; no version
pins promoted to `defaults`; no new version-tracking wiring added.

**Scale/Scope**: 5 new roles, 1 trimmed role, 2 playbook plays updated, ~21
Molecule verify assertions redistributed.

## Constitution Check

*GATE: must pass before implementation begins.*

- **Principle II (Role-First Organisation)**: PASS — this is the change that
  restores conformance. Each new role has single responsibility; each gets a
  full Molecule scenario (Principle II Molecule requirement).
- **Principle III (Test Locally Before Cloud)**: PASS — `molecule test` per
  role is the acceptance gate; no cloud apply until local validation passes.
- **Principle IV (Simplicity/YAGNI)**: PASS, with one accepted exception (see
  Complexity Tracking) — system-dep duplication (`git`/`curl`) across roles
  is a deliberate, logged trade-off in favor of order-independence, not
  speculative generalisation.
- **Principle XI (DRY)**: Two logged exceptions — see Complexity Tracking.
  Both are idempotent config duplication, not behavioral-logic duplication.
- **Principle XII (Fail Loud)**: PASS — existing `assert`/`fail` guards
  (e.g. ai-agent-workspace "≥1 skill dir found") move verbatim, unweakened.
- **Principle VIII (No Untracked Technical Debt)**: PASS — the pre-existing
  version-tracking gap for rtk/beads/omc_cli/specify_cli is explicitly
  **not** fixed inline (out of scope for a pure extraction) and is instead
  logged as a required tracked follow-up issue before this feature's cover
  issue closes.

No other Constitution principle is implicated by this change.

## Project Structure

### Documentation (this feature)

```text
specs/016-extract-cross-harness-roles/
└── plan.md              # This file — the full consensus plan + ADR
```

No `spec.md`/`research.md`/`data-model.md`/`quickstart.md`/`tasks.md` — this
is a direct engineering refactor scoped and approved via `/ralplan`, not a
user-facing feature run through the full `/speckit-specify` → `/speckit-tasks`
pipeline. `plan.md` alone is the durable blueprint (Constitution Principle X).

### Source code (repository root)

```text
roles/
├── claude_code/            # TRIMMED — Claude-Code-only mechanisms remain
│   ├── tasks/
│   │   ├── install-claude-code.yml   # unchanged
│   │   ├── install-deps.yml          # trimmed: git+jq only (pipx removed)
│   │   ├── install-addons.yml        # trimmed: plugins/MCP/symlink only
│   │   └── configure.yml             # trimmed: rtk init -g removed
│   └── molecule/default/
│       ├── prepare.yml               # trimmed: nvm/curl/node fixture removed
│       ├── converge.yml              # composes ai_agent_workspace + claude_code
│       └── verify.yml                # trimmed to 12 retained assert blocks
├── rtk/                     # NEW — binary install + `rtk init -g`
├── beads/                   # NEW — bd/bv install + beads source clone
├── omc_cli/                 # NEW — omc npm CLI + oh-my-claudecode source clone
├── specify_cli/             # NEW — spec-kit pipx install
└── ai_agent_workspace/      # NEW — ai-agent-workspace clone only

playbooks/
└── configure-profile-roles.yml   # both plays gain the 5 new roles, before claude_code
```

**Structure Decision**: Option A (five single-purpose roles) — see Viable
Options below for alternatives considered and rejected.

## Viable Options

### Option A — Five single-purpose roles (CHOSEN)

New roles: `rtk`, `beads`, `omc_cli`, `specify_cli`, `ai_agent_workspace`.

- Pros: restores Principle II single-responsibility exactly; matches
  existing convention (`github_cli`, `opencode`, `dolt_sql_server`); small,
  fast, independent Molecule scenarios; correct home for any future
  version-tracking.
- Cons: five new Molecule scenarios to author; five insertions × two plays;
  `omc_cli`/`beads`/`rtk` each carry more than one task.

### Option B — Coarse grouping into one `cross_harness_cli` role (REJECTED)

- Re-introduces the exact Principle II violation being fixed (a grab-bag
  role). `ai_agent_workspace` must stand alone anyway (composed into
  `claude_code`'s own Molecule converge for the skill-symlink dependency),
  so bundling the rest would force `claude_code`'s test to pull in the
  whole grab-bag regardless. Invalidated by the same driver that justifies
  Option A.

### Option C — Four roles, fold ai_agent_workspace clone into claude_code (REJECTED)

- Leaves a plainly-generic skill-library clone trapped inside
  `claude_code`, perpetuating the Principle II violation and contradicting
  the extraction's own goal. The composition cost avoided (one extra role
  in `claude_code`'s Molecule converge) is small and is the correct
  modeling of a real data dependency (the symlink needs the clone to
  exist).

## Extraction map (what moves, what stays)

| Source (in `claude_code`) | Destination | Notes |
|---|---|---|
| rtk `install.sh` download+run+cleanup + `rtk init -g` per desktop user | **`rtk`** role | Verbatim, incl. `rtk_pre_install_stat` guard. No version pin (always installs `master`). `rtk init -g` moves here too (was in `configure.yml`) — it is global rtk config, harness-agnostic, not a Claude-Code-specific concern. Role gets its own `prepare.yml` (testuser, since init runs per-user). Needs its own `curl` apt task (rtk's installer hard-requires curl, no wget fallback) — no `git` task: verified empirically that `install.sh` never invokes `git`, and `rtk init -g` is a plain command with no VCS operation. |
| beads `bd` install, `bv` install, beads source-repo clone | **`beads`** role | Verbatim, `creates:`/git guards preserved. Sibling to existing `dolt_sql_server` (beads is Dolt-backed). Needs its own `git` + `curl` apt tasks (bd/bv installers need curl-or-wget; bare Ubuntu has neither). |
| oh-my-claudecode **source repo** clone + `omc` CLI `npm install -g oh-my-claude-sisyphus` | **`omc_cli`** role | Distinct from the "oh-my-claudecode" Claude-Code **plugin** (marketplace-installed via `claude plugin install`), which stays in `claude_code` unchanged. Needs its own `git` apt task. `curl` stays test-only in this role's own Molecule `prepare.yml` (feeds the nvm/Node-LTS fixture copied verbatim from `claude_code`'s current `prepare.yml`) — production nvm/node is not provisioned by any role in `configure-profile-roles.yml`, it comes from the separate existing `playbooks/setup-nodejs.yml`; this is a known, documented limitation (omc_cli's green Molecule run does not validate that cross-playbook production dependency), noted in `omc_cli/DESIGN.md`. |
| ai-agent-workspace **clone only** | **`ai_agent_workspace`** role | Needs its own `git` apt task. The symlink task (ensure `~/.claude/skills` exists, find skill dirs in the clone, assert ≥1 found, symlink each in) does **not** move — stays in `claude_code`, referencing the clone's well-known path `~/Documents/Cline/ai-agent-workspace` as a hardcoded literal (not a shared variable — sharing one would re-couple the two roles this split separates). Consequence: `claude_code`'s own Molecule `converge.yml` composes `ai_agent_workspace` + `claude_code` together, since the retained symlink verification needs a real clone to link against. |
| specify-cli `pipx install git+https://github.com/github/spec-kit.git@v0.8.18` | **`specify_cli`** role | Version `v0.8.18` stays hardcoded **inline** in the shell command — deliberately **not** promoted to `defaults/main.yml`, because doing so would newly trigger Constitution Principle II's mandatory version-update-mechanism wiring (fetch task + `query-versions.yml` + `perform-updates.yml` registration), out of scope for a pure extraction (none of rtk/beads/omc_cli/specify_cli are version-tracked today — confirmed by grep, a pre-existing gap). Needs its own `pipx` **and** `git` apt tasks — the `git+https` VCS-style pip install shells out to system `git` to clone. |
| `install-claude-code.yml` (whole) | stays | Claude binary fetch/verify/install — Claude-Code core, untouched. |
| Plugin marketplace/install tasks (oh-my-claudecode plugin, caveman plugin, claude-plugins-official marketplace, context7 plugin) | stays | Claude-Code-only mechanism (`claude plugin marketplace add/list`, `claude plugin install`). Confirmed real consumer of system `git` (`claude plugin marketplace add <url>` performs an actual git clone under the hood — verified via a live `.git/` directory under `~/.claude/plugins/marketplaces/omc/`), so `git` stays installed in `claude_code`, not removed. |
| Exa MCP server config (`claude mcp add`) | stays | Claude-Code-only mechanism. |
| `configure.yml` minus `rtk init -g` (jq `settings.json` merge incl. `rtk hook claude`/bd-guard hook wiring as JSON **text** only, `CLAUDE_PLUGIN_ROOT` env, `setup-omc-prompt.md` copy) | stays | The `rtk hook claude` and bd-guard hook checks reference command strings only — they never execute the rtk/bd binaries, so no Molecule composition is needed for them. |
| `jq` apt install | stays | Sole remaining consumer is `configure.yml`'s settings.json merge. |
| `pipx` apt install | leaves entirely → `specify_cli` | specify was the only pipx consumer in `claude_code`. |

### System-deps matrix (per role, after extraction)

| Role | git | pipx | curl (production task) | curl (test-only, prepare.yml) |
|---|---|---|---|---|
| `claude_code` (trimmed) | retained (plugin-marketplace clone) | removed | yes (its own official installer script hard-requires curl/wget; discovered during standalone Molecule isolation — see Pre-mortem scenario 4 addendum below) | no (nvm fixture removed) |
| `rtk` | — (no git consumer) | — | yes (installer hard-requires) | — |
| `beads` | yes | — | yes (bd/bv installers) | — |
| `omc_cli` | yes | — | no | yes (nvm install script, test-only) |
| `specify_cli` | yes (VCS pip install) | yes | — | — |
| `ai_agent_workspace` | yes | — | — | — |

Ordering rule, every role: the system-dep apt task(s) must be textually
ordered **before** the consuming task (git clone / pipx install / install.sh
download) in `tasks/main.yml` — never appended at the end.

`tar`/`gzip`/`ca-certificates` are intentionally **not** made into explicit
per-role tasks: `tar`/`gzip` are Ubuntu `Priority: required` base-OS tools
already present in the `ubuntu:24.04` Molecule image (the current combined
pipeline already extracts bd/bv/rtk via `tar -xzf` with no `tar` apt task
anywhere, proving this); `ca-certificates` is a Dockerfile-level concern
per the `molecule-testing` skill contract (needed by any role whose
Molecule scenario uses `get_url`/git over HTTPS), not a role-task concern.

## verify.yml assertion-migration map

`roles/claude_code/molecule/default/verify.yml` contains exactly **21**
`ansible.builtin.assert` blocks. Task bodies moving is not sufficient on its
own — each assertion block must be migrated to its destination role's own
`verify.yml`, or the trimmed `claude_code`'s Molecule test fails on the
first `stat` assertion for a tool that's no longer installed there.

| Count | Assertion blocks | Destination |
|---|---|---|
| 2 | rtk-binary-stat; `rtk init -g` coverage | `roles/rtk/molecule/default/verify.yml` |
| 3 | `bd`-stat; `bv`-stat; beads-clone-stat | `roles/beads/molecule/default/verify.yml` |
| 2 | omc-source-clone-stat; `omc --version` (via nvm) | `roles/omc_cli/molecule/default/verify.yml` |
| 1 | `specify --version` | `roles/specify_cli/molecule/default/verify.yml` |
| 1 | ai-agent-workspace clone-stat (clone only — no skills-dir-content assertion; that check is a **task-level** `assert` inside `install-addons.yml`, stays with the symlink task in `claude_code`) | `roles/ai_agent_workspace/molecule/default/verify.yml` |
| 12 | claude-binary+checksum+PATH; `jq`-stat; omc-plugin-cache; context7-plugin-cache; settings.json key assertions (env vars + teammateMode); rtk-hook-string-check; bd-guard-string-check; `CLAUDE_PLUGIN_ROOT`-bashrc-grep; exa-MCP-registered; context7-MCP-registered; symlink-present-assert; symlink-resolves-assert | **remains in** `roles/claude_code/molecule/default/verify.yml` |

`2 + 3 + 2 + 1 + 1 + 12 = 21`.

**Coverage-preservation control**: the reconciliation baseline is **21**.
At execution time, re-derive this count by grepping the actual pre-refactor
file — do not trust this table blindly, it is a checklist, not a substitute
for counting the real file. After the refactor, the sum of `assert` blocks
across the five new `verify.yml` files plus the trimmed
`claude_code/verify.yml` must equal 21. This is a hard gate at the end of
implementation, not a suggestion.

## Design decisions

- **D1 — Skill-symlink seam.** Clone moves to `ai_agent_workspace`; the
  `~/.claude/skills` symlink task (and its task-level "≥1 skill dir found"
  assert) stays in `claude_code`. Consequence: `claude_code`'s Molecule
  `converge.yml` composes `ai_agent_workspace` + `claude_code`. The
  hardcoded clone path now lives in two files — logged DRY exception.
- **D2 — `rtk init -g` moves into the `rtk` role.** It is global rtk
  configuration, harness-agnostic, not Claude-Code-specific — despite
  having lived in `claude_code/configure.yml` historically. Consequence:
  `claude_code`'s Molecule composition need reduces to `{ai_agent_workspace}`
  only (rtk is referenced only as JSON text in `settings.json`, never
  executed, by claude_code's retained tasks).
- **D3 — specify version pin stays inline; version-tracking not wired.**
  Explicit non-goal for this extraction (YAGNI/Principle IV) — tracked as a
  separate follow-up issue, not inline work.
- **D4 — role name `omc_cli`.** Chosen to stay distinct from "the
  oh-my-claudecode plugin" (which stays in `claude_code` unchanged).
- **D5 — each new role installs its own system deps** (`git`/`pipx`/`curl`)
  rather than depending on `claude_code` having installed them first —
  order-independence and self-containment. Logged as a Principle XI (DRY)
  exception (see Complexity Tracking) rather than centralized into a shared
  base-deps role, which would itself be a Principle IV (YAGNI)
  over-abstraction for one-line idempotent apt tasks and would re-introduce
  the exact cross-role coupling this extraction removes.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| `git` apt-install task duplicated across 5 consumers — `claude_code` (retained), `beads`, `omc_cli`, `ai_agent_workspace`, `specify_cli` | Order-independence + role self-containment (D5). Each is a one-line idempotent apt task (config, not logic). | A shared "base system deps" role — over-abstraction (Principle IV/YAGNI) for a single apt line; re-couples roles the refactor is decoupling. |
| Clone path `~/Documents/Cline/ai-agent-workspace` hardcoded in both `ai_agent_workspace` (clone destination) and `claude_code`'s symlink task | D1 split: generic clone and Claude-specific symlink genuinely live in different roles; the path is a stable, already-hardcoded-elsewhere convention. | Cross-role variable plumbing — heavier than a convention string for a stable path (Principle IV/YAGNI); would re-couple the two roles. |
| "Ensure `~/.claude` directory exists" task duplicated in `rtk` (precondition for `rtk init -g`, discovered during implementation — see "Implementation findings") and `claude_code`'s `configure.yml` | Both roles perform an operation under `~/.claude` and can no longer rely on task-ordering across roles to guarantee the directory exists first (D5: self-contained, order-independent roles). One-line idempotent `file: state=directory` task. | A shared "ensure `~/.claude`" role/task-file — over-abstraction (Principle IV/YAGNI) for a single directory-creation task; re-couples the two roles. |
| `curl` apt-install task duplicated into `rtk` and `beads` (2 consumers) as a genuine production system-dep | rtk's installer hard-requires curl (no wget fallback, verified against the real script); beads/bv installers need curl-or-wget and a bare `ubuntu:24.04` image has neither. Distinct from `omc_cli`'s test-only curl (prepare.yml scope, not a production task — see `omc_cli/DESIGN.md`). | Shared base-deps role — same rejection as the `git` row above. |

## Tracked follow-ups (Principle VIII — separate issues, not inline)

- Wire `rtk`, `beads`, `omc_cli` (`oh-my-claude-sisyphus`), and
  `specify_cli` into the version-update mechanism
  (`playbooks/update-versions/`) if/when version pinning is desired.
  Pre-existing gap — none of the four are version-tracked today. File
  before closing this feature's cover issue.
- Re-evaluate whether `rtk init -g` genuinely belongs in the `rtk` role
  (D2) vs. being a Claude-consumer concern, if a non-Claude harness later
  needs rtk init and the current placement proves wrong in practice.

## Pre-mortem (4 scenarios)

1. **`omc_cli` Molecule fails on nvm/node.** Fixture copy misses an
   env-sourcing line or its `curl` install, so `command -v omc` returns
   non-zero mid-converge, or `npm install -g` reports `changed` on the
   idempotence run. Mitigation: copy the fixture verbatim (incl. `curl`),
   keep the `omc` detection guard exactly as original, run idempotence
   explicitly; apply the `fix-problem` skill one issue at a time if it
   flaps.
2. **`claude_code` verify breaks because the symlink has no clone.** If
   `ai_agent_workspace` is omitted from `claude_code`'s converge, the
   retained "≥1 skill dir found" assert (and the symlink-resolves verify
   blocks) fail loudly. Mitigation: the converge composition is explicit in
   this plan; the full verification gate (below) catches any regression.
3. **Production play-ordering regression.** New roles land after
   `claude_code` in `playbooks/configure-profile-roles.yml`, or one play is
   missed, leaving the symlink task pointing at a missing clone on real
   hosts. Mitigation: both plays must list all five new roles before
   `claude_code`, `ai_agent_workspace` specifically first; the composed
   Molecule scenario is the proxy test for this ordering contract.
4. **A verify.yml assertion is silently dropped during migration, or a
   transitive system dep is missed** — both are "invisible until per-role
   isolation exposes them" failure classes (this is exactly how the
   `git`-for-specify_cli and `curl`-for-rtk/beads gaps were found during
   this plan's own review). Mitigation: the 21-block migration map plus the
   count-reconciliation gate catches (a); the per-role system-deps matrix
   plus each role's own standalone `molecule test` (which converges in a
   clean container with only its own `prepare.yml`) catches (b).

### Implementation findings (scenario 4 materialized twice more)

Standalone Molecule isolation surfaced two further transitive-dependency gaps
this plan's own review did not catch, confirming pre-mortem scenario 4 as the
dominant risk class for this extraction:

- **`claude_code` needs its own `curl`.** `install-claude-code.yml`'s official
  installer script hard-requires `curl`/`wget` on the target host. Neither the
  original combined role nor this plan's system-deps matrix ever declared
  `curl` as a `claude_code` dependency — it was silently satisfied in the old
  combined Molecule scenario by the nvm/curl test fixture running before
  converge (a coincidence of task ordering, not a declared dependency).
  Isolating `claude_code`'s own scenario (removing that fixture per this
  plan) exposed the gap immediately. Fixed by adding `curl` to
  `claude_code/tasks/install-deps.yml`'s own apt task (D5-consistent:
  self-contained, not reliant on `rtk`/`beads` happening to run earlier in
  the production play).
- **`rtk` needs `~/.claude` to exist before `rtk init -g`.** In the original
  combined role, `configure.yml`'s "Ensure ~/.claude directory exists" task
  ran before "Initialize rtk globally", satisfying this implicitly. Once
  `rtk init -g` moved to the standalone `rtk` role (D2), that ordering
  guarantee no longer existed. Fixed by adding an "Ensure ~/.claude directory
  exists" task to `roles/rtk/tasks/main.yml` immediately before `rtk init -g`.

## Acceptance Criteria (the "done" gate)

1. `molecule test` passes the full `create→prepare→converge→idempotence→verify→destroy`
   lifecycle for **each** of `rtk`, `beads`, `omc_cli`, `specify_cli`,
   `ai_agent_workspace`.
2. `molecule test` for the trimmed `claude_code` passes full lifecycle;
   converge composes `ai_agent_workspace` + `claude_code`; verify retains
   only the 12 "remains" blocks and still asserts the skill symlink.
3. Verify-assertion reconciliation passes: every one of the 21 original
   blocks is present in exactly one destination `verify.yml`; the
   block-count totals reconcile against the executor-derived baseline.
4. System deps correct per the matrix above: `git` in 5 places; `pipx` in
   `specify_cli` only (removed from `claude_code`); `jq` in `claude_code`;
   `curl` as a real `tasks/main.yml` task in `rtk` + `beads`, test-only in
   `omc_cli`'s `prepare.yml`.
5. `claude_code`'s `prepare.yml` contains only testuser + `Documents/Cline`
   dir creation; the nvm/node (+ its curl) fixture lives solely in
   `omc_cli`'s `prepare.yml`.
6. Every new role scaffolded via `scripts/new-role.sh` (never hand-created);
   no empty/`.gitkeep`/SPDX-only files (Principle XIII); each new role has
   a `README.md`; `DESIGN.md` present only for `omc_cli` and
   `ai_agent_workspace` (the two roles with a non-obvious decision to
   record).
7. Both plays in `playbooks/configure-profile-roles.yml` list all five new
   roles before `claude_code`; `ai_agent_workspace` specifically precedes
   `claude_code`.
8. specify version pin remains inline; no new version-update wiring added
   (D3 non-goal honored).
9. No moved task body or assertion is altered in behavior (diff shows
   relocation, not rewrite); all `creates:`/`stat`/`assert` guards
   preserved; every system-dep apt task ordered before its consumer.
10. All touched Markdown is lint-clean (`format-markdown`) and
    documentation-reviewed (`review-documentation-here`).

## ADR

- **Decision**: Extract five single-purpose roles (`rtk`, `beads`,
  `omc_cli`, `specify_cli`, `ai_agent_workspace`) from `claude_code`; move
  `rtk init -g` with the rtk binary (D2); keep the ai-agent-workspace
  skill-symlink in `claude_code` (D1); retain `git` in `claude_code` and
  duplicate it into four new roles (D5); add `curl` as a real production
  dependency in `rtk` and `beads`.
- **Drivers**: Constitution Principle II single-responsibility conformance
  (primary, non-speculative — the bundled code is a standing violation);
  test-isolation/verify-migration/transitive-system-dep risk; skill-symlink
  entanglement. Cross-harness reuse is a supporting, not primary, driver.
- **Alternatives considered**: Option B (one coarse grab-bag role) —
  rejected, re-creates the Principle II violation. Option C (fold
  `ai_agent_workspace` into `claude_code`) — rejected, traps a generic
  clone and contradicts the extraction's own goal.
- **Why chosen**: matches the repo's existing single-tool-role convention;
  restores single-responsibility; the Molecule/verify migration cost is
  mechanical and fully gated by the acceptance criteria above.
- **Consequences**: five new Molecule scenarios and `verify.yml` files to
  maintain; `claude_code`'s Molecule composes `ai_agent_workspace`; three
  logged Principle XI (DRY) exceptions (git duplication, curl duplication,
  clone-path duplication); production plays gain five ordered entries in
  two places; `rtk init -g` ownership shifts to the `rtk` role.
- **Follow-ups (tracked, not inline)**: version-update wiring for the four
  previously-untracked tools (D3); revisit `rtk init -g` placement (D2) if
  a non-Claude harness later needs rtk init.

## Consensus record

Produced via `/ralplan --interactive`. Planner → Architect → Critic loop,
3 revision iterations (of a 5-iteration cap):

- **Iteration 1**: Architect found a composition hole (`rtk init -g` needed
  `rtk` composed into `claude_code`'s converge) and ruled D2 (move `rtk
  init -g` into the `rtk` role), which resolved it. Critic returned
  **ITERATE**: unspecified `verify.yml` migration (critical), `git`
  wrongly assumed removable from `claude_code` (major — empirically
  disproven via a live `.git/` check under
  `~/.claude/plugins/marketplaces/omc/`), orphaned nvm/node
  `prepare.yml` fixture (major).
- **Iteration 2**: All three Critic findings addressed. Architect
  re-verified against actual source (line numbers, real assert-block
  count) and confirmed all three resolved, but found two new gaps exposed
  by the more-explicit rev-2 system-deps matrix: `specify_cli` needs `git`
  too (VCS-style `pipx install git+https://...`), and `rtk`/`beads` need
  `curl` (verified empirically by fetching and grepping the real install
  scripts from their GitHub URLs — rtk hard-requires curl with no wget
  fallback; beads/beads_viewer fall back to wget but a bare `ubuntu:24.04`
  image has neither).
  Architect also found the `verify.yml` migration-map prose was
  inaccurate (invented a non-existent "caveman plugin cache" assertion and
  a non-existent "non-empty skills dir" `verify.yml` assertion — the real
  file has exactly 21 `assert` blocks, confirmed by direct count).
- **Iteration 3**: All fixes applied and independently re-verified.
  Critic's final verdict: **APPROVE**, with one non-blocking
  documentation-only caveat (the plan doesn't explicitly state why
  `tar`/`gzip` were excluded from the system-deps matrix — they are
  Ubuntu `Priority: required` base-OS tools already present in the
  Molecule test image, confirmed by the fact that the current combined
  pipeline already extracts bd/bv/rtk via `tar -xzf` with no `tar` apt
  task anywhere).
