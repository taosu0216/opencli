# YAML Pipeline CLI Definitions

> Declarative adapters for commands that can be expressed as a linear sequence
> of browser, fetch, and transform steps.

## How YAML Adapters Enter The Runtime

There are two registration paths:

1. Development path
   - `discoverClisFromFs()` scans `src/clis/**` and parses `.yaml` files at
     runtime (`src/engine.ts:94-160`).
2. Build path
   - `npm run build` copies YAML files into `dist/clis/` and compiles
     `dist/cli-manifest.json` (`package.json:16-28`,
     `src/build-manifest.ts:170-198`).
   - `loadFromManifest()` registers YAML commands directly from the manifest on
     startup (`src/engine.ts:43-88`).

That means YAML adapters must stay serializable. Everything the command needs
must be representable as plain YAML data.

## Top-Level Schema

`registerYamlCli()` and `scanYaml()` agree on this shape
(`src/engine.ts:116-156`, `src/build-manifest.ts:45-81`):

| Field | Required | Runtime mapping | Notes |
|------|----------|-----------------|-------|
| `site` | No | `CliCommand.site` | Defaults to the directory name |
| `name` | No | `CliCommand.name` | Defaults to the filename stem |
| `description` | No | `CliCommand.description` | Empty string if omitted |
| `domain` | No | `CliCommand.domain` | Used by browser-backed commands |
| `strategy` | No | `CliCommand.strategy` | String is uppercased into the `Strategy` enum |
| `browser` | No | `CliCommand.browser` | Defaults to `false` only for `public` |
| `args` | No | `CliCommand.args` | YAML object becomes an `Arg[]` |
| `columns` | No | `CliCommand.columns` | Output column order |
| `pipeline` | Yes in practice | `CliCommand.pipeline` | Executed by `executePipeline()` |
| `timeout` | No | `CliCommand.timeoutSeconds` | Stored as seconds |

Defaulting is important:

- `strategy` defaults to `public` when `browser: false`, otherwise `cookie`
  (`src/engine.ts:124-126`).
- `browser` defaults to `true` unless the resolved strategy is `PUBLIC`
  (`src/engine.ts:124-126`).

## Args Syntax

YAML args are keyed by name, then converted into `Arg` objects with `help`
filled from `description` or `help` (`src/engine.ts:128-140`).

```yaml
# src/clis/reddit/search.yaml:8-15
args:
  query:
    type: string
    required: true
  limit:
    type: int
    default: 15
```

Supported properties:

| Property | Meaning |
|----------|---------|
| `type` | Common values are `str`, `string`, and `int` |
| `default` | Used by Commander and by the command implementation |
| `required` | Required CLI option when true |
| `description` | Becomes `Arg.help` |
| `help` | Alternate way to set `Arg.help` |
| `choices` | Preserved into the manifest and registry |

Repo conventions:

- Use `description` in YAML, not a separate comment block.
- Keep arg names simple. Commander exposes them as `--<name>` from `main.ts`.
- Prefer explicit `required: true` over throwing late in `evaluate` code.

## Template Interpolation

`${{ ... }}` expressions are evaluated by `render()` and `evalExpr()` in
`src/pipeline/template.ts:12-159`.

Available context:

- `args`: command arguments
- `data`: current pipeline output
- `item`: current element inside `map` and `filter`
- `index`: current array index

Supported expression families:

- Dot-path lookup: `${{ item.title }}`
- Implicit item lookup: `${{ title }}` also resolves against `item`
- Arithmetic with an integer literal: `${{ index + 1 }}`
- Fallback with `||`: `${{ item.tweetCount || 'N/A' }}`
- Filter pipes: `${{ args.limit | default(20) }}`

Supported filters from `applyFilter()` (`src/pipeline/template.ts:80-125`):

- `default(val)`
- `join(sep)`
- `upper`
- `lower`
- `trim`
- `truncate(n)`
- `replace(old,new)`
- `keys`
- `length`
- `first`
- `last`
- `json`

Real examples:

- `${{ index + 1 }}` in `src/clis/twitter/trending.yaml:33-38`
- `${{ item }}` for primitive IDs in `src/clis/hackernews/top.yaml:20-25`
- `${{ args.symbol | json }}` inside embedded JavaScript in
  `src/clis/xueqiu/stock.yaml:14-18`
- `${{ args.limit | default(20) }}` in `src/clis/xiaohongshu/feed.yaml:25-32`

Important limitations:

- This is not full JavaScript evaluation. Comparisons like
  `${{ item.score > 10 }}` are not implemented.
- Full-expression templates return native types, but inline templates are always
  stringified (`src/pipeline/template.test.ts:86-107`).
- Missing paths return `undefined` or `null` depending on the traversal path, so
  use `default(...)` when you need stable output.

## Pipeline Patterns In This Repo

### Public JSON API

Use `strategy: public` and `browser: false` when the site exposes a stable API
without a logged-in browser.

```yaml
# src/clis/v2ex/hot.yaml:1-25
pipeline:
  - fetch:
      url: https://www.v2ex.com/api/topics/hot.json
  - map:
      rank: ${{ index + 1 }}
      title: ${{ item.title }}
      replies: ${{ item.replies }}
  - limit: ${{ args.limit }}
```

### Batch Per-Item Fetch

`stepFetch()` enters batch mode only when current `data` is an array and the
URL string literally contains `item` (`src/pipeline/steps/fetch.ts:107-139`).

```yaml
# src/clis/hackernews/top.yaml:14-34
pipeline:
  - fetch:
      url: https://hacker-news.firebaseio.com/v0/topstories.json
  - limit: 30
  - map:
      id: ${{ item }}
  - fetch:
      url: https://hacker-news.firebaseio.com/v0/item/${{ item.id }}.json
```

### Navigate Then Evaluate

Use this for sites that need cookies or DOM state before calling an internal
API.

```yaml
# src/clis/bilibili/hot.yaml:12-36
pipeline:
  - navigate: https://www.bilibili.com
  - evaluate: |
      (async () => {
        const res = await fetch('https://api.bilibili.com/x/web-interface/popular?ps=${{ args.limit }}&pn=1', {
          credentials: 'include'
        });
        const data = await res.json();
        return (data?.data?.list || []).map((item) => ({
          title: item.title,
          author: item.owner?.name,
          play: item.stat?.view,
          danmaku: item.stat?.danmaku,
        }));
      })()
```

### Browser Fetch With Cookie/Header Bootstrapping

Twitter bookmarks and trending manually extract cookies and inject headers in
the evaluated script:

- `src/clis/twitter/bookmarks.yaml:16-75`
- `src/clis/twitter/trending.yaml:15-31`

This is still a YAML-friendly pattern because the runtime only needs to render
one string and send it to `page.evaluate()`.

### Store Action Bridge

YAML is not limited to plain `fetch`. The `tap` step drives SPA store actions:

- `src/clis/xiaohongshu/feed.yaml:16-32`
- `src/clis/xiaohongshu/notifications.yaml:20-38`

Choose this when the site already has a client-side store action that triggers
the right network request.

## When To Choose YAML

YAML is the right fit when:

- The command is a straight pipeline of fetch, browser, and transform steps.
- Metadata can stay literal and serializable.
- You want the adapter fully inlined in the manifest.
- The extraction logic fits comfortably inside one `evaluate` block or one step
  chain.

Prefer TypeScript when:

- You need loops, retries, helper functions, or non-trivial parsing logic.
- You need multiple strategies or fallback branches inside one adapter.
- You want to reuse helpers like `apiGet()` or normalization utilities.
- Manifest metadata would become too dynamic to express safely in YAML.

## Common Mistakes And Anti-Patterns

- Do not use `scroll` as a YAML step. `validate.ts` accepts it, but
  `executePipeline()` does not register a handler, so it is ignored at runtime.
- Do not assume `filter` can evaluate arbitrary JavaScript. It only uses the
  template expression engine.
- Do not put secrets or non-serializable state into YAML. It is copied into the
  build manifest.
- Do not rely on implicit batching in `fetch` unless the URL contains `item`.
- Do not forget `browser: false` or `strategy: public` for true Node-side API
  commands like `hackernews/top` and `v2ex/hot`.
- Do not overuse YAML when the `evaluate` block starts becoming a full program.
  At that point, convert the adapter to TypeScript.
