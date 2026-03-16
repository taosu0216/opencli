# Error Handling

> Error propagation, try/catch patterns, timeout handling, and debug mode in opencli.

---

## Error Handling Philosophy

opencli follows a **fail-gracefully, log-to-stderr** approach. Since CLI commands interact with external websites and browser processes that can fail unpredictably, the codebase prefers returning empty results over crashing, with detailed error messages available via `--verbose`.

## Top-Level Error Boundary

The main command action handler in `src/main.ts` is the final catch-all. It catches all errors from command execution and displays them:

```typescript
// src/main.ts lines 147-162
subCmd.action(async (actionOpts) => {
  try {
    // ... command execution ...
    if (actionOpts.verbose && (!result || (Array.isArray(result) && result.length === 0))) {
      console.error(chalk.yellow(`[Verbose] Warning: Command returned an empty result. ...`));
    }
    renderOutput(result, { fmt: actionOpts.format, columns: cmd.columns, ... });
  } catch (err: any) {
    if (actionOpts.verbose && err.stack) { console.error(chalk.red(err.stack)); }
    else { console.error(chalk.red(`Error: ${err.message ?? err}`)); }
    process.exitCode = 1;
  }
});
```

Key patterns:
- **Stack trace only in verbose mode** -- hides implementation details from normal users
- **`process.exitCode = 1`** instead of `process.exit(1)` -- allows cleanup to finish
- **`err.message ?? err`** -- handles both Error objects and plain strings

## CLI Adapter Error Pattern

CLI adapters (TS files in `src/clis/`) use try/catch at the data extraction level to skip malformed items while processing the rest:

```typescript
// src/clis/twitter/search.ts lines 32-60
let results: any[] = [];
for (const req of requests) {
  try {
    const insts = req.data.data.search_by_raw_query.search_timeline.timeline.instructions;
    const addEntries = insts.find((i: any) => i.type === 'TimelineAddEntries');
    if (!addEntries) continue;

    for (const entry of addEntries.entries) {
      if (!entry.entryId.startsWith('tweet-')) continue;
      let tweet = entry.content?.itemContent?.tweet_results?.result;
      if (!tweet) continue;
      // ... extract fields ...
      results.push({ id: tweet.rest_id, author: ..., text: ... });
    }
  } catch (e) {
    // ignore parsing errors for individual payloads
  }
}
return results.slice(0, kwargs.limit);
```

This pattern:
- Wraps each response payload in its own try/catch
- Uses `continue` to skip unparseable items
- Returns whatever was successfully extracted (possibly empty array)
- Always returns a valid array, never throws to the caller

## CLI Discovery Warnings

The engine logs warnings to stderr but continues processing other CLIs:

```typescript
// src/engine.ts lines 86-88 (manifest loading failure)
} catch (err: any) {
  process.stderr.write(`Warning: failed to load manifest ${manifestPath}: ${err.message}\n`);
}

// src/engine.ts lines 105-108 (module import failure)
import(`file://${filePath}`).catch((err: any) => {
  process.stderr.write(`Warning: failed to load module ${filePath}: ${err.message}\n`);
})

// src/engine.ts lines 157-159 (YAML parsing failure)
} catch (err: any) {
  process.stderr.write(`Warning: failed to load ${filePath}: ${err.message}\n`);
}
```

Pattern: Use `process.stderr.write()` (not `console.error()`) for warnings during discovery. This keeps stdout clean for data output.

## Timeout Handling

Two timeout utilities in `src/runtime.ts` handle different granularities:

### runWithTimeout (seconds-level, for commands)

```typescript
// src/runtime.ts lines 15-20
export async function runWithTimeout<T>(
  promise: Promise<T>,
  opts: { timeout: number; label?: string },
): Promise<T> {
  return withTimeoutMs(promise, opts.timeout * 1000, `${opts.label ?? 'Operation'} timed out after ${opts.timeout}s`);
}
```

### withTimeoutMs (milliseconds-level, for internal operations)

```typescript
// src/runtime.ts lines 25-33
export function withTimeoutMs<T>(promise: Promise<T>, timeoutMs: number, message: string): Promise<T> {
  return new Promise<T>((resolve, reject) => {
    const timer = setTimeout(() => reject(new Error(message)), timeoutMs);
    promise.then(
      (value) => { clearTimeout(timer); resolve(value); },
      (error) => { clearTimeout(timer); reject(error); },
    );
  });
}
```

### Configurable Timeout Constants

Timeouts are configurable via environment variables with sensible defaults:

```typescript
// src/runtime.ts lines 7-10
export const DEFAULT_BROWSER_CONNECT_TIMEOUT = parseInt(process.env.OPENCLI_BROWSER_CONNECT_TIMEOUT ?? '30', 10);
export const DEFAULT_BROWSER_COMMAND_TIMEOUT = parseInt(process.env.OPENCLI_BROWSER_COMMAND_TIMEOUT ?? '45', 10);
export const DEFAULT_BROWSER_EXPLORE_TIMEOUT = parseInt(process.env.OPENCLI_BROWSER_EXPLORE_TIMEOUT ?? '120', 10);
export const DEFAULT_BROWSER_SMOKE_TIMEOUT = parseInt(process.env.OPENCLI_BROWSER_SMOKE_TIMEOUT ?? '60', 10);
```

Commands can also specify per-command timeouts via `timeoutSeconds`:

```typescript
// src/main.ts line 151
runWithTimeout(executeCommand(cmd, page, kwargs, actionOpts.verbose),
  { timeout: cmd.timeoutSeconds ?? DEFAULT_BROWSER_COMMAND_TIMEOUT, label: fullName(cmd) })
```

## Browser Connection Error Handling

The `PlaywrightMCP.connect()` method uses a structured error classification system with descriptive, actionable messages:

```typescript
// src/browser.ts lines 24-25
type ConnectFailureKind = 'missing-token' | 'extension-timeout' | 'extension-not-installed' | 'mcp-init' | 'process-exit' | 'unknown';
```

Each failure kind produces a specific, helpful error message:

```typescript
// src/browser.ts lines 42-88
export function formatBrowserConnectError(input: ConnectFailureInput): Error {
  if (input.kind === 'missing-token') {
    return new Error(
      'Failed to connect to Playwright MCP Bridge: PLAYWRIGHT_MCP_EXTENSION_TOKEN is not set.\n\n' +
      'Without this token, Chrome will show a manual approval dialog ...'
    );
  }
  if (input.kind === 'extension-not-installed') {
    return new Error(
      'Failed to connect to Playwright MCP Bridge: the browser extension did not attach.\n\n' +
      'Make sure Chrome is running and the "Playwright MCP Bridge" extension is installed ...'
    );
  }
  // ... other kinds
}
```

The `browserSession()` helper ensures cleanup even on failure:

```typescript
// src/runtime.ts lines 41-52
export async function browserSession<T>(BrowserFactory: new () => IBrowserFactory, fn): Promise<T> {
  const mcp = new BrowserFactory();
  try {
    const page = await mcp.connect({ timeout: DEFAULT_BROWSER_CONNECT_TIMEOUT });
    return await fn(page);
  } finally {
    await mcp.close().catch(() => {});  // swallow close errors
  }
}
```

## Pipeline Step Error Propagation

Pipeline steps let errors propagate up naturally. The executor does not catch step errors -- they bubble up to `executeCommand()` and then to the top-level handler in `main.ts`:

```typescript
// src/pipeline/executor.ts lines 56-58
const handler = STEP_HANDLERS[op];
if (handler) {
  data = await handler(page, params, data, args);  // errors propagate
}
```

Unknown steps are silently ignored in normal mode, logged in debug mode:

```typescript
// src/pipeline/executor.ts lines 59-61
} else {
  if (debug) process.stderr.write(`  ${chalk.yellow('...')}  Unknown step: ${op}\n`);
}
```

## Verbose Mode and Internal `debug` Flag

The public CLI surface exposes `--verbose` on dynamic site commands (`src/main.ts` line 138). Internally, `main.ts` passes that boolean through `executeCommand(..., debug)` and `executePipeline(..., { debug })`. Debug output still goes to stderr, keeping stdout clean for piping:

```typescript
// src/main.ts line 148
if (actionOpts.verbose) process.env.OPENCLI_VERBOSE = '1';

// src/pipeline/executor.ts lines 69-93
function debugStepStart(stepNum, total, op, params): void {
  process.stderr.write(`  ${chalk.dim(`[${stepNum}/${total}]`)} ${chalk.bold.cyan(op)}${preview}\n`);
}
function debugStepResult(op, data): void {
  if (Array.isArray(data)) {
    process.stderr.write(`       ${chalk.dim(`-> ${data.length} items`)}\n`);
  }
  // ...
}
```

## Process Cleanup

`PlaywrightMCP` registers global cleanup handlers to kill orphaned browser processes:

```typescript
// src/browser.ts lines 272-285
private static _registerGlobalCleanup() {
  const cleanup = () => {
    for (const inst of this._activeInsts) {
      if (inst._proc && !inst._proc.killed) {
        try { inst._proc.kill('SIGKILL'); } catch {}
      }
    }
  };
  process.on('exit', cleanup);
  process.on('SIGINT', () => { cleanup(); process.exit(130); });
  process.on('SIGTERM', () => { cleanup(); process.exit(143); });
}
```

## Anti-Patterns

- **Do not** use `process.exit()` to handle errors. Set `process.exitCode` and let the process exit naturally.
- **Do not** throw errors from CLI adapter `func()` for expected failures (e.g., no results found). Return an empty array instead.
- **Do not** log errors to stdout. Use `process.stderr.write()` or `console.error()` for all diagnostic output.
- **Do not** catch and swallow errors silently in pipeline steps. Let them propagate so the top-level handler can display them.
- **Do not** hardcode timeout values. Use the constants from `src/runtime.ts` or per-command `timeoutSeconds`.
- **Do not** forget `catch(() => {})` when calling `mcp.close()` in finally blocks -- the close itself can fail if the process already exited.
- **Do not** document a separate end-user `--debug` flag unless the CLI actually exposes one. In this codebase, `debug` is an internal boolean propagated from `--verbose`.
