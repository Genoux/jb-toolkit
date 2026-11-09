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
- 2026-11-08: services ship as public ghcr.io images built in CI (R8), not
  built on the server. The server then needs no git access, no build
  toolchain and no registry credentials; secrets stay in the server's
  environment and never enter an image.

## Project rules

- After changing `templates/managed/` or `RULES.md` in a way projects must
  pick up, bump `VERSION`; `jb check` then flags projects on the older one.
