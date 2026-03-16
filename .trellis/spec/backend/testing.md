# Testing

> Vitest patterns, mock strategies, test organization, and conventions in opencli.

---

## Test Framework & Configuration

The project uses **Vitest** (v4.1+) with ESM support. Configuration is minimal:

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['src/**/*.test.ts', 'tests/**/*.test.ts'],
  },
});
```

Run tests:
```bash
npm test          # vitest run (single pass)
npm run test:watch  # vitest (watch mode)
```

## Test File Organization

The repo uses three test layers:

1. **Co-located unit tests in `src/`** for pure modules and internal helpers
2. **`tests/e2e/` subprocess tests** for the built CLI (`dist/main.js`)
3. **`tests/smoke/` live API health tests** for scheduled/manual validation

Unit tests are still primarily co-located using the `*.test.ts` suffix:

```
src/engine.ts            -> src/engine.test.ts
src/browser.ts           -> src/browser.test.ts
src/registry.ts          -> src/registry.test.ts
src/interceptor.ts       -> src/interceptor.test.ts
src/output.ts            -> src/output.test.ts
src/doctor.ts            -> src/doctor.test.ts
src/pipeline/executor.ts -> src/pipeline/executor.test.ts
src/pipeline/template.ts -> src/pipeline/template.test.ts
src/pipeline/steps/transform.ts -> src/pipeline/transform.test.ts
```

Higher-level coverage lives outside `src/`:

```text
tests/e2e/helpers.ts              -> subprocess wrapper around `node dist/main.js`
tests/e2e/public-commands.test.ts -> public-command integration coverage
tests/smoke/api-health.test.ts    -> live API health checks + registry validation
```

Note: `transform.test.ts` lives at `src/pipeline/transform.test.ts` (one level up from `steps/transform.ts`). That is the main unit-test exception inside `src/`.

## Import Convention

Tests import from the `.js` extension path, consistent with the ESM convention:

```typescript
// src/engine.test.ts lines 2-3
import { discoverClis, executeCommand } from './engine.js';
import { getRegistry, cli, Strategy } from './registry.js';
```

## Mock Patterns

### IPage Mock (Most Common)

The `createMockPage()` factory function creates a minimal IPage mock with all 17 methods. This is the standard pattern for testing anything that touches the browser:

```typescript
// src/pipeline/executor.test.ts lines 10-31
function createMockPage(overrides: Partial<IPage> = {}): IPage {
  return {
    goto: vi.fn(),
    evaluate: vi.fn().mockResolvedValue(null),
    snapshot: vi.fn().mockResolvedValue(''),
    click: vi.fn(),
    typeText: vi.fn(),
    pressKey: vi.fn(),
    wait: vi.fn(),
    tabs: vi.fn().mockResolvedValue([]),
    closeTab: vi.fn(),
    newTab: vi.fn(),
    selectTab: vi.fn(),
    networkRequests: vi.fn().mockResolvedValue([]),
    consoleMessages: vi.fn().mockResolvedValue(''),
    scroll: vi.fn(),
    autoScroll: vi.fn(),
    installInterceptor: vi.fn(),
    getInterceptedRequests: vi.fn().mockResolvedValue([]),
    ...overrides,
  };
}
```

Usage with specific method overrides:

```typescript
// src/pipeline/executor.test.ts lines 53-61
it('executes evaluate + select pipeline', async () => {
  const page = createMockPage({
    evaluate: vi.fn().mockResolvedValue({ data: { list: [{ name: 'a' }, { name: 'b' }] } }),
  });
  const result = await executePipeline(page, [
    { evaluate: '() => ({ data: { list: [{name: "a"}, {name: "b"}] } })' },
    { select: 'data.list' },
  ]);
  expect(result).toEqual([{ name: 'a' }, { name: 'b' }]);
});
```

### Console/Stderr Mocking

For testing output-producing functions, mock `console.log` or `process.stderr.write`:

```typescript
// src/output.test.ts lines 13-19
it('renders JSON output', () => {
  const log = vi.spyOn(console, 'log').mockImplementation(() => {});
  render([{ title: 'Hello', rank: 1 }], { fmt: 'json' });
  expect(log).toHaveBeenCalledOnce();
  const output = log.mock.calls[0]?.[0];
  const parsed = JSON.parse(output);
  expect(parsed).toEqual([{ title: 'Hello', rank: 1 }]);
});
```

Always restore mocks in `afterEach`:

```typescript
// src/output.test.ts lines 8-10
afterEach(() => {
  vi.restoreAllMocks();
});
```

### Environment Variable Mocking

For testing environment-dependent behavior, save and restore env vars manually:

```typescript
// src/browser.test.ts lines 52-79
it('builds extension MCP args in local mode (no CI)', () => {
  const savedCI = process.env.CI;
  delete process.env.CI;
  try {
    expect(__test__.buildMcpArgs({
      mcpPath: '/tmp/cli.js',
    })).toEqual(['/tmp/cli.js', '--extension']);
  } finally {
    if (savedCI !== undefined) {
      process.env.CI = savedCI;
    } else {
      delete process.env.CI;
    }
  }
});
```

### Testing Internal Functions via __test__

Private functions are exposed through a `__test__` export for testing without making them public:

```typescript
// src/browser.test.ts lines 1-2
import { PlaywrightMCP, __test__ } from './browser.js';

// Then test internals directly
it('creates JSON-RPC requests with unique ids', () => {
  const first = __test__.createJsonRpcRequest('tools/call', { name: 'browser_tabs' });
  const second = __test__.createJsonRpcRequest('tools/call', { name: 'browser_snapshot' });
  expect(second.id).toBe(first.id + 1);
});
```

## Test Patterns by Module Type

### Registry Tests (src/registry.test.ts)

Test the `cli()` builder, strategy defaults, and registry operations:

```typescript
// src/registry.test.ts lines 8-23
it('registers a command and returns it', () => {
  const cmd = cli({
    site: 'test-registry',
    name: 'hello',
    description: 'A test command',
    strategy: Strategy.PUBLIC,
    browser: false,
  });
  expect(cmd.site).toBe('test-registry');
  expect(cmd.strategy).toBe(Strategy.PUBLIC);
  expect(cmd.browser).toBe(false);
  expect(cmd.args).toEqual([]);
});
```

### Engine Tests (src/engine.test.ts)

Test command discovery and execution. Uses `cli()` to register test commands directly:

```typescript
// src/engine.test.ts lines 17-31
it('executes a command with func', async () => {
  const cmd = cli({
    site: 'test-engine',
    name: 'func-test',
    description: 'test command with func',
    browser: false,
    strategy: Strategy.PUBLIC,
    func: async (_page, kwargs) => {
      return [{ title: kwargs.query ?? 'default' }];
    },
  });
  const result = await executeCommand(cmd, null, { query: 'hello' });
  expect(result).toEqual([{ title: 'hello' }]);
});
```

### Pipeline Step Tests (src/pipeline/transform.test.ts)

Test individual step functions directly with sample data. No page mock needed for transform steps:

```typescript
// src/pipeline/transform.test.ts lines 14-19
it('selects nested path', async () => {
  const data = { result: { items: [1, 2, 3] } };
  const result = await stepSelect(null, 'result.items', data, {});
  expect(result).toEqual([1, 2, 3]);
});
```

### Template Engine Tests (src/pipeline/template.test.ts)

Test template rendering with various expression types:

```typescript
// src/pipeline/template.test.ts lines 86-107
describe('render', () => {
  it('renders full expression', () => {
    expect(render('${{ args.limit }}', { args: { limit: 30 } })).toBe(30);
  });
  it('renders inline expression in string', () => {
    expect(render('Hello ${{ item.name }}!', { item: { name: 'World' } })).toBe('Hello World!');
  });
  it('returns non-string values as-is', () => {
    expect(render(42, {})).toBe(42);
    expect(render(null, {})).toBeNull();
  });
});
```

### Code Generator Tests (src/interceptor.test.ts)

Test JavaScript code generators by asserting on the generated source string:

```typescript
// src/interceptor.test.ts lines 9-15
it('generates valid JavaScript function source', () => {
  const js = generateInterceptorJs('"api/search"');
  expect(js).toContain('window.fetch');
  expect(js).toContain('XMLHttpRequest');
  expect(js).toContain('"api/search"');
  expect(js.trim()).toMatch(/^\(\)\s*=>/);
});
```

### State Machine Tests (src/browser.test.ts)

Test PlaywrightMCP state transitions without actually connecting to a browser:

```typescript
// src/browser.test.ts lines 114-146
describe('PlaywrightMCP state', () => {
  it('transitions to closed after close()', async () => {
    const mcp = new PlaywrightMCP();
    expect(mcp.state).toBe('idle');
    await mcp.close();
    expect(mcp.state).toBe('closed');
  });
  it('rejects connect() after the session has been closed', async () => {
    const mcp = new PlaywrightMCP();
    await mcp.close();
    await expect(mcp.connect()).rejects.toThrow('Playwright MCP session is closed');
  });
});
```

### Built-CLI E2E Tests (`tests/e2e/*.test.ts`)

E2E tests run the built CLI as a subprocess instead of importing internal modules directly:

```typescript
// tests/e2e/helpers.ts lines 26-50
export async function runCli(
  args: string[],
  opts: { timeout?: number; env?: Record<string, string> } = {},
): Promise<CliResult> {
  const { stdout, stderr } = await exec('node', [MAIN, ...args], { /* ... */ });
  return { stdout, stderr, code: 0 };
}
```

Example usage:

```typescript
// tests/e2e/public-commands.test.ts lines 11-19
it('hackernews top returns structured data', async () => {
  const { stdout, code } = await runCli(['hackernews', 'top', '--limit', '3', '-f', 'json']);
  expect(code).toBe(0);
  const data = parseJsonOutput(stdout);
  expect(data[0]).toHaveProperty('title');
});
```

### Smoke Tests (`tests/smoke/*.test.ts`)

Smoke tests are reserved for live health checks and registry integrity:

```typescript
// tests/smoke/api-health.test.ts lines 50-54
it('all adapter definitions are valid', async () => {
  const { stdout, code } = await runCli(['validate']);
  expect(code).toBe(0);
  expect(stdout).toContain('PASS');
});
```

## Test Naming Conventions

- Describe blocks use the function/class/module name
- Test names use natural language describing the behavior: `'handles non-existent directories gracefully'`
- Use site prefix `test-*` for test-only registry entries to avoid collisions: `site: 'test-engine'`

## What to Test

| Module Type | What to Test |
|-------------|-------------|
| Pipeline steps | Input/output data transformation with mock data |
| Template engine | Expression parsing, filter application, edge cases |
| Registry | Builder defaults, registration, lookups |
| Engine | Discovery (happy path + missing dirs), execution dispatch |
| Browser | Internal helpers, state machine transitions, error formatting |
| Output | Format rendering (JSON/CSV/MD/YAML/table) |
| Interceptor | Generated JS source contains expected code |
| E2E helpers | Built binary execution, stdout/stderr parsing, exit codes |
| Smoke suite | Live API shape checks, registry integrity, validate command |

## Anti-Patterns

- **Do not** test CLI adapters (files in `src/clis/`) with unit tests. They depend on real websites and are better covered by E2E/smoke tests.
- **Do not** mock module-level side effects like `cli()` calls. Test the `cli()` function directly instead.
- **Do not** use `vi.mock()` for internal modules when you can pass the dependency as a parameter (prefer DI via `IBrowserFactory`).
- **Do not** write tests that depend on test execution order. Each test should be independent.
- **Do not** assert on chalk-colored output directly. Strip ANSI codes first (see `src/doctor.test.ts` line 89: `const strip = (s: string) => s.replace(/\x1b\[[0-9;]*m/g, '');`).
- **Do not** create mock data files. Define test data inline in the test (see `SAMPLE_DATA` in `src/pipeline/transform.test.ts`).
- **Do not** push subprocess or live-network assertions into unit tests. Put built-CLI checks in `tests/e2e/` and flaky/live endpoint checks in `tests/smoke/`.
