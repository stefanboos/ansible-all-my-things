# Concept: Claude Code Binary Integrity Verification

## Feature Name

Pre-installation checksum verification for the Claude Code binary.

## Description

The Claude Code version and its per-platform SHA256 checksums are pinned in
`roles/claude_code/defaults/main.yml`, refreshed only by
`playbooks/update-versions/perform-updates.yml`. The playbook asserts that the
host architecture is supported, then downloads the binary directly from the
manifest-derived URL for the pinned version, verifying its checksum against
the pinned value via `get_url`'s native `checksum:` parameter BEFORE the file
is placed on disk. A `stat` gate on the already-installed binary short-
circuits the whole download-and-verify block on re-converge, so an
already-installed binary is never re-verified or re-downloaded.

## User Value

A compromised or corrupted binary could execute arbitrary code with the
user's privileges. By verifying the checksum against the pinned value before
the binary is ever placed or executed, this feature detects tampered or
corrupted binaries before they are used. This reduces the supply-chain risk
of downloading and executing software from the internet, which was the
highest-severity finding from the original code review.

## Design Rationale

### Never execute the binary before verifying it

Version discovery via `claude --version` was rejected: running the binary
before its integrity is confirmed defeats the purpose of the check. The
pinned `claude_code_version` and per-platform checksums come from
`defaults/main.yml`, not from querying the binary itself.

### Verify before placement, not after

The current design does not use `install.sh`. Instead, `get_url` fetches the
binary directly from the version-pinned manifest URL and verifies its checksum
via its native `checksum:` parameter *before* the file is ever written to its
final path. There is no window where an unverified artefact executes.

### Static pin instead of always-latest

Per decision `ansible-all-my-things-jvs4.1.14`, the version and checksums are
static values in `defaults/main.yml` (the `flutter` role's pattern),
refreshed only by `playbooks/update-versions/perform-updates.yml` -- not
resolved dynamically at every converge. This matches the shared
frozen-at-first-install idempotency model used by `rtk`, `beads`, and
`nodejs`: a `stat` gate on the installed binary path is the sole idempotency
condition, and its *source* is now the static pin rather than a live "latest"
API resolution.

### Flat task list instead of block/rescue

An early implementation used Ansible's `block`/`rescue` structure. This was
replaced with a flat task list. A `rescue` block wraps all failures from its
`block` into a single generic error, obscuring which task actually failed. A
flat list surfaces the name of the failing task directly, making diagnosis
faster.

## Out of Scope

- **Code signature verification**: Anthropic signs macOS and Windows
  binaries, but not Linux binaries. Signature verification is not applicable
  to the current Linux-only target.
- **musl-based distributions** (e.g. Alpine): the platform mapping covers
  glibc-based `linux-arm64` and `linux-x64` only. musl support can be added
  later if needed.
- **Auto-update verification**: Claude Code auto-updates in the background
  after installation. Verifying binaries after auto-updates is outside the
  scope of this Ansible role.
- **Automatic version bumps**: refreshing the pinned version and checksums to
  a newer upstream release is a manual, reviewed action via
  `playbooks/update-versions/perform-updates.yml` followed by a `git diff`
  review and commit -- this role never auto-upgrades on its own.
