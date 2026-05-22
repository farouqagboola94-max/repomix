---
description: Core project guidelines for the Repomix codebase. Apply these rules when working on any code, documentation, or configuration files within the Repomix project.
alwaysApply: true
---

# Repomix

Repomix (v1.14.0) packs an entire repository's contents into a single AI-friendly file. It is used by AI tools (Claude, ChatGPT, Gemini, etc.) to ingest codebases for code review, documentation, debugging, and security analysis. Output formats: XML (default), Markdown, JSON, plain text.

Full project overview → `README.md` · Contribution procedures → `CONTRIBUTING.md` · AI-assistant overview → `repomix-instruction.md`

---

## Directory Structure

```
repomix/
├── bin/                         # Compiled CJS CLI entry point (repomix.cjs)
├── browser/                     # Browser extension (separate npm workspace)
│   ├── entrypoints/             # WXT entry points (content, background, popup)
│   ├── public/                  # Static assets
│   ├── tests/                   # Extension tests
│   ├── wxt.config.ts            # WXT bundler config (Chrome/Firefox/Edge)
│   └── CLAUDE.md                # Browser-specific AI guidelines
├── src/
│   ├── cli/
│   │   ├── actions/             # Command implementations
│   │   │   ├── defaultAction.ts # Main pack action; builds + merges config, calls pack()
│   │   │   ├── remoteAction.ts  # Clone remote repo, then delegates to defaultAction
│   │   │   ├── initAction.ts    # Scaffold repomix.config.json interactively
│   │   │   └── migrationAction.ts # Auto-migrate old config formats
│   │   ├── prompts/             # @clack/prompts interactive flows (skill location, init)
│   │   ├── reporters/           # CLI output reporters
│   │   ├── cliRun.ts            # Entry: commander setup, dispatches to actions
│   │   ├── cliReport.ts         # Formats and prints pack result summary to terminal
│   │   ├── cliSpinner.ts        # picospinner integration
│   │   └── types.ts             # CliOptions type (all CLI flags as typed fields)
│   ├── config/
│   │   ├── configLoad.ts        # Loads file config (json/ts/rc), merges with CLI + defaults
│   │   ├── configSchema.ts      # Valibot schemas: Base/File/CLI/Default/Merged + defineConfig()
│   │   ├── defaultIgnore.ts     # Built-in ignore patterns (node_modules, .git, lock files, …)
│   │   └── globalDirectory.ts   # XDG-compliant global config directory path
│   ├── core/
│   │   ├── file/
│   │   │   ├── fileSearch.ts    # Glob search with globby; respects .gitignore + custom patterns
│   │   │   ├── fileCollect.ts   # Read file contents; enforces maxFileSize; detects binary/encoding
│   │   │   ├── fileProcess.ts   # Apply transforms: removeComments, removeEmptyLines, compress
│   │   │   ├── filePathSort.ts  # Deterministic path sort (sortPaths)
│   │   │   ├── fileTreeGenerate.ts # Build directory tree string from file list
│   │   │   ├── fileStdin.ts     # Read file paths from stdin (--stdin mode)
│   │   │   ├── fileTypes.ts     # ProcessedFile, RawFile types
│   │   │   └── packageJsonParse.ts # Read package.json version for MCP server
│   │   ├── git/
│   │   │   ├── gitDiffHandle.ts # Run `git diff` for output.git.includeDiffs
│   │   │   ├── gitLogHandle.ts  # Run `git log` for output.git.includeLogs
│   │   │   └── gitRemoteParse.ts # Parse remote URL/shorthand for remoteAction
│   │   ├── metrics/
│   │   │   ├── calculateMetrics.ts  # Orchestrates char + token count; uses worker pool
│   │   │   ├── TokenCounter.ts      # Public API: count tokens by encoding
│   │   │   ├── tokenEncodings.ts    # Supported encoding list (TOKEN_ENCODINGS)
│   │   │   └── (workers)            # tinypool worker files for parallel tokenisation
│   │   ├── output/
│   │   │   ├── outputGenerate.ts    # Main output assembler; calls style decorators
│   │   │   ├── outputStyleDecorate.ts # Wrap files per style (XML tags, MD fences, etc.)
│   │   │   ├── outputStyleUtils.ts  # Shared style helpers (fences, escaping)
│   │   │   ├── outputSort.ts        # Sort processed files by git-change frequency
│   │   │   ├── outputSplit.ts       # Split output into N files (--split-output)
│   │   │   ├── outputGeneratorTypes.ts # OutputGeneratorContext type
│   │   │   └── outputStyles/        # Per-format generators: xml, markdown, json, plain
│   │   ├── packager/
│   │   │   └── produceOutput.ts     # Write output file(s), clipboard, stdout; returns outputForMetrics
│   │   ├── security/
│   │   │   ├── securityCheck.ts     # secretlint scan → SuspiciousFileResult[]
│   │   │   └── validateFileSafety.ts # Run securityCheck + filter git diff/log results
│   │   ├── skill/
│   │   │   ├── packSkill.ts         # Generate Claude Agent Skill from codebase (--skill-generate)
│   │   │   └── skillUtils.ts        # generateDefaultSkillName and helpers
│   │   ├── tokenCount/
│   │   │   └── (encoding utilities) # Token encoding abstractions
│   │   ├── treeSitter/
│   │   │   ├── languageConfig.ts    # LANGUAGE_CONFIGS registry (16 languages)
│   │   │   ├── languageParser.ts    # Load WASM + parse file AST
│   │   │   ├── loadLanguage.ts      # WASM path resolution + setWasmBasePath()
│   │   │   ├── parseFile.ts         # parseFile() → compressed representation
│   │   │   ├── parseStrategies/     # Per-language strategies: TypeScript, Python, Go, CSS, Vue, Default
│   │   │   └── queries/             # Tree-sitter query strings per language (queryTs, queryPy, …)
│   │   └── packager.ts              # pack() — top-level orchestrator (see Pipeline section)
│   ├── mcp/
│   │   ├── tools/                   # One file per MCP tool registration
│   │   ├── prompts/                 # MCP prompt registrations
│   │   └── mcpServer.ts             # createMcpServer() / runMcpServer() — stdio transport
│   ├── shared/
│   │   ├── asyncMap.ts              # Bounded-concurrency async map
│   │   ├── errorHandle.ts           # RepomixError class + rethrow helpers
│   │   ├── logger.ts                # Levelled logger (trace/debug/info/warn/error)
│   │   ├── memoryUtils.ts           # logMemoryUsage / withMemoryLogging
│   │   ├── patternUtils.ts          # splitPatterns (comma/space-delimited glob lists)
│   │   ├── processConcurrency.ts    # tinypool worker pool management
│   │   ├── sizeParse.ts             # Parse human-readable sizes ("50MB" → bytes)
│   │   ├── unifiedWorker.ts         # Unified worker handler (for bundled/browser environments)
│   │   └── types.ts                 # RepomixProgressCallback type
│   ├── types/                       # Global TypeScript ambient declarations
│   └── index.ts                     # Public library API (all named exports)
├── tests/
│   ├── cli/
│   ├── config/
│   ├── core/                        # Mirrors src/core/ subdirectory structure
│   ├── integration-tests/           # End-to-end CLI and pack() tests
│   ├── shared/
│   └── testing/                     # Shared test utilities and fixtures
├── website/
│   ├── client/                      # VitePress docs site (Vue.js, markdown in en/ja/…)
│   └── server/                      # Backend API for remote repository processing
├── repomix.config.json              # Self-packaging config (used by npm run repomix)
├── repomix-instruction.md           # Structural overview for AI code assistants
├── biome.json                       # Biome linter + formatter config
├── vitest.config.ts                 # Vitest config
└── .claude/
    ├── settings.json                # { "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
    ├── skills/                      # Claude Agent Skills
    ├── plugins/                     # Claude Code plugins
    ├── agents -> ../.agents/        # Symlink
    └── commands -> ../.claude-plugin/commands/  # Symlink
```

---

## Key Entry Points

| Path | Export / Purpose |
|---|---|
| `bin/repomix.cjs` | Installed CLI binary |
| `src/cli/cliRun.ts` | `run()` / `runCli()` — commander setup, dispatches actions |
| `src/cli/actions/defaultAction.ts` | `runDefaultAction()` + `buildCliConfig()` |
| `src/core/packager.ts` | `pack()` — main programmatic API |
| `src/mcp/mcpServer.ts` | `runMcpServer()` — starts MCP stdio server |
| `src/index.ts` | All public library exports |

---

## Public API (`src/index.ts`)

```typescript
// Core
import { pack } from 'repomix';                       // PackResult
import { collectFiles, processFiles, searchFiles,
         sortPaths, generateFileTree } from 'repomix';

// Git
import { parseRemoteValue, isValidRemoteValue } from 'repomix';

// Security
import { runSecurityCheck } from 'repomix';           // SuspiciousFileResult[]

// Metrics
import { TokenCounter } from 'repomix';

// Tree-sitter
import { parseFile, setWasmBasePath } from 'repomix';

// Config
import { loadFileConfig, mergeConfigs,
         defineConfig, defaultIgnoreList } from 'repomix';
import type { RepomixConfig } from 'repomix';          // = RepomixConfigFile

// CLI
import { runCli, runDefaultAction,
         runRemoteAction, runInitAction,
         buildCliConfig } from 'repomix';
import type { CliOptions } from 'repomix';

// Worker (bundled environments)
import { unifiedWorkerHandler,
         unifiedWorkerTermination } from 'repomix';
```

---

## Configuration System

Config is three layers merged via `mergeConfigs(cwd, fileConfig, cliConfig)`. All fields are optional except `cwd` (injected by mergeConfigs).

**Config file discovery order**: `repomix.config.json` → `.repomixrc` → `repomix.config.ts` (in `cwd`). Custom path via `--config`.

**Schema**: Valibot — `repomixConfigMergedSchema = intersect([defaultSchema, fileSchema, cliSchema])`.

### All Config Options with Defaults

```jsonc
{
  "input": {
    "maxFileSize": 52428800          // 50 MB; files larger are skipped
  },
  "output": {
    "filePath": "repomix-output.xml", // xml|md|txt|json based on style
    "style": "xml",                  // "xml" | "markdown" | "json" | "plain"
    "parsableStyle": false,           // strict format compliance (escaped XML, dynamic fences)
    "headerText": "",                // prepended to output header
    "instructionFilePath": "",       // path to custom instruction file embedded in output
    "fileSummary": true,             // top-files token summary table
    "directoryStructure": true,      // directory tree in output
    "files": true,                   // include file contents (set false for structure-only)
    "removeComments": false,         // strip code comments
    "removeEmptyLines": false,       // strip blank lines
    "compress": false,               // Tree-sitter AST compression (signatures only)
    "showLineNumbers": false,
    "truncateBase64": false,         // truncate embedded base64 blobs
    "copyToClipboard": false,        // also copy output to clipboard
    "includeEmptyDirectories": false,
    "includeFullDirectoryStructure": false, // show ALL dirs, not just included files
    "splitOutput": null,             // integer N → split into N files
    "tokenCountTree": false,         // true|number|string → show token counts per dir in tree
    "git": {
      "sortByChanges": true,         // sort files by git-change frequency (hottest first)
      "sortByChangesMaxCommits": 100,
      "includeDiffs": false,         // append `git diff` output
      "includeLogs": false,          // append `git log` output
      "includeLogsCount": 50
    }
  },
  "include": [],                     // glob patterns to include (empty = all)
  "ignore": {
    "useGitignore": true,
    "useDotIgnore": true,            // also honour .repomixignore
    "useDefaultPatterns": true,      // built-in ignore list in defaultIgnore.ts
    "customPatterns": []             // additional gitignore-style patterns
  },
  "security": {
    "enableSecurityCheck": true      // secretlint scan; suspicious files excluded
  },
  "tokenCount": {
    "encoding": "o200k_base"         // gpt-tokenizer encoding
  }
}
```

**Token encodings**: `o200k_base` (GPT-4o, default) · `cl100k_base` (GPT-4/3.5) · `p50k_base` · `r50k_base` · `gpt2`

### CLI-Only Flags

| Flag | Type | Purpose |
|---|---|---|
| `--stdout` / `-` | bool | Write to stdout instead of file |
| `--skill-generate [name]` | str\|bool | Generate Claude Agent Skill |
| `--skill-output <path>` | str | Non-interactive skill output path |
| `--skill-project-name` | str | Override project name in skill |
| `--skill-source-url` | str | Source URL embedded in skill |
| `--stdin` | bool | Read file paths from stdin |
| `--split-output <n>` | int | Split into N output files |
| `--config <path>` | str | Custom config file path |
| `--skip-local-config` | bool | Ignore local repomix.config.json |

**`--no-*` flag behaviour** (Commander.js): Flags like `--no-gitignore`, `--no-default-patterns`, `--no-file-summary` only take effect when explicitly passed as `false`. When omitted, Commander sets the value to `true`, but `buildCliConfig()` only writes it when `=== false`, letting the config file retain control.

### Conflicting Option Pairs

| Option A | Option B | Reason |
|---|---|---|
| `--split-output` | `--stdout` | Requires filesystem |
| `--split-output` | `--skill-generate` | Skill output is a directory |
| `--split-output` | `--copy` | Multiple files can't go to clipboard |
| `--skill-generate` | `--stdout` | Skill output requires filesystem |
| `--skill-generate` | `--copy` | Directory can't be clipboard-copied |

---

## `pack()` Pipeline

`src/core/packager.ts` — all stages run with maximum parallelism:

```
pack(rootDirs, config, progressCallback, deps?, explicitFiles?, options?)
│
├─ [background] prefetchSortData()     ← git log for sortByChanges
│
├─ [parallel per rootDir] searchFiles()
│       globby + gitignore + .repomixignore + customPatterns
│
├─ sortPaths()                         ← deterministic sort
│
├─ createMetricsTaskRunner()           ← pre-warm tinypool workers (overlaps next stages)
│
├─ [parallel]
│   ├─ collectFiles()                  ← read disk; maxFileSize guard; encoding detection
│   ├─ getGitDiffs()                   ← spawn git subprocess
│   └─ getGitLogs()                    ← spawn git subprocess
│
├─ [parallel]
│   ├─ validateFileSafety()            ← secretlint workers → suspiciousFilesResults
│   └─ processFiles()                  ← removeComments / compress / showLineNumbers
│
├─ sortOutputFiles()                   ← sort by git-change freq (cache hit from prefetch)
│
├─ [early return if --skill-generate]
│   └─ packSkill()                     ← generate Claude Agent Skill, return PackResult
│
├─ [parallel]
│   ├─ produceOutput()                 ← generateOutput() → write file / clipboard / stdout
│   └─ calculateMetrics()             ← char + token counts via worker pool
│
└─ return PackResult
```

**Dependency injection**: All deps default-valued in `defaultDeps`. Pass overrides via `overrideDeps` in tests — no `vi.mock()` needed.

**`packSkill`** is lazy-imported (`await import('./skill/packSkill.js')`) to save ~25ms on startup when not used.

---

## Tree-sitter Code Compression

When `output.compress: true`, `processFiles()` calls `parseFile()` which uses `web-tree-sitter` (WASM via `@repomix/tree-sitter-wasms`) to extract signatures only — stripping function bodies to reduce token count dramatically.

### Supported Languages (16)

| Language | Extensions | Parse Strategy |
|---|---|---|
| TypeScript | `ts tsx mts mtsx cts` | TypeScriptParseStrategy |
| JavaScript | `js jsx cjs mjs mjsx` | TypeScriptParseStrategy |
| Python | `py` | PythonParseStrategy |
| Go | `go` | GoParseStrategy |
| Vue | `vue` | VueParseStrategy |
| CSS | `css` | CssParseStrategy |
| Rust | `rs` | DefaultParseStrategy |
| Java | `java` | DefaultParseStrategy |
| C# | `cs` | DefaultParseStrategy |
| Ruby | `rb` | DefaultParseStrategy |
| PHP | `php` | DefaultParseStrategy |
| Swift | `swift` | DefaultParseStrategy |
| C | `c h` | DefaultParseStrategy |
| C++ | `cpp hpp` | DefaultParseStrategy |
| Solidity | `sol` | DefaultParseStrategy |
| Dart | `dart` | DefaultParseStrategy |

Lookup maps (`extensionToLanguageMap`, `languageNameToConfigMap`) are built lazily on first access. Duplicate extension detection throws at build time.

To add a language: add a `LanguageConfig` entry in `languageConfig.ts`, add a Tree-sitter query file in `queries/`, and create or reuse a `ParseStrategy` in `parseStrategies/`.

---

## Output System

`src/core/output/`:

| File | Role |
|---|---|
| `outputGenerate.ts` | Assembles full output string; calls style decorators |
| `outputStyleDecorate.ts` | Wraps each file block per style (XML tags, MD code fences) |
| `outputStyleUtils.ts` | Shared: fence generation, XML escaping, token-count tree |
| `outputSort.ts` | `sortOutputFiles()` + `prefetchSortData()` — git-change frequency sort |
| `outputSplit.ts` | `splitOutput()` — divides output into N balanced files |
| `outputStyles/` | Per-format generators (xml, markdown, json, plain) |

`src/core/packager/produceOutput.ts` — writes file(s) to disk, handles `--copy` (tinyclip) and `--stdout`.

---

## MCP Server

Start: `repomix --mcp` (stdio transport, `@modelcontextprotocol/sdk`)

| MCP Tool | Function |
|---|---|
| `pack_codebase` | Pack a local directory |
| `pack_remote_repository` | Clone + pack a GitHub/GitLab/Bitbucket URL |
| `generate_skill` | Generate a Claude Agent Skill from a codebase |
| `attach_packed_output` | Reference an already-packed output file |
| `read_repomix_output` | Read a packed output file (chunked) |
| `grep_repomix_output` | Regex search within a packed output |
| `file_system_read_file` | Read any individual file |
| `file_system_read_directory` | List a directory |

Prompts: `pack_remote_repository` — guided prompt for remote repo analysis.

Config: `.claude/settings.json` enables `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` for Claude Code sessions in this repo.

---

## Browser Extension (`browser/`)

Separate npm workspace. **Do not run root `npm install` from `browser/`** — it has its own lockfile.

- **Framework**: WXT (Web Extension Tools) — Manifest V3
- **Targets**: Chrome, Firefox, Edge
- **Function**: Injects a "Repomix" button into GitHub repository pages
- **Languages**: 11 locales (en, ja, de, fr, es, pt-BR, id, vi, ko, zh-CN/TW, hi)

```bash
cd browser
npm install
npm run dev chrome      # Dev mode, Chrome
npm run dev firefox     # Dev mode, Firefox
npm run build-all       # Production build for all browsers
npm run lint            # TypeScript type-check
npm run test            # Vitest tests
npm run generate-icons  # Regenerate icon set from SVG
```

Full browser extension guidelines: `browser/CLAUDE.md`

---

## CI / GitHub Actions Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| `ci.yml` | push/PR to main | Main CI: lint + test on Node 20/22 + OS matrix |
| `ci-browser.yml` | push/PR | Browser extension lint + test |
| `ci-quality.yml` | push/PR | Extra quality gates (typos, secrets, pinact) |
| `ci-website.yml` | push/PR | Website build validation |
| `npm-publish.yml` | release tag | Publish to npm |
| `docker.yml` | release tag | Build + push Docker image |
| `homebrew.yml` | release tag | Trigger Homebrew formula update |
| `benchmark.yml` | push/PR | Run hyperfine benchmarks |
| `perf-benchmark.yml` | manual/push | Detailed perf benchmark with comparison |
| `perf-benchmark-history.yml` | scheduled | Track perf over time |
| `claude.yml` | issue/PR comment | Claude Code action integration |
| `claude-code-review.yml` | PR opened | Automated Claude code review |
| `claude-issue-triage.yml` | issue opened | Claude issue classification |
| `claude-issue-similar.yml` | issue opened | Find similar existing issues |
| `codeql.yml` | push/schedule | GitHub CodeQL security analysis |
| `pack-repository.yml` | push | Self-pack the repo (repomix output artifact) |
| `schema-update.yml` | release | Update JSON schema at repomix.com |
| `autofix.yml` | PR | Auto-fix lint issues |
| `pinact.yml` | schedule | Pin GitHub Actions to exact SHAs |

---

## Development Setup

```bash
git clone https://github.com/yamadashy/repomix.git
cd repomix
npm install
```

### Run the CLI

```bash
npm run repomix                # Pack this repo (uses repomix.config.json)
npm run repomix-src            # Pack src/ + tests/ only
npm run repomix-website        # Pack website/ only

# Or with source maps:
node --enable-source-maps --trace-warnings bin/repomix.cjs [args]
```

### Self-Packaging Config (`repomix.config.json`)

This repo packs itself with:
- `instructionFilePath: "repomix-instruction.md"` — AI instruction file embedded in output
- `git.includeDiffs: true` + `git.includeLogs: true` — git history included
- `tokenCountTree: 50000` — show token counts for dirs with >50K tokens
- `truncateBase64: true` — truncate base64 data URLs
- `includeEmptyDirectories: true`

### Build

```bash
npm run build              # rimraf lib/ && tsc -p tsconfig.build.json
npm run build-bun          # Bun variant
```

Output goes to `lib/` (ESM + type declarations).

### Linting (4 linters, all run via `npm run lint`)

| Command | Tool | Config | Purpose |
|---|---|---|---|
| `npm run lint-biome` | Biome | `biome.json` | Format + lint + import order (--write) |
| `npm run lint-oxlint` | oxlint | `.oxlintrc.json` | Additional lint rules (--fix) |
| `npm run lint-ts` | tsgo | `tsconfig.json` | Native TypeScript type-check (--noEmit) |
| `npm run lint-secretlint` | secretlint | `.secretlintrc.json` | Detect leaked secrets |

```bash
npm run lint               # All four in sequence
npm run lint-biome         # Biome only (fastest for format)
```

### Testing

```bash
npm run test               # Vitest (watch: false, timeout: 15s, node env)
npm run test-coverage      # With V8 coverage report (text + json + html)
```

Vitest config (`vitest.config.ts`):
- `environment: 'node'`
- `globals: true` (no imports needed for describe/it/expect)
- `include: 'tests/**/*.test.ts'`
- `testTimeout: 15000` (allows for slower integration tests)
- Coverage excludes `src/index.ts` (re-export barrel)

### Performance

```bash
npm run bench              # hyperfine 10-run benchmark (requires hyperfine)
npm run bench:cores        # Core-scaling benchmark script
npm run time-node          # `time node bin/repomix.cjs`
npm run time-bun           # `time bun bin/repomix.cjs`
npm run memory-check       # Verbose run, grep Memory usage lines
npm run memory-check-one-file  # Same, for a single file
```

### Website (Docker required)

```bash
npm run website            # Build + start dev server at http://localhost:5173/
npm run website-bundle     # Bundled build variant
npm run website-generate-schema  # Regenerate JSON schema from Valibot schema
```

### Docker

```bash
docker build -t repomix .
docker run -v ./:/app -it --rm repomix
```

---

## Coding Guidelines

### Structure
- **File size cap**: Split any file exceeding 250 lines by functionality
- **No cross-layer imports**: `cli/` must not import from `mcp/`; `core/` must not import from `cli/` or `mcp/`; `shared/` has no domain imports
- **Feature isolation**: Each `src/core/*/` subdirectory is self-contained; avoid importing across feature directories

### Dependency Injection (mandatory for all I/O)

Every async function that performs I/O or spawns workers must expose a `deps` parameter:

```typescript
const defaultDeps = {
  readFile,
  writeFile,
  someWorker,
};

export const myFunction = async (
  param1: Type1,
  deps: Partial<typeof defaultDeps> = {}
): Promise<Result> => {
  const { readFile, writeFile, someWorker } = { ...defaultDeps, ...deps };
  // use deps, never call defaultDeps functions directly
};
```

- Mock in tests by passing overrides through the `deps` object
- Use `vi.mock()` only when DI is not feasible (e.g., module-level side effects)

### Code Style
- Biome enforces all formatting — do not hand-format; run `npm run lint-biome`
- Comments in English only; explain *why*, never *what*
- TypeScript: prefer explicit return types on exported functions
- No `any` unless unavoidable; prefer `unknown` + type guards

### Tests
- Mirror `src/` directory structure in `tests/`
- Unit test every new exported function
- Integration tests go in `tests/integration-tests/`
- Shared test fixtures and helpers → `tests/testing/`
- Test files: `*.test.ts`

### Before Submitting

```bash
npm run lint    # Must pass (0 errors)
npm run test    # Must pass (all tests green)
```

---

## Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
type(scope): Description
```

**Types**: `feat` · `fix` · `docs` · `style` · `refactor` · `test` · `chore` · `perf`

**Scopes**: `cli` · `core` · `config` · `mcp` · `security` · `output` · `treeSitter` · `metrics` · `browser` · `website`

```
feat(cli): Add --stdin flag for reading file paths from pipe
fix(security): Handle special characters in file paths
perf(metrics): Overlap output generation with token counting
refactor(output): Extract outputSplit into separate module
test(config): Cover mergeConfigs edge cases
docs(mcp): Document generate_skill tool parameters
```

For commit body, follow the `contextual-commit` skill: `.claude/skills/contextual-commit/SKILL.md`

---

## Pull Request Guidelines

- For features or behaviour changes: open/comment on an issue **before** writing code
- Template: `.github/pull_request_template.md`
- Reference issues with `#issue-number`
- Update `README.md` when adding or changing functionality
- All CI checks must pass (lint, test, type-check)
- PRs without prior issue discussion may be closed

---

## Architecture Notes

### Parallelism Strategy

`pack()` overlaps every independent stage:
1. Git sort data prefetched while search + collection are in flight
2. `collectFiles` + `getGitDiffs` + `getGitLogs` run concurrently (pure I/O)
3. Security check (worker threads) + file processing (main thread) run concurrently
4. Metrics worker pool pre-warmed during security/process stages
5. `produceOutput` + `calculateMetrics` run concurrently

Result: on a large repo like `facebook/react`, processing is ~29× faster than single-threaded.

### Security Scanning

`@secretlint/core` with `@secretlint/secretlint-rule-preset-recommend` scans every collected file. Suspicious files (API keys, tokens, credentials) are excluded from output and reported in `PackResult.suspiciousFilesResults`. Git diff and log output are also scanned separately.

### Stdin Mode (`--stdin`)

When `--stdin` is passed, Repomix reads a newline-delimited list of file paths from stdin instead of globbing. Useful for piping from `git diff --name-only`, `find`, or other tools. Only `.` is accepted as the directory argument in this mode.

### Skill Generation (`--skill-generate`)

Generates a Claude Agent Skill directory from the packed codebase. This is an early-return path in `pack()` — metrics are skipped. Interactive prompt asks where to save the skill; use `--skill-output` for non-interactive.

### Output Formats

| Format | Default file | Notes |
|---|---|---|
| `xml` | `repomix-output.xml` | Default; best AI parsing, most widely tested |
| `markdown` | `repomix-output.md` | Human-readable; code fences per file |
| `json` | `repomix-output.json` | Structured; machine-parseable |
| `plain` | `repomix-output.txt` | Minimal; no markup |

`parsableStyle: true` → strictly escaped XML or dynamically sized markdown fences (avoids fence collision).

### Token Counting

`TokenCounter` wraps `gpt-tokenizer`. Heavy bulk counting uses a `tinypool` worker pool pre-warmed during earlier pipeline stages. The `extractOutputWrapper` fast-path walks file contents through the output string in sorted order to avoid double-parsing.
