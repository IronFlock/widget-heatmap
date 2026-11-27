# widget-heatmap development instructions

## Working agreements

The organization maintains these agreements in the IronFlock `.github` repository at `.ai/working-agreements.md`.

- Check the branch and working tree before editing. On main/master, create a
  topic branch; otherwise keep the current branch. If another session owns
  the branch or has uncommitted edits, ask where this work belongs before editing.
  Do not create a worktree without explicit consent.
- Preserve other sessions' changes. Re-read a file immediately before changing
  it; do not overwrite a file that appeared after inspection.
- Commit or push only when Marko explicitly requests it for this piece of work.
  Do not amend, merge, rebase, tag, stash, or change remote settings implicitly.
  Never add attribution trailers or generated-by notices to Git messages or PRs.
- Production changes, releases, and registry pushes require explicit authorization
  for the exact action. Do not hide production mutations in validation recipes.
- Never print secret-file contents or copy credentials into project knowledge.
  Do not describe private signing-key whereabouts in public documentation.
- For code changes, add user-visible release notes for the version that will
  actually ship them, preserving existing entries. Documentation-only setup,
  tests, and invisible refactoring need no invented release-note entry.

## Shared knowledge

Store durable, verified project facts in the repository documentation; store task status and
remaining work in `docs/handoff.md`. Include evidence paths and distinguish
observations from assumptions. Shared files are reviewed repository content,
not an automatic mirror of either agent's private memory or chat history.

Skills live in `.agents/skills/`; `.claude/skills/` contains relative links to
those same folders. Edit the canonical skill, not a duplicate. Tool permissions,
credentials, hooks, and agent-specific runtime settings remain separate.

Read the relevant repository documentation when starting work. When resuming work, read the relevant
handoff and verify it against the current branch and files. Record checks
and remaining work there when handing a task to another developer or agent.

## Commands

- `npm run build` — Rollup bundle to `dist/widget-heatmap.js` (ESM, sourcemaps).
- `npm run watch` — Rollup in watch mode.
- `npm start` — Runs `watch` + `@web/dev-server` (`wds`) concurrently; serves `demo/index.html` (config opens `/demo/`).
- `npm run types` — Regenerates `src/definition-schema.d.ts` from `src/definition-schema.json` via `json-schema-to-typescript` (`json2ts`). Run this whenever `definition-schema.json` changes.
- `just lint` — what the CI gate checks besides the build: regenerating the committed generated files (`npm run types`) changes nothing. `just build` = `npm run build`.
- `just merge` / `just release-patch` (`release-minor`, `release-major`) — pull requests into `main` from a feature branch with everything committed: a no-release one, or a release one whose merge publishes to npm in CI (`.github/workflows/build-publish.yml`), then the release GitHub App tags the bare version and creates the GitHub release. Never `npm publish`, tag or edit the version by hand.
- `npm run link` / `npm run unlink` — Build and `npm link` this package into the studio checkout at `../../studio` (the host app, `~/git/ironflock/studio`), or put the published package back there. Only studio's appliance build (`build:appliance`) reads the widget from `node_modules`: its dev server (`npm run dev`) already aliases `widget-heatmap` to this checkout's `dist/` and serves its `src/*.json`, so `npm run watch` here is enough for dev. Studio tracks both `package-lock.json` and `bun.lock` (its Dockerfile runs `bun install --frozen-lockfile`), and `unlink` reinstalls with npm only: run `bun install` in studio as well if that moved the version.

No test runner is configured.

Node `>=24.9.0`, npm `>=10.0.2` (CI uses Node 24).

## Architecture

Single-file LitElement web component published as `@ironflock/widget-heatmap`. Part of the wider `ironflock-widgets` collection — sibling repos (widget-linechart, widget-gauge, etc.) share the same shape.

### Versioned custom element tag

The element is registered as `widget-heatmap-versionplaceholder` in source. Rollup's `@rollup/plugin-replace` substitutes `versionplaceholder` with `npmPackage.version` from `package.json` at build time (see `rollup.config.js`). The demo and host platform construct the tag dynamically, e.g. `widget-heatmap-${packageJson.version}`. This lets multiple versions of the same widget coexist on one page in the IronFlock dashboard.

### Entry point and integration contract

- Source entry: `src/widget-heatmap.ts` — the single `WidgetHeatmap` LitElement.
- Two reactive `@property` inputs are the entire host contract:
  - `inputData: HeatmapConfiguration` — typed by `src/definition-schema.d.ts`, generated from `src/definition-schema.json`.
  - `theme: { theme_name: string; theme_object: any }` — registered with ECharts via `echarts.registerTheme`. Theme colors are also overridable through the CSS custom property `--re-text-color` read in `registerTheme()`, which takes precedence over `theme_object`. These are not snapshotted: `registerTheme()` stores a `var(--re-…, <theme value>)` chain, so a change to the host property repaints the tile live without the widget being told. The widget paints no background of its own: studio's `widget-item` already paints `--re-tile-background-color` on the tile, and painting it again here composited a translucent tile colour twice.
- `src/definition-schema.json` is the source of truth for the widget's configurable schema; the IronFlock platform uses it to render the widget configuration UI. AI-readable `title`/`description` fields drive both UX and downstream agent tooling — keep them meaningful, then run `npm run types`.
- `src/default-data.json` documents the expected `inputData` shape (used by the demo).

### Data flow

`update()` reacts to `inputData` and `theme` changes:

1. `transformData()` — pivots `inputData.dataseries[].data` by an optional `pivot` field. Each distinct pivot value yields its own ECharts instance keyed in `canvasList: Map<label, { echart, series, element, doomed }>`. Old charts are marked `doomed` at the start of each run and disposed if not refreshed.
2. `setupChart(label)` — lazily creates a `<div class="sizer">` inside `.chart-container` and an `echarts.init` instance per pivot.
3. `applyData()` — merges the pivoted series into a cloned `template` `EChartsOption`, applying axis types (`xAxisType()`/`yAxisType()` auto-detect numeric vs. category vs. timeseries), visualMap colors (with theme/default fallbacks), and calls `setOption` with `notMerge` when the series count changes.

A `ResizeObserver` on `.chart-container` resizes every chart on host resize.

### ECharts tree-shaking

Only the components actually used are registered via `echarts.use([...])` at the top of `widget-heatmap.ts` (`HeatmapChart`, `Tooltip/Grid/VisualMap/Title` components, `CanvasRenderer`). Add to that list when introducing new ECharts features — otherwise they will be missing at runtime.

### Build pipeline

Rollup (`rollup.config.js`) with: `replace` (NODE_ENV + version), `typescript`, `node-resolve`, `commonjs`, `babel` (bundled helpers). Single ESM output to `dist/`. `treeshake.moduleSideEffects: false` is required for ECharts tree-shaking to work.

## `aiSelection` in `src/definition-schema.json`

The schema root carries an `aiSelection` block next to `title` and `description`. It is **not** JSON Schema and describes no config field — it exists so the IronFlock AI's Widget Builder can pick the right widget for a given shape of data, using knowledge only the widget author has:

```jsonc
"aiSelection": {
  "dataShape": "…what columns this widget consumes and what each one means…",
  "useWhen":   ["…a situation, naming the properties that express it…"],
  "notFor":    ["…a situation this widget is wrong for, naming the widget to use instead…"]
}
```

It is inert everywhere else, and must stay that way: `json2ts` ignores it (the generated `.d.ts` is byte-identical with and without it), the dashboard config editor renders only `schema.properties`, and the AI service's `validate_widget` validates *configs* against the schema, skipping unknown Draft-7 keywords.

When maintaining it:

- `notFor` is the high-value half and the part plain descriptions always omit. Every entry must name the widget that *should* be used, or it rejects without routing.
- Write for an LLM with no other documentation: describe the visible result and the user's intent, not the implementation.
- Prefer entries that discriminate against a *neighbouring* widget. Generic rejections are cheap; the ones that pay are those an author could plausibly get wrong.
- The `notFor` lists are a set across all `widget-*` repos and are meant to be reciprocal — if this widget routes to another for some case, that widget should usually route back for the converse. Changing one side is a cue to check the other.
- Update it whenever a property changes what this widget can *do*, not just how it looks.
