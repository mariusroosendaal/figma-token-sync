# figma-token-sync

A deterministic pipeline that pulls Figma variables and text styles into a
single **`frontend.config.json`** for
[`@area17/a17-tailwind-plugins`](https://github.com/area17/tailwind-plugins), and
optionally opens a PR when the design drifts. One-way: **Figma owns the values.** This
repo only *produces* the config; the consuming app builds its CSS from it.

## Layout

```
figma-token-sync/
  figma-sync.config.mjs       wiring: output dir + which transformer
  frontend.config.json        GENERATED — the deliverable
  tokens/figma-sync/          the toolkit
    transform.a17.mjs(.test)  the deterministic core: export → config (+ offline test)
    diff.mjs(.test)           pure token-diff helpers, shared by CLI + plugin drift check
    sync.mjs                  CLI: export.json → config         (npm run sync)
    sync-server.mjs           localhost receiver for the plugin (npm run sync:serve)
    install.sh                vendors the toolkit into a consuming app
    github-workflow.template.yml   the "drift → PR" Action — copy into the consuming app
    plugin/                   the Figma plugin (vanilla JS, no build)
```

The `tokens/figma-sync/` nesting mirrors the *install layout* — where `install.sh`
vendors the toolkit — and the repo root doubles as a sample consuming app
(`tokensDir: '.'`), so the toolkit can be dogfooded in place.

## What it generates

`frontend.config.json` has four sections; all routing lives in `CONFIG` at the top of
`transform.a17.mjs`.

| Section | Source in Figma |
| --- | --- |
| `structure.breakpoints` | the `responsive` collection's `…/breakpoint` variable (per mode) |
| `structure.columns` | `layout/columns` (per breakpoint) |
| `structure.gutters.inner` / `.outer` | `layout/gutter` / `layout/margin` |
| `structure.container` | defaults to `"auto"` — no Figma source |
| `spacing.tokens.scaler` + `.groups` | `CONFIG.spacingScaler` + the `space/*` scale |
| `color.tokens` | color primitives, names flattened (`color/gray/950` → `gray-950`) |
| `color.border` / `.text` / `.background` | the semantic `color` collection (light theme) |
| `typography.families` | `font family/<x>` (+ `-stack`) primitives |
| `typography.typesets` | text styles named `<role>/<bp>` → one typeset per role |

### Conventions

- **A variable's name is its path** — `layout/columns`, `color/gray/950`.
- **Private** — names starting with `_` or `.` are design-time aids, never emitted; a
  token that *aliases* a private value still resolves it.
- **`font family/<x>-stack`** holds the CSS fallback stack for `<x>` (quoting normalized).
- **Text styles** `<role>/<bp>` (or `<bp-range>`, `+ -strong`) → one responsive typeset
  per role; `-strong` sets the typeset's `bold-weight`. Unrecognized trailing segments are
  skipped with a warning (`report.unparsed`).
- **`2xl` → `xxl`** (a CSS class can't start with a digit); **`X/X` → `X`**
  (`white/white` → `white`).

## Use it — local loop

```bash
npm run sync:serve            # start the localhost receiver
# In Figma: plugin → Local server tab → "Read & Sync to repo"
git diff frontend.config.json # review, then commit
```

Or sync from a downloaded export via the CLI:

```bash
npm run sync -- export.json [--dry-run] [--report]   # --report = coverage, write nothing
```

> Restart the server after editing `transform.a17.mjs` (Node caches modules at start; the
> CLI runs fresh). Port: `FIGMA_SYNC_PORT=xxxx` — also update `plugin/manifest.json`.

## Use it — GitHub PR loop (no local server)

> **One-time setup in the consuming app:** copy `github-workflow.template.yml` to
> `.github/workflows/figma-token-sync.yml` (or run `install.sh`), and enable Settings →
> Actions → General → Workflow permissions → *Allow GitHub Actions to create and approve
> pull requests*.

On the plugin's **GitHub** tab, set the repo (`owner/name` — the *consuming app*) and a
fine-grained PAT (Contents: Read/Write), then **Sync to GitHub & open PR**. The plugin
commits the export to `figma-sync/incoming`; that push triggers the Action, which
transforms it and opens/updates a single PR — only if the config drifted. Repo/token are
remembered in Figma `clientStorage` (local). The export never reaches `main`.

**Check drift vs GitHub** answers "does Figma differ from the repo?" without a sync. It
fetches the committed config *and the repo's own `transform.a17.mjs` + `diff.mjs`* off the
default branch, runs the transform in the plugin, and diffs — so it always runs the same
logic as CI. It reports *that* they differ, not which side is right. (Prototype: assumes
`frontend.config.json` at the repo root; loads the modules via `eval` in the UI iframe.)

## Installing the plugin (once)

Figma → **Plugins → Development → Import plugin from manifest…** → pick
`tokens/figma-sync/plugin/manifest.json`. Runs locally, no publishing; network limited to
`localhost:41789` and `api.github.com`.

## Tuning & robustness

If your Figma names differ, run a real export through `--report` and adjust `CONFIG` in
`transform.a17.mjs` (`collections`, `structure` leaf names, `breakpoints` /
`breakpointAlias`, `spacingPrefix` / `spacingScaler`, `fontFamilyStacks`).

**Solid:** structure, colors, families, typesets. **Verify on first run** (flagged in
`report.notes`): **spacing** (numeric `space/*` → a17 `groups`), **container** (defaults to
`"auto"`; real widths you set are preserved across syncs), and **dark theme** (a17 color is
single-value, so only light values emit).

## Notes

- **One-way.** Per-token docs live in Figma variable/text-style descriptions. Editing a
  Figma-owned value in the config is futile — the next sync reverts it.
- **Reconciliation.** A sync rebuilds only what Figma owns and carries forward the rest —
  `structure.container`, `ratios`, app-added keys (listed in `report.preserved`).
- **Exports aren't tracked** — gitignored; the GitHub flow commits them only to
  `figma-sync/incoming`.
