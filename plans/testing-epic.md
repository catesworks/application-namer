# Automated Test Coverage — Implementation Plan

**Epic:** `application-namer-9w9` — (EPIC) Automated Test Coverage
**Scope:** Add a test suite for the *already-implemented* v1 code. No feature work, no app redesign.
**Reference:** `.omc/plans/application-namer-v1.md` (architecture, registry contracts, error schema)

---

## Requirements Summary

The repository currently has **zero test tooling**: no `vitest`/`jest`, no `test` script in `package.json`, and CI (`.github/workflows/ci.yml`) runs only `pnpm lint` and `pnpm build`. Every behavior in the v1 plan — validation rules, timeout/error normalization, cache eviction, per-registry status mapping, GitHub rate-limit detection, AI provider routing, and the API error responses — is unverified.

> **Status codes: what the routes actually emit.** The v1 plan's *API Error Response Schema* documents four codes (400/429/500/503). The implementation emits only **three**, and only from two of the three routes:
>
> | Route | Emits |
> |---|---|
> | `POST /api/check` | `400` (validation), `200` (success). No `catch` block — an unexpected throw is not converted to a `500` by this handler. |
> | `POST /api/suggest` | `400` (validation), `503` (provider failure, from its `catch`), `200` (success). |
> | `GET /api/providers` | `200` only. No error branch at all. |
>
> **No route ever emits `429`.** GitHub rate limiting surfaces as `status: 'rate_limited'` *inside* a `200` body, not as an HTTP status. Likewise, no route emits an explicit `500`. Tests must assert only the codes above; asserting a `429` or a handler-produced `500` would be writing tests against a spec that was never implemented. The gap between the v1 schema and the implementation is logged in Open Questions.

This epic delivers:

1. A test runner and harness configured for a Next.js 16 App Router + TypeScript + pnpm project.
2. **Unit tests** for `src/lib/**` — validation, HTTP client, cache, all five registry checkers, the orchestrator, and all four suggestion providers.
3. **Integration tests** for the three API route handlers, invoked directly as functions (no running Next.js server).
4. **Coverage thresholds** enforced in CI, and a `pnpm test` step wired into `.github/workflows/ci.yml`.

**Explicitly out of scope for this epic:** React component tests (`src/components/**`), the `use-name-check` hook, and browser E2E. These are deferred (see Follow-ups) so the epic stays sized and so coverage thresholds can be scoped tightly to `src/lib/**` + `src/app/api/**`.

**Explicitly out of scope: changing production behavior.** Where the current implementation has a rough edge (see Open Questions), tests **characterize** the existing behavior and a separate bug ticket is filed. Tests must not be written against behavior that does not exist yet.

---

## Acceptance Criteria

1. `pnpm test` runs the full suite locally in watch mode; `pnpm test:run` runs once and exits non-zero on failure.
2. The suite completes in **under 30 seconds** on a cold run and makes **zero real network requests**. Any unhandled outbound HTTP request **fails the test that made it** — enforced by an explicit `request:unhandled` recorder that throws in `afterEach`, *not* by `onUnhandledRequest: 'error'` alone (see Decision 2 for why that setting is insufficient here).
3. `src/lib/validation.ts`, `src/lib/registry-client.ts`, and `src/lib/registries/cache.ts` each have dedicated unit tests covering every documented branch, including the `214`-char boundary, the `4000 ms` default timeout, the `300_000 ms` TTL boundary, and the `1000`-entry eviction cap.
4. All five registry checkers have tests for `available` (404), `taken` (200), `error` (non-ok), and thrown-error/abort paths. GitHub additionally has tests for `rate_limited` (403 and 429), the batched `+OR+` query, and the empty-input early return.
5. All four suggestion providers have tests covering success, auth-error re-throw, silent-`[]` fallback, and malformed-response parsing.
6. Each of `POST /api/check`, `POST /api/suggest`, `GET /api/providers` has integration tests asserting exact HTTP status and exact JSON body shape for **every status code that handler actually emits** — `400`/`200` for check, `400`/`503`/`200` for suggest, `200` for providers. No test asserts a `429` or a handler-emitted `500`, because no route produces them.
7. Coverage over `src/lib/**` + `src/app/api/**` is gated by `vitest run --coverage`, which exits non-zero below threshold. The threshold *values* are fixed once during Step 14's one-time calibration and recorded in `vitest.config.mts`; from that point they are a ratchet (see Decision 8).
8. CI runs `pnpm test:ci` on every push and PR to `main`, ordered after `pnpm lint` and before `pnpm build`.
9. `pnpm build` and `pnpm lint` continue to pass unchanged — adding tests must not slow or break the production build.
10. Tests are deterministic: no reliance on wall-clock sleeps beyond ~200 ms, no shared state leaking between test files or between test cases within a file.

---

## Architecture & Tooling Decisions

### Decision 1 — Test runner: **Vitest 3**

**Chosen: Vitest.** Rejected: Jest.

| | Vitest | Jest |
|---|---|---|
| Next.js 16 support | First-class; official guide ships in `node_modules/next/dist/docs/01-app/02-guides/testing/vitest.md` | Also documented, via `next/jest` |
| ESM | Native. `@anthropic-ai/sdk` and `openai` are ESM-first — no interop shims needed | Needs `transformIgnorePatterns` surgery or SWC/babel transform config |
| `@/*` path alias | Free, via `vite-tsconfig-paths` reading the existing `tsconfig.json` | Manual `moduleNameMapper` duplication |
| Env stubbing | `vi.stubEnv` + `unstubEnvs: true` auto-restore | Manual `process.env` save/restore boilerplate |
| Cold start | Fast (esbuild) | Slower (SWC/babel pipeline) |

Rationale: the code under test is heavily env-var- and module-mock-driven, and both AI SDKs are ESM. Vitest removes the two largest sources of Jest config friction here at zero cost. It is also the runner the bundled Next.js docs recommend for this exact stack.

**Dev dependencies to add:**

```
vitest
@vitest/coverage-v8
vite-tsconfig-paths
msw
```

Deliberately **not** added: `@vitejs/plugin-react`, `jsdom`, `@testing-library/react`. This epic tests only server-side modules, so the whole suite runs in `environment: 'node'`. Adding React/jsdom deps now would be dead weight; they belong to the deferred component-test epic (which should add a second Vitest *project* rather than switching the global environment).

**`globals: false`** — tests import `describe`/`it`/`expect`/`vi` explicitly from `vitest`. This avoids touching `tsconfig.json`'s `types` array and keeps ESLint happy with no new globals config.

### Decision 2 — Mocking external HTTP: **MSW v2 (`setupServer`)**

**Chosen: MSW.** Rejected: `vi.stubGlobal('fetch', vi.fn())`.

Every outbound registry call and every MCP bridge call goes through the **global `fetch`** (`registry-client.ts` calls `fetch(url, { signal: AbortSignal.timeout(...), headers })` directly; `mcp-bridge.ts` calls `fetch` directly). There is no injectable HTTP seam anywhere in `src/lib/registries/**`.

Why MSW over a `fetch` stub:

- **It exercises the real code path.** `AbortSignal.timeout`, `Headers` normalization, `res.ok`, `res.status`, `await res.json()` all behave for real. A `vi.fn()` stub returning a hand-rolled `{ ok, status, json }` object would silently pass even if `registryFetch` stopped passing the signal.
- **URL-shape assertions come free.** Tests can assert the GitHub batch query is literally `q=a+OR+b+OR+c+in:name&sort=stars&per_page=30` by matching on the request URL, instead of string-matching a `vi.fn()` call argument.
- **It models non-JSON bodies.** `mcp-bridge.ts` parses a Server-Sent-Events text body (`data: {...}` lines) with a plain-JSON fallback. MSW returns a real text body; a fetch stub would need a bespoke fake `.text()`.

#### The network firewall needs more than `onUnhandledRequest: 'error'`

`onUnhandledRequest: 'error'` alone **does not reliably fail a test in this codebase.** It makes the unhandled `fetch` *reject* — but every registry module wraps its fetch in `try`/`catch` and converts any thrown error into `{ status: 'error', error: String(err) }`. So an unmodelled request is silently absorbed into a normal-looking `error` result. A test that asserts `status === 'error'` (and several legitimately do) would **pass while secretly attempting a real network call**, and the MSW console error is easy to miss in CI output.

The firewall must therefore be built on MSW's `request:unhandled` event, which fires independently of what the application code does with the rejection:

```ts
// tests/msw/server.ts
import { afterAll, afterEach, beforeAll } from 'vitest'   // globals:false — these MUST be imported
import { setupServer } from 'msw/node'
import { handlers } from './handlers'

export const server = setupServer(...handlers)

const unhandled: string[] = []
server.events.on('request:unhandled', ({ request }) => {
  unhandled.push(`${request.method} ${request.url}`)
})

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))

afterEach(() => {
  server.resetHandlers()
  const seen = unhandled.splice(0)          // splice so one failure doesn't cascade
  if (seen.length > 0) {
    throw new Error(`Test attempted ${seen.length} unmodelled request(s):\n  ${seen.join('\n  ')}`)
  }
})

afterAll(() => server.close())
```

`onUnhandledRequest: 'error'` is kept as a second layer (it prevents the request from actually leaving the machine); the `afterEach` throw is what converts that into a *visible test failure*. Draining via `splice` matters — otherwise the first offending test poisons every subsequent `afterEach` in the file.

Default handlers live in `tests/msw/handlers.ts` and cover the happy path for all five registries. Individual tests override per-case with `server.use(http.get(url, () => HttpResponse.json(body, { status })))`.

> **Note on `globals: false`:** every snippet in this plan that calls `beforeAll` / `afterEach` / `afterAll` / `describe` / `it` / `expect` / `vi` must import them from `vitest`. Snippets here show the imports where they are the point being made and elide them elsewhere; the imports are never optional.

**Timeout/abort testing without fake timers.** `registryFetch(url, options)` already accepts `{ timeout?: number }` and defaults to `4000`. Tests pass an explicit `{ timeout: 50 }` and pair it with an MSW handler that `await delay(200)` before responding. Real elapsed time is ~200 ms, the abort path is genuinely exercised, and no timer patching is required. **Do not** try to fake-time `AbortSignal.timeout` — Node's implementation is not patched by Vitest's fake timers.

### Decision 3 — Mocking the AI SDKs: **`vi.mock` at the module boundary**

**Chosen: module mocks for `@anthropic-ai/sdk` and `openai`.** Rejected: MSW-intercepting the SDKs' own wire traffic.

Both SDK clients are constructed **inside** the exported provider function (`new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY })` per call), not at module scope. That means a module mock is clean and leak-free — every call gets a fresh mock client, and `process.env` is read at call time so `vi.stubEnv` works without module resets.

Why not MSW here:

- The contract worth pinning is the **request options object**, not the wire bytes: `model: 'claude-sonnet-4-20250514'`, `max_tokens: 1024`, `tool_choice: { type: 'tool', name: 'suggest_names' }` for Claude; `model: 'gpt-4o-mini'`, `response_format: { type: 'json_object' }` for OpenAI. A module mock asserts these directly.
- Both SDKs **auto-retry** on 429/5xx. Driving error paths through MSW would either add real retry delays (slow) or require per-test retry config, for no fidelity gain.
- Reconstructing a valid `tool_use` content-block wire response for Anthropic is more brittle than returning the object the SDK would have produced.

**Critical implementation detail 1 — `vi.mock` factories are hoisted.** Vitest hoists `vi.mock(...)` calls above all imports and above every `const` in the file. A factory that closes over an ordinary module-scope `const mockCreate = vi.fn()` throws `ReferenceError: Cannot access 'mockCreate' before initialization` at load time. The shared mock function must be created inside **`vi.hoisted()`**, which is hoisted alongside the factory:

```ts
import { vi } from 'vitest'

const { mockCreate } = vi.hoisted(() => ({ mockCreate: vi.fn() }))
```

**Critical implementation detail 2 — the error class must be real.** Both providers branch on `err instanceof Anthropic.AuthenticationError` / `err instanceof OpenAI.AuthenticationError`. The factory must expose a **constructible class** on that static, or the `instanceof` check falls through to the swallow-and-return-`[]` branch and the 503 test passes for the wrong reason:

```ts
vi.mock('@anthropic-ai/sdk', () => {
  class AuthenticationError extends Error {}
  const Anthropic = vi.fn(() => ({ messages: { create: mockCreate } }))
  // @ts-expect-error attaching a static to a mock constructor
  Anthropic.AuthenticationError = AuthenticationError
  return { default: Anthropic, AuthenticationError }
})
```

Two further requirements when authoring these:

- **Match the source's real import form.** Read the top of `claude.ts` / `openai.ts` first — a default-import source needs `{ default: ... }` in the factory, a named-import source needs the named key. Getting this wrong produces a confusing "not a constructor" failure.
- **Throw the mock's own class in auth tests.** The test must reject with an instance of the `AuthenticationError` returned *by the factory* (import it back from the mocked module), not a locally-declared look-alike — `instanceof` compares class identity, and two structurally identical classes are not the same class.

Assert the exact rethrown message (`"Invalid ANTHROPIC_API_KEY. Check your API key configuration."`) so a broken `instanceof` surfaces as a failure rather than a silent `[]`.

The **MCP bridge** is *not* SDK-based — it is raw `fetch` to `${serverUrl}/mcp`, so it is tested through MSW like the registries.

### Decision 4 — Cache time control: `vi.useFakeTimers({ toFake: ['Date'] })`

`cache.ts` calls `Date.now()` directly and compares `Date.now() > entry.expiresAt` (strictly greater — exactly at `expiresAt` is still a **hit**). Vitest's fake timers do patch `Date`, so TTL boundary tests use `vi.setSystemTime()` to step across `300_000 ms`.

Scoping to `toFake: ['Date']` leaves `setTimeout`/`queueMicrotask`/promise scheduling real, so cache tests don't need to interoperate with MSW's internals. Cache unit tests do no HTTP at all, keeping the two mechanisms fully separate.

### Decision 5 — Test isolation for module-scope singletons

Two modules hold process-wide mutable state:

- `registries/cache.ts` — a module-scope `Map` plus an `insertionCounter`. It already exports **`clearCache()`**; call it in `beforeEach` in every file that touches the cache, directly or transitively (that includes `registries/index.ts`, `/api/check`, and `/api/suggest` tests).
- `suggestions/mcp-bridge.ts` — `mcpClaudeProvider` / `mcpCodexProvider` / `mcpCopilotProvider` are built **at module-load time**, capturing `process.env.MCP_*_URL ?? 'http://localhost:896x'` in a closure. Setting those env vars after import has **no effect**.

  **Strategy:** test the exported **`mcpBridgeProvider(serverUrl, serverName, toolConfig)` factory directly** with a test URL. It is fully parameterized, so this covers all bridge logic without module gymnastics. Only where the *singletons themselves* must be exercised (`suggestions/index.ts` routing, `GET /api/providers`) use `vi.resetModules()` + `await import(...)` after `vi.stubEnv`, or mock the `mcp-bridge` module wholesale.

Vitest's default per-file isolation gives each test file a fresh module registry, so the two mechanisms above only need to handle within-file isolation. Config sets `clearMocks: true`, `restoreMocks: true`, `unstubEnvs: true`, `unstubGlobals: true`.

**Env hygiene:** `tests/setup.ts` deletes `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, and `GITHUB_TOKEN` from `process.env` before any test runs. The repo has a committed `.env`; without this, a developer's local keys would change `getAvailableProviders()` results and make `GET /api/providers` tests pass locally and fail in CI (or vice versa). Tests that need a key set it with `vi.stubEnv`.

### Decision 6 — Integration tests hit route handlers as plain functions

All three routes use `NextRequest`/`NextResponse` from `next/server` and export bare `POST`/`GET` async functions. **No server, no `next build`, no `fetch` to `localhost` is needed.** Import the handler and call it:

```ts
import { POST } from '@/app/api/check/route'

const res = await POST(
  new NextRequest('http://localhost/api/check', {
    method: 'POST',
    body: JSON.stringify({ name: 'express' }),
    headers: { 'content-type': 'application/json' },
  }),
)
expect(res.status).toBe(200)
await expect(res.json()).resolves.toMatchObject({ results: { npm: { status: 'taken' } } })
```

`NextResponse` extends the web `Response`, so `res.status` / `await res.json()` work directly under `environment: 'node'` on Node 22 (undici globals). MSW handlers supply the registry responses underneath, so these are true integration tests through `route → validation → orchestrator → cache → registryFetch`, with only the network faked.

If `next/server` fails to resolve under Vitest's transform pipeline, the fallback is `test.server.deps.inline: ['next']`. Prove this in Step 1's smoke test before building on it.

### Decision 7 — Test file location: top-level `tests/`, not colocated

Tests live in `tests/`, mirroring `src/`. Reasons: files under `src/app/**` are route-scanned by the App Router and stray non-route modules there are a known source of build confusion; and a separate directory lets us keep tests out of the production `tsc` pass.

`tsconfig.json` currently includes `**/*.ts`, which would pull `tests/**` into `next build`'s type check. Add `"tests"` to `tsconfig.json`'s `exclude`, and add `tsconfig.test.json` extending it with `include: ["src/**/*.ts", "tests/**/*.ts"]` plus a `typecheck:test` script. Production build stays untouched (AC #9); tests still get full type checking on demand and in the editor.

```
tests/
├── setup.ts                       # env scrubbing, global hooks
├── msw/
│   ├── server.ts                  # setupServer + listen/reset/close
│   └── handlers.ts                # default happy-path handlers
├── fixtures/                      # trimmed real API responses
│   ├── npm-express.json           ├── github-search.json
│   ├── npm-downloads.json         ├── github-batch.json
│   ├── brew-formula-git.json      ├── anthropic-tool-use.json
│   ├── brew-cask-vscode.json      ├── openai-json-mode.json
│   ├── pypi-requests.json         └── mcp-sse-response.txt
├── unit/lib/
│   ├── validation.test.ts
│   ├── registry-client.test.ts
│   ├── registries/{cache,npm,homebrew,pypi,github,index}.test.ts
│   └── suggestions/{claude,openai,mcp-bridge,index}.test.ts
└── integration/api/{check,suggest,providers}.test.ts
```

Fixtures are **trimmed real responses** (a few representative fields, not full multi-hundred-KB registry payloads), captured once by hand and committed. This keeps fixtures reviewable and pins the exact fields the parsers read (`dist-tags.latest`, `desc` vs `description`, `info.summary`, `items[].name`).

### Decision 8 — Coverage thresholds

Provider: `v8`. Scoped with `coverage.include` so untested React components don't drag the global number down:

```ts
coverage: {
  provider: 'v8',
  include: ['src/lib/**', 'src/app/api/**'],
  exclude: ['src/lib/types.ts', 'src/lib/utils.ts'],  // type-only; shadcn cn() helper
  reporter: ['text', 'html', 'lcov'],
  thresholds: {
    statements: 85, branches: 80, functions: 90, lines: 85,
    // Per-file thresholds are keyed by GLOB, not by relative path.
    '**/src/lib/validation.ts': { statements: 100, branches: 100, functions: 100, lines: 100 },
    autoUpdate: false,
  },
}
```

`validation.ts` is held to 100% because it is a pure function with a small, fully enumerable branch set and it is the security boundary against path traversal (v1 plan risk table, `Path traversal in name input`, impact HIGH).

**The numbers above are provisional targets, not the committed gate.** They are an estimate made before a single test exists, and committing them as-is would either gate on nothing or cause CI churn. Step 14 resolves this in one pass:

1. **Calibrate once (Step 14 only).** Run `vitest run --coverage` against the completed suite and read the achieved numbers. For each metric, set the committed threshold to `floor(achieved)` rounded **down** to the nearest 5, capped at the provisional target above. Example: 87.4% statements → commit `85`; 92.1% → still commit `85`, since the provisional target caps it.
2. **If an achieved number falls more than 5 points below its provisional target**, that is a signal of a real coverage gap, not a reason to lower the bar — identify the uncovered branches and either add the missing test or record an explicit, justified exclusion in Open Questions. Do **not** write filler tests purely to move a number.
3. **`validation.ts` is exempt from downward calibration.** Its 100% requirement is a hard floor (AC #3), not a target.
4. **After Step 14, thresholds only ratchet up.** Lowering a committed threshold requires the same justification as removing a test. This is what makes AC #7 meaningful: the gate's *values* are settled once, then enforced.

Record the final committed values in Step 14's completion note so the calibration is auditable.

---

## Implementation Steps

> Each step is one child ticket (~45–90 min). `Depends on` lines map directly to `bd dep add` edges.

### Step 1 — Install and configure Vitest; prove the harness end to end
**Depends on:** — (root of the graph)
**Est:** 60–90 min

Add dev deps `vitest`, `@vitest/coverage-v8`, `vite-tsconfig-paths`. Create `vitest.config.mts`:

```ts
import { defineConfig } from 'vitest/config'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [tsconfigPaths()],
  test: {
    environment: 'node',
    globals: false,
    setupFiles: ['./tests/setup.ts'],
    include: ['tests/**/*.test.ts'],
    clearMocks: true,
    restoreMocks: true,
    unstubEnvs: true,
    unstubGlobals: true,
  },
})
```

Create `tests/setup.ts` that deletes `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GITHUB_TOKEN` from `process.env`. Add scripts `test` (`vitest`), `test:run` (`vitest run`), `typecheck:test` (`tsc -p tsconfig.test.json --noEmit`). Add `"tests"` to `tsconfig.json`'s `exclude`; create `tsconfig.test.json`. Add `coverage/` and `test-results/` to `.gitignore`. Add an ESLint override block for `tests/**` relaxing `@typescript-eslint/no-explicit-any` (unavoidable in mock factories).

Write **two** throwaway-but-kept smoke tests: one importing `validatePackageName` via the `@/` alias (proves alias resolution), one importing `POST` from `@/app/api/check/route` and merely asserting `typeof POST === 'function'` (proves `next/server` resolves under Vitest — the single riskiest unknown in this plan; if it fails, apply `server.deps.inline: ['next']` here, not later).

**Acceptance criteria:**
- `pnpm test:run` exits 0 with 2 passing tests.
- `pnpm lint`, `pnpm build`, and `pnpm typecheck:test` all pass.
- Importing `@/app/api/check/route` inside a test does not throw. Any workaround needed is committed in `vitest.config.mts` with an explanatory comment.
- `git status` shows no untracked build artifacts (coverage/test-results are ignored).

**Notes:** This step de-risks every later step. Do not proceed to Step 2 if the `next/server` import is unresolved.

---

### Step 2 — MSW harness, network firewall, and registry fixtures
**Depends on:** 1
**Est:** 60–90 min

Add `msw`. Create `tests/msw/server.ts` exactly as specified in Decision 2 — `setupServer()`, `server.listen({ onUnhandledRequest: 'error' })`, **and the `request:unhandled` recorder that throws in `afterEach`**. The recorder is the load-bearing part: without it, an unmodelled request is swallowed by the registry modules' `try`/`catch` into a normal-looking `{ status: 'error' }` and the test passes. Wire it from `tests/setup.ts`. Remember the explicit `vitest` imports (`globals: false`).

Create `tests/msw/handlers.ts` with default happy-path handlers for all five registry endpoints (exact URLs in the v1 plan's *Registry Check Strategy*, matching what the source builds with `encodeURIComponent`).

Capture and commit trimmed fixtures: `npm-express.json` (must include `dist-tags.latest` + `description`), `npm-downloads.json`, `brew-formula-git.json` (uses `desc`), `brew-cask-vscode.json`, `pypi-requests.json` (`info.summary`/`info.version`/`info.author`), `github-search.json` (an `items` array with an exact-name match plus decoys), `github-batch.json` (items matching several names), `mcp-sse-response.txt` (a real `data: {...}` SSE frame).

**Prove the firewall is armed, at the level that matters.** A guard test at the `registryFetch` level is not sufficient — the failure mode being defended against is a *registry module* absorbing the rejection. Write the guard test through `checkNpm`:

```ts
// The firewall must fail this test even though checkNpm() returns cleanly.
it('fails when a registry module absorbs an unmodelled request', async () => {
  server.use()                                   // no handler for the downloads endpoint
  const result = await checkNpm('some-package')  // resolves to { status: 'error' } — no throw
  expect(result.status).toBe('error')            // this assertion PASSES...
})                                               // ...and the afterEach recorder still fails the test
```

Because this test is expected to fail, it cannot live in the normal suite. Verify it manually once during this step, record the observed failure message in the ticket, then convert it to a documented `it.skip` with a comment explaining what it demonstrated. This is a one-time proof that the recorder works, not a permanent test.

**Acceptance criteria:**
- A test calling `registryFetch('https://registry.npmjs.org/express')` receives the fixture body, status 200.
- The guard test above was run and **failed** with the recorder's `"Test attempted N unmodelled request(s)"` message, and that message is pasted into the ticket. It is left in the tree as `it.skip` with an explanatory comment.
- The recorder drains its buffer between tests (`splice`), verified by confirming a second, clean test after the guard test passes rather than inheriting the failure.
- Fixtures are valid JSON, each under ~5 KB, and contain every field the corresponding parser reads.
- Running the suite with the machine's network disabled produces identical results.

**Notes:** Handler URL patterns must tolerate the `encodeURIComponent`'d path segment the source produces. Prefer `https://registry.npmjs.org/:name` style params over literal strings.

---

### Step 3 — Unit tests: `src/lib/validation.ts`
**Depends on:** 1
**Est:** 45–60 min

Pure function, no mocks, held to 100% coverage. Cover, in the source's own check order: empty (`""`, whitespace-only, `undefined` cast); `@`-prefix rejection; length boundary at exactly 214 (valid) and 215 (invalid); regex failures (uppercase, leading `-`/`_`/`.`, spaces, unicode, `../etc/passwd` path traversal); and the warnings contract.

**Acceptance criteria:**
- Every documented error string is asserted verbatim, not by substring.
- A 214-char name is `valid: true`; a 215-char name is `valid: false`.
- `validatePackageName('mytool')` returns `warnings === undefined` — **not** `[]`. (The source uses `warnings.length > 0 ? warnings : undefined`; asserting `toEqual([])` would be wrong.)
- `validatePackageName('my-tool')`, `'my_tool'`, and `'my.tool'` each produce exactly one PyPI-normalization warning.
- `'../etc/passwd'`, `'@scope/pkg'`, and `'Foo'` are all rejected.
- File-level coverage for `validation.ts` is 100/100/100/100.

---

### Step 4 — Unit tests: `src/lib/registries/cache.ts`
**Depends on:** 1
**Est:** 60–75 min

No HTTP. Use `vi.useFakeTimers({ toFake: ['Date'] })` + `vi.setSystemTime`, and `clearCache()` in `beforeEach`.

Cover: set/get round-trip; miss returns `undefined`; key namespacing (same name under `npm` and `pypi` do not collide); PyPI key normalization (`foo_bar`, `foo.bar`, `foo-bar`, `Foo__Bar` all resolve to one entry) while `npm` keys stay case- and separator-sensitive; TTL boundary — a read at exactly `expiresAt` is a **hit**, at `expiresAt + 1` a **miss**, and the expired read *deletes* the entry; eviction at `MAX_ENTRIES = 1000` (insert 1000, insert one more, assert the oldest is gone and the newest present); and that **overwriting an existing key does not trigger eviction** (the source guards on `!cache.has(key)`); `clearCache()` resets both the map and the insertion counter.

**Acceptance criteria:**
- TTL tests assert the `>` (not `>=`) boundary explicitly, with a comment naming the constant `300_000`.
- The eviction test proves FIFO-by-insertion, not true LRU: read entry #1 to "use" it, then overflow, then assert entry #1 was still evicted. This pins the actual implemented semantics.
- `clearCache()` in `beforeEach`; two consecutive tests inserting the same key see no cross-contamination.
- Test file completes in under 1 second despite the 1001-entry loop.

**Notes:** The plan doc calls this "LRU"; the implementation is FIFO-by-insertion. Test the code, and record the naming mismatch (see Open Questions).

---

### Step 5 — Unit tests: `src/lib/registry-client.ts`
**Depends on:** 2
**Est:** 45–60 min

Cover: an explicit `{ timeout: 50 }` against an MSW handler that `await delay(200)` produces a **rejected promise** (abort), and the rejection propagates rather than being swallowed; `headers` are passed through verbatim and absent when not supplied; a 404 and a 500 both **resolve** with the corresponding `Response` (this function does not interpret status — the callers do).

**Testing the `4000 ms` default.** An `AbortSignal` exposes no readable timeout value, so the default *cannot* be asserted by inspecting the signal handed to `fetch`. Spy on the factory instead:

```ts
const timeoutSpy = vi.spyOn(AbortSignal, 'timeout')   // spyOn keeps the real implementation
await registryFetch('https://registry.npmjs.org/express')
expect(timeoutSpy).toHaveBeenCalledWith(4000)

timeoutSpy.mockClear()
await registryFetch('https://registry.npmjs.org/express', { timeout: 50 })
expect(timeoutSpy).toHaveBeenCalledWith(50)
```

This asserts the constant in ~0 ms and needs no sleep. `restoreMocks: true` un-patches the global after each test.

**Acceptance criteria:**
- The abort test asserts a rejection and completes in under 500 ms.
- The default-timeout test asserts `AbortSignal.timeout` was called with `4000`, via a spy that preserves the real implementation. No test sleeps for 4 seconds, and no test attempts to read a timeout value off a signal object.
- A companion test asserts an explicit `{ timeout: 50 }` reaches `AbortSignal.timeout` as `50`, proving the `?? 4000` default is a default and not a hard-code.
- A test proves `registryFetch` does **not** convert a 404 into a thrown error.
- After this file runs, `AbortSignal.timeout` is the native implementation again (assert once, or rely on `restoreMocks` and verify no cross-file bleed by running the full suite shuffled).

---

### Step 6 — Unit tests: `npm.ts`, `homebrew.ts`, `pypi.ts`
**Depends on:** 2, 5
**Est:** 75–90 min

**npm** — 404 → `available`; 200 → `taken` with `extra.latestVersion`, `extra.weeklyDownloads`, and `url: https://www.npmjs.com/package/{name}`; 500 → `{ status: 'error', error: 'HTTP 500' }`; the downloads endpoint failing/timing out independently must **not** fail the check (`weeklyDownloads: 'N/A'`); malformed downloads JSON also yields `'N/A'`; **a downloads value of `0` is preserved as `0`, not replaced by `'N/A'`** (the source checks `!= null`, not truthiness — this is the highest-value edge case in the file); a thrown/aborted package fetch → `status: 'error'`; missing `dist-tags`/`description` degrade gracefully.

**Homebrew** — formula and cask each hit their distinct URL; 404/200/non-ok on both; `desc` (cask) and `description` (formula) precedence; a missing `homepage` leaves `url` `undefined`.

**PyPI** — 404/200/non-ok; success parses `info.summary`/`info.version`/`info.author`; a payload with **no `info` key at all** falls back to `{}` without throwing; thrown error → `status: 'error'`.

**Acceptance criteria:**
- 14+ tests across the three modules; every `RegistryResult.status` value except `rate_limited` observed at least once per module.
- The npm `0`-downloads test exists and asserts `extra.weeklyDownloads === 0`.
- Every assertion on an error result checks the `error` string shape (`HTTP {status}`), not just `status === 'error'`.
- Each module's outbound URL is asserted to match the v1 plan's documented endpoint exactly.

---

### Step 7 — Unit tests: `src/lib/registries/github.ts`
**Depends on:** 2, 5
**Est:** 75–90 min

`checkGitHub(name)` — **403 → `rate_limited`** and **429 → `rate_limited`** (assert both; the source checks these *before* `!res.ok`, so a regression reordering the branches would silently downgrade them to `error`); other non-ok (500, 401) → `error`; exact name match is **case-insensitive** (`items[].name === 'MyTool'` matches query `mytool`) → `taken`; no exact match despite non-empty `items` → `available`; empty `items` array → `available` with `extra` degrading safely; `GITHUB_TOKEN` set → request carries `Authorization: Bearer <token>`; unset → **no** `Authorization` header (assert absence, via `vi.stubEnv` — the source reads the env fresh per call, so no module reset is needed).

`checkGitHubBatch(names)` — **`[]` returns `{}` with zero network requests** (assert via MSW request counter); the query is built as `encodeURIComponent(each) joined by '+OR+'` plus `+in:name&sort=stars&per_page=30`; 403/429 → **every** name maps to `rate_limited`; non-ok → every name maps to `error`; success partitions items per name case-insensitively; a name with no matching item → `available`; a thrown error → every name maps to `error`.

**Acceptance criteria:**
- Separate, individually-named tests for 403 and 429 (not a `.each` that could mask one).
- The empty-input test asserts the request count is 0, not merely that the result is `{}`.
- The batch URL assertion is on the literal query string, so a change in join separator fails the test.
- Both the token-present and token-absent header cases are asserted.

**Notes:** `checkGitHub` sources its description from `items[0]` (top-starred overall), while `checkGitHubBatch` sources from the top-starred *matching* item. Characterize both as-implemented; the inconsistency is logged, not fixed (see Open Questions).

---

### Step 8 — Unit tests: `src/lib/registries/index.ts` (orchestrator)
**Depends on:** 4, 6, 7
**Est:** 75–90 min

`clearCache()` in `beforeEach`. Cover `checkName`: returns all five `REGISTRY_IDS` keys; a single failing registry does not prevent the other four from returning (the `Promise.allSettled` contract — the headline resilience guarantee from the v1 plan); the `warnings` field is populated only when the name contains `[-_.]`, and carries **`index.ts`'s** message, which differs in wording from `validation.ts`'s; `warnings` is `undefined`, not `[]`, otherwise; a second `checkName` for the same name is served from cache and issues **zero** new requests.

`checkNames`: `[]` → `{}` with zero requests; npm/homebrew/pypi are checked per name while GitHub is a **single batched call** — assert the GitHub request count is exactly 1 for N names (v1 verification step #8, the rate-limit mitigation); names already cached for GitHub are excluded from the batch (pre-seed the cache, assert the batch query omits them); if the batch result omits a requested name, that name falls back to `error: 'GitHub batch failed'` (force by mocking `./github` for this one case); duplicate names in the input array.

**Acceptance criteria:**
- A request counter (MSW `server.events.on('request:start')`) proves "1 GitHub call for N names".
- The cache-hit test proves zero repeat network calls, not merely equal return values.
- The `'GitHub batch failed'` fallback string is asserted verbatim.
- The `warnings` test pins `index.ts`'s exact message, and a comment records that it differs from `validation.ts`'s.
- No test in this file depends on execution order; the file passes when run with `--sequence.shuffle`.

---

### Step 9 — Unit tests: `claude.ts` and `openai.ts`
**Depends on:** 1
**Est:** 75–90 min

`vi.mock('@anthropic-ai/sdk')` and `vi.mock('openai')` per Decision 3, with real `AuthenticationError` classes on the mock constructors.

**Claude** — happy path returns the tool-use names array; the request options are asserted (`model: 'claude-sonnet-4-20250514'`, `max_tokens: 1024`, `tool_choice: { type: 'tool', name: 'suggest_names' }`, `input_schema.required` includes `names`); the prompt embeds the input name and `context.takenOn.join(', ')`; `AuthenticationError` → **rejects** with exactly `'Invalid ANTHROPIC_API_KEY. Check your API key configuration.'`; **any other** thrown error → resolves to `[]` (silent-swallow — the behavioral difference that produces 503 vs. empty suggestions downstream); no `tool_use` block in the response → `[]`; `input.names` not an array → `[]`; a mixed array `['a', 5, null, 'b']` → `['a', 'b']`; `takenOn: []` does not throw.

**OpenAI** — happy path; options asserted (`model: 'gpt-4o-mini'`, `response_format: { type: 'json_object' }`); `AuthenticationError` → rejects with `'Invalid OPENAI_API_KEY. …'`; other errors → `[]`; missing `choices[0].message.content` → `[]`; content that is **malformed JSON** → `[]`; parsed object without a `names` array → `[]`; non-string filtering.

**Acceptance criteria:**
- Auth-error tests assert the **exact rethrown message**, guaranteeing the `instanceof` branch was actually taken (a broken mock class would produce `[]` and fail the test).
- A test distinguishes the auth-error path (rejects) from the generic-error path (resolves `[]`) for each provider.
- Model IDs and `max_tokens`/`response_format` are asserted, so a silent model swap fails CI.
- Both files run without any real API key and without network access.

---

### Step 10 — Unit tests: `src/lib/suggestions/mcp-bridge.ts`
**Depends on:** 2
**Est:** 75–90 min

Test the exported **`mcpBridgeProvider(url, name, toolConfig)` factory** with an MSW-backed test URL (per Decision 5) — do not fight the pre-built singletons here.

`isAvailable()` — POST `${url}/mcp` returning 200 **with** an `mcp-session-id` header → `true`; 200 **without** the header → `false`; non-ok → `false`; connection error/abort → `false` (never throws — this is the default state in CI, where no MCP server runs).

`generateSuggestions()` — the two-call sequence (`initialize` then `tools/call`); the tool-call request carries the `Mcp-Session-Id` header and a body of `{ jsonrpc: '2.0', method: 'tools/call', params: { name: toolConfig.toolName, arguments: { [toolConfig.paramName]: prompt } }, id: 2 }` (parameterize the test across the `{toolName:'ask',paramName:'question'}` and `{toolName:'codex',paramName:'prompt'}` shapes); session-init failure → throws `'Failed to initialize MCP session with {serverName}'`; non-ok tool call → throws `'MCP bridge {serverName} returned {status}: {statusText}'`; unparseable body → throws `'Could not parse response from {serverName}'`; empty `result.content[0].text` → throws `'Empty or unexpected response format from {serverName}'`.

Response-parsing matrix (via the returned text): SSE `data: {...}` frame; a **plain JSON** body with no `data:` prefix (the fallback path); an SSE body whose first `data:` line is malformed but a later one parses; a bare JSON array text; a `{ "suggestions": [...] }` object; a **markdown-fenced** array (```` ```json\n["a","b"]\n``` ````) recovered by the `/\[[\s\S]*?\]/` regex; totally unparseable text → `[]`.

**Acceptance criteria:**
- Every thrown-error message is asserted verbatim (these become the 503 bodies in Step 13).
- The two-request sequence is asserted in order, with the session ID from response 1 appearing in request 2's headers.
- At least 6 distinct response-shape parsing tests.
- `isAvailable()` never rejects in any of its four cases.

**Notes:** Do not test the hard-coded `2_000`/`5_000`/`60_000` ms timeouts by waiting them out. Assert the abort path with a short MSW delay only where the code lets you influence timing; otherwise leave the constants uncovered and note it.

---

### Step 11 — Unit tests: `src/lib/suggestions/index.ts` (provider router)
**Depends on:** 9, 10
**Est:** 60–75 min

`getAvailableProviders()` — returns exactly **5** entries in the fixed order `claude, openai, mcp-claude, mcp-codex, mcp-copilot` with the documented display names; `available` for `claude`/`openai` tracks `!!process.env.ANTHROPIC_API_KEY` / `!!process.env.OPENAI_API_KEY` via `vi.stubEnv` — including that an **empty string** key yields `available: false`; one unreachable MCP bridge does not prevent the other two or the API-key providers from being reported correctly (the `Promise.all` never rejects because `isAvailable()` self-catches); with no keys and no MCP servers, all five report `available: false`.

`generateSuggestions(name, providerId, context)` — each of the five IDs routes to the correct underlying provider (`vi.mock` the three provider modules and assert which was called with which args); an unknown `providerId` **throws** `'Unknown provider: {id}'`; an empty-string ID also throws; errors from a provider propagate **un-caught** (this is what becomes the 503 in Step 13).

**Acceptance criteria:**
- Provider list order and IDs are asserted as an exact array, so a reorder fails.
- The empty-string-API-key case is a distinct test.
- All five routing branches plus the `default` throw are covered.
- No test in this file makes a real network call or requires a running MCP server.

---

### Step 12 — Integration tests: `POST /api/check` and `GET /api/providers`
**Depends on:** 8, 11
**Est:** 75–90 min

Import handlers directly per Decision 6. MSW supplies registry responses. `clearCache()` in `beforeEach`.

**`POST /api/check`** — valid name → **200** with the full `CheckResponse` shape (all five `results` keys present); invalid name → **400** with body exactly `{ error: <validation message> }` (single key — assert `Object.keys(body)` to catch shape drift); `{}` body (missing `name`) → 400 `'Name cannot be empty'`; `'../etc/passwd'` → 400 **and zero outbound requests** (v1 verification step #6 — the path-traversal guard must short-circuit *before* any URL is built); a name with `-`/`_`/`.` → 200 with a populated `warnings` array; a registry returning 500 → still **200** overall, with that one registry showing `status: 'error'` (graceful degradation); GitHub 403 → 200 with GitHub `rate_limited`; a **malformed JSON body** → characterize the current behavior (the handler has no `try`/`catch` around `request.json()`) and assert it explicitly rather than assuming.

**`GET /api/providers`** — always 200; returns exactly 5 `ProviderInfo` objects; `available` flags respond to stubbed env keys; unreachable MCP bridges yield `available: false` without error.

**Acceptance criteria:**
- Every status code the handlers can emit has at least one test.
- The path-traversal test asserts a request count of 0.
- Error-body shapes are asserted by exact key set, not `toMatchObject`.
- The malformed-JSON test documents actual observed behavior in a comment and links the follow-up ticket.
- All tests pass with no API keys and no network.

---

### Step 13 — Integration tests: `POST /api/suggest`
**Depends on:** 8, 11, 12
**Est:** 75–90 min

The most branch-dense route. Mock the AI provider layer (`vi.mock('@/lib/suggestions')` or the individual provider modules); let registry re-checks flow through MSW. `clearCache()` in `beforeEach`.

Cover: happy path → **200** with `{ suggestions: [{ name, results }] }`, each suggestion carrying all five registry keys; invalid **input** name → **400** `{ error }`; unknown provider → **503** with body `{ error: 'Unknown provider: bogus', provider: 'bogus' }` — assert **both** keys, since this route's error body shape differs from `/api/check`'s single-key shape; a provider throwing an auth error → 503 carrying that provider's exact message; a provider throwing an MCP bridge error → 503 with that message; the AI returning a mix of valid and invalid names → only the valid ones reach `checkNames` and appear in the response (assert the invalid ones are absent and were never fetched); the AI returning **only** invalid names, or `[]` → **200** with `suggestions: []` (not an error — an easy-to-get-wrong branch); GitHub re-check for N suggestions issues exactly **1** GitHub request (v1 verification step #8); a suggestion missing from the `checkNames` output degrades to `results: {}` rather than throwing; malformed JSON body characterized as in Step 12.

**Acceptance criteria:**
- 503 body asserted as an exact two-key object for at least three distinct failure causes.
- The "zero valid suggestions → 200 with empty array" test exists and is explicitly named.
- A GitHub request counter proves the single batched call.
- The suggestion-filtering test asserts the rejected names were never sent to any registry.
- Suite still runs with no API keys present.

---

### Step 14 — Coverage thresholds and CI wiring
**Depends on:** 3, 8, 11, 12, 13
**Est:** 60–75 min
**Closes bead:** `application-namer-upf`

Add `@vitest/coverage-v8` config per Decision 8. Run `vitest run --coverage`, read the actual numbers, then apply the calibration rule (measure → set; drop a threshold to the measured floor rounded down to the nearest 5 rather than writing filler tests). Add scripts `test:ci` (`vitest run --coverage --reporter=default --reporter=junit --outputFile=./test-results/junit.xml`).

Edit `.github/workflows/ci.yml` to insert `- run: pnpm test:ci` **between** `pnpm lint` and `pnpm build` (fail fast on cheap checks before the expensive build). Add `- run: pnpm typecheck:test`. Optionally upload `coverage/` as an artifact with `if: always()`.

Verify the gate actually bites: temporarily delete a test file, confirm CI fails on the threshold; restore.

**Acceptance criteria:**
- `pnpm test:ci` exits 0 locally and prints a coverage table meeting the final thresholds.
- A deliberate coverage regression makes `pnpm test:ci` exit non-zero (demonstrated, then reverted).
- CI runs lint → test → typecheck:test → build, and a red test fails the workflow.
- Total CI wall-clock increase is under ~60 s.
- Final threshold values are recorded in the ticket's completion note.
- `coverage/` and `test-results/` are gitignored; no artifacts are committed.

---

## Step Dependency Graph

```
1 ──┬─→ 2 ──┬─→ 5 ──┬─→ 6 ──┐
    │       │       └─→ 7 ──┼─→ 8 ──┬─→ 12 ──→ 13 ──┐
    │       └─→ 10 ─┐       │       │               │
    ├─→ 3 ──────────┼───────┼───────┼───────────────┼─→ 14
    ├─→ 4 ──────────┼───────┘       │               │
    └─→ 9 ──────────┴─→ 11 ─────────┴───────────────┘
```

| Step | Depends on |
|---|---|
| 1 | — |
| 2 | 1 |
| 3 | 1 |
| 4 | 1 |
| 5 | 2 |
| 6 | 2, 5 |
| 7 | 2, 5 |
| 8 | 4, 6, 7 |
| 9 | 1 |
| 10 | 2 |
| 11 | 9, 10 |
| 12 | 8, 11 |
| 13 | 8, 11, 12 |
| 14 | 3, 8, 11, 12, 13 |

**Parallelizable once Step 1 lands:** {2, 3, 4, 9}. After Step 2: {5, 10}. After Step 5: {6, 7}.

### Mapping to existing beads

| Bead | Covered by |
|---|---|
| `application-namer-ef1` — unit tests for validation/registry-client/cache | Steps 3, 4, 5 |
| `application-namer-l35` — integration tests for API routes | Steps 12, 13 |
| `application-namer-upf` — CI test step | Step 14 |
| *(new)* harness + remaining lib coverage | Steps 1, 2, 6, 7, 8, 9, 10, 11 |

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation | Step |
|---|---|---|---|---|
| `next/server` fails to resolve or transform under Vitest, blocking all route tests | Medium | High | Proven in Step 1's smoke test **before** any dependent work; fallback `server.deps.inline: ['next']` documented in-config | 1 |
| Tests silently hit the real network, making CI flaky and rate-limited | Medium | High | MSW `onUnhandledRequest: 'error'` plus an explicit firewall-guard test | 2 |
| A developer's local `.env` keys change `getAvailableProviders()` results → passes locally, fails in CI | High | Medium | `tests/setup.ts` deletes `ANTHROPIC_API_KEY`/`OPENAI_API_KEY`/`GITHUB_TOKEN`; tests opt in via `vi.stubEnv` with `unstubEnvs: true` | 1, 11, 12 |
| Cache module singleton leaks state between tests → order-dependent failures | High | Medium | `clearCache()` in `beforeEach` everywhere the cache is reachable; Vitest per-file isolation; Step 8 verified under `--sequence.shuffle` | 4, 8, 12, 13 |
| SDK mock's `AuthenticationError` isn't a real class → `instanceof` falls through and the 503 test passes for the wrong reason | Medium | High | Mock factory defines a real `class extends Error`; auth tests assert the **exact rethrown message**, not just "it threw" | 9 |
| MCP bridge singletons freeze `MCP_*_URL` at import → env stubbing appears to work but doesn't | Medium | Medium | Test the parameterized `mcpBridgeProvider` factory directly; use `vi.resetModules()` + dynamic import only where singletons are unavoidable | 10, 11 |
| Timeout tests wait out real 4 s / 60 s constants → slow suite | Medium | Medium | Pass explicit small `timeout` to `registryFetch` with a ~200 ms MSW delay; never fake-time `AbortSignal.timeout`; leave hard-coded MCP timeouts uncovered and documented | 5, 10 |
| Coverage thresholds set too high → chronic CI churn and filler tests | Medium | Medium | Measure-then-set calibration rule; thresholds scoped to `src/lib/**` + `src/app/api/**`; 100% required only for `validation.ts` | 14 |
| `tests/**` pulled into `next build`'s type check, slowing or breaking the production build | Medium | Medium | `tests` added to `tsconfig.json` `exclude`; separate `tsconfig.test.json` + `typecheck:test` script; AC #9 guards it | 1 |
| Tests are written against the v1 *plan* rather than the *code* where the two disagree (MCP ports, LRU-vs-FIFO, warning wording) | High | Medium | Every divergence is enumerated in Open Questions; tests characterize the implementation and reference the mismatch in a comment | 4, 8, 10 |
| Full-size registry fixtures become unreviewable and hide parser assumptions | Medium | Low | Fixtures trimmed to ~5 KB, retaining only fields the parsers read | 2 |

---

## Verification Steps

1. `pnpm install && pnpm test:run` passes from a clean checkout with **no `.env` file present**.
2. The same command passes with the machine's network interface disabled — proving zero real outbound requests.
3. `pnpm test:run` completes in under 30 seconds cold.
4. `pnpm test:run -- --sequence.shuffle` passes — proving no inter-test order dependence.
5. Running any single test file in isolation (`pnpm vitest run tests/unit/lib/registries/index.test.ts`) passes — proving no cross-file state coupling.
6. `pnpm test:ci` prints a coverage table; every threshold in Decision 8 is met or exceeded.
7. Deleting `tests/unit/lib/validation.test.ts` makes `pnpm test:ci` exit non-zero on the `validation.ts` 100% threshold; restoring it goes green.
8. Introducing a deliberate regression — change `validation.ts`'s max length from 214 to 100 — fails the boundary test; revert.
9. Introducing a deliberate regression — reorder `github.ts` so `!res.ok` is checked before the 403/429 branch — fails the `rate_limited` tests; revert.
10. Introducing a deliberate regression — make `checkNames` call `checkGitHub` per name instead of `checkGitHubBatch` — fails the "exactly 1 GitHub request" assertion; revert.
11. `pnpm lint` and `pnpm typecheck:test` both pass over `tests/**`.
12. `pnpm build` succeeds and its wall-clock time is unchanged from before this epic (± noise) — confirming tests are excluded from the production type check.
13. A pushed branch shows CI running lint → test → typecheck → build, with the test step visible and green.
14. A pushed branch containing one failing test turns the CI workflow red at the test step, before `pnpm build` runs.

---

## Open Questions

Logged to `.omc/plans/open-questions.md`. Summary:

1. **Malformed JSON request bodies.** Neither `/api/check` nor `/api/suggest` wraps `await request.json()` in a `try`/`catch`, so a non-JSON body throws inside the handler rather than returning a clean 400. This plan **characterizes** current behavior. Should a fix be filed as a separate bug ticket?
2. **MCP default port drift.** `.omc/plans/application-namer-v1.md` AC #9 documents defaults `8940/8941/8945`; the code and `.env.example` both use `8960/8961/8962`. Which is canonical? Tests will pin the code's values.
3. **"LRU" naming.** The v1 plan describes LRU eviction; `cache.ts` implements FIFO-by-insertion (reads do not refresh recency). Tests pin the implemented FIFO behavior — is that the intended semantics, or a bug?
4. **`checkGitHub` description sourcing.** It takes `description`/`owner`/`stars` from `items[0]` (top-starred *overall*), which may be a different repo than the exact-name match. `checkGitHubBatch` correctly uses the top-starred *matching* item. Characterize as-is, or file a bug?
5. **`SuggestResponse.errors`.** Declared in `types.ts` but never populated by `/api/suggest`. Should tests assert its absence, or should the field be removed?
6. **Duplicate PyPI-normalization warning text.** `validation.ts` and `registries/index.ts` emit *different* wording for the same concept; `/api/check` surfaces `index.ts`'s. Consolidate, or pin both?
7. **Coverage thresholds.** Are 85/80/90/85 (statements/branches/functions/lines) over `src/lib/**` + `src/app/api/**`, with 100% on `validation.ts`, the right initial bar?
8. **Component/hook test scope.** `src/components/**` and `use-name-check.ts` are deferred to a follow-up epic. Confirm they are out of scope here.

---

## Follow-ups (out of scope for this epic)

- Component tests for `search-form`, `results-grid`, `result-card`, `suggestions-panel`, `provider-selector` — add as a second Vitest project with `environment: 'jsdom'` + `@vitejs/plugin-react` + Testing Library, rather than changing the global environment.
- Hook tests for `use-name-check.ts` (debounce, `AbortController` cancellation, stale-response race — v1 verification step #13).
- Playwright E2E for the v1 verification steps that need a real browser (responsive layout at 375 px, error-boundary isolation).
- Bug tickets arising from Open Questions 1, 3, and 4.
