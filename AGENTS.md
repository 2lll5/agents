# AGENTS.md

## Project overview

Cloudflare Agents SDK — a framework for building stateful AI agents on Cloudflare Workers. This is a monorepo containing the core SDK packages, examples, guides, sites, and documentation.

## Repository structure

```
packages/          # Published npm packages (need changesets for changes)
  agents/          # Core SDK (see packages/agents/AGENTS.md)
  ai-chat/         # @cloudflare/ai-chat — higher-level AI chat agent
  hono-agents/     # Hono framework integration
  codemode/        # @cloudflare/codemode — experimental code generation

examples/          # Self-contained demo apps (see examples/AGENTS.md)
  playground/      # Main showcase app — all SDK features in one UI (uses Kumo design system)
  mcp/             # MCP server example
  mcp-client/      # MCP client example
  ...              # ~20 examples total

experimental/      # Work-in-progress experiments (not published, no stability guarantees)

site/              # Deployed websites
  agents/          # agents.cloudflare.com (Astro)
  ai-playground/   # Workers AI playground (React + Vite)

guides/            # In-depth pattern tutorials with narrative READMEs (see guides/AGENTS.md)
  anthropic-patterns/
  human-in-the-loop/

openai-sdk/        # Examples using @openai/agents SDK
  basic/ chess-app/ handoffs/ human-in-the-loop/ ...

docs/              # Markdown docs for developers.cloudflare.com (see docs/AGENTS.md)
design/            # Architecture and design decision records (see design/AGENTS.md)
scripts/           # Repo-wide tooling (typecheck, export checks, update checks)
```

## Nested AGENTS.md files

Some directories have their own AGENTS.md with deeper guidance:

| File                        | Scope                                                                     |
| --------------------------- | ------------------------------------------------------------------------- |
| `packages/agents/AGENTS.md` | Core SDK internals — exports, source layout, build, testing, architecture |
| `examples/AGENTS.md`        | Example conventions — required structure, consistency rules, known issues |
| `guides/AGENTS.md`          | Guide conventions — how guides differ from examples, README expectations  |
| `docs/AGENTS.md`            | Writing user-facing docs — Diátaxis framework, upstream sync, style       |
| `design/AGENTS.md`          | Design records and RFCs — format, workflow, relationship to docs          |

## Setup

```bash
pnpm install       # installs all workspaces
```

Node 24+ required. Uses pnpm workspaces with [Nx](https://nx.dev) for task orchestration, caching, and affected detection.

## Commands

Run from the repo root:

| Command                            | What it does                                                       |
| ---------------------------------- | ------------------------------------------------------------------ |
| `pnpm run build`                   | Builds all packages via Nx (cached, dependency-ordered)            |
| `pnpm run check`                   | Full CI check: sherif + export checks + oxfmt + oxlint + typecheck |
| `pnpm run test`                    | Runs all tests via Nx (cached)                                     |
| `pnpm run test:react`              | Runs Playwright-based React hook tests for agents                  |
| `pnpm run typecheck`               | TypeScript type checking across the repo (custom script)           |
| `pnpm run format`                  | Oxfmt format all files                                             |
| `pnpm run check:exports`           | Verifies package.json exports match actual build output            |
| `pnpm exec nx affected -t build`   | Build only packages affected by current changes                    |
| `pnpm exec nx affected -t test`    | Test only packages affected by current changes                     |
| `pnpm exec nx run <project>:build` | Build a single project (and its dependencies)                      |

Run an example locally:

```bash
cd examples/playground   # or any example
pnpm dev                 # starts Vite dev server + Workers runtime via @cloudflare/vite-plugin
```

Example apps will normally hot reload when the dev server is running. When the dev server is running, make sure to rebuild changed packages (`pnpm run build`) to see changes reflected in the running app.

## Code standards

### TypeScript

- Strict mode enabled (`agents/tsconfig`)
- Target: ES2021, module: ES2022, moduleResolution: bundler
- `verbatimModuleSyntax: true` — use explicit `import type` for type-only imports
- JSX: `react-jsx`

### Linting — Oxlint

Config in `.oxlintrc.json`. Plugins: `react`, `jsx-a11y`, `typescript`, `react-hooks`. Key rules:

- `no-explicit-any: "error"` — never use `any`, use `unknown` and narrow
- `no-unused-vars: "error"` — with `varsIgnorePattern: "^_"` and `argsIgnorePattern: "^_"`
- `correctness` category set to `"error"` — catches common mistakes
- `jsx-a11y` rules enabled — accessibility violations are errors
- `react-hooks/exhaustive-deps: "warn"` — warns on missing hook dependencies

Oxlint does **not** handle formatting — Oxfmt does.

### Formatting — Oxfmt

- Run `pnpm run format` or rely on lint-staged (auto-formats on commit via husky)
- Config in `.oxfmtrc.json` (`trailingComma: "none"`, `printWidth: 80`)

### Workers conventions

- Always TypeScript, always ES modules
- `wrangler.jsonc` (not `.toml`) for configuration
- All wrangler configs use `compatibility_date: "2026-06-11"` and `compatibility_flags: ["nodejs_compat"]`
- Never hardcode secrets — use `wrangler secret put` or `.env`
- No native/FFI dependencies (must run in Workers runtime)

## Testing

Tests use **vitest** with `@cloudflare/vitest-pool-workers` for running inside the Workers runtime.

```bash
pnpm run test             # agents + ai-chat unit/integration tests
pnpm run test:react       # Playwright-based React hook tests (agents package)
```

Test locations:

- `packages/agents/src/tests/` — core SDK tests
- `packages/agents/src/react-tests/` — React hook tests (Playwright + vitest-browser-react)
- `packages/ai-chat/src/tests/` — AI chat tests
- `packages/agents/src/tests-d/` — type-level tests (`.test-d.ts`)

Each test directory has its own `vitest.config.ts` and (for Workers tests) a `wrangler.jsonc`.

## Contributing

### Changesets

Changes to `packages/` that affect the public API or fix bugs need a changeset:

```bash
pnpm exec changeset       # interactive prompt — pick packages, semver bump, description
```

This creates a markdown file in `.changeset/` that gets consumed during release.

Examples, guides, and sites don't need changesets.

### Pull request process

CI runs on every PR (`pnpm install --frozen-lockfile && pnpm run build && pnpm run check && pnpm exec nx affected -t test`); the workflow is in `.github/workflows/pullrequest.yml`. On push to `main` the Release workflow (`.github/workflows/release.yml`) runs the same steps but uses `nx run-many -t test` as a safety net against under-reported affected projects, then publishes via changesets. All checks must pass.

### Generated files

- `env.d.ts` files are generated by `wrangler types` — regenerate with `pnpm exec wrangler types` inside the relevant example/package, don't hand-edit
- `pnpm-lock.yaml` — regenerated by `pnpm install`, don't hand-edit

## Learned Workspace Facts

- `packages/shell/` is published as `@cloudflare/shell` — an experimental sandboxed JS execution and filesystem runtime for agents, built on the same dynamic Worker loader machinery as `@cloudflare/codemode`.
- To run code against a `Workspace`: import `stateTools` from `@cloudflare/shell/workers` and `DynamicWorkerExecutor`/`resolveProvider` from `@cloudflare/codemode`; use `executor.execute(code, [resolveProvider(stateTools(workspace))])`.

## Learned User Preferences

- Keep `Workspace` as a pure durable filesystem — do not embed execution or session logic inside it. Execution is a caller concern wired via `@cloudflare/codemode` + `stateTools`.
- When a package boundary feels wrong (e.g., a helper package depending on a larger package just for an adapter), prefer moving the adapter out rather than carrying the dependency.

## Boundaries

**Always:**

- Run `pnpm run check` before considering work done
- Use `import type` for type-only imports (enforced by `verbatimModuleSyntax`)
- Keep examples simple and self-contained — they're user-facing learning material
- Use Cloudflare Workers APIs (KV, D1, R2, Durable Objects, etc.) over third-party equivalents
- Use Workers AI for LLM calls in examples — not third-party APIs like OpenAI or Anthropic

**Ask first:**

- Adding new dependencies to `packages/` (these ship to users)
- Changing `wrangler.jsonc` compatibility dates across the repo
- Modifying CI workflows

**Never:**

- Hardcode secrets or API keys
- Add native/FFI/C-binding dependencies
- Use `any` — Oxlint will reject it
- Use CommonJS or Service Worker format — ES modules only
- Modify `node_modules/` or `dist/` directories
- Force push to main

## Google Drive — Repository-specific Build Delivery (Required)

**所有 Codex／開發 Agent 必須遵守**：每個 GitHub Repository 有獨立交付資料夾 `Google Drive / AI Project/agents/`，名稱必須與 Repo 完全相同；AI Project folder ID: `1Yhfro6Ss5TSCXCVIZIHEu5h7V1gL8b27`。

1. **最新完整 Build 直接放專案資料夾根目錄**，包含版本化檔名的 ZIP／EXE 包、SHA-256 及必要的分片、還原腳本／使用說明；多平台最新版本可並存。不要在 AI Project 根目錄放任何 Build，也不要每次 Build 都建立版本資料夾。
2. **僅重要版本差異**（重要功能、架構、相容性、畫面里程碑）才在專案內建立 `vX.Y.Z-說明/` 子資料夾保存；一般舊版本非破壞性移入 `歷史封存/`，不可逕自刪除任何舊檔。
3. **先確認新版本完整，才能更換最新版本**：上傳後讀回 Google Drive 名稱、parent、大小及 checksum；只看到 SHA、缺少 ZIP 或缺一個分割檔的版本屬「不完整」，不得取代前一完整最新版。
4. 尚未實際編譯且驗證成功，不得宣稱已產出、已上傳或可執行。每次新開發推送 GitHub，正式 Build/發布的 GitHub Actions 優先 `workflow_dispatch` 或 `release: published`，避免每次推送 main 觸發高成本 Build。
5. 回報最新版本、Drive **專案資料夾連結**、檔案清單、checksum 和實測／上傳狀態；第一次成功 Build 才需建立專案資料夾。

**Coding agents (English):** Publish only the newest *complete and verified* build directly under `AI Project/agents/`. Preserve milestone versions in `vX.Y.Z-description/` only for material differences; archive routine older versions under `歷史封存/` without deleting them. Never promote incomplete uploads.
