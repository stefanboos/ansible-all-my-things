<!-- SPDX-License-Identifier: MIT-0 -->
# omc_cli

Ansible role that clones the
[oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode) source
repository and installs the `omc` CLI (`oh-my-claude-sisyphus` npm package)
for each desktop user.

omc is cross-harness — usable from Claude Code, opencode, Codex, and other
agent harnesses — so it is a standalone role rather than bundled into
`claude_code`. See `DESIGN.md` for why this is distinct from the
oh-my-claudecode Claude Code **plugin** (which stays in `claude_code`), and
for a known limitation around production Node.js provisioning.

## Requirements

- Ansible 2.19+
- `~/.nvm/nvm.sh` present for each target user (provisioned in production by
  `playbooks/setup-nodejs.yml`, not by this role)
- Internet access from target hosts

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `desktop_user_names` | *(required)* | List of local usernames to install omc for |

## Dependencies

None. See `meta/main.yml` for details.

## Example Playbook

```yaml
- hosts: desktops
  roles:
    - role: omc_cli
      vars:
        desktop_user_names:
          - alice
          - bob
```

## What This Role Does

1. Installs `git`
2. Clones the oh-my-claudecode source repo to `~/Documents/Cline/oh-my-claudecode`
3. Installs the `omc` CLI via `npm install -g oh-my-claude-sisyphus` (skipped
   if already installed)

## License

MIT-0
