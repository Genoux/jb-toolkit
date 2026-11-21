# AGENTS.md

How jb-toolkit is built and why. The rules it ships are `RULES.md`; this file
is about changing the toolkit itself. Read `PRODUCT.md` first.

## Stack

- bash, POSIX sh for the git hooks, jq — why: projects are Rust, JS and
  static HTML, so the toolkit cannot live in any one ecosystem, and these are
  already on every machine (and every CI runner).

## Layout

- `RULES.md` — the standard. `jb sync` copies it into each project's
  `.toolkit/RULES.md`, which the project's `AGENTS.md` points to.
- `bin/jb` — the agent's tool: install, session, new, sync, check, bump, tag.
  `jb sync` copies it into each project as `.toolkit/bin/jb`, the copy the
  project's agent hooks call; that copy refuses `new`, `sync` and `install`
  (they need the toolkit's templates) and reads the machine's toolkit, when
  `jb install` linked one, for the kit and the latest version.
- `kit.json` — the global baseline: the few plugins and tools every project
  needs (pinned; moving a pin is a deliberate edit), no skills. `retired` lists what the kit once installed and dropped, each with its own
  `present` test and `remove` step: `jb install` removes them, `jb session`
  reports them.
- `skills/` — jb's own skills, installed into every project by `jb sync`
  like the stack's skills (`.agents/skills/<name>`, linked into
  `.claude/skills`, never committed); a project picks up edits on its next sync.
- `.github/workflows/verify.yml` — the reusable CI every project's `ci.yml`
  calls: `scripts/check`, `jb check --ci`, gitleaks. The repo is public so
  public projects can call it.
  A project whose own CI job builds (system libraries, a macOS runner)
  keeps that job and calls this one with `scripts-check: false`.
- `templates/managed/` — overwritten in projects by `jb sync`.
- `templates/project/` — created once in a project, then the project's.
- `templates/stacks/<stack>/` — starting `scripts/check` and `.gitignore` lines,
  and the stack's skills: `skills.txt` (`<repo> <path> [name]`, fetched at
  their latest commit) and `skill-commands` (`<name> <command>`, a tool that
  installs its own skill: impeccable for `js` and `static`). `jb sync`
  installs both into the project; jb's lists, not ib-toolkit's.
- `scripts/check` — syntax, shellcheck when installed, and an end-to-end run
  in a temp dir: new project, `jb check`, silent `jb session`, hooks refusing,
  `jb bump`.

## Agents

`kit.json` › `agents` is the one list of agents the owner uses, each with the id
every external tool knows it by. `bin/jb` names agents only in the adapter
functions (`agent_dir`, `agent_rules_file`, `agent_project_hook_file`, ...),
one case per agent, keyed by that list. Enforcement does not depend on any
agent: git hooks, `scripts/check` and CI do the work; agent hooks (`jb guard`,
`jb stop`, `jb session`) are optional adapters, and every agent reads the
same rules through the project's `AGENTS.md`, which points to
`.toolkit/RULES.md`. Nothing is wired at user level any more: `jb sync` wires
each project's own hook file, calling the project's `.toolkit/bin/jb`, for
Claude Code always and for the other agents installed on the machine, and
only into folders the managed `.gitignore` keeps untracked (R3.4). Checked
against the vendors' docs on 2026-11-21:

| Agent | Rules | Project hooks (jb sync) | Source |
|---|---|---|---|
| claude-code | `CLAUDE.md` imports `AGENTS.md` | `.claude/settings.json`: SessionStart, PreToolUse (Bash), Stop, calling `"$CLAUDE_PROJECT_DIR"/.toolkit/bin/jb` | https://code.claude.com/docs/en/hooks ("Hook locations"; `${CLAUDE_PROJECT_DIR}` "the project root where the session started") |
| codex | `AGENTS.md` | `.codex/hooks.json`: same three, calling `"$(git rev-parse --show-toplevel)"/.toolkit/bin/jb` (hooks run in the session's cwd; the docs say to resolve from the git root). Codex loads them only in a trusted project, after the owner approves them in `/hooks`; sync says so the first time | https://learn.chatgpt.com/docs/hooks (was developers.openai.com/codex/hooks) |
| cursor | `AGENTS.md` | `.cursor/hooks.json`: beforeShellExecution (guard), stop (followup_message), calling `.toolkit/bin/jb` ("Project hooks run from the project root"); no session hook: sessionStart output is undocumented | https://cursor.com/docs/agent/hooks |
| github-copilot (CLI) | `AGENTS.md` | not wired: repository hooks are `.github/hooks/*.json` (or `.github/copilot/settings.json`), committed with the project; `settings.local.json` there is only "typically gitignored". `--agent github-copilot` keeps its shapes | https://docs.github.com/en/copilot/reference/hooks-configuration |
| opencode | `AGENTS.md` | none: only JS plugins, no JSON hook system | — |

Unverified in the docs and so assumed: Copilot's `toolArgs` JSON string;
`jb guard` reads it and passes when it finds no command.

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
  `jb`, wired into Claude Code and Codex (by `jb sync`, per project, since
  2026-11-21). Both take the same
  hook JSON shape, so no adapter. `jb stop` blocks with
  `{"decision":"block","reason":...}` and returns early on `stop_hook_active`.
  Codex runs hooks only after the owner trusts them in `/hooks`, so
  `jb sync` prints that line: this slightly widens "the owner runs only
  `jb install`" to "plus one approval in Codex".
- 2026-11-07: nothing in the kit is left to the owner:
  `rtk` is a `release` item: the pinned GitHub release asset for the
  current os-arch (linux x86_64/aarch64, darwin arm64/x86_64), its sha256
  from the release's `checksums.txt` recorded in `kit.json` and verified
  before unpacking into `~/.local/bin`, then `rtk init -g --auto-patch`.
  No remote script is piped into sh.
- 2026-11-21: ponytail and caveman are out of the kit. Both act on every
  response: ponytail's SessionStart hook demands the shortest diff even for
  UI and UX decisions, and caveman's tool hooks elided sections of file reads
  the agent was editing. Lean-code and compression tools are each developer's
  choice, installed by hand, never pushed by the kit.
  Dropping an item from `items` uninstalled nothing, so the machine kept
  both, and caveman's proxy kept compressing every session: they are
  `retired` entries now, removed through each tool's own commands (`claude
  plugin uninstall`, `cave disable`, `npm uninstall -g`). `~/.caveman` (its
  databases and credentials) is the owner's data and stays; a caveman proxy
  still running serves the sessions started before, until they restart.
  superpowers is retired with them: its session hook told the agent to load
  a skill on a 1% chance, so a hundred skill descriptions competed with
  RULES.md on every request.
- 2026-11-21: fresh start, owner's choice: nothing that tells the agent how
  to work is installed globally any more. 115 global skills (five UI
  authorities that contradicted each other) and gentle-ai's 540-line
  orchestrator in every rules file made sessions lose the thread. The global
  kit keeps tools only: engram, context7, egm, rtk. gentle-ai (orchestrator,
  review agents, skills, its duplicate engram block), impeccable and the
  shared skill list are `retired`; gentle-ai 4.4.0's own `uninstall` leaves
  its orchestrator blocks, so `retire_gentle_ai` strips every
  `<!-- gentle-ai:* -->` block afterwards. A retire step removes only what the
  retired tool installed (impeccable's folders hold its `scripts/` engine;
  empty hook lists the owner had stay); a block without its end marker or a
  hooks file jq cannot read is reported and left byte for byte. Skills, impeccable included as the
  one UI authority, go into each project by stack, in ib-toolkit's shape
  (`.agents/skills/<name>`, linked into `.claude/skills`), from jb's own
  lists: the two toolkits look alike and share nothing (owner: one for
  personal projects, one for work). Each installed folder holds a
  `.gitignore` of `*`, so nothing jb installs is committed and the project's
  own skills beside it still are; the list's state lives in the git dir.
  impeccable installs with `--project --no-hooks`: its engine binary is
  per-platform, and its hooks are what the fresh start removed.
- 2026-11-21: jb is per project (owner: "jb should be per proejct now").
  `jb sync` copies the CLI into `.toolkit/bin/jb`, wires each agent's
  project hooks to that copy and installs jb's own skills into the project;
  `jb install` keeps tools only: the `~/.local/bin/jb` link the agent runs
  `new`, `sync` and `check` with, the import of the owner's `personal.md`,
  the kit and its `retired` removals. The global RULES.md block, the global
  hooks and the global skill links are a `retired` entry ("global jb rules
  and hooks") that removes exactly those, recognised by marker, command or
  link target. A session hook in a project now runs the version that project
  was synced at, so an old toolkit elsewhere never judges it, and a clone
  works with the hooks of its own commit. Rejected: keeping the global hooks
  (they ran in every folder, adopted or not, and the rules they carried
  reached projects the owner keeps off the standard); hooks calling
  `~/.local/bin/jb` from the project (a project would still depend on a
  global install); wiring Copilot CLI (its project hooks sit in
  `.github/hooks`, which a project commits). Gemini CLI is no longer a
  supported agent: the owner does not use it. The cost: a repository not yet
  adopted gets no session hook and no rules, so adoption starts when the
  owner asks, from that repo or from a session in another project.
- 2026-11-08: the agent's rules file belongs to `jb install`, so it turns a symlinked one into a
  regular file once `personal.md` exists beside it in the agent's config dir;
  the owner's own rules live there, in the dotfiles, and the generated file
  imports it. Without `personal.md` the link is left alone, so no content is
  ever dropped. `jb install` also runs kit commands on regular copies of any
  symlinked rules or hook file and writes the result back through the link,
  because some tools refuse a symlinked config ("refusing to read symlink"). `@RTK.md`, added to CLAUDE.md by
  `rtk init`, is not carried over when CLAUDE.md is regenerated; the rtk hook
  works without it.
- 2026-11-08: nothing the owner wrote is ever lost to a failed edit. JSON goes
  through `json_edit` (temp file beside it, valid and non-empty, then written
  in place; unparsable files skipped with a note; one backup, never
  overwritten). `put_block` refuses unless each marker is a whole line
  appearing once, end after start, and never writes through a symlink.
  `jb install` refuses to run from a linked worktree (its `~/.local/bin/jb`
  link would dangle).
  `jb stop` blocks the same failing output at most once, keyed
  by a hash in the git dir, whatever the agent reports about loops.
  `jb session` caches the kit check for an hour (keyed by kit, state files
  and binaries) because it runs under a 15 s hook timeout.
- 2026-11-08: agent hooks call jb by a path that survives moving the
  toolkit (since 2026-11-21 the project's own `.toolkit/bin/jb`). `scripts/check` runs under bash 3.2 (stock macOS) and keeps `grep -q`
  and `head` out of pipes because pipefail turns their early exit into a
  failure. `jb stop` bounds `scripts/check` itself (`JB_STOP_TIMEOUT`, 150 s)
  because stock macOS has no timeout(1).
- 2026-11-21: `jb check` judges commit history only from the commit that
  adopted the standard (the one adding `.toolkit/toolkit-version`). overview
  carried three attribution lines from before its adoption; rewriting a
  public history was rejected, and an R3.4 exception would also hide the
  agent-folder check.
- 2026-11-10: apps update themselves from their latest published GitHub
  Release (R9), never by pulling source. One feed format for every stack
  (Tauri's `latest.json`) so a CLI and a desktop app read the same thing. App
  repos are public (owner's choice): private release assets need a token,
  and a token shipped inside an app is a leak. A separate public releases
  repo was the alternative, kept for when an app's source must stay private.
  The toolkit's `release.yml` only passes the signing secrets; per-platform
  build matrices stay in each app's own `release.yml`.
- 2026-11-07: the toolkit holds mentality, tools, patterns and rules, not a
  UI system or design tokens; those stay in the projects that own them.

## Project rules

- After changing `templates/managed/` or `RULES.md` in a way projects must
  pick up, bump `VERSION`; `jb check` then flags projects on the older one.
