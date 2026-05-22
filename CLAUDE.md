---
description: Core project guidelines for the Repomix codebase. Apply these rules when working on any code, documentation, or configuration files within the Repomix project.
alwaysApply: true
---

# Repomix

Repomix (v1.14.0) packs an entire repository's contents into a single AI-friendly file. It supports XML, Markdown, JSON, and plain text output formats and is used by AI tools (Claude, ChatGPT, Gemini, etc.) to ingest codebases for code review, documentation, debugging, and security analysis.

Refer to `README.md` for the full project overview, `CONTRIBUTING.md` for contribution procedures, and `repomix-instruction.md` for a structural overview aimed at AI code assistants.

## Directory Structure

```
repomix/
├── bin/                   # Compiled CJS entry point (repomix.cjs)
├── browser/               # Browser extension source code
├── src/                   # Main TypeScript source
│   ├── cli/               # CLI layer
│   │   ├── actions/       # Command implementations: defaultAction, remoteAction, initAction
│   │   ├── prompts/       # Interactive @clack/prompts flows
│   │   ├── reporters/     # CLI output reporters
│   │   ├── cliRun.ts      # Entry: parses args via commander, dispatches to actions
│   │   ├── cliReport.ts   # Formats and prints the pack result summary
│   │   ├── cliSpinner.ts  # Progress spinner integration
│   │   └── types.ts       # CliOptions type
│   ├── config/            # Configuration system
│   │   ├── configLoad.ts  # Loads + merges file config, CLI config, defaults
│   │   ├── configSchema.ts# Valibot schemas: Base/File/CLI/Merged + defaultConfig
│   │   ├── defaultIgnore.ts # Built-in ignore patterns
│   │   └── globalDirectory.ts # XDG-compliant global config path
│   ├── core/              # Domain logic
│   │   ├── file/          # File search, collection, processing, tree generation, sorting
│   │   ├── git/           # gitDiffHandle, gitLogHandle, gitRemoteParse
│   │   ├── metrics/       # Token counting (TokenCounter), character metrics, worker pool
│   │   ├── output/        # Output generation per style (XML/MD/JSON/plain), headers, sorting
│   │   ├── packager/      # produceOutput — writes output files, handles clipboard/stdout
│   │   ├── security/      # securityCheck (secretlint) + validateFileSafety
│   │   ├── skill/         # packSkill — generates Claude Agent Skills from codebases
│   │   ├── tokenCount/    # Token encoding utilities
│   │   └── treeSitter/    # AST-based code parsing with web-tree-sitter (WASM)
│   │   └── packager.ts    # pack() — top-level orchestrator
│   ├── mcp/               # MCP server integration
│   │   ├── tools/         # MCP tools: pack_codebase, pack_remote_repository, read/grep output, skill, fs
│   │   ├── prompts/       # MCP prompts: pack_remote_repository
│   │   └── mcpServer.ts   # createMcpServer / runMcpServer (stdio transport)
│   ├── shared/            # Shared utilities
│   │   ├── asyncMap.ts    # Concurrent async map with concurrency limit
│   │   ├── errorHandle.ts # RepomixError, structured error handling
│   │   ├── logger.ts      # Levelled logger (trace/debug/info/warn/error)
│   │   ├── memoryUtils.ts # logMemoryUsage / withMemoryLogging
│   │   ├── processConcurrency.ts # Worker thread pool via tinypool
│   │   ├── unifiedWorker.ts # Unified worker handler (bundled environments)
│   │   └── types.ts       # RepomixProgressCallback
│   ├── types/             # Global TypeScript type declarations
│   └── index.ts           # Public API surface (all named exports)
├── tests/                 # Vitest tests mirroring src/ layout
│   ├── cli/
│   ├── config/
│   ├── core/
│   ├── integration-tests/
│   ├── shared/
│   └── testing/           # Test utilities and fixtures
└── website/               # VitePress documentation site (Docker-based dev)
    ├── client/            # Vue.js frontend, VitePress config, markdown docs (en, ja, …)
    └── server/            # Backend API for remote repository processing
```

## Key Entry Points

| Entry | Purpose |
|---|---|
| `bin/repomix.cjs` | CLI binary (installed by npm) |
| `src/cli/cliRun.ts` | `run()` / `runCli()` — parses CLI args, dispatches actions |
| `src/core/packager.ts` | `pack()` — main programmatic API |
| `src/mcp/mcpServer.ts` | `runMcpServer()` — starts MCP stdio server |
| `src/index.ts` | All public exports for library consumers |

## Public API (`src/index.ts`)

```typescript
import { pack, collectFiles, processFiles, searchFiles, TokenCounter,
         parseFile, loadFileConfig, mergeConfigs, defineConfig, runCli } from 'repomix';
```

Key exports:
- `pack(rootDirs, config, progressCallback, deps?, explicitFiles?, options?)` → `PackResult`
- `collectFiles`, `processFiles`, `searchFiles`, `sortPaths`, `generateFileTree`
- `runSecurityCheck`, `TokenCounter`, `parseFile`
- `loadFileConfig`, `mergeConfigs`, `defineConfig`, `defaultIgnoreList`
- `runCli`, `runDefaultAction`, `runRemoteAction`, `runInitAction`
- `unifiedWorkerHandler`, `unifiedWorkerTermination` (for bundled environments)

## Configuration System

Config is composed of three layers (all optional except `cwd`) merged via `mergeConfigs`:

1. **File config** (`repomix.config.json` / `.repomixrc` / `repomix.config.ts`) — `RepomixConfigFile`
2. **CLI flags** — `RepomixConfigCli`
3. **Defaults** — `RepomixConfigDefault` (applied by Valibot schema)

Key config options with their defaults:

```jsonc
{
  "input": { "maxFileSize": 52428800 },          // 50 MB
  "output": {
    "filePath": "repomix-output.xml",
    "style": "xml",                              // xml | markdown | json | plain
    "parsableStyle": false,
    "fileSummary": true,
    "directoryStructure": true,
    "removeComments": false,
    "removeEmptyLines": false,
    "compress": false,                          // Tree-sitter AST compression
    "showLineNumbers": false,
    "copyToClipboard": false,
    "splitOutput": null,                        // split into N files
    "tokenCountTree": false,
    "git": {
      "sortByChanges": true,                    // sort by git change frequency
      "sortByChangesMaxCommits": 100,
      "includeDiffs": false,
      "includeLogs": false
    }
  },
  "ignore": { "useGitignore": true, "useDotIgnore": true, "useDefaultPatterns": true },
  "security": { "enableSecurityCheck": true },
  "tokenCount": { "encoding": "o200k_base" }    // gpt-tokenizer encoding
}
```

Tokenizer encodings supported: `o200k_base`, `cl100k_base`, `p50k_base`, `r50k_base`, `gpt2`.

## `pack()` Pipeline

The `pack()` function in `src/core/packager.ts` runs the following stages, heavily parallelised:

1. **Prefetch git sort data** (background) — git log for `sortByChanges`
2. **Search** — `searchFiles()` per root dir (globby + gitignore + custom patterns)
3. **Sort** — `sortPaths()` deterministic sort
4. **Collect** — `collectFiles()` reads disk concurrently (with `maxFileSize` guard)
5. **Security + Process** — run in parallel:
   - `validateFileSafety()` — secretlint worker threads
   - `processFiles()` — strip comments / compress / line numbers
6. **Output + Metrics** — run in parallel:
   - `produceOutput()` — generate and write output file(s)
   - `calculateMetrics()` — token count via tinypool worker pool
7. Return `PackResult` with file counts, token counts, and security results

**Dependency injection**: All expensive default deps are passed via a `deps` object, allowing full replacement in tests without `vi.mock()`.

## MCP Server

Start with: `repomix --mcp` (stdio transport)

Registered tools:
| Tool | Purpose |
|---|---|
| `pack_codebase` | Pack a local directory into XML |
| `pack_remote_repository` | Clone + pack a GitHub/GitLab URL |
| `generate_skill` | Generate Claude Agent Skill from a codebase |
| `attach_packed_output` | Reference an existing packed file |
| `read_repomix_output` | Read a packed output file |
| `grep_repomix_output` | Search within a packed output |
| `file_system_read_file` | Read an individual file |
| `file_system_read_directory` | List a directory |

## Development Setup

```bash
git clone https://github.com/yamadashy/repomix.git
cd repomix
npm install

# Run the CLI against this repo
npm run repomix

# Run against src + tests only
npm run repomix-src

# Build
npm run build              # tsc → lib/

# Run with source maps
node --enable-source-maps --trace-warnings bin/repomix.cjs
```

### Linting

The project uses four linters chained via `npm run lint`:

| Linter | Config | Purpose |
|---|---|---|
| Biome | `biome.json` | Format + lint (replaces ESLint + Prettier) |
| oxlint | `.oxlintrc.json` | Additional lint rules |
| tsgo | `tsconfig.json` | TypeScript type-check (native preview compiler) |
| secretlint | `.secretlintrc.json` | Scan for leaked secrets |

```bash
npm run lint          # Run all four
npm run lint-biome    # Biome format + lint (--write)
npm run lint-ts       # Type-check only
npm run lint-secretlint
```

### Testing

```bash
npm run test               # Vitest (watch: false, timeout: 15s)
npm run test-coverage      # With V8 coverage
```

Tests live in `tests/` mirroring `src/`. Integration tests in `tests/integration-tests/`.

Vitest config: `vitest.config.ts` — `environment: 'node'`, `globals: true`, `testTimeout: 15000`.

### Performance Benchmarks

```bash
npm run bench              # hyperfine 10-run benchmark
npm run bench:cores        # Core-scaling benchmark (bash script)
npm run memory-check       # Run with --verbose and grep Memory
```

### Website (Docker required)

```bash
npm run website            # Dev server at http://localhost:5173/
```

## Coding Guidelines

- Follow Biome rules (`biome.json`) — auto-applied by `npm run lint-biome`
- **File size limit**: Split files exceeding 250 lines into multiple files by functionality
- **No cross-feature imports**: `src/cli` must not import from `src/mcp`; `src/core` must not import from `src/cli`
- **Dependency injection via `deps`**: Every async function that calls external I/O or spawns workers must accept a `deps` parameter with defaults:

  ```typescript
  export const myFunction = async (
    param1: Type1,
    deps = { readFile, writeFile }
  ) => {
    // use deps.readFile(), never readFile() directly
  };
  ```

- Mock only through the `deps` object in tests; use `vi.mock()` only when DI is infeasible
- Comments in English only; comment the *why*, not the *what*
- Add unit tests for all new features before submitting a PR
- Verify before submitting:

  ```bash
  npm run lint
  npm run test
  ```

## Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/) with scope:

```
type(scope): Description
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`

Scopes: `cli`, `core`, `config`, `mcp`, `security`, `website`, `output`, `treeSitter`, `metrics`

Examples:
```
feat(cli): Add --no-progress flag
fix(security): Handle special characters in file paths
perf(metrics): Overlap output generation with token counting
refactor(core): Split packager into produceOutput module
test(config): Add mergeConfigs edge-case tests
```

For the commit body, follow the `contextual-commit` skill (`.claude/skills/contextual-commit/SKILL.md`).

## Pull Request Guidelines

- For new features or behaviour changes: open or comment on an issue **before** writing code
- Follow `.github/pull_request_template.md`
- Reference issues with `#issue-number`
- Include summary of changes and update `README.md` if functionality changed
- All CI checks (lint, test) must pass

## Architecture Notes

### Parallelism Strategy

`pack()` is carefully pipelined for performance:
- Git sort data is pre-fetched while search/collection is in flight
- `collectFiles` + `getGitDiffs` + `getGitLogs` run concurrently (independent I/O)
- Security check (worker threads) + file processing (main thread) run concurrently
- Output generation + metrics calculation run concurrently
- Metrics worker pool is pre-warmed to overlap `gpt-tokenizer` load time

### Tree-sitter (Code Compression)

When `output.compress: true`, `parseFile()` in `src/core/treeSitter/` uses `web-tree-sitter` (WASM, via `@repomix/tree-sitter-wasms`) to extract signatures and structure, stripping function bodies. This reduces token count significantly for AI consumption.

### Security Scanning

`runSecurityCheck()` uses `@secretlint/core` with the recommended preset to detect API keys, tokens, and other secrets. Suspicious files are excluded from output and reported in `PackResult.suspiciousFilesResults`.

### Output Formats

| Format | Default file | Notes |
|---|---|---|
| `xml` | `repomix-output.xml` | Default; best for AI parsing |
| `markdown` | `repomix-output.md` | Human-readable with code fences |
| `json` | `repomix-output.json` | Structured, machine-parseable |
| `plain` | `repomix-output.txt` | Minimal, no markup |

`parsableStyle: true` enforces strict format compliance (escaped XML, dynamic markdown fences).

### Token Counting

`TokenCounter` in `src/core/metrics/` uses `gpt-tokenizer` with configurable encoding (`o200k_base` default for GPT-4o compatibility). Heavy counting runs in a `tinypool` worker pool to avoid blocking the main thread.
