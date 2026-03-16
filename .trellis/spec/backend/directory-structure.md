# Directory Structure

> Module organization and file layout for the opencli codebase.

---

## Top-Level Layout

```
opencli/
  src/
    main.ts              # CLI entry point (Commander.js program)
    engine.ts            # CLI discovery + command execution
    registry.ts          # CliCommand registry, cli() builder, Strategy enum
    types.ts             # IPage interface (browser abstraction)
    runtime.ts           # IBrowserFactory, browserSession(), timeout utilities
    browser.ts           # Page class, PlaywrightMCP process manager
    interceptor.ts       # XHR/Fetch interception JS generators
    output.ts            # Multi-format output renderer (table/json/csv/md/yaml)
    doctor.ts            # Playwright MCP diagnostics + token management
    setup.ts             # Interactive setup wizard
    validate.ts          # YAML CLI definition validation
    verify.ts            # Validate + smoke test runner
    explore.ts           # URL exploration (discovers APIs via browser)
    synthesize.ts        # Generate CLI YAML from exploration results
    generate.ts          # One-shot: explore -> synthesize -> register
    cascade.ts           # Strategy cascade probing
    tui.ts               # Interactive terminal UI mode
    pipeline.ts          # Re-export shim (backward compat -> pipeline/)
    pipeline/
      index.ts           # Re-exports executePipeline from executor.ts
      executor.ts        # Pipeline runner with STEP_HANDLERS registry
      template.ts        # ${{ expr }} template interpolation engine
      steps/
        browser.ts       # navigate, click, type, wait, press, snapshot, evaluate
        fetch.ts         # HTTP fetch step (single + batch)
        transform.ts     # select, map, filter, sort, limit
        intercept.ts     # Declarative XHR interception step
        tap.ts           # Store-action bridge step (Pinia/Vuex)
    clis/
      {site}/            # One directory per website
        {command}.ts     # TypeScript programmatic adapter
        {command}.yaml   # YAML declarative pipeline
    *.test.ts            # Tests co-located with source files
  tests/
    e2e/                # Built-CLI subprocess coverage (`runCli()` helper)
    smoke/              # Scheduled/manual live API health checks
  tsconfig.json
  vitest.config.ts
  package.json
```

## Conventions

### Flat Core Modules

Core modules live as flat files directly under `src/`. There is no `src/lib/`, `src/utils/`, or `src/core/` directory. Each file is a self-contained module with a clear single responsibility.

```
src/engine.ts      -- discovery logic
src/registry.ts    -- command types + registry
src/runtime.ts     -- timeout + browser session helpers
src/browser.ts     -- Page + PlaywrightMCP classes
src/interceptor.ts -- JS codegen for request interception
src/output.ts      -- output formatting
```

This flat layout keeps imports short and avoids deep nesting:

```typescript
// src/engine.ts line 14
import { type CliCommand, type InternalCliCommand, type Arg, Strategy, registerCommand } from './registry.js';
```

### Pipeline Sub-Module

The only sub-directory for core logic is `src/pipeline/`. It follows a barrel pattern with `index.ts` re-exporting the public API. For backward compatibility, `src/pipeline.ts` re-exports from `src/pipeline/index.js`.

```typescript
// src/pipeline.ts lines 6-8 (backward-compat shim)
export { executePipeline, type PipelineContext } from './pipeline/index.js';
```

The `steps/` directory inside `pipeline/` groups step handlers by concern: browser interactions, HTTP fetching, data transforms, interception, and store tapping.

### CLI Adapter Directories

Site-specific adapters live under `src/clis/{site}/`. Each site gets its own directory. Commands within a site can be either TypeScript files (`.ts`) or YAML pipelines (`.yaml`).

```
src/clis/
  bilibili/
    ranking.ts         # TS adapter using cli() builder
    hot.yaml           # YAML declarative pipeline
  hackernews/
    top.yaml           # Public API, no browser needed
  twitter/
    search.ts          # TS adapter with INTERCEPT strategy
    trending.yaml      # YAML pipeline
  reddit/
    search.yaml        # YAML pipeline with browser
    hot.yaml           # YAML pipeline
```

Command names derive from the filename (minus extension). The site name is the directory name.

### Test File Placement

Most fast unit tests are co-located with their source files using the `*.test.ts` naming convention. The repo also keeps higher-level suites under `tests/` when verification needs the built CLI binary or live integrations.

```
src/
  engine.ts            # Source
  engine.test.ts       # Test
  browser.ts           # Source
  browser.test.ts      # Test
  pipeline/
    executor.ts        # Source
    executor.test.ts   # Test
    template.ts        # Source
    template.test.ts   # Test
    transform.test.ts  # Tests for steps/transform.ts (lives one level up)
tests/
  e2e/
    helpers.ts         # runCli() subprocess wrapper for dist/main.js
    public-commands.test.ts
  smoke/
    api-health.test.ts
```

The `vitest.config.ts` includes both patterns:

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    include: ['src/**/*.test.ts', 'tests/**/*.test.ts'],
  },
});
```

### File Naming Conventions

| Convention | Example | Scope |
|-----------|---------|-------|
| mostly kebab-case files | `build-manifest.ts`, `add-to-cart.ts` | New multi-word files should prefer this |
| legacy camelCase files | `snapshotFormatter.ts` | Existing files may keep established names |
| camelCase exports | `discoverClis`, `executePipeline`, `browserSession` | Functions |
| PascalCase classes | `Page`, `PlaywrightMCP` | Classes |
| PascalCase enums | `Strategy` | Enums |
| PascalCase interfaces | `IPage`, `IBrowserFactory`, `CliCommand` | Interfaces (`I`-prefix for abstractions only) |
| UPPER_SNAKE constants | `DEFAULT_BROWSER_COMMAND_TIMEOUT`, `STEP_HANDLERS` | Module-level constants |

### Build Output

TypeScript compiles to `dist/` via `tsc`. YAML files are copied separately by the build script. The manifest is generated post-build:

```
dist/
  main.js              # Compiled entry point
  clis/                 # Copied YAML + compiled TS adapters
  cli-manifest.json    # Pre-compiled command registry for fast startup
```

## Anti-Patterns

- **Do not** create `src/utils/` or `src/helpers/` directories. Keep utility functions in the module that uses them, or create a dedicated flat file (e.g., `src/interceptor.ts`).
- **Do not** nest core modules deeper than one level. The only exception is `src/pipeline/steps/`.
- **Do not** move ordinary unit tests into a catch-all root folder. Keep unit tests next to source, and reserve `tests/` for subprocess/E2E/smoke coverage.
- **Do not** use `index.ts` barrel files for core modules. Only `src/pipeline/` uses this pattern, for backward compatibility.
- **Do not** name CLI adapter files with the site prefix (e.g., `bilibili-ranking.ts`). The site is already the directory name.
