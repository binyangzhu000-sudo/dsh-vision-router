# Web Client development reload (Issue #697)

`pnpm dev:web` is an **opt-in developer tool**. It watches the canonical
`src/` tree and relevant build scripts, checks the generated maintenance
Client code, runs the existing `pnpm build`, checks the runtime entrypoints,
and verifies the source-to-artifact map before reporting a successful build.
Builds never overlap. A failed/incomplete build does **not** trigger any
browser reload, and the watcher keeps working after an error.

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
