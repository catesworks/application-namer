# QA Epic — Manual QA & Release Verification (application-namer v1)

**Beads epic:** `application-namer-avp`
**Child issues:** `application-namer-zok` (env config), `application-namer-c9j` (manual QA walkthrough)
**Source of truth for criteria:** `.omc/plans/application-namer-v1.md` — "Acceptance Criteria", "API Error Response Schema", "Verification Steps" (1–14)
**Status of subject app:** feature-complete v1, never verified end-to-end, zero test tooling in repo.
**Revision:** v2 — post-consensus (architect review applied; see Changelog).

---

## Requirements Summary

Prove that application-namer v1 satisfies its own acceptance criteria before it is called done, and leave behind a repeatable verification mechanism rather than a one-time human walkthrough.

Three things must be produced:

1. **A local secret configuration** that unlocks the never-exercised code paths: Claude/OpenAI suggestion providers, the MCP bridge providers, and the authenticated (30 req/min) GitHub path.
2. **A verification harness** covering the deterministic subset of the plan's Verification Steps, runnable on demand and in CI.
3. **A human checklist** covering the irreducibly manual subset (cost-incurring AI calls, external MCP processes, subjective visual judgement), with recorded evidence.

The epic also closes the gap between the v1 plan's **API Error Response Schema** section (a documented contract) and what the routes actually return — that contract was never verified and is known to be partially unimplemented.

**Explicitly out of scope:** fixing any defect found. This epic *finds, pins, and records* defects; fixes are filed as new beads issues and executed separately. Also out of scope: unit/integration tests for `lib/` (owned by epic `application-namer-9w9`) and deployment (`application-namer-aqq`).

**Expected-fail is a first-class outcome.** Several checks in this plan are written against the *documented* contract, which current code is already known to violate. Those specs use Playwright's `test.fail()` so the suite stays green while pinning the defect — and flips to a hard failure the moment someone fixes the bug without updating the spec. A check marked **XFAIL** below is expected to fail today; that is the deliverable, not a problem to work around.

---

## Acceptance Criteria (for this epic)

1. `.env.local` exists locally with at least one AI provider key and a `GITHUB_TOKEN`; no secret value is committed, and `git status` is clean of env files after configuration.
2. Three reproducible run profiles are documented and shown to work: **full-keys**, **no-keys**, **no-github-token**.
3. Every one of the v1 plan's Verification Steps 1–14 has a recorded outcome (pass / xfail / fail / blocked) with evidence (assertion, screenshot, log excerpt, or HTTP transcript).
4. Every gap check added by this plan (**V15–V27** in the Coverage Matrix) also has a recorded outcome.
5. The automated portion runs green via a single command (`pnpm test:e2e`) against a locally-booted app, with **zero** third-party network traffic — provable by running it with the machine offline.
6. The manual portion is written down as a checklist a second person could execute without reading this plan.
7. Every failure found is filed as a beads issue linked to epic `application-namer-avp`, with reproduction steps; the epic is not closed with silent failures.
8. No API key, token, or credential appears in any committed file, Playwright trace, screenshot, HTML report, or beads issue body.

---

## Current State (verified against source, 2026-09-05)

| Fact | Detail |
|------|--------|
| Framework | Next.js **16.2.6**, React 19.2.4, App Router, `src/` dir, Tailwind v4, pnpm 10.33.0 |
| Node | `.nvmrc` pins **22**; the local machine is running **v24.1.0**. CI uses `node-version-file: .nvmrc` → 22. **The harness must run on the `.nvmrc` version**, because fetch interception is undici-version-sensitive |
| Scripts | `dev`, `build`, `start`, `lint` — **no test script** |
| Test tooling | **None.** No Playwright, Vitest, Jest, or config for any of them. `undici` is **not** in `node_modules` |
| CI | `.github/workflows/ci.yml` runs `pnpm lint` + `pnpm build` only |
| `next.config.ts` | Empty (`{}`) — no `instrumentationHook` concerns; `src/instrumentation.ts` does not exist yet |
| Env files | `.env.example` **is tracked** (force-added past the `.env*` ignore rule). An **untracked `.env` also exists**, holding only the three `MCP_*_URL` defaults; all key vars present but empty |
| `.env.local` | Does **not** exist |
| MCP port drift | v1 plan AC #9 says `8940/8941/8945`; `.env.example` and `src/lib/suggestions/mcp-bridge.ts` both use `8960/8961/8962`. Code and template agree; **the plan is the stale artifact** |
| Providers route | `GET /api/providers` returns a **bare array** of `ProviderInfo`, not a wrapped object |
| Error schema reality | `400 {error}` ✅ implemented (both routes) · `503 {error, provider}` ✅ implemented (suggest) · `429 {error, retryAfter}` ❌ **not implemented** — GitHub rate limiting surfaces only as a per-registry `status: "rate_limited"` · `500 {error, registry}` ❌ **not implemented** |
| Unguarded `request.json()` | Both `check` and `suggest` call `await request.json()` with no try/catch — a malformed body throws and yields a framework 500 that does not match the documented schema |
| `SuggestResponse.errors` | Typed in `src/lib/types.ts`, **never populated** by `src/app/api/suggest/route.ts` |
| Error-result caching | `fetchWithCache` in `src/lib/registries/index.ts` caches `status: "error"` and `status: "rate_limited"` results for the full 5-min TTL — a transient failure is sticky |
| Batch failure path | `checkGitHubBatch` wraps its whole body in `try/catch` and returns a per-name `error` map; it **does not reject**, so `Promise.all` in `checkNames` is not at risk. *(Corrected from v1 of this plan, which claimed the opposite.)* |
| **`extra`-key contract drift** | `result-card.tsx` renders only `extra.version`, `extra.downloads`, `extra.homepage`. But `npm.ts` writes `latestVersion` / `weeklyDownloads`; `pypi.ts` writes `latestVersion` / `author`; `github.ts` writes `owner` / `stars` / `description`. **Only Homebrew's `homepage` matches.** npm version+downloads, PyPI version+author, and GitHub owner+stars are computed and then silently dropped by the UI |
| GitHub batch operator limit | `checkGitHubBatch` joins N names with `+OR+`. GitHub's search API documents a maximum of **five** boolean operators per query; 8 suggestions produce **seven** `OR`s → expected `422`, which the code maps to an `error` for every suggestion, and those errors are then cached for 5 minutes |
| GitHub batch recall | Batch query uses `per_page=30` sorted by stars; a low-star exact match can fall outside the first 30 results → reported **available when actually taken** (violates the v1 plan's first principle) |
| `checkGitHub` (single) | `topRepoExtra(items)` reads `items[0]` — the top-starred result overall, **not** the exact-name match that produced the `taken` verdict — so owner/stars/description can describe a different repository |
| Cache eviction | `cache.ts` tracks `insertedAt` and never refreshes it on read → **FIFO**, not the LRU the v1 plan specifies |
| `clearCache` | Exported from `cache.ts`, **zero call sites** — dead code today, and exactly the seam the harness needs |
| Warning-string duplication | `validation.ts` and `registries/index.ts` emit two different wordings of the same PyPI normalization warning; `/api/check` surfaces the `index.ts` one |
| Homebrew Cask extra | v1 plan promises "App name, homepage URL"; `homebrew.ts` captures only `homepage` — the cask's `name` array is never read |
| Statuses | `RegistryStatus` has **four** members (`available`/`taken`/`error`/`rate_limited`); v1 AC #3 only names three |
| Abort handling | `useNameCheck` already wires `AbortController` on both check and suggest paths |
| Available tooling | Playwright MCP browser tools are available to agents in this environment; a 1Password MCP server with Environments + local `.env` file generation is also available |

---

## Decision 1 — Browser automation vs. pure manual checklist

### Recommendation: **Hybrid, automation-first — add `@playwright/test` to the repo, automate the deterministic majority, keep a short human checklist for the rest.**

**Option A — Commit a Playwright suite (chosen).** Converts most of the verification surface into a permanent regression guard. Three of the fourteen original steps (exactly-one-GitHub-call, network timeout, sub-300 ms race cancellation) are unreliable or impossible to judge by hand but trivial with request interception. The repo already carries committed intent to gain test tooling (`application-namer-9w9`, `application-namer-upf`), so config and CI overhead is amortized. Cost: one devDependency plus a browser download, a stub fixture to maintain, and roughly half a day of authoring.

**Option B — Pure manual checklist.** Zero footprint and fastest to a first answer, but not repeatable: the next regression goes undetected, and V8/V10/V13 plus the 5 s p95 criterion cannot be judged reliably by a human. *Invalidation rationale:* the epic exists to produce release verification that will be needed again after every change; a manual-only pass costs nearly as much per run and answers the question only once.

**Option C — Agent-driven Playwright **MCP** only.** No dependency, immediate start, but nothing committed, nothing CI-runnable, and non-deterministic run to run. *Invalidation rationale:* rejected as the **deliverable**, adopted as a **tool** — an agent should use Playwright MCP while authoring specs to discover real selectors and confirm DOM structure, then encode findings into committed specs.

### What stays manual, and why

| Step | Why it is not automated |
|------|-------------------------|
| V7 (real AI suggestions) | Costs money per run and is non-deterministic. Automation stubs the provider and asserts response *shape*; real output is confirmed once, by hand. |
| V9 (zero API keys) | Requires a second server process under a different env profile; the value is a one-time confirmation, not a regression guard. |
| V12 (MCP bridges up/down) | Depends on external local processes the runner does not own. |
| Visual polish / AC #3 legibility | Colour, contrast, and "would a user understand this" are human judgements. |

---

## Decision 2 — Environment variables and where secrets live

### Where secrets live

**`.env.local`, and nowhere else.**

- `.gitignore` already carries a blanket `.env*` rule, so `.env.local` cannot be committed by accident (`.env.example` is tracked only because it was force-added).
- Next.js load precedence is **real process env > `.env.local` > `.env.<NODE_ENV>` > `.env`**. Putting secrets in `.env.local` means the existing `.env` cannot shadow them, and a shell-level override can still force the no-keys profile without editing a file.
- The stray `.env` is a hazard: it re-declares every key as an empty string and duplicates MCP defaults already hardcoded in `mcp-bridge.ts`.

**Do not delete `.env`.** It is the user's untracked local file and may exist for a reason not visible in the repo. **Rename it with consent** — `mv .env .env.bak` — after asking, and add `.env.bak` to the secret-scan exclusion list rather than to git.

**Secret origin (recommended):** store the three credentials in a **1Password Environment** and materialize `.env.local` from it (a 1Password MCP server with `create_environment` / `create_local_env_file` is available). Fallback: hand-edit `.env.local` and `chmod 600` it.

**Hard rule:** real keys are pasted into `.env.local` only. Never into `.env`, `.env.bak`, `.env.example`, a Playwright config, a spec file, a CI secret for this epic, or any evidence artifact. **The Playwright harness never reads `.env.local`** — it pins its own dummy values (see Step 6).

### Exact setup sequence

```bash
# 0. Use the pinned Node version — interception behaviour is undici-version-sensitive
nvm use            # reads .nvmrc -> 22

# 1. Neutralize the ambiguous stray env file (ASK FIRST — it is the user's file)
mv .env .env.bak

# 2. Create the real local env from the tracked template
cp .env.example .env.local
chmod 600 .env.local

# 3. Fill in — via 1Password Environments, or by hand:
#      ANTHROPIC_API_KEY=...      (at least one provider required)
#      OPENAI_API_KEY=...         (optional; enables provider-switch checks)
#      GITHUB_TOKEN=...           (no scopes needed; lifts 10 -> 30 req/min)
#    Leave MCP_*_URL at the .env.example defaults (8960/8961/8962).

# 4. Confirm nothing leaked (widened pattern — classic + fine-grained + service-account tokens)
git status --short          # must show no .env* entry
grep -rnE 'sk-ant-|sk-proj-|sk-svcacct-|ghp_|github_pat_|gh[opsu]_' \
  --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=.git \
  --exclude=.env.local --exclude=.env.bak .
```

### Run profiles

| Profile | Command | Unlocks |
|---------|---------|---------|
| **full-keys** (default) | `pnpm dev` | V7, V12, authenticated GitHub, provider switching |
| **no-keys** | `ANTHROPIC_API_KEY= OPENAI_API_KEY= GITHUB_TOKEN= MCP_CLAUDE_URL=http://127.0.0.1:9 MCP_CODEX_URL=http://127.0.0.1:9 MCP_COPILOT_URL=http://127.0.0.1:9 pnpm dev` | V9 (AC #8). Real process env outranks `.env.local`; empty string is falsy in `!!process.env.X`, so providers report unavailable, and port 9 (discard) makes the MCP health check fail fast without editing files |
| **no-github-token** | `GITHUB_TOKEN= pnpm dev` | V19 — the unauthenticated 10 req/min rate-limit path |

---

## Decision 3 — How server-side fetches are intercepted (resolved, not spiked)

Every registry call happens inside a Next.js route handler, so Playwright's browser-level `page.route()` cannot see it. The interception must happen **inside the Next.js server process**.

### Chosen mechanism: `src/instrumentation.ts` + undici `MockAgent`

Next.js calls `register()` in `src/instrumentation.ts` once per server process, before any request is served. Under a Node runtime this is the correct hook for installing a global fetch dispatcher.

```ts
// src/instrumentation.ts
export async function register() {
  if (process.env.NEXT_RUNTIME !== "nodejs") return;
  if (process.env.E2E_MOCK_REGISTRIES !== "1") return;   // inert unless explicitly enabled
  await import("../e2e/mocks/install-mock-agent");        // dev-only module
}
```

The mock module builds an undici `MockAgent`, registers interceptors for `registry.npmjs.org`, `api.npmjs.org`, `formulae.brew.sh`, `pypi.org`, and `api.github.com`, calls `mockAgent.disableNetConnect()`, and installs it with `setGlobalDispatcher`.

**Why this works, and the one thing that must be verified first:** Node's built-in `fetch` reads its dispatcher from `globalThis[Symbol.for('undici.globalDispatcher.1')]`, and the npm `undici` package's `setGlobalDispatcher` writes that same symbol — so an installed `undici` devDependency can steer Node's built-in fetch. This is **version-sensitive**: it holds only while the installed `undici` major agrees with the one bundled in the running Node. Step 4 therefore begins with a five-line assertion that interception actually took effect, and the harness runs on the `.nvmrc` Node version, not the machine default (v24.1.0).

**Named fallback:** if the symbol handshake does not hold on Node 22, switch to `msw/node` `setupServer()`, which performs the same job through a maintained interception layer that also covers `http`/`https`. The interceptor module is the only file that changes; every spec is written against the fixture API, not against undici.

**Safety:** the mock module lives under `e2e/`, is imported dynamically, and is reached only when `E2E_MOCK_REGISTRIES=1`. It is never bundled into a production build path and never enabled by any committed script other than the stubbed Playwright config.

### Cache neutralization

`cache.ts` holds a module-level `Map` that lives for the whole server process, so a second test would otherwise read the first test's results — and the sticky-error behaviour under test (V21) would leak across specs. Two complementary measures:

1. **Unique names by default.** Every spec that does not specifically test caching searches a per-test unique name (e.g. `qa-fixture-${testInfo.testId}`), so cache keys never collide. The MockAgent matches on a path pattern, not a literal name.
2. **An explicit reset seam.** `POST /api/__e2e__/reset` calls the already-exported-but-dead `clearCache()` and returns `204`. The route returns `404` unless `process.env.E2E_TEST_HOOKS === "1"`, which only the stubbed Playwright config sets. Specs that must re-search the *same* name (V13 race, V21 sticky cache) call it in `beforeEach`.

This also converts `clearCache` from dead code into a used export — but note that the underlying dead-code finding is still filed as a bug bead, because a production export with no call sites is a code-health defect independent of this harness.

---

## Test Architecture

```
application-namer/
├── src/
│   ├── instrumentation.ts             # NEW — env-gated hook; inert unless E2E_MOCK_REGISTRIES=1
│   └── app/api/__e2e__/reset/route.ts # NEW — env-gated cache reset; 404 unless E2E_TEST_HOOKS=1
├── playwright.config.ts               # STUBBED run: projects `stubbed` + `mobile`
├── playwright.live.config.ts          # LIVE run: project `live`, real network, no mocks
├── e2e/
│   ├── mocks/
│   │   ├── install-mock-agent.ts      # MockAgent + disableNetConnect + setGlobalDispatcher
│   │   └── registry-payloads.ts       # canned npm / brew / pypi / github JSON bodies
│   ├── fixtures/test-base.ts          # `test` extended with resetServerCache() + requestCounter()
│   ├── search.spec.ts                 # V2–V5 (stubbed), V6, V14, V16, V23, V26
│   ├── resilience.spec.ts             # V10, V13, V19, V21
│   ├── responsive.spec.ts             # V11, V17
│   ├── api-contract.spec.ts           # V15, V18, V20, V24
│   └── live-smoke.spec.ts             # @live — V2–V5 live, V8, V22, V27
└── docs/
    └── qa-manual-checklist.md         # V1, V7, V9, V12 + visual pass, with evidence slots
```

### Two configs, because `webServer` is global

Playwright's `webServer` is a top-level option, not per-project, so a single config cannot boot one server with mocks installed and another without. The stubbed and live runs therefore live in **separate config files**:

| | `playwright.config.ts` (stubbed) | `playwright.live.config.ts` |
|---|---|---|
| `webServer.command` | `pnpm dev` locally · `pnpm build && pnpm start` in CI | `pnpm dev` |
| `webServer.port` | 3100 | 3101 |
| `webServer.reuseExistingServer` | `!process.env.CI` | `false` |
| `webServer.env` | fully pinned (below) | inherits the developer's `.env.local` |
| Projects | `stubbed` (1280×720), `mobile` (375×812) | `live` (`grep: /@live/`) |
| Network | `disableNetConnect()` — any unmocked call throws | real |
| Runs in CI | yes | **no** |

### Pinned `webServer.env` for the stubbed config

The stubbed server must be byte-for-byte deterministic and must never touch a real credential or a real localhost service:

```ts
webServer: {
  command: process.env.CI ? "pnpm build && pnpm start" : "pnpm dev",
  port: 3100,
  reuseExistingServer: !process.env.CI,
  env: {
    E2E_MOCK_REGISTRIES: "1",
    E2E_TEST_HOOKS: "1",
    // dummy provider credential: makes `claude` report available so the suggest
    // path is reachable, while the provider call itself is intercepted
    ANTHROPIC_API_KEY: "sk-ant-e2e-dummy-not-a-real-key",
    OPENAI_API_KEY: "",
    GITHUB_TOKEN: "",
    // point every MCP health check at the discard port so it fails fast and
    // never contacts a bridge that happens to be running on the dev machine
    MCP_CLAUDE_URL: "http://127.0.0.1:9",
    MCP_CODEX_URL: "http://127.0.0.1:9",
    MCP_COPILOT_URL: "http://127.0.0.1:9",
  },
}
```

The dummy Anthropic key is a literal, not a secret, and is the one string the secret grep must be taught to ignore (it is matched by the `sk-ant-` pattern by design — see Step 4's AC).

---

## Coverage Matrix

`Auto` = executed by the Playwright suite. `Human` = blocked-on-user, recorded in `docs/qa-manual-checklist.md`. **XFAIL** = written with `test.fail()` against the documented contract; expected to fail today.

| ID | Check | Source | Mode | Step |
|----|-------|--------|------|------|
| V1 | `pnpm dev` starts clean on a fresh install | plan VS1 | **Human** | 12 |
| V2 | "express" → taken on npm, PyPI, GitHub | plan VS2 | Auto (status) + **XFAIL** (npm version/downloads not rendered) | 7 |
| V3 | "xyzzy-nonexistent-pkg-12345" → available everywhere | plan VS3 | Auto | 7 |
| V4 | "git" → taken on Homebrew formulae + GitHub | plan VS4 | Auto (status + brew extra) + **XFAIL** (GitHub owner/stars not rendered) | 7 |
| V5 | "visual-studio-code" → taken on Homebrew Cask | plan VS5 | Auto (status + homepage) + **XFAIL** (cask app name never captured) | 7 |
| V6 | `"../etc/passwd"` → 400, zero outbound registry calls | plan VS6 | Auto | 7 |
| V7 | Real AI suggestions are valid names with per-registry results | plan VS7 | **Human** | 12 |
| V8 | Suggestion re-check issues exactly **one** GitHub call — and survives it | plan VS8 | **Auto @live** + **XFAIL** (7 `OR`s exceed the 5-operator limit → 422) | 11 |
| V9 | Zero-key profile: registries work, provider list empty, suggest disabled | plan VS9 | **Human** | 12 |
| V10 | Network failure → per-registry error states, no crash | plan VS10 | Auto | 8 |
| V11 | 375 px: cards stack, no horizontal scroll | plan VS11 | Auto (`mobile`) | 9 |
| V12 | MCP providers available when bridges up, greyed when down | plan VS12 | **Human** | 12 |
| V13 | Rapid re-search aborts the first request; no stale results | plan VS13 | Auto | 8 |
| V14 | "my-tool" shows the PyPI normalization warning | plan VS14 | Auto | 7 |
| V15 | **Gap:** `400` body matches `{ error: string }` on both routes | Error Schema | Auto | 10 |
| V16 | **Gap:** all **four** statuses render a distinct indicator, incl. `rate_limited` | AC #3 vs types | Auto | 7 |
| V17 | **Gap:** 1024 px desktop layout is multi-column | AC #10 | Auto | 9 |
| V18 | **Gap:** malformed JSON body → schema-conforming error | Error Schema | Auto **XFAIL** | 10 |
| V19 | **Gap:** GitHub 403/429 → `rate_limited` surfaced; `429 {error, retryAfter}` contract | Error Schema | Auto (status) + **XFAIL** (contract) | 8 |
| V20 | **Gap:** `500 {error, registry}` contract | Error Schema | Auto **XFAIL** | 10 |
| V21 | **Gap:** an `error`/`rate_limited` result is cached for 5 min and survives recovery | code review | Auto **XFAIL** | 8 |
| V22 | **Gap:** results render within 5 s p95 with 4 s per-registry timeouts | AC #2 | **Auto @live** (meaningless against instant mocks) | 11 |
| V23 | **Gap:** `Enter` submits natively; `@scope/name` and a 215-char name rejected | AC #1, validation | Auto | 7 |
| V24 | **Gap:** `/api/providers` returns a bare `ProviderInfo[]`; `/api/suggest` returns ≤8 suggestions and never populates `errors` | AC #6, types | Auto | 10 |
| V25 | **Gap:** `pnpm lint` and `pnpm build` clean on the `.nvmrc` Node version | CI parity | Auto | 1 |
| V26 | **Gap:** `checkGitHub` extra info describes the *matching* repo, not `items[0]` | code review | Auto **XFAIL** | 7 |
| V27 | **Gap:** batch recall — a low-star exact match outside the top 30 is still reported `taken` | code review | **Auto @live** **XFAIL** | 11 |

---

## Implementation Steps

Each step is atomic, carries embedded acceptance criteria, an explicit dependency, and a human-in-the-loop flag.

### Phase 0 — Baseline and environment

**Step 1 — Establish a clean baseline on the pinned Node version.** *Depends on: none. Agent-executable. → bead `c9j`.*
`nvm use` (Node 22 per `.nvmrc`), then `pnpm install --frozen-lockfile`, `pnpm exec eslint . --max-warnings=0`, `pnpm build`; record all output.
*AC:* lint exits 0 **with `--max-warnings=0`** — the bare `pnpm lint` script (`"lint": "eslint"`) exits 0 even when warnings exist, which is not what "no warnings" means, so the check must pass the flag explicitly; `pnpm build` succeeds; the Node version used is recorded alongside the output and matches `.nvmrc`; the local/`.nvmrc` mismatch (v24.1.0 vs 22) is noted in the evidence log. Any failure is filed as a beads issue **before** any other step proceeds, with one exception: a baseline defect blocking Step 1 itself is fixed inline rather than deadlocking the epic. Satisfies **V25**.

**Step 2 — Neutralize the stray `.env` and configure `.env.local`.** *Depends on: 1.* 🧑 **BLOCKED-ON-USER — requires consent to rename `.env`, and requires the user to supply `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, and `GITHUB_TOKEN`.** *→ bead `zok`.*
Ask before touching `.env`; on consent run `mv .env .env.bak`. Then `cp .env.example .env.local`, `chmod 600`, and populate (1Password Environment preferred).
*AC:* `.env` no longer exists, `.env.bak` does, and the user explicitly approved the rename; `git status --short` shows no `.env*` entry; the widened secret grep from Decision 2 returns nothing outside `.env.local` and `.env.bak`.

**Step 3 — Prove the three run profiles.** *Depends on: 2. Agent-executable. → bead `zok`.*
Boot the app under each profile and inspect `GET /api/providers`.
*AC:* **full-keys** reports `available: true` for `claude` (and `openai` if supplied); **no-keys** reports `available: false` for all five entries and the suggest button is disabled; **no-github-token** still returns GitHub results. Each profile's exact command and observed JSON is recorded. Closes `application-namer-zok`.

### Phase 1 — Harness

**Step 4 — Install the interception layer and prove it works.** *Depends on: 1. Agent-executable. → new bead.*
Add `@playwright/test` and `undici` as devDependencies, create `src/instrumentation.ts` and `e2e/mocks/install-mock-agent.ts` per Decision 3, and write a single throwaway assertion that a route-handler fetch is intercepted.
*AC:* with `E2E_MOCK_REGISTRIES=1`, a request to `/api/check` returns mocked registry data and **no packet leaves the machine** (verified by `disableNetConnect()` plus a run with the network disabled); with the flag unset, `register()` is a no-op and the app behaves exactly as before; the interception assertion passes on the `.nvmrc` Node version; if the undici symbol handshake fails, the module is switched to `msw/node` and only that file changes. The dummy key literal `sk-ant-e2e-dummy-not-a-real-key` is added to the secret-scan allowlist with a comment explaining why.

**Step 5 — Add the cache-neutralization seam.** *Depends on: 4. Agent-executable. → new bead.*
Add `src/app/api/__e2e__/reset/route.ts` calling `clearCache()`, gated on `E2E_TEST_HOOKS === "1"`; add the per-test unique-name helper to `e2e/fixtures/test-base.ts`.
*AC:* `POST /api/__e2e__/reset` returns `204` with the flag set and `404` with it unset (asserted both ways); a spec that searches the same name twice observes a cold cache after calling reset; `pnpm build` output shows the route present but inert without the flag.

**Step 6 — Author the two Playwright configs.** *Depends on: 4, 5. Agent-executable. → new bead.*
`playwright.config.ts` (projects `stubbed` + `mobile`, port 3100, fully pinned `webServer.env` exactly as specified in Test Architecture) and `playwright.live.config.ts` (project `live`, port 3101, `grep: /@live/`). Add `test:e2e`, `test:e2e:mobile`, `test:e2e:live`, `test:e2e:ui` scripts. Add `test-results/`, `playwright-report/`, `blob-report/`, `.playwright/` to `.gitignore`.
*AC:* `pnpm test:e2e` boots on port 3100, runs a smoke assertion, exits 0, and reads **no** value from `.env.local`; `pnpm test:e2e:live` boots on 3101 with mocks off; the two never collide on a port; `npx playwright test --list` on the default config shows zero `@live` tests; no real credential appears in either config.

### Phase 2 — Specs (Steps 7–10 are mutually independent and may run in parallel)

**Step 7 — `search.spec.ts` — core availability behaviour.** *Depends on: 6. Agent-executable. → new bead.* Covers V2–V5, V6 (client half only), V14, V16, V23 (client half only), V26.
*AC:* each canonical search asserts exact per-registry status in a **passing** test, with the promised extra-info rendering asserted in a **separate `test.fail()`** test — npm version+downloads, PyPI version+author, GitHub owner+stars, and cask app name are all expected to be missing, and Homebrew's homepage is expected to render; `"../etc/passwd"` is rejected **client-side** by `validatePackageName` before submission, and the test asserts the visible error text plus that the request counter stays at **zero** — this proves client-side validation, *not* server behavior, since the browser never sends the request (the server-side half of V6/V23 is asserted for real in Step 10, which can observe an actual HTTP response); `"my-tool"` renders the PyPI warning and the exact string is recorded (two different wordings exist in the codebase); all four `RegistryStatus` values render distinct indicators; `Enter` submits without a click; `@scope/name` and a 215-character name are rejected **client-side** the same way; V26 pins that GitHub extra info currently comes from `items[0]` rather than the matching repo.

**Step 8 — `resilience.spec.ts` — failure and race behaviour.** *Depends on: 6. Agent-executable. → new bead.* Covers V10, V13, V19, V21.
*AC:* V10 makes every mocked registry fail and asserts five `error` cards with no `pageerror`; V13 fires two searches <300 ms apart (`beforeEach` reset, same name) and asserts the first is aborted and never renders; V19 mocks GitHub `403` and asserts a `rate_limited` indicator (passing) plus the documented `429 {error, retryAfter}` contract (`test.fail()`); V21 fails a registry, recovers the mock, re-searches the same name inside the TTL, and pins the stale cached error via `test.fail()`.

**Step 9 — `responsive.spec.ts`.** *Depends on: 6. Agent-executable. → new bead.* Covers V11, V17.
*AC:* at 375×812, `document.documentElement.scrollWidth <= clientWidth` and result cards are vertically stacked; at 1280×720 the grid is multi-column; screenshots captured at both widths and attached to the report.

**Step 10 — `api-contract.spec.ts` — error-schema conformance.** *Depends on: 6. Agent-executable. → new bead.* Covers V6 (server half), V15, V18, V20, V23 (server half), V24. Uses the `request` fixture, no browser — this is what actually exercises server behavior, since Step 7's client-side spec can't (the form blocks the request before it's sent).
*AC:* both routes return `400` with exactly `{ error: string }` for an invalid name sent **directly via the `request` fixture** — `"../etc/passwd"` (V6) and a 215-char / `@scope/name` value (V23) — proving server-side validation independent of the client, with **zero** outbound registry calls in each case; a malformed JSON body is sent to both routes and the documented schema asserted via `test.fail()`, with the actual status and body recorded verbatim; the `500 {error, registry}` contract asserted via `test.fail()`; `/api/providers` asserted to be a bare array of `{id, name, available}`; `/api/suggest` (provider intercepted) returns ≤8 suggestions each carrying a full `Record<RegistryId, RegistryResult>`, and `errors` is asserted absent with a comment linking the spec-drift bead.

**Step 11 — `live-smoke.spec.ts`.** *Depends on: 6, 7. Agent-executable. → new bead.* Covers V8, V22, V27 and re-runs V2–V5 against real registries.
*AC:* V8 asserts exactly one outbound `api.github.com` call for an 8-suggestion re-check (passing) **and** that the call succeeds (`test.fail()` — 7 `OR` operators exceed GitHub's documented five-operator limit, so a `422` is expected); V22 measures 10 `/api/check` runs against `pnpm start` and reports p95 < 5 s, recorded as an observation rather than a hard gate; V27 searches a known low-star exact-match name through the batch path and pins any false `available`; the file header states it is excluded from CI and may be flaky by design; any live divergence from the v1 plan's predictions is filed as a bead.

### Phase 3 — Human verification (may run in parallel with Phase 1–2)

**Step 12 — Author and execute `docs/qa-manual-checklist.md`.** *Depends on: 3.* 🧑 **HUMAN-IN-THE-LOOP.** *→ bead `c9j`.*
Cover V1, V7, V9, V12 and a visual pass, each with an evidence slot (screenshot path / log excerpt / pass-fail / notes).
*AC:* self-checkable (exact commands, exact expected output, no prior context assumed); all five entries have a recorded outcome; V7 records the actual suggestion list from a real Claude call and confirms every returned name is a valid package name; V9 is performed under the **no-keys** profile and confirms the disabled suggest button and its "No AI providers configured" tooltip — **and, as a required control, the same check is repeated under full-keys with a provider not yet selected**, confirming that case does *not* show the same tooltip, since `page.tsx:33`/`suggestions-panel.tsx:90-92` render near-identical text for both states and the check is worthless without the contrast; V12 is performed twice — bridges running and bridges stopped — or explicitly recorded as `blocked` with a reason. Closes the manual half of `application-namer-c9j`.

### Phase 4 — Reconcile and close

**Step 13 — Full run and evidence report.** *Depends on: 7–12. Agent-executable. → new bead.*
Run `pnpm test:e2e`, `pnpm test:e2e:mobile`, then `pnpm test:e2e:live`; merge with the manual outcomes into `.omc/plans/qa-epic-results.md`.
*AC:* every row V1–V27 has pass / xfail / fail / blocked recorded with evidence; a second invocation reproduces the same result set for the stubbed projects; every `test.fail()` that unexpectedly **passed** is called out prominently, since that means a defect was silently fixed and the plan's assumptions are stale.

**Step 14 — File every finding as a linked beads issue.** *Depends on: 13. Agent-executable. → new beads.*
*AC:* one issue per distinct defect with title, reproduction command, expected vs actual, and `--parent application-namer-avp` (or a new defect epic); every XFAIL in the matrix has a corresponding issue; no failure from Step 13 is left unfiled; no issue body contains a credential.

**Step 15 — Reconcile plan-vs-implementation drift.** *Depends on: 13. Agent-executable, documentation only. → new bead.*
Correct `.omc/plans/application-namer-v1.md` where the **plan** is the stale artifact: MCP default ports (`8940/8941/8945` → `8960/8961/8962`), AC #3's three statuses → four, and a note that `/api/providers` returns a bare array.
*AC:* documentation changes only, no source file touched; each edit cites a row from the Current State table. Cases where the **code** is wrong (FIFO vs LRU, `extra` keys, batch operator limit) are *not* edited into the plan — they are bugs, and go to Step 14.

**Step 16 — Wire the stubbed suite into CI.** *Depends on: 13. Agent-executable. Optional — coordinate with bead `application-namer-upf`.*
Add `pnpm exec playwright install --with-deps chromium` and `pnpm test:e2e` to `.github/workflows/ci.yml` after `pnpm build`, with browser caching and the live config excluded.
*AC:* CI green on a PR; added wall-clock under 4 minutes; the job requires **no** secret; `playwright-report` uploaded on failure only; the live config is provably not invoked.

**Step 17 — Close the epic.** *Depends on: 13, 14, 15.* 🧑 **BLOCKED-ON-USER — the user decides whether outstanding defects block release.**
*AC:* `application-namer-zok` and `application-namer-c9j` are closed; `application-namer-avp` closes **only** after the user has reviewed the defect list and explicitly accepted or deferred each item; deferred items stay open with a rationale.

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation | Step |
|------|-----------|--------|------------|------|
| undici `setGlobalDispatcher` fails to steer Node's built-in fetch (version skew) | Medium | **High** | Interception is proven by an explicit assertion *before* any spec is written; harness pinned to the `.nvmrc` Node version; `msw/node` named as a drop-in fallback that changes exactly one file | 4 |
| Test-only routes/flags leak into production behaviour | Low | **High** | Both seams are env-gated (`E2E_MOCK_REGISTRIES`, `E2E_TEST_HOOKS`), asserted to 404/no-op when unset, and set only by the stubbed Playwright config | 4, 5 |
| Module-level cache leaks state between tests, producing false passes | **High** | Medium | Unique per-test names by default plus an explicit reset seam for the specs that must repeat a name | 5 |
| Stubbed run silently makes real network calls | Medium | Medium | `disableNetConnect()` turns any unmocked call into a hard error; the whole suite is re-run with the machine offline as an epic-level verification step | 4, 13 |
| Live registry data drifts (a name is unpublished or claimed) | Medium | Medium | Deterministic assertions live in the stubbed config; live checks are quarantined in `live-smoke.spec.ts`, excluded from CI, and expected to need occasional fixture updates | 11 |
| GitHub unauthenticated rate limit throttles the live run | Medium | Medium | Stubbed run makes zero GitHub calls; live run is a single pass with `GITHUB_TOKEN` present | 3, 11 |
| Secret leaks into a trace, screenshot, or HTML report | Low | **High** | Reports gitignored; the harness pins dummy credentials and never reads `.env.local`; widened secret grep (`sk-ant-`, `sk-proj-`, `sk-svcacct-`, `ghp_`, `github_pat_`, `gh[opsu]_`) run at Step 2 and again before the Step 13 report is written | 2, 6, 13 |
| Real AI calls incur cost or non-determinism | Medium | Low | The provider is intercepted in automation; real output is verified exactly once, manually | 10, 12 |
| Scope creep — the epic starts fixing defects | **High** | Medium | XFAIL is the designed outcome; Step 14 files issues rather than patches; Step 15 is explicitly documentation-only | 7–11, 14, 15 |
| An XFAIL quietly starts passing and nobody notices | Medium | Medium | Playwright reports an unexpected pass as a failure; Step 13's AC requires those to be called out explicitly | 13 |
| MCP bridges unavailable on the QA machine, blocking V12 | Medium | Low | V12 recorded as `blocked` with a reason rather than silently skipped; the bridges-down half still covers graceful degradation | 12 |
| p95 timing (V22) flaky on a loaded machine | Medium | Low | Measured against `pnpm start`, 10 samples, live config only, reported as an observation not a gate | 11 |
| Playwright browser download bloats CI | Low | Low | `chromium` only, `--with-deps`, Actions cache; live config excluded | 16 |

---

## Verification Steps (for this epic)

1. `mv .env .env.bak && cp .env.example .env.local` leaves exactly one active local env file. (Note: `git status --short` showing no `.env*` path is **not** a meaningful check here — `.gitignore`'s blanket `.env*` rule already hides these paths from git regardless of what this epic does, so that assertion cannot fail. The real check is: `git check-ignore -q .env.local && git check-ignore -q .env.bak` both exit 0, **and** `git ls-files | grep -E '^\.env'` returns only `.env.example`.)
2. `GET /api/providers` under **full-keys** returns at least one `available: true`; under **no-keys** all five are `available: false` — proving the profile switch works and AC #8 holds.
3. `pnpm test:e2e` exits 0 on a clean checkout **with `.env.local` absent** — the stubbed run must not require any secret.
4. Running `pnpm test:e2e` twice in a row produces an identical pass/xfail set.
5. **Disconnect the machine's network and re-run `pnpm test:e2e` — it still passes.** This is the definitive proof that interception works and no third-party call escapes.
6. With `E2E_MOCK_REGISTRIES` and `E2E_TEST_HOOKS` unset, `pnpm dev` behaves identically to today and `POST /api/__e2e__/reset` returns 404.
7. `npx playwright test --list` on the default config shows zero `@live` tests; `pnpm test:e2e:live` runs them.
8. `docs/qa-manual-checklist.md` is self-checkable: every row names an exact command and an exact expected string, with no step requiring context that isn't on the page. (This is a solo project — "handed to a second person" is not executable; the bar is that a cold reader of the doc alone, with no other context, could run it.)
9. `.omc/plans/qa-epic-results.md` has a non-empty outcome and evidence reference for every ID V1–V27.
10. `grep -rnE 'sk-ant-|sk-proj-|sk-svcacct-|ghp_|github_pat_|gh[opsu]_'` over the tree (excluding `node_modules`, `.git`, `.next`, `.env.local`, `.env.bak`) returns only the allowlisted dummy key literal — including inside `playwright-report/` and `test-results/`.
11. `bd list` shows one open issue per unresolved defect, each parented to `application-namer-avp`; `application-namer-zok` and `application-namer-c9j` are closed.
12. CI on a PR runs lint + build + stubbed e2e green with no secrets configured (if Step 16 is taken).

---

## RALPLAN-DR Summary

### Principles
1. **Verification is a deliverable, not an activity** — the epic must leave behind something re-runnable.
2. **Find and pin, don't fix** — defects are encoded as XFAIL specs and filed as beads; mixing verification with repair makes it impossible to say what was verified against what.
3. **Automate what a human does badly** — request counts, sub-second races, offline simulation, schema shape. Keep the human for judgement and cost-bearing calls.
4. **Hermetic by default, live by exception** — the routine suite must pass with the network unplugged; anything that genuinely needs the real internet is quarantined and excluded from CI.
5. **Test seams are inert in production** — every hook is env-gated and asserted inert when the flag is unset.
6. **Secrets have exactly one home** — `.env.local`; the harness pins dummies and never reads it.

### Decision Drivers
1. **Repeatability** — this verification recurs after every change; a one-shot manual pass fails that test.
2. **Determinism under third-party dependence** — five registries and three AI providers make naive end-to-end testing flaky; the stub/live split is forced by this.
3. **Secret exposure surface** — real keys must unlock real code paths without ever reaching a committed file or a test artifact.

### Options
Documented in **Decision 1**: Option A (commit a Playwright suite — chosen), Option B (pure manual — invalidated for non-repeatability), Option C (agent-driven Playwright MCP only — invalidated as a deliverable, adopted as an authoring tool).

### ADR-1: Automation-first hybrid QA harness

- **Decision:** Add `@playwright/test` with a hermetic stubbed config (projects `stubbed` + `mobile`) and a separate quarantined live config; keep a four-item human checklist for AI-provider, MCP-bridge, zero-key, and visual verification.
- **Drivers:** repeatability across future releases; determinism despite five third-party registries; the impossibility of hand-verifying request counts, sub-300 ms races, and offline behaviour; existing committed intent to add test tooling.
- **Alternatives considered:** pure manual checklist — rejected for non-repeatability; agent-driven Playwright MCP only — rejected as a deliverable, retained as an authoring aid.
- **Why chosen:** converts most of the verification surface into a permanent regression guard for a one-time half-day cost, while honestly conceding the remainder to a human rather than building brittle automation around money-spending, process-dependent, and subjective checks.
- **Consequences:** two devDependencies plus a browser download; a mock fixture that must track registry response shapes; two production-tree files (`instrumentation.ts`, the reset route) that exist only for testing, both env-gated.
- **Follow-ups:** align the runner choice with epic `application-namer-9w9`; revisit `@live` fixture assumptions periodically; if the app is deployed (`application-namer-aqq`), point the live config at the deployed URL as a post-deploy smoke test.

### ADR-2: In-process undici interception over an external mock server

- **Decision:** Intercept registry traffic inside the Next.js server process via `src/instrumentation.ts` + undici `MockAgent` + `disableNetConnect()`, rather than running a separate mock HTTP server or making registry base URLs configurable.
- **Drivers:** all registry fetches are server-side, so browser-level routing cannot see them; making base URLs configurable would require changing production source purely for testability, which is out of scope for a QA epic; `disableNetConnect()` gives a hard guarantee that nothing escapes, which an external mock server cannot.
- **Alternatives considered:** (a) a local mock registry host with env-injected base URLs — rejected because it requires production code changes and provides no escape guarantee; (b) `/api/*`-contract assertions over the live network — rejected as non-deterministic and rate-limit-bound; (c) `msw/node` — not rejected, retained as the named fallback if the undici symbol handshake fails on Node 22.
- **Why chosen:** it is the only option that needs no production behaviour change (the hook is inert without an env flag), gives a provable no-egress guarantee, and keeps the spec-facing fixture API identical whichever interceptor sits underneath.
- **Consequences:** interception is undici-version-sensitive, so the harness must run on the `.nvmrc` Node version and must assert interception before any spec is trusted; two env-gated files live in the production tree.
- **Follow-ups:** if `application-namer-9w9`'s unit/integration work adopts a different mocking layer, converge on one; revisit if Next.js ever changes the `instrumentation.ts` contract.

---

## Changelog (consensus revisions)

### From Architect Review (arch-qa)
- [x] **Resolved the Step 4 stubbing spike outright** — replaced the timeboxed "pick one of three approaches" with a specified design: `src/instrumentation.ts` + undici `MockAgent` + `setGlobalDispatcher` + `disableNetConnect()`, env-gated on `E2E_MOCK_REGISTRIES`, with `msw/node` as the named fallback (new Decision 3, ADR-2)
- [x] **Split `webServer` into two configs** — `playwright.config.ts` (stubbed, port 3100) and `playwright.live.config.ts` (live, port 3101), because Playwright's `webServer` is global rather than per-project
- [x] **Pinned `webServer.env`** for the stubbed config — dummy `ANTHROPIC_API_KEY`, empty `OPENAI_API_KEY`/`GITHUB_TOKEN`, and `MCP_*_URL=http://127.0.0.1:9` so a bridge running on the dev machine cannot influence a test run; harness never reads `.env.local`
- [x] **Added cache neutralization** — the module-level `Map` in `cache.ts` is never cleared; added per-test unique names plus an env-gated `POST /api/__e2e__/reset` seam that finally gives dead-code `clearCache()` a caller (Step 5)
- [x] **Moved V8 and V22 to `@live`** — the stubbed run cannot exercise GitHub's real ~5-operator OR-query limit, and a p95 timing assertion against instant mocks is meaningless
- [x] **Reclassified V2 / V4 / V5 as split pass + XFAIL** — status assertions pass, but the promised extra-info rendering fails because of the confirmed `extra`-key contract drift
- [x] **Struck the false `checkGitHubBatch` claim** — it wraps its body in `try/catch` and returns a per-name error map; it does **not** reject, so `Promise.all` in `checkNames` was never at risk
- [x] **Changed `rm .env` to `mv .env .env.bak` with explicit user consent** — it is the user's untracked file, not the plan's to delete
- [x] **Fixed the off-by-one AC range** — epic AC #4 now reads V15–V27 (was V15–V24 against a matrix ending at V25)
- [x] **Widened the secret grep** — now `sk-ant-|sk-proj-|sk-svcacct-|ghp_|github_pat_|gh[opsu]_`, with the dummy harness key explicitly allowlisted

### From planner re-verification against source
- [x] Documented the full **`extra`-key drift** per registry (npm `latestVersion`/`weeklyDownloads`, PyPI `latestVersion`/`author`, GitHub `owner`/`stars` — none rendered; only Homebrew's `homepage` matches)
- [x] Added **V26** — `checkGitHub` sources extra info from `items[0]` rather than the exact-name match
- [x] Added **V27** — batch `per_page=30` sorted by stars can miss a low-star exact match and report `available` when taken
- [x] Recorded the **Node version mismatch** (`.nvmrc` 22 vs local v24.1.0) and made "run on the `.nvmrc` version" an AC of Step 1 and Step 4, since interception is undici-version-sensitive
- [x] Recorded **FIFO-not-LRU** eviction, the **duplicate PyPI warning strings**, and the **uncaptured cask app name** as Current State facts feeding Steps 7 and 14
- [x] Made **expected-fail a first-class concept** with `test.fail()`, and added a Step 13 AC requiring unexpected passes to be surfaced
- [x] Added an epic-level verification step: **run the stubbed suite with the network disconnected** — the definitive proof of hermeticity
- [x] Added an epic-level verification step confirming both test seams are **inert when their env flags are unset**
- [x] Renumbered the implementation steps from 14 to 17 and reordered into four phases so that Steps 7–10 are explicitly parallelizable

### From Critic Review (critic-qa)
- [x] **MF-3 already satisfied by this revision's phase structure** — re-checked: Step 4 (harness) depends only on Step 1, not Step 2; Steps 6-11 (the stubbed suite, ~70% of the epic's value) chain from Step 4 and never touch Step 2's blocked-on-user gate. Only Step 3 (proving run profiles) and Step 12 (human checklist) depend on 2. No further split needed.
- [x] **MF-5 fixed** — V6 and V23 were asserted only in the browser-level `search.spec.ts` (Step 7), where client-side `validatePackageName` blocks the request before it's ever sent, so the "zero outbound calls" assertion proved client validation, not server behavior. Split: Step 7 now explicitly scopes itself to the client half; Step 10 (`api-contract.spec.ts`, uses the `request` fixture, no browser) now covers the server half for real, sending the same payloads directly to the route handlers.
- [x] **MF-9 fixed** — Step 12's V9 check for the "No AI providers configured" tooltip added a required control: repeat the check under full-keys with no provider selected yet, and confirm that state does *not* show the same tooltip (`page.tsx:33`/`suggestions-panel.tsx:90-92` render near-identical text for both, so the original check couldn't distinguish "no keys" from "haven't picked one yet").
- [x] **MF-7 (four unfalsifiable ACs), three of four fixed:** Step 1's lint AC now specifies `--max-warnings=0` (the bare `pnpm lint` script exits 0 with warnings present); Verification Step 1's `git status --short` check replaced with `git check-ignore` + `git ls-files`, since `.env*` is already blanket-ignored and the original check could never fail; Verification Step 8's "handed to a second person" replaced with a self-checkability bar, since this is a solo project. (The fourth — Step 16's "under 4 minutes" CI wall-clock bound — is left as an approximate, non-blocking target; not worth more precision for a side project.)
- [x] Cross-checked MF-6 (the false `checkGitHubBatch`-rejects claim) against this plan's Current State table — already absent; arch-qa and critic-qa both independently caught and struck the same false claim from earlier drafts, so no drift was inherited.
- Not folded in (minor, logged for a future pass if this epic is picked up): MF-1's determinism framing was independently addressed by arch-qa's cache-neutralization seam (Step 5) rather than critic-qa's exact wording, MF-2's `registry-client.ts` naming is implicit in Decision 3 rather than stated explicitly, and MF-4/MF-8 were already fixed by arch-qa's pass (extra-key XFAIL reclassification, widened secret grep) before critic-qa's findings arrived.
