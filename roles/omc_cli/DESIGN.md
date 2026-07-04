<!-- SPDX-License-Identifier: MIT-0 -->
# omc_cli — Design Notes

## Distinct from the oh-my-claudecode Claude Code plugin

This role installs the `omc` npm CLI (`oh-my-claude-sisyphus`) and clones the
oh-my-claudecode **source repository** for reference. It is unrelated to the
oh-my-claudecode **Claude Code plugin**, which is installed via
`claude plugin install` and stays in the `claude_code` role — that mechanism
is Claude-Code-specific (marketplace registration), while the CLI and source
clone are cross-harness.

## Known limitation: production Node.js is provisioned elsewhere

This role's tasks assume `~/.nvm/nvm.sh` already exists so `npm install -g`
can run. In production, nvm/Node.js is provisioned by the separate
`playbooks/setup-nodejs.yml` playbook, not by any role in
`configure-profile-roles.yml`.

This role's own Molecule scenario provisions nvm/Node LTS itself in
`molecule/default/prepare.yml` (copied verbatim from the pre-extraction
`claude_code` fixture) purely as a test fixture, so `molecule test` can
converge and verify the `omc` CLI in isolation. A green `molecule test` run
for this role therefore does **not** validate the real cross-playbook
dependency on `playbooks/setup-nodejs.yml` having already run against a
target host. This is a known, accepted gap (see plan.md
`specs/016-extract-cross-harness-roles/plan.md`), not a defect introduced by
this role.
