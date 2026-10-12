# Web Client development reload (Issue #697)

`pnpm dev:web` is an **opt-in developer tool**. It watches the canonical
`src/` tree and relevant build scripts, checks the generated maintenance
Client code, builds the same 183 artifacts into an isolated temporary directory,
verifies the source-to-artifact map **before publication**, and atomically
publishes the entry files with `lib/client.js` **last**. Builds never overlap.
A compile/verification failure leaves every previously published artifact
untouched, preventing upstream DSH's independent Client-HMR stat watcher from
mistaking an incomplete build for a new revision. Watcher errors do not kill
future builds. The formal `pnpm build` / release packaging remains unchanged.

Start in build-only mode:

```sh
pnpm install --frozen-lockfile
pnpm dev:web
```

### Optional automatic Web refresh

Start a **throwaway DSH Web profile** using the plugin entry from the *same*
checkout (`entry.js` or the appropriate compiled package entry), not a stale
installed npm tarball. Then, in the checkout's developer terminal:

```sh
export DVR_DEV_WEB_URL='http://127.0.0.1:PORT/'
export DVR_DEV_CHROME='/absolute/path/to/Google Chrome'
pnpm dev:web
```

The script opens a private Chrome window after the first successful build and
performs a full-page reload after subsequent successful builds. Authenticate
in that window if DSH asks you to. An isolated DSH token URL can be supplied
via the environment variable, but **never paste it into a committed script or
logs**. The program does not print the URL. `DVR_DEV_HEADLESS=1` is available
for automated smoke tests. Only HTTP loopback addresses are allowed; remote
web pages, remote debugging ports, and external browser requests are blocked.
Chrome is controlled through its private CDP stdio pipe, not via a WebSocket
listener exposed to other users. Stop the watcher with Ctrl+C to close Chrome.

This is **full-page Live Reload, not module-level HMR**: the page, React
component state and Client loader initialize from scratch. This is intentional
because Vision Router has both structured `webserver/index-inject` and
`tapIndex()` script carriers protected by a per-page one-shot registry. In-place
factory reinjection could bypass lifecycle ownership and duplicate Host/UI
effects. No carrier registry changes, no production runtime dependency, no
changes to package exports or Host authority are part of this feature.

If `src/lib/client-maintenance-actions.js` changes, the embedded generated
section in `src/lib/client.js` must be synchronized deliberately by running
`pnpm client:sync` before the watcher can pass the embedded-source check.
It is deliberately **not** rewritten by the watcher: an automated source
rewrite would conceal unreviewed Client changes.

**Scope:** Web-only development. Desktop's custom scheme is unverified and
must not be treated as HMR. Source changes that affect Host runtime services
or settings require an isolated Host restart; reloading Chrome alone does not
reload Host modules. Run established Settings and Issue #684 ON→OFF real-Host
regression gates separately before claiming all-round compatibility.

## Upstream native Web HMR (opt-in)

DSH 0.2.0-rc.2 already provides its own Web plugin HMR transport:
`@deepseek-ai/dsh-client-hmr` stat-polls the canonical built `lib/client.js`
and publishes `rebuilt` events through `/plugins/events`. The browser Client
Modules controller invalidates and replaces the plugin fiber, with **no page
navigation**. It is not React Fast Refresh: local component state inside the
plugin resets, while unrelated Web page/session state survives. Electron's
Desktop scheme does **not** use this SSE transport.

With the isolated DSH Web Host pointing to this checkout's Client artifacts
and the usual `DVR_DEV_WEB_URL` / `DVR_DEV_CHROME` variables, use:

```sh
export DVR_DEV_RELOAD_MODE=native-hmr
pnpm dev:web
```

The dev controller probes the HMR row in DSH's boot manifest and an
HTTP 200 `text/event-stream` response from `/plugins/events`. If either is
unavailable, it closes private Chrome and reports an error rather than
pretending to hot-swap the UI. `DVR_DEV_RELOAD_MODE=page` (the default) keeps
the older full-page fallback.

**Important DVR boundary:** native HMR replaces the DSH `dsh.client` factory,
not the Host-injected `client-script-carrier` bootstrap JS/CSS or server-side
settings bridges. The latter retain their one-shot per-page fence and may
need a Host restart and a fresh page to update. Do not claim that changing
Host-side JS automatically updates those browser carriers. Do not interpret
HMR as fixing Issue #684's Host selection/recovery bug; that needs its own
real Desktop ON→OFF evidence.

### Why a staged developer build is necessary

The production artifact builder removes `lib/`, `entry.js`, and `index.js`
before compiling to their final paths. That is acceptable for CI/release
packaging, but **unsafe as a development HMR trigger**: a TypeScript compiler
failure can leave deleted `.js` outputs and still change the live `client.js`
file metadata while upstream DSH polls it. The developer watcher instead calls
`scripts/dev-staged-build.mjs` to compile into a temporary folder under the
checkout, validates the complete map using
`verify-source-artifact-map.mjs --artifact-root <stage>`, and publishes files
with atomic rename operations. The watched Client entry is written last.

The verified publication is not a full database transaction; a disk error
halfway through publishing siblings can still need a clean rebuild/restart.
No claim is made that Host runtime modules themselves are hot replaced.

A real DSH 0.2.0-rc.2/Chrome/Node24 acceptance test covered:
`src/lib/client.js` A→B through the dev watcher, invalid TypeScript compile
with old B artifact unchanged, then repair B→C, without any browser navigation
or unexpected JavaScript exceptions.
