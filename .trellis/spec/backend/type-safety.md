# Type Safety

> TypeScript conventions, key interfaces, and type patterns in opencli.

---

## TypeScript Configuration

From `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "dist",
    "rootDir": "src",
    "strict": false,
    "strictNullChecks": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "incremental": true
  }
}
```

Key settings:
- **`strict: false`** but **`strictNullChecks: true`** -- the project enforces null safety without the full strict suite (no `noImplicitAny`, no `strictBindCallApply`, etc.)
- **`module: Node16`** with **ESM** (`"type": "module"` in package.json) -- requires `.js` extensions in import paths
- **`declaration: true`** -- generates `.d.ts` files in `dist/`

## ESM Import Convention

All imports must use `.js` extensions, even when importing `.ts` files. This is required by Node16 module resolution with ESM:

```typescript
// CORRECT -- .js extension required
import { type CliCommand, Strategy, registerCommand } from './registry.js';
import type { IPage } from './types.js';
import { executePipeline } from './pipeline.js';
import { render } from '../template.js';

// WRONG -- will fail at runtime
import { Strategy } from './registry';
import type { IPage } from './types';
```

Relative imports from CLI adapters use `../../` to reach core modules:

```typescript
// src/clis/twitter/search.ts line 1
import { cli, Strategy } from '../../registry.js';
```

## Key Interfaces

### IPage (src/types.ts)

The primary browser abstraction. All pipeline steps and CLI adapters should use this interface instead of concrete `Page` or `PlaywrightMCP` types:

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
  tabs(): Promise<any>;
  closeTab(index?: number): Promise<void>;
  newTab(): Promise<void>;
  selectTab(index: number): Promise<void>;
  networkRequests(includeStatic?: boolean): Promise<any>;
  consoleMessages(level?: string): Promise<any>;
  scroll(direction?: string, amount?: number): Promise<void>;
  autoScroll(options?: { times?: number; delayMs?: number }): Promise<void>;
  installInterceptor(pattern: string): Promise<void>;
  getInterceptedRequests(): Promise<any[]>;
}
```

Note: Many return types are `Promise<any>` because the MCP server returns varied structures (strings, JSON objects, markdown text). This is intentional -- MCP responses are inherently untyped.

### CliCommand & CliOptions (src/registry.ts)

`CliCommand` is the registered command descriptor. `CliOptions` is the input to the `cli()` builder:

```typescript
// src/registry.ts lines 15-37
export interface Arg {
  name: string;
  type?: string;       // 'str' | 'int' | 'float' | 'bool' | 'string'
  default?: any;
  required?: boolean;
  help?: string;
  choices?: string[];
}

export interface CliCommand {
  site: string;
  name: string;
  description: string;
  domain?: string;
  strategy?: Strategy;
  browser?: boolean;
  args: Arg[];
  columns?: string[];
  func?: (page: IPage, kwargs: Record<string, any>, debug?: boolean) => Promise<any>;
  pipeline?: any[];
  timeoutSeconds?: number;
  source?: string;
}
```

`InternalCliCommand` extends `CliCommand` with lazy-loading metadata (not in public API):

```typescript
// src/registry.ts lines 39-43
export interface InternalCliCommand extends CliCommand {
  _lazy?: boolean;
  _modulePath?: string;
}
```

### IBrowserFactory (src/runtime.ts)

Factory interface for browser creation, enabling dependency injection in tests:

```typescript
// src/runtime.ts lines 36-39
export interface IBrowserFactory {
  connect(opts?: { timeout?: number }): Promise<IPage>;
  close(): Promise<void>;
}
```

### PipelineContext (src/pipeline/executor.ts)

Context object passed to the pipeline executor:

```typescript
// src/pipeline/executor.ts lines 13-16
export interface PipelineContext {
  args?: Record<string, any>;
  debug?: boolean;
}
```

### StepHandler (src/pipeline/executor.ts)

The function type that all pipeline step handlers conform to:

```typescript
// src/pipeline/executor.ts line 19
type StepHandler = (page: IPage | null, params: any, data: any, args: Record<string, any>) => Promise<any>;
```

### ManifestEntry (src/build-manifest.ts)

The manifest compiler keeps its own private build-time shape instead of reusing `CliCommand`. This is intentional: the manifest needs serialized metadata for both YAML and lazy TS adapters.

```typescript
// src/build-manifest.ts lines 21-42
interface ManifestEntry {
  site: string;
  name: string;
  description: string;
  domain?: string;
  strategy: string;
  browser: boolean;
  args: Array<{ name: string; type?: string; default?: any; required?: boolean; help?: string; choices?: string[] }>;
  columns?: string[];
  pipeline?: any[];
  timeout?: number;
  type: 'yaml' | 'ts';
  modulePath?: string;
}
```

Use `CliCommand` for runtime registry entries and `ManifestEntry` only inside `src/build-manifest.ts`.

### RenderContext (src/pipeline/template.ts)

Context for template expression rendering:

```typescript
// src/pipeline/template.ts lines 6-11
export interface RenderContext {
  args?: Record<string, any>;
  data?: any;
  item?: any;
  index?: number;
}
```

### RenderOptions (src/output.ts)

Options for the output rendering function:

```typescript
// src/output.ts lines 9-15
export interface RenderOptions {
  fmt?: string;
  columns?: string[];
  title?: string;
  elapsed?: number;
  source?: string;
}
```

## Strategy Enum

```typescript
// src/registry.ts lines 7-13
export enum Strategy {
  PUBLIC = 'public',
  COOKIE = 'cookie',
  HEADER = 'header',
  INTERCEPT = 'intercept',
  UI = 'ui',
}
```

The Strategy enum is used as a string enum (values are lowercase strings). It determines both `browser` flag defaults and data-fetching approach:

```typescript
// src/registry.ts lines 64-66 -- strategy/browser defaults in cli() builder
strategy: opts.strategy ?? (opts.browser === false ? Strategy.PUBLIC : Strategy.COOKIE),
browser: opts.browser ?? (opts.strategy === Strategy.PUBLIC ? false : true),
```

## When `any` Is Intentional

Pipeline data is intentionally typed as `any` throughout the pipeline. This is by design because:

1. Pipeline steps transform data through arbitrary shapes (JSON API responses, DOM snapshots, user-defined mappings)
2. The `${{ expr }}` template engine evaluates expressions at runtime
3. MCP server responses are inherently unstructured

Places where `any` is correct:
- `executePipeline()` return type: `Promise<any>`
- `StepHandler` params/data: `any`
- `CliCommand.func` return: `Promise<any>`
- `CliCommand.pipeline`: `any[]`
- `Arg.default`: `any`

## Type Import Convention

Use `type` imports when importing only types (enforced by convention, not compiler flag):

```typescript
// src/engine.ts line 15
import type { IPage } from './types.js';

// src/pipeline/executor.ts line 6
import type { IPage } from '../types.js';

// Mixed import: types + values from same module
import { type CliCommand, type InternalCliCommand, type Arg, Strategy, registerCommand } from './registry.js';
```

## Exposing Test-Only Internals

When internal functions need to be tested but should not be part of the public API, export them via a `__test__` object:

```typescript
// src/browser.ts lines 628-635
export const __test__ = {
  createJsonRpcRequest,
  extractTabEntries,
  diffTabIndexes,
  appendLimited,
  buildMcpArgs,
  withTimeoutMs,
};
```

Then in tests:

```typescript
// src/browser.test.ts line 2
import { PlaywrightMCP, __test__ } from './browser.js';

it('creates JSON-RPC requests with unique ids', () => {
  const first = __test__.createJsonRpcRequest('tools/call', { name: 'browser_tabs' });
  // ...
});
```

## Anti-Patterns

- **Do not** use `@ts-ignore` or `@ts-expect-error` to suppress type errors. Fix the types or use an explicit `any` with a comment explaining why.
- **Do not** import concrete `Page` or `PlaywrightMCP` types in pipeline steps or CLI adapters. Use `IPage`.
- **Do not** omit `.js` extensions in imports. The build will succeed but runtime will fail.
- **Do not** add `noImplicitAny` to tsconfig -- the project intentionally uses `any` for pipeline data flow. Adding it would require extensive refactoring with no practical benefit.
- **Do not** use `as` type assertions to cast MCP responses to specific types. The response shape can change between MCP versions; handle gracefully with optional chaining instead.
- **Do not** define new interfaces in CLI adapter files. Core types belong in `src/types.ts`, `src/registry.ts`, or `src/runtime.ts`.
- **Do not** export build-only shapes such as `ManifestEntry` into runtime modules. Keep manifest serialization concerns isolated to `src/build-manifest.ts`.
