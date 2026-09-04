---
name: skill-hub
description: >
  Manage the personal skills library (default ~/skills-hub) end to end:
  install / update / sync / doctor / adopt / enable-disable per device.
  Use whenever the user says things like "install a skill", "update my skills",
  "sync skills", "a skill is broken / duplicated / not loading", "put skill X
  on my other machine too", "adopt this directory", "clean broken symlinks",
  "处理 hub 待办", or names `hub sync` / `hub doctor` directly. Day-to-day
  maintenance is driven through this skill — the user never hand-edits the
  skills directories.
---

# skill-hub

## 0. Ground truth

- Canonical library: `$SKILL_HUB_ROOT` (default `~/skills-hub`), a git repo.
  Entities live in `skills/`, pristine upstream snapshots in `upstream/`.
- CLI: `hub <cmd> --json` (in `skills/skill-hub/scripts/hub`; also linked at
  `~/.local/bin/hub`). ALL operations go through it — never hand-craft
  `ln`/`rm` in agent skill directories.
- Device identity comes from `~/.agents/device-id` (written by `hub onboard`).
- Iron rules: entities exist only inside the library; never touch `synced/`
  or `.system`; never symlink anything under `plugins/cache`; structural
  changes to `~/.claude` (moving/removing the directory itself) are done
  OUTSIDE agent sessions via a generated script the user runs; never force
  push the hub repo.

## 1. Intent → command map

| User intent | Action |
|---|---|
| install X / bring X in | `hub install <source> [--profile P]` — source is `git+URL#ref=..&subdir=..` or a local path; if unclear, search or ask one question |
| update X / update everything | `hub update [X|--all]`; on conflicts follow §3, then `hub update X --continue` |
| sync / I changed things on another machine | `hub sync`; exit 1 → read `NEEDS_ATTENTION.md` and handle |
| X on device B too / only on this machine | edit `devices/<id>.toml` (`extra` / `disable`) or `hub.toml` profiles → `hub sync` |
| something feels off / checkup | `hub doctor` (`--fix` auto-fixes safe items only; confirm the rest with the user) |
| adopt this directory | `hub adopt <path>` (origin auto-detected: git remote → vendor, else self) |
| what did I change vs upstream | `hub diff X` |
| new machine | three commands: `git clone <instance-repo> ~/skills-hub` → `hub onboard` → `hub sync` |
| 处理 hub 待办 | work through `NEEDS_ATTENTION.md` item by item, delete the file when done |

## 2. Output contract

Every command supports `--json`: `{ok, cmd, actions, warnings, needs_attention,
data}`. Exit codes: 0 done · 1 agent attention needed · 2 environment/config
error (report to user, do NOT retry blindly).

## 3. Conflict playbook (update / sync rebase)

1. Read `NEEDS_ATTENTION.md` to locate conflicted files.
2. Understand all three sides (base = `upstream/<name>`, ours = `skills/<name>`,
   theirs = incoming) and write a semantic merge: preserve the intent of local
   patches, absorb upstream improvements.
3. `hub update <name> --continue` (or `git rebase --continue` for sync).
4. `hub sync` to finish; commit message must state the merge decision.

## 4. Escalate to the human first (ask before acting)

- Deleting any foreign entity (something hub does not manage).
- Anything requiring force push / overwriting the remote.
- Same-name-different-content adoption choices.
- Editing `[agents.*]` structure in hub.toml, or the hub script itself.
- Touching Claude's own plugin cache.

## 5. Self-modification discipline

After changing `scripts/hub` or `scripts/guard`: `hub selftest` must pass
before committing; run any destructive subcommand of the new version with
`--dry-run` first.

## 6. Periodic routine (when guard prompts, or on request)

`hub sync` → `hub update --check --all` → `hub doctor` → if
`NEEDS_ATTENTION.md` is non-empty, handle every item → report a one-line
summary.

## 7. Verification moves

After structural changes: new session `/context` shows no doubled skills
token cost; the target agents list the skills; `hub status` is green.

## 8. Known boundaries

Cloud sessions / Cowork don't read local directories. For a skill needed in
the cloud, enable it on claude.ai (it lands in `synced/`, which hub never
touches) or expose it via the optional `.claude-plugin/marketplace.json`
facade.
