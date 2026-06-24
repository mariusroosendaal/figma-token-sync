# figma-token-sync

A **deterministic, no-LLM** pipeline that pulls Figma variables, text styles, and
effect styles into a single **`frontend.config.json`** for
[`@area17/a17-tailwind-plugins`](https://github.com/area17/tailwind-plugins) — and
(optionally) opens a GitHub PR whenever the design drifts from the repo. One-way:
**Figma owns the values**, the config is regenerated from them.

This repo only **produces** `frontend.config.json`. The consuming app (e.g. an
[a17 Tailwind](https://area17.github.io/tailwind-plugins/) project) builds its CSS
from that config with its own `tailwind.config`.

> Why no LLM: the Figma **Variables REST API is Enterprise-only**, and relaying a
> few hundred values through a model is exactly the non-determinism this avoids. A
> plugin reads variables via the Plugin API (every plan), and a pure transformer
> writes the result.

## Layout

```
figma-token-sync/
  figma-sync.config.mjs       wiring: output dir + which transformer
  frontend.config.json        GENERATED from Figma — the deliverable
  tokens/figma-sync/          the toolkit
    transform.a17.mjs         the deterministic core: export → frontend.config.json
    transform.a17.test.mjs    offline proof of the core (npm run sync:test)
    sync.mjs                  CLI: export.json → frontend.config.json   (npm run sync)
    sync-server.mjs           localhost receiver the plugin posts to    (npm run sync:serve)
    plugin/                   the Figma plugin (vanilla JS, no build step)
  .github/workflows/          the "drift → PR" Action
```

## What it generates

`transform.a17.mjs` reads the Figma export and emits the four sections the a17
plugins consume:

| `frontend.config.json` | from Figma |
| --- | --- |
| `structure` — `breakpoints`, `columns`, `gutters.{inner,outer}`, `container` | the `responsive` collection: `…/breakpoint`, `layout/columns`, `layout/gutter` (inner), `layout/margin` (outer) |
| `spacing` — `tokens.scaler` + per-breakpoint `groups` | the `space/*` scale |
| `color` — flat `tokens` + semantic `border`/`text`/`background` | color primitives + the semantic `color` collection (light theme) |
| `typography` — `families` + responsive `typesets` | `font family/*` stacks + text styles (`<role>/<bp>`) |

### Conventions (how the Figma file is read)

- A **variable's name is its path** — `layout/columns`, `color/gray/950`.
- **`_` / `.`-prefixed** collections, groups, modes, and variables are **private** —
  design-time aids, never emitted (e.g. a `_grid` group, a `_wireframe` proofing mode).
  A token that *aliases* a private value still resolves it.
- **`font family/<x>-stack`** holds the CSS fallback stack for family `<x>`.
- **Text styles** are named `<role>/<bp>` (or `<role>/<bp-range>`, `+ -strong`) →
  one responsive typeset per role; `-strong` becomes the typeset's `bold-weight`.
- Breakpoint keys are kept as named, except **`2xl` → `xxl`** (a leading digit is an
  invalid CSS class prefix). Color names are flattened, collapsing `X/X` → `X`.

## Use it — local loop

```bash
npm run sync:serve            # start the localhost receiver
# In Figma: run the plugin → "Read & Sync to repo"
git diff frontend.config.json # review, then commit
```

CLI (from a downloaded export):

```bash
npm run sync -- export.json              # write frontend.config.json
npm run sync -- export.json --dry-run    # preview, write nothing
npm run sync -- export.json --report     # coverage only (use this when adapting)
```

## Use it — GitHub PR loop (no local server)

1. In the plugin, set **GitHub repo** (`owner/name`) + a **fine-grained PAT**
   (Contents: Read/Write on this repo), then **Sync to GitHub & open PR**.
2. The plugin commits the export to the **`figma-sync/incoming`** branch; that push
   triggers `.github/workflows/figma-token-sync.yml`, which transforms it and
   opens/updates a single PR — only if `frontend.config.json` drifted.
3. Review and merge.

One repo setting: **Settings → Actions → General → Workflow permissions →
"Allow GitHub Actions to create and approve pull requests."**

## Installing the plugin (once)

Figma → **Plugins → Development → Import plugin from manifest…** → pick
`tokens/figma-sync/plugin/manifest.json`. Runs locally; no publishing. Network
access is limited to `http://localhost:41789` and `https://api.github.com`.

See `tokens/figma-sync/README.md` for the full mapping reference and tuning notes.
