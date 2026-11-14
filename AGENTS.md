# AGENTS.md

How jb-toolkit is built and why. The rules it ships are `RULES.md`; this file
is about changing the toolkit itself. Read `PRODUCT.md` first.

## Stack

- bash, POSIX sh for the git hooks, jq — why: projects are Rust, JS and
  static HTML, so the toolkit cannot live in any one ecosystem, and these are
  already on every machine (and every CI runner).

## Layout

- `RULES.md` — the standard. Imported live from `~/.claude/CLAUDE.md`.
- `bin/jb` — the agent's tool: install, session, new, sync, check, bump, tag.
- `skills/` — linked into `~/.claude/skills` by install; edits apply at once.
- Third-party skills: `jb sync` reads the list from ib-toolkit's
  `packages/standards/external-skills.txt` on GitHub (main branch), downloads
  each skill into the project's `.agents/skills/` and links it into
  `.claude/skills/`. Edit the list there, never a copy here: one list keeps
  personal and work projects on the same skills.
- `templates/managed/` — overwritten in projects by `jb sync`.
- `templates/project/` — created once in a project, then the project's.
- `templates/stacks/<stack>/` — starting `scripts/check` and `.gitignore` lines.
- `scripts/check` — syntax, shellcheck when installed, and an end-to-end run
  in a temp dir: new project, `jb check`, silent `jb session`, hooks refusing,
  `jb bump`.

## Decisions

- 2026-11-06: scaffold + sync + global rules, not a package. A package would
  bind the standard to one ecosystem; a copy-once template drifts.
- 2026-11-06: the owner runs only `jb install`. Everything else is triggered
  by the SessionStart hook (`jb session`), the skills and `RULES.md` R0. New
  features follow the same shape: never a command the owner has to type.
- 2026-11-06: `put_block` passes the block through the environment, because
  macOS (BSD) awk refuses newlines in `awk -v`.
- 2026-11-06: `jb session` prints nothing for a clean project; its output
  becomes every session's context, so noise there costs every session.

- 2026-11-14: third-party skills are downloaded at their latest version on
  every sync, not vendored (owner's choice), with plain `git` (sparse,
  depth 1) rather than the `skills` npm CLI, so sync needs nothing beyond git.
  Measured: the full list (27 skills) takes about 17 s and 41 MB, 37 MB of
  it taste-skill's images. They go per project, not into `~/.claude/skills`,
  so a project's skills are the same for whoever runs sync.

- 2026-11-14: skills live in `.agents/skills/` (Codex, Cursor, Gemini CLI,
  Copilot, OpenCode) with a link in `.claude/skills/`, because Claude Code
  reads only its own folder; a copy replaces the link where symlinks fail.
- 2026-11-14: `jb install` writes a launcher to `~/.local/bin/jb`, not a
  symlink: Git Bash on Windows turns symlinks into copies, and a copied `jb`
  cannot find the toolkit. CI runs `scripts/check` on Linux, macOS and
  Windows; `bin/jb` must stay bash 3.2 compatible (macOS).

## Project rules

- After changing `templates/managed/` or `RULES.md` in a way projects must
  pick up, bump `VERSION`; `jb check` then flags projects on the older one.
