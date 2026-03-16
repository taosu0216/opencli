# Architecture

> Core module relationships, data flow, and design patterns in opencli.

---

## Module Dependency Graph

```
main.ts
  |-- engine.ts (discoverClis, executeCommand)
  |     |-- registry.ts (CliCommand, cli(), registerCommand, Strategy)
  |     |-- pipeline.ts -> pipeline/executor.ts (executePipeline)
  |     +-- types.ts (IPage)
  |-- registry.ts (getRegistry, fullName, strategyLabel)
  |-- runtime.ts (browserSession, runWithTimeout)
  |-- browser.ts (PlaywrightMCP)
  +-- output.ts (render)

pipeline/executor.ts
  |-- pipeline/steps/browser.ts (stepNavigate, stepClick, ...)
  |-- pipeline/steps/fetch.ts (stepFetch)
  |-- pipeline/steps/transform.ts (stepSelect, stepMap, ...)
  |-- pipeline/steps/intercept.ts (stepIntercept)
  +-- pipeline/steps/tap.ts (stepTap)

browser.ts
  |-- types.ts (IPage)
  |-- runtime.ts (withTimeoutMs)
  |-- interceptor.ts (generateInterceptorJs, generateReadInterceptedJs)
  +-- pipeline/template.ts (normalizeEvaluateSource)
```

## End-to-End Data Flow

The complete path for a user running `opencli twitter search --query "AI"`:

```
1. main.ts: Commander.js parses argv -> finds "twitter search" subcommand
2. main.ts: Coerces args (--query -> kwargs), checks cmd.browser flag
3. main.ts: Calls browserSession(PlaywrightMCP, fn) if browser=true
4. runtime.ts: browserSession() -> new PlaywrightMCP() -> mcp.connect() -> Page
5. main.ts: runWithTimeout(executeCommand(cmd, page, kwargs), timeout)
6. engine.ts: executeCommand() dispatches to cmd.func() or executePipeline()
7. cmd.func(): Site-specific logic runs (navigate, intercept, extract data)
8. main.ts: renderOutput(result, { fmt, columns, title })
9. output.ts: Formats as table/json/csv/md/yaml -> stdout
```

Key code from `src/main.ts` lines 147-162:

```typescript
// main.ts lines 150-156 — dynamic site command action handler
if (cmd.browser) {
  result = await browserSession(PlaywrightMCP, async (page) =>
    runWithTimeout(executeCommand(cmd, page, kwargs, actionOpts.verbose),
      { timeout: cmd.timeoutSeconds ?? DEFAULT_BROWSER_COMMAND_TIMEOUT, label: fullName(cmd) }));
} else {
  result = await executeCommand(cmd, null, kwargs, actionOpts.verbose);
}
```

## CLI Discovery Flow

Two discovery modes, selected automatically:

### Fast Path (Production: manifest)

```
main.ts -> discoverClis(BUILTIN_CLIS, USER_CLIS)
  engine.ts: checks for cli-manifest.json next to clis/ dir
  engine.ts: loadFromManifest() -> JSON.parse -> registerCommand() for each entry
    YAML entries: pipeline inlined in manifest, registered directly
    TS entries: lightweight stub with _lazy=true, _modulePath set
```

### Fallback (Development: filesystem scan)

```
main.ts -> discoverClis(BUILTIN_CLIS, USER_CLIS)
  engine.ts: discoverClisFromFs() -> reads directories under clis/
    .yaml files: registerYamlCli() parses YAML -> registerCommand()
    .js files: dynamic import() -> module calls cli() at load time
```

Lazy loading for TS modules (`src/engine.ts` lines 171-193):

```typescript
// engine.ts lines 172-193
const internal = cmd as InternalCliCommand;
if (internal._lazy && internal._modulePath) {
  const modulePath = internal._modulePath;
  if (!_loadedModules.has(modulePath)) {
    await import(`file://${modulePath}`);
    _loadedModules.add(modulePath);
  }
  // After loading, the module's cli() call will have updated the registry
  const updated = getRegistry().get(fullName(cmd));
  if (updated && updated.func) return updated.func(page!, kwargs, debug);
  if (updated && updated.pipeline) return executePipeline(page, updated.pipeline, { args: kwargs, debug });
}
```

## Build-Time Manifest Compiler

`src/build-manifest.ts` is the compile-time counterpart to `discoverClis()`. The build script runs `node dist/build-manifest.js`, which scans `src/clis/`, inlines YAML pipelines, and emits lightweight metadata for TS adapters:

```typescript
// src/build-manifest.ts lines 45-81
function scanYaml(filePath: string, site: string): ManifestEntry | null {
  // parses YAML and returns an inlined pipeline entry
}

// src/build-manifest.ts lines 88-167
function scanTs(filePath: string, site: string): ManifestEntry {
  // extracts metadata from cli({ ... }) source and stores modulePath
}
```

That build artifact is written to `dist/cli-manifest.json` (`src/build-manifest.ts` lines 171-198). At runtime, `discoverClis()` can therefore register YAML commands without reparsing YAML and register TS commands as lazy stubs that `executeCommand()` resolves on first use.

## Pipeline Executor Architecture

The pipeline executor (`src/pipeline/executor.ts`) uses a **handler registry pattern**. All step handlers conform to the same function signature and are registered in a static `STEP_HANDLERS` record:

```typescript
// src/pipeline/executor.ts lines 18-38
type StepHandler = (page: IPage | null, params: any, data: any, args: Record<string, any>) => Promise<any>;

const STEP_HANDLERS: Record<string, StepHandler> = {
  navigate: stepNavigate,
  fetch: stepFetch,
  select: stepSelect,
  evaluate: stepEvaluate,
  snapshot: stepSnapshot,
  click: stepClick,
  type: stepType,
  wait: stepWait,
  press: stepPress,
  map: stepMap,
  filter: stepFilter,
  sort: stepSort,
  limit: stepLimit,
  intercept: stepIntercept,
  tap: stepTap,
};
```

Pipeline execution is sequential. Each step receives the accumulated `data` from the previous step and returns new data:

```typescript
// src/pipeline/executor.ts lines 40-67
export async function executePipeline(page: IPage | null, pipeline: any[], ctx: PipelineContext = {}): Promise<any> {
  let data: any = null;
  for (let i = 0; i < pipeline.length; i++) {
    const step = pipeline[i];
    if (!step || typeof step !== 'object') continue;
    for (const [op, params] of Object.entries(step)) {
      const handler = STEP_HANDLERS[op];
      if (handler) {
        data = await handler(page, params, data, args);
      }
    }
  }
  return data;
}
```

## Browser Abstraction Layers

Three layers of abstraction separate pipeline steps from the underlying Playwright MCP server:

```
IPage (interface, src/types.ts)
  ^
  |  implements
Page (class, src/browser.ts)
  |  uses
  |  JSON-RPC over stdin/stdout
  v
PlaywrightMCP (class, src/browser.ts)
  |  spawns
  v
@playwright/mcp Node.js process
```

### IPage Interface (17 methods)

Defined in `src/types.ts`, this is the contract that all pipeline steps and CLI adapters program against:

```typescript
// src/types.ts lines 8-26
export interface IPage {
  goto(url: string): Promise<void>;
  evaluate(js: string): Promise<any>;
  snapshot(opts?: { interactive?: boolean; compact?: boolean; maxDepth?: number; raw?: boolean }): Promise<any>;
  click(ref: string): Promise<void>;
  typeText(ref: string, text: string): Promise<void>;
  pressKey(key: string): Promise<void>;
  wait(options: number | { text?: string; time?: number; timeout?: number }): Promise<void>;
  // ... tabs, scroll, interceptor methods
}
```

### Page Class

`Page` wraps a JSON-RPC `_request` function, translating high-level method calls into `tools/call` JSON-RPC messages:

```typescript
// src/browser.ts lines 155-157
async goto(url: string): Promise<void> {
  await this.call('tools/call', { name: 'browser_navigate', arguments: { url } });
}
```

### IBrowserFactory + browserSession()

`IBrowserFactory` is the factory interface (`src/runtime.ts` lines 36-39) that enables dependency injection:

```typescript
export interface IBrowserFactory {
  connect(opts?: { timeout?: number }): Promise<IPage>;
  close(): Promise<void>;
}
```

`browserSession()` manages the connect/close lifecycle:

```typescript
// src/runtime.ts lines 41-52
export async function browserSession<T>(
  BrowserFactory: new () => IBrowserFactory,
  fn: (page: IPage) => Promise<T>,
): Promise<T> {
  const mcp = new BrowserFactory();
  try {
    const page = await mcp.connect({ timeout: DEFAULT_BROWSER_CONNECT_TIMEOUT });
    return await fn(page);
  } finally {
    await mcp.close().catch(() => {});
  }
}
```

## Two CLI Definition Modes

### YAML Declarative Pipelines

YAML files define commands as declarative step sequences. Example from `src/clis/hackernews/top.yaml`:

```yaml
site: hackernews
name: top
description: Hacker News top stories
strategy: public
browser: false
pipeline:
  - fetch:
      url: https://hacker-news.firebaseio.com/v0/topstories.json
  - limit: 30
  - map:
      id: ${{ item }}
  - fetch:
      url: https://hacker-news.firebaseio.com/v0/item/${{ item.id }}.json
  - map:
      rank: ${{ index + 1 }}
      title: ${{ item.title }}
      score: ${{ item.score }}
columns: [rank, title, score, author, comments]
```

### TypeScript Programmatic Adapters

TS files use the `cli()` builder for full programmatic control. Example from `src/clis/bilibili/ranking.ts`:

```typescript
import { cli, Strategy } from '../../registry.js';
import { apiGet } from '../../bilibili.js';

cli({
  site: 'bilibili',
  name: 'ranking',
  strategy: Strategy.COOKIE,
  func: async (page, kwargs) => {
    const payload = await apiGet(page, '/x/web-interface/ranking/v2', { params: { rid: 0, type: 'all' }, signed: false });
    return payload?.data?.list ?? [];
  },
});
```

## Strategy Enum

The `Strategy` enum (`src/registry.ts` lines 7-13) classifies how commands interact with websites:

| Strategy | Value | Browser | Use Case |
|----------|-------|---------|----------|
| `PUBLIC` | `'public'` | No | Public APIs, no auth (e.g., Hacker News) |
| `COOKIE` | `'cookie'` | Yes | Browser cookies for auth (e.g., Bilibili) |
| `HEADER` | `'header'` | Yes | Custom auth headers |
| `INTERCEPT` | `'intercept'` | Yes | XHR/Fetch interception (e.g., Twitter) |
| `UI` | `'ui'` | Yes | UI interaction only (e.g., posting, clicking) |

## Anti-Patterns

- **Do not** import `PlaywrightMCP` directly in pipeline steps or CLI adapters. Always use the `IPage` interface.
- **Do not** bypass `browserSession()` to manage browser lifecycle manually. The try/finally pattern in `browserSession()` ensures cleanup.
- **Do not** add new step types by modifying `executePipeline()` directly. Add a handler function and register it in `STEP_HANDLERS`.
- **Do not** hand-edit `dist/cli-manifest.json` or treat it as source of truth. Source definitions live in `src/clis/` and the manifest is regenerated at build time.
- **Do not** mix YAML and TS definitions for the same command in the same site directory. Use one or the other per command.
- **Do not** import from `./pipeline/executor.js` directly in site adapters. Use `./pipeline.js` for the public re-export.
