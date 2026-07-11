# Plan: split beads role into beads_go + beads_viewer

## Summary

Rename `roles/beads` to `roles/beads_go` (the Go beads issue tracker `bd`
plus its source clone) and extract the beads viewer (`bv`,
`Dicklesworthstone/beads_viewer`) into a new standalone `roles/beads_viewer`.

The rename clears the `beads` name so a future `beads_rust` role — a Rust
reimplementation of the tracker — can be added alongside `beads_go` without
ambiguity about which implementation the bare `beads` role installs.

## Rationale

`bd` and `bv` are two independent upstream projects, published by different
authors (`gastownhall/beads` vs `Dicklesworthstone/beads_viewer`) with
independent release cadences. Modelling them as one role conflated two
release streams; splitting them makes each role a single-responsibility unit
(Constitution Principle II) and lets a future host install one without the
other.

The viewer stays a single `beads_viewer` role rather than a per-tracker
variant: `bv` is a read-only view over beads data and is tracker-agnostic,
so one viewer role serves both `beads_go` and a future `beads_rust`.

## Decisions

- Pin/checksum variable names match the role names: `beads_go_version` /
  `beads_go_sha256_amd64` / `beads_go_sha256_arm64` in `beads_go`, and
  `beads_viewer_version` / `beads_viewer_sha256_amd64` /
  `beads_viewer_sha256_arm64` in `beads_viewer`. The version-update wiring
  (`query-versions.yml`, `perform-updates.yml`) renames the matching regexes
  and its `fetched_*` / `current_*` scratch facts in lockstep, so no
  `bd`/`bv`-prefixed identifier remains. This keeps the `beads_go`/`beads_rust`
  namespace unambiguous when the Rust tracker is added.
- The architecture-to-filename map is renamed per role
  (`beads_go_platform_map`, `beads_viewer_platform_map`) so the two roles do
  not share a global variable name.
- `beads_viewer` declares `dependencies: []`: no `beads_viewer` task
  hard-fails without `beads_go` or `dolt_sql_server` having run first (its
  dependence on beads data and Dolt is a runtime relationship, not an
  install-time one). See
  `docs/architecture/concepts/role-dependency-declaration.md`.

## Complexity Tracking

| Principle | Deviation | Justification |
| --- | --- | --- |
| XI (DRY) | The two-key architecture map (`{x86_64: amd64, aarch64: arm64}`) is duplicated as `beads_go_platform_map` and `beads_viewer_platform_map`, one identical copy per role. | Role self-containment (Principle II) favours each standalone role owning its own `defaults/`. Sharing the map via `group_vars` or a common role would re-couple the two roles, defeating the split. The value is a trivial, stable 2-key dict; the duplication is preferred over the coupling. |

## Acceptance criteria

1. `molecule test` passes in `roles/beads_go` (asserts `bd` binary + source
   clone; no `bv`).
2. `molecule test` passes in `roles/beads_viewer` (asserts only `bv`).
3. `ansible-playbook playbooks/update-versions/query-versions.yml` runs and
   reports both the `bd` and `bv` rows with no undefined-variable or
   empty-regex errors.
4. `ansible-playbook --syntax-check playbooks/update-versions/perform-updates.yml`
   is clean.
5. No stale bare-`beads` role reference remains (role directory, playbook
   role lists, molecule image name, or version-update defaults path).
6. `roles/beads_viewer` contains no empty scaffolding files.
