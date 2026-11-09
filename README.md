# raidr_agent_client

HTTP client for raidr_agent_api plus TanStack Query hooks. All HTTP goes through an injected `NetworkClient`; the client never calls `fetch` directly.

**Package:** `@sudobility/raidr_agent_client` — published to **GitHub Packages** (`npm.pkg.github.com`) under the `@sudobility` scope.

Part of **raidr_agent**: a native agent app that turns a user request into an intent (via shapeshyft), picks matching raidr MCP servers by label, signs the user in to those sites in a web view (session tokens stay on device), calls them through raidr's hosted MCP, and renders the results with the Vercel AI SDK.

Layering: `raidr_agent_types` → `raidr_agent_client` → `raidr_agent_lib` → `raidr_agent_api` / `raidr_agent_app_rn`.

## What it provides

- `RaidrAgentClient` — an HTTP class constructed with `{ baseUrl, networkClient }`. Methods cover the agent flow (`understandIntent`, `prepare`), per-site lookups (`getSiteContext`, `getSiteAuth`, `getSiteManifest`, `getSiteIcon`, `searchSites`), run history (`getRuns`, `getRun`, `importRun`), local mode (`getLlmPayload`, `getCandidates`), and `runsUrl()` for the streaming endpoint. Every response is checked with `validateResponse` before it is returned.
- TanStack Query hooks (`useUnderstandIntent`, `usePrepare`, `useSiteContext`, `useSiteAuth`, `useRuns`, `useRun`, `useLlmPayload`, `useCandidates`, `useSiteManifest`, `useImportRun`, `useHealth`) that wrap the client methods.
- `QUERY_KEYS`, `DEFAULT_STALE_TIME`, `DEFAULT_GC_TIME` and the `FirebaseIdToken` type from `types.ts`.

Protected endpoints take a `FirebaseIdToken` (sent as `Authorization: Bearer <token>`). Hooks take a `TokenGetter` (`() => Promise<string | null>`) that is called fresh per request.

## Install

Bun only — never npm, yarn or pnpm.

```bash
bun install
```

Dependencies resolve the `@sudobility` scope from GitHub Packages (see `.npmrc`), so you need a GitHub token with `read:packages` in your `~/.npmrc`:

```
//npm.pkg.github.com/:_authToken=<token>
```

## Usage

```typescript
import { RaidrAgentClient, useUnderstandIntent } from '@sudobility/raidr_agent_client';

// Direct client
const client = new RaidrAgentClient({ baseUrl, networkClient });
const res = await client.understandIntent({ request: 'book a table tonight' }, idToken);

// React hook (inside a QueryClientProvider)
const mutation = useUnderstandIntent(networkClient, baseUrl, getToken);
mutation.mutate({ request: 'book a table tonight' });
```

## Commands

```bash
bun run typecheck     # tsc --noEmit
bun run lint          # eslint src/
bun run test:unit     # vitest run
bun run build         # tsc -p tsconfig.build.json → dist/, then fix ESM imports
bun run verify        # typecheck + test:unit + build
```

The build compiles with `moduleResolution: "bundler"` and then runs `scripts/fix-esm-imports.mjs`, which rewrites relative import/export specifiers in the emitted `.js`/`.d.ts` to fully-qualified paths so Node's native ESM loader can resolve them.

## CI/CD

`.github/workflows/ci-cd.yml` calls `johnqh/workflows/.github/workflows/unified-cicd.yml@main` on push and PR to `main` and `develop`. It runs typecheck, lint, tests and build, and publishes to GitHub Packages (`publish-target: github`) from `main`.

## License

BUSL-1.1
