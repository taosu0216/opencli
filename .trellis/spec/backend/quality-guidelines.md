# Quality Guidelines

> Code standards, style, linting, forbidden patterns, and output conventions in opencli.

---

## Build & Check Commands

```bash
npm run typecheck    # tsc --noEmit (type checking only)
npm run lint         # same as typecheck (alias)
npm run build        # tsc + copy YAML + build manifest
npm test             # vitest run
npm run test:watch   # vitest (watch mode)
```

The project uses `tsc --noEmit` as its lint command. There is no ESLint, Prettier, or other linter configured. TypeScript itself is the quality gate.

`vitest.config.ts` includes both in-source unit tests and root integration suites:

```typescript
// vitest.config.ts lines 3-6
export default defineConfig({
  test: {
    include: ['src/**/*.test.ts', 'tests/**/*.test.ts'],
  },
});
```

## TypeScript Compiler Settings

From `tsconfig.json`:

- **`strict: false`** -- no `noImplicitAny`, allowing `any` where needed for pipeline data
- **`strictNullChecks: true`** -- null/undefined checks are enforced
- **`target: ES2022`** -- top-level await, optional chaining, nullish coalescing are all available
- **`module: Node16`** -- ESM with `.js` import extensions required

## Code Style

### Semicolons

The project does **not** use semicolons at the end of statements. Statements are separated by newlines:

```typescript
// src/registry.ts -- no trailing semicolons on exports
export function getRegistry(): Map<string, CliCommand> {
  return _registry
}
```

However, note that many files do use semicolons -- the style is mixed. Follow the style of the file you are editing.

### Indentation

2-space indentation throughout. No tabs.

### Line Length

No enforced line length limit. Some lines are quite long (especially in `main.ts` where commands are defined compactly on single lines). Prefer readability over strict line limits.

### Compact Single-Line Patterns

The codebase frequently uses compact single-line patterns for simple logic:

```typescript
// src/main.ts line 54
for (const cmd of commands) { const g = sites.get(cmd.site) ?? []; g.push(cmd); sites.set(cmd.site, g); }

// src/registry.ts line 64
strategy: opts.strategy ?? (opts.browser === false ? Strategy.PUBLIC : Strategy.COOKIE),
```

This is the project's established style for simple operations. Do not expand these into multi-line blocks.

### Import Organization

Imports follow this order (by convention, not enforced):
1. Node.js built-in modules (`node:fs`, `node:path`, `node:os`)
2. Third-party packages (`commander`, `chalk`, `js-yaml`)
3. Local modules (`./registry.js`, `./types.js`)

```typescript
// src/main.ts lines 6-16
import * as os from 'node:os';
import * as path from 'node:path';
import { fileURLToPath } from 'node:url';
import { Command } from 'commander';
import chalk from 'chalk';
import { discoverClis, executeCommand } from './engine.js';
import { type CliCommand, fullName, getRegistry, strategyLabel } from './registry.js';
```

Node built-ins use the `node:` prefix (`node:fs`, `node:path`, not `fs`, `path`).

## Output Conventions

### stdout vs stderr

This separation is critical for piping:

| Channel | Content | API |
|---------|---------|-----|
| **stdout** | Data output (command results) | `console.log()`, `renderOutput()` |
| **stderr** | Diagnostics, warnings, debug info | `process.stderr.write()`, `console.error()` |

```typescript
// src/engine.ts line 87 -- warnings go to stderr
process.stderr.write(`Warning: failed to load manifest ${manifestPath}: ${err.message}\n`);

// src/output.ts line 63 -- data goes to stdout
console.log(JSON.stringify(data, null, 2));
```

### chalk Usage

Use `chalk` for colored terminal output. The project uses these conventions:

```typescript
import chalk from 'chalk';

// Error messages
console.error(chalk.red(`Error: ${err.message}`));
console.error(chalk.red(err.stack));

// Warnings
console.error(chalk.yellow(`[Verbose] Warning: ...`));

// Status labels
chalk.green('[OK]')
chalk.red('[MISSING]')
chalk.yellow('[MISMATCH]')
chalk.yellow('[WARN]')

// Dimmed supplementary text
chalk.dim(`${rows.length} items`)
chalk.dim(` -- ${cmd.description}`)

// Bold headers
chalk.bold('  opencli')
chalk.bold.cyan(`  ${site}`)
```

### Multi-Format Output

The `render()` function in `src/output.ts` supports five output formats, selectable via `--format`:

| Format | Flag | Notes |
|--------|------|-------|
| `table` | `--format table` (default) | cli-table3, with chalk bold headers |
| `json` | `--format json` | Pretty-printed, 2-space indent |
| `csv` | `--format csv` | RFC 4180 quoting for commas and quotes |
| `md` / `markdown` | `--format md` | GitHub-flavored markdown table |
| `yaml` / `yml` | `--format yaml` | js-yaml dump, 120-char line width |

When adding new commands, always set the `columns` field to control which fields appear and in what order.

## ESM Module Conventions

### .js Extensions in Imports

All relative imports must use `.js` extensions. This is required by Node16 module resolution:

```typescript
// CORRECT
import { executePipeline } from './pipeline.js';
import type { IPage } from '../types.js';

// WRONG -- will fail at runtime
import { executePipeline } from './pipeline';
```

### Top-Level Await

Top-level `await` is used in `main.ts` for CLI discovery:

```typescript
// src/main.ts line 23
await discoverClis(BUILTIN_CLIS, USER_CLIS);
```

This is valid because `target: ES2022` and `module: Node16` with `"type": "module"`.

### Dynamic Imports

Heavy modules are loaded lazily via dynamic `import()` to keep startup fast:

```typescript
// src/main.ts line 65 -- validate loaded only when subcommand is used
.action(async (target) => {
  const { validateClisWithTarget, renderValidationReport } = await import('./validate.js');
  // ...
});
```

## Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `OPENCLI_BROWSER_CONNECT_TIMEOUT` | `30` | Browser connection timeout (seconds) |
| `OPENCLI_BROWSER_COMMAND_TIMEOUT` | `45` | Command execution timeout (seconds) |
| `OPENCLI_BROWSER_EXPLORE_TIMEOUT` | `120` | URL exploration timeout (seconds) |
| `OPENCLI_BROWSER_SMOKE_TIMEOUT` | `60` | Smoke test timeout (seconds) |
| `OPENCLI_VERBOSE` | unset | Enable verbose output when set to `'1'` |
| `OPENCLI_MCP_SERVER_PATH` | unset | Override Playwright MCP server path |
| `OPENCLI_BROWSER_EXECUTABLE_PATH` | unset | Custom Chrome executable path |
| `PLAYWRIGHT_MCP_EXTENSION_TOKEN` | unset | Auth token for Playwright MCP Bridge |
| `DEBUG` | unset | Include `opencli:mcp` for MCP debug logging |

## Forbidden Patterns

### Never Use Playwright Directly

All browser interaction must go through the `IPage` interface. Never import Playwright APIs directly in pipeline steps or CLI adapters:

```typescript
// WRONG
import { chromium } from 'playwright';
const browser = await chromium.launch();

// CORRECT
func: async (page: IPage, kwargs) => {
  await page.goto('https://example.com');
  const data = await page.evaluate('() => document.title');
}
```

### Never Hardcode Timeouts

Use the constants from `src/runtime.ts`:

```typescript
// WRONG
setTimeout(() => reject(new Error('timeout')), 30000);

// CORRECT
import { DEFAULT_BROWSER_COMMAND_TIMEOUT, runWithTimeout } from './runtime.js';
runWithTimeout(promise, { timeout: DEFAULT_BROWSER_COMMAND_TIMEOUT, label: 'my-op' });
```

### Never Log Data to stderr

Data output (command results) must go to stdout. Diagnostics go to stderr:

```typescript
// WRONG -- data on stderr
console.error(JSON.stringify(results));

// CORRECT -- data on stdout
console.log(JSON.stringify(results, null, 2));
```

### Never Use console.log for Debug Output

```typescript
// WRONG
console.log('Debug: processing step', stepName);

// CORRECT
if (debug) process.stderr.write(`  ${chalk.dim(`[${stepNum}/${total}]`)} ${chalk.bold.cyan(op)}\n`);
```

### Never Modify Global State in Pipeline Steps

Pipeline steps should be pure functions of their inputs (page, params, data, args). They should not modify global variables or set environment variables.

### Never Ship Core Changes Without the Right Test Layer

The repo uses mixed test layers. Prefer a co-located `src/**/*.test.ts` when the behavior is unit-testable, and use `tests/e2e/` or `tests/smoke/` when the behavior must exercise the built CLI, subprocess boundaries, or live endpoints.

## Pre-Commit Checklist

Before considering code ready:

1. `npm run typecheck` passes (no TypeScript errors)
2. `npm test` passes (all Vitest tests green)
3. All imports use `.js` extensions
4. Data output goes to stdout, diagnostics to stderr
5. No hardcoded timeout values
6. New pipeline steps are registered in `STEP_HANDLERS`
7. New CLI adapters use `IPage` interface, not concrete `Page` class
8. New behavior is covered by the appropriate test layer (`src/**/*.test.ts`, `tests/e2e`, or `tests/smoke`)
