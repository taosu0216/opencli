# Pipeline Steps Reference

> Runtime contract for every YAML pipeline step supported by
> `executePipeline()`.

## Core Contract

All runtime-supported steps are registered in `STEP_HANDLERS`
(`src/pipeline/executor.ts:21-38`). Each handler conforms to:

`(page, params, data, args) => Promise<any>`

The executor behavior is simple and important (`src/pipeline/executor.ts:40-67`):

- steps run sequentially
- each step receives the previous step's `data`
- unknown steps are ignored, with a warning only in debug mode
- the final `data` value becomes the command result

Runtime-supported step names in this repo:

- `navigate`
- `fetch`
- `select`
- `evaluate`
- `snapshot`
- `click`
- `type`
- `wait`
- `press`
- `map`
- `filter`
- `sort`
- `limit`
- `intercept`
- `tap`

Important caveat:

- `validate.ts` also recognizes `scroll`, but there is no `scroll` handler in
  `STEP_HANDLERS`. Do not document or author `scroll` as a real YAML step until
  runtime support exists.

## Step Summary Table

| Step | Source | Input `data` | Output | Real examples |
|------|--------|--------------|--------|---------------|
| `navigate` | `browser.ts` | preserved | unchanged `data` | `twitter/trending.yaml:12-13` |
| `click` | `browser.ts` | preserved | unchanged `data` | no built-in YAML usage today |
| `type` | `browser.ts` | preserved | unchanged `data` | no built-in YAML usage today |
| `wait` | `browser.ts` | preserved | unchanged `data` | `twitter/bookmarks.yaml:14-15` |
| `press` | `browser.ts` | preserved | unchanged `data` | no built-in YAML usage today |
| `snapshot` | `browser.ts` | ignored | snapshot result | intended for debugging pipelines |
| `evaluate` | `browser.ts` | previous `data` available in templates | return value of `page.evaluate()` | `bilibili/hot.yaml:15-27` |
| `fetch` | `fetch.ts` | optional previous `data` | parsed JSON or array of JSON payloads | `v2ex/hot.yaml:14-23`, `hackernews/top.yaml:14-34` |
| `select` | `transform.ts` | object or array | nested value | test-backed in `executor.test.ts:52-61` |
| `map` | `transform.ts` | array, object, or `data.data` | array of rows | almost every YAML adapter |
| `filter` | `transform.ts` | array | filtered array | supported, but unused in built-in YAML today |
| `sort` | `transform.ts` | array | sorted copy | supported, but unused in built-in YAML today |
| `limit` | `transform.ts` | array | sliced array | `v2ex/latest.yaml:18-23` |
| `intercept` | `intercept.ts` | preserved if no match | captured payload or selection | supported, but no built-in YAML currently uses it |
| `tap` | `tap.ts` | preserved until store call returns | captured payload or selection | `xiaohongshu/feed.yaml:19-24` |

## Browser Steps

### `navigate`

Implementation: `src/pipeline/steps/browser.ts:9-13`

- `params`: string URL template
- effect: calls `page.goto(render(params, { args, data }))`
- return: original `data`

Use it to establish cookies or page context before `evaluate`, `intercept`, or
`tap`.

### `click`

Implementation: `src/pipeline/steps/browser.ts:15-18`

- `params`: element ref string
- `@` prefix is stripped before clicking
- return: original `data`

Use refs that come from snapshots or previous exploration. The step does not
wait automatically after clicking.

### `type`

Implementation: `src/pipeline/steps/browser.ts:20-28`

- `params`: object with `ref`, `text`, and optional `submit`
- renders both `ref` and `text`
- if `submit` is truthy, presses `Enter` after typing

Example shape:

```yaml
- type:
    ref: "@search-box"
    text: ${{ args.query }}
    submit: true
```

### `wait`

Implementation: `src/pipeline/steps/browser.ts:30-41`

Accepted forms:

- number: `- wait: 3`
- object with `text` and optional `timeout`
- object with `time`
- string template that renders to a number

Built-in examples:

- `src/clis/twitter/bookmarks.yaml:14-15`
- `src/clis/xiaohongshu/notifications.yaml:21-22`

### `press`

Implementation: `src/pipeline/steps/browser.ts:43-46`

- `params`: rendered key string
- return: original `data`

This is a thin wrapper over `page.pressKey()`.

### `snapshot`

Implementation: `src/pipeline/steps/browser.ts:48-51`

- `params`: object with `interactive`, `compact`, `max_depth`, and `raw`
- note the YAML key is `max_depth`, not `maxDepth`
- return: snapshot payload

This is mainly a debugging step, not a data extraction primitive.

### `evaluate`

Implementation: `src/pipeline/steps/browser.ts:53-64`

- `params`: string JavaScript source, usually a block scalar
- source is normalized through `normalizeEvaluateSource()`
  (`src/pipeline/template.ts:151-159`)
- stringified JSON results are auto-parsed back into objects or arrays

Real examples:

- `src/clis/twitter/trending.yaml:15-31`
- `src/clis/zhihu/hot.yaml:15-35`
- `src/clis/reddit/search.yaml:20-26`

## Fetch And Capture Steps

### `fetch`

Implementation: `src/pipeline/steps/fetch.ts:100-139`

Accepted forms:

- simple string URL
- object with `url`, `method`, `params`, `headers`, and optional `concurrency`

Behavior:

- If `page === null`, uses Node `fetch()` and returns `resp.json()`
  (`src/pipeline/steps/fetch.ts:42-45`).
- If `page` exists, fetches inside the browser with `credentials: "include"`
  (`src/pipeline/steps/fetch.ts:47-57`).
- If current `data` is an array and the URL template contains `item`, it enters
  per-item batch mode (`src/pipeline/steps/fetch.ts:107-139`).

Gotchas:

- Response parsing is always `resp.json()`. Do not use this step for XML or
  HTML payloads.
- `params` and `headers` are rendered before request dispatch.
- Per-item batching checks for the literal substring `item` in the URL template.

### `intercept`

Implementation: `src/pipeline/steps/intercept.ts:9-56`

Accepted config:

- `capture`: required URL substring or pattern string
- `trigger`: one of `navigate:<url>`, `evaluate:<js>`, `click:<ref>`, or
  literal `scroll`
- `timeout`: seconds, capped to `3` during the wait call
- `select`: dot-path to pick part of the captured payload

Flow:

1. inject interception JavaScript
2. trigger a page action
3. wait briefly
4. read captured responses
5. optionally select into the payload

Use this when the right data only appears in XHR/fetch traffic after a user
action.

### `tap`

Implementation: `src/pipeline/steps/tap.ts:16-100`

Accepted config:

- `store`: required store name
- `action`: required store action name
- `args`: optional action argument list
- `capture`: URL pattern for the resulting request
- `select`: dot-path into the captured payload
- `timeout`: seconds
- `framework`: optional `pinia` or `vuex`

Real examples:

- `src/clis/xiaohongshu/feed.yaml:19-24`
- `src/clis/xiaohongshu/notifications.yaml:23-30`

This is the repo's SPA-specific bridge step. It patches fetch/XHR, finds the
store, dispatches the action, and returns the captured response.

## Transform Steps

### `select`

Implementation: `src/pipeline/steps/transform.ts:7-19`

- `params`: dot-path string like `data.list` or `list.1`
- supports array index traversal for numeric path parts
- returns `null` when traversal fails

See tests in `src/pipeline/transform.test.ts:14-35`.

### `map`

Implementation: `src/pipeline/steps/transform.ts:21-33`

- `params`: object of output-key to template string
- accepts:
  - an array
  - a single object, wrapped as a one-element array
  - an object with a `data` property, in which case it maps `data.data`

Real examples:

- `src/clis/v2ex/hot.yaml:18-23`
- `src/clis/hackernews/top.yaml:26-34`
- `src/clis/xiaohongshu/notifications.yaml:31-38`

### `filter`

Implementation: `src/pipeline/steps/transform.ts:35-38`

- `params`: expression string
- each array item is kept when `evalExpr()` returns a truthy value

Important limitation:

- The expression engine is intentionally small. Think truthy path checks and
  template helpers, not arbitrary JS predicates.

### `sort`

Implementation: `src/pipeline/steps/transform.ts:40-45`

Accepted forms:

- string key, for ascending sort
- object `{ by, order: 'asc' | 'desc' }`

Behavior:

- returns a copied array
- compares with JavaScript `<` and `>` on the selected key

See tests in `src/pipeline/transform.test.ts:75-90`.

### `limit`

Implementation: `src/pipeline/steps/transform.ts:47-50`

- `params`: number-like value or template
- only affects arrays
- implemented as `data.slice(0, Number(render(params, ...)))`

Real examples:

- `src/clis/v2ex/latest.yaml:23`
- `src/clis/twitter/trending.yaml:38`
- `src/clis/hackernews/top.yaml:18` and `:34`

## Recommended Chaining Patterns

- Public API: `fetch -> map -> limit`
- Cookie-backed internal API: `navigate -> evaluate -> map -> limit`
- Fan-out fetch: `fetch -> limit -> map -> fetch -> map -> limit`
- SPA store bridge: `navigate -> wait -> tap -> map -> limit`

## Anti-Patterns

- Do not author `scroll` as a standalone step.
- Do not expect `navigate`, `click`, `type`, `wait`, or `press` to modify data;
  they preserve the previous payload.
- Do not use `fetch` for non-JSON responses.
- Do not rely on `filter` for complex boolean logic.
- Do not forget that unknown steps are silently ignored unless debug mode is on
  (`src/pipeline/executor.test.ts:121-128`).
