# Fill Backend Core Spec

## Goal

Analyze the opencli codebase architecture and fill all spec files under `.trellis/spec/backend/` with real code examples, actual patterns, and project-specific anti-patterns.

## Context

**opencli** is a TypeScript CLI tool: "Make any website your CLI. AI-powered." It turns websites into command-line interfaces by automating browsers and intercepting API requests.

### Core Architecture

| Module | File | Purpose |
|--------|------|---------|
| **Entry Point** | `src/main.ts` | Commander.js CLI with subcommands (run, list, doctor, setup, explore, generate) |
| **Registry** | `src/registry.ts` | CLI command registry (`Map<string, CliCommand>`), `cli()` builder function, `Strategy` enum |
| **Engine** | `src/engine.ts` | `discoverClis()` finds CLI defs from YAML+TS, `loadFromManifest()` loads YAML CLIs |
| **Pipeline Executor** | `src/pipeline/executor.ts` | `executePipeline()` runs YAML pipeline steps sequentially via `STEP_HANDLERS` map |
| **Pipeline Steps** | `src/pipeline/steps/` | Individual step handlers: browser.ts, fetch.ts, transform.ts, intercept.ts, tap.ts |
| **Pipeline Template** | `src/pipeline/template.ts` | `${{ expr }}` template interpolation for YAML values |
| **Browser** | `src/browser.ts` | `Page` class (wraps Playwright MCP), `PlaywrightMCP` class (connects to Playwright MCP server) |
| **Types** | `src/types.ts` | `IPage` interface — browser page abstraction with 17 methods |
| **Runtime** | `src/runtime.ts` | `IBrowserFactory` interface, `browserSession()` helper, timeout utilities |
| **Interceptor** | `src/interceptor.ts` | XHR/fetch request interception via browser evaluate |
| **Output** | `src/output.ts` | Table/JSON output formatting with `cli-table3` |
| **TUI** | `src/tui.ts` | Interactive terminal UI mode |
| **Doctor** | `src/doctor.ts` | Health diagnostics (Chrome, Playwright MCP, connectivity checks) |
| **Setup** | `src/setup.ts` | Token management (read/write tokens to shell RC files) |
| **Validate** | `src/validate.ts` | YAML CLI definition validation |
| **Explore** | `src/explore.ts` | URL exploration — navigates a URL, intercepts APIs, builds CLI spec |
| **Synthesize** | `src/synthesize.ts` | Generates CLI YAML from exploration results |
| **Build Manifest** | `src/build-manifest.ts` | Scans YAML files and generates `manifest.json` for runtime discovery |

### Key Design Patterns

1. **Two CLI Definition Modes**: YAML declarative pipelines (`src/clis/*/_.yaml`) and TypeScript programmatic (`src/clis/*/*.ts` using `cli()` builder)
2. **Strategy Enum**: `PUBLIC | COOKIE | HEADER | INTERCEPT | UI` — determines how authentication/data-fetching works
3. **Pipeline Step Handler Pattern**: `STEP_HANDLERS` record maps operation names to handler functions, all conforming to `(page, params, data, args) => Promise<any>`
4. **IPage Abstraction**: Interface decouples pipeline steps from Playwright MCP implementation
5. **IBrowserFactory**: Factory interface enables dependency injection for testing
6. **Template Interpolation**: `${{ expr }}` in YAML values gets eval'd with `item`, `index`, `data`, `args` in scope

### Supported Sites (17)

bilibili, twitter, youtube, hackernews, reddit, coupang, bbc, reuters, v2ex, weibo, xiaohongshu, zhihu, boss, smzdm, yahoo-finance, ctrip, xueqiu

### Tech Stack

- TypeScript (ESM, `"type": "module"`)
- Node.js >= 18
- Commander.js (CLI framework)
- Playwright MCP (browser automation via MCP protocol)
- Vitest (testing)
- chalk (terminal colors)
- cli-table3 (table output)
- js-yaml (YAML parsing)

## Tools Available

You have two MCP servers configured — use both for accurate specs:

### GitNexus MCP (architecture-level: clusters, execution flows, impact)

| Tool | Purpose | Example |
|------|---------|---------|
| `gitnexus_query` | Find execution flows by concept | `gitnexus_query({query: "error handling", repo: "opencli"})` |
| `gitnexus_context` | 360-degree symbol view | `gitnexus_context({name: "executePipeline", repo: "opencli"})` |
| `gitnexus_impact` | Blast radius analysis | `gitnexus_impact({target: "Page", direction: "upstream", repo: "opencli"})` |
| `gitnexus_cypher` | Direct graph queries | `gitnexus_cypher({query: "MATCH (n:Class) RETURN n.name, n.filePath LIMIT 20", repo: "opencli"})` |

### ABCoder MCP (symbol-level: AST nodes, signatures, cross-file deps)

| Tool | Purpose | Example |
|------|---------|---------|
| `get_repo_structure` | Full file listing | `get_repo_structure({repo_name: "opencli"})` |
| `get_file_structure` | All nodes in a file | `get_file_structure({repo_name: "opencli", file_path: "src/engine.ts"})` |
| `get_ast_node` | Code + deps + refs | `get_ast_node({repo_name: "opencli", node_ids: [{mod_path: "@jackwener/opencli", pkg_path: "src/engine.ts", name: "discoverClis"}]})` |

### Recommended Workflow

1. GitNexus first — find relevant execution flows and clusters
2. ABCoder second — get exact code patterns and signatures
3. Read source files — for full context where needed
4. Write specs — with real code examples from steps 2-3

## Files to Fill

### `.trellis/spec/backend/directory-structure.md`
- Document the `src/` layout: core modules, pipeline/, clis/, steps/
- Explain the flat-file convention for core modules
- Document the `src/clis/{site}/{command}.ts|yaml` convention
- Show where tests live (`*.test.ts` alongside source)
- Naming conventions (kebab-case files, camelCase exports)

### `.trellis/spec/backend/architecture.md`
- Core module dependency graph (main → engine → registry → pipeline)
- Pipeline executor architecture (STEP_HANDLERS pattern)
- Browser abstraction layers (IPage → Page → PlaywrightMCP)
- CLI discovery flow (engine scans clis/ dir + manifest.json)
- Data flow: CLI command → browser session → pipeline execution → output formatting
- Read `src/main.ts`, `src/engine.ts`, `src/registry.ts`, `src/pipeline/executor.ts` for details

### `.trellis/spec/backend/type-safety.md`
- Key interfaces: IPage, CliCommand, CliOptions, IBrowserFactory, PipelineContext, Arg, ManifestEntry
- Strategy enum usage patterns
- StepHandler type signature
- When to use `any` vs typed (pipeline data is intentionally `any`)
- Import patterns (`.js` extensions for ESM)
- Read `src/types.ts`, `src/registry.ts`, `src/runtime.ts`, `src/pipeline/executor.ts`

### `.trellis/spec/backend/error-handling.md`
- try/catch patterns in CLI functions (catch and return empty array)
- Timeout handling (`runWithTimeout`, `withTimeoutMs`)
- Browser connection error handling
- Pipeline step error propagation
- Debug mode (`--debug` flag) for verbose error output
- Read `src/runtime.ts`, `src/pipeline/executor.ts`, `src/clis/twitter/search.ts`, `src/doctor.ts`

### `.trellis/spec/backend/testing.md`
- Vitest setup and configuration
- Test file naming (`*.test.ts` beside source)
- Mock patterns for browser/page (IBrowserFactory enables DI)
- Pipeline step testing (unit test each step handler)
- Registry testing patterns
- Read `src/engine.test.ts`, `src/browser.test.ts`, `src/pipeline/executor.test.ts`, `src/interceptor.test.ts`

### `.trellis/spec/backend/quality-guidelines.md`
- TypeScript strict mode settings
- ESM module conventions (`.js` imports)
- Code style (no semicolons? indentation? etc.)
- Forbidden patterns (direct Playwright usage without IPage, hardcoded timeouts, etc.)
- chalk usage for colored output
- process.stderr for debug output, stdout for data
- Read `tsconfig.json`, `package.json` scripts, any linter configs

## Important Rules

### Spec files are NOT fixed — adapt to reality
- Delete template files that don't apply to this project
- Create new files for patterns templates don't cover
- Rename files if template names don't fit
- Update index.md to reflect the final set

### Parallel agents — stay in your lane
- ONLY modify files under `.trellis/spec/backend/`
- DO NOT modify source code, other spec directories, or task files
- DO NOT run git commands
- You may read any file for analysis

## Acceptance Criteria

- [ ] Real code examples from the actual codebase (with file paths and line numbers)
- [ ] Anti-patterns documented (what NOT to do)
- [ ] No placeholder text remaining ("To be filled", template comments)
- [ ] index.md reflects actual file set
- [ ] Each file is at least 80 lines with substantive content

## Technical Notes

- Package: `@jackwener/opencli` v0.6.2
- Language: TypeScript (ESM)
- Build: `tsc` → `dist/`
- Test: `vitest run`
- Node: >= 18.0.0
