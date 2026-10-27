# Fork notes

This repository is a personal fork of
[Fisfzy/dsh-ego-browser](https://github.com/Fisfzy/dsh-ego-browser).
`master` tracks upstream; the fixes below live on the `fixes` branch.

Nothing here is intended as an upstream pull request. The goal is to keep a
small, well-separated set of local fixes that can be carried across upstream
refreshes with minimal conflict.

## What this fork changes

| # | Defect | Location | Upstream has source? |
|---|---|---|---|
| 1 | `ego_tabs switch` / `open` did not survive into the next tool call | `src/index.ts` | yes |
| 2 | `ego_scroll` reported pre-scroll offsets | `src/index.ts` | yes |
| 3 | `waitForRequest` / `waitForResponse` never matched glob patterns | `runtime/ego-browser/dist/out/index.js` | **no** |
| 4 | `task space not found` did not name the existing spaces | `runtime/ego-browser/dist/out/index.js` | **no** |

Each fix is marked with a `[fork]` comment at the change site. Search for that
marker to find every divergence:

```sh
grep -rn '\[fork\]' src runtime
```

## Why two of them are patched in a build product

`runtime/ego-browser/` is vendored from
[CitroLabs/ego-lite](https://github.com/CitroLabs/ego-lite). `tsdown.config.ts`
builds only `src/index.ts`, `src/client/index.ts` and
`src/worker/ego-cast-worker.ts` — it never rebuilds `runtime/`. So fixes 3 and 4
can only be applied to the committed bundle, exactly the way upstream already
tracks its own vendored changes in `runtime/PATCHES.md`.

## Merging an upstream refresh

```sh
git remote add upstream https://github.com/Fisfzy/dsh-ego-browser.git
git fetch upstream
git checkout master
git merge upstream/master        # or: git reset --hard upstream/master
git checkout fixes
git rebase master
```

Conflict expectations:

- **`src/index.ts` (fixes 1, 2)** — small, local edits near `ensureRealTab` and
  the `ego_scroll` tool. Should merge cleanly unless upstream rewrites those
  areas.
- **`runtime/ego-browser/dist/out/index.js` (fixes 3, 4)** — a single generated
  file. Any upstream ego-lite bump rewrites it wholesale and this WILL conflict.
  Re-apply by hand:
  1. `networkMatches()` — the `typeof matcher === "string"` branch. Add glob
     matching via the module's existing `globToRegExp` (do not add a second
     copy; one already exists for `urlMatches`).
  2. `waitForNetworkMatch()` — replace `void networkEvents.ready` with
     `await networkEvents.ready`.
  3. Three `throw new Error(\`task space not found: ${nameOrId}\`)` sites — add
     `${describeAvailableSpaces(spaces)}` and copy the helper from this fork.
- **`lib/index.js`** — generated from `src/index.ts`. Do not hand-merge; run
  `pnpm build` after resolving `src/index.ts`.

## Rebuilding

```sh
pnpm install --ignore-scripts
pnpm build
```

`lib/client.js`, `lib/client.js.map`, `bin/ego-cast-worker.mjs` and
`bin/ego-cast-worker.mjs.map` are rebuilt too but this fork changes nothing in
those entries, so they are restored to the upstream bytes afterwards to keep
the diff limited to `lib/index.js`.

Note: upstream does not commit `pnpm-lock.yaml` (it is gitignored), so the
bundler version floats. A build with a different rolldown release renames
internal locals (`resolve` -> `resolve$1`) and inflates the `lib/index.js`
diff with non-functional churn. Only `src/index.ts` is authoritative.

## Verification performed

Run against a real Chrome 127 on Windows, headless:

- `waitForResponse('**/robots.txt')` matched a response (`status=404` for
  `example.com/robots.txt`, i.e. the glob matched, not the status).
- `ego_tabs switch` then `ego_js` returned the switched tab's URL; repeated
  switches stayed correct; closing the pinned tab fell back cleanly.
- `ego_scroll({deltaY: 300})` returned `scrollY: 300` matching
  `window.scrollY`.
- `ego_space_close({name: 'nope'})` reported `(available: dsh-agent)`.

## Not fixed here

- `waitForResponse(...).text()` / `.body()` still fail with
  `No resource with given identifier found`: CDP discards the response body
  before it is read. Use `ego_http` with `mode: "server"` or read the body
  inside `page.evaluate()` instead.
- `ego_script` does not inject `ensureRealTab()`, so it resolves the "current
  tab" independently of the structured tools. Pass an explicit
  `browser.switchTab(...)` inside the script when mixing the two.
- `navigator.webdriver` is `true` in the headed path (the automation flag is
  only stripped on the headless launch).
