---
name: jb-release
description: Cut a release of a jb-toolkit project (version bump, pull request, tag, draft GitHub release). Use when the owner asks to release, ship, publish a version, tag, or bump the version.
---

# Release

The owner's request to release is the permission to commit, push and open the pull request for it (RULES.md R1, R4). Publishing the drafted GitHub release stays the owner's call.

1. Decide the version from the commits since the last tag (`git describe --tags --abbrev=0`, then `git log <tag>..HEAD --format=%s`): any `!`/BREAKING → major (minor while `0.y`), any `feat` → minor, else patch. Say which and why in one line. A pre-release (`-alpha.N`, `-rc.N`, …) only if the owner asks.
2. `scripts/check` must pass on main first.
3. `git switch -c release/vX.Y.Z` from up-to-date main, `jb bump X.Y.Z`, `git push -u origin release/vX.Y.Z`, then `gh pr create --fill`.
4. Wait for CI (`gh pr checks --watch`). Merge with `gh pr merge --rebase --delete-branch`.
5. `git switch main && git pull --ff-only`, `jb tag`, `git push origin vX.Y.Z`.
6. Watch the Release workflow (`gh run watch`). Report the draft's URL (`gh release view vX.Y.Z --json url -q .url`) and tell the owner it is a draft waiting for them to publish.

If the project has no remote, stop after step 3's commit and `jb tag` on main after merging locally (`git merge --ff-only`), and say there is no release workflow to run.
