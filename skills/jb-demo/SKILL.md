---
name: jb-demo
description: Show the owner a demo, mockup, preview, prototype, visual explanation or comparison as a local HTML file. Use whenever you would otherwise publish an Artifact, a Claude Doc or any hosted page, or when the owner asks to see, preview or mock something up.
---

# Local demo

Never publish an Artifact, Claude Doc, Gist or hosted page (RULES.md R2). Instead:

1. Write one self-contained file: `.demo/<yyyy-mm-dd>-<slug>.html` in the project root (outside a project: the session scratchpad). Inline all CSS and JS; no build step, no CDN unless unavoidable. Support light and dark (`prefers-color-scheme`).
2. Open it: `open .demo/<file>` (macOS) or `xdg-open` (Linux). If it needs a server (fetch, modules): `python3 -m http.server -d .demo 4173` in the background and give the URL.
3. To check it yourself, screenshot with headless Chrome into `.demo/shots/`.
4. `.demo/` is gitignored. If the owner wants to keep it, move it to `prototype/` and add a line to `AGENTS.md`.
