# CLAUDE.md — Context7 Monorepo

## Project Overview

Context7 is an Upstash project that provides up-to-date, version-specific documentation and code examples for libraries, delivered directly to AI coding assistants via the Model Context Protocol (MCP). It solves the problem of LLMs generating code with outdated or hallucinated APIs by pulling current documentation from library sources and inserting it into prompts.

**Repository:** `github.com/upstash/context7`
**License:** MIT
**Node.js requirement:** >=18

## Repository Structure

```
context7/
├── packages/
│   ├── mcp/              # @upstash/context7-mcp — MCP server (stdio + HTTP)
│   ├── sdk/              # @upstash/context7-sdk — TypeScript/JS client SDK
│   ├── cli/              # ctx7 — Command-line interface
│   └── tools-ai-sdk/     # @upstash/context7-tools-ai-sdk — Vercel AI SDK integration
├── plugins/
│   ├── claude/           # Claude Code plugin (agents, commands, skills)
│   └── cursor/           # Cursor IDE plugin (agents, rules, skills)
├── docs/                 # Documentation
├── i18n/                 # Internationalization (14 languages)
├── public/               # Static assets (icons, images)
├── .github/workflows/    # CI/CD (test, release, canary, ECR deploy)
└── .changeset/           # Changesets for versioning
```

## Package Manager & Monorepo

- **Package manager:** pnpm v10 (do NOT use npm or yarn)
- **Workspace config:** `pnpm-workspace.yaml` includes `packages/*`
- **Install:** `pnpm install` (use `--frozen-lockfile` in CI)

## Quick Reference — Common Commands

```bash
pnpm install              # Install all dependencies
pnpm build                # Build all packages
pnpm test                 # Run all tests
pnpm lint:check           # Check linting (no auto-fix)
pnpm lint                 # Lint with auto-fix
pnpm format:check         # Check formatting
pnpm format               # Format with auto-fix
pnpm typecheck            # TypeScript type-check all packages
pnpm clean                # Remove all dist/ and node_modules/
```

### Per-package commands

```bash
pnpm build:sdk            # Build SDK only
pnpm build:mcp            # Build MCP only
pnpm build:ai-sdk         # Build AI SDK tools only
pnpm test:sdk             # Test SDK only
pnpm test:tools-ai-sdk    # Test AI SDK tools only
```

## Packages

### `@upstash/context7-mcp` (packages/mcp)

MCP server implementation exposing two tools:
- **resolve-library-id** — Resolves library names to Context7 IDs
- **query-docs** — Fetches documentation content for a library

Transport modes: `stdio` (default) and `http` (Express v5, port 3000).

- **Build:** `tsc` (plain TypeScript compiler, outputs to `dist/`)
- **Dev:** `tsc --watch`
- **Start HTTP:** `node dist/index.js --transport http`
- **No tests yet** — test script is a no-op

Key source files:
- `src/index.ts` — Entry point, CLI arg parsing, server setup
- `src/lib/api.ts` — Backend API calls
- `src/lib/encryption.ts` — Client context / encryption
- `src/lib/jwt.ts` — JWT validation
- `src/lib/utils.ts` — Formatting utilities
- `src/lib/constants.ts` — Server version, URLs

### `@upstash/context7-sdk` (packages/sdk)

TypeScript client SDK with two main methods:
- `Context7.searchLibrary(query, libraryName, options?)` — Search for libraries
- `Context7.getContext(query, libraryId, options?)` — Fetch documentation

Both support `type: "json"` or `type: "txt"` return formats.

- **Build:** `tsup` (outputs ESM + CJS to `dist/`)
- **Test:** `vitest run`
- **Path aliases:** `@error`, `@http`, `@commands/*`, `@utils/*` (defined in `tsconfig.json`)
- API base URL: `https://context7.com/api`

Key source structure:
- `src/client.ts` — Main `Context7` class
- `src/commands/` — Command pattern (`search-library/`, `get-context/`)
- `src/error/` — Custom `Context7Error` class
- `src/http/` — HTTP client with retry/backoff

### `ctx7` (packages/cli)

CLI tool with commands: `skill`, `auth`, `generate`.

- **Build:** `tsup` (ESM only, with shebang banner)
- **Binary name:** `ctx7`
- **No tests** — no test script defined

Key source structure:
- `src/index.ts` — Entry point
- `src/commands/` — CLI commands (skill, auth, generate)
- `src/utils/` — Utilities (api, auth, deps, github, ide, installer, logger, etc.)

### `@upstash/context7-tools-ai-sdk` (packages/tools-ai-sdk)

Vercel AI SDK integration providing agents, tools, and prompts.

- **Build:** `tsup` (ESM + CJS, with subpath export `./agent`)
- **Test:** `vitest run`
- **Peer dependency:** `ai` (Vercel AI SDK v6+)

Key source structure:
- `src/index.ts` — Main exports
- `src/agents/` — AI SDK agent implementations
- `src/tools/` — Tool definitions
- `src/prompts/` — Prompt templates

## Code Style & Conventions

### TypeScript

- **Strict mode** enabled globally
- **Target:** ES2022 (root), ESNext (SDK)
- **Module system:** ESM primary; SDK and tools-ai-sdk also produce CJS
- **Bundler:** `tsup` for SDK, CLI, and tools-ai-sdk; plain `tsc` for MCP

### ESLint (v9, flat config)

Config: `eslint.config.js`
- Extends: `typescript-eslint` recommended
- `@typescript-eslint/no-unused-vars`: error (prefix unused args with `_`)
- `@typescript-eslint/no-explicit-any`: warn
- Prettier integration via `eslint-plugin-prettier`
- Ignores: `node_modules/`, `build/`, `dist/`, `.git/`, `.github/`

### Prettier

Config: `prettier.config.mjs`
- Double quotes (not single)
- 2-space indentation
- 100 character print width
- Trailing commas: ES5
- LF line endings
- Arrow parens: always

### Naming Conventions

- **Package names:** `@upstash/context7-*` scoped, `ctx7` for CLI
- **Files:** camelCase for most (`client.ts`, `api.ts`); commands use directory-per-command pattern
- **Types:** Co-located in `types.ts` files
- **Test files:** `*.test.ts` co-located with source
- **Unused function params:** Prefix with `_` (e.g., `_req`)

### Import Conventions

- SDK uses path aliases: `@error`, `@http`, `@commands/*`, `@utils/*`
- MCP uses relative imports with `.js` extensions (Node16 module resolution)
- Use `type` imports where possible (`import type { ... }`)

## CI/CD Pipeline

### test.yml (PRs and pushes to master)

Runs in order: `lint:check` → `format:check` → `build` → `typecheck` → `test`

Environment: Node 20, pnpm 10, Ubuntu latest.

Required secrets for tests: `CONTEXT7_API_KEY`, `AWS_REGION`, `AWS_BEARER_TOKEN_BEDROCK`.

### release.yml (push to master)

Uses `@changesets/cli` for automated versioning and npm publishing.

### Other workflows

- `canary-release.yml` — Snapshot/canary releases
- `changeset-check.yml` — Validates PRs have changeset files
- `mcp-registry.yml` — MCP marketplace integration
- `ecr-deploy.yml` — ECR container deployment

## Versioning

Uses **Changesets** (`@changesets/cli`). When making changes:
1. Run `pnpm changeset` to create a changeset file describing your change
2. Select affected packages and bump type (patch/minor/major)
3. Changeset files go in `.changeset/` and are committed with the PR
4. The `changeset-check.yml` workflow will validate PRs include changesets

## Environment Variables

| Variable | Purpose | Required |
|---|---|---|
| `CONTEXT7_API_KEY` | API authentication (all packages) | Recommended |
| `HTTPS_PROXY` / `HTTP_PROXY` | Proxy configuration (MCP) | No |
| `CTX7_TELEMETRY_DISABLED` | Disable CLI telemetry | No |

## Architecture Notes

- The MCP server uses `AsyncLocalStorage` for per-request context in HTTP mode
- The SDK `HttpClient` has built-in retry with exponential backoff (5 retries)
- The MCP server supports both anonymous and authenticated requests
- Authentication: `CONTEXT7_API_KEY` header, `X-API-Key` header, or `Authorization: Bearer <token>`
- API key prefix convention: `ctx7sk`
- Backend API: `https://context7.com/api` with v2 endpoints (`/v2/libs/search`, `/v2/context`)

## Before Submitting a PR

1. `pnpm lint:check` — No lint errors
2. `pnpm format:check` — Code is properly formatted
3. `pnpm build` — All packages build successfully
4. `pnpm typecheck` — No type errors
5. `pnpm test` — All tests pass
6. Include a changeset (`pnpm changeset`) if your change affects published packages
