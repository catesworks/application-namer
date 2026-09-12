# Deployment & Release — Implementation Plan (v2 — post-consensus revision)

**Driving bead:** `application-namer-dzy` — "Decide on and execute deployment (Vercel or other)"
**Parent epic:** `application-namer-aqq` — Deployment & Release Planning
**Status:** PLAN ONLY — nothing in this document has been executed. Steps 1, 2, 11, 12, 13 and 16 are **blocked on Andrew** and must be resolved before any deploy happens.

---

## Requirements Summary

Take `application-namer` from local-dev-only (`pnpm dev`) to a running, publicly reachable deployment, and establish the operational basics that a deployment implies: secrets provisioning, a defensible caching story for the target runtime, a domain, and a rollback path.

The v1 plan (`application-namer-v1.md`) declared the architecture "Vercel-compatible" but deliberately deferred the hosting decision. Its ADR left two explicit follow-ups that this plan must now close:

> "If deployed publicly, add server-side rate limiting per IP. Consider Redis cache if multiple server instances are needed."

Both are load-bearing here, not optional polish. See Decision Drivers.

### Current state (verified 2026-09-05)

| Fact | Value | Source |
|---|---|---|
| Framework | **Next.js 16.2.6**, React 19.2.4 | `package.json` |
| `cacheComponents` | **not enabled** (empty `next.config.ts`) | `next.config.ts` |
| `output: 'standalone'` | **absent** | `next.config.ts` |
| Node | 22 (`.nvmrc`), no `engines` field | `.nvmrc`, `package.json` |
| Package manager | pnpm 10.33.0 (`packageManager` field set) | `package.json` |
| Repo | `catesandrew/application-namer` — **PUBLIC** on GitHub | `gh repo view` |
| CI | lint + build on push/PR to `main`. No test step, no deploy step | `.github/workflows/ci.yml` |
| Tests | **none exist** (tracked by `application-namer-ef1`, `-l35`) | repo scan |
| Route segment config | **none anywhere** — no `runtime`, `dynamic`, `revalidate`, `maxDuration`, `fetchCache` | `src/**` |
| Proxy/middleware layer | **none** — no `proxy.ts`, no `middleware.ts` | repo scan |
| Deployment config | none — no `vercel.json`, no Dockerfile, no `.vercel` | repo root |

**Note on framework version:** this is Next **16**, not 15. Per `AGENTS.md`, the vendored docs in `node_modules/next/dist/docs/` are authoritative and differ from training data. All Next.js claims in this plan are cited from those vendored docs. Two v16 specifics matter here:

- v16 introduced a first-class, provider-agnostic **Deployment Adapter API** (`01-app/03-api-reference/07-adapters/`). The only *verified* adapters today are **Vercel** and **Bun**, with Cloudflare and Netlify "working on" verified adapters.
- **Middleware was renamed to Proxy** (`03-file-conventions/proxy.md`; there is no `middleware.md`). Critically, and contrary to Next 15 behavior, that doc states: *"Proxy defaults to using the Node.js runtime. The `runtime` config option is not available in Proxy files. Setting the `runtime` config option in Proxy will throw an error."* This directly affects where rate limiting can live — see Step 3.

### Runtime surface to be deployed

Three route handlers, all returning buffered `NextResponse.json(...)` (no streaming):

| Route | Work performed | Bounded? |
|---|---|---|
| `POST /api/check` | 5 registries fan-out via `Promise.allSettled` | Yes — 4s per fetch (`registry-client.ts:5`), ~4-5s total |
| `GET /api/providers` | env-var checks + liveness probes of 3 MCP bridge servers | Yes — 2s × 3 |
| `POST /api/suggest` | 1 AI provider call, then `checkNames()` registry fan-out | **NO — worst case ~30 minutes. See Driver 2.** |

---

## Acceptance Criteria

1. A hosting provider is chosen, with the decision recorded in an ADR and **explicitly signed off by Andrew** before any account is created or any secret is entered into a third-party dashboard.
2. The production deployment serves the app over HTTPS at a stable URL.
3. `POST /api/check` works in production against all 5 registries.
4. `GET /api/providers` in production returns a **non-empty** provider list reflecting the secrets actually provisioned in the host, and does not advertise MCP bridge providers.
5. `POST /api/suggest` works in production and completes within the host's function duration limit, or fails with the app's own JSON error schema — never with a platform 504/HTML error page, **and never with a silent HTTP 200 carrying an empty suggestion list**.
6. No secret is present in the git history, in the deployed bundle, or in any client-visible asset.
7. Every upstream call made during a request is bounded in application code — by **both** an explicit timeout **and** an explicit retry cap — to a total strictly less than the host's function duration limit.
8. Public abuse of the AI suggestion path cannot silently produce an unbounded API bill. The maximum financial loss in a billing period is a **stated, finite number** (ADR-4), not merely "unlikely".
9. The caching strategy for the chosen runtime is decided and documented, including the explicit consequence of not changing it.
10. A rollback procedure is documented **and rehearsed at least once** against the real production deployment.
11. The deployment is reproducible from the repo: a new maintainer can read `README.md` plus this plan and redeploy without tribal knowledge.

---

## RALPLAN-DR Summary

Mode: **DELIBERATE** — this plan spends real money, provisions live credentials into a third-party dashboard, and exposes an API-key-backed endpoint to the public internet. Pre-mortem and expanded test plan are included below.

### Principles

1. **The human owns money and identity.** Any step that creates an account, selects a billing plan, or types a credential into a third-party dashboard is Andrew's, not an agent's. Agents prepare; Andrew provisions.
2. **Bound every upstream call in app code.** Correctness must not depend on a platform-specific timeout value that can change under us or differ per host.
3. **Deployment must not silently change behavior.** If a capability (MCP providers, cache hit rate) cannot work in production, it should be explicitly disabled and observable — not left to fail quietly on every request.
4. **Prefer reversible over optimal.** A one-click rollback matters more than shaving latency; pick the host that makes mistakes cheap.
5. **Don't pay for infrastructure to fix a problem you don't have.** Add a shared cache when a measured symptom demands it, not because serverless "usually" needs one.
6. **Bound the loss, don't just reduce the likelihood.** For anything touching Andrew's billing, the plan must state a finite worst-case number. A mitigation that only makes abuse *less likely* is not a bound.

### Decision Drivers

1. **Exposure of paid API keys to the public internet.** `POST /api/suggest` calls Anthropic/OpenAI on Andrew's keys with no authentication and no rate limiting. Publishing the current code as-is is a direct, uncapped financial liability. This is the single largest driver and it dominates the hosting choice, which is comparatively low-stakes.

2. **Unbounded request duration on `/api/suggest` — worst case is roughly 30 minutes, not 65 seconds.** The v1 revision of this plan understated this by ~30×; the corrected analysis:
   - `claude.ts:23` and `openai.ts:23` construct their SDK clients with **only** `apiKey`. Neither `timeout` nor `maxRetries` is set, so both inherit the SDK defaults — a **10-minute** request timeout and **2 automatic retries** (3 attempts total). Worst case per provider call is therefore ≈ **3 × 10 min = 30 min**, entirely inside a single `/api/suggest` invocation, before `checkNames()` even starts.
   - The MCP path is separately bounded at ~65s (`mcp-bridge.ts:97` 5s init + `:118` 60s tool call), which is the *smaller* problem and is removed from production entirely by Step 5.
   - Consequence: **Step 6 must set both `timeout` and `maxRetries`.** Setting `timeout` alone leaves 3× the intended bound, because retries are counted per attempt.

3. **Timeouts currently fail silently as HTTP 200.** `suggest/route.ts:21-28` does wrap `generateSuggestions()` in a try/catch that returns a correct 503 — but that catch is unreachable for this class of error, because `claude.ts:50-57` and `openai.ts:39-46` re-throw **only** `AuthenticationError` and `return []` for everything else. A timeout, an abort, a 429, or a 529 therefore becomes an empty array, which flows to `suggest/route.ts:41-42` as a **200 with `{suggestions: []}`**. Bounding the calls (Driver 2) is necessary but not sufficient: without error classification, a faster timeout just produces the silent-success failure sooner. Step 7 exists for this reason.

4. **Operational surface Andrew has to own forever.** This is a personal side project. Every piece of infrastructure introduced (a Dockerfile, a Redis instance, a TLS cert, a VPS to patch) is a permanent maintenance tax on a tool whose purpose is to check whether a name is free.

### Viable Options

Two decision axes are genuinely independent and should be decided separately. Conflating them is why this decision has stalled.

#### Axis 1 — Hosting provider

**Option A: Vercel — RECOMMENDED DEFAULT**
- *Pros:* Zero deployment config for Next.js 16; **Vercel is one of only two verified Deployment Adapters** in the vendored v16 docs (`07-adapters/index.md`), so framework/host drift is minimal. Git-integrated preview deploys per PR. One-click instant rollback to any prior deployment. Free tier exists for non-commercial use. No infrastructure to own — no Dockerfile, no TLS, no patching. Matches the v1 ADR's stated intent.
- *Cons:* Serverless → the in-memory cache's cross-request tier is per-instance and lost on cold start (see ADR-2). Function duration is capped by plan tier and must be verified (Step 6). Free-tier terms restrict commercial use. Some vendor lock-in, materially reduced by the Adapter API being provider-agnostic.

**Option B: Fly.io (Docker, long-lived Node process)**
- *Pros:* A single long-lived process means the cache's cross-request tier behaves as it does in local dev — no fragmentation, no cold-start loss. No function duration limit, so Driver 2 becomes a UX concern rather than a truncation one. Portable (plain Docker).
- *Cons:* Requires adding `output: 'standalone'` plus a Dockerfile — and per the vendored docs, standalone output "does not copy the `public` or `.next/static` folders by default," so the build must manually `cp -r public .next/standalone/ && cp -r .next/static .next/standalone/.next/`. That is a real, easy-to-get-wrong build step this repo does not have. Machines sleep and cold-start. Ongoing cost. Rollback is CLI-driven, not one-click.

**Option C: Render (Web Service)**
- *Pros:* Long-lived process like Option B (same cache characteristics), git-integrated, simple dashboard.
- *Cons:* Free tier spins down after idle, producing a long (tens of seconds) cold start on the first request of a session — poor for a tool used in short bursts, which is exactly this app's usage pattern. Not a verified adapter.

**Option D: Self-hosted Node (VPS / homelab)**
- *Pros:* Total control; no per-request billing.
- *Cons:* Andrew personally owns TLS renewal, OS patching, uptime, reverse proxy config (the vendored self-hosting doc recommends nginx in front, plus `X-Accel-Buffering: no` if streaming is ever added), and process supervision — for a name-checking tool. Highest ongoing tax, directly against Principle 5. Rollback is bespoke.

> **Correction on documentation status (v2).** An earlier draft of this plan claimed Render was "listed in the vendored Next.js deploy docs as a supported provider." A reviewer then asserted the opposite — that neither Render nor Fly.io appears in the vendored docs at all. **Both statements are wrong.** The vendored `01-app/01-getting-started/17-deploying.md` lists, at lines 58-62, under the **Docker** section and introduced by "Additionally, hosting providers offer guidance on deploying Next.js": DigitalOcean, **Fly.io** (`:59`), Google Cloud Run, **Render** (`:61`), and SST. These are external guidance links, **not** endorsements or verified adapters, and their placement under *Docker* means the vendored docs do not describe a Dockerfile-free path for either. Option C's "no Dockerfile required" claim rests on Render's own native-Node runtime support, not on anything in the Next.js docs, and should be verified against Render's live documentation if Option C is selected.

**Recommendation: Option A (Vercel).** Drivers 1 and 4 dominate. The cache fragmentation that Options B/C appear to solve is, per the corrected ADR-2, confined to an optimization tier with no correctness dependency — so B and C pay real ongoing operational cost for a non-problem, and Vercel's instant rollback directly serves Principle 4.

*No option is invalidated here — all four remain viable, which is precisely why this needs Andrew's sign-off rather than an agent's judgment.*

#### Axis 2 — Public exposure posture for AI suggestions

Independent of the host. Must be decided even if Vercel is chosen instantly.

**Option W: Public app, AI suggestions enabled, rate-limited + spend-capped — RECOMMENDED**
- *Pros:* Ships the full product. Worst-case loss becomes a stated, finite number (ADR-4).
- *Cons:* Requires Steps 3, 4 and 13 before launch. Residual risk is nonzero and is bounded by the spend cap, not by the rate limiter — see ADR-4.

**Option X: Public app, registry checks only; AI suggestions disabled in production**
- *Pros:* Reduces financial risk to **zero** — no keys are provisioned in the host, so Steps 3, 4 and 13 become unnecessary and the critical path shortens considerably. The registry-checking half is genuinely the useful half.
- *Cons:* Ships a visibly degraded product; the suggest button is permanently disabled for every public visitor.

**Option Y: Private deployment (host access protection / Tailscale)**
- *Pros:* Full product, zero public abuse surface.
- *Cons:* Defeats most of the point of deploying — if only Andrew can reach it, `pnpm dev` already works. Justifiable only as a temporary staging posture.

**Recommendation: Option W**, with Option X as a fully respectable fallback if Steps 3-4 prove more work than Andrew wants to fund now. Option X can be upgraded to W later; it is not a failure mode.

---

### ADR-1: Hosting Provider — Vercel (PROPOSED, awaiting Andrew's sign-off)

- **Status:** **PROPOSED — NOT ACCEPTED.** Requires Andrew's explicit approval at Step 1. No account creation, plan selection, or secret entry may occur before that approval.
- **Decision:** Deploy to Vercel via GitHub integration on `catesandrew/application-namer`, free/Hobby tier, using the host-provided `*.vercel.app` domain initially.
- **Drivers:** (1) exposure of paid API keys — addressed host-independently by ADR-3/ADR-4; (2) unbounded `/api/suggest` duration — addressed in app code at Steps 6-7, not by host choice; (4) minimizing permanently-owned operational surface.
- **Alternatives considered:** (B) Fly.io — rejected for v1: requires `output: 'standalone'` plus a Dockerfile with a manual static-asset copy step, to solve a cache-fragmentation problem that ADR-2 shows is confined to an optimization tier. (C) Render — rejected on free-tier idle spin-down, which penalizes this app's burst usage pattern. (D) Self-hosted — rejected on ongoing maintenance burden per Principle 5.
- **Why chosen:** Verified Deployment Adapter (minimal framework/host drift), no deployment config needed in this repo, per-PR preview deploys, one-click rollback (Principle 4). Honors the v1 plan's "Vercel-compatible" intent, so no architectural rework is implied.
- **Consequences:** The cross-request tier of `src/lib/registries/cache.ts` becomes per-instance and cold-start-ephemeral (ADR-2). Function duration is capped by plan tier, forcing Step 6. Free-tier terms prohibit commercial use, constraining any future monetization. Preview deployments of a **public** repo need their secret scope explicitly verified (Step 12).
- **Follow-ups:** Revisit if the app needs commercial-tier terms, sustained sub-100ms cached responses, or if GitHub rate limiting becomes user-visible (ADR-2 trigger conditions).

### ADR-2: Keep the In-Memory Cache Unchanged on Serverless (ACCEPTED — technical, no sign-off needed)

> **Revised in v2.** The v1 rationale claimed the cache was "purely an optimization with no correctness dependency." **That was wrong**, and the error is worth recording because it nearly justified a bad alternative. The decision survives; the reasoning is replaced.

- **Status:** Accepted, conditional on ADR-1 resolving to a serverless host. If Andrew picks Option B or C, this ADR is largely moot.
- **Decision:** Deploy `src/lib/registries/cache.ts` **as-is**. Do not introduce Redis, Upstash, Vercel KV, or a Next.js custom `cacheHandler` as part of this deployment.
- **Drivers:** Principle 5; the measured behavior of the cache in code; the actual downstream consequence of a cache miss.

- **The corrected reasoning.** The cache has **two distinct tiers of use**, and they have different criticality:

  1. **Cross-request tier (a true optimization).** In `checkName` (`registries/index.ts:45-51`), `fetchWithCache` treats a miss as "go fetch it." No correctness dependency. Fragmenting this across instances cannot produce a wrong answer — only more upstream calls.

  2. **Intra-request tier (load-bearing state, NOT an optimization).** In `checkNames` (`registries/index.ts:87-124`), the GitHub batch result is written into the cache at `:108-110` and then read back out at `:123-124` as the **only channel** by which it reaches the response — with `errorResult(name, "GitHub batch failed")` as the fallback when the read misses. The cache is functioning as an in-process data bus here, not a cache.

  This distinction is what saves the decision. The intra-request tier is written and read **within a single invocation on a single instance**, so it is completely unaffected by cross-instance fragmentation or cold starts. Only the optimization tier degrades on serverless. Therefore: no shared cache is needed for correctness.

  What *does* degrade is the optimization tier's hit rate, whose only real downstream consequence is more GitHub traffic — and GitHub's search API is the tightest limit (10/min unauthenticated, 30/min authenticated). Note the counterintuitive part: the authenticated budget is **per-token and shared across all instances**, so scaling out makes token burn worse and a shared cache would help. But the far cheaper fix for the same symptom is (a) always set `GITHUB_TOKEN` in production, tripling the budget, and (b) rate-limit `/api/suggest`, the only path that fans out to many names at once. Both are already required for other reasons (Steps 3, 12). Redis buys a marginal further improvement in exchange for a permanent new service, credential, failure mode, and latency hop.

  The vendored docs separately confirm Next's *own* Data/Full Route Cache "is not shared across instances," offering `cacheHandler` + `cacheMaxMemorySize: 0` as the remedy. Not adopted: these are dynamic POST handlers with no ISR/revalidation semantics to keep consistent — wrong layer.

- **Alternatives considered:**
  - *(i) Upstash Redis / Vercel KV shared cache* — rejected as premature. Adds a service, a credential, a failure mode, and latency to solve an unmeasured problem.
  - *(ii) Custom Next.js `cacheHandler` with `cacheMaxMemorySize: 0`* — rejected as addressing the framework cache, not this app's registry cache. Wrong layer.
  - *(iii) Delete `cache.ts` entirely, on the theory it is ineffective on serverless* — **rejected because it would break correctness outright.** Removing it would sever the intra-request data channel described above, causing every `/api/suggest` response to report GitHub as `"GitHub batch failed"` for every suggestion. *(v1 of this plan rejected this alternative for the wrong reason — "warm containers still yield real hits" — which understated the consequence from "breaks the product" to "slightly less efficient." Corrected here.)*

- **Consequences:** Cross-request hit rate in production will be materially lower than in local dev and will vary with instance scaling and cold starts. Expect more upstream registry traffic per session than local testing suggests. `GITHUB_TOKEN` graduates from "optional" (as `.env.example` currently describes it) to **effectively required in production**. (An earlier draft flagged the intra-request GitHub data channel itself as a latent eviction fragility — R-9 — but that was refuted by simulation; see Changelog. The data channel is sound, it just must not be deleted, per alternative (iii) above.)
- **Follow-ups / explicit trigger conditions for revisiting:** Introduce a shared cache if *either* (a) GitHub `rate_limited` statuses become visible to users in normal single-user operation, or (b) upstream registry call volume becomes a cost or politeness concern. Until one is observed, do nothing.

### ADR-3: Bound Upstream Calls in Application Code, Not via Platform Limits (ACCEPTED — technical, no sign-off needed)

- **Status:** Accepted. Host-independent — applies under every Axis-1 option.
- **Decision:** Add an explicit **timeout** *and* an explicit **retry cap** to every upstream call made during a request, sized so worst-case total duration is comfortably below the host's function limit. Additionally, classify and propagate the resulting errors (Step 7) so they surface as the documented schema. Do not rely on the platform's timeout as the bounding mechanism.
- **Drivers:** Drivers 2 and 3; Principle 2.
- **Why chosen:** When the platform enforces the limit, the client receives a platform-generated 504 (typically HTML), not the app's JSON error schema — so the failure surfaces as a JSON parse error in the browser rather than a handled error state. Bounding in app code means the app always gets to write its own response, and behaves identically in local dev where no platform limit exists. **Timeout alone is insufficient:** with SDK defaults of 2 retries, a 20s timeout still yields a 60s worst case, so `maxRetries` must be set explicitly alongside it.
- **Alternatives considered:** Raise `maxDuration` to exceed the ~30min worst case — rejected as impossible on any realistic tier and a terrible user experience regardless; nobody waits 30 minutes for name suggestions.
- **Consequences:** Some very slow provider responses that would eventually have succeeded will now be cut off and reported as errors. Intended trade. Reducing `maxRetries` also removes automatic recovery from transient 429/529 responses, which will make transient provider errors more user-visible — acceptable, and preferable to invisible 30-minute hangs.
- **Follow-ups:** If a legitimate provider call regularly approaches the bound, revisit by streaming rather than by raising the timeout.

### ADR-4: Rate-Limiter Shared State — Accept Per-Instance Counting, Bound the Loss with a Spend Cap (ACCEPTED — technical; depends on Axis 2 = Option W)

- **Status:** Accepted if Andrew selects Option W at Step 2. Void under Option X (no keys deployed, nothing to bound).
- **Context:** The rate limiter in Step 3 keeps its counters in module-level memory, exactly like `cache.ts`. On a serverless host these counters are **per-instance**, so N concurrent instances permit roughly N × the configured rate. A limiter whose state is not shared cannot, by itself, enforce a global rate.
- **Decision:** Accept per-instance counting. Do **not** introduce shared state (Redis/Upstash/KV) for rate limiting. Instead, treat the **provider-side hard spend cap (Step 13) as the actual bound**, and state the resulting loss ceiling explicitly.
- **Drivers:** Principle 5 (don't add infrastructure for an unmeasured problem); Principle 6 (bound the loss, don't just reduce likelihood); consistency with ADR-2 — it would be incoherent to reject Redis for caching and then require it for rate limiting on the same traffic.
- **The loss ceiling, stated explicitly:** The per-IP limiter reduces *casual* abuse (a single client looping the endpoint) to the configured rate. It does **not** bound a distributed or IP-rotating attacker, whose effective ceiling is instead:

  > **Maximum loss in a billing period = the hard monthly spend cap set on each provisioned key at Step 13.**

  This is the number Andrew is actually accepting when he chooses Option W. It should be set to an amount he would be willing to lose outright, because in the worst case he will. It is a finite, chosen number rather than an open-ended exposure — which is the entire point of the decision.
- **Alternatives considered:**
  - *Shared-state limiter (Upstash/Redis/KV)* — rejected for v1: adds a service and a credential to enforce a global rate that the spend cap already bounds financially. Reconsider if abuse actually occurs, at which point real traffic data will inform the limits.
  - *Option X (deploy no AI keys)* — **not rejected.** This is the strictly safer choice and remains on the table at Step 2. It sets the loss ceiling to exactly zero and makes this ADR unnecessary. If Andrew is uncomfortable naming a number he'd accept losing, that discomfort is itself the signal to pick Option X.
- **Consequences:** A determined distributed attacker can burn up to the monthly cap before spend halts. When the cap trips, AI suggestions stop working for legitimate users until the next period or a manual raise — a availability-for-cost trade, chosen deliberately. Step 13's alerting is what makes this observable rather than merely survivable.
- **Follow-ups:** If a spend cap is ever actually hit, that is the trigger to add either a shared-state limiter or authentication — not before.

---

## Implementation Steps

Dependency notation: **`⟵ needs: N`** means the step cannot start until step N is complete. Steps with no `needs` can start immediately and in parallel.

### Phase 0 — Decisions (BLOCKED ON ANDREW)

**1. 🔴 BLOCKED-ON-USER — Andrew chooses the hosting provider and plan tier.**
*⟵ needs: nothing. Blocks: 6, 11-19.*

Present Axis 1 (Options A-D) with the ADR-1 recommendation of Vercel/Hobby. Andrew confirms provider **and** tier, since tier determines the function duration limit that Step 6 must size against.

- **Acceptance criteria:** ADR-1's Status is updated from PROPOSED to ACCEPTED (or replaced with the chosen alternative and its rationale) in this file, naming provider and tier. No account created and no credential entered anywhere prior to this.
- **Why this is Andrew's:** creates a billing relationship and an account identity.

---

**2. 🔴 BLOCKED-ON-USER — Andrew chooses the public exposure posture for AI suggestions, and names the loss ceiling.**
*⟵ needs: nothing (independent of Step 1). Blocks: 3, 4, 12, 13.*

Present Axis 2 (W/X/Y) with the recommendation of Option W. Present ADR-4's loss ceiling honestly: under Option W the bound is the monthly spend cap, not the rate limiter. Ask Andrew directly for the number he would accept losing.

- **Acceptance criteria:** A recorded decision of W, X, or Y in this file. **If W:** the specific dollar figure for Step 13's cap is recorded, and ADR-4 is marked ACCEPTED. **If X:** Steps 3, 4 and 13 are struck, ADR-4 is marked VOID, and Step 12 provisions **no** AI provider keys. **If Y:** Step 14 additionally enables host-level access protection.
- **Why this is Andrew's:** it is a risk-tolerance judgment about his own money, and ADR-4 deliberately requires him to name the number rather than letting an agent infer it.

---

### Phase 1 — Pre-deploy hardening (host-agnostic except Step 6; may start immediately)

**3. Add per-IP rate limiting to `POST /api/suggest` and `POST /api/check`.**
*⟵ needs: 2 (Option W only; struck under Option X). Implements ADR-4.*

Closes the v1 ADR's explicit follow-up. `/api/suggest` is the priority — it is the path that spends money. Three details determine whether this is a real control or a decorative one:

   1. **Client IP source — do not trust a raw header.** `x-forwarded-for` is attacker-controlled: a client can send `X-Forwarded-For: <random>` on every request and get a fresh bucket, making a naive limiter a no-op. Use the host's own trusted client-IP mechanism (on Vercel, the platform-populated header documented in its current docs), or if reading a forwarded chain, take the entry the platform appends — **never** the client-supplied leftmost value. Verify against the chosen host's live documentation at implementation time.
   2. **Placement — in the route handlers, not `proxy.ts`.** This repo has no proxy/middleware layer today. In Next 16, `proxy.ts` (renamed from `middleware.ts`) *does* default to the Node.js runtime per the vendored `proxy.md`, so an in-memory counter there is not immediately disqualified as it would have been under Next 15's Edge default. However, proxy and route handlers are not guaranteed to share a module instance or process, so counters set in one may be invisible to the other. Put the limiter in a shared module imported directly by the route handlers; introducing a `proxy.ts` solely for this is unnecessary surface.
   3. **Cap the limiter's own memory.** The counter map is unbounded by default and keyed by client IP, which is attacker-controlled cardinality — an IP-rotating attacker turns the limiter itself into a memory-exhaustion vector. Bound it with a max entry count and sweep expired windows on write, mirroring `cache.ts`'s `MAX_ENTRIES` approach.

- **Acceptance criteria:** Exceeding the limit returns HTTP **429** with the v1 error schema (`{ error, retryAfter }` — see `application-namer-v1.md` § API Error Response Schema), not a crash or a 500. Under the limit, behavior is byte-identical to today. Limits configurable by env var with a safe default. **Verified by three explicit tests:** (a) N+1 rapid requests → 429; (b) N+1 requests each carrying a *different* spoofed `x-forwarded-for` → still 429, proving the IP source is not client-controlled; (c) many distinct client IPs → limiter memory stays bounded.
- **Explicitly does NOT achieve:** a global rate limit. See ADR-4 — the financial bound is Step 13.

---

**4. Bound per-request cost and harden the `/api/suggest` request body.**
*⟵ needs: 2 (Option W only). Independent of Step 3 — different failure class.*

Rate limiting bounds *how many* requests arrive; this step bounds *how expensive one request can be*. Currently a single well-formed request can be arbitrarily costly, because `suggest/route.ts:8-13` reads the body and casts it with `as` — performing **zero** runtime validation of `provider` or `context`:

   1. **Guard the body parse.** `await request.json()` at `:8` is unguarded; a malformed body throws before any handler logic, producing an unhandled 500 outside the documented error schema. Wrap it and return 400.
   2. **Validate `context.takenOn`.** It is consumed as `context.takenOn.join(", ")` in `claude.ts:9` and `openai.ts:9` and interpolated directly into the prompt. Today it can be *absent* (→ `TypeError` on `.join`, unhandled 500) or *enormous* (→ a multi-megabyte prompt billed as input tokens on Andrew's key, from one request that never trips the rate limiter). Validate: is an array, bounded element count, bounded element length, elements are known registry IDs.
   3. **Validate `provider`.** Unknown values already throw at `suggestions/index.ts:56`, which the route converts to a 503; a 400 is the correct code for a bad client-supplied value. Low severity, but trivial to fix while here.
   4. **Cap OpenAI output tokens.** `claude.ts:29` sets `max_tokens: 1024`, but `openai.ts:27-38` sets **no** output cap — an asymmetry with no apparent rationale, leaving output length bounded only by the model default. Set an explicit cap.
   5. **Bound overall body size**, so an oversized payload is rejected before parsing rather than after.

- **Acceptance criteria:** A request with a missing, non-array, or oversized `context.takenOn` returns **400** with the documented schema — never a 500 and never an upstream AI call. A malformed JSON body returns 400. An unknown `provider` returns 400. The OpenAI call sends an explicit output cap. Maximum billable tokens per `/api/suggest` request is computable from the source and recorded in the PR description.
- **Why this is separate from Step 3:** a rate limiter set to N requests/minute provides no protection whatsoever if one request can cost 1000× a normal one.

---

**5. Gate the MCP bridge providers behind an explicit env flag, defaulting to off.**
*⟵ needs: nothing.*

`MCP_CLAUDE_URL`/`MCP_CODEX_URL`/`MCP_COPILOT_URL` default to `localhost:8960-8962` (`mcp-bridge.ts:195-207`). In **any** hosted environment those are unreachable, so `getAvailableProviders()` (`suggestions/index.ts:11-16`) burns up to 2s × 3 on liveness probes guaranteed to fail, on **every** call to `GET /api/providers` — i.e. on every page load. Pure latency for a guaranteed-negative result.

Add e.g. `ENABLE_MCP_PROVIDERS` (default off unless explicitly set), skipping the probes entirely when off, and reject `mcp-*` provider IDs in `generateSuggestions` when disabled.

- **Acceptance criteria:** With the flag off, `GET /api/providers` performs zero network probes and returns in single-digit milliseconds, listing only API-key-backed providers. A direct `POST /api/suggest` with `provider: "mcp-claude"` while disabled returns 400 rather than attempting a localhost connection. With the flag on (local dev), behavior is unchanged. `.env.example` documents the new variable.
- **Bonus:** removes the ~65s MCP path from production entirely, so only the SDK bound in Step 6 remains to manage.

---

**6. Set explicit `timeout` AND `maxRetries` on both AI SDK clients, plus `maxDuration` on the route.**
*⟵ needs: 1 (needs the chosen tier's duration limit to size against). Implements ADR-3.*

   1. `claude.ts:23` — `new Anthropic({ apiKey })` sets neither `timeout` nor `maxRetries`, inheriting a 10-minute timeout and 2 retries. Set both explicitly.
   2. `openai.ts:23` — `new OpenAI({ apiKey })`, same defaults, same fix.
   3. Optionally reduce `mcp-bridge.ts:118`'s `AbortSignal.timeout(60_000)`, though Step 5 already keeps that path out of production.
   4. Add `export const maxDuration = <n>` to `src/app/api/suggest/route.ts`. Per the vendored `route-segment-config/maxDuration.md`, the literal form is `export const maxDuration = 5`, and "Deployment platforms can use `maxDuration` from the Next.js build output to add specific execution limits."

   **Verify the chosen host's actual current duration limit for the chosen tier from the host's own live documentation** — do not assume a number from memory; these limits have changed repeatedly across tiers.

   **Sizing rule:** worst case ≈ `timeout × (1 + maxRetries)` **plus** the `checkNames()` fan-out (~4-5s), and that total must sit below the host limit with margin. Setting `timeout` while leaving `maxRetries` at its default silently triples the intended bound — the specific error this step exists to prevent.

- **Acceptance criteria:** Worst-case `/api/suggest` duration is computable from the source, is written into the PR description as an explicit `timeout × (1 + maxRetries) + fan-out` calculation, and is below the host limit with margin. A deliberately-slowed provider produces a bounded failure at the computed time — not at 10 minutes, and not at 30.

---

**7. Classify and propagate AI provider errors instead of swallowing them.**
*⟵ needs: 6 (bounding produces the errors this step must surface). Required for AC #5 — Step 6 alone cannot satisfy it.*

`claude.ts:50-57` and `openai.ts:39-46` re-throw **only** `AuthenticationError` and `return []` for every other error. Timeouts, aborts, connection failures, 429s and 529s therefore become an empty array, which `suggest/route.ts:36-42` renders as a **200 with `{suggestions: []}`** — indistinguishable to the client from "the AI had no ideas." The 503 path at `suggest/route.ts:23-28` is correct but unreachable for these cases.

Without this step, Step 6 changes a 30-minute silent success into a 20-second silent success. The user-visible bug is unchanged.

Distinguish at minimum: auth failure (503, existing behavior), timeout/abort (504 or 503 with a timeout message), rate limited / overloaded upstream (429 with `retryAfter` per the v1 schema), and genuine empty-but-successful responses (200, the only legitimate empty case). Malformed-response paths (`claude.ts:59-67`, `openai.ts:48-61`) may legitimately continue returning `[]`, but should be distinguishable in logs.

- **Acceptance criteria:** A stubbed provider that times out produces a non-200 response carrying the documented JSON error schema. A stubbed provider returning a valid-but-empty list still produces 200. The two cases are distinguishable by the client and in host logs. **Regression test for the exact reported defect:** no configuration of provider failure yields `200 {suggestions: []}` except a genuine empty success.

---

**8. Make `GET /api/providers` explicitly dynamic.**
*⟵ needs: nothing.*

This route's response is a pure function of runtime environment state (which keys are set, which bridges are alive) and must never be evaluated at build time — a build-time snapshot would freeze the provider list to the state in which no runtime secrets exist. Since `cacheComponents` is off, previous-model segment exports apply (vendored `caching-without-cache-components.md`): add `export const dynamic = 'force-dynamic'` (and/or `revalidate = 0`) explicitly rather than relying on a default.

Related, from the vendored env-var doc: only `NEXT_PUBLIC_`-prefixed variables are inlined into the client bundle at build time. All six of this app's variables are correctly server-only and read via `process.env` in server code. Confirm this remains true.

- **Acceptance criteria:** A production build followed by a runtime env-var change is reflected in `/api/providers` without a rebuild. `grep -r "NEXT_PUBLIC_" src/` returns nothing. Build output does not list `/api/providers` as statically prerendered.

---

**9. Verify no secret has ever been committed to this PUBLIC repository.**
*⟵ needs: nothing. Blocks: 14 (do not expose a deployment publicly until this passes).*

`.gitignore` contains `.env*` and `.env.example` is the only tracked env file (verified). But a local `.env` with real values exists on disk and the repo is public — confirm the history, not just the working tree.

- **Acceptance criteria:** A scan of full git history (`gitleaks`/`trufflehog`, or `git log -p --all` filtered for key patterns) reports no `sk-`, `sk-ant-`, `ghp_`, or `github_pat_` material. GitHub Secret Scanning alerts for the repo are clean. If anything is found: **rotate the affected key first**, then decide on history rewriting — and do not proceed to Step 14.

---

**10. Confirm a clean production build and record the baseline.**
*⟵ needs: 3, 4, 5, 6, 7, 8 (build what will actually ship).*

Run `pnpm install --frozen-lockfile && pnpm lint && pnpm build`, then `pnpm start`, and exercise all three routes against the production build (not `pnpm dev`).

- **Acceptance criteria:** Build succeeds with no errors and no new warnings versus `main`. `pnpm start` serves the app. All three routes return correct responses. The route list from the build output is captured in the PR description so Step 15's production behavior can be compared against a known-good baseline.
- **Note:** `output: 'standalone'` is deliberately **not** added under ADR-1/Option A — unnecessary on Vercel. It becomes required only under Option B, in which case add it **and** the `cp -r public .next/standalone/ && cp -r .next/static .next/standalone/.next/` post-build step that the vendored docs flag as not automatic.

---

### Phase 2 — Provisioning (BLOCKED ON ANDREW)

**11. 🔴 BLOCKED-ON-USER — Andrew creates the hosting account and connects the repository.**
*⟵ needs: 1. Blocks: 12, 14.*

For Vercel: sign in with GitHub, import `catesandrew/application-namer`, accept the auto-detected Next.js preset. Confirm build command (`pnpm build`), install command, and Node version — the repo pins `packageManager: pnpm@10.33.0` and `.nvmrc: 22`, and the host must honor both. **Do not trigger a production deployment yet** — Step 12 must land first so the first build already has its secrets.

- **Acceptance criteria:** Project exists in the host dashboard, linked to the GitHub repo, with correct framework/build/Node detection. No deployment promoted to production.
- **Why this is Andrew's:** account identity, OAuth grant to a public repo, plan selection.

---

**12. 🔴 BLOCKED-ON-USER — Andrew provisions environment variables in the host dashboard.**
*⟵ needs: 2, 11. Blocks: 14.*

Andrew enters values directly into the host dashboard. **Agents must never be given, asked for, or shown these values.** Full set is `.env.example`; scope per Step 2's decision:

| Variable | Production scope | Notes |
|---|---|---|
| `GITHUB_TOKEN` | **Required** | Promoted from "optional" to required by ADR-2 — triples the GitHub budget, which matters more once the optimization tier is fragmented. No scopes needed for public repos. |
| `ANTHROPIC_API_KEY` | Option W only | Omit entirely under Option X. |
| `OPENAI_API_KEY` | Option W only | Omit entirely under Option X. |
| `ENABLE_MCP_PROVIDERS` | **Do not set** | Step 5's flag stays off in production. |
| `MCP_*_URL` | **Do not set** | localhost — meaningless in production. |

Two things that are easy to get wrong:
   - **Scope the variables to Production explicitly.** Setting them only in Preview or Development scope is the single most common cause of a deployment that builds fine and then reports "no AI providers configured" forever (pre-mortem scenario 2).
   - **The repo is PUBLIC.** Verify how the host exposes env vars to preview deployments built from pull requests, especially fork PRs, before enabling previews with real keys. Prefer leaving AI keys out of Preview scope entirely, or issuing separate low-limit keys.

- **Acceptance criteria:** Variables present in Production scope. `MCP_*` and `ENABLE_MCP_PROVIDERS` absent. Preview-scope exposure consciously decided rather than defaulted. Andrew confirms completion; no agent has seen a value.

---

**13. 🔴 BLOCKED-ON-USER — Andrew sets provider-side hard spend caps and billing alerts.**
*⟵ needs: 2 (Option W only). Must complete before 14. **This is the actual bound from ADR-4, not a backstop.***

Set a hard monthly usage limit and an alert threshold in the Anthropic Console and the OpenAI platform dashboard, on the specific keys provisioned at Step 12, at the figure Andrew named in Step 2.

- **Acceptance criteria:** A hard monthly spend cap exists on each provisioned key, at the amount recorded in Step 2. Alert emails route somewhere Andrew actually reads. The cap figure is written into ADR-4 so the accepted loss ceiling is documented rather than remembered.
- **Why this is Andrew's:** billing configuration on his accounts.
- **Why this outranks Step 3:** per ADR-4, the Step 3 limiter counts per instance and does not bound aggregate spend. This cap is the only thing that makes the worst case a finite number. **Under Option W, deploying without it means deploying with an unbounded liability.**

---

### Phase 3 — Deploy and verify

**14. Deploy to a preview/staging URL and run the smoke checklist.**
*⟵ needs: 9, 10, 11, 12, 13.*

Deploy the hardened branch to a non-production URL first and run the Test Plan's smoke checklist in full. Under Option Y, enable host-level access protection now.

- **Acceptance criteria:** Every item in **Deploy smoke (manual)** passes against the preview URL. Specifically: `/api/providers` returns a **non-empty** list containing exactly the providers whose keys were provisioned and **no** MCP providers; `/api/check` returns correct results for a known-taken and a known-free name; `/api/suggest` completes end-to-end within the Step 6 bound.

---

**15. Promote to production.**
*⟵ needs: 14.*

Promote the verified preview deployment (Vercel: promote the same build artifact rather than rebuilding, so what was verified is what ships). Re-run the smoke checklist against the production URL — env-var scoping differs between Preview and Production, which is exactly the class of bug this catches.

- **Acceptance criteria:** Production URL serves the app over HTTPS. Full smoke checklist passes **against production specifically**, not merely preview. Deployment ID and commit SHA recorded on bead `application-namer-dzy`.

---

**16. 🟡 BLOCKED-ON-USER (optional) — Andrew decides on a custom domain.**
*⟵ needs: 15. Optional — the plan is complete without it.*

Default recommendation: **skip it.** Use the host-provided `*.vercel.app` URL. A custom domain adds registrar cost, DNS records, and a renewal to remember, for cosmetics. If wanted, Andrew supplies the domain, adds the host's DNS records at his registrar, and waits for propagation and certificate issuance.

- **Acceptance criteria (only if pursued):** Custom domain serves the app over HTTPS with a valid auto-renewing certificate; the host-provided URL redirects or remains a working alias; smoke checklist passes against the custom domain.
- **Why this is Andrew's:** domain ownership and registrar access.

---

### Phase 4 — Operability

**17. Document and rehearse the rollback procedure.**
*⟵ needs: 15.*

A rollback path that has never been executed is a hypothesis, not a procedure. Write it down, then actually run it once against production while nobody depends on the app.

Per host:
   - **Vercel:** promote the previous production deployment (dashboard "Instant Rollback", or `vercel rollback`). Near-instant, no rebuild. Follow with a `git revert` on `main` so repo state matches deployed state — otherwise the next push silently re-deploys the bad build.
   - **Fly.io:** `flyctl releases list` → `flyctl releases rollback` (or redeploy a previous image).
   - **Render:** redeploy a previous successful deploy from the dashboard's deploy history.

Also document the **secret-compromise rollback**, a different procedure: rotate the key at the provider, update it in the host dashboard, redeploy. Rolling back code does not un-leak a credential.

- **Acceptance criteria:** A "Deployment" section in `README.md` documents the host, required env vars, deploy trigger, and both rollback procedures. A rollback has been **executed once** against production and the app verified working afterward, with elapsed time recorded so future-Andrew knows what he's committing to under pressure.

---

**18. File the application bugs found during planning that this epic does not fix.**
*⟵ needs: nothing (can be done immediately). Does not block deployment.*

Planning surfaced defects that are genuine application bugs rather than deployment work. Several are fixed as a side effect of Steps 3-7; the remainder must not be lost. File the unaddressed ones as bug beads under the appropriate epic — see the **Application Bugs Found During Planning** section below. (B-7/R-9, the suspected cache-eviction/GitHub data-channel fragility, was investigated further during review and refuted by simulation — do not file it.)

- **Acceptance criteria:** Every item in that section is either (a) covered by a step in this plan, with the step named, or (b) filed as its own bead with a repro. Nothing is left only in this document.

---

**19. Close out: update docs, CI, and beads.**
*⟵ needs: 15, 17.*

- Update `README.md` with the live URL and the Deployment section from Step 17.
- Update `.env.example` to reflect ADR-2 (`GITHUB_TOKEN` required in production, not optional) and Step 5's new `ENABLE_MCP_PROVIDERS` flag.
- Record ADR-1's final accepted state, and ADR-4's accepted cap figure, in this file.
- Consider adding a deploy-gating check to `.github/workflows/ci.yml`. Note CI currently runs only `pnpm lint` and `pnpm build` with **no test step**, because no tests exist — a deploy gate is therefore weak today. Do not block deployment on this; it is properly the work of beads `application-namer-ef1`, `-l35`, `-upf`.
- Close `application-namer-dzy`; assess whether parent epic `application-namer-aqq` can close.

- **Acceptance criteria:** Docs reflect reality. `application-namer-dzy` closed with production URL and chosen provider recorded. Deferred work captured as beads rather than left in this document.

---

## Pre-Mortem

*It is three months from now and this deployment has gone badly. What happened?*

### Scenario 1 — The bill

The app gets linked somewhere. Someone notices `/api/suggest` is unauthenticated and points a script at it — or simply sends one request with a 5MB `context.takenOn`, billed as input tokens. Andrew finds out from a billing email, or from a declined card.

- **Warning signs:** unexplained traffic spike; provider usage climbing while the app has no real users.
- **Mitigations:** Step 3 (per-IP limiting — bounds casual abuse only), Step 4 (per-request cost bounding — closes the "one expensive request" variant), **Step 13 (hard spend cap — the actual bound, per ADR-4)**, Step 2 Option X (eliminates exposure entirely).
- **Residual risk, stated honestly:** a distributed or IP-rotating attacker defeats per-IP limiting. Per ADR-4 the true ceiling is the monthly cap. **This is why Step 13 is not optional under Option W** — without it the ceiling is unbounded.

### Scenario 2 — The silent half-deployment

The deploy succeeds. The page loads. Registry checks work. But "Suggest alternatives" is permanently greyed out with "No AI providers configured," because keys went into the Preview scope rather than Production — or because `/api/providers` was evaluated at build time when no runtime secrets existed. Nothing errors. Nothing logs. The app is half-broken, and because the disabled state is a *designed* state rather than an error state, it looks intentional.

- **Warning signs:** none by design — which is what makes this the most likely scenario in this plan.
- **Mitigations:** Step 8 (force dynamic so env is read at request time), Step 12 (explicit Production-scope warning), Steps 14 **and** 15 (assert a **non-empty** provider list against production specifically, since the two scopes differ).
- **Why it ranks high:** needs no attacker, no load, no bad luck — just one mis-scoped dropdown.

### Scenario 3 — The failure that reports success

A user picks Claude, the API is slow or rate-limited. Per Driver 3, `claude.ts:56` returns `[]`, the route returns **200 `{suggestions: []}`**, and the UI shows "no suggestions." The user concludes the feature is useless. No error is logged, no alert fires, nothing appears in host logs as a failure — the request was, as far as every metric is concerned, a success. Andrew cannot reproduce it because his own key isn't rate-limited.

- **Warning signs:** none — this is the worst property of this scenario. It is invisible in logs, metrics, and error tracking simultaneously.
- **Mitigations:** Step 7 (error classification — the *only* mitigation), Step 6 (bounds duration so the failure at least arrives quickly), Step 5 (removes the MCP variant).
- **Note:** this scenario is why Step 7 exists as a separate step with its own AC. Step 6 alone converts a 30-minute silent failure into a 20-second silent failure — faster, equally invisible. **If only one thing from Phase 1 ships, it should be Step 7, not Step 6.**

---

## Test Plan

This app has **no automated tests** (beads `application-namer-ef1`, `-l35`). This plan therefore **does not depend on a test suite that does not exist** — deployment verification is manual and checklist-driven. The unit/integration rows are specified so the test beads can absorb them; they are not blockers for deploying.

### Unit (deferred to `application-namer-ef1`)
- Rate limiter (Step 3): allows N in window, rejects N+1 with correct `retryAfter`; window resets; **spoofed `x-forwarded-for` does not create a new bucket**; limiter memory stays bounded under many distinct IPs.
- Body validation (Step 4): missing/non-array/oversized `context.takenOn` → 400, and **no upstream AI call is made**; malformed JSON → 400; unknown `provider` → 400.
- Provider gating (Step 5): with `ENABLE_MCP_PROVIDERS` unset, `getAvailableProviders()` makes no network calls and returns no MCP providers.
- Timeout/retry bounds (Step 6): a stubbed slow provider aborts at `timeout`, and total attempts equal `1 + maxRetries` — not the SDK default of 3.
- Error classification (Step 7): each of {auth failure, timeout, 429, malformed response, genuine empty} maps to its intended status code; **only** genuine empty yields 200.

### Integration (deferred to `application-namer-l35`)
- `GET /api/providers` reflects env-var state at **request** time, not build time — set a var after building and assert the response changes.
- `POST /api/suggest` with the AI provider stubbed to hang returns the app's JSON error schema within the Step 6 bound, with a **non-200** status.
- `POST /api/check` behavior unchanged by the rate limiter when under the limit.
- `checkNames()` returns real GitHub results for all suggestions (basic correctness of the batch data channel — not guarding against a known risk; see Changelog on R-9).

### Deploy smoke (manual — **required**, run at Step 14 against preview *and* Step 15 against production)
1. Homepage loads over HTTPS; no console errors; no mixed content.
2. `GET /api/providers` → **non-empty** list, matching exactly the keys provisioned at Step 12, containing **no** `mcp-*` provider. Returns in well under a second.
3. `POST /api/check` with `express` → taken on npm/PyPI/GitHub with version and download metadata.
4. `POST /api/check` with a known-free name → available across all five registries.
5. `POST /api/check` with `../etc/passwd` → **400**, no outbound registry request (v1 AC #6).
6. `POST /api/suggest` end-to-end → suggestions returned, each with per-registry results, within the Step 6 bound. **GitHub status is a real value, not `"GitHub batch failed"`** (basic correctness check).
7. Rate limiting (Option W): N+1 rapid requests → **429** with documented schema, not a 500 or HTML.
8. Body hardening (Option W): a request with an oversized `context.takenOn` → **400**, and provider usage dashboards show **no** corresponding API call.
9. Mobile viewport at 375px → no horizontal scroll (v1 AC #10).
10. Cold start: wait for the instance to go cold, repeat check 3 — confirm correct results, note latency delta. **A cold cache must never change correctness** (ADR-2's core claim; this check falsifies it if wrong).
11. View source / network tab → no API key material in any client-visible asset.

### Observability (minimum viable — this is a side project, not a service)
- **Host logs:** confirm function logs are visible and an app-thrown error appears with a usable stack trace.
- **Spend alerts:** Step 13's provider alerts are the primary "something is wrong" signal and the only thing that catches Scenario 1.
- **Rate-limit visibility:** log 429s so abuse is distinguishable from normal traffic.
- **Provider-failure visibility:** log classified provider errors from Step 7. Without this, Scenario 3 stays invisible even after being fixed.
- **GitHub rate-limit visibility:** log when a registry check returns `rate_limited` — the specific metric that triggers ADR-2's follow-up. Without it, the decision to skip a shared cache cannot be revisited on evidence.
- **Explicitly out of scope:** uptime monitoring, APM, error-tracking SaaS, dashboards. Add them if the app earns them.

---

## Risks and Mitigations

| ID | Risk | Likelihood | Impact | Mitigation | Step |
|---|---|---|---|---|---|
| R-1 | Public unauthenticated endpoint burns AI credits | **High** | **High** | Per-IP limit + per-request cost bound + **hard spend cap (the actual bound)** | 3, 4, 13, ADR-4 |
| R-2 | AI failures return silent `200 {suggestions:[]}` | **High** | **High** | Error classification | 7 |
| R-3 | Env vars mis-scoped → prod silently has no AI providers | **High** | Medium | Force dynamic; assert non-empty list against prod, not just preview | 8, 12, 14, 15 |
| R-4 | `/api/suggest` runs up to ~30 min (SDK 10-min timeout × 3 attempts) | **High** | Medium | Explicit `timeout` **and** `maxRetries`; `maxDuration` | 6 |
| R-5 | Rate limiter bypassed via spoofed `x-forwarded-for` | Medium | **High** | Use host-trusted client IP, never the client-supplied leftmost value | 3 |
| R-6 | Rate limiter itself becomes a memory-exhaustion vector | Medium | Medium | Bounded entry count + sweep on write | 3 |
| R-7 | Unvalidated `context.takenOn` → giant prompt or `TypeError` 500 | Medium | **High** | Runtime validation of body; explicit OpenAI output cap | 4 |
| R-8 | MCP probes waste ~6s per `/api/providers` call in prod | **High** | Low | `ENABLE_MCP_PROVIDERS`, default off | 5 |
| R-9 | ~~Cache eviction breaks the GitHub data channel~~ — **REFUTED, see Changelog.** critic-deploy simulated `cache.ts`'s exact eviction semantics: a saturated 1000-entry cache plus a normal 8-suggestion `checkNames` call produces 0 spurious failures (self-eviction needs >1000 GitHub entries in one call, which the app never generates), and the write/read span at `registries/index.ts:106-147` has no `await` between them, so no concurrent request can interleave on a single-threaded runtime. Not a real risk; no mitigation needed. | N/A | N/A | Refuted — no action | — |
| R-10 | Cache fragmentation → more GitHub calls → user-visible rate limiting | Medium | Low | `GITHUB_TOKEN` required in prod (3× budget); log `rate_limited` as ADR-2's revisit trigger | 12, ADR-2 |
| R-11 | Secret exposed via public-repo preview deploys / fork PRs | Low | **High** | Consciously decide preview-scope exposure; prefer no AI keys in Preview | 12 |
| R-12 | Secret already in git history (public repo) | Low | **High** | History scan before public exposure; rotate first if found | 9 |
| R-13 | Host build differs from local (Node/pnpm drift) | Low | Medium | `.nvmrc` 22 + `packageManager` pin honored by host; compare to baseline | 10, 11 |
| R-14 | Bad deploy with no practiced way back | Low | Medium | Documented **and rehearsed** rollback, elapsed time recorded | 17 |
| R-15 | Deploy gate is weak (CI has no tests) | **High** | Low | Accepted knowingly; deferred to the test-coverage epic | 19 |

---

## Application Bugs Found During Planning

These are defects in the **application code**, not in this plan. Listed so Step 18 can file the ones this epic does not fix.

| # | Bug | Location | Fixed by |
|---|---|---|---|
| B-1 | AI SDK clients set neither `timeout` nor `maxRetries`, inheriting a 10-min timeout × 3 attempts ≈ 30 min worst case | `claude.ts:23`, `openai.ts:23` | Step 6 |
| B-2 | All non-auth provider errors (timeout, abort, 429, 529) swallowed into `return []`, surfacing as `200 {suggestions:[]}` | `claude.ts:50-57`, `openai.ts:39-46` | Step 7 |
| B-3 | `await request.json()` unguarded → malformed body throws an unhandled 500 outside the documented schema | `suggest/route.ts:8` | Step 4 |
| B-4 | `provider` and `context` cast with `as` and never validated; `context.takenOn` can be absent (`TypeError` → 500) or unbounded (prompt-size/cost blowup) | `suggest/route.ts:9-13`, consumed at `claude.ts:9`, `openai.ts:9` | Step 4 |
| B-5 | OpenAI call sets no output token cap, while the Claude call sets `max_tokens: 1024` — unexplained asymmetry | `openai.ts:27-38` | Step 4 |
| B-6 | MCP bridge providers probe `localhost:8960-8962` on every `/api/providers` call with no off switch — ~6s of guaranteed-failing latency per page load in any hosted environment | `mcp-bridge.ts:195-207`, `suggestions/index.ts:11-16` | Step 5 |
| ~~B-7~~ | ~~GitHub batch results routed through the cache as a data channel; entries can evict each other between the write loop and the read loop~~ — **REFUTED by critic-deploy's simulation** (0 spurious failures at capacity under realistic 8-suggestion batches; the write/read span is synchronous with no `await`, so no interleaving is possible). Do not file as a bug. | `registries/index.ts:108-124`, `cache.ts:56-72` | Refuted — no action |
| B-8 | `evictOldest()` evicts by insertion order (FIFO), but `application-namer-v1.md` § Caching Strategy specifies **LRU** — plan/implementation drift. (Note: this no longer "makes B-7 reachable" since B-7 is refuted; it is purely a docs/naming mismatch, low severity.) | `cache.ts:26-40` | **Not fixed — file as a bug (Step 18)** |

---

## Blocked-On-User Summary

| Step | Decision | Recommendation | Blocks |
|---|---|---|---|
| **1** | Hosting provider + tier | **Vercel, Hobby** | 6, 11-19 |
| **2** | AI exposure posture **+ the loss-ceiling figure** | **Option W**; Option X is a fine lower-effort launch that sets the ceiling to zero | 3, 4, 12, 13 |
| **11** | Create account, connect repo | — | 12, 14 |
| **12** | Enter secrets in host dashboard | Prod scope only; `GITHUB_TOKEN` required; no MCP vars | 14 |
| **13** | Provider hard spend caps + alerts | The figure named at Step 2 — the real bound per ADR-4 | 14 |
| **16** | Custom domain | **Skip** — use the host-provided URL | — |

Steps 3-5 and 7-10 are host-agnostic and can be worked immediately in parallel with Steps 1-2, except Step 6 (needs the tier from Step 1) and Step 7 (needs Step 6). Step 18 can start at any time.

---

## Verification of This Plan

- Every Next.js claim is cited from the vendored docs at `node_modules/next/dist/docs/` for the installed version **16.2.6**, per `AGENTS.md`. Nothing about Next.js is asserted from memory. Where a review claim conflicted with the vendored docs, the docs were re-read and the conflict resolved explicitly in-line (see the Axis-1 correction note on Fly.io/Render).
- Every claim about this codebase is cited to a file and line and was verified by reading the source, including all claims inherited from review feedback.
- Host-specific limits (function duration, tier terms, trusted client-IP header, preview-deploy secret exposure) are deliberately **not** stated as fixed numbers — they change, and staleness would be dangerous. Steps 3, 6 and 12 require verifying them against the host's live documentation at execution time.

---

## Changelog (consensus revisions)

### From Architect Review (arch-deploy)
- [x] **CRITICAL:** Corrected Driver 2 — worst case is ~30 min, not ~65s. Both SDK clients omit `timeout` **and** `maxRetries`, inheriting a 10-min timeout × 3 attempts. Step 6 now requires setting both, with an explicit `timeout × (1 + maxRetries) + fan-out` sizing rule.
- [x] **CRITICAL:** Added Step 7 (error classification) as a separate step. Step 5's AC in v1 was unachievable because `claude.ts:50-57`/`openai.ts:39-46` swallow all non-auth errors into `return []` → `200 {suggestions:[]}`. Added as Driver 3, Risk R-2, and pre-mortem Scenario 3 (which replaces v1's weaker "timeout that doesn't look like a timeout").
- [x] **CRITICAL:** Rewrote ADR-2's rationale. v1 claimed the cache was "purely an optimization with no correctness dependency" — false. `registries/index.ts:108-124` uses it as an intra-request data channel for GitHub batch results. Introduced the two-tier framing (cross-request optimization vs. intra-request data channel); the *decision* survives because only the optimization tier is affected by fragmentation.
- [x] **CRITICAL:** Corrected ADR-2 alternative (iii). v1 rejected "delete `cache.ts`" for the wrong reason (efficiency); it would in fact break every `/api/suggest` GitHub result. Correction called out explicitly so the error is not silently overwritten.
- [x] Expanded Step 3 with the three details that decide whether the limiter is real: trusted client-IP source (never client-supplied `x-forwarded-for`), placement (route handlers, not `proxy.ts`), and a memory cap on the limiter's own map. Added Risks R-5, R-6 and two new limiter tests.
- [x] Added Step 4 (per-request cost bounding) as a distinct step: guarded `request.json()`, runtime validation of `context.takenOn` and `provider`, explicit OpenAI output cap, body-size bound. Added Risk R-7.
- [x] Added **ADR-4** (rate-limiter shared state): accept per-instance counting, with the hard provider spend cap as the real bound and the loss ceiling stated explicitly. Step 2 now requires Andrew to name the figure; Step 13 records it. Added Principle 6.
- [x] **Rejected one architect claim, with evidence.** The assertion that "neither Render nor Fly.io appear in the vendored Next docs" is false — both appear at `01-app/01-getting-started/17-deploying.md:59,61`. v1's wording was nonetheless imprecise, so it was corrected to describe them accurately as provider-guidance links under the *Docker* section, which also weakens Option C's "no Dockerfile needed" claim (that rests on Render's own docs, not Next's).

### From independent verification during revision
- [x] Discovered and documented **B-7/R-9** (later refuted — see below): at the 1000-entry cap, `setInCache`'s FIFO eviction was suspected to drop a GitHub entry between `checkNames`' write loop and read loop, producing spurious `"GitHub batch failed"`.
- [x] Discovered **B-8**: `evictOldest()` is FIFO, but `application-namer-v1.md` specifies LRU — plan/implementation drift (docs/naming only; does not itself cause data loss, see below).
- [x] Corrected a Next 15 assumption that would have been wrong here: middleware is renamed **`proxy.ts`** in Next 16 and **defaults to the Node.js runtime**, not Edge (`03-file-conventions/proxy.md`). This changes the Step 3 placement argument from "impossible in middleware" to "possible but not guaranteed to share state."
- [x] Confirmed `checkGitHubBatch` catches internally and cannot reject, so `checkNames`' `Promise.all` has no unhandled-rejection path — a suspected issue that was investigated and **not** included, rather than asserted.
- [x] Added the **Application Bugs Found During Planning** table (B-1..B-8) and Step 18 to ensure bugs surfaced during planning become beads instead of dying in this document.
- [x] Strengthened smoke checks: added spoofed-header and oversized-body checks.

### From Critic Review (critic-deploy)
- [x] **REFUTED B-7/R-9 by simulation, with evidence.** critic-deploy modeled `cache.ts`'s exact eviction semantics: a saturated 1000-entry cache plus a realistic 8-suggestion `checkNames` call produces **0 spurious failures** — self-eviction needs >1000 GitHub entries written in a single call, which this app never generates (max 8 suggestions). Separately, the write loop (`registries/index.ts:108-110`) and read loop (`:123-124`) have **no `await` between them**, so the span is synchronous and no concurrent request can interleave on a single-threaded runtime — the "more likely on long-lived hosts" framing was backwards, not just overstated. B-7 and R-9 struck from both tables; every downstream mention (Axis-1 cons for Options B/C, the Vercel recommendation rationale, ADR-1's alternatives, ADR-2's status line, the two smoke checks) corrected to remove the refuted claim. The Vercel recommendation itself is unaffected — it never depended on this argument, only cited it as a bonus. B-8 (FIFO-vs-LRU naming drift) survives as a low-severity docs mismatch, no longer described as "reachable."
- [x] Confirmed independently: catch-all error swallow (B-2), missing SDK timeout/retry config (B-1), unguarded `request.json()` (B-3), unvalidated `context` (B-4), and ADR-2's corrected cache rationale all match critic-deploy's own findings — cross-verification raises confidence in both reviews.
- [x] Adjudicated a disagreement between arch-deploy and critic-deploy over whether Render/Fly.io appear in the vendored Next docs: critic-deploy was right (they do, at `17-deploying.md:59,61`, under the Docker section) — already reflected in the "Correction on documentation status (v2)" note above.
- [x] Answered critic-deploy's open question on AC #11: `.omc/` is excluded by a **global** gitignore (`~/.gitignore-global:180`), not a project rule, confirmed via `git check-ignore -v`. This plan file is therefore never committed. AC #11 ("a new maintainer can read README.md plus this plan") does not hold for a maintainer who only clones the repo. **Resolution:** the four ADRs in this plan should be copied into `docs/decisions/` (tracked, per this repo's ADR convention) once Andrew signs off on ADR-1 — that satisfies AC #11 without requiring `.omc/` to be committed. Not yet done; worth its own step/bead if this epic proceeds.

### Outstanding (minor, not yet folded in)
- [ ] critic-deploy also flagged: smoke check 9 pointed at the wrong route (should re-verify which check this refers to against the current smoke list above), Axis 2's Option Y+W framing possibly mis-typed, the per-instance rate limiter described as "permissive in aggregate" without stating the multiplier explicitly, the pre-mortem missing a process-level failure scenario, one risk-table row rated "High \| Low" that reads oddly, AC #4's provider-list "shape" being ambiguous (omit vs. mark-unavailable), and a few missing items (do-nothing cost baseline, abort criteria, `engines` field, a named tool for Step 7). None are blocking; fold in before Andrew signs off on ADR-1 if this epic is picked up.
