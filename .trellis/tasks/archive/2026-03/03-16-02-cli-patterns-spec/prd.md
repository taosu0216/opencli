# Fill CLI Definition Patterns Spec

## Goal

Analyze the opencli CLI definition patterns and fill all spec files under `.trellis/spec/cli-patterns/` with real code examples, complete references, and step-by-step guides for adding new website CLIs.

## Context

**opencli** turns websites into CLI commands. There are two ways to define a CLI command:

### Pattern 1: YAML Pipeline (Declarative)

```yaml
# src/clis/twitter/trending.yaml
site: twitter
name: trending
description: Twitter/X trending topics
domain: x.com

args:
  limit:
    type: int
    default: 20

pipeline:
  - navigate: https://x.com/explore/tabs/trending
  - evaluate: |
      (async () => { /* JS to extract data */ })()
  - map:
      rank: ${{ index + 1 }}
      topic: ${{ item.name }}
  - limit: ${{ args.limit }}

columns: [rank, topic, tweets]
```

### Pattern 2: TypeScript Function (Programmatic)

```typescript
// src/clis/twitter/search.ts
import { cli, Strategy } from '../../registry.js';

cli({
  site: 'twitter',
  name: 'search',
  description: 'Search Twitter/X for tweets',
  domain: 'x.com',
  strategy: Strategy.INTERCEPT,
  browser: true,
  args: [
    { name: 'query', type: 'string', required: true },
    { name: 'limit', type: 'int', default: 15 },
  ],
  columns: ['id', 'author', 'text', 'likes', 'views', 'url'],
  func: async (page, kwargs) => {
    // Navigate, intercept, extract, return data
  }
});
```

### Strategy Enum

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| `PUBLIC` | No auth needed, public API/scraping | Public pages, no login required |
| `COOKIE` | Uses browser cookies for auth | Sites where user is logged in via browser |
| `HEADER` | Custom headers (API keys, tokens) | API-first sites with token auth |
| `INTERCEPT` | XHR/fetch interception | Sites with complex internal APIs (Twitter, Bilibili) |
| `UI` | Full UI automation | Sites requiring form interaction |

### Pipeline Steps Available

| Step | Purpose | Module |
|------|---------|--------|
| `navigate` | Go to URL | `pipeline/steps/browser.ts` |
| `click` | Click element | `pipeline/steps/browser.ts` |
| `type` | Type text | `pipeline/steps/browser.ts` |
| `wait` | Wait for time/element | `pipeline/steps/browser.ts` |
| `press` | Press keyboard key | `pipeline/steps/browser.ts` |
| `snapshot` | Take DOM snapshot | `pipeline/steps/browser.ts` |
| `evaluate` | Run JavaScript in browser | `pipeline/steps/browser.ts` |
| `fetch` | HTTP fetch with custom headers | `pipeline/steps/fetch.ts` |
| `select` | JSONPath/dot-path data selection | `pipeline/steps/transform.ts` |
| `map` | Transform array items | `pipeline/steps/transform.ts` |
| `filter` | Filter array items | `pipeline/steps/transform.ts` |
| `sort` | Sort array items | `pipeline/steps/transform.ts` |
| `limit` | Limit result count | `pipeline/steps/transform.ts` |
| `intercept` | Install XHR interceptor | `pipeline/steps/intercept.ts` |
| `tap` | Debug output (side effect) | `pipeline/steps/tap.ts` |

### Template Interpolation

YAML values containing `${{ expr }}` are evaluated as JavaScript with these variables in scope:
- `item` — current array element (in map/filter)
- `index` — current index (in map)
- `data` — current pipeline data
- `args` — CLI arguments from user

### Supported Sites (17)

| Site | Commands | Strategy |
|------|----------|----------|
| twitter | search, timeline, profile, followers, following, like, reply, post, delete, notifications, trending, bookmarks | INTERCEPT, COOKIE |
| bilibili | search, hot, ranking, feed, dynamic, history, favorite, following, me, user-videos, subtitle | COOKIE, PUBLIC |
| youtube | search | PUBLIC |
| hackernews | top | PUBLIC |
| v2ex | hot, latest, topic, daily, me, notifications | COOKIE, PUBLIC |
| coupang | search, add-to-cart | COOKIE |
| bbc | news | PUBLIC |
| reuters | search | PUBLIC |
| weibo | hot | PUBLIC |
| xiaohongshu | search, user | INTERCEPT |
| zhihu | question | PUBLIC |
| boss | search | COOKIE |
| smzdm | search | PUBLIC |
| yahoo-finance | quote | PUBLIC |
| ctrip | search | PUBLIC |
| xueqiu | search, stock, hot-stock | PUBLIC |
| reddit | (via YAML) | PUBLIC |

## Tools Available

You have two MCP servers configured — use both for accurate specs:

### GitNexus MCP (architecture-level: clusters, execution flows, impact)

| Tool | Purpose | Example |
|------|---------|---------|
| `gitnexus_query` | Find execution flows by concept | `gitnexus_query({query: "pipeline step", repo: "opencli"})` |
| `gitnexus_context` | 360-degree symbol view | `gitnexus_context({name: "stepFetch", repo: "opencli"})` |
| `gitnexus_impact` | Blast radius analysis | `gitnexus_impact({target: "stepNavigate", direction: "upstream", repo: "opencli"})` |
| `gitnexus_cypher` | Direct graph queries | `gitnexus_cypher({query: "MATCH (f:Function) WHERE f.name STARTS WITH 'step' RETURN f.name, f.filePath", repo: "opencli"})` |

### ABCoder MCP (symbol-level: AST nodes, signatures, cross-file deps)

| Tool | Purpose | Example |
|------|---------|---------|
| `get_repo_structure` | Full file listing | `get_repo_structure({repo_name: "opencli"})` |
| `get_file_structure` | All nodes in a file | `get_file_structure({repo_name: "opencli", file_path: "src/pipeline/steps/browser.ts"})` |
| `get_ast_node` | Code + deps + refs | `get_ast_node({repo_name: "opencli", node_ids: [{mod_path: "@jackwener/opencli", pkg_path: "src/pipeline/steps/fetch.ts", name: "stepFetch"}]})` |

### Recommended Workflow

1. GitNexus first — find relevant execution flows
2. ABCoder second — get exact function signatures and code
3. Read YAML/TS CLI files — for real examples
4. Write specs — with real code examples from steps 2-3

## Files to Fill

### `.trellis/spec/cli-patterns/yaml-pipeline.md`
- Full YAML schema documentation (site, name, description, domain, args, pipeline, columns)
- Args definition syntax (name, type, default, required, help, choices)
- Pipeline step ordering best practices
- Template interpolation (`${{ expr }}`) — what's available in scope
- Real examples: read `src/clis/twitter/trending.yaml`, `src/clis/hackernews/top.yaml`, `src/clis/bilibili/hot.yaml`, `src/clis/v2ex/hot.yaml`
- When to choose YAML over TypeScript
- Common mistakes in YAML definitions

### `.trellis/spec/cli-patterns/typescript-cli.md`
- `cli()` builder function API
- CliCommand and CliOptions interfaces explained
- Using `Strategy` enum correctly
- The `func` callback signature: `(page: IPage, kwargs: Record<string, any>, debug?: boolean) => Promise<any>`
- Pattern: navigate → intercept/evaluate → extract → return
- Real examples: read `src/clis/twitter/search.ts`, `src/clis/bilibili/search.ts`, `src/clis/coupang/search.ts`
- When to choose TypeScript over YAML
- Error handling within `func`

### `.trellis/spec/cli-patterns/pipeline-steps-reference.md`
- Complete reference for ALL 15 pipeline steps
- For each step: name, params, input/output types, example usage
- Read ALL files in `src/pipeline/steps/` to get exact signatures
- Template interpolation in step params
- Step chaining patterns (navigate → evaluate → select → map → limit)
- Read `src/pipeline/template.ts` for interpolation logic

### `.trellis/spec/cli-patterns/strategy-guide.md`
- Detailed guide for each Strategy enum value
- Decision tree: when to use which strategy
- Real examples from the codebase for each strategy
- How strategy affects browser session management
- Cookie extraction patterns
- Header injection patterns
- Intercept patterns (installInterceptor + getInterceptedRequests)
- Read `src/clis/twitter/search.ts` (INTERCEPT), `src/clis/bilibili/search.ts` (COOKIE), `src/clis/hackernews/top.yaml` (PUBLIC)

### `.trellis/spec/cli-patterns/adding-new-site.md`
- Step-by-step tutorial for adding a new website
- Step 1: Choose YAML vs TypeScript
- Step 2: Create file in `src/clis/{site}/{command}.ts|yaml`
- Step 3: Define site, name, args, columns
- Step 4: Implement data extraction (pipeline or func)
- Step 5: Test with `npx tsx src/main.ts run site:command`
- Step 6: Run `npm run build` to update manifest
- Checklist for new site PRs
- Tips for reverse-engineering website APIs

## Important Rules

### Spec files are NOT fixed — adapt to reality
- Delete template files that don't apply
- Create new files for patterns templates don't cover
- Rename files if template names don't fit
- Update index.md to reflect the final set

### Parallel agents — stay in your lane
- ONLY modify files under `.trellis/spec/cli-patterns/`
- DO NOT modify source code, other spec directories, or task files
- DO NOT run git commands
- You may read any file for analysis

## Acceptance Criteria

- [ ] Real code examples from the actual codebase (with file paths and line numbers)
- [ ] Complete pipeline steps reference (all 15 steps documented)
- [ ] Anti-patterns documented (what NOT to do)
- [ ] No placeholder text remaining
- [ ] index.md reflects actual file set
- [ ] Each file is at least 80 lines with substantive content
- [ ] Adding-new-site guide is actionable and complete

## Technical Notes

- YAML CLIs are discovered by `src/engine.ts:loadFromManifest()` via `manifest.json`
- TS CLIs self-register via `cli()` which calls `registerCommand()`
- `manifest.json` is built by `src/build-manifest.ts` during `npm run build`
- Template engine is in `src/pipeline/template.ts`
- All pipeline steps conform to `StepHandler` type: `(page, params, data, args) => Promise<any>`
