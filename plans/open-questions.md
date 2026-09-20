# Open Questions

## Deployment Epic (`.omc/plans/deployment-epic.md`) — 2026-09-05

### Blocked on Andrew (gate execution)
- [ ] **Which hosting provider and plan tier?** (Plan Step 1, ADR-1) — Recommendation: Vercel/Hobby. Creates a billing relationship and account identity, so it cannot be an agent decision. Blocks Steps 9–16, and Step 5 needs the tier to size timeouts against.
- [ ] **Public exposure posture for AI suggestions?** (Plan Step 2) — Option W (public + rate-limited + spend-capped, recommended), Option X (registry checks only, zero financial exposure), or Option Y (private). Risk-tolerance judgment about Andrew's own money. Determines whether Steps 3 and 11 exist at all.
- [ ] **Provider-side spend caps: what amount?** (Plan Step 11) — Should be an amount Andrew would accept losing outright. This is the real backstop against abuse; the in-app rate limiter counts per instance and does not bound the aggregate.
- [ ] **Custom domain — yes or no?** (Plan Step 14) — Recommendation: skip; use the host-provided URL. Adds registrar cost, DNS records, and a renewal to remember.
- [ ] **How should preview deployments of a PUBLIC repo handle secrets?** (Plan Step 10) — Fork PRs could otherwise obtain a deployment carrying real API keys. Needs a conscious decision: no AI keys in Preview scope, separate low-limit keys, or previews disabled.

### To verify at execution time (do not assume from memory)
- [ ] **The chosen host's current function duration limit for the chosen tier** (Plan Step 5) — these limits have changed repeatedly across plan tiers; must be read from the host's live docs before sizing app-side timeouts.
- [ ] **Whether `GET /api/providers` is statically prerendered by default in Next 16.2.6 with `cacheComponents` off** (Plan Step 6) — the plan makes it explicitly dynamic regardless, but the actual default should be confirmed from the build output.

### Deferred / adjacent (not blockers for this epic)
- [ ] **Should the client tolerate non-JSON error responses defensively?** (Pre-mortem scenario 3) — If a platform-level 504 ever reaches the browser as HTML, the client's JSON parse throws and the UI shows a generic crash. UI work adjacent to this epic; consider filing as its own bead.
- [ ] **When does a shared cache (Redis/Upstash/KV) become justified?** (ADR-2 follow-up) — Explicit trigger conditions recorded: (a) GitHub `rate_limited` statuses visible to users in normal single-user operation, or (b) upstream registry call volume becoming a cost or politeness concern. Requires the `rate_limited` logging from the plan's Observability section to be decidable on evidence.
- [ ] **Should CI gain a deploy gate?** (Plan Step 16) — CI currently runs only `pnpm lint` + `pnpm build` with no test step, so a gate would be weak today. Properly the work of beads `application-namer-ef1`, `-l35`, `-upf`.
- [ ] **Should `GITHUB_TOKEN` be reclassified as required?** — ADR-2 makes it effectively required in production (it triples the GitHub rate budget, which matters more once the cache is fragmented across instances), but `.env.example` still documents it as optional. Plan Step 16 updates the doc; flagging in case the code should also warn when it is unset in production.

## Testing Epic (`.omc/plans/testing-epic.md`) — 2026-09-05

### Decisions needed before/while executing
- [ ] **Are the proposed coverage thresholds the right initial bar?** (Plan Step 14, Decision 8) — 85/80/90/85 statements/branches/functions/lines over `src/lib/**` + `src/app/api/**`, with 100% required on `validation.ts`. Set too high they cause chronic CI churn and filler tests; too low and the gate is decorative. The plan uses a measure-then-set calibration rule, but the starting numbers are a judgment call.
- [ ] **Confirm component and hook tests are out of scope for this epic.** (Plan scope section) — `src/components/**` and `src/hooks/use-name-check.ts` are deferred so the suite can stay `environment: 'node'` and thresholds can be scoped tightly. If they belong in this epic instead, the tooling decision changes (adds jsdom, `@vitejs/plugin-react`, Testing Library, and a second Vitest project).

### Implementation/spec mismatches — characterize now, decide later
- [ ] **Malformed JSON request bodies produce an uncaught throw, not a clean 400.** (Plan Steps 12, 13) — Neither `/api/check` nor `/api/suggest` wraps `await request.json()` in a try/catch. Tests will pin whatever Next.js currently does. Matters because the v1 plan's API error schema implies every failure has a structured JSON body; file a bug bead?
- [ ] **MCP bridge default ports disagree between plan and code.** (Plan Step 10) — `application-namer-v1.md` AC #9 documents `8940/8941/8945`; `mcp-bridge.ts` and `.env.example` both use `8960/8961/8962`. Tests will pin the code's values. Which is canonical?
- [ ] **Cache eviction is FIFO-by-insertion, not LRU.** (Plan Step 4) — `application-namer-v1.md` specifies "LRU eviction"; `cache.ts` never refreshes recency on read, so a hot entry can still be evicted first. Tests will pin the implemented FIFO behavior. Intended semantics, or a bug worth its own bead?
- [ ] **`checkGitHub` may report a description from the wrong repo.** (Plan Step 7) — It sources `description`/`owner`/`stars` from `items[0]` (top-starred result overall), which is not necessarily the exact-name match that determined the `taken` verdict. `checkGitHubBatch` does this correctly, using the top-starred *matching* item. Characterize as-is, or file a bug?
- [ ] **`SuggestResponse.errors` is declared but never populated.** (Plan Step 13) — `types.ts` declares it; `/api/suggest` never sets it, so provider errors surface only as a 503 and partial-failure information is lost. Should tests assert its absence, or should the field be removed / actually populated?
- [ ] **Two different PyPI-normalization warning strings exist.** (Plan Steps 3, 8) — `validation.ts` and `registries/index.ts` emit different wording for the same concept, and `/api/check` surfaces the `index.ts` one. Tests will pin both separately. Consolidate into one shared message?

### Known coverage gap accepted by this plan
- [ ] **The hard-coded MCP bridge timeouts (2 s init / 5 s init / 60 s tool call) are not directly asserted.** (Plan Step 10, Risks) — Unlike `registryFetch`, `mcp-bridge.ts` takes no timeout parameter, so proving them would require waiting them out or patching `AbortSignal.timeout`. Left uncovered and documented. Worth adding a timeout-injection seam later?
