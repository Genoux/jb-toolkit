# Rules

The standard every project follows, whatever its stack. Agents read this file
through the global `~/.claude/CLAUDE.md` import (`jb install`), so a project
never has to restate it. A project's own `AGENTS.md` may add rules or record a
deliberate exception (with the reason); it never silently contradicts these.

Rules are numbered so a review or a commit can cite them (`R4.2`).

## R0. The agent runs the toolkit

The owner pitches and decides; the agent does every toolkit step. Never ask
the owner to run `jb` or any procedure below: run it yourself and report.
`jb` is on PATH (`~/.local/bin/jb`); `jb help` lists its commands.

1. **New project.** When the owner pitches or names something new to build,
   follow the `jb-new-project` skill (it runs `jb new`).
2. **Every session in a project.** The SessionStart hook runs `jb session`
   and puts any departure from these rules in your context; an agent without
   that hook (Codex, Cursor, Gemini CLI, ...) runs `jb session` itself before
   other work. Fix them as part of the session, on a branch: `jb sync`
   re-applies managed files; the rest (`scripts/check`, docs, workflows) you
   edit by hand. Run `jb check` again
   to confirm, and tell the owner in one line what changed.
3. **Project not on the standard yet.** Offer once to adopt it. On yes:
   `git switch -c chore/adopt-jb-toolkit`, `jb sync`, make `scripts/check`
   run what the project's CI already runs, make the existing workflows call
   `scripts/check` and `.toolkit/version`, `jb check` until clean, commit.
   Rewrite nothing the project already does well; record deliberate
   exceptions in its `AGENTS.md`.
4. **Release.** When the owner asks to release or ship, follow the
   `jb-release` skill.
5. **Showing something.** Follow the `jb-demo` skill (R2).
6. **Before saying work is done.** Run `scripts/check` and report its result.
7. **Commits** go through the hooks; never bypass them (`--no-verify`). If a
   hook refuses, fix the cause.

## R1. Working with the owner

1. Commit only when the owner asks, or when the step is part of a toolkit
   procedure the owner started (a new project, an adoption, a release).
   Push only when the owner asks; asking for a release counts.
2. Ask before anything leaves this machine: publishing, deploying, opening a
   pull request, creating a repo, posting to any service.
3. When a choice is the owner's (product, money, security, data, anything
   hard to reverse), ask. Otherwise pick the sensible default, say which, and go.
4. Report what was actually run and what it showed. A check that was skipped
   is reported as skipped.

## R2. Demos, previews, prototypes: local only

1. Never publish work as a claude.ai Artifact, Claude Doc, Gist, pastebin,
   Vercel preview or any account-bound page. The owner switches between
   several Claude accounts and one of them is shared with other people.
   In a toolkit project the Artifact and Claude Docs tools are denied in
   `.claude/settings.json`; a denial is the rule working, not a problem to
   route around.
2. A throwaway demo is a self-contained HTML file in the project's `.demo/`
   folder (gitignored): `.demo/<yyyy-mm-dd>-<slug>.html`. Inline the CSS and
   JS; no build step. Open it with `open .demo/<file>` or serve the folder with
   `python3 -m http.server -d .demo 4173`.
3. A prototype worth keeping goes in `prototype/` and is committed, with a
   line in `AGENTS.md` saying what it shows.
4. Screenshots for review go in `.demo/shots/` (headless Chrome or the
   project's own preview command).

## R3. Git

1. `main` is always releasable and only moves through pull requests.
   `.githooks/pre-commit` refuses a commit on `main` (the first commit of a new
   repo excepted).
2. Branches: `<type>/<short-kebab-slug>`, type one of `feat fix chore docs
   refactor perf test ci build release experiment`. Example:
   `feat/hands-free-dictation`. `.githooks/pre-push` enforces it.
3. Commits are [Conventional Commits](https://www.conventionalcommits.org):
   `<type>(<optional scope>): <subject>`.
   - Types: `feat fix chore docs refactor perf test ci build style revert`.
   - Subject: lower case, no trailing period, at most 72 characters, says what
     the change does for the user or the code, in plain words:
     `fix: give a tap time to become a double-tap`.
   - Breaking change: `feat!: …` plus a `BREAKING CHANGE:` line in the body.
   - The body (optional) says why, not what.
   `.githooks/commit-msg` enforces the header.
4. No agent attribution, ever: no `Co-authored-by:` trailers, no "Generated
   with" lines, no tool signatures, no agent-named branches. Agent tooling
   folders (`.claude`, `.cursor`, `.codex`, `.serena`, `.engram`, `.atl`) stay
   untracked. The hooks refuse both.
5. Merge by rebase: history stays linear and each commit message is the
   record. Delete the branch after merging.
6. One concern per pull request. A pull request passes CI before it merges.
7. Never rewrite `main`'s published history without the owner asking.

## R4. Versions and releases

1. [SemVer](https://semver.org). `0.y.z` while the product is finding its
   shape. Pre-releases: `X.Y.Z-alpha.N`, `-beta.N`, `-rc.N`, `-experimental.N`.
2. The version lives in the stack's manifest (`Cargo.toml`, `package.json`,
   `pyproject.toml`) or, for projects without one, a `VERSION` file.
   `.toolkit/version` reads it and fails if two manifests disagree.
3. Release flow (the agent runs it, `jb-release` skill):
   1. On a branch: `jb bump X.Y.Z` (commits `chore: bump version to X.Y.Z`).
   2. Merge the pull request once CI passes.
   3. On an up-to-date `main`: `jb tag` (creates annotated tag `vX.Y.Z`),
      then `git push origin vX.Y.Z`.
   4. `.github/workflows/release.yml` refuses a tag that disagrees with the
      manifest, runs `scripts/check`, runs `scripts/build` if present, and
      drafts a GitHub release with generated notes and everything in `dist/`.
   5. Publishing the draft is the owner's decision.
4. Release notes come from the commit subjects, which is why R3.3 matters.

## R5. Project layout

Every project has:

| Path | Holds |
|---|---|
| `README.md` | What it is, how to run it, how to check it. For a stranger. |
| `AGENTS.md` | Why it is built the way it is: stack and why, layout, decisions with what was measured or the mistake each guards, project-specific rules. For the next agent. |
| `CLAUDE.md` | `@AGENTS.md` only. One source of truth for every agent. |
| `PRODUCT.md` | Who it is for, what it must do, what it must not. Written before code. |
| `scripts/check` | The one command that says the project is healthy: format, lint, types, tests. CI runs exactly this. |
| `scripts/build` | Optional. Produces release files in `dist/`. |
| `.toolkit/` | Managed by `jb sync`. Do not edit. |
| `.githooks/` | Managed by `jb sync`. Do not edit. |
| `.demo/` | Throwaway demos (R2). Gitignored. |

Optional, when the project has them: `DESIGN.md` (UI system),
`TROUBLESHOOTING.md` (user-facing fixes), `docs/adr/YYYY-MM-DD-<slug>.md`
(decisions too big for a bullet in `AGENTS.md`), `prototype/`.

## R6. Stack and code

1. The stack is chosen per project and recorded in `AGENTS.md` › Stack with
   the reason. The toolkit does not pick frameworks or UI libraries.
2. Pin toolchains (`rust-toolchain.toml`, `.tool-versions`, `packageManager`)
   and commit the lockfile. Add a dependency only when it earns its place;
   say why in the commit body.
3. Strict mode on: TypeScript `strict`, Rust `clippy -D warnings`, linters at
   error. Formatting is automatic and checked in `scripts/check`.
4. Tests sit next to what they test or in the stack's usual place; a bug fix
   comes with the test that would have caught it, when one can be written.
5. Secrets live in `.env` (gitignored) or the OS keychain. Commit
   `.env.example` with names and no values.
6. Record measurements, not beliefs: when a decision rests on a number (speed,
   size, accuracy), `AGENTS.md` keeps the number, the date and how it was taken.

## R7. Hosting

1. Repos are private on the owner's personal GitHub account unless the owner
   says otherwise. Never an organisation account without being asked.
2. Merge settings: rebase only, delete branch on merge (`jb new --github`
   sets these; for an existing repo, `gh repo edit --enable-rebase-merge
   --enable-squash-merge=false --enable-merge-commit=false --delete-branch-on-merge`).
3. Shared design work (e.g. the Signal design system in `../signal`) is used
   when the owner asks for it; a project never copies and edits it locally.
