<!-- SPDX-License-Identifier: MIT-0 -->
# nodejs

Ansible role that installs a specific Node.js LTS release system-wide,
plus a set of frequently used global npm packages (`eslint`,
`markdownlint-cli`, `prettier`, `typescript`).

Node.js is cross-harness — several roles and CLIs in this repo (notably
`claude_code`'s `omc` CLI install) need a working Node/npm toolchain, so
it is a standalone role rather than bundled into any one consumer.

## Requirements

- Ansible 2.19+
- Internet access from target hosts (queries nodejs.org's release index and
  downloads the release archive)

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `node_platform_map` | `{x86_64: x64, aarch64: arm64}` | Maps `ansible_architecture` to nodejs.org's release-archive platform fragment |

The installed version is frozen on first install: the role resolves whichever
Node.js LTS release is latest at that moment (via nodejs.org's
`index.json`) and does not auto-upgrade on later runs — this is a
deliberate trade-off, not an oversight. Different VMs provisioned at
different times may land on different Node.js LTS majors, since no version
is pinned; this favors the disposable, create-and-destroy VM model this repo
uses over cross-provision reproducibility. A `stat` gate on
`/usr/local/bin/node` short-circuits the whole resolve-and-download block
once Node.js is installed, so a new upstream LTS release is never pulled in
silently. To upgrade, remove `/usr/local/bin/node` and re-run the role.

## Dependencies

None. See `meta/main.yml` for details.

## Example Playbook

```yaml
- hosts: desktops
  roles:
    - role: nodejs
```

## What This Role Does

Skipped entirely (Node.js install) once `/usr/local/bin/node` exists. On
first install it:

1. Resolves the latest LTS release from nodejs.org's release index
2. Downloads the architecture-specific release archive and verifies it
   against the release's `SHASUMS256.txt` using Ansible-native SHA256
   checking, which fails atomically (leaving no partial file) on mismatch
3. Extracts the verified archive to `/usr/local/lib/nodejs` and symlinks
   `node`, `npm`, `npx`, and `corepack` into `/usr/local/bin`

Regardless of whether Node.js was just installed or already present, it then
installs the global npm packages (`eslint`, `markdownlint-cli`, `prettier`,
`typescript`) system-wide via `npm install --global --prefix /usr/local`,
skipped once already installed.

## License

MIT-0
