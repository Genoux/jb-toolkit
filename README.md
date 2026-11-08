# jb-toolkit

One standard for every project: how to branch, commit, version, release,
lay out a repo, and how agents behave (local-only demos, no attribution,
commit only when asked). Stack and UI stay free per project.

**The owner runs one command, once per machine. The agent does everything else.**

```sh
~/Desktop/@john/jb-toolkit/bin/jb install
```

After that, you pitch and decide; agents scaffold, adopt, fix drift, release
and demo by themselves. Install wires these into every agent in `kit.json` › `agents` that is installed
(Claude Code, Codex, Cursor, Gemini CLI, GitHub Copilot CLI, opencode: what each
really supports is in the table in `AGENTS.md`). For Claude Code that is:

| What | Where | Effect |
|---|---|---|
| `RULES.md` | imported from `~/.claude/CLAUDE.md` | every session, in any folder, knows the standard (R0 tells the agent it runs the toolkit) |
| Session hook | `SessionStart` in `~/.claude/settings.json` | in a project, runs `jb session`: silent when the project is clean; otherwise lists what to fix, or says the project is not adopted yet |
| Guard hook | `PreToolUse` (Bash) | `jb guard` refuses `--no-verify`, `commit -n`, `core.hooksPath` changes and `HUSKY=0`/`LEFTHOOK=0`/`SKIP=` on git commands |
| Stop hook | `Stop` | `jb stop` runs `scripts/check` when the tree has changes and sends the agent back to work if it fails; honors `stop_hook_active`, silent otherwise |
| Skills | `~/.claude/skills/jb-*` | `jb-new-project` (you pitch → it scaffolds), `jb-adopt` (existing repo → standard, in a worktree), `jb-release` (you say "release" → bump, PR, tag, draft), `jb-demo` (anything to show → local `.demo/*.html`) |
| Baseline kit | `kit.json` | plugins, skills (pinned to a commit) and tools every machine should have; `jb install` adds what is missing, `jb session` names what is |
| `jb` | `~/.local/bin/jb` | the agent's tool: `new`, `sync`, `check`, `bump`, `tag` |

Then, in every toolkit project, git hooks refuse off-standard commits and
branches, and `.claude/settings.json` denies the Artifact and Claude Docs
tools. The rules are enforced, not only written down.

Re-run `jb install` after moving the toolkit; it is idempotent and backs up
`settings.json` to `settings.json.jb-backup`. Codex gets a pointer to
`RULES.md` in `~/.codex/AGENTS.md` and the same three hooks in
`~/.codex/hooks.json`; Codex runs them only after you trust them in `/hooks`.

## Why a scaffold that syncs, not a package

| Option | Problem |
|---|---|
| npm / cargo package | Locks the standard to one ecosystem. Projects here are Rust, Bun/TS, Next, static HTML. |
| Copy-once scaffold (template repo) | Drifts: fix a hook in one project and the others never get it. |
| **Scaffold + sync + global rules** (this) | `jb new` creates, `jb sync` re-applies the managed files at any time, `jb check` reports drift, and `~/.claude/CLAUDE.md` imports `RULES.md` so every agent session follows it, scaffolded or not. |

The rules live in one place and every project points to them. Change a rule
here and it applies everywhere: agents read it live, and `jb sync` pushes the
file-level parts (hooks and so on) into each repo.

## The agent's commands

You never need these; they are what the agent runs.

```sh
jb new hangar --stack js --github   # new project in ~/Desktop/@john (override: JB_PROJECTS or --dir)
jb sync                             # in a project: re-apply managed files, adopt an existing repo
jb check [path]                     # what departs from RULES.md (read-only)
jb bump 0.4.0                       # on a branch: set the version, commit "chore: bump version to 0.4.0"
jb tag                              # on up-to-date main after merge: tag v0.4.0, then git push origin v0.4.0
```

Stacks (`--stack`; `jb sync` detects one when omitted) only decide the
starting `scripts/check` and a few `.gitignore` lines:

| Stack | `scripts/check` |
|---|---|
| `rust` | `cargo fmt --check`, `clippy -D warnings`, `cargo test` |
| `js` | installs with the lockfile's package manager (bun, pnpm, npm), then `check`, or `format:check`, `lint`, `typecheck`, `test` |
| `static` | every local `href`/`src` in the HTML resolves |
| `none` | a placeholder until a stack is chosen |

### Services

A repo with a `Dockerfile` at the root or one folder down is a service (R8).
`jb sync` adds the managed `.github/workflows/image.yml`, and `jb check`
requires an image-only `compose.yaml` at the root (no `build:`, no host bind
mounts). Deploy flow: tag a release, CI publishes `ghcr.io/<owner>/<repo>[-<folder>]`,
paste `compose.yaml` into Portainer or a ZimaOS custom app, set the env vars,
and once per new package make it public in the GitHub package settings. The image is
published only after the release workflow passed for the tag. GitHub fires `workflow_run` only
for workflows on the default branch, so this starts working once `image.yml` is merged.

## What a project gets

| Managed (overwritten by `jb sync`) | Owned (created once, then the project's) |
|---|---|
| `.githooks/pre-commit`: no commits on main, no agent folders, no `.env` | `README.md`, `AGENTS.md`, `CLAUDE.md` (`@AGENTS.md`), `PRODUCT.md` |
| `.githooks/commit-msg`: Conventional Commits, 72 chars, no attribution | `scripts/check`: the one health command |
| `.githooks/pre-push`: branch names, no pushes to main | `.github/workflows/ci.yml`: runs `scripts/check` |
| `.toolkit/version`: reads and cross-checks manifest versions | `.github/workflows/release.yml`: tag vs version, check, build, draft release |
| `.github/workflows/image.yml`: services only, builds and publishes the image on a tag | |
| `.editorconfig`, PR template, the `jb-toolkit` block in `.gitignore` | `VERSION` (only when there is no manifest) |

`core.hooksPath` is set to `.githooks` by `jb new`/`jb sync`, unless the
project already uses a hook manager (lefthook, husky, pre-commit or its own
`core.hooksPath`); then `jb sync` leaves it alone and `jb check` reports
"hooks managed by <manager>". Each clone needs it once: `jb sync` or
`git config core.hooksPath .githooks`. Deliberate departures from the rules
go in `.jb-exceptions`, one `R<id>: reason` per line.

## Adopting an existing project

Open a session in it. The hook flags it as not adopted and the agent offers
to adopt it (RULES.md R0.3) on a `chore/adopt-jb-toolkit` branch. Gramly is
closest already: same hooks idea, same version and tag flow.

## Changing the standard

Ask an agent to change `RULES.md`, `skills/` or `templates/` and bump
`VERSION`. Rules and skills apply at once (they are linked, not copied);
each project's next session sees it is on an older toolkit and the agent
re-syncs it.

## What to add next

Ordered by value for the effort:

1. **Secret scanning**: run `gitleaks protect --staged` in pre-commit when it
   is installed.
2. **`jb doctor`**: check the machine (gh auth, toolchains, hooksPath in every
   repo under `~/Desktop/@john`) and report every project's `jb check` in one table.
3. **Stack recipes** (`templates/stacks/<x>/recipe.md`): the default choices
   for each stack (Next + Bun, Rust CLI/daemon, Tauri), so "use the usual"
   means something specific. These are opinions, not mandatory.
4. **ADR template + `docs/adr/` index** for decisions too big for a bullet in
   `AGENTS.md`.
5. **Dependabot / Renovate config** in the managed set, grouped weekly.
6. **Branch protection** via `jb new --github` once the account is on a plan
   that allows it for private repos (GitHub Free does not).
