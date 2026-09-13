# Application Namer

Check if your app or package name is available across **npm**, **Homebrew**, **PyPI**, and **GitHub** — all in one search. When a name is taken, get AI-powered alternative suggestions with availability pre-checked across all registries.

![Next.js](https://img.shields.io/badge/Next.js-16-black?logo=next.js)
![TypeScript](https://img.shields.io/badge/TypeScript-5-blue?logo=typescript)
![License](https://img.shields.io/badge/license-MIT-green)

## Features

- **Multi-registry lookup** — Checks name availability across 5 registries simultaneously:
  - **npm** — with version, description, and weekly download counts
  - **Homebrew Formulae** — CLI tools (`brew install`)
  - **Homebrew Cask** — Desktop apps (`brew install --cask`)
  - **PyPI** — Python packages with author info
  - **GitHub** — Public repository search by stars
- **AI-powered suggestions** — When a name is taken, generate 8 creative alternatives using:
  - Claude API (Anthropic)
  - OpenAI API
  - Local MCP bridge servers (Claude, Codex, Copilot via [mcp-agent-bridge](https://github.com/catesandrew/mcp-agent-bridge))
- **Smart re-checking** — Every AI suggestion is automatically verified against all 5 registries before display
- **Batched GitHub queries** — Suggestion re-checks use a single GitHub API call (OR query) to stay within rate limits
- **In-memory caching** — 5-minute TTL with LRU eviction (1000 entries) to avoid redundant API calls
- **Input validation** — Strict package name rules enforced server-side with PyPI normalization warnings
- **Responsive UI** — Works on mobile (375px+) and desktop with shadcn/ui components
- **Graceful degradation** — Individual registry or provider failures never block the rest of the results

## Quick Start

### Prerequisites

- [Node.js](https://nodejs.org/) 18+
- [pnpm](https://pnpm.io/) (`npm install -g pnpm`)

### Install and Run

```bash
git clone https://github.com/catesworks/application-namer.git
cd application-namer
pnpm install
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000) and type a name to check.

> Registry checks work immediately with no configuration. AI suggestions require at least one provider to be configured (see below).

### Configure AI Providers

Copy the example environment file and add your keys:

```bash
cp .env.example .env.local
```

Edit `.env.local` with at least one AI provider:

```env
# Anthropic Claude — https://console.anthropic.com
ANTHROPIC_API_KEY=sk-ant-...

# OpenAI — https://platform.openai.com/api-keys
OPENAI_API_KEY=sk-...
```

**Optional settings:**

```env
# Increases GitHub search rate limit from 10 to 30 req/min
# Create at https://github.com/settings/tokens (no scopes needed)
GITHUB_TOKEN=ghp_...

# MCP bridge server URLs (if using mcp-agent-bridge locally)
MCP_CLAUDE_URL=http://localhost:8960
MCP_CODEX_URL=http://localhost:8961
MCP_COPILOT_URL=http://localhost:8962
```

## How It Works

### 1. Search

Type a package name and hit Enter (or click **Check**). The app validates the name and queries all 5 registries in parallel with a 4-second timeout per registry.

### 2. Review Results

Each registry shows one of:
- **Available** (green) — The name is free
- **Taken** (red) — The name exists, with metadata (version, downloads, description)
- **Error** (yellow) — The registry couldn't be reached
- **Rate Limited** (blue) — GitHub API limit hit (use a `GITHUB_TOKEN` to increase)

### 3. Get Suggestions

If the name is taken on any registry, click **Suggest alternatives**. Choose an AI provider from the dropdown, and the app will:
1. Generate 8 creative name alternatives
2. Validate each suggestion against package naming rules
3. Re-check every suggestion across all 5 registries
4. Display results with per-registry availability for each suggestion

## Architecture

```
Browser (React)
  |
  |-- POST /api/check      --> Registry orchestrator (Promise.allSettled)
  |-- POST /api/suggest     --> AI provider --> Validate --> Batch re-check
  |-- GET  /api/providers   --> Provider availability detection
  |
  v
Server-side (Next.js API Routes)
  |
  +-- src/lib/registries/   --> npm, Homebrew, PyPI, GitHub checkers
  |     +-- cache.ts        --> In-memory LRU cache (5-min TTL)
  |     +-- index.ts        --> checkName() / checkNames() orchestrator
  |
  +-- src/lib/suggestions/  --> Claude, OpenAI, MCP bridge providers
  |     +-- index.ts        --> Provider router + availability detection
  |
  +-- src/lib/validation.ts --> Shared name validation (server + client)
```

All registry checks and AI calls happen server-side to keep API keys secure and avoid CORS issues.

## Registries

| Registry | API | Auth | Extra Info |
|----------|-----|------|------------|
| npm | `registry.npmjs.org` + `api.npmjs.org/downloads` | None | Version, description, weekly downloads |
| Homebrew | `formulae.brew.sh/api/formula` | None | Description, homepage |
| Homebrew Cask | `formulae.brew.sh/api/cask` | None | App name, homepage |
| PyPI | `pypi.org/pypi/{name}/json` | None | Summary, version, author |
| GitHub | `api.github.com/search/repositories` | Optional token | Owner, stars, description |

### PyPI Name Normalization

PyPI treats hyphens (`-`), underscores (`_`), and dots (`.`) as equivalent. If your name contains any of these characters, a warning is displayed. For example, `my-tool` and `my_tool` are the same package on PyPI.

### GitHub Rate Limits

Without a token: 10 requests/minute. With a `GITHUB_TOKEN`: 30 requests/minute. The app detects rate limiting (403/429 responses) and shows a "Rate Limited" status instead of failing.

## AI Providers

| Provider | Model | Config | How It Works |
|----------|-------|--------|-------------|
| Claude API | claude-sonnet-4-20250514 | `ANTHROPIC_API_KEY` | Tool use for structured JSON output |
| OpenAI API | gpt-4o-mini | `OPENAI_API_KEY` | JSON mode with function calling |
| Claude (MCP) | Via bridge | `MCP_CLAUDE_URL` | Streamable HTTP to local mcp-agent-bridge |
| Codex (MCP) | Via bridge | `MCP_CODEX_URL` | Streamable HTTP to local mcp-agent-bridge |
| Copilot (MCP) | Via bridge | `MCP_COPILOT_URL` | Streamable HTTP to local mcp-agent-bridge |

### Using MCP Bridge Providers

The MCP bridge providers connect to locally-running [mcp-agent-bridge](https://github.com/catesandrew/mcp-agent-bridge) servers. These bridge Claude Code, Codex CLI, and Copilot CLI into MCP-compatible HTTP servers.

To use them:

1. Install and run [mcp-agent-bridge](https://github.com/catesandrew/mcp-agent-bridge)
2. Ensure the servers are running on the configured ports (default: 8960, 8961, 8962)
3. The app auto-detects running servers — available providers appear in the dropdown

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | [Next.js](https://nextjs.org/) 16 (App Router) |
| Language | [TypeScript](https://www.typescriptlang.org/) 5 |
| Styling | [Tailwind CSS](https://tailwindcss.com/) 4 |
| Components | [shadcn/ui](https://ui.shadcn.com/) with [@base-ui/react](https://base-ui.com/) |
| Icons | [Lucide React](https://lucide.dev/) |
| AI SDKs | [@anthropic-ai/sdk](https://docs.anthropic.com/), [openai](https://platform.openai.com/docs) |

## Project Structure

```
src/
├── app/
│   ├── page.tsx                # Main search page
│   ├── layout.tsx              # Root layout with metadata
│   └── api/
│       ├── check/route.ts      # POST: check name across registries
│       ├── suggest/route.ts    # POST: generate + re-check suggestions
│       └── providers/route.ts  # GET: list available AI providers
├── components/
│   ├── search-form.tsx         # Name input with validation
│   ├── results-grid.tsx        # Registry results display
│   ├── result-card.tsx         # Individual registry card
│   ├── provider-selector.tsx   # AI provider dropdown
│   └── suggestions-panel.tsx   # AI suggestions with error boundary
├── hooks/
│   └── use-name-check.ts      # Client state + request management
└── lib/
    ├── types.ts                # Shared TypeScript interfaces
    ├── validation.ts           # Package name validation rules
    ├── registry-client.ts      # HTTP fetch with 4s timeout
    ├── registries/
    │   ├── npm.ts              # npm registry (2 concurrent fetches)
    │   ├── homebrew.ts         # Homebrew formulae + cask
    │   ├── pypi.ts             # PyPI with normalization
    │   ├── github.ts           # GitHub search (single + batch)
    │   ├── cache.ts            # LRU cache (1000 entries, 5-min TTL)
    │   └── index.ts            # Registry orchestrator
    └── suggestions/
        ├── claude.ts           # Claude API provider
        ├── openai.ts           # OpenAI API provider
        ├── mcp-bridge.ts       # MCP bridge (Claude/Codex/Copilot)
        └── index.ts            # Provider router
```

## Development

```bash
pnpm dev        # Start dev server (http://localhost:3000)
pnpm build      # Production build
pnpm start      # Start production server
pnpm lint       # Run ESLint
```

## Name Validation Rules

The app enforces these rules for package names (shared between client and server):

- Must start with a lowercase letter or digit
- Only lowercase letters, digits, hyphens, underscores, and dots allowed
- Maximum 214 characters (npm limit)
- Cannot start with `@` (scoped packages not supported in v1)
- Names with `-`, `_`, or `.` trigger a PyPI normalization warning

## License

MIT
