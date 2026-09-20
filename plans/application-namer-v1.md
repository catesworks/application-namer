# Application Namer - Implementation Plan (v2 — post-consensus revision)

## Requirements Summary

Build a Next.js web application that checks application/package name availability across multiple registries simultaneously and offers AI-generated alternative name suggestions when a name is taken.

**Target registries:**
- **npm** (npmjs.com) — CLI tools and Node packages
- **Homebrew formulae** — CLI tools installed via `brew install`
- **Homebrew Cask** — Desktop applications installed via `brew install --cask`
- **PyPI** — Python packages and CLI executables
- **GitHub repositories** — Top-level public repo name search

**AI suggestion providers (user-selectable):**
- Claude API (direct, via Anthropic SDK)
- OpenAI API (direct, via OpenAI SDK)
- MCP Agent Bridge servers (Claude/Codex/Copilot at localhost) — for local development use

**Deployment target:** Local development (`pnpm dev`). Architecture is Vercel-compatible for future deployment.

---

## Acceptance Criteria

1. User can type a name into a search input and submit it via `<form>` submission (Enter key works natively)
2. App checks all 5 registries concurrently and displays results within 5 seconds (p95), enforced by per-registry 4-second fetch timeouts
3. Each registry shows one of: `available`, `taken`, or `error` with a clear visual indicator (green check, red X, yellow warning)
4. When a name is taken on any registry, a "Suggest alternatives" button appears
5. User can select their AI provider (Claude API, OpenAI API, or MCP bridge server) from a dropdown
6. AI suggestions return 8 alternative names, each re-checked server-side against all registries before being returned to the client. Results include per-suggestion registry availability. GitHub re-checks use a batched OR query (1 API call for all suggestions) to stay within rate limits.
7. App handles GitHub rate limiting gracefully: detects 403/429 responses, shows "rate limited" status for GitHub results, does not block other registries
8. App works without any API keys configured (registry checks are unauthenticated) — AI suggestions require at least one provider configured
9. Environment variables: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GITHUB_TOKEN` (optional), `MCP_CLAUDE_URL` (default `http://localhost:8940`), `MCP_CODEX_URL` (default `http://localhost:8941`), `MCP_COPILOT_URL` (default `http://localhost:8945`)
10. Responsive design: works at viewport widths >= 375px (mobile) and >= 1024px (desktop)

---

## Architecture

### Tech Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Framework | Next.js 15 (App Router) | User preference, SSR for SEO, API routes built-in |
| Language | TypeScript | Type safety across registry responses |
| Styling | Tailwind CSS v4 | Fast iteration, CSS-based config (no tailwind.config.ts) |
| UI Components | shadcn/ui (via CLI: `npx shadcn@latest init/add`) | Polished components, no runtime overhead |
| Runtime | Node.js (server-side fetches) | Avoids CORS issues, keeps API keys server-side |
| Package Manager | pnpm | Fast, disk-efficient |

### Project Structure

```
application-namer/
├── src/
│   ├── app/
│   │   ├── layout.tsx              # Root layout with error boundary
│   │   ├── page.tsx                # Main search page
│   │   ├── api/
│   │   │   ├── check/route.ts      # POST: check name across all registries
│   │   │   ├── suggest/route.ts    # POST: generate + re-check AI suggestions
│   │   │   └── providers/route.ts  # GET: list available AI providers
│   │   └── globals.css             # Tailwind v4 CSS imports + theme
│   ├── components/
│   │   ├── search-form.tsx         # <form> with name input + submit
│   │   ├── results-grid.tsx        # Registry availability results
│   │   ├── result-card.tsx         # Individual registry result
│   │   ├── suggestions-panel.tsx   # AI-generated alternatives (wrapped in error boundary)
│   │   ├── provider-selector.tsx   # AI provider picker
│   │   └── header.tsx              # App header/title
│   ├── lib/
│   │   ├── validation.ts           # Shared name validation (server + client)
│   │   ├── registry-client.ts      # Shared HTTP fetch with timeout + error normalization
│   │   ├── registries/
│   │   │   ├── npm.ts              # npm registry check (2 calls: registry + downloads)
│   │   │   ├── homebrew.ts         # Homebrew formulae + cask check
│   │   │   ├── pypi.ts             # PyPI registry check
│   │   │   ├── github.ts           # GitHub repo search (single + batched OR query)
│   │   │   ├── cache.ts            # In-memory cache with 5-min TTL
│   │   │   └── index.ts            # Orchestrator: single name + batch names
│   │   ├── suggestions/
│   │   │   ├── claude.ts           # Claude API provider
│   │   │   ├── openai.ts           # OpenAI API provider
│   │   │   ├── mcp-bridge.ts       # MCP agent bridge provider
│   │   │   └── index.ts            # Provider router with availability detection
│   │   └── types.ts                # Shared types + error response schemas
│   └── hooks/
│       └── use-name-check.ts       # Client state: debounce, AbortController, results
├── .env.example                    # Environment variable template
├── package.json
├── tsconfig.json
├── next.config.ts
└── postcss.config.mjs
```

**Note:** Tailwind CSS v4 uses CSS-based configuration (`globals.css`), not `tailwind.config.ts`. No separate Tailwind config file is needed.

---

## Registry Check Strategy

Each registry is checked via its public HTTP API from server-side route handlers. All fetches go through the shared `registry-client.ts` which provides: 4-second timeout, error normalization to `RegistryResult`, status code interpretation, and cache integration.

### 1. npm Registry
- **Endpoint 1:** `GET https://registry.npmjs.org/{name}` — package metadata
- **Endpoint 2:** `GET https://api.npmjs.org/downloads/point/last-week/{name}` — weekly downloads (separate API)
- **Available:** Endpoint 1 returns HTTP 404
- **Taken:** Endpoint 1 returns HTTP 200 (response includes description, latest version)
- **Rate limit:** Generous, no auth needed
- **Extra info to show:** Latest version, description, weekly downloads (from endpoint 2)
- **Implementation note:** Both endpoints are fetched concurrently. If downloads endpoint fails, show "N/A" for downloads — don't fail the whole check.

### 2. Homebrew Formulae
- **Endpoint:** `GET https://formulae.brew.sh/api/formula/{name}.json`
- **Available:** HTTP 404
- **Taken:** HTTP 200
- **Rate limit:** No auth needed, CDN-backed
- **Extra info to show:** Description, homepage URL

### 3. Homebrew Cask
- **Endpoint:** `GET https://formulae.brew.sh/api/cask/{name}.json`
- **Available:** HTTP 404
- **Taken:** HTTP 200
- **Rate limit:** No auth needed, CDN-backed
- **Extra info to show:** App name, homepage URL

### 4. PyPI
- **Endpoint:** `GET https://pypi.org/pypi/{name}/json`
- **Available:** HTTP 404
- **Taken:** HTTP 200
- **Rate limit:** No auth needed
- **Extra info to show:** Summary, latest version, author
- **Note on normalization:** PyPI normalizes `-`, `_`, and `.` to be equivalent. Display a note when the user's name contains these characters: "PyPI treats hyphens, underscores, and dots as equivalent."

### 5. GitHub Repositories
- **Single check endpoint:** `GET https://api.github.com/search/repositories?q={name}+in:name&sort=stars&per_page=5`
- **Batch check endpoint:** `GET https://api.github.com/search/repositories?q={name1}+OR+{name2}+OR+{name3}+in:name&sort=stars&per_page=30`
- **Available:** No exact name match in results (case-insensitive)
- **Taken:** Exact name match found (case-insensitive)
- **Rate limit:** 10 req/min unauthenticated, 30 req/min with `GITHUB_TOKEN`
- **Extra info to show:** Top matching repo's owner, stars, description
- **Batch strategy for suggestions:** When re-checking AI suggestions, combine all 8 names into a single OR query. This costs 1 API call instead of 8, then match results against each suggestion name.

---

## Caching Strategy (v1)

In-memory `Map<string, { result: RegistryResult, expiresAt: number }>` in `src/lib/registries/cache.ts`:
- **Key:** `${registryId}:${normalizedName}` — for PyPI, normalize `-`, `_`, `.` to `-` in the key (since PyPI treats them as equivalent)
- **TTL:** 5 minutes
- **Max entries:** 1000 (LRU eviction — evict oldest entry when limit reached)
- **Scope:** Per-process (resets on server restart, which is fine for local dev)
- **Integration:** The orchestrator (`index.ts`) checks cache before calling registry fetchers. Cache is populated after successful checks.
- **Why v1 needs this:** Without caching, the suggestion re-check feature fires up to 40 registry calls. With caching, a name that was already checked (e.g., the original search) is served from cache. The batched GitHub OR query further reduces calls from 8 to 1.

---

## Input Validation

Shared validation function in `src/lib/validation.ts`, used by both API routes (server-side, mandatory) and the search form (client-side, for UX).

```typescript
interface ValidationResult {
  valid: boolean;
  error?: string;
  warnings?: string[]; // e.g., "PyPI normalizes hyphens and underscores"
}

function validatePackageName(name: string): ValidationResult
```

**Rules:**
- Must match regex: `/^[a-z0-9][a-z0-9._-]*$/` (cannot start with `.` or `_`)
- Maximum length: 214 characters (npm limit — most restrictive)
- Cannot be empty
- No URL-unsafe characters, no path traversal sequences
- **Warnings** (non-blocking): If name contains `-`, `_`, or `.`, warn that PyPI treats these as equivalent

**Server-side enforcement:** Both `POST /api/check` and `POST /api/suggest` validate the name and return `400 { error: string }` if invalid, before constructing any URLs.

**Scoped packages:** Not supported in v1. Names starting with `@` are rejected. (Scoped packages are npm-specific and don't apply to other registries.)

---

## AI Suggestion Strategy

### Provider Interface

```typescript
interface SuggestionProvider {
  id: string;
  name: string;
  generateSuggestions(name: string, context: NameContext): Promise<string[]>;
}

interface NameContext {
  takenOn: string[]; // registries where original name is taken
}
```

**Note:** The `type` field (cli/desktop/library) from the original plan is removed. The UI doesn't collect this, and the AI can infer context from the registry results (e.g., taken on Homebrew Cask suggests desktop app).

### Provider Implementations

1. **Claude API** (`@anthropic-ai/sdk`)
   - Model: `claude-sonnet-4-20250514` (fast, creative)
   - Structured output via tool_use to get JSON array of names
   - Available when `ANTHROPIC_API_KEY` env var is set
   - Error handling: catch `AuthenticationError` (invalid key) → return provider unavailable; catch malformed response → return empty suggestions with error message

2. **OpenAI API** (`openai`)
   - Model: `gpt-4o-mini` (fast, cheap)
   - JSON mode (`response_format: { type: "json_object" }`) for structured output
   - Available when `OPENAI_API_KEY` env var is set
   - Error handling: same pattern as Claude

3. **MCP Bridge** (HTTP to localhost)
   - Uses the `ask` tool on Claude/Codex/Copilot servers
   - Server URLs configurable via `MCP_CLAUDE_URL`, `MCP_CODEX_URL`, `MCP_COPILOT_URL` env vars (defaults: `:8940`, `:8941`, `:8945`)
   - Available when respective server responds to HTTP health check (2-second timeout)
   - Error handling: connection refused or timeout → mark as unavailable, don't error
   - Falls back to text parsing if response isn't valid JSON

### Suggestion Prompt Template

```
Suggest 8 creative, memorable alternative names for a software project called "{name}".

Context: This name is already taken on: {takenOn}.

Requirements for suggestions:
- Each name should be a single word or hyphenated (valid package name: lowercase, alphanumeric, hyphens)
- Names should be memorable, easy to spell, and related to the original concept
- Avoid generic prefixes like "my-" or "the-"
- Mix approaches: synonyms, metaphors, portmanteaus, related concepts

Return ONLY a JSON array of strings, e.g.: ["name1", "name2", ...]
```

### Suggest Route Architecture (critical path)

The `POST /api/suggest` route handles the full suggest-and-recheck flow server-side:

1. Validate input name (400 if invalid)
2. Call the selected AI provider to generate 8 name suggestions
3. Validate each suggestion against `validatePackageName()` — discard invalid ones
4. Re-check all valid suggestions against all registries using the batch orchestrator:
   - npm, Homebrew, PyPI: individual calls per suggestion (cache deduplicates repeats)
   - GitHub: single batched OR query for all suggestions (1 API call)
5. Return response:
```typescript
{
  suggestions: Array<{
    name: string;
    results: Record<RegistryId, RegistryResult>;
  }>;
  errors?: string[]; // provider errors, rate limit warnings
}
```

This keeps all registry calls server-side, maximizes cache hits, and uses the batched GitHub query to avoid rate limits.

**Edge cases:**
- If the AI returns fewer than 8 valid names (after validation filtering), return what we have — the client displays however many are available with no special messaging needed.
- If all AI providers are unavailable, the `/api/providers` route returns an empty list and the "Suggest alternatives" button is disabled in the UI with a tooltip: "No AI providers configured."

---

## API Error Response Schema

All API routes return errors in a consistent format:

```typescript
// 400 Bad Request — invalid input
{ error: "Name must match pattern: lowercase alphanumeric, hyphens, dots, underscores" }

// 429 Too Many Requests — rate limited
{ error: "GitHub API rate limit exceeded. Try again in 45 seconds.", retryAfter: 45 }

// 500 Internal Server Error — unexpected failure
{ error: "Failed to check npm registry", registry: "npm" }

// 503 Service Unavailable — AI provider down
{ error: "Claude API is not available. Check your ANTHROPIC_API_KEY.", provider: "claude" }
```

---

## Implementation Steps

### Phase 1: Project Setup
1. Initialize Next.js 15 project: `pnpm create next-app@latest --typescript --tailwind --app --src-dir`
2. Initialize shadcn/ui via CLI: `npx shadcn@latest init`, then add components: `npx shadcn@latest add button input card badge select`
3. Install AI SDKs: `pnpm add @anthropic-ai/sdk openai`
4. Set up `.env.example` with all environment variables (see AC #9)
5. Configure `tsconfig.json` path aliases (`@/*` → `./src/*`)

### Phase 2: Core Library
6. Create `src/lib/types.ts` — shared types: `RegistryId`, `RegistryResult`, `RegistryStatus`, `SuggestionResult`, `CheckResponse`, `SuggestResponse`, error response types
7. Create `src/lib/validation.ts` — `validatePackageName()` with regex, length, and normalization warnings (used by both API routes and client form)
8. Create `src/lib/registry-client.ts` — shared `registryFetch(url, options)`: 4-second timeout via `AbortSignal.timeout()`, response status interpretation (404=available, 200=taken, else=error), error normalization to `RegistryResult`

### Phase 3: Registry Checkers
9. Implement `src/lib/registries/npm.ts` — two concurrent fetches: registry metadata + downloads API. Downloads failure returns "N/A", doesn't fail the check.
10. Implement `src/lib/registries/homebrew.ts` — fetch formulae + cask (2 calls via `registryFetch`)
11. Implement `src/lib/registries/pypi.ts` — fetch PyPI JSON API via `registryFetch`
12. Implement `src/lib/registries/github.ts` — two exports: `checkGitHub(name)` for single checks, `checkGitHubBatch(names)` for batched OR query. Both use `registryFetch`. Handle 403/429 rate limit responses explicitly → return `{ status: "rate_limited" }`.
13. Create `src/lib/registries/cache.ts` — in-memory Map with 5-min TTL, keyed by `registryId:name`
14. Create `src/lib/registries/index.ts` — two orchestrator functions:
    - `checkName(name)`: runs all 5 registries concurrently via `Promise.allSettled`, uses cache, returns `Record<RegistryId, RegistryResult>`
    - `checkNames(names)`: runs npm/homebrew/pypi per-name (with cache), GitHub as single batched OR query. Returns `Record<string, Record<RegistryId, RegistryResult>>`

### Phase 4: API Routes
15. Create `src/app/api/check/route.ts` — POST handler: validate name via `validatePackageName()` (return 400 if invalid), call `checkName()`, return results with cache headers
16. Create `src/app/api/suggest/route.ts` — POST handler: validate name, call AI provider, validate suggestions, call `checkNames()` for re-check, return `SuggestResponse`
17. Create `src/app/api/providers/route.ts` — GET handler: check which AI providers are available (env vars set, MCP servers reachable), return list

### Phase 5: AI Suggestion Providers
18. Implement `src/lib/suggestions/claude.ts` — Claude API with tool_use for structured JSON output. Handle auth errors, malformed responses.
19. Implement `src/lib/suggestions/openai.ts` — OpenAI API with JSON mode. Handle auth errors, malformed responses.
20. Implement `src/lib/suggestions/mcp-bridge.ts` — HTTP POST to MCP bridge `ask` tool. Parse JSON from response text. Handle connection refused, timeout. Server URLs from env vars.
21. Create `src/lib/suggestions/index.ts` — provider router: `getAvailableProviders()`, `generateSuggestions(name, provider, context)`

### Phase 6: UI Components
22. Create `src/components/header.tsx` — app title and one-line description
23. Create `src/components/search-form.tsx` — `<form>` with input (client-side validation via `validatePackageName`), submit button, loading spinner. Form submission triggers search.
24. Create `src/components/result-card.tsx` — registry result: icon (check/x/warning), registry name, status badge, expandable extra info (version, downloads, etc.)
25. Create `src/components/results-grid.tsx` — responsive grid of result cards. Shows PyPI normalization warning when applicable.
26. Create `src/components/provider-selector.tsx` — `<Select>` dropdown, fetches available providers from `/api/providers` on mount, disables unavailable ones
27. Create `src/components/suggestions-panel.tsx` — displays AI suggestions with their registry results. Wrapped in React Error Boundary to isolate crashes. Shows loading state during the re-check phase ("Checking availability of suggestions...").
28. Wire everything together in `src/app/page.tsx` — error boundary wrapping suggestions panel

### Phase 7: Client State & Integration
29. Create `src/hooks/use-name-check.ts`:
    - 300ms debounce on input (or instant on form submit)
    - `AbortController` to cancel in-flight requests when a new search starts
    - State: `{ query, results, suggestions, isChecking, isSuggesting, error }`
    - Calls `/api/check` for registry checks, `/api/suggest` for AI suggestions
30. Responsive design: test at 375px (mobile) and 1024px+ (desktop), stack cards vertically on mobile
31. Add `.env.example` documentation comments explaining each variable

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation | Implementation Step |
|------|-----------|--------|------------|---------------------|
| GitHub rate limiting (10 req/min unauth) | High | Medium | Batched OR query for suggestions (1 call instead of 8), in-memory cache (5-min TTL), detect 429/403 → show "rate limited" status, optional `GITHUB_TOKEN` | Steps 12, 13, 14 |
| Registry API changes/downtime | Low | Medium | Each check is independent via `Promise.allSettled` — one failure doesn't block others. Show `error` state per registry | Step 14 (orchestrator) |
| MCP bridge servers not running | Medium | Low | Runtime availability detection via health check with 2s timeout. Grey out unavailable providers in UI | Steps 17, 20, 26 |
| npm/PyPI name squatting | Medium | Low | Show extra info (last publish date, weekly downloads, version) so user can judge abandonment | Steps 9, 11 |
| AI returning invalid/unparseable names | Medium | Low | Validate each suggestion via `validatePackageName()` before re-checking. Discard invalid ones silently. | Step 16 |
| Malformed AI response crashes UI | Low | Medium | Error boundary around suggestions panel. Suggest route catches parse errors → returns empty suggestions + error message | Steps 16, 27 |
| Path traversal in name input | Low | High | Server-side validation via strict regex in both API routes, before any URL construction | Steps 7, 15, 16 |
| Stale results from in-flight race condition | Medium | Low | `AbortController` cancels previous request when new search starts | Step 29 |

---

## Verification Steps

1. `pnpm dev` starts without errors on a clean install
2. Searching "express" shows taken on npm (with version + downloads), PyPI, and GitHub; shows extra info for each
3. Searching "xyzzy-nonexistent-pkg-12345" shows available on all registries
4. Searching "git" shows taken on Homebrew formulae and GitHub
5. Searching "visual-studio-code" shows taken on Homebrew Cask
6. Submitting an invalid name (e.g., "../etc/passwd") returns 400 error, not a fetch to registries
7. AI suggestions return valid package names, each displayed with per-registry availability
8. GitHub re-check for suggestions uses only 1 API call (verify via network tab or server logs)
9. App works with zero API keys: registry checks work, AI provider list shows none available, suggest button is disabled
10. App handles network timeout: disconnect from internet, search shows error states (not crash)
11. Mobile viewport (375px) renders all cards stacked vertically, no horizontal scroll
12. MCP bridge providers show as available when servers are running, unavailable (greyed out) when stopped
13. Searching twice rapidly cancels the first request (no stale results displayed)
14. Searching a name with hyphens (e.g., "my-tool") shows PyPI normalization warning

---

## RALPLAN-DR Summary

### Principles
1. **Correctness over speed** — every displayed availability result must be accurate; never show "available" when a name is taken
2. **Graceful degradation** — individual registry or provider failures never block the rest of the results
3. **Rate limit awareness** — the app must work within API limits without requiring tokens
4. **Server-side security** — all user input validated server-side before constructing URLs or calling external APIs
5. **Simple v1, extensible later** — in-memory cache, no database, no auth; but architecture allows adding these

### Decision Drivers
1. **GitHub rate limits** — 10 req/min unauth is the tightest constraint; drives the batched OR query design
2. **AI suggestion re-check cost** — 8 suggestions x 5 registries = 40 calls without optimization; drives caching + batching
3. **API key security** — keys must stay server-side; drives the Next.js API route architecture

### Viable Options

**Option A: Server-side API routes (chosen)**
- Pros: API keys secure, CORS-free, caching is straightforward, Vercel-compatible
- Cons: Requires Node.js server (can't static export), adds one network hop

**Option B: Static SPA with client-side fetches**
- Pros: Simpler deployment (CDN/static host), no server to maintain
- Cons: API keys exposed in browser, CORS issues with GitHub API, cache limited to localStorage, no batch optimization possible server-side
- **Invalidation rationale:** AC #9 requires API keys for AI providers. Storing these client-side is a security anti-pattern. GitHub API also benefits from server-side token + caching. The marginal deployment simplicity doesn't justify the security tradeoff for a tool that handles API credentials.

### ADR: Server-Side Registry Checking Architecture

- **Decision:** All registry checks and AI suggestion generation happen in Next.js API routes, not client-side.
- **Drivers:** API key security, CORS avoidance, server-side caching for rate limit management, batch GitHub queries.
- **Alternatives considered:** (A) Full client-side SPA — rejected due to API key exposure and CORS. (B) Hybrid (client-side for CORS-friendly registries, server for GitHub/AI) — rejected for inconsistent architecture and split caching.
- **Why chosen:** Keeps all credentials server-side, enables in-memory caching that works across suggestion re-checks, allows batched GitHub queries, and is compatible with both local dev and Vercel deployment.
- **Consequences:** Requires running a Node.js server (can't deploy as static files). Adds ~50ms latency per request for the server hop.
- **Follow-ups:** If deployed publicly, add server-side rate limiting per IP. Consider Redis cache if multiple server instances are needed.

---

## Changelog (consensus revisions)

### From Architect Review
- [x] Added shared `registry-client.ts` with standardized timeout, error normalization, and cache hook
- [x] Added `AbortController` + 300ms debounce to client hook spec
- [x] Added server-side input validation in both API routes (moved before Phase 6)
- [x] Made MCP bridge ports configurable via environment variables
- [x] Added React Error Boundary around suggestions panel
- [x] Added batched/throttled re-checking strategy for AI suggestions

### From Architect Review (Iteration 2 — minor improvements)
- [x] Added PyPI cache key normalization (`-`, `_`, `.` → `-` in cache key)
- [x] Added LRU eviction cap (max 1000 entries) on in-memory cache
- [x] Fixed path alias: `@/*` → `./src/*` (Next.js convention)

### From Critic Review (Iteration 2 — minor improvements)
- [x] Added edge case spec for < 8 AI suggestions (return what we have)
- [x] Added spec for disabled suggest button when no AI providers available
- [x] Noted MCP bridge request body left to executor (follows MCP SDK conventions)

### From Critic Review (Iteration 1)
- [x] **CRITICAL:** Resolved caching contradiction — moved caching from "Open Decisions" to v1 implementation (Step 13), connected to risk mitigation
- [x] Added npm downloads as explicit second API endpoint (Step 9)
- [x] Specified suggest route re-check architecture explicitly (server-side, with batch GitHub query)
- [x] Expanded input validation rules: no leading `.`/`_`, 214-char max, PyPI normalization warning
- [x] Fixed shadcn/ui installation: CLI-based (`npx shadcn@latest init/add`), not npm package
- [x] Fixed Tailwind v4 config: CSS-based, removed `tailwind.config.ts` from project structure
- [x] Standardized suggestion count to 8 (aligned prompt and AC)
- [x] Removed redundant "Enter to search" step (native `<form>` behavior)
- [x] Added API error response schema for all error types
- [x] Added per-registry fetch timeout (4s) to enforce 5s p95 acceptance criteria
- [x] Removed unused `NameContext.type` field (UI doesn't collect it)
- [x] Added verification steps for: invalid input rejection, GitHub batch query, PyPI normalization, race condition, Homebrew Cask
- [x] Added implementation step references to risk mitigations
