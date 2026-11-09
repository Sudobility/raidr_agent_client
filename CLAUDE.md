> **Git policy — never auto-commit or auto-push.** Leave work in the working tree. Run git commit, git push, gh pr create or a release script only when the user explicitly asks in that turn.

# raidr_agent_client

HTTP client for raidr_agent_api plus TanStack Query hooks. All HTTP goes through an injected `NetworkClient`; the client never calls `fetch` directly.

**Package**: `@sudobility/raidr_agent_client`, published to **GitHub Packages** (`npm.pkg.github.com`), not npmjs.

## What raidr_agent is

A native agent app. It takes a user's request in plain language and:

1. finds the intent via **shapeshyft** (a hosted structured-AI endpoint);
2. filters the **raidr** catalog of MCP servers by label to the sites that can serve that intent;
3. signs the user in to those sites in an in-app **web view** — site session tokens stay on the device, never sent to the raidr_agent backend;
4. calls the sites through **raidr's hosted MCP** endpoint (raidr_api);
5. renders the results with the **Vercel AI SDK**.

## Layering

```
raidr_agent_types -> raidr_agent_client -> raidr_agent_lib -> raidr_agent_app_rn
raidr_agent_types ------------------------------------------> raidr_agent_api
```

- `raidr_agent_types` — shared TS types and response helpers (no runtime deps).
- `raidr_agent_client` — this repo: `RaidrAgentClient` + TanStack Query hooks.
- `raidr_agent_lib` — business logic: Zustand stores and hooks composed from client hooks.
- `raidr_agent_api` — Hono + Postgres (Drizzle) backend; Firebase auth; talks to shapeshyft and raidr.
- `raidr_agent_app_rn` — the React Native app (separate repo).

All repos use **Bun only** — never npm, yarn or pnpm.

## Commands (all verified)

```bash
bun install           # resolves @sudobility scope from GitHub Packages
bun run typecheck     # tsc --noEmit
bun run lint          # eslint src/
bun run test:unit     # vitest run (32 tests, 4 files)
bun run build         # tsc -p tsconfig.build.json -> dist/, then fix-esm-imports
bun run verify        # typecheck + test:unit + build
```

## Project structure

```
src/
├── index.ts                        # barrel: re-exports network, hooks, utils + QUERY_KEYS & defaults
├── types.ts                        # FirebaseIdToken, QUERY_KEYS, DEFAULT_STALE_TIME / DEFAULT_GC_TIME
├── network/RaidrAgentClient.ts     # getHealth, getUser, understandIntent (POST /intent),
│                                   # classifyIntent (deprecated alias, string | IntentRequest), prepare,
│                                   # getSiteContext, getSiteAuth, getSiteManifest, getSiteIcon, searchSites,
│                                   # getRuns, getRun, importRun, getLlmPayload, getCandidates, runsUrl
├── hooks/useAgent.ts               # useUnderstandIntent, useClassifyIntent (deprecated), usePrepare,
│                                   # useSiteContext, useSiteAuth, useRuns, useRun,
│                                   # useLlmPayload, useCandidates, useSiteManifest, useImportRun (invalidates runs)
├── hooks/useHealth.ts              # placeholder query hook; the hook pattern to copy
└── utils/raidr-agent-helpers.ts    # createAuthHeaders, createHeaders, buildUrl, handleApiError
```

Tests are colocated `*.test.ts` (vitest); they are excluded from the build via `tsconfig.build.json`.

## Conventions

- **Dependency injection**: `RaidrAgentClient` takes `{ baseUrl, networkClient }`; it never calls `fetch`. Hooks build a memoized client and take a `TokenGetter` (`() => Promise<string | null>`), called fresh per request.
- **Auth**: protected endpoints take a `FirebaseIdToken` (a `string` alias) sent as `Authorization: Bearer`.
- **Responses**: the client wraps everything in `BaseResponse<T>` and runs `validateResponse` (checks the `success` boolean) before returning. Hooks `unwrap` it and throw on `!success`.
- **Query keys** are namespaced `['raidr_agent', ...]`; add new keys to `QUERY_KEYS` in `types.ts` (currently: health, user, siteAuth, runs, run, candidates, siteManifest, siteIcon, siteSearch, siteContext).
- `getSiteIcon` and `searchSites` exist on the client but have no hook yet.

## Gotchas

- **Build**: `scripts/fix-esm-imports.mjs` runs after `tsc` to add explicit extensions/`index.js` to relative specifiers (tsc uses `moduleResolution: "bundler"` and emits them verbatim, which Node's ESM loader rejects). It is idempotent and never fails the build; it only warns on unresolvable specifiers.
- **Private registry**: `bun install` needs a GitHub token with `read:packages` in `~/.npmrc` (`//npm.pkg.github.com/:_authToken=...`). The `@sudobility` scope is mapped to GitHub Packages in `.npmrc`.
- Placeholder/scaffold exports: `getHealth()` + `useHealth` hook, and `getUser()` (kept from the template).
- `classifyIntent`/`useClassifyIntent` are deprecated aliases for `understandIntent`/`useUnderstandIntent`; a bare string is sent as `{ request }`.

## Origin

Scaffolded from the mogulgame repos (themselves a copy of the company "starter" template), with the game domain removed.
