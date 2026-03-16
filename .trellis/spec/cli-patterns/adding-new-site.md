# Adding A New Site

> Practical workflow for creating a new `opencli` adapter that survives build,
> validation, and runtime discovery.

## 1. Start With The Simplest Viable Strategy

Before writing code, figure out how the target site exposes data:

1. Try `PUBLIC`
   - anonymous API, RSS, or HTML that Node can fetch directly
2. Try `COOKIE`
   - browser session plus internal fetch or DOM extraction
3. Try `HEADER`
   - same as cookie, but with explicit CSRF/header requirements
4. Escalate to `INTERCEPT`
   - capture the site's own XHR/fetch traffic
5. Use `UI` last
   - direct DOM automation for write flows or stubborn pages

Useful built-in tools:

- `opencli explore <url> --site <name>` to inspect the page
  (`README.md:177-185`)
- `opencli cascade <api-url>` to probe `PUBLIC -> COOKIE -> HEADER`
  (`README.md:187-189`, `src/cascade.ts:143-174`)

## 2. Choose YAML Or TypeScript

Use YAML when:

- the command is a straight pipeline
- metadata is fully serializable
- one `evaluate` block plus a few transform steps are enough

Use TypeScript when:

- the adapter needs helpers, loops, retries, or parsing branches
- you need direct `IPage` methods like `installInterceptor()` or `autoScroll()`
- the site shares utilities with sibling commands

Reference files:

- YAML: `src/clis/hackernews/top.yaml`, `src/clis/xiaohongshu/feed.yaml`
- TypeScript: `src/clis/twitter/search.ts`, `src/clis/bilibili/search.ts`

## 3. Create The File In The Right Place

Built-in adapters live under:

- `src/clis/<site>/<command>.yaml`
- `src/clis/<site>/<command>.ts`

Discovery behavior:

- development mode scans the filesystem (`src/engine.ts:94-160`)
- build mode uses `dist/cli-manifest.json` (`src/engine.ts:25-88`)

There is no central site index to update. Discovery is directory-based.

## 4. Keep Metadata Literal

For either adapter type, define these fields clearly:

- `site`
- `name`
- `description`
- `domain`
- `strategy`
- `browser`
- `args`
- `columns`

For TypeScript this is non-negotiable because `scanTs()` extracts manifest
metadata with regexes (`src/build-manifest.ts:88-167`).

Good pattern:

```ts
cli({
  site: 'mysite',
  name: 'search',
  description: 'Search MySite posts',
  domain: 'www.mysite.com',
  strategy: Strategy.COOKIE,
  browser: true,
  args: [
    { name: 'query', required: true, help: 'Search keyword' },
    { name: 'limit', type: 'int', default: 20, help: 'Result count' },
  ],
  columns: ['rank', 'title', 'url'],
  func: async (page, kwargs) => { ... },
});
```

## 5. Implement The Extraction Path

Recommended decision patterns:

### Public JSON Or RSS

- YAML: `fetch -> map -> limit`
- TS: use Node `fetch()` directly like `src/clis/bbc/news.ts:17-41`

### Cookie-Backed API

- navigate first if the site needs cookies to warm up
- use `page.evaluate(fetch(... credentials: 'include' ...))` in YAML or TS
- examples:
  - `src/clis/reddit/search.yaml:18-34`
  - `src/clis/bilibili/hot.yaml:12-36`

### Intercepted Traffic

- install an interceptor or use the YAML `intercept`/`tap` step
- trigger the site behavior with scroll, click, or store dispatch
- parse the captured payloads into stable rows
- examples:
  - `src/clis/twitter/search.ts:21-64`
  - `src/clis/xiaohongshu/feed.yaml:19-32`

### Full UI Automation

- validate the page exists
- navigate to the concrete route
- act through DOM selectors and return explicit status rows
- examples:
  - `src/clis/twitter/post.ts:18-66`
  - `src/clis/twitter/reply.ts:19-60`

## 6. Validate During Development

Use the source entrypoint while iterating:

```bash
npx tsx src/main.ts list
npx tsx src/main.ts validate <site>/<command>
npx tsx src/main.ts <site> <command> --arg value
```

Notes:

- The CLI shape is `opencli <site> <command>`, not `run site:command`.
- `main.ts` creates Commander subcommands dynamically from the registry
  (`src/main.ts:124-163`).
- YAML validation only checks `.yaml` files today (`src/validate.ts:14-65`).

If the adapter is browser-backed, test both the happy path and the empty-data
path. Existing adapters often return `[]` when the site gives no rows.

## 7. Rebuild The Manifest

Always run:

```bash
npm run build
```

Why:

- compiles TypeScript
- copies YAML into `dist/clis`
- regenerates `dist/cli-manifest.json`

Source: `package.json:16-28`.

After building, smoke test the built artifact too:

```bash
node dist/main.js list
node dist/main.js <site> <command> --arg value
```

This catches manifest extraction mistakes that source-mode testing can miss.

## 8. Reverse-Engineering Tips

- If the page is an SPA, check whether a store action can drive the request.
  `tap` may be simpler than a custom interceptor.
- If the request only appears after scrolling, prefer `INTERCEPT` over trying to
  reproduce complex signed requests by hand.
- If a previous interceptor approach breaks, try DOM extraction before jumping
  straight to `UI`. `src/clis/xiaohongshu/search.ts:1-68` is exactly that
  migration.
- If you need embedded JavaScript in YAML, use `${{ args.foo | json }}` when
  inserting user-controlled strings into script source.

## 9. New Adapter Checklist

- File is under `src/clis/<site>/`
- Metadata is literal and complete
- `strategy` and `browser` match the actual implementation
- output rows are normalized, not raw site payloads
- source mode works with `npx tsx src/main.ts ...`
- build mode works after `npm run build`
- YAML adapters pass `validate`
- no reliance on the unsupported `scroll` YAML step
