# Node Hotlist Collector Design

## Goal

Build a complete Node.js data collection system under `数据采集/` that reproduces the current Python project's hotlist acquisition capabilities, adds compatibility aliases from README and related naming drift, and ships with root-level documentation files `数据收集.md`, `API.md`, and `介绍.md`.

The new system must collect from upstream APIs, HTML pages, and dynamic pages when needed, normalize all results into a strict JSON schema, and expose a unified execution entry for single-platform and multi-platform collection.

## Scope

### Included Platforms

The implementation will cover the platform set registered in `app/services/sites/factory.py` plus compatibility aliases:

- `baidu`
- `shaoshupai` with alias `sspai`
- `weibo`
- `zhihu`
- `36kr` with alias `tskr`
- `52pojie` with alias `ftpojie`
- `bilibili`
- `douban`
- `hupu`
- `tieba`
- `juejin`
- `douyin`
- `v2ex` with alias `vtex`
- `jinritoutiao`
- `tenxunwang`
- `stackoverflow`
- `github`
- `hackernews`
- `sina_finance`
- `eastmoney`
- `xueqiu`
- `cls`

### Compatibility Layer

The Node implementation will accept canonical IDs and alias IDs. Compatibility IDs will resolve to the same crawler implementation and will be preserved in documentation:

- `shaoshupai -> sspai`
- `36kr -> tskr`
- `52pojie -> ftpojie`
- `v2ex -> vtex`
- `weixin` documented as unsupported in the active Python factory but reserved as an optional extension slot for future dynamic collection

### Out of Scope

- Rebuilding the Python-side FastAPI analysis endpoints
- Reproducing MySQL or Redis persistence in the first implementation pass
- Shipping a browser UI

## Architecture

### Top-Level Layout

The repository root will contain:

- `数据收集.md` for upstream data source inventory, availability, response shapes, and scraping strategies
- `API.md` for both upstream API definitions and the Node collector's own programmatic interfaces and CLI contracts
- `介绍.md` for architecture, runtime setup, commands, extension guide, and maintenance notes
- `数据采集/` for the executable Node project

The `数据采集/` project will follow this structure:

```text
数据采集/
  package.json
  .env.example
  src/
    cli/
    config/
    core/
    platforms/
      aliases/
    scheduler/
    schemas/
    utils/
    index.js
  test/
```

### Module Responsibilities

#### `src/config/`

Loads environment variables from `.env`, validates runtime options, and exposes immutable configuration for HTTP timeout, retry limits, concurrency, proxy settings, user agent, and optional tokens.

#### `src/core/`

Provides shared primitives:

- `httpClient` with timeout, proxy, headers, retry, and response decoding
- `rateLimiter` for per-platform request pacing
- `concurrencyPool` for bounded parallel execution
- `backoff` using exponential delay with jitter
- `errors` for typed operational failures
- `logger` for structured console output

#### `src/platforms/`

One crawler module per platform. Each crawler exposes a consistent async `collect` function that returns normalized data plus metadata. Platform modules declare source type, aliases, upstream endpoints, headers, parser logic, and fallback strategy.

#### `src/platforms/aliases/`

Contains alias resolution tables and helper functions to map README or historical names to canonical crawler IDs.

#### `src/schemas/`

Defines runtime validation for standardized output objects and error payloads. Validation ensures all crawlers align on the same JSON structure before returning results.

#### `src/scheduler/`

Implements orchestration for:

- single-platform collection
- multi-platform collection
- collect-all runs
- optional periodic execution in-process

The scheduler will enforce concurrency caps, isolate crawler failures, and aggregate per-platform results into one report object.

#### `src/cli/`

Provides command-line entrypoints such as:

- collect one platform
- collect multiple platforms
- collect all platforms
- print platform list
- output JSON to stdout or file

## Data Model

### Normalized Collector Response

Each successful collection returns:

```json
{
  "platform": "baidu",
  "canonicalPlatform": "baidu",
  "aliases": ["baidu"],
  "displayName": "百度热搜",
  "sourceType": "json-api",
  "sourceUrl": "https://top.baidu.com/api/board?platform=wise&tab=realtime",
  "fetchedAt": "2026-04-02T21:00:00.000Z",
  "success": true,
  "items": [
    {
      "id": "baidu-1",
      "rank": 1,
      "title": "示例标题",
      "url": "https://example.com",
      "mobileUrl": null,
      "hot": "4955232",
      "summary": "",
      "author": null,
      "category": null,
      "tags": [],
      "publishTime": null,
      "metadata": {},
      "raw": {}
    }
  ],
  "warnings": []
}
```

### Error Response

Each failed platform still returns a structured payload:

```json
{
  "platform": "weibo",
  "canonicalPlatform": "weibo",
  "aliases": ["weibo"],
  "displayName": "微博热搜",
  "sourceType": "json-api",
  "sourceUrl": "https://weibo.com/ajax/side/hotSearch",
  "fetchedAt": "2026-04-02T21:00:00.000Z",
  "success": false,
  "items": [],
  "warnings": ["upstream returned 403"],
  "error": {
    "code": "UPSTREAM_HTTP_ERROR",
    "message": "Request failed with status 403",
    "retryable": true
  }
}
```

### Aggregated Run Response

Multi-platform execution returns:

```json
{
  "requestedPlatforms": ["baidu", "weibo"],
  "resolvedPlatforms": ["baidu", "weibo"],
  "startedAt": "2026-04-02T21:00:00.000Z",
  "finishedAt": "2026-04-02T21:00:03.000Z",
  "successCount": 1,
  "failureCount": 1,
  "results": []
}
```

## Collection Strategy by Source Type

### JSON API Platforms

Default to direct HTTP requests with explicit headers and retry. This group includes `baidu`, `bilibili`, `cls`, `douyin`, `eastmoney`, `github`, `jinritoutiao`, `juejin`, `shaoshupai`, `stackoverflow`, `tenxunwang`, `tieba`, `weibo`, `zhihu`, `sina_finance`, `xueqiu`, and `36kr`.

Handling rules:

- prefer upstream JSON endpoint when available
- include platform-specific headers where needed
- use retry only for transient network and 5xx failures
- for anti-bot targets such as `weibo` and `xueqiu`, allow cookie injection and proxy configuration
- retain raw upstream record in the normalized `raw` field for debugging

### HTML Parsing Platforms

Use direct HTTP fetch plus `cheerio` parsing. This group includes `52pojie`, `douban`, `hupu`, `v2ex`, and the primary path for `hackernews`.

Handling rules:

- fetch HTML with realistic user-agent and language headers
- parse only stable structural selectors
- normalize relative URLs to absolute URLs
- add warnings when selectors partially fail but enough data remains to proceed

### Dynamic / Browser Fallback Platforms

Use Playwright only as a fallback or explicit mode for platforms where static HTTP parsing is not reliable. The first implementation will prepare browser fallback helpers without making them the default path except where required.

Handling rules:

- browser mode is opt-in globally or per-platform
- fallback triggers only after static strategy fails with a retryable cause
- browser logic stays isolated in `core/browserClient` to avoid platform duplication

## Anti-Bot and Resilience Strategy

The collector will include the following defenses:

- exponential backoff with jitter for transient failures
- request timeout guardrails
- per-platform rate limits to avoid burst traffic
- bounded concurrency to reduce coordinated detection risk
- optional HTTP and HTTPS proxy from `.env`
- configurable cookie headers for sites with session requirements
- upstream preheat flow for `xueqiu`
- header profiles with platform overrides

The documentation will explicitly mark which platforms currently appear directly usable from public network tests and which are conditional or blocked.

## Testing Strategy

The implementation will follow TDD.

### Red-Green Sequence

1. write schema validation and alias resolution tests
2. write HTTP client retry, timeout, and rate-limit behavior tests
3. write scheduler aggregation tests
4. write one platform parser test per crawler using recorded fixtures
5. write integration-style tests for collect-one and collect-all commands

### Test Shape

- use fixture-based tests for upstream JSON and HTML snapshots
- avoid hitting live upstream services in unit tests
- reserve live smoke checks for optional manual verification scripts
- verify normalized output shape for every crawler

## Documentation Deliverables

### `数据收集.md`

Will describe:

- all upstream APIs and pages used by the Python project and Node rewrite
- source platform descriptions
- example response shapes
- current availability assessment
- collection strategy per platform
- anti-bot notes and known failure modes

### `API.md`

Will define:

- upstream endpoint URL
- method
- authentication or cookie needs
- parameters
- response fields
- availability status
- example invocations
- Node collector entrypoints and standardized output contracts

### `介绍.md`

Will explain:

- folder structure of `数据采集/`
- role of each module
- environment variables and `.env` usage
- install and run commands
- maintenance and extension guide
- how to add a new platform crawler

## Implementation Phases

### Phase 1

Create Node project skeleton, configuration loader, shared schemas, alias registry, and core utilities.

### Phase 2

Implement scheduler and CLI with one full crawler end-to-end to prove the architecture.

### Phase 3

Implement all JSON API platforms.

### Phase 4

Implement all HTML parsing platforms and browser fallback hooks.

### Phase 5

Complete root documentation, fixture tests, and verification commands.

## Risks and Mitigations

### Upstream Instability

Many hotlist endpoints can change fields or block requests. Mitigation is fixture-backed parsers, defensive null handling, and warning-rich result payloads.

### Anti-Bot Escalation

Some targets such as `weibo`, `xueqiu`, and dynamic pages may intermittently reject requests. Mitigation is proxy support, cookies, fallback headers, and optional browser mode.

### Naming Drift

The Python project, README, and file names do not always match. Mitigation is a formal alias map and canonical platform registry.

### Over-Coupled Platform Logic

Crawler-specific hacks can spread into shared modules. Mitigation is strict separation between core transport primitives and platform-specific adapters.

## Success Criteria

The design is considered successful when:

- every registered Python platform has a matching Node crawler module or an explicitly documented compatibility mapping
- all crawlers return validated normalized JSON
- root-level documentation fully covers APIs, sources, and strategies
- the scheduler can collect one, many, or all platforms without aborting on partial failure
- configuration and maintenance are clear from `介绍.md`
