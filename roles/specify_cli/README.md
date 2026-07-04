<!-- SPDX-License-Identifier: MIT-0 -->
# specify_cli

Ansible role that installs [spec-kit](https://github.com/github/spec-kit)'s
`specify` CLI via `pipx` for each desktop user.

specify-cli is cross-harness — usable from Claude Code, opencode, Codex, and
other agent harnesses — so it is a standalone role rather than bundled into
`claude_code`.

## Requirements

- Ansible 2.19+
- Internet access from target hosts (VCS-style pipx install clones spec-kit
  via `git`)

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `desktop_user_names` | *(required)* | List of local usernames to install specify-cli for |

The installed version (`v0.8.18`) is pinned inline in the `pipx install`
command in `tasks/main.yml`, deliberately not exposed as a role variable —
promoting it would trigger the Constitution's mandatory version-update-
mechanism wiring, which is out of scope for this role (tracked as a
follow-up, not implemented here).

## Dependencies

None. See `meta/main.yml` for details.

## Example Playbook

```yaml
- hosts: desktops
  roles:
    - role: specify_cli
      vars:
        desktop_user_names:
          - alice
          - bob
```

## What This Role Does

1. Installs `pipx` and `git`
2. Installs `specify` via `pipx install git+https://github.com/github/spec-kit.git@v0.8.18`
   (skipped if already installed)

## License

MIT-0
