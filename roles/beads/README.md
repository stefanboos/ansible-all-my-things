<!-- SPDX-License-Identifier: MIT-0 -->
# beads

Ansible role that installs the [beads](https://github.com/steveyegge/beads)
issue tracker (`bd`), the [beads viewer](https://github.com/Dicklesworthstone/beads_viewer)
(`bv`), and clones the beads source repository for each desktop user.

beads is cross-harness — usable from Claude Code, opencode, Codex, and other
agent harnesses — so it is a standalone role rather than bundled into
`claude_code`. It is a sibling of `dolt_sql_server` (beads is Dolt-backed).

## Requirements

- Ansible 2.19+
- Internet access from target hosts (downloads the bd/bv installers)

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `desktop_user_names` | *(required)* | List of local usernames to install beads for |

No version is pinned: the installers always install the latest release.

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

1. Installs `git` and `curl`
2. Installs the beads issue tracker (`bd`) to `~/.local/bin/bd` for each user
3. Installs the beads viewer (`bv`) to `~/.local/bin/bv` for each user
4. Clones the beads source repository to `~/Documents/Cline/beads`

## License

MIT-0
