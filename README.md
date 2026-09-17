# skill-hub

**One canonical library for all your agent skills — across agents, devices, and upstreams. Maintained by the agents themselves.**

skill-hub is a small, dependency-free CLI (one Python 3.11+ file, standard library only) plus a `SKILL.md` that teaches your coding agent how to run it. It turns the sprawl of `~/.claude/skills`, `~/.agents/skills`, `~/.codex/skills`, plugin caches, `npx` installers, cloned repos and hand-written one-offs into **one git repository** from which every agent's skill directory is rendered.

- [The problem](#the-problem)
- [The doctrine](#the-doctrine)
- [Four words you need](#four-words-you-need)
- [How it works](#how-it-works)
- [Quick start](#quick-start)
- [Concepts](#concepts): [instance layout](#instance-layout) · [`hub.toml`](#hubtoml--the-manifest) · [device files](#deviceidtoml--per-device-selection) · [lockfile](#hublockjson--provenance) · [origins](#three-origins-self-vendor-external) · [why vendor, not submodules](#why-vendor-not-submodules) · [selection algorithm](#selection-algorithm)
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
- Installers leave `foo-1.0.1` directories that no one updates; hand copies of upstream skills fork and can never be merged back.
- A second machine is a re-do of all of the above.

## The doctrine

1. **One canonical git repo** (the *instance*, default `~/skills-hub`) holds every skill as a real, vendored directory. Nothing else is a source of truth.
2. **Render targets are not data.** `~/.claude/skills` and friends (the directories agents read — called *render targets* below) contain only per-skill symlinks or managed copies. They can be deleted and rebuilt with one command.
3. **Two layers, declared.** The library (`skills/`) can hold hundreds of skills; `devices/<id>.toml` says which of them *this machine* renders. Installing puts a skill in the library — enabling it is a separate, per-device decision. One file per device, so manifests never conflict between machines.
4. **Agents do the upkeep.** Every command is idempotent, has `--json` output, exit-code semantics and a preview mode. The bundled `SKILL.md` gives the agent the full routine. Humans state intent; agents run the plumbing.

Three iron rules fall out of this:

| # | Rule |
|---|---|
| 1 | Real skill directories (*entities*) exist **only** inside the instance's `skills/`. Render targets hold links or managed copies. |
| 2 | **Never** symlink anything under `~/.claude/plugins/cache/`. Plugin skills are the plugin system's job; a link into the cache is treated as damage and removed on sight. |
| 3 | hub only deletes what it created (plus cache links and its own dead links). Everything else is *foreign*: reported, never touched. |

## Four words you need

| word | meaning |
|---|---|
| **instance** | your library repo, default `~/skills-hub` (override with `$SKILL_HUB_ROOT`; hub never depends on the current directory) |
| **profile** | a named group of skills in `hub.toml`, e.g. `base`, `dingtalk` |
| **device file** | `devices/<id>.toml` — which profiles this machine enables and into which render targets |
| **origin** | `self` (yours, no upstream) or `vendor` (has a git upstream; a pristine copy is kept so upstream updates merge with your local patches) |

Two render targets are scaffolded by default: `~/.claude/skills` (Claude Code) and `~/.agents/skills` (`agents-std` — the emerging cross-agent convention read by Codex, Cursor, opencode and others). Add more, or remove one, in `hub.toml`.

## How it works

```
 upstream sources                  the instance (git repo)                 render targets
 ─────────────────                 ───────────────────────                 ──────────────
 GitHub repo / subdir  ─install─▶  skills/<name>/     ◀── the entities ────┬─▶ ~/.claude/skills/<name>  (symlink)
 local directory       ─adopt───▶  upstream/<name>/   ◀── pristine copies  ├─▶ ~/.agents/skills/<name>  (symlink)
 existing mess         ─census──▶  hub.toml           ◀── sources/profiles ├─▶ ~/.codex/skills/<name>   (copy, optional)
                                   devices/<id>.toml  ◀── this machine     └─▶ any directory you declare
                                   hub.lock.json      ◀── provenance
                                            ▲  ▼
                                   git pull --rebase / push  ◀──▶  any git remote (or none)
```

`hub sync` is the only daily verb: commit local changes → pull → **materialize** (place links, clean up what hub owns) → push. `hub update` merges upstream changes into your possibly-patched copy using `upstream/` as the merge base. `hub doctor` audits the whole thing. A `SessionStart` hook surfaces pending work to the agent; `launchd` runs the safe parts unattended.

## Quick start

Requirements: Python 3.11+, git, macOS or Linux (symlinks; the `launchd` automation is macOS-only — on Linux schedule the same two commands with cron/systemd).

**1. Create an instance and vendor the tool into it (bootstrap)**

```bash
git clone https://github.com/xiaolinfrank/skill-hub.git ~/repos/skill-hub
~/repos/skill-hub/scripts/hub init          # scaffolds ~/skills-hub (or $SKILL_HUB_ROOT)
~/repos/skill-hub/scripts/hub adopt ~/repos/skill-hub --name skill-hub --origin vendor \
    --source "git+https://github.com/xiaolinfrank/skill-hub#ref=main"
```

This is the bootstrap: the clone's script builds the library, then the tool is absorbed into the library as `skills/skill-hub/` (origin `vendor`, upstream = this repo). From here on **that copy is the one you run and the one you may patch**; `~/repos/skill-hub` can be deleted. Later, `hub update skill-hub` pulls new releases and keeps your patches.

**2. Register this machine**

```bash
~/skills-hub/skills/skill-hub/scripts/hub onboard --device my-laptop
```

`onboard` writes the device id to `~/.agents/device-id`, registers the lockfile merge driver in this clone, creates `devices/my-laptop.toml` (enabling profile `base`, which already contains `skill-hub`), and links `hub` into `~/.local/bin`. It does **not** edit your shell `PATH`: if `which hub` prints nothing, add `export PATH="$HOME/.local/bin:$PATH"` to your shell rc (or pass `--no-bin` and alias it yourself). Add `--with-hook --with-launchd` for automation (see [The agent contract](#the-agent-contract)).

**3. Render**

```bash
hub sync
```

By default this links every enabled skill into both `~/.claude/skills` and `~/.agents/skills`. Want only one? Edit `agents` in your device file.

**If this machine already has skills** (it probably does): the first `sync` lists every existing entry in those directories as *foreign* — that is expected and **nothing of yours is touched**. To fold them into the library, run `hub census` → `hub adopt --from-census` (see [Migrating from a messy setup](#migrating-from-a-messy-setup)); `hub sync --dry-run` previews any run.

**4. Bring in a skill**

```bash
hub install "git+https://github.com/anthropics/skills#ref=main&subdir=skills/canvas-design" --profile base
hub sync
```

`install` without `--profile` only adds the skill to the library; nothing renders until a profile or your device file's `extra` list names it. To enable something on this machine only, without touching shared profiles, add it to `extra` in `devices/my-laptop.toml`.

**Join an existing instance from another machine**

```bash
git clone <your-instance-remote> ~/skills-hub
~/skills-hub/skills/skill-hub/scripts/hub onboard --device my-desktop --with-hook --with-launchd
hub sync                      # then edit devices/my-desktop.toml to taste, sync again
```

Because everything is vendored, `clone` is the whole restore.

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

Skill names must match `^[a-z0-9]+(-[a-z0-9]+)*$` (some agents enforce this). `adopt` and `install` normalize names: lower-case, `_`/`.`/space → `-`, version suffixes like `-1.0.1` stripped; `adopt` records the original directory name under `aliases` in the lockfile when it differs. Nested `.git` directories are never copied in.

Two names inside render targets are always left alone: `synced/` (where claude.ai places skills it syncs to your machine) and `.system` (Codex's own directory). Add others per target with `protect`.

### `hub.toml` — the manifest

```toml
[hub]
schema = 1
default_agents = ["claude", "agents-std"]   # agents that receive every enabled skill by default

[agents.claude]                 # a render target
target  = "~/.claude/skills"
mode    = "symlink"             # symlink | copy
protect = ["synced"]            # entries hub must never touch (synced/ and .system are always protected)

[agents.agents-std]             # ~/.agents/skills: cross-agent convention (Codex, Cursor, opencode, …)
target = "~/.agents/skills"
mode   = "symlink"

[agents.codex-legacy]           # example: an agent that only reads its own dir or cannot follow symlinks
target  = "~/.codex/skills"     # → managed copies instead of links; add it to a device's `agents`
mode    = "copy"
protect = [".system"]

[profiles]                      # named sets; patterns are shell globs over skills/ names
base     = ["skill-hub", "crawl", "search"]
dingtalk = ["dingtalk-*"]

[skills.domain-modeling]        # vendor-origin skills get a block; undeclared skills are self
origin = "vendor"               # documentation only — the lockfile decides (see below)
source = "git+https://github.com/mattpocock/skills#ref=main&subdir=skills/engineering/domain-modeling"

[skills.onepassword]
origin = "vendor"
source = "git+https://github.com/example/skills#ref=main&subdir=1password"
pin    = true                   # frozen: `update` skips it
# agents = ["claude"]           # optional: restrict this skill to some agents (default: default_agents)
```

Which agent gets a skill: an agent listed in `default_agents` receives every enabled skill unless a `[skills.<name>]` block narrows it with `agents = [...]`. An agent **not** in `default_agents` receives only skills whose block lists it — so when you add a new render target you normally add it to `default_agents` too. If your agent reads a directory of its own or does not follow symlinks, declare a second target with `mode = "copy"` (like `codex-legacy`) and add it to the device's `agents`.

hub writes to `hub.toml` in exactly two ways: `install`/`adopt` **append** a `[skills.<name>]` block (append-only text, no TOML serializer round-trip), and `install --profile P` inserts the name into the single-line array of profile `P` inside the `[profiles]` section. Everything else — reorganizing profiles, removing blocks — is a text edit by you or your agent; hub validates with `tomllib` on the next read and refuses to run (exit 2) on syntax errors. Note that `origin` in `hub.toml` is informational: the lockfile's `origin` is what `update` consults, and it is set by `install` / `adopt --origin vendor`. Hand-placing a directory into `skills/` makes it `self`.

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
      "vendored_at": "2026-09-05T10:12:00+08:00",
      "updated_at":  "2026-09-12T08:00:00+08:00"
    },
    "onepassword": { "origin": "vendor", "source": "…", "aliases": ["1password-1.0.1"], "adopted_at": "…" },
    "crawl": { "origin": "self", "local_hash": "sha256:…", "adopted_at": "…" }
  }
}
```

`patched` is `upstream_hash != local_hash` — you changed a vendored skill. Hashes are content hashes of the tree (symlinks hash as their target text, `.git` and `.DS_Store` ignored). `sync` refreshes `local_hash`/`patched` for every skill (stamping `updated_at` when content changed) and drops entries for skills that no longer exist.

Two machines both touching the lockfile would be git's classic JSON-conflict hell, so `.gitattributes` declares `hub.lock.json merge=hub-lock` and `onboard` registers the driver (`hub lock-merge %O %A %B`) in each clone: keys are unioned, same-key conflicts resolved by newer `updated_at`. Doctor check 13 verifies the driver is registered.

### Three origins: self, vendor, external

| origin | what it means | `upstream/` | `update` |
|---|---|---|---|
| **self** | you wrote it, or no upstream can be traced. The default; zero ceremony. | no | n/a |
| **vendor** | has a git source. Pristine snapshot kept; local patches survive upstream updates via 3-way merge. | yes | yes |
| **external** | declared in a device file only; content never enters the instance. Rendered as a symlink to the absolute path. For things that must not be in any repo. | no | n/a |

### Why vendor, not submodules

Three reasons the library copies skills instead of nesting checkouts:

1. Render targets must contain plain directories — several agents choke on nested `.git`, and doctor check 4 treats one as a defect. Vendored copies also make `git clone` a complete restore with no second step.
2. Local patches need a stable merge base. `upstream/<name>` is that base, kept automatically; submodules make "I changed two lines of an upstream skill" a detached-HEAD chore across machines.
3. `hub diff <name>` shows exactly what you changed against pristine upstream — ready-made PR material.

### Selection algorithm

For this device: `enabled = ⋃ profiles (globs expanded over skills/) ∪ extra − disable`, validated so every name exists. For each agent in the device's `agents`: `want = {s ∈ enabled | agent ∈ (skills[s].agents or default_agents)}`. Externals are added per their own `agents` list.

## Materialization rules

Materialization is the second half of `hub sync` (and the only part that touches render targets). In symlink mode it is stateless — ownership is decided by *shape*, not by a database; copy-mode targets keep a small `.hub-state.json` listing the directories hub created.

**Placement** (for every wanted skill, in every target):

| the slot `target/<name>` currently holds | symlink mode | copy mode |
|---|---|---|
| nothing | create link | copy |
| a link that already points at `skills/<name>` | keep | replace the link with a copy (it is ours) |
| a link elsewhere into the instance's `skills/`, into the plugin cache, or a dead link | replace | blocked (report) |
| a directory hub copied earlier (in `.hub-state.json`) | — | refresh if content differs |
| **anything else** — a real directory, a file, a link to somewhere of yours | **blocked**: reported as foreign, `needs_attention`, exit 1 | same |

**Cleanup** (for every entry in every target, except `protect` names, `synced/`, `.system`, `.hub-state.json`):

| entry | action |
|---|---|
| link resolving under `~/.claude/plugins/cache/` | **remove** (rule 2 — dead or alive) |
| link exactly of the shape hub creates (`→ skills/<name>` with the same name), skill no longer enabled | remove |
| link to a *still-declared* external, external not enabled for this agent | remove (delete the `[[external]]` block and the link becomes foreign: kept, reported) |
| dead link that pointed into the instance's `skills/` | remove |
| dead link pointing anywhere else (unmounted volume, someone else's tool) | **keep**, report |
| link into `skills/` but not hub-shaped (your own alias or deep link) | keep, silently |
| any other link | keep, report as foreign |
| directory hub copied (copy mode, in `.hub-state.json`), no longer enabled | remove |
| any other directory or file | keep, report as foreign |

`hub sync --dry-run` runs the same decision tree (including the content-hash check in copy mode) and prints `[dry] …` for each action it would take.

## `hub sync`

```
0. unfinished update in .hub-pending/?  → only materialize runs; commit/pull/push and the lock refresh
   are skipped, the run exits 1 (finish with `hub update --continue`)
1. commit drift: any uncommitted change in the instance — whoever made it — becomes
   "sync(<device>): auto-commit drift"
2. `git pull --rebase --autostash`  (skipped with no remote or --no-pull)
      real rebase conflict → `git rebase --abort` (worktree stays usable), recorded in NEEDS_ATTENTION.md, exit 1
      other failure (offline, auth) → warning, continue with local state, push skipped this round
3. materialize (see above)
4. `git push`  (skipped with no remote or --no-push; a failed push is a warning, retried next round)
5. refresh lockfile hashes → commit "sync(<device>): refresh lock" → push
6. stamp .logs/last_sync
```

**Machine-specific symlinks are never committed.** A symlink inside the library whose target resolves outside it dangles on every other device, so sync leaves it on disk, keeps it out of the drift commit, and names it in a warning. Links that stay inside the library — the relative ones a skill ships — are committed normally. A symlinked directory is checked as a link and never descended into, so a self-referential one cannot loop. If such a link is already in history (an older sync, or a hand commit), sync escalates instead: it cannot untrack it for you, because the machine where the link resolves is using it. Untrack it there, so that working copy survives.

Any `needs_attention` item raised during a sync is also appended (deduplicated) to `NEEDS_ATTENTION.md`, so unattended runs leave a trail the next session can pick up. A change you make on machine A — even by editing through the symlink inside an agent session — is committed by A's next sync and arrives at B on B's next sync. Two machines editing the same file between syncs is a rebase conflict: aborted, recorded, resolved semantically by the agent in a session. hub never force-pushes.

## Install and update (3-way merge)

**Sources.** `git+<url>#ref=<branch|tag>&subdir=<path>` or a local directory. `install` fetches the source (see **Fetching** below), takes the subdir, copies it to both `upstream/<name>` and `skills/<name>` (minus `.git`), appends the manifest block, records `upstream_commit` and hashes, and commits just those paths. The name is `--name` if given; otherwise the subdir's basename, else the repository name (git source without subdir), else the local directory's basename — then normalized. Reinstalling a name that exists in `skills/` or as a `[skills.<name>]` block is refused (exit 2).

**Fetching.** A git source is cloned `--depth 1 --filter=blob:none --sparse` and only the wanted `subdir` is checked out (cone mode), so vendoring one skill out of a large monorepo downloads that skill, not the repository. Each `(url, ref)` is cloned **once per command** and reused, with every subdir asked for in that command added to the same sparse set: `hub update --all` across 40 skills that share one upstream does one fetch, not 40. If the server refuses partial clone, or a path cone mode will not take shows up, hub silently falls back to a plain shallow clone of the whole tree. Clones live in a temp directory removed when the command ends, so a later run always sees new upstream commits.

**Freshness.** `hub update --check` (all vendor skills, or the names given) runs `git ls-remote <url> <ref>` and compares the exact `refs/heads/<ref>` (or the peeled tag) with `upstream_commit`; local-path sources are compared by content hash. Entries without a recorded commit (e.g. adopted from a plain copy) are reported as `unknown_freshness`, not as updatable, so a weekly check never spams. `--write-attention` records updatable skills in `NEEDS_ATTENTION.md`.

**Apply.** `hub update <name…>` (or `--all`; names are required otherwise) fetches the new snapshot into `.hub-pending/<name>` and merges into `skills/<name>` file by file with `base = upstream/<name>`, `ours = skills/<name>`, `theirs = pending`:

| situation | result |
|---|---|
| upstream added a file | added |
| upstream deleted, you never touched it | deleted |
| both sides deleted | nothing |
| upstream deleted, you modified it | conflict |
| you deleted, upstream modified | conflict |
| only upstream changed | taken |
| only you changed / identical | kept |
| both changed (text) | `git merge-file` — clean or `<<<<<<<` markers left in place |
| both changed (binary / unmergeable) | your version kept, upstream saved as `<file>.upstream` sidecar |
| a symlink changed on both sides | conflict (links are never text-merged and never written through) |

No conflicts → `upstream/` replaced by the new snapshot, lockfile updated (`patched` recomputed), that skill committed. Conflicts → that skill is not committed, the list goes to `NEEDS_ATTENTION.md`, exit 1; other skills in the same run are still processed and committed individually (commits are path-scoped, so conflict markers never ride along). The agent resolves markers / sidecars and runs `hub update <name> --continue` (no name = every pending update), which refuses while any marker or sidecar remains. `pin = true` skips a skill entirely. Re-running `update` on a skill with a pending merge is refused until `--continue`.

Self-origin skills have no update concept: edit, `hub sync`, done.

## Adopt and census

`hub census [--dirs …] [--out file]` scans render targets (default: `~/.claude/skills`, `~/.agents/skills`, `~/.codex/skills`) read-only — it needs no instance — and labels every entry:

| label | meaning | `adopt --from-census` |
|---|---|---|
| `entity` / `entity-versioned` | real directory (the latter with a version suffix) | adopt (name normalized) |
| `nested-git` | real directory that is itself a git repo | adopt without `.git`; remote + current branch recorded as vendor source |
| `external-link` | link to a directory outside the scanned dirs | adopt from the target; if it lives in a git checkout, remote + branch + subdir recorded as vendor source |
| `cross-link` | link to an entity in another scanned dir | deduplicated by real path |
| `cache-link` | link into the plugin cache | dropped |
| `broken` | dead link | dropped |
| `hub-managed` / `internal` / `file` | already ours / dotfiles / stray files | ignored |

Same-name candidates are compared by content hash: identical copies collapse to one (preferring the copy that carries provenance); different contents are flagged `CONFLICT` and skipped, so a human or agent decides with `hub adopt <path> --name <other>`. `--skip a,b` excludes names (e.g. ones you want as externals). Adoption persists the lockfile after every skill, so an interrupted run loses nothing.

`hub adopt <path>` for a single directory auto-detects provenance: inside a git checkout with a remote → vendor, with the checkout's current branch as `ref` (`--origin self` to override, `--source` to set the spec explicitly); otherwise self.

## Doctor

`hub doctor [--fix] [--write-attention]` runs these checks (numbers are stable identifiers used in output):

| # | check | `--fix` |
|---|---|---|
| 1 | dead links that pointed into the instance's `skills/` or to a declared external | remove |
| 2 | links into the plugin cache | remove |
| 3 | foreign links, dead foreign links, foreign directories | report only |
| 4 | nested `.git` under `skills/` or `upstream/` | report |
| 5 | non-compliant skill names | report |
| 6 | profile patterns matching nothing (skills in no profile are listed as `library_only` info, not an issue) | report |
| 7 | lockfile hashes stale (edited but not synced) | refresh hashes + commit |
| 8 | uncommitted / unpushed / behind remote | commit (unless an update is pending) |
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
- hub never force-pushes. Commits made by `install`/`update` are scoped to that skill's paths, and `sync`/`doctor --fix` refuse to commit while an update is pending, so conflict markers never leave the machine.
- A conflicting rebase is aborted, not left half-applied in the worktree.
- Structural changes to `~/.claude` (moving or removing the skills directory itself) are for a human to run, not for an agent to do inline; hub only manages entries inside it. Other sessions need not be stopped: agents hot-reload the skills directory, and a same-filesystem `mv` keeps running scripts alive on their open handles.
- Previews: `sync --dry-run`, `update --check`, `census`, `doctor` without `--fix`.
- Corrupt `hub.toml` / `hub.lock.json` → exit 2 with the parser error, never a traceback.

## Dogfooding

This repository *is* a skill. An instance vendors it as `skills/skill-hub/` (`origin = vendor`, source = this repo), so the tool updates itself through `hub update skill-hub`, and local patches to the tool survive upstream updates exactly like any other skill. `hub diff skill-hub` shows what you changed — ready-made PR material.

## Migrating from a messy setup

The playbook that produced this tool (two same-remote clones, a symlink loop, 104 cache links, 373 entries → 224 clean skills):

0. **Seal the evidence** (outside agent sessions): `tar` the directories; in each old git clone `git switch -c rescue/<dir>-<date> && git add -A && git commit && git push -u origin HEAD`. Symlinks are stored as their target text, so even broken ones commit — nothing is lost.
1. **Census** (read-only, no instance needed yet): `~/repos/skill-hub/scripts/hub census --out census.json`; review the `CONFLICT` groups (usually a couple of stale copies).
2. **Build the instance**: Quick start step 1 (init + vendor skill-hub), then `hub adopt --from-census census.json --skip <names-you-want-as-externals>`, resolve conflicts with explicit `hub adopt <path>`, write profiles and `devices/<id>.toml`, `hub sync --dry-run`, commit, push.
3. **Rewire** (you run this, not the agent): clone and `hub onboard --with-hook --with-launchd` first, then move each old directory aside (`mv ~/.claude/skills ~/.claude/skills.pre-hub`), recreate it empty, copy back the protected directories (`synced/` for Claude, `.system` for Codex — see [Instance layout](#instance-layout)), `hub sync`. Doing the clone and onboard before the move keeps the window where skills are missing down to the sync itself.
4. **Verify**: agents list the skills once (no doubled context), `hub doctor` is green, `hub status` shows the expected enabled count.
5. After a week or two, delete the `*.pre-hub` directories. Keep the tarball and the rescue branches.

Second machine: seal (step 0), clone the instance, `hub census` for anything unique to that machine, adopt it, rewire, verify.

## Command reference

| command | notes |
|---|---|
| `hub init` | scaffold an instance at `$SKILL_HUB_ROOT` (default `~/skills-hub`), `git init` if needed |
| `hub onboard [--device NAME] [--no-bin] [--with-hook] [--with-launchd]` | register this machine; idempotent |
| `hub census [--dirs D…] [--out FILE]` | classify existing render targets, read-only |
| `hub adopt PATH [--name N] [--origin self\|vendor] [--source SPEC]` | absorb a directory |
| `hub adopt --from-census FILE [--skip a,b]` | batch adoption |
| `hub sync [--dry-run] [--no-pull] [--no-push]` | the daily verb |
| `hub install SPEC [--name N] [--profile P] [--no-git]` | vendor from `git+URL#ref=…&subdir=…` or a local path |
| `hub update NAME… \| --all [--no-git]` | 3-way merge upstream changes (names or `--all` required) |
| `hub update --check [NAME…] [--write-attention]` | freshness check (all vendor skills by default) |
| `hub update [NAME…] --continue` | finalize after resolving conflicts (all pending by default) |
| `hub doctor [--fix] [--write-attention]` | health checks |
| `hub status` | device, enabled count, patched skills, git state, last sync |
| `hub diff NAME` | `diff -ru upstream/NAME skills/NAME` (in `data.diff` with `--json`) |
| `hub lock-merge BASE OURS THEIRS` | git merge driver for `hub.lock.json` |
| `hub selftest` | end-to-end scenario in a temp sandbox; never touches the real instance |

Environment: `SKILL_HUB_ROOT` (instance path), `SKILL_HUB_DEVICE_FILE` (default `~/.agents/device-id`).

## Limitations and roadmap

- No `uninstall` yet: delete `skills/<name>`, `upstream/<name>`, the `[skills.<name>]` block, and the name from any `[profiles]` array or device `extra` list; the next `sync` drops the lock entry and removes the links.
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
