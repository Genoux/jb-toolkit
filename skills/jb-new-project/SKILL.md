---
name: jb-new-project
description: Start a new project on the owner's jb-toolkit standard. Use whenever the owner pitches, describes or names a new project, app, tool or idea to build ("I want to build…", "new project…", "let's make…"), before writing any code for it.
---

# Start a new project

The owner pitches; you do every step. The owner never runs `jb`.

1. From the pitch, settle three things, asking only for what the pitch does not say:
   - **name**: short kebab-case (propose one if none was given);
   - **stack**: `rust`, `js`, `static`, or `none` if it is too early. Propose the fitting one with one line of why; the owner confirms or overrides;
   - **GitHub**: create the private repo now or later (default: ask once).
2. Run `jb new <name> --stack <stack>` (add `--github` only if the owner said yes). It creates `~/Desktop/@john/<name>` with git, hooks, docs, CI and the first commit.
3. Work in that folder from now on. Write `PRODUCT.md` from the pitch (users, purpose, what it is not, platform) and fill `AGENTS.md` › Stack with the choice and why. Show the owner `PRODUCT.md` in a few lines and ask what is wrong with it.
4. `git switch -c feat/<first-slice>` and build the smallest version worth using. If the owner wants to see an idea first, use the `jb-demo` skill.
5. Before saying a step is done, run `scripts/check` and report what it printed.
