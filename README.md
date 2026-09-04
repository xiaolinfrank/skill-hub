# skill-hub

**One canonical library for all your agent skills — across agents, devices, and upstreams.**

Agent skills today end up scattered: `~/.claude/skills`, `~/.agents/skills`,
`~/.codex/skills`, plugin caches, `npx` installers, cloned GitHub repos,
hand-written one-offs, and a web of symlinks that silently rots (plugin cache
directories rotate their hashes and take your symlinks down with them).

skill-hub applies a simple doctrine:

1. **One canonical git repo** (default `~/skills-hub`) holds every skill as a
   real, vendored entity. `upstream/` keeps pristine snapshots so *upstream
   updates merge cleanly with your local patches* (true 3-way merge).
2. **Discovery directories are render targets, not data.** `~/.claude/skills`
   and friends contain only per-skill symlinks (or managed copies) and can be
   rebuilt from scratch at any time. Anything pointing into a plugin cache is
   treated as damage and removed on sight.
3. **Declarative selection.** `hub.toml` declares sources and profiles;
   `devices/<id>.toml` declares what each machine enables — one file per
   device, so manifests never conflict across machines.
4. **Agents do the upkeep.** Every command is idempotent, speaks `--json`,
   has exit-code semantics and `--dry-run`. A `SessionStart` guard surfaces
   pending items; the bundled `SKILL.md` teaches your agent the full routine.
   Humans state intent; agents run the plumbing.

## Install

```bash
git clone https://github.com/xiaolinfrank/skill-hub ~/repos/skill-hub
~/repos/skill-hub/scripts/hub init        # scaffold ~/skills-hub
~/skills-hub/skills/... # or vendor skill-hub into the instance: see below
hub onboard                               # device id + merge driver + PATH link
hub sync
```

Requires Python 3.11+ and git. No third-party dependencies.

## Sync transport: git, not GitHub

`hub sync` only needs *a* git remote — or none:

- **Single machine**: no remote configured → sync skips pull/push. Zero accounts.
- **Your own box as the remote**: `git init --bare skills-hub.git` on any
  SSH-reachable machine (Tailscale works great). Data never leaves your devices.
- **GitHub/GitLab/...**: `gh repo create --private` if you like hosted remotes.

There is deliberately **no** "sync a live worktree via Dropbox/iCloud" mode —
file-sync tools corrupt `.git` under concurrency, and everything valuable here
(3-way merges, history, rescue branches) lives on git.

## Dogfooding

This repository *is* a skill. A hub instance vendors it as
`skills/skill-hub/` (`origin=vendor`), so the tool updates itself through its
own `hub update skill-hub` — local patches to the tool survive upstream
updates the same way as for any other skill.

## Commands

| command | what it does |
|---|---|
| `hub init` | scaffold a new instance |
| `hub onboard` | register this device (id, merge driver, PATH link; `--with-hook --with-launchd` for automation) |
| `hub census` | classify the current mess (entities, broken/cache/cross links, nested git…) |
| `hub adopt` | absorb an existing directory (or a whole census) into the library |
| `hub sync` | commit drift → pull --rebase → materialize links → push |
| `hub install` | vendor a skill from `git+URL#ref=..&subdir=..` or a local path |
| `hub update` | 3-way merge upstream changes into your (possibly patched) copy |
| `hub doctor` | 13 health checks; `--fix` handles the safe ones |
| `hub status` / `hub diff` | what's enabled, what's patched, what changed |
| `hub selftest` | full sandboxed end-to-end test of the tool itself |

## License

MIT
