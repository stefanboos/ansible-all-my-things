<!-- SPDX-License-Identifier: MIT-0 -->
# rtk

Ansible role that installs [rtk (Rust Token Killer)](https://github.com/rtk-ai/rtk),
a token-optimized CLI proxy for AI coding agents, to `/usr/local/bin/rtk`.

rtk's binary is cross-harness — usable from Claude Code, opencode, Codex, and
other agent harnesses — so it is a standalone role rather than bundled into
`claude_code`. Initializing rtk's Claude-Code-specific global config
(`rtk init -g`, which writes into `~/.claude`) is done by the `claude_code`
role instead — see its `DESIGN.md` for why.

## Requirements

- Ansible 2.19+
- Internet access from target hosts (downloads the rtk installer)

No version is pinned: the installer always installs the `master` branch build.

## Dependencies

None. See `meta/main.yml` for details.

## Example Playbook

```yaml
- hosts: desktops
  roles:
    - role: rtk
```

## What This Role Does

1. Installs `curl`
2. Downloads and runs the official rtk installer to `/usr/local/bin/rtk`
   (skipped if already installed)

## License

MIT-0
