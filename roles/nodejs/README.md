<!-- SPDX-License-Identifier: MIT-0 -->
# nodejs

Ansible role that installs [nvm](https://github.com/nvm-sh/nvm) and the
Node.js LTS release (`lts/jod`) for each desktop user, plus a set of
frequently used global npm packages (`eslint`, `markdownlint-cli`,
`prettier`, `typescript`).

Node.js is cross-harness — several roles and CLIs in this repo (notably
`claude_code`'s `omc` CLI install) need a working Node/npm toolchain, so
it is a standalone role rather than bundled into any one consumer.

## Requirements

- Ansible 2.19+
- Internet access from target hosts (downloads the nvm install script and
  Node.js release)

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `desktop_user_names` | *(required)* | List of local usernames to install Node.js for |
| `first_desktop_user` | `{{ desktop_user_names[0] }}` | The user whose resolved Node.js version is used to key the global npm packages' idempotency guard (every desktop user installs the identical `lts/jod` alias, so one user's resolved version is representative) |

The nvm version (`v0.40.3`) is pinned as an inline literal in
`tasks/main.yml`, not in `defaults/main.yml` — deliberately, to avoid
triggering Constitution Principle II's mandatory version-update-mechanism
wiring (see `roles/specify_cli`'s identical treatment of its own pinned
version for the same reasoning).

## Dependencies

None. See `meta/main.yml` for details.

## Example Playbook

```yaml
- hosts: desktops
  roles:
    - role: nodejs
      vars:
        desktop_user_names:
          - alice
          - bob
```

## What This Role Does

1. Installs `curl`
2. Installs nvm for each desktop user
3. Installs and sets Node.js `lts/jod` as the default for each user
4. Installs `eslint`, `markdownlint-cli`, `prettier`, and `typescript` as
   global npm packages

## License

MIT-0
