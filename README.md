# skill-hub

**One canonical library for all your agent skills — across agents, devices, and upstreams. Maintained by the agents themselves.**

skill-hub is a small, dependency-free CLI (Python 3.11+ standard library) plus a `SKILL.md` that teaches your coding agent how to run it. It turns the sprawl of `~/.claude/skills`, `~/.agents/skills`, `~/.codex/skills`, plugin caches, `npx` installers, cloned repos and hand-written one-offs into **one git repository** from which every agent's skill directory is rendered.

- [The problem](#the-problem)
- [The doctrine](#the-doctrine)
- [How it works](#how-it-works)
- [Quick start](#quick-start)
- [Concepts](#concepts): [instance layout](#instance-layout) · [`hub.toml`](#hubtoml--the-manifest) · [device files](#deviceidtoml--per-device-selection) · [lockfile](#hublockjson--provenance) · [origins](#three-origins-self-vendor-external) · [selection algorithm](#selection-algorithm)
- [Materialization rules](#materialization-rules)
- [`hub sync`](#hub-sync)
- [Install and update (3-way merge)](#install-and-update-3-way-merge)
- [Adopt and census](#adopt-and-census)
- [Doctor](#doctor)
- [The agent contract](#the-agent-contract)
- [Sync transport: git, not GitHub](#sync-transport-git-not-github)
- [Safety guarantees](#safety-guarantees)
- [Dogfooding](#dogfooding)
- [Migrating from a messy setup](#migrating-from-a-messy-setup)
- [Command reference](#command-reference)
- [Limitations and roadmap](#limitations-and-roadmap)
- [Development](#development)

---

## The problem

Every agent reads skills from its own directory. Skills arrive from many sources: a Claude Code plugin marketplace, `npx skills`-style installers, a GitHub repo you cloned, something a colleague sent, a file you wrote at 2 a.m. To share them across agents people symlink directories into each other — and it rots:

- Plugin caches rotate their hash-named directories on update; every symlink into them dies silently (a real machine had **104** of those).
- Two directories end up as two clones of the same repo, each with hundreds of uncommitted changes, symlinked to each other in a loop.
- Installers leave `foo-1.0.1` directories that no one updates; hand copies of upstream skills drift and can never be merged back.
- A second machine is a re-do of all of the above.

## The doctrine

1. **One canonical git repo** (the *instance*, default `~/skills-hub`) holds every skill as a real, vendored entity. Nothing else is a source of truth.
2. **Discovery directories are render targets, not data.** `~/.claude/skills` and friends contain only per-skill symlinks (or managed copies) and can be deleted and rebuilt with one command.
3. **Declarative selection.** `hub.toml` declares where things come from and how they group; `devices/<id>.toml` declares what each machine enables. One file per device, so manifests never conflict between machines.
4. **Agents do the upkeep.** Every command is idempotent, has `--json` output, exit-code semantics and (where destructive) `--dry-run`. The bundled `SKILL.md` gives the agent the full routine. Humans state intent; agents run the plumbing.

Three iron rules fall out of this:

| # | Rule |
|---|---|
| 1 | Entities exist **only** inside the instance's `skills/`. Agent directories hold links or managed copies. |
| 2 | **Never** symlink anything under `~/.claude/plugins/cache/`. Plugin skills are the plugin system's job; a link into the cache is treated as damage and removed on sight. |
| 3 | hub only deletes what it created (plus cache links and its own dead links). Everything else is *foreign*: reported, never touched. |

## How it works

```
 upstream sources                  the instance (git repo)                 render targets
 ─────────────────                 ───────────────────────                 ──────────────
 GitHub repo / subdir  ─install─▶  skills/<name>/     ◀── the only entities ─┬─▶ ~/.claude/skills/<name>  (symlink)
 local directory       ─adopt───▶  upstream/<name>/   ◀── pristine snapshot  ├─▶ ~/.agents/skills/<name>  (symlink)
 existing mess         ─census──▶  hub.toml           ◀── sources/profiles   ├─▶ ~/.codex/skills/<name>   (copy, optional)
                                   devices/<id>.toml  ◀── this machine       └─▶ any agent dir you declare
                                   hub.lock.json      ◀── provenance
                                            ▲  ▼
                                   git pull --rebase / push  ◀──▶  any git remote (or none)
```

`hub sync` is the only daily verb: commit drift → pull → **materialize** (place links, clean up what hub owns) → push. `hub update` merges upstream changes into your possibly-patched copy using `upstream/` as the merge base. `hub doctor` audits the whole thing. A `SessionStart` hook surfaces pending work to the agent; `launchd` runs the safe parts unattended.

## Quick start

**Create a new instance**

```bash
git clone https://github.com/xiaolinfrank/skill-hub.git ~/repos/skill-hub
~/repos/skill-hub/scripts/hub init                      # scaffolds ~/skills-hub (or $SKILL_HUB_ROOT)
cd ~/skills-hub
~/repos/skill-hub/scripts/hub adopt ~/repos/skill-hub --name skill-hub --origin vendor \
    --source "git+https://github.com/xiaolinfrank/skill-hub#ref=main"   # vendor the tool itself
~/skills-hub/skills/skill-hub/scripts/hub onboard --device my-laptop   # from now on: `hub`
hub sync
```

`onboard` writes the device id to `~/.agents/device-id`, registers the lockfile merge driver in this clone, links `hub` into `~/.local/bin`, and creates `devices/my-laptop.toml` (enabling profile `base`, which already contains `skill-hub`). Add `--with-hook --with-launchd` for automation (see [The agent contract](#the-agent-contract)).

**Join an existing instance from another machine**

```bash
git clone <your-instance-remote> ~/skills-hub
~/skills-hub/skills/skill-hub/scripts/hub onboard --device my-desktop --with-hook --with-launchd
hub sync
```

Because everything is vendored, `clone` is the whole restore. Edit `devices/my-desktop.toml` to choose profiles, `hub sync` again.

**Bring in a skill**

```bash
hub install "git+https://github.com/anthropics/skills#ref=main&subdir=skills/canvas-design" --profile base
hub sync
```

Requirements: Python 3.11+, git. macOS or Linux (symlinks; the launchd automation is macOS-only — on Linux schedule the same commands with cron/systemd).

## Concepts

### Instance layout

```
~/skills-hub/
├── hub.toml              manifest: render targets, profiles, vendor sources
├── devices/<id>.toml     one file per machine: profiles / agents / extra / disable / external
├── hub.lock.json         provenance lockfile — machine-maintained, never hand-edited
├── skills/<name>/        THE entity layer, flat, one directory per skill
├── upstream/<name>/      pristine upstream snapshot for vendor-origin skills (merge base)
├── NEEDS_ATTENTION.md    async inbox for the maintaining agent (absent when clean)
├── .hub-pending/<name>/  incoming snapshot of an unfinished `update` (gitignored)
├── .logs/                launchd logs + last_sync timestamp (gitignored)
├── .gitattributes        hub.lock.json merge=hub-lock
└── .gitignore
```

Skill names must match `^[a-z0-9]+(-[a-z0-9]+)*$` (some agents enforce this). `adopt`/`install` normalize names: lower-case, `_`/`.`/space → `-`, version suffixes like `-1.0.1` stripped, the original recorded under `aliases` in the lockfile. Nested `.git` directories are never copied in.

### `hub.toml` — the manifest

```toml
[hub]
schema = 1
default_agents = ["claude", "agents-std"]   # agents that receive every enabled skill by default

[agents.claude]                 # a render target
target  = "~/.claude/skills"
mode    = "symlink"             # symlink | copy
protect = ["synced"]            # entries hub must never touch (synced/ and .system are always protected)

[agents.agents-std]             # the emerging cross-agent convention (Codex, Cursor, opencode, …)
target = "~/.agents/skills"
mode   = "symlink"

[agents.codex-legacy]           # for agents that cannot follow symlinks: rsync-style managed copies
target  = "~/.codex/skills"
mode    = "copy"
protect = [".system"]

[profiles]                      # named sets; patterns are shell globs over skills/ names
base     = ["skill-hub", "crawl", "search"]
dingtalk = ["dingtalk-*"]

[skills.domain-modeling]        # only vendor-origin skills need a block; undeclared = self
origin = "vendor"
source = "git+https://github.com/mattpocock/skills#ref=main&subdir=skills/engineering/domain-modeling"

[skills.onepassword]
origin = "vendor"
source = "git+https://github.com/example/skills#ref=main&subdir=1password"
pin    = true                   # frozen: `update` skips it
# agents = ["claude"]           # optional: restrict this skill to some agents (default: default_agents)
```

hub writes to `hub.toml` in exactly two ways: `install`/`adopt` **append** a `[skills.<name>]` block (append-only text, no TOML serializer round-trip), and `install --profile P` inserts the name into the single-line array of profile `P` inside the `[profiles]` section. Everything else — reorganizing profiles, removing blocks — is a text edit by you or your agent; hub validates with `tomllib` on the next read and refuses to run (exit 2) on syntax errors.

### `devices/<id>.toml` — per-device selection

```toml
profiles = ["base", "dingtalk"]          # union of these profiles …
agents   = ["claude", "agents-std"]      # … rendered into these targets
extra    = ["grilling"]                  # plus these (globs allowed)
disable  = ["dingtalk-mail"]             # minus these (globs allowed)

[[external]]                             # a directory that stays OUTSIDE the instance
name   = "web-lookup"
path   = "~/repos/company/skills/web-lookup"
agents = ["claude"]                      # optional; default: this device's agents
```

The device id comes from `~/.agents/device-id` (written by `onboard`), **not** from the hostname — hostnames drift and rarely match what you call your machines. If the file or the matching `devices/<id>.toml` is missing, `hub sync` exits 2 with instructions instead of silently falling back to some default set.

Each machine edits only its own file, so concurrent edits on two machines never produce a manifest conflict.

### `hub.lock.json` — provenance

```json
{
  "schema": 1,
  "skills": {
    "domain-modeling": {
      "origin": "vendor",
      "source": "git+https://github.com/mattpocock/skills#ref=main&subdir=skills/engineering/domain-modeling",
      "upstream_commit": "9f2c41d8…",
      "upstream_hash": "sha256:1c0f…",
      "local_hash":    "sha256:88ae…",
      "patched": true,
      "aliases": [],
      "vendored_at": "2026-09-05T10:12:00+08:00",
      "updated_at":  "2026-09-05T10:12:00+08:00"
    },
    "crawl": { "origin": "self", "local_hash": "sha256:…", "adopted_at": "…" }
  }
}
```

`patched` is `upstream_hash != local_hash` — you changed a vendored skill. Hashes are content hashes of the tree (symlinks hash as their target text, `.git` and `.DS_Store` ignored). `sync` refreshes `local_hash`/`patched` for every skill and drops entries for skills that no longer exist.

Two machines both touching the lockfile would be git's classic JSON-conflict hell, so `.gitattributes` declares `hub.lock.json merge=hub-lock` and `onboard` registers the driver (`hub lock-merge %O %A %B`) in each clone: keys are unioned, same-key conflicts resolved by newer `updated_at`. Doctor check 13 verifies the driver is registered.

### Three origins: self, vendor, external

| origin | what it means | `upstream/` | `update` |
|---|---|---|---|
| **self** | you wrote it, or no upstream can be traced. The default; zero ceremony. | no | n/a |
| **vendor** | has a git source. Pristine snapshot kept; local patches survive upstream updates via 3-way merge. | yes | yes |
| **external** | declared in a device file only; content never enters the instance. Rendered as a symlink to the absolute path. For things that must not be in any repo. | no | n/a |

### Selection algorithm

For this device: `enabled = ⋃ profiles (globs expanded over skills/) ∪ extra − disable`, validated so every name exists. For each agent in the device's `agents`: `want = {s ∈ enabled | agent ∈ (skills[s].agents or default_agents)}`. Externals are added per their own `agents` list.

## Materialization rules

Materialization is the second half of `hub sync` (and the only part that touches agent directories). It is stateless for symlink mode — ownership is decided by *shape*, not by a database — and uses a small `.hub-state.json` inside copy-mode targets.

**Placement** (for every wanted skill, in every target):

| the slot `target/<name>` currently holds | symlink mode | copy mode |
|---|---|---|
| nothing | create link | copy |
| a link that already points at `skills/<name>` | keep | — |
| a link into the instance's `skills/`, into the plugin cache, or a dead link | replace | blocked (report) |
| **anything else** — a real directory, a file, a link to somewhere of yours | **blocked**: reported as foreign, `needs_attention`, exit 1 | same; a copy-mode directory is only refreshed if `.hub-state.json` says hub created it |

**Cleanup** (for every entry in every target, except `protect` names, `synced/`, `.system`, `.hub-state.json`):

| entry | action |
|---|---|
| link resolving under `~/.claude/plugins/cache/` | **remove** (rule 2 — dead or alive) |
| link exactly of the shape hub creates (`→ skills/<name>` with the same name), skill no longer enabled | remove |
| link to a declared external path, external no longer enabled for this agent | remove |
| dead link that pointed into the instance | remove |
| dead link pointing anywhere else (unmounted volume, someone else's tool) | **keep**, report |
| link into the instance but not hub-shaped (your own alias or deep link) | keep, silently |
| link to anywhere else | keep, report as foreign |
| directory hub copied (copy mode, in `.hub-state.json`), no longer enabled | remove |
| any other directory or file | keep, report as foreign |

`--dry-run` runs the same decision tree and prints `[dry] …` for each action.

## `hub sync`

```
0. refuse to commit/pull/push while .hub-pending/ holds an unfinished update (never spread conflict markers)
1. adopt drift: `git add -A && git commit -m "sync(<device>): auto-commit drift"` — whoever made the change
2. `git pull --rebase --autostash`  (skipped with no remote or --no-pull)
      conflict → `git rebase --abort` (worktree stays usable), details → NEEDS_ATTENTION.md, exit 1
3. materialize (see above)
4. `git push`  (skipped with no remote or --no-push; a failed push is a warning, retried next round)
5. refresh lockfile hashes → commit "sync(<device>): refresh lock" → push; write .logs/last_sync
```

Any `needs_attention` item raised during a sync is also appended (deduplicated) to `NEEDS_ATTENTION.md`, so unattended runs leave a trail the next session can pick up. A change you make on machine A — even by editing through the symlink inside an agent session — is committed by A's next sync and arrives at B on B's next sync. Two machines editing the same file between syncs is a rebase conflict: aborted, recorded, resolved semantically by the agent in a session. hub never force-pushes.

## Install and update (3-way merge)

**Sources.** `git+<url>#ref=<branch|tag>&subdir=<path>` or a local directory. `install` does a shallow clone, takes the subdir, copies it to both `upstream/<name>` and `skills/<name>` (minus `.git`), appends the manifest block, records `upstream_commit` and hashes, and commits. The name defaults to the subdir's basename, else the repo name, else `--name`. Reinstalling an existing name is refused (exit 2) — remove `skills/<name>`, `upstream/<name>` and the manifest block first.

**Freshness.** `hub update --check [--all|names]` runs `git ls-remote <url> <ref>` and compares the exact `refs/heads/<ref>` (or tag) SHA with `upstream_commit`. Entries without a recorded commit (e.g. adopted from a plain copy) are reported as `unknown_freshness`, not as updatable, so a weekly check never spams. `--write-attention` records updatable skills in `NEEDS_ATTENTION.md`.

**Apply.** `hub update <name>` fetches the new snapshot into `.hub-pending/<name>` and merges into `skills/<name>` file by file with `base = upstream/<name>`, `ours = skills/<name>`, `theirs = pending`:

| situation | result |
|---|---|
| upstream added a file | added |
| upstream deleted, you never touched it | deleted |
| upstream deleted, you modified it | conflict |
| you deleted, upstream modified | conflict |
| only upstream changed | taken |
| only you changed / identical | kept |
| both changed (text) | `git merge-file` — clean or `<<<<<<<` markers left in place |
| both changed (binary / unmergeable) | your version kept, upstream saved as `<file>.upstream` sidecar |
| a symlink changed on both sides | conflict (links are never text-merged and never written through) |

No conflicts → `upstream/` replaced by the new snapshot, lockfile updated (`patched` recomputed), committed. Conflicts → nothing committed, the list goes to `NEEDS_ATTENTION.md`, exit 1. The agent resolves markers / sidecars and runs `hub update <name> --continue` (no name = every pending update), which refuses while any marker or sidecar remains. `pin = true` skips a skill entirely. Re-running `update` on a skill with a pending merge is refused until `--continue`.

Self-origin skills have no update concept: edit, `hub sync`, done.

## Adopt and census

`hub census [--dirs …] [--out file]` scans agent directories (default: `~/.claude/skills`, `~/.agents/skills`, `~/.codex/skills`) read-only and labels every entry:

| label | meaning | `adopt --from-census` |
|---|---|---|
| `entity` / `entity-versioned` | real directory (the latter with a version suffix) | adopt (name normalized) |
| `nested-git` | real directory that is itself a git repo | adopt without `.git`; remote recorded as vendor source |
| `external-link` | link to a directory outside the scanned dirs | adopt from the target; if it lives in a git checkout, remote + subdir recorded as vendor source |
| `cross-link` | link to an entity in another scanned dir | deduplicated by real path |
| `cache-link` | link into the plugin cache | dropped |
| `broken` | dead link | dropped |
| `hub-managed` / `internal` / `file` | already ours / dotfiles / stray files | ignored |

Same-name candidates are compared by content hash: identical copies collapse to one (preferring the copy that carries provenance); different contents are flagged `CONFLICT` and skipped, so a human or agent decides with `hub adopt <path> --name <other>`. `--skip a,b` excludes names (e.g. ones you want as externals). Adoption persists the lockfile after every skill, so an interrupted run loses nothing.

`hub adopt <path>` for a single directory auto-detects provenance: inside a git checkout with a remote → vendor (`--origin self` to override), otherwise self.

## Doctor

`hub doctor [--fix] [--write-attention]` runs these checks (numbers are stable identifiers used in output):

| # | check | `--fix` |
|---|---|---|
| 1 | dead links that pointed into the instance or to a declared external | remove |
| 2 | links into the plugin cache | remove |
| 3 | foreign links, dead foreign links, foreign directories | report only |
| 4 | nested `.git` under `skills/` or `upstream/` | report |
| 5 | non-compliant skill names | report |
| 6 | profile patterns matching nothing (skills in no profile are listed as `library_only` info, not an issue) | report |
| 7 | lockfile hash drift (edited but not synced) | refresh hashes + commit |
| 8 | uncommitted / unpushed / behind remote | commit |
| 9 | **old dual-clone revival**: `~/.claude/skills` or `~/.agents/skills` has become a git repo again | report (red) |
| 12 | device id file missing; last sync older than 3 days | report |
| 13 | lockfile merge driver not registered in this clone | report |

Checks 1, 2, 4, 9, 13 are *hard*: if any remains unfixed, doctor exits 1 with `needs_attention`. `--fix` only acts on targets of agents enabled in **this device's** file; other targets are report-only. Checks 10 (same skill reachable via several paths) and 11 (stale plugin-cache audit) are reserved.

## The agent contract

- **Output.** Every command accepts `--json` (after the subcommand) and prints `{ok, cmd, actions[], warnings[], needs_attention[], data{}}`.
- **Exit codes.** `0` done · `1` needs agent attention (read `needs_attention` / `NEEDS_ATTENTION.md`) · `2` environment or configuration error (report to the human, do not retry blindly). `lock-merge` is the exception: it is a git merge driver with git's own contract.
- **`SKILL.md`.** The repository root is a skill. Vendored into an instance it teaches the agent: the intent → command map, the conflict playbook, when to escalate to the human (deleting foreign entries, force operations, same-name adoption choices, editing `[agents.*]` or the hub script itself), and the self-modification discipline (`hub selftest` must pass before committing changes to `scripts/hub`).
- **`NEEDS_ATTENTION.md`.** Written (deduplicated) by unattended runs; the agent works through it and deletes it.
- **`scripts/guard`.** A ~20-line bash `SessionStart` hook: prints one line if `NEEDS_ATTENTION.md` exists and one if the last sync is older than 3 days. It never mutates anything. `hub onboard --with-hook` adds it to `~/.claude/settings.json`.
- **launchd** (`hub onboard --with-launchd`, macOS): `com.skillhub.sync` runs `hub sync` daily at 09:30; `com.skillhub.weekly` runs `hub update --check --all --write-attention` and `hub doctor --write-attention` on Mondays at 10:00. Plists carry an explicit `PATH` (launchd's default one finds the system Python) and log to `.logs/`. Unattended jobs only do read-only checks and the safe sync; content-changing `update` happens in sessions.
- **Determinism vs judgment.** Scripts hash, link, pull, push and audit. Agents decide semantic merges, adoption choices and escalation.

## Sync transport: git, not GitHub

`hub sync` needs *a* git remote — or none:

| mode | setup | what you get |
|---|---|---|
| single machine | no remote | everything except multi-device sync; zero accounts |
| your own box as remote | `git init --bare skills-hub.git` on any SSH-reachable machine (Tailscale works great), `git remote add origin …` | multi-device sync, data never leaves your devices |
| hosted | GitHub / GitLab / Gitea / Codeberg, private | the usual |

There is deliberately **no** "sync the worktree with Dropbox/iCloud/Syncthing" mode: file-sync tools corrupt `.git` under concurrency, and everything valuable here (3-way merges, history, rescue branches, the lock merge driver) lives on git. Lightweight means *git without GitHub*, not *sync without git*.

## Safety guarantees

- hub never deletes a real directory or file it did not create. Foreign entries block placement and are reported; they are not removed.
- hub never writes through a symlink during merges; links are compared and replaced as links.
- hub never force-pushes, never commits while a merge is pending, and aborts a conflicting rebase instead of leaving markers in the worktree.
- Structural changes to `~/.claude` (moving or removing the skills directory itself) are for a human to run outside agent sessions; hub only manages entries inside it.
- All destructive subcommands have `--dry-run` (`sync`) or a read-only preview (`update --check`, `census`, `doctor` without `--fix`).
- Corrupt `hub.toml` / `hub.lock.json` → exit 2 with the parser error, never a traceback.

## Dogfooding

This repository *is* a skill. An instance vendors it as `skills/skill-hub/` (`origin = vendor`, source = this repo), so the tool updates itself through `hub update skill-hub`, and local patches to the tool survive upstream updates exactly like any other skill. `hub diff skill-hub` shows what you changed — ready-made PR material.

## Migrating from a messy setup

The playbook that produced this tool (two same-remote clones, a symlink loop, 104 cache links, 373 entries → 224 clean skills):

0. **Seal the evidence** (outside agent sessions): `tar` the directories; in each old git clone `git switch -c rescue/<dir>-<date> && git add -A && git commit && git push -u origin HEAD`. Symlinks are stored as their target text, so even broken ones commit — nothing is lost.
1. **Census**: `hub census --out census.json`; review the `CONFLICT` groups (usually a couple of stale copies).
2. **Build the instance**: `hub init` (or clone your instance repo and scaffold on a branch), vendor skill-hub, `hub adopt --from-census census.json --skip <externals>`, resolve conflicts with explicit `hub adopt <path>`, write profiles and `devices/<id>.toml`, `hub sync --dry-run`, commit, push.
3. **Rewire** (outside agent sessions!): move each old directory aside (`mv ~/.claude/skills ~/.claude/skills.pre-hub`), recreate it empty, copy back protected dirs (`synced/`, `.system`), `hub onboard --with-hook --with-launchd`, `hub sync`.
4. **Verify**: agents list the skills once (no doubled context), `hub doctor` is green, `hub status` shows the expected enabled count.
5. After a week or two, delete the `*.pre-hub` directories. Keep the tarball and the rescue branches.

Second machine: seal (step 0), clone the instance, `hub census` for anything unique to that machine, adopt it, rewire, verify.

## Command reference

| command | notes |
|---|---|
| `hub init` | scaffold an instance at `$SKILL_HUB_ROOT` (default `~/skills-hub`), `git init` if needed |
| `hub onboard [--device NAME] [--no-bin] [--with-hook] [--with-launchd]` | register this machine; idempotent |
| `hub census [--dirs D…] [--out FILE]` | classify existing agent directories, read-only |
| `hub adopt PATH [--name N] [--origin self\|vendor] [--source SPEC]` | absorb a directory |
| `hub adopt --from-census FILE [--skip a,b]` | batch adoption |
| `hub sync [--dry-run] [--no-pull] [--no-push]` | the daily verb |
| `hub install SPEC [--name N] [--profile P] [--no-git]` | vendor from `git+URL#ref=…&subdir=…` or a local path |
| `hub update [NAME…] [--all] [--check] [--continue] [--no-git] [--write-attention]` | freshness check / 3-way merge / finalize |
| `hub doctor [--fix] [--write-attention]` | health checks |
| `hub status` | device, enabled count, patched skills, git state, last sync |
| `hub diff NAME` | `diff -ru upstream/NAME skills/NAME` (in `data.diff` with `--json`) |
| `hub lock-merge BASE OURS THEIRS` | git merge driver for `hub.lock.json` |
| `hub selftest` | end-to-end scenario in a temp sandbox; never touches the real instance |

Environment: `SKILL_HUB_ROOT` (instance path), `SKILL_HUB_DEVICE_FILE` (default `~/.agents/device-id`).

## Limitations and roadmap

- No `uninstall` yet: delete `skills/<name>`, `upstream/<name>` and the `[skills.<name>]` block; the next `sync` drops the lock entry and removes the links.
- `update` has no `--dry-run`/`--abort`; preview with `--check` and `hub diff`, revert with git.
- Source types are `git+…` and local paths; installer registries (ClawHub-style) are planned.
- Doctor checks 10 and 11 are reserved (see table).
- Automation is macOS `launchd`; Linux users schedule the two commands themselves. Windows is not supported (symlink semantics).
- Cloud/hosted agent sessions do not read local directories; expose skills to them by their own mechanisms.

## Development

```bash
./scripts/hub selftest      # sandboxed: init → onboard → install → sync → cleanup rules → 3-way merge → conflict → continue → doctor → lock-merge
```

The whole tool is one file, standard library only. Contributions that keep it that way are welcome. MIT licensed.
