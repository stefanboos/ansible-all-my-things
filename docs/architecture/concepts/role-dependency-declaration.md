<!-- SPDX-License-Identifier: MIT-0 -->

# Role dependency declaration: `meta/main.yml` vs. explicit ordering

Every role in this repository is authored with `dependencies: []` in
`meta/main.yml`; cross-role ordering is instead expressed explicitly, by
listing roles in the required order in a playbook's `roles:` list (or a
Molecule scenario's `converge.yml`). This document is the authoritative
source of truth for when that default should be overridden — i.e. when a
role's `meta/main.yml` MUST declare a real Ansible role dependency instead
of, or alongside, explicit ordering.

## The decision test

Ask one question about the role being authored or changed: **does one of
this role's own tasks unconditionally require an artefact (binary, file,
clone, config) that only another specific role provisions, with no guard,
such that the task would hard-fail for any consumer playbook that has not
already applied that other role first?**

- **Yes** → this is a hard, role-intrinsic dependency. Declare it in
  `meta/main.yml`'s `dependencies:` list. The requirement is a property of
  the role itself, not of any one playbook's ordering choice, and must not
  rely on every future consumer independently rediscovering it by reading
  the role's tasks.
- **No** → the relationship is a project-specific orchestration choice
  (e.g. "these two roles happen to run well together in this pipeline"),
  not a functional requirement. Express it via explicit ordering only —
  do not add a `meta/main.yml` dependency for convenience-only sequencing.

## Worked examples from this repo

**`claude_code` → `rtk` (hard dependency).** `claude_code/tasks/configure.yml`
runs:

```yaml
- name: Initialize rtk globally for each desktop user
  ansible.builtin.command:
    cmd: rtk init -g
```

An unconditional `command` task, no existence check, no `failed_when:
false`. If the `rtk` binary is absent, this task fails outright. A
playbook that lists `claude_code` without having first applied `rtk` will
break at apply time — this is exactly the "hard-fails for any consumer"
signal.

**`claude_code` → `ai_agent_workspace` (hard dependency).** The skills
symlink task in `claude_code/tasks/install-addons.yml` asserts `matched > 0`
after searching the `ai_agent_workspace` clone's `.claude/skills` directory
— a deliberate, fail-loud (Constitution Principle XII) guard for the same
kind of cross-role artefact dependency.

Both qualify under the decision test above. **Neither currently declares a
`meta/main.yml` dependency** — see "Current state and rationale" below for
why this is a known, accepted gap rather than an oversight to silently
work around.

**Counter-example: role co-location in a playbook.** Two roles that happen
to run in the same play purely because they configure the same host profile
(e.g. `nerd_font` and `google_chrome` both applying to the `desktop` group)
have no functional ordering requirement between them — no meta dependency
is warranted; the playbook's role list is the only ordering mechanism
needed, and none exists here today.

## Why declare the dependency even when explicit ordering already works

Ansible deduplicates role execution by role name **and** identical resolved
variables: if a role is both declared as a `meta/main.yml` dependency of
another role **and** listed explicitly in the same play, it runs exactly
once. Declaring a genuine hard dependency therefore costs nothing at
runtime when the calling playbook already lists the roles in the correct
order — it adds a second, independent enforcement path:

- **Meta dependency**: makes the role self-defending. Any current or future
  consumer — a new playbook, a Molecule `converge.yml`, a role composed
  into another project — gets the correct ordering automatically, without
  having to read the role's task internals to discover the requirement.
- **Explicit playbook/converge ordering**: keeps the full apply order
  human-auditable from one flat file, without recursively tracing
  `meta/main.yml` across every role in the chain — valuable in a
  security/audit-conscious repo (see Constitution Principle IX) where
  reviewers need to reconstruct exactly what runs, in what order, without
  spelunking through role metadata.

These are not competing choices for a genuine hard dependency: keep both.
The redundancy is intentional, not duplication to eliminate — it is the
same "log the accepted tradeoff" posture the Complexity Tracking practice
already applies to other deliberate small overlaps in this codebase (e.g.
per-role `git`/`curl` installation instead of a shared base-deps role).

## Current state and rationale

As of this writing, no role in this repository declares a non-empty
`meta/main.yml` `dependencies:` list — every role, including `claude_code`
(which has at least two hard dependencies per the worked examples above),
relies solely on explicit ordering. This is a known gap against the
decision test above, not evidence that the test doesn't apply here.
Closing it — adding `dependencies: [rtk, ai_agent_workspace]` to
`claude_code/meta/main.yml` — is tracked as follow-up work rather than
folded silently into whatever change prompted writing this document, so it
gets its own review and its own Molecule verification pass.

## Caveats when declaring a dependency

- **`meta/main.yml` dependencies are only honored via the `roles:` play
  keyword** (and transitively, when a dependency role itself has
  dependencies). They are **not** honored when a role is invoked via
  `ansible.builtin.include_role` (dynamic inclusion) — only reliably via
  `import_role` and the `roles:` keyword. If this repo ever moves toward
  conditional, task-based role inclusion for a given role, a `meta/main.yml`
  dependency on it would silently stop being enforced; re-express the
  ordering explicitly at that point.
- **Keep dependency chains shallow.** Ansible resolves nested dependencies
  breadth-first in a way that becomes hard to reason about more than one
  level deep. This repo's roles are flat and single-purpose (Constitution
  Principle II), so dependency chains here should stay one level (a role
  depends directly on the role(s) it hard-needs, not on a chain of
  dependencies-of-dependencies).
- **Variables inherited by a dependency role follow normal Ansible
  precedence** (play vars, host vars, etc. apply as usual); `meta/main.yml`
  can override specific vars per dependency if a role genuinely needs
  different parameters than its sibling explicit invocation, but prefer
  identical vars across both invocation paths so Ansible's dedup actually
  collapses them to a single run.
