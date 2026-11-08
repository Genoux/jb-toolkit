---
name: jb-adopt
description: Bring an existing project onto the owner's jb-toolkit standard without disturbing it. Use when jb session says a project is not on the standard yet and the owner agrees to adopt it, or when the owner asks to adopt, onboard or standardise an existing repo.
---

# Adopt an existing project

Adoption changes the project's scaffolding, never its application code. The owner's uncommitted work is theirs: do not touch it.

1. **Work in a worktree.** `git worktree add ../.wt/<name>-adopt -b chore/adopt-jb-toolkit` from the default branch. The main checkout and its uncommitted changes stay untouched. If the repo is not on `main`, say which branch you based it on.
2. **Survey before changing.** Read, in this order, and note each finding in one line:
   - hook manager: lefthook, husky, pre-commit, or a foreign `core.hooksPath`. `jb sync` leaves it in place and `jb check` accepts it; do not replace it.
   - branch scheme and commit style in `git log`; existing CI under `.github/workflows/`;
   - lockfiles: if more than one package manager's lockfile exists (a stale `bun.lock` next to `package-lock.json`), ask the owner which one is real. Do not pick.
   - documented gates: `README.md`, `AGENTS.md`, `CONTRIBUTING.md`, CI steps.
3. **Sync.** `jb sync` in the worktree. It adds the managed files and creates only what is missing.
4. **`scripts/check` mirrors what the project already gates on**: the CI steps, or the documented commands. Do not invent stricter checks. Make the existing workflows call `scripts/check` and `.toolkit/version`; leave their other jobs alone.
5. **Record exceptions, do not rewrite.** Where the project deliberately differs from `RULES.md`, add `R<id>: reason` to `.jb-exceptions` (one per line). `jb check` then reports it as excepted. A habit you dislike is not an exception; a decision the project already made is.
6. **Never fix application code.** If `scripts/check` fails on code the adoption did not touch, record the failure in `AGENTS.md` (what fails, since when it is known) and report it. Fixing it is a separate branch the owner chooses.
7. **Docs.** Missing `PRODUCT.md`: draft it from the code and the README, headed with "Draft from the code — owner to confirm." Fill `AGENTS.md` › Stack from what the project actually uses. Do not edit docs that exist.
8. **Verify.** `jb check` until it reports no problems (excepted lines are fine), then run `scripts/check` and report its real result, including pre-existing failures.
9. **Stop at the commit.** Commit on the adoption branch (part of an adoption the owner approved, R1.1). Never push or open a pull request; say what is ready and let the owner decide.

Report in a few lines: what was added, what was left alone, exceptions recorded, failures that predate the adoption, and any question that needs the owner (lockfile, branch base).
