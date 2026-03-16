# TypeScript CLI Definitions

> Programmatic adapters that self-register by calling `cli({...})` at module
> import time.

## Registration Model

TypeScript adapters do not export a registry object. They call `cli()` as a
side effect:

```ts
// src/clis/twitter/search.ts:3-15
cli({
  site: 'twitter',
  name: 'search',
  description: 'Search Twitter/X for tweets',
  strategy: Strategy.INTERCEPT,
  browser: true,
  func: async (page, kwargs) => { ... }
});
```

The registry contract lives in `src/registry.ts:7-93`:

- `Strategy` enum defines the command mode.
- `Arg` defines CLI option metadata.
- `CliOptions` is the builder input shape.
- `CliCommand` is the normalized runtime shape.
- `cli()` stores the command under `site/name`.

`cli()` also applies defaults (`src/registry.ts:59-76`):

- `strategy` becomes `PUBLIC` only when `browser === false`; otherwise it
  becomes `COOKIE`.
- `browser` becomes `false` only when `strategy === Strategy.PUBLIC`; otherwise
  it becomes `true`.
- `args` defaults to an empty array.

## Manifest Constraints

Build-time manifest generation for TypeScript is static and regex-based
(`src/build-manifest.ts:88-167`). That has two practical consequences:

1. Keep metadata literal
   - `description`, `domain`, `strategy`, `browser`, `columns`, and `args`
     should appear as inline literals inside the `cli({...})` call.
2. Avoid clever indirection
   - If you compute `description` or build the `args` array through helpers, the
     manifest compiler may miss it and the generated stub will be incomplete.

The scanner currently extracts only a subset of arg metadata
(`src/build-manifest.ts:130-160`):

- supported today: `name`, `type`, `default`, `required`, `help`
- not extracted today: `description`, `choices`, or custom computed fields

Use `help` for TypeScript adapter option text. Do not assume `choices` will be
available in the manifest-driven Commander wiring until `scanTs()` learns it.

At build time, `scanTs()` writes a manifest stub with `type: 'ts'` and
`modulePath`. At runtime, `loadFromManifest()` registers a lazy stub, and the
first `executeCommand()` call imports the real module and re-fetches the command
from the registry (`src/engine.ts:64-84` and `src/engine.ts:171-193`).

This means TypeScript adapters should preserve a one-command-per-file pattern.
Re-registering the same `site/name` key is allowed and intentionally overwrites
the previous entry (`src/registry.test.ts:56-62`).

## `CliOptions` And `CliCommand`

Important fields from `src/registry.ts:24-55`:

| Field | Meaning |
|------|---------|
| `site` / `name` | Registry key and CLI path |
| `description` | Commander subcommand description |
| `domain` | Human/browser context only |
| `strategy` | Mode label plus defaulting hint |
| `browser` | Actual switch used by `main.ts` to create a browser session |
| `args` | Commander option definitions |
| `columns` | Preferred output field order |
| `func` | Async implementation callback |
| `pipeline` | Optional YAML-style pipeline for TS adapters |
| `timeoutSeconds` | Per-command timeout enforced by `runWithTimeout()` |

`main.ts` builds CLI flags directly from `args` (`src/main.ts:132-146`), so the
`name`, `required`, `default`, `help`, and `type` fields have user-facing
effects.

## `func` Callback Contract

The normalized callback signature is:

`(page: IPage, kwargs: Record<string, any>, debug?: boolean) => Promise<any>`

Source: `src/registry.ts:33` and `src/registry.ts:53`.

Repo-specific caveat:

- `main.ts` passes `null` as the page for non-browser commands
  (`src/main.ts:150-152`), even though `CliCommand.func` is typed as `IPage`.
- Public commands therefore either ignore `page` entirely or annotate their
  local callback argument more loosely, as in `src/clis/twitter/post.ts:15-67`
  and `src/clis/bbc/news.ts:17-41`.

Treat the runtime contract as:

- Browser command: `page` is a real `IPage`
- Non-browser command: `page` may be `null`; do not touch it

## Common Implementation Patterns

### 1. Intercept + Scroll + Parse Payloads

`twitter/search` is the canonical intercept adapter
(`src/clis/twitter/search.ts:15-64`):

- navigate to a page that will trigger the desired API
- install an interceptor with a targeted pattern
- drive the page (`autoScroll`, click, or navigate)
- inspect `getInterceptedRequests()`
- parse the captured payloads defensively

`twitter/followers` follows the same pattern with a click trigger and
deduplication (`src/clis/twitter/followers.ts:36-117`).

### 2. Cookie-Backed Helper API Call

`bilibili/search` uses a shared helper rather than inline browser code
(`src/clis/bilibili/search.ts:1-25`):

- keep metadata literal for the manifest
- reuse helpers like `apiGet()` and `stripHtml()`
- map into stable output rows near the end

This is the preferred shape once multiple adapters on the same site need the
same auth/signing logic.

### 3. Browser DOM Extraction

`xiaohongshu/search` moved from interception to direct DOM parsing after the old
API became unreliable (`src/clis/xiaohongshu/search.ts:1-68`).

This is a good example of when `Strategy.COOKIE` is still correct even though
the adapter is not doing raw `fetch()` calls. The command needs a logged-in
browser page and rendered DOM.

### 4. Pure Public Command In TypeScript

`bbc/news` uses `Strategy.PUBLIC` and never touches the browser
(`src/clis/bbc/news.ts:7-41`).

Use this shape when:

- Node `fetch()` is enough
- you do not need cookies
- the parsing logic is easier to express in TypeScript than YAML

### 5. Full UI Automation

`twitter/post` and `twitter/reply` are the reference UI adapters
(`src/clis/twitter/post.ts:15-67`, `src/clis/twitter/reply.ts:16-62`).

They:

- validate the presence of a browser page early
- navigate to the concrete UI route
- use `page.evaluate()` for DOM interaction
- return structured success/failure rows instead of raw booleans

## Error Handling Conventions In Existing Adapters

Current adapters use a few repeatable patterns:

- Throw early when a browser session is required but absent
  (`src/clis/twitter/post.ts:16`, `src/clis/twitter/reply.ts:17`).
- Return `[]` for empty upstream data instead of throwing when the adapter can
  legitimately produce no rows (`src/clis/twitter/search.ts:28-30`,
  `src/clis/xiaohongshu/user.ts:25-27`).
- Swallow per-record parse failures inside loops to keep partial data flowing
  (`src/clis/twitter/search.ts:32-60`, `src/clis/twitter/followers.ts:77-109`).
- Prefer one final `slice(0, kwargs.limit)` or mapped output stage at the end
  of the function.

Do not over-handle errors inside the registry layer. `executeCommand()` already
throws a clear error when the command has neither `func` nor `pipeline`
(`src/engine.ts:195-201`).

## When To Choose TypeScript

Choose TypeScript when any of these are true:

- The adapter needs helpers, loops, branching, or retry behavior.
- The parser needs site-specific normalization logic.
- The extraction path combines several fallback strategies.
- You need direct access to `IPage` methods like `installInterceptor()` or
  `autoScroll()`.
- The adapter should share utilities with sibling commands.

Stay with YAML when the command is just a serializable pipeline.

## Anti-Patterns And Gotchas

- Do not hide command metadata behind variables or helper factories unless you
  also update `scanTs()`. The manifest compiler is not a real AST parser.
- Do not assume `strategy` alone controls execution. `main.ts` uses `cmd.browser`
  to decide whether to allocate a browser page.
- Do not call `page.*` in a public command unless you explicitly force
  `browser: true`.
- Do not return raw site payloads if the command already knows its output shape.
  Normalize into stable rows before returning.
- Do not define multiple `cli()` calls in one file unless you are deliberately
  handling registry overwrites and manifest extraction edge cases.
