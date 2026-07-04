<!-- SPDX-License-Identifier: MIT-0 -->
# rtk

Ansible role that installs [rtk (Rust Token Killer)](https://github.com/rtk-ai/rtk),
a token-optimized CLI proxy for AI coding agents, and initializes it globally
for each desktop user.

rtk is cross-harness — usable from Claude Code, opencode, Codex, and other
agent harnesses — so it is a standalone role rather than bundled into
`claude_code`.

## Requirements

- Ansible 2.19+
- Internet access from target hosts (downloads the rtk installer)

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `desktop_user_names` | *(required)* | List of local usernames to run `rtk init -g` for |

No version is pinned: the installer always installs the `master` branch build.

## Dependencies

None. See `meta/main.yml` for details.

## Example Playbook

```yaml
- hosts: desktops
  roles:
    - role: rtk
      vars:
        desktop_user_names:
          - alice
          - bob
```

## What This Role Does

1. Installs `curl`
2. Downloads and runs the official rtk installer to `/usr/local/bin/rtk`
   (skipped if already installed)
3. Ensures `~/.claude` exists for each desktop user (a precondition for
   `rtk init -g`)
4. Runs `rtk init -g` for each desktop user to initialize global rtk config

## License

MIT-0
