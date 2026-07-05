# ADR-005: Extract Cross-Harness CLI Tools out of `claude_code`

Date: 2026-07-04
Status: Accepted
Deciders: Stefan (Product Owner)

## Context and Problem Statement

`roles/claude_code` bundled Claude-Code-specific mechanisms (binary
install, plugin marketplace/install, MCP server config, `settings.json`
hooks) together with several CLI tools and repos that are cross-harness
— usable from opencode, antigravity, codex, etc., not just Claude Code.
This was a standing Constitution Principle II (single-responsibility
role organisation) violation: the deciding test is whether a concern
has any life outside a Claude Code session, and `rtk`, `beads` (`bd`/
`bv`), the ai-agent-workspace skill library, and `specify-cli` all do.

Decision: **should the cross-harness pieces be split out of
`claude_code`, and if so, into how many roles?**

### Scope

The five cross-harness mechanisms living inside `claude_code`: the
`rtk` binary install and its `rtk init -g` config step, the `bd`/`bv`
binary installs plus the beads source-repo clone, the `omc` npm CLI
install plus the oh-my-claudecode source-repo clone, the ai-agent-
workspace clone, and the `specify-cli` pipx install. Claude-Code-only
mechanisms (the Claude binary itself, plugin marketplace/install, MCP
config, `settings.json` hook wiring) are explicitly out of scope and
stay in `claude_code` unchanged.

## Decision Drivers

- **Role-First Organisation (Constitution II).** Restoring single
  responsibility is the primary driver — not a speculative
  generalisation, but fixing an existing, confirmed violation.
- **Test Locally Before Cloud (Constitution III).** Each new role needs
  its own full Molecule lifecycle, isolated from the others.
- **Simplicity / YAGNI (Constitution IV).** Behavior-preserving
  extraction only — no new version pins, no new abstractions beyond
  what single-responsibility requires.
- **DRY (Constitution XI).** Splitting roles this way does introduce a
  small number of genuine duplication points (see Complexity Tracking
  below); each is weighed against the alternative of re-coupling roles
  the split is meant to separate.
- Cross-harness reuse (other harnesses being able to consume `rtk`,
  `beads`, etc. independently) is a **supporting** benefit, not the
  primary justification.

## Considered Options

1. **Five single-purpose roles** — `rtk`, `beads`, `omc_cli`,
   `specify_cli`, `ai_agent_workspace` — one per tool/concern,
   following the repo's existing one-role-per-tool convention
   (`roles/github_cli`, `roles/opencode`, `roles/dolt_sql_server`).
2. **One coarse `cross_harness_cli` grouping role.**
3. **Four roles, folding the ai-agent-workspace clone into
   `claude_code`.**

## Decision Outcome

Chosen: **Option 1 — five single-purpose roles**, matching the repo's
existing convention and restoring single-responsibility exactly.

### Why not the alternatives

- **Option 2** re-introduces the exact grab-bag violation this decision
  exists to fix. The ai-agent-workspace clone must stand alone anyway
  (see D1 below), so bundling the rest would still force `claude_code`'s
  own Molecule test to pull in the whole grab-bag regardless —
  invalidated by the same driver that justifies Option 1.
- **Option 3** leaves a plainly-generic skill-library clone trapped
  inside `claude_code`, perpetuating the violation the extraction
  targets. The one extra role composed into `claude_code`'s Molecule
  converge (Option 1's actual cost) is the correct modeling of a real
  data dependency — the skill symlink needs the clone to exist — not an
  avoidable one.

### Design decisions

- **D1 — Skill-symlink seam.** The ai-agent-workspace clone moved to
  its own `ai_agent_workspace` role; the `~/.claude/skills` symlink task
  (and its task-level "≥1 skill dir found" assert) stayed in
  `claude_code`. Consequence: `claude_code`'s Molecule `converge.yml`
  composes `ai_agent_workspace` alongside it, and the clone destination
  path is hardcoded identically in both roles (see Complexity Tracking).
- **D2 — `rtk init -g` placement revisited twice.** Originally moved
  into the `rtk` role (harness-agnostic global config, despite having
  lived in `claude_code/configure.yml` historically). **Superseded**:
  commit `a989ff0` folded `rtk init -g` and its `~/.claude` wiring back
  into `claude_code`, since it writes into `claude_code`'s own config
  directory and no other harness reads it — the binary install stays in
  `rtk`; the Claude-Code-targeting init runs in `claude_code`.
- **D3 — `specify-cli` version pin stays inline, never promoted to
  `defaults/main.yml`.** Deliberately keeps this pin out of Constitution
  II's mandatory version-update-mechanism trigger (which fires only for
  pins living in `defaults/main.yml`). The same treatment was later
  applied to `nodejs`'s nvm pin (see "Later additions" below) for the
  identical reason.
- **D4 — role name `omc_cli`.** Chosen to stay distinct from "the
  oh-my-claudecode Claude Code **plugin**" (marketplace-installed,
  stays in `claude_code`). **Superseded**: commit `689a316` folded
  `omc_cli` back into `claude_code` entirely — the deciding test
  (does this concern have any life outside a Claude Code session?)
  came back "no" for the `omc` CLI itself, unlike `rtk` or `beads`.
- **D5 — each new role installs its own system deps** (`git`/`pipx`/
  `curl`) rather than depending on `claude_code` having installed them
  first — order-independence and self-containment over a shared
  base-deps role (see Complexity Tracking).

### Complexity Tracking

Re-derived against the current codebase, not copied from this ADR's
predecessor draft — some entries below have already resolved since the
original extraction:

| Duplication | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| `git` apt-install task duplicated across 4 current consumers — `claude_code` (plugin-marketplace clone), `beads` (source-repo clone), `specify_cli` (VCS-style pipx install), `ai_agent_workspace` (clone) | Order-independence + role self-containment (D5). Each is a one-line idempotent apt task. | A shared "base system deps" role — over-abstraction (Principle IV/YAGNI) for a single apt line; re-couples roles this split separates. |
| Clone path `~/Documents/Cline/ai-agent-workspace` hardcoded in both `ai_agent_workspace` (clone destination) and `claude_code`'s symlink task | D1 split: generic clone and Claude-specific symlink genuinely live in different roles; the path is a stable, already-hardcoded convention. | Cross-role variable plumbing — heavier than a convention string for a stable path; would re-couple the two roles. |
| "Ensure `~/.claude` exists" duplicated in `claude_code` (precondition for both `rtk init -g` and `configure.yml`'s settings.json wiring, now both in the same role after D2's reversal) | No longer a cross-role duplication as of `a989ff0` — resolved by D2's supersession. | N/A — no longer applicable. |
| `curl` apt-install task, originally logged as duplicated into `rtk` and `beads` as a genuine production dep | **Resolved, not by design intent**: both roles later replaced their vendor-installer-script approach with direct-download + Ansible-native checksum verification (`get_url`/`uri`, pure Python — see `docs/architecture/concepts/checksum-verification-pattern.md`), which needs no `curl` at all. Neither role installs `curl` today. `claude_code` and the later `nodejs` role each still have their own independent, non-duplicated need for it. | N/A — the duplication this row originally tracked no longer exists. |

## Later additions to this extraction

Beyond the original five-role split, two further findings from the PR
review of this work (tracked under bd issue `jvs4.1`) extended the same
single-responsibility logic:

- **`nodejs` role** (bd issue `jvs4.1.6`): `playbooks/setup-nodejs.yml`
  was itself a Principle II violation (raw tasks in a playbook) that
  pre-dated this extraction. Its nvm/Node.js/global-npm-package
  provisioning was moved into a new `nodejs` role, composed as a hard
  `meta/main.yml` dependency of `claude_code` (whose `omc` CLI install
  unconditionally sources `~/.nvm/nvm.sh`). Its nvm version pin follows
  D3's inline-literal precedent for the same Principle II reason.
- **Checksum verification** (bd issues `jvs4.1.1`–`jvs4.1.3`): `rtk` and
  `beads` (`bd`/`bv`) originally downloaded and shell-ran a vendor
  `install.sh` blind. Both were replaced with stat-gated direct-download
  plus Ansible-native checksum verification. This is a **deliberate**
  choice of reproducibility (Constitution I, idempotency) over
  auto-patch-freshness: these installs are frozen at whatever release
  was present on first install and do not silently auto-upgrade when a
  newer upstream release ships. The only intended path to ever refresh
  them is the version-update-mechanism backlog tracked under bd issue
  `ansible-all-my-things-m0av` (which should list `nodejs`/nvm alongside
  its existing `rtk`/`beads`/`specify_cli` scope) — this is a load-bearing
  follow-up, not an aspiration, since no other mechanism in this repo
  refreshes a frozen-at-first-install tool.

### Consequences

- Five new roles (`rtk`, `beads`, `specify_cli`, `ai_agent_workspace`,
  and later `nodejs`) each with their own Molecule scenario;
  `claude_code`'s own Molecule composes `rtk`, `nodejs`, and
  `ai_agent_workspace` alongside it.
- Three logged Principle XI (DRY) exceptions remain live (`git`
  duplication across 4 roles, the ai-agent-workspace clone-path string,
  and — as of the later checksum-verification work — none for `curl`,
  since that duplication resolved itself as a side effect).
- Production plays (`playbooks/configure-profile-roles.yml`) list the
  new roles in explicit dependency order ahead of `claude_code`.
- `rtk init -g` and the `omc_cli` role both moved back into
  `claude_code` after this ADR's initial decision (D2, D4) — recorded
  here as the decision trail, not erased, since an ADR documents
  decisions as made, including ones later reversed.
- `rtk`/`beads`/`nodejs` installs are frozen-at-first-install by design;
  refreshing them depends on the still-open version-update-mechanism
  follow-up (`m0av`).

### Follow-ups (tracked, not inline)

- Wire `nodejs`/nvm, `rtk`, `beads`, and `specify_cli` into the
  version-update mechanism (`playbooks/update-versions/`) if/when
  version refresh is desired — bd issue `ansible-all-my-things-m0av`.
