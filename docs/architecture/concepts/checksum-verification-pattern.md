<!-- SPDX-License-Identifier: MIT-0 -->

# Checksum-verification pattern for web-hosted installer scripts

Several roles in this repo install a binary tool distributed as a
GitHub release rather than a distro package: `claude_code` (the Claude
Code binary), `rtk`, and `beads` (`bd`/`bv`). This document is the
authoritative source of truth for the shared verification pattern they
follow, so a future tool doesn't have to rediscover it from scratch.

## The pattern

1. **Assert supported architecture** up front — fail loud (Constitution
   Principle XII) before attempting any download if
   `ansible_facts['architecture']` isn't a key in the role's
   `*_platform_map` (defined in `defaults/main.yml`).
2. **Resolve the version**, only if the checksum source or archive URL
   needs a version-tagged path — a `uri` GET against
   `https://api.github.com/repos/<owner>/<repo>/releases/latest`,
   extracting `tag_name`. Not every tool needs this: `rtk`'s archive
   filenames are version-agnostic, so version resolution there exists
   only to build the versioned `checksums.txt` URL, not the archive URL.
3. **Resolve the checksum source** — see the three shapes below.
4. **Download and verify**, then only extract/place the artefact after
   verification succeeds.
5. **Gate the whole resolve-and-download block behind a `stat` check**
   on the already-installed artefact (see "Idempotency semantics"
   below) — this is not optional.

## Three checksum-source shapes seen in this repo

Different upstreams publish release checksums differently. All three
appear across the roles that follow this pattern:

| Shape | Example | How it's consumed |
| --- | --- | --- |
| Aggregate `checksums.txt` (sha256sum format, `<hash>  <filename>`) | `rtk-ai/rtk`, `gastownhall/beads` | Passed directly as a URL to `get_url`'s native `checksum:` parameter — Ansible fetches and parses the file itself. |
| Release manifest JSON (`artifacts[].name`/`.sha256`) | `Dicklesworthstone/beads_viewer`'s `beads_viewer-v<version>-manifest.json`; `claude_code`'s own `manifest.json` | Fetched via `uri`, the matching artifact's hash extracted with a Jinja `selectattr`/`map` filter chain, `assert`ed non-empty, then passed as a **literal** `checksum: "sha256:{{ expected }}"` — manifest JSON isn't a format `get_url`'s own URL-lookup can parse. |
| Per-asset `.sha256` sidecar (`<archive>.sha256`) | Available for `beads_viewer` releases | Not chosen anywhere in this repo — the aggregate manifest was already present for that tool and is a closer match to the JSON-manifest shape used elsewhere, so one lookup style could be reused instead of two. |

For the manifest-JSON shape, the Jinja expression that extracts the matching
artifact's hash is:

```jinja
{{ (manifest.content | from_json).artifacts
   | selectattr('name', 'equalto', archive_name)
   | map(attribute='sha256') | first }}
```

### The `get_url` basename-matching constraint

When using the checksums.txt-URL form, `get_url` finds the matching
line by comparing its own `dest` filename's **basename** against each
entry in the fetched checksums file. The `dest` you choose MUST be
named exactly the same as the upstream release asset (e.g.
`rtk-x86_64-unknown-linux-musl.tar.gz`, `beads_1.2.3_linux_amd64.tar.gz`)
— renaming it on download breaks the lookup, and `get_url` fails loud
("Unable to find a checksum for file...") rather than silently
skipping verification, so this is a footgun to know about, not a
silent gap.

## Idempotency semantics: frozen-at-first-install, not auto-upgrading

The `rtk` and `beads` roles gate their whole resolve-and-download block
behind a `stat` check on the artefact already being present (a single
scalar check for system-wide `rtk`; a per-user loop for `beads`, since
`bd`/`bv` install into each desktop user's own `~/.local/bin`). This is
deliberate and load-bearing, not just an idempotency nicety:

- **Without the gate**, a `get_url` task using a checksum-**URL** would
  re-fetch `checksums.txt` (or the manifest) on every converge to
  compare against the already-downloaded file. The moment upstream
  ships a new release, the expected hash changes — for a tool like
  `rtk` whose archive filenames never change across releases, this
  would cause `get_url` to silently re-download and replace the
  installed binary on every subsequent run: an unrequested,
  un-idempotent auto-upgrade (Constitution Principle I violation).
- **With the gate**, verification only ever runs once, at first
  install. The version present on a host is frozen at whatever release
  was current then, and stays that way until the binary is manually
  removed and the role re-run.

This is a **different** idempotency model from `claude_code`'s own
`install-claude-code.yml`, which does the inverse: it always re-runs
the installer script unconditionally, then always re-verifies the
*installed binary's* checksum against the manifest afterward — deleting
and failing on a still-working binary the moment an upstream release
bumps the expected checksum. That always-reverify behavior is
deliberately **not** copied by `rtk`/`beads`: it produces install-time
churn on every apply once a tool has multiple active releases, which
this pattern avoids by verifying only once, at install time.

Refreshing a frozen-at-first-install tool to a newer release is
therefore a manual action (remove the binary, re-run the role) until a
version-update-mechanism entry exists for it — tracked as
`ansible-all-my-things-m0av`.

## Reference implementations

- `roles/claude_code/tasks/install-claude-code.yml` — the original
  pattern in this repo: architecture assert, dynamic latest-version
  resolution, manifest-based checksum, always-reverify-on-rerun (a
  model this repo's newer roles deliberately diverge from, see above).
- `roles/rtk/tasks/main.yml` — aggregate `checksums.txt` via `get_url`'s
  native `checksum:` URL form, stat-gated, single system-wide artefact.
- `roles/beads/tasks/main.yml` — both shapes in one file: `bd` uses the
  `checksums.txt` form (like `rtk`); `bv` uses the manifest-JSON form
  (like `claude_code`'s own manifest), plus the per-user
  stat-loop/shared-extract/per-user-`copy` mechanics described below.

## The shared-extract-then-per-user-copy mechanics (bd/bv only)

Unlike `claude_code`'s and `rtk`'s single system-wide binary, `bd` and
`bv` install into each desktop user's own `~/.local/bin`. Downloading
and verifying the archive once per host — rather than once per user —
avoids redundant network calls:

1. `stat` the target path for every user in `desktop_user_names`,
   looped, registering one result per user.
2. Gate the download+verify+extract block on "any user missing it"
   (`results | selectattr('stat.exists', 'equalto', false) | list |
   length > 0`).
3. `unarchive` (`remote_src: true`) the verified tarball into a neutral
   `/tmp` staging directory — not directly into any user's home.
4. A separate `copy` task (`remote_src: true`, `mode: '0755'`,
   `become_user: "{{ item.item }}"`) loops over the stat results and
   places the binary only for the specific users who don't already
   have it (`when: not item.stat.exists`).

This mechanism has no other precedent in this repo (`claude_code`'s own
per-user installs re-run a shell installer per user instead), so it's
worth this explicit write-up rather than leaving it to be inferred from
the task file alone.
