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
- `kit.json` — the baseline kit: plugins, pinned skills, tools. Moving a pin is a deliberate edit.
- `skills/` — linked into `~/.claude/skills` by install; edits apply at once.
- `templates/managed/` — overwritten in projects by `jb sync`.
- `templates/project/` — created once in a project, then the project's.
- `templates/stacks/<stack>/` — starting `scripts/check` and `.gitignore` lines.
- `scripts/check` — syntax, shellcheck when installed, and an end-to-end run
  in a temp dir: new project, `jb check`, silent `jb session`, hooks refusing,
  `jb bump`.

## Agents

`kit.json` › `agents` is the one list of agents the owner uses, each with the id
every external tool knows it by. `bin/jb` names agents only in the adapter
functions (`agent_dir`, `agent_rules_file`, `agent_hook_file`, ...), one case
per agent, keyed by that list. Enforcement does not depend on any agent: git
hooks, `scripts/check` and CI do the work; agent hooks (`jb guard`, `jb stop`,
`jb session`) are optional adapters, and every agent reads the same rules
through the project's `AGENTS.md`, which points to `.toolkit/RULES.md`. What
is wired for each, checked against the vendors' docs on 2026-11-08:

| Agent | Rules (user level) | Project rules | Hooks (jb adapter) | Skills CLI / gentle-ai / impeccable |
|---|---|---|---|---|
| claude-code | `~/.claude/CLAUDE.md` `@import` | `CLAUDE.md` imports `AGENTS.md` | `settings.json`: SessionStart, PreToolUse (Bash), Stop | claude-code / claude-code / claude |
| codex | `~/.codex/AGENTS.md` pointer | `AGENTS.md` | `hooks.json`: same three; owner trusts them in `/hooks` | codex / codex / codex |
| cursor | none documented (user rules live in its settings UI) | `AGENTS.md` | `hooks.json`: beforeShellExecution (guard), stop (followup_message); no session hook: sessionStart output is undocumented | cursor / cursor / cursor |
| gemini-cli | `~/.gemini/GEMINI.md` `@import` | `AGENTS.md`, because install adds it to `context.fileName` | `settings.json`: SessionStart (additionalContext), BeforeTool (run_shell_command), AfterAgent (deny + reason) | gemini-cli / gemini-cli / gemini |
| github-copilot (CLI) | `~/.copilot/copilot-instructions.md` pointer | `AGENTS.md` | `~/.copilot/hooks/jb-toolkit.json`: sessionStart, preToolUse, agentStop (block + reason) | github-copilot / github-copilot-cli / copilot |
| opencode | `~/.config/opencode/AGENTS.md` pointer | `AGENTS.md` | none: only JS plugins, no JSON hook system | opencode / opencode / opencode |

Copilot's user-level hooks directory (`~/.copilot/hooks/*.json`, or
`$COPILOT_HOME/hooks`) is documented for the CLI on both the "About hooks" and
the hooks-reference pages of docs.github.com (re-checked 2026-11-08); VS Code
and the cloud agent read repository hooks only, so they are not covered.

Unverified in the docs and so assumed: Gemini's `tool_input.command` argument
name and Copilot's `toolArgs` JSON string; `jb guard` reads both shapes and
passes when it finds no command.

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

- 2026-11-07: a project with its own hook manager (lefthook, husky,
  pre-commit, foreign `core.hooksPath`) keeps it: `jb sync` does not set
  `core.hooksPath`. Per-worktree config (`extensions.worktreeConfig`) was
  rejected: it needs a repo-level extension every tool must understand, for
  a case (two managers at once) nobody wants.
- 2026-11-07: exceptions live in `.jb-exceptions` at the repo root, not in
  `AGENTS.md`: `.toolkit/` is overwritten by sync, and a fenced block in
  prose is easy to break and harder to parse than one `R<id>: reason` per line.
- 2026-11-07: `.toolkit/version` reads only the top-level version (`jq` for
  package.json, the `[package]`/`[project]` table for TOML). It once read
  pnpm's version from `devEngines`; no version is a failure, never a guess.
- 2026-11-07: `jb guard` (PreToolUse) and `jb stop` (Stop) are subcommands of
  `jb`, wired into Claude Code and Codex by `jb install`. Both take the same
  hook JSON shape, so no adapter. `jb stop` blocks with
  `{"decision":"block","reason":...}` and returns early on `stop_hook_active`.
  Codex runs hooks only after the owner trusts them in `/hooks`, so
  `jb install` prints that line: this slightly widens "the owner runs only
  `jb install`" to "plus one approval in Codex".
- 2026-11-07: baseline kit skills install through the `skills` CLI pinned to
  a version, from `repo#<sha>` with `-s <names>`: the CLI records `ref` in
  `~/.agents/.skill-lock.json`, which is how `jb` tells a stale pin from a
  current one. Nothing in the kit is left to the owner:
  `rtk` is a `release` item: the pinned GitHub release asset for the
  current os-arch (linux x86_64/aarch64, darwin arm64/x86_64), its sha256
  from the release's `checksums.txt` recorded in `kit.json` and verified
  before unpacking into `~/.local/bin`, then `rtk init -g --auto-patch`.
  No remote script is piped into sh. `impeccable` installs non-interactively
  with `npx impeccable@4.8.0 install --yes --user`; the skill's engine (0.1.x)
  is downloaded by that CLI and floats, so only the CLI is pinned, and
  "present" means the skill exists.
- 2026-11-08: gentle-ai 4.4.0 is a kit `release` item (pinned asset, sha256
  from its `checksums.txt`), then `gentle-ai install --agent claude-code
  --component persona,engram,skills --persona gentleman --skill <list>`. The
  explicit `--skill` list is the catalog minus `branch-pr`,
  `issue-creation` and the retired `sdd-*` skills (4.x has a native SDD
  dispatcher; the orchestrator block installs without them); gentle-ai records it in its state, so `gentle-ai sync`
  does not bring them back (checked in a throwaway HOME). Files installed
  before this stay: gentle-ai never deletes them, and RULES.md R1.5 tells
  agents they do not apply. Its skills are not pinned individually: two
  managers would fight over the same files.
- 2026-11-08: gentle-ai replaces its own binary on any invocation, even
  `--version` or `--help` (internal/app/selfupdate.go). `jb` exports
  `GENTLE_AI_NO_SELF_UPDATE=1` for everything it runs, so the pin only moves
  by editing `kit.json`. Never run gentle-ai bare against a real home.
- 2026-11-08: gentle-ai 4.4.0 refuses to write a symlinked
  `~/.claude/CLAUDE.md` ("refusing to read symlink"; no override) and rolls
  back, leaving the file untouched. With a regular file, `jb install` and
  gentle-ai `install`/`sync` keep each other's blocks in either order, twice
  (scripts/check). `jb install` runs the kit last, after anchoring RULES.md.
- 2026-11-08: the agent's rules file belongs to the installers (jb and
  gentle-ai each own a block), so `jb install` turns a symlinked one into a
  regular file once `personal.md` exists beside it in the agent's config dir;
  the owner's own rules live there, in the dotfiles, and the generated file
  imports it. Without `personal.md` the link is left alone, so no content is
  ever dropped. `jb install` also runs kit commands on regular copies of any
  symlinked rules or hook file and writes the result back through the link,
  because gentle-ai will not touch a symlink. `@RTK.md`, added to CLAUDE.md by
  `rtk init`, is not carried over when CLAUDE.md is regenerated; the rtk hook
  works without it.
- 2026-11-08: gentle-ai telemetry is switched off with its own opt-out
  (`gentle-ai telemetry disable`, persisted state, checked through
  `telemetry status --json`) as a kit item, not by editing the hook entries
  it keeps re-adding on every `sync`. The hooks stay but send nothing.
- 2026-11-08: nothing the owner wrote is ever lost to a failed edit. JSON goes
  through `json_edit` (temp file beside it, valid and non-empty, then written
  in place; unparsable files skipped with a note; one backup, never
  overwritten). `put_block` refuses unless each marker is a whole line
  appearing once, end after start, and never writes through a symlink.
  `jb install` refuses to run from a linked worktree (its links would dangle),
  and kit items marked `writesRules` (gentle-ai) wait until no present agent's
  rules file is still linked into the dotfiles: merge dotfiles, restow, then
  `jb install`. `jb stop` blocks the same failing output at most once, keyed
  by a hash in the git dir, whatever the agent reports about loops.
  `jb session` caches the kit check for an hour (keyed by kit, state files
  and binaries) because it runs under a 15 s hook timeout.
- 2026-11-08: gentle-ai is two kit items: the `release` binary and a
  `command` item `gentle-ai config` (`writesRules`) whose present test,
  `gentle_ai_configured`, looks for gentle-ai's own persona block in each
  rules file it manages plus its state file. Having the binary at the pin is
  not enough: on the first real install CLAUDE.md is regenerated and must be
  given gentle-ai's blocks again. Agent hooks call `$HOME/.local/bin/jb`, not
  the toolkit's path. `scripts/check` runs under bash 3.2 (stock macOS) and
  keeps `grep -q` and `head` out of pipes because pipefail turns their early
  exit into a failure.
- 2026-11-08: Codex keeps its built-in prompt. gentle-ai's engram component
  always sets `model_instructions_file` and `experimental_compact_prompt_file`
  in `~/.codex/config.toml` (no option skips them), so `jb install` removes
  exactly those two top-level keys after the kit has run, idempotently, through
  a temp file written in place (links stay), leaving every other byte alone;
  keys inside a table are kept. gentle-ai may still manage Codex's skills and
  AGENTS.md blocks. gentle-ai is only given agents whose CLI is on PATH
  (`gentleAiNeeds` in `kit.json`; it errors without `opencode`), and its
  post-apply check expects `~/.claude.json`, which exists once Claude Code has
  run. `jb stop` bounds `scripts/check` itself (`JB_STOP_TIMEOUT`, 150 s)
  because stock macOS has no timeout(1).
- 2026-11-07: the toolkit holds mentality, tools, patterns and rules, not a
  UI system or design tokens; those stay in the projects that own them.

## Project rules

- After changing `templates/managed/` or `RULES.md` in a way projects must
  pick up, bump `VERSION`; `jb check` then flags projects on the older one.
