---
name: personal-project-bootstrap
description: Interactively scaffold a runnable personal full-stack Bun monorepo with TypeScript, React, tRPC, Tailwind, shadcn/ui, and optional SQLite. TRIGGERS - bootstrap personal project, new personal project, create Bun React app, scaffold tRPC monorepo, start a new app, personal project starter.
disable-model-invocation: true
---

# Personal Project Bootstrap

Create a **runnable, verified repository**, not merely a plan. This is the
standard baseline for a new personal web project unless the user explicitly
overrides a choice.

## Outcome

Deliver a Bun workspace with a React SPA and tRPC API on one origin. It must
install, type-check, lint, test, build, start in Docker, and provide a working
`bun run dev` command with automatic server reload and browser updates.

Do not use Vite, Next.js, Express, Prisma, Drizzle, Turborepo, or a cloud
database by default.

---

## Standard Stack

| Area | Standard |
| --- | --- |
| Runtime and package manager | Bun `1.4.x` |
| Language | TypeScript `7.x`, strict mode |
| Workspace | Bun workspaces only; no task-runner dependency |
| Frontend | React `19` SPA, Bun HTML/bundler flow |
| Routing | File-based TanStack Router |
| API | tRPC v11 with Zod contracts and SuperJSON |
| Default server | Direct `Bun.serve` with `@trpc/server/adapters/fetch` |
| Higher-level server option | Hono plus `@hono/trpc-server`, still hosted by Bun |
| Server data | tRPC TanStack React Query integration |
| Client-only state | React state/context and URL state first; no Zustand by default |
| Forms | React Hook Form, Zod, and `@hookform/resolvers` |
| CSS | Tailwind CSS v4, CSS-first configuration, `tailwindcss-bun-plugin` |
| Components | shadcn/ui; start with a minimal shell and Button |
| Dashboard components | BoardUI only when the project needs dashboard-oriented UI |
| Persistence | Ask first; when needed, SQLite with `bun:sqlite`, Kysely, and migrations |
| Configuration | Zod validation at the environment boundary |
| Logs and health | Pino structured logs and `GET /healthz` |
| Quality | Biome, Bun test, React Testing Library, TypeScript type-checking |
| CI | GitHub Actions runs install, checks, tests, and build |
| Containers | Production `Dockerfile` and `.dockerignore` |
| License | MIT |
| Git | Initialize locally and make an initial commit after verification; do not create or push a remote |

Use current compatible semver ranges for application dependencies. Commit the
generated `bun.lock`; do not hand-write a lockfile or pin every dependency
exactly. Pin the runtime through `mise.toml`, `packageManager`, and `engines`.

---

## Startup Interview — Ask Before Creating Files

Keep this interview short. Do not repeat the settled defaults above.

1. **Project identity:** What is the project name, target directory, one-line
   purpose, and preferred package scope? Refuse to overwrite a non-empty
   directory without explicit confirmation.
2. **Server mode:**
   - **Default — direct Bun.serve:** minimal HTTP surface, one origin, and
     first-party tRPC Fetch adapter.
   - **Hono option:** choose this for richer middleware/routing, many webhooks
     or non-tRPC HTTP endpoints, portability needs, or a deliberately larger
     HTTP layer. Use `hono` and `@hono/trpc-server`, with `app.fetch` supplied
     to `Bun.serve`.
3. **Persistence:** Does the first version need durable data? If yes, use
   SQLite unless the user gives a concrete reason for another database. Ask for
   the initial entities and relationships, not a speculative schema.
4. **Authentication:** Ask explicitly. If needed, establish the identity
   providers, session model, roles, protected routes/procedures, and secret
   storage before selecting an auth library. Do not silently install Better
   Auth or another auth system.
5. **Real-time needs:** Default to none. If needed, distinguish:
   - queries/mutations: Fetch adapter;
   - one-way live updates: tRPC v11 SSE with `httpSubscriptionLink`;
   - truly bidirectional WebSockets: discuss the community
     `trpc-bun-adapter` trade-off before adopting it.
6. **Initial UI:** Confirm whether the product needs dashboard UI immediately.
   The default is a minimal branded shell plus a shadcn Button; do not add
   BoardUI components until they solve a real dashboard need.

Restate the resulting choices in a compact implementation summary and get the
user's confirmation if a choice materially affects scope, credentials, or
data modeling.

---

## Guardrails

- Verify Bun is `1.4.x` before installing. Generate this project-local file:

  ```toml
  # mise.toml
  [tools]
  bun = "1.4.0"
  ```

  Also set `packageManager` to `bun@1.4.0` and constrain `engines.bun` to the
  `1.4.x` line in the root `package.json`.
- Bun transpiles but does not replace type-checking. A TypeScript 7 type-check
  command is mandatory. Avoid tools that require an unstable TypeScript
  compiler API unless their TypeScript 7 support has been verified.
- Keep the SPA and API on one Bun origin in development and production. Use
  relative `/trpc` URLs; do not add CORS or a development proxy without a
  concrete cross-origin requirement.
- Use narrow ordinary HTTP routes only for static assets, `/healthz`, webhooks,
  uploads, OAuth callbacks, or similarly non-tRPC boundaries. App queries and
  mutations belong in tRPC.
- Never place secrets in source, examples, logs, images, or the initial commit.
  Include `.env.example` with non-secret placeholder values and ignore `.env*`
  while preserving `.env.example`.
- Do not introduce state-management, real-time, auth, database, dashboard, or
  deployment complexity merely because it might be useful later.
- `tailwindcss-bun-plugin` is the selected Bun plugin. Before writing its
  configuration, inspect the installed release's README/API and register it
  through Bun's currently supported plugin mechanism. Do not silently replace
  it with a Tailwind CLI or Vite plugin if it has drifted; explain the verified
  incompatibility and ask the user instead.

---

## Required Repository Layout

Use the project's normalized slug as the workspace scope, for example
`@garden-journal/web`.

```text
.
├── apps/
│   ├── server/                 # Bun host, HTTP boundaries, tRPC router/context
│   └── web/                    # React SPA, HTML entry point, routes, UI
├── packages/
│   ├── shared/                 # Zod schemas, domain types, serialization helpers
│   └── db/                     # Only when persistence is selected
├── .github/workflows/ci.yml
├── .dockerignore
├── .env.example
├── .gitignore
├── biome.jsonc
├── bunfig.toml
├── Dockerfile
├── LICENSE
├── mise.toml
├── package.json
├── README.md
├── tsconfig.base.json
└── tsconfig.json
```

### Package Boundaries

| Package | Owns | May depend on |
| --- | --- | --- |
| `apps/web` | React UI, TanStack Router routes, tRPC client, forms | `packages/shared`; type-only `AppRouter` export from server |
| `apps/server` | Bun entry point, HTTP routes, tRPC context/router, logging | `packages/shared`; `packages/db` when selected |
| `packages/shared` | Zod schemas and inferred domain types used on both sides | Zod/SuperJSON-compatible utilities only |
| `packages/db` | `bun:sqlite` connection, Kysely database type, migrations, repositories | Kysely and SQLite adapter only |

The web app's `AppRouter` import must be type-only, so no server implementation
is bundled for the browser. Do not duplicate request/response interfaces: define
Zod schemas in `packages/shared`, infer their types, validate inputs again in
tRPC procedures, and reuse those schemas in React Hook Form resolvers.

Keep persistence behind `packages/db`; neither React components nor shared
contracts may import `bun:sqlite`, Kysely, or server environment values.

---

## Build the Baseline

### 1. Root Workspace and Tooling

Create the root workspace manifest with `apps/*` and `packages/*` workspaces.
Include scripts with these stable user-facing names:

```text
bun run dev        # dynamic route generation plus full-stack Bun development
bun run build      # generate routes, then produce production assets/server
bun run test       # Bun tests, including React Testing Library tests
bun run lint       # Biome linting
bun run format     # Biome formatting
bun run typecheck  # TypeScript 7 check across all packages
bun run check      # lint + typecheck + test
bun run start      # production Bun server
```

Use Bun's native parallel script execution for `dev`; do not add `concurrently`
or another orchestration package. The root development command must at minimum:

1. run `tsr watch` in `apps/web` to maintain its generated route tree;
2. run the Bun server in `--hot` or `--watch` mode; and
3. make browser changes visible automatically through Bun's current full-stack
   development/HMR support. A full browser refresh is acceptable; manual
   rebuilding or refreshing is not.

Use `tsr generate` before type-checking and production builds so a fresh clone
does not rely on a stale generated route tree. Configure file-based routes with
`tsr.config.json` in `apps/web`; do not add a Vite plugin just to generate them.

Configure strict TypeScript with React's automatic JSX runtime and
`moduleResolution: "bundler"`. Generate a root project configuration that checks
each workspace without emitting duplicate JavaScript. The Bun build owns emitted
application assets.

Set up Biome before adding application code. Its check must cover TypeScript,
TSX, JSON/JSONC, YAML, and Markdown where supported by the installed version.

### 2. Browser Application

Create a React 19 SPA that Bun serves from the same origin as the API. Use Bun's
HTML import/bundling path rather than Vite. The server owns the SPA fallback for
TanStack Router deep links.

Set up:

- file-based TanStack Router routes, including a root route and index route;
- `@tanstack/react-query` plus tRPC's TanStack React Query integration;
- a `QueryClient` and typed tRPC options proxy in the router context so route
  loaders can prefetch and invalidate API data;
- an `httpBatchLink` targeting relative `/trpc` and configured with SuperJSON;
- React Hook Form plus `@hookform/resolvers/zod`;
- only React state, context, and URL state for local UI state. Do not install
  Zustand initially;
- a minimal product-named shell and one shadcn Button component.

Use Tailwind v4's CSS-first form. The global stylesheet begins with:

```css
@import "tailwindcss";
```

Initialize shadcn/ui in `apps/web` with valid TypeScript path aliases and the
Tailwind v4 theme tokens. Use its current Bun-compatible CLI invocation, then
add only the Button. Do not generate a generic dashboard.

When a dashboard is actually selected, configure BoardUI as a shadcn-compatible
registry and add only the chosen components (for example `@boardui/<component>`).
BoardUI Pro licensing must be confirmed before installing a Pro component.

### 3. API and Server Modes

#### Default: Direct Bun.serve

Use a single `Bun.serve` handler for the SPA, `/trpc`, `/healthz`, and approved
HTTP escape hatches. Mount tRPC HTTP queries and mutations using the first-party
Fetch adapter:

```ts
import { fetchRequestHandler } from "@trpc/server/adapters/fetch";

return fetchRequestHandler({
  endpoint: "/trpc",
  req,
  router: appRouter,
  createContext,
});
```

Create a small context containing validated server configuration, the Pino
logger, and the database only when selected. Export `type AppRouter` from the
server router. Configure the same SuperJSON transformer in the server tRPC
initializer and client.

Add `GET /healthz`, returning a small non-secret health response. Log startup
and unexpected failures with Pino's structured API; keep request bodies,
credentials, and sensitive headers out of logs. Return typed tRPC errors rather
than leaking stack traces.

#### Higher-Level Option: Hono

If the user selects Hono, make Hono the request-composition layer while Bun
remains the runtime/server. Use `hono` plus `@hono/trpc-server`, mount tRPC at
`/trpc/*`, and pass `app.fetch` to `Bun.serve`. Preserve the same one-origin
SPA, `/healthz`, Zod/SuperJSON contracts, logging, and package boundaries.

Do not add Hono to the default direct-Bun scaffold.

### 4. Optional SQLite and Kysely

Only create `packages/db` when persistence was selected. Use `bun:sqlite` with
Kysely and a Bun-compatible SQLite dialect (currently `kysely-bun-sqlite` after
verifying its installed API). Keep the database file outside committed source,
typically beneath an ignored runtime `data/` directory controlled by a validated
`DATABASE_PATH` environment value.

The database package must include:

- a typed Kysely `Database` interface;
- a single database factory/connection boundary;
- SQLite pragmas appropriate for an application database, including foreign keys
  and a busy timeout; use WAL only after confirming the project's deployment
  filesystem supports it;
- ordered, versioned Kysely migrations and `db:migrate` / `db:migrate:latest`
  scripts runnable with Bun;
- a small initial migration only for the entities the user approved; and
- temporary or isolated SQLite databases for database-dependent tests.

Never implement an ad-hoc schema reset as the production migration strategy.
If the user requests a non-SQLite database, document why SQLite is unsuitable
and obtain explicit confirmation before changing the persistence choice.

### 5. Configuration, Tests, CI, and Docker

Validate environment values once at the server boundary with Zod. Keep client
configuration absent unless it is truly needed; the one-origin tRPC client needs
no API base URL. Include `.env.example` with at least the non-secret port and,
when applicable, database path placeholders.

Use Bun's test runner. Add:

- one server/router unit test that proves a tRPC procedure and validation path;
- one React Testing Library test for the initial UI;
- the minimum DOM test environment/setup required by the chosen Bun test setup.

Generate `.github/workflows/ci.yml` using Bun `1.4.0`. CI must run a frozen Bun
install, `bun run check`, and `bun run build` from a clean checkout.

Generate a production-oriented, multi-stage `Dockerfile` and `.dockerignore`.
It must build the workspace, expose the configured server port, run the Bun
production entry point, and retain a usable location for an optional mounted
SQLite data directory. Do not add Compose unless the user later has a genuine
multi-service local-development need.

Write a short README covering prerequisites, `mise install`, installation,
development, checks, migrations when present, Docker build/run, environment
variables, and the selected server mode.

---

## Verification and Handoff

Do not call the scaffold complete until all applicable checks pass:

1. `mise install` installs/activates Bun 1.4 when mise is available, and
   `bun --version` reports a `1.4.x` runtime.
2. `bun install` completes from the workspace root and creates `bun.lock`.
3. Route generation succeeds from a clean state.
4. `bun run check` and `bun run build` pass.
5. `bun run test` passes, including both server and React Testing Library tests.
6. `bun run dev` serves the SPA, `/healthz`, and a tRPC smoke request from one
   origin. Change a harmless route or component and verify the server/browser
   updates without manually restarting or refreshing.
7. When Docker is available, build the image, run it, and verify `/healthz`.
   If Docker is unavailable, state that the Docker smoke test was not run.
8. Inspect `git status`, the diff, ignored files, and the lockfile. Confirm no
   secret, database file, build output, or dependency directory is staged.

Initialize a local Git repository and add the generated project files only.
After the checks pass, create the initial Conventional Commit:

```text
chore: bootstrap project
```

Respect the active harness's commit-authorization policy: if it requires an
explicit final confirmation, show the staged file list and ask before committing.
Never create a GitHub repository, add a remote, or push unless the user asks in
that invocation.

End with the project path, server mode, database/auth/realtime choices, exact
commands that passed, Docker verification status, and the initial commit SHA.
