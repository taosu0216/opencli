# Strategy Guide

> How `opencli` uses `Strategy.PUBLIC`, `COOKIE`, `HEADER`, `INTERCEPT`, and
> `UI` in practice.

## What `strategy` Actually Does In This Repo

`Strategy` lives in `src/registry.ts:7-13`, but the runtime uses it in three
different ways:

1. Metadata and display
   - `opencli list` prints the strategy label from the registry
     (`src/main.ts:34-61`).
2. Defaulting
   - `cli()` and `registerYamlCli()` infer `strategy` and `browser` from each
     other (`src/registry.ts:65-71`, `src/engine.ts:124-126`).
3. Strategy probing
   - `cascadeProbe()` tries `PUBLIC -> COOKIE -> HEADER` before giving up
     (`src/cascade.ts:16-23` and `src/cascade.ts:143-174`).

Critical nuance:

- `main.ts` decides whether to create a browser page from `cmd.browser`, not
  from `cmd.strategy` (`src/main.ts:150-152`).
- In other words, `strategy` tells humans and tooling how the adapter works,
  while `browser` is the operational switch.

## Strategy Table

| Strategy | Typical behavior | Real adapters |
|----------|------------------|---------------|
| `PUBLIC` | No auth required. Often runs with `browser: false` and Node `fetch()` | `src/clis/v2ex/hot.yaml`, `src/clis/hackernews/top.yaml`, `src/clis/bbc/news.ts` |
| `COOKIE` | Needs an authenticated browser session or browser cookies | `src/clis/reddit/search.yaml`, `src/clis/bilibili/search.ts`, `src/clis/yahoo-finance/quote.ts` |
| `HEADER` | Adds auth headers derived from browser cookies or tokens | no production adapter yet; see `src/cascade.ts:115-122` |
| `INTERCEPT` | Captures site traffic triggered by UI actions | `src/clis/twitter/search.ts`, `src/clis/twitter/followers.ts`, `src/clis/xiaohongshu/user.ts` |
| `UI` | Drives the DOM directly for mutations or hard-to-capture flows | `src/clis/twitter/post.ts`, `src/clis/twitter/reply.ts`, `src/clis/twitter/like.ts` |

## Choosing A Strategy

Start from the simplest mode that can work:

1. `PUBLIC`
   - Use when the endpoint works without cookies or special headers.
2. `COOKIE`
   - Use when the browser session must provide cookies, signed URLs, or a
     hydrated page.
3. `HEADER`
   - Use when the site accepts a normal fetch once you add CSRF or session
     headers.
4. `INTERCEPT`
   - Use when the API request is hard to reproduce manually but easy to capture.
5. `UI`
   - Use when the flow is inherently interactive or stateful and API capture is
     not reliable.

The project README recommends the same escalation path through `opencli cascade`
for reverse engineering (`README.md:177-189`).

## `PUBLIC`

Reference patterns:

- `src/clis/v2ex/hot.yaml:1-25`
- `src/clis/hackernews/top.yaml:1-36`
- `src/clis/bbc/news.ts:7-41`

Use `PUBLIC` when:

- the site exposes a stable anonymous JSON or RSS endpoint
- the command can run entirely from Node
- you do not need `page.goto()` to establish cookies

Recommended pairing:

- set `browser: false` explicitly in YAML
- in TypeScript, either set `browser: false` or rely on
  `strategy: Strategy.PUBLIC` to default it to `false`

## `COOKIE`

Reference patterns:

- `src/clis/reddit/search.yaml:18-34`
- `src/clis/bilibili/hot.yaml:12-36`
- `src/clis/bilibili/search.ts:13-24`
- `src/clis/yahoo-finance/quote.ts:17-73`
- `src/clis/xiaohongshu/search.ts:22-67`

Use `COOKIE` when:

- you need `credentials: 'include'`
- the site blocks anonymous requests but works with an existing browser login
- you need to navigate first, then call the API from inside the page

Common repo patterns:

- `navigate -> evaluate(fetch(... credentials: 'include' ...))`
- site helpers like `apiGet(page, ...)`
- DOM extraction on a logged-in page when the API is unstable

## `HEADER`

There is no built-in production adapter using `Strategy.HEADER` today. The best
real reference is the strategy cascade probe in `src/cascade.ts:44-122`.

What the repo means by `HEADER`:

- start from a cookie-backed browser page
- extract CSRF tokens from `document.cookie`
- send those values as `X-Csrf-Token` or `X-XSRF-Token`

Use it when:

- a plain cookie-backed fetch still fails
- the target site expects an extra CSRF header
- you can reproduce the request with a stable header set

Do not claim `HEADER` just because a request has headers. In this repo it means
"cookie-backed request plus extra auth headers."

## `INTERCEPT`

Reference patterns:

- `src/clis/twitter/search.ts:15-64`
- `src/clis/twitter/followers.ts:36-117`
- `src/clis/xiaohongshu/user.ts:15-44`
- YAML-side SPA capture: `src/clis/xiaohongshu/feed.yaml:16-32`

Use `INTERCEPT` when:

- the site's internal API is GraphQL-heavy, signed, or annoyingly stateful
- reproducing the exact request by hand is fragile
- the network request only appears after a real user interaction

Two implementation styles exist:

1. Imperative TypeScript
   - `page.installInterceptor()`
   - trigger with `autoScroll()`, click, or navigate
   - inspect `page.getInterceptedRequests()`
2. Declarative YAML
   - `intercept` step for generic capture flows
   - `tap` step for store-action-based SPAs

Important: setting `strategy: INTERCEPT` does not install anything by itself.
The adapter must still call the interception APIs or use the corresponding YAML
step.

## `UI`

Reference patterns:

- `src/clis/twitter/post.ts:15-67`
- `src/clis/twitter/reply.ts:16-62`
- `src/clis/twitter/like.ts` and `src/clis/twitter/delete.ts` follow the same
  idea

Use `UI` when:

- the command performs a mutation
- the user action matters more than the API payload
- the site has anti-bot logic that makes direct API recreation brittle

Typical repo shape:

- guard that a browser page exists
- navigate to the exact screen
- interact through `page.evaluate()` or browser primitives
- wait briefly for the mutation to settle
- return a structured success/failure row

## Browser Session Behavior

The browser lifecycle is controlled from `main.ts:140-162`:

- if `cmd.browser` is truthy, the command runs inside `browserSession(...)`
- otherwise `executeCommand()` is called with `page = null`

Implications:

- `PUBLIC` usually means `browser: false`, but not always
- `COOKIE`, `INTERCEPT`, and `UI` usually mean `browser: true`
- if you set `browser: true` on a `PUBLIC` command, `main.ts` will still create
  a browser session

Document and set `browser` deliberately. Do not rely on readers to infer it
from `strategy`.

## Anti-Patterns

- Do not choose `INTERCEPT` when a stable `PUBLIC` or `COOKIE` fetch already
  exists.
- Do not choose `UI` for read-only data when interception or a direct API call
  works.
- Do not label a command `HEADER` unless extra auth headers are actually part of
  the solution.
- Do not forget that the build manifest stores the strategy string. Keep it
  literal and static in TypeScript adapters.
