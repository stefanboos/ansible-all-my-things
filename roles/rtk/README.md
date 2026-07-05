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
- Internet access from target hosts (queries the GitHub Releases API and
  downloads the release archive)

The installed version is frozen on first install: the role downloads whatever
release is latest at that moment and does not auto-upgrade on later runs. A
`stat` gate short-circuits the whole download-and-verify block once
`/usr/local/bin/rtk` exists, so a new upstream release is never pulled in
silently. To upgrade, remove the binary and re-run the role.

## Dependencies

None. See `meta/main.yml` for details.

## Example Playbook

```yaml
- hosts: desktops
  roles:
    - role: rtk
```

## What This Role Does

Skipped entirely once `/usr/local/bin/rtk` exists. On first install it:

1. Resolves the latest release tag from the GitHub Releases API
2. Downloads the architecture-specific release archive and verifies it against
   the release's `checksums.txt` using Ansible-native SHA256 checking, which
   fails atomically (leaving no partial file) on mismatch
3. Extracts the verified `rtk` binary to `/usr/local/bin/rtk`

## License

MIT-0
