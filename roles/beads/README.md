<!-- SPDX-License-Identifier: MIT-0 -->
# beads

Ansible role that installs the [beads](https://github.com/gastownhall/beads)
issue tracker (`bd`), the [beads viewer](https://github.com/Dicklesworthstone/beads_viewer)
(`bv`), and clones the beads source repository for each desktop user.

beads is cross-harness — usable from Claude Code, opencode, Codex, and other
agent harnesses — so it is a standalone role rather than bundled into
`claude_code`. It is a sibling of `dolt_sql_server` (beads is Dolt-backed).

## Requirements

- Ansible 2.19+
- Internet access from target hosts (downloads the bd/bv release archives)

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `desktop_user_names` | *(required)* | List of local usernames to install beads for |

Each binary is fetched directly from its GitHub release with an Ansible-native
checksum verification, and its version is frozen at whatever release was present
on the target when the binary was first installed. The role does not
auto-upgrade an already-installed `bd` or `bv` when a newer upstream release
ships; remove the binary to force a fresh download of the current latest.

## Dependencies

None. See `meta/main.yml` for details.

## Example Playbook

```yaml
- hosts: desktops
  roles:
    - role: beads
      vars:
        desktop_user_names:
          - alice
          - bob
```

## What This Role Does

1. Installs `git` (needed to clone the beads source repository)
2. Installs the beads issue tracker (`bd`) to `/usr/local/bin/bd`,
   system-wide, by downloading the latest
   [gastownhall/beads](https://github.com/gastownhall/beads) release archive
   and verifying it against the release `checksums.txt` before extraction.
   Skipped if `bd` is already installed.
3. Installs the beads viewer (`bv`) to `/usr/local/bin/bv`, system-wide, by
   downloading the latest
   [Dicklesworthstone/beads_viewer](https://github.com/Dicklesworthstone/beads_viewer)
   release archive and verifying it against the sha256 recorded in that
   release's aggregate manifest. Skipped if `bv` is already installed.
4. Clones the beads source repository to `~/Documents/Cline/beads` for each
   desktop user

## License

MIT-0
