# Tech stack

Settled (D-094 through D-097, D-101, D-102) — this file owns the full treatment behind those [decision log](../decisions/log.md) entries.

## Backend

- **Language: Python** — for the hub, the runner, and the CLI (D-094).
- **Libraries:** FastAPI (HTTP APIs + SSE), SQLAlchemy (persistence), click (CLI), uv (packaging/venv) — and, as further needs arise, the libraries already standard across winter-cli and the other winter projects rather than new picks (D-094).
- **SQL is configurable on both daemons** (D-095). The hub store and the runner store each run on **sqlite or postgres**, selected by configuration — neither daemon may depend on backend-specific behavior. sqlite is the fast local default and what tests run against ([testing.md](./testing.md)); postgres support is held by staying inside SQLAlchemy's portable surface, not by a duplicated test matrix.

## Quality toolchain (D-101)

- **Python:** ruff (lint + format) and pyright.
- **TypeScript/Angular:** eslint. **No prettier** — formatting concerns that matter are eslint rules; a second formatter is a second opinion.
- Enforced by the CI gates in [build.md](./build.md) from the first commit.

## Logging (D-102)

**structlog**, with output selected by configuration: **JSON renderer** when agents, services, or CI consume the logs; a **human console renderer** (colored key-value) for interactive runs — default chosen by TTY detection, overridable by config/env.
One logging call-site convention regardless of renderer; log-level conventions follow the winter standards the harness carries over.

## Frontend

- **Angular**, as a plain client-rendered SPA — **no SSR, no prerendering** (D-096).
- The frontend is **one Angular workspace, two thin application entrypoints**: a hub app and a runner app, each composing shared library projects — fleet views live once in a shared library, the runner app adds the local-panel library. Both builds ship as assets inside the one binary and are served by the daemons' `host` personalities — that shape is owned by [design/runner/web-app.md](../design/runner/web-app.md) (D-048/D-061).

### Frontend libraries (D-097)

| Concern | Choice |
|---------|--------|
| UI components | **Angular CDK + custom components — no Material.** The board's design language is the bespoke mission-control aesthetic ([discovery/mockups/mission-control.html](../discovery/mockups/mission-control.html)); a styled component library would fight it. CDK supplies unstyled behavior — overlay, virtual scrolling, a11y/focus — and the mockup's `:root` custom-property block ports verbatim as the shared library's design-token stylesheet. |
| Server reads | **TanStack Query for Angular.** Request/response reads (lists, chunk detail, transcripts) go through its query cache and invalidation. |
| Live updates | **Hand-rolled SSE service** in the shared library: native `EventSource` → RxJS → signals, with reconnect-then-re-GET gap recovery. SSE events invalidate or patch TanStack queries so live views stay streaming while the cache stays truthful. The transport swaps to fetch-based SSE when auth arrives (D-018) — `EventSource` cannot set headers; nothing above the service changes. |
| Client state | **Signals** component-locally; **NgRx SignalStore** for shared feature state. Classic NgRx Store is not used. |
| Change detection | **Zoneless**, from day one. |
| Frontend unit/component tests | **Vitest**; Playwright already owns the service and e2e tiers ([testing.md](./testing.md)). |
| Deliberately absent | No chart library, no Tailwind — the board's visuals are typography and CSS grid over the token layer. |

## Hosting Angular + FastAPI as one application

Researched July 2026. Angular's rendering story is `@angular/ssr` with per-route render modes — `Server` (per-request SSR, requires a Node runtime via the `AngularNodeAppEngine` handler), `Prerender` (build-time SSG), and `Client` (CSR, no server runtime at all).

**The choice: CSR, served by FastAPI (D-096).**
FastAPI mounts the Angular build output (StaticFiles with an SPA fallback to `index.html`) and serves it alongside `/api` and the SSE streams from the same process and origin.

- One process, one port, no Node at runtime — the only topology that keeps D-061's one-binary promise with zero caveats; the frontend build is a wheel-embedded asset, and there is no CORS or proxy layer.
- SSR earns its keep on SEO and anonymous cold-visit first paint; an authless, operator-facing board has neither concern.
- Every interesting view is live data over SSE and renders client-side after data arrives regardless — server-rendered HTML would be an empty shell.
- Prerendering was considered and rejected: it demands build-time-enumerable routes (`getPrerenderParams()` for parameterized ones like a chunk detail page) to prerender pages whose content is entirely dynamic.

### Rejected alternatives

- **Prerendered (SSG) build served by FastAPI** — same one-process topology, but pays the route-enumeration build friction above for no dynamic-content benefit.
- **Node SSR server fronting FastAPI** (the `@angular/ssr` Node handler proxying `/api`, or a reverse proxy splitting routes) — full per-request SSR, but a second runtime and process that breaks the one-binary shape; a single supervised container is only a packaging variant of the same cost.

### Reversibility

Angular's per-route render modes make CSR a floor, not a ceiling: a future page that genuinely needs SSR or SSG flips its route's mode then — component code does not change shape, and only the serving side (the Node handler) is introduced at that point.

Sources: [angular.dev SSR guide](https://angular.dev/guide/ssr), [FastAPI static files](https://fastapi.tiangolo.com/tutorial/static-files/), [FastAPI deployment](https://fastapi.tiangolo.com/deployment/), [serving a SPA from FastAPI](https://davidmuraya.com/blog/serving-a-react-frontend-application-with-fastapi/), [Angular SSR proxying to an existing API (angular-cli #26819)](https://github.com/angular/angular-cli/issues/26819), [Deploying Angular in 2026](https://dev.to/kafeel-ahmad/deploying-angular-in-2026-an-architects-guide-to-build-once-run-everywhere-34eh).
