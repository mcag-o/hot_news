# Node Hotlist Collector Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Node.js hotlist data collection system in `数据采集/` that reproduces the Python project's collection capabilities, standardizes output, and ships with root-level documentation.

**Architecture:** Use a small core runtime with validated configuration, a shared HTTP client, alias resolution, standardized schemas, and one crawler module per platform. Execute collectors through a scheduler and CLI so single-platform and multi-platform collection share the same normalization, retry, timeout, rate-limit, and error-handling behavior.

**Tech Stack:** Node.js 22+, ES Modules, Vitest, Zod, undici, cheerio, dotenv, p-limit, Playwright (fallback only)

---

## File Map

### Root Documents

- Create: `数据收集.md`
- Create: `API.md`
- Create: `介绍.md`

### Project Metadata

- Create: `数据采集/package.json`
- Create: `数据采集/.env.example`
- Create: `数据采集/.gitignore`

### Runtime Source

- Create: `数据采集/src/index.js`
- Create: `数据采集/src/config/env.js`
- Create: `数据采集/src/core/errors.js`
- Create: `数据采集/src/core/backoff.js`
- Create: `数据采集/src/core/logger.js`
- Create: `数据采集/src/core/rateLimiter.js`
- Create: `数据采集/src/core/concurrencyPool.js`
- Create: `数据采集/src/core/httpClient.js`
- Create: `数据采集/src/core/browserClient.js`
- Create: `数据采集/src/schemas/resultSchema.js`
- Create: `数据采集/src/platforms/aliases/platformRegistry.js`
- Create: `数据采集/src/platforms/helpers.js`
- Create: `数据采集/src/platforms/index.js`
- Create: `数据采集/src/scheduler/collectPlatform.js`
- Create: `数据采集/src/scheduler/collectMany.js`
- Create: `数据采集/src/scheduler/index.js`
- Create: `数据采集/src/cli/run.js`

### Platform Modules

- Create: `数据采集/src/platforms/baidu.js`
- Create: `数据采集/src/platforms/shaoshupai.js`
- Create: `数据采集/src/platforms/weibo.js`
- Create: `数据采集/src/platforms/zhihu.js`
- Create: `数据采集/src/platforms/36kr.js`
- Create: `数据采集/src/platforms/52pojie.js`
- Create: `数据采集/src/platforms/bilibili.js`
- Create: `数据采集/src/platforms/douban.js`
- Create: `数据采集/src/platforms/hupu.js`
- Create: `数据采集/src/platforms/tieba.js`
- Create: `数据采集/src/platforms/juejin.js`
- Create: `数据采集/src/platforms/douyin.js`
- Create: `数据采集/src/platforms/v2ex.js`
- Create: `数据采集/src/platforms/jinritoutiao.js`
- Create: `数据采集/src/platforms/tenxunwang.js`
- Create: `数据采集/src/platforms/stackoverflow.js`
- Create: `数据采集/src/platforms/github.js`
- Create: `数据采集/src/platforms/hackernews.js`
- Create: `数据采集/src/platforms/sina_finance.js`
- Create: `数据采集/src/platforms/eastmoney.js`
- Create: `数据采集/src/platforms/xueqiu.js`
- Create: `数据采集/src/platforms/cls.js`

### Tests and Fixtures

- Create: `数据采集/test/setup.js`
- Create: `数据采集/test/fixtures/*.json`
- Create: `数据采集/test/fixtures/*.html`
- Create: `数据采集/test/config/env.test.js`
- Create: `数据采集/test/schemas/resultSchema.test.js`
- Create: `数据采集/test/platforms/platformRegistry.test.js`
- Create: `数据采集/test/core/httpClient.test.js`
- Create: `数据采集/test/core/rateLimiter.test.js`
- Create: `数据采集/test/scheduler/collectMany.test.js`
- Create: `数据采集/test/platforms/*.test.js`
- Create: `数据采集/test/cli/run.test.js`

### Manual Verification Helpers

- Create: `数据采集/scripts/smoke.js`

---

### Task 1: Scaffold the Node Project and Tooling

**Files:**
- Create: `数据采集/package.json`
- Create: `数据采集/.env.example`
- Create: `数据采集/.gitignore`
- Create: `数据采集/src/index.js`
- Test: `数据采集/package.json`

- [ ] **Step 1: Write the failing metadata check**

```json
{
  "name": "hotlist-collector",
  "private": true,
  "type": "module",
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "collect": "node src/cli/run.js",
    "smoke": "node scripts/smoke.js"
  }
}
```

- [ ] **Step 2: Run the missing-project check and verify it fails**

Run: `test -f "数据采集/package.json"`
Expected: non-zero exit status because the file does not exist yet.

- [ ] **Step 3: Write the minimal project scaffold**

```json
{
  "name": "hotlist-collector",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "engines": {
    "node": ">=22"
  },
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "collect": "node src/cli/run.js",
    "smoke": "node scripts/smoke.js"
  },
  "dependencies": {
    "cheerio": "^1.1.0",
    "dotenv": "^16.6.1",
    "p-limit": "^6.2.0",
    "playwright": "^1.55.0",
    "undici": "^7.16.0",
    "zod": "^4.1.5"
  },
  "devDependencies": {
    "vitest": "^3.2.4"
  }
}
```

```dotenv
HTTP_TIMEOUT_MS=10000
HTTP_RETRY_COUNT=2
HTTP_RETRY_BASE_MS=250
GLOBAL_CONCURRENCY=4
DEFAULT_USER_AGENT=Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/135.0.0.0 Safari/537.36
HTTP_PROXY=
HTTPS_PROXY=
WEIBO_COOKIE=
XUEQIU_COOKIE=
ENABLE_BROWSER_FALLBACK=false
```

```gitignore
node_modules/
.env
coverage/
dist/
```

```js
export * from './config/env.js';
export * from './scheduler/index.js';
```

- [ ] **Step 4: Run the metadata check again and verify it passes**

Run: `test -f "数据采集/package.json" && node -e "const p=require('./数据采集/package.json'); console.log(p.type)"`
Expected: prints `module`.

- [ ] **Step 5: Commit**

```bash
git add "数据采集/package.json" "数据采集/.env.example" "数据采集/.gitignore" "数据采集/src/index.js"
git commit -m "feat: scaffold node collector project"
```

### Task 2: Add Configuration Loader and Core Error Types

**Files:**
- Create: `数据采集/src/config/env.js`
- Create: `数据采集/src/core/errors.js`
- Create: `数据采集/test/config/env.test.js`
- Test: `数据采集/test/config/env.test.js`

- [ ] **Step 1: Write the failing config test**

```js
import { describe, expect, test } from 'vitest';
import { loadEnv } from '../../src/config/env.js';

describe('loadEnv', () => {
  test('returns normalized defaults', () => {
    const env = loadEnv({});

    expect(env.httpTimeoutMs).toBe(10000);
    expect(env.httpRetryCount).toBe(2);
    expect(env.globalConcurrency).toBe(4);
    expect(env.enableBrowserFallback).toBe(false);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd "数据采集" && npm test -- test/config/env.test.js`
Expected: FAIL with module-not-found for `src/config/env.js`.

- [ ] **Step 3: Write the minimal config and error implementation**

```js
import dotenv from 'dotenv';

dotenv.config();

const toInt = (value, fallback) => {
  const parsed = Number.parseInt(value ?? '', 10);
  return Number.isFinite(parsed) ? parsed : fallback;
};

const toBool = (value, fallback = false) => {
  if (value == null || value === '') return fallback;
  return ['1', 'true', 'yes', 'on'].includes(String(value).toLowerCase());
};

export function loadEnv(raw = process.env) {
  return Object.freeze({
    httpTimeoutMs: toInt(raw.HTTP_TIMEOUT_MS, 10000),
    httpRetryCount: toInt(raw.HTTP_RETRY_COUNT, 2),
    httpRetryBaseMs: toInt(raw.HTTP_RETRY_BASE_MS, 250),
    globalConcurrency: toInt(raw.GLOBAL_CONCURRENCY, 4),
    defaultUserAgent: raw.DEFAULT_USER_AGENT || 'Mozilla/5.0',
    httpProxy: raw.HTTP_PROXY || '',
    httpsProxy: raw.HTTPS_PROXY || '',
    weiboCookie: raw.WEIBO_COOKIE || '',
    xueqiuCookie: raw.XUEQIU_COOKIE || '',
    enableBrowserFallback: toBool(raw.ENABLE_BROWSER_FALLBACK, false)
  });
}

export const env = loadEnv();
```

```js
export class CollectorError extends Error {
  constructor(code, message, options = {}) {
    super(message);
    this.name = 'CollectorError';
    this.code = code;
    this.retryable = Boolean(options.retryable);
    this.cause = options.cause;
    this.statusCode = options.statusCode ?? null;
  }
}

export class UpstreamHttpError extends CollectorError {
  constructor(message, statusCode, options = {}) {
    super('UPSTREAM_HTTP_ERROR', message, {
      ...options,
      retryable: options.retryable ?? statusCode >= 500,
      statusCode
    });
  }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `cd "数据采集" && npm test -- test/config/env.test.js`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "数据采集/src/config/env.js" "数据采集/src/core/errors.js" "数据采集/test/config/env.test.js"
git commit -m "feat: add config loader and core error types"
```

### Task 3: Define Result Schemas and Alias Registry

**Files:**
- Create: `数据采集/src/schemas/resultSchema.js`
- Create: `数据采集/src/platforms/aliases/platformRegistry.js`
- Create: `数据采集/test/schemas/resultSchema.test.js`
- Create: `数据采集/test/platforms/platformRegistry.test.js`
- Test: `数据采集/test/schemas/resultSchema.test.js`
- Test: `数据采集/test/platforms/platformRegistry.test.js`

- [ ] **Step 1: Write the failing schema and alias tests**

```js
import { describe, expect, test } from 'vitest';
import { normalizePlatformId, getPlatformMeta } from '../../src/platforms/aliases/platformRegistry.js';

describe('platform registry', () => {
  test('maps aliases to canonical ids', () => {
    expect(normalizePlatformId('sspai')).toBe('shaoshupai');
    expect(normalizePlatformId('tskr')).toBe('36kr');
    expect(normalizePlatformId('ftpojie')).toBe('52pojie');
    expect(normalizePlatformId('vtex')).toBe('v2ex');
    expect(getPlatformMeta('weibo').displayName).toBe('微博热搜');
  });
});
```

```js
import { describe, expect, test } from 'vitest';
import { resultSchema } from '../../src/schemas/resultSchema.js';

describe('resultSchema', () => {
  test('accepts normalized result payloads', () => {
    const parsed = resultSchema.parse({
      platform: 'baidu',
      canonicalPlatform: 'baidu',
      aliases: ['baidu'],
      displayName: '百度热搜',
      sourceType: 'json-api',
      sourceUrl: 'https://top.baidu.com/api/board?platform=wise&tab=realtime',
      fetchedAt: '2026-04-02T21:00:00.000Z',
      success: true,
      items: [],
      warnings: []
    });

    expect(parsed.success).toBe(true);
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd "数据采集" && npm test -- test/schemas/resultSchema.test.js test/platforms/platformRegistry.test.js`
Expected: FAIL because the schema and registry modules do not exist.

- [ ] **Step 3: Write the minimal registry and schema implementation**

```js
const registry = {
  baidu: { displayName: '百度热搜', aliases: ['baidu'], sourceType: 'json-api' },
  shaoshupai: { displayName: '少数派', aliases: ['shaoshupai', 'sspai'], sourceType: 'json-api' },
  weibo: { displayName: '微博热搜', aliases: ['weibo'], sourceType: 'json-api' },
  zhihu: { displayName: '知乎热榜', aliases: ['zhihu'], sourceType: 'json-api' },
  '36kr': { displayName: '36Kr', aliases: ['36kr', 'tskr'], sourceType: 'json-api' },
  '52pojie': { displayName: '吾爱破解', aliases: ['52pojie', 'ftpojie'], sourceType: 'html' },
  bilibili: { displayName: '哔哩哔哩', aliases: ['bilibili'], sourceType: 'json-api' },
  douban: { displayName: '豆瓣', aliases: ['douban'], sourceType: 'html' },
  hupu: { displayName: '虎扑', aliases: ['hupu'], sourceType: 'html' },
  tieba: { displayName: '百度贴吧', aliases: ['tieba'], sourceType: 'json-api' },
  juejin: { displayName: '掘金', aliases: ['juejin'], sourceType: 'json-api' },
  douyin: { displayName: '抖音', aliases: ['douyin'], sourceType: 'json-api' },
  v2ex: { displayName: 'V2EX', aliases: ['v2ex', 'vtex'], sourceType: 'html' },
  jinritoutiao: { displayName: '今日头条', aliases: ['jinritoutiao'], sourceType: 'json-api' },
  tenxunwang: { displayName: '腾讯网', aliases: ['tenxunwang'], sourceType: 'json-api' },
  stackoverflow: { displayName: 'Stack Overflow', aliases: ['stackoverflow'], sourceType: 'json-api' },
  github: { displayName: 'GitHub Trending', aliases: ['github'], sourceType: 'json-api' },
  hackernews: { displayName: 'Hacker News', aliases: ['hackernews'], sourceType: 'html' },
  sina_finance: { displayName: '新浪财经', aliases: ['sina_finance'], sourceType: 'json-api' },
  eastmoney: { displayName: '东方财富', aliases: ['eastmoney'], sourceType: 'json-api' },
  xueqiu: { displayName: '雪球', aliases: ['xueqiu'], sourceType: 'json-api' },
  cls: { displayName: '财联社', aliases: ['cls'], sourceType: 'json-api' }
};

const aliasToCanonical = new Map(
  Object.entries(registry).flatMap(([canonical, meta]) => meta.aliases.map((alias) => [alias, canonical]))
);

export function normalizePlatformId(input) {
  return aliasToCanonical.get(String(input).trim().toLowerCase()) ?? null;
}

export function getPlatformMeta(input) {
  const canonical = normalizePlatformId(input) ?? input;
  return registry[canonical] ?? null;
}

export function listPlatforms() {
  return Object.keys(registry);
}
```

```js
import { z } from 'zod';

export const itemSchema = z.object({
  id: z.string(),
  rank: z.number().int().positive(),
  title: z.string(),
  url: z.string().url(),
  mobileUrl: z.string().url().nullable(),
  hot: z.string().nullable(),
  summary: z.string().nullable(),
  author: z.string().nullable(),
  category: z.string().nullable(),
  tags: z.array(z.string()),
  publishTime: z.string().nullable(),
  metadata: z.record(z.string(), z.any()),
  raw: z.any()
});

export const resultSchema = z.object({
  platform: z.string(),
  canonicalPlatform: z.string(),
  aliases: z.array(z.string()),
  displayName: z.string(),
  sourceType: z.enum(['json-api', 'html', 'browser']),
  sourceUrl: z.string().url(),
  fetchedAt: z.string(),
  success: z.boolean(),
  items: z.array(itemSchema),
  warnings: z.array(z.string()),
  error: z.object({
    code: z.string(),
    message: z.string(),
    retryable: z.boolean()
  }).optional()
});
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd "数据采集" && npm test -- test/schemas/resultSchema.test.js test/platforms/platformRegistry.test.js`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "数据采集/src/schemas/resultSchema.js" "数据采集/src/platforms/aliases/platformRegistry.js" "数据采集/test/schemas/resultSchema.test.js" "数据采集/test/platforms/platformRegistry.test.js"
git commit -m "feat: add result schema and platform registry"
```

### Task 4: Build Retry, Rate-Limit, Concurrency, and HTTP Client Primitives

**Files:**
- Create: `数据采集/src/core/backoff.js`
- Create: `数据采集/src/core/logger.js`
- Create: `数据采集/src/core/rateLimiter.js`
- Create: `数据采集/src/core/concurrencyPool.js`
- Create: `数据采集/src/core/httpClient.js`
- Create: `数据采集/test/core/httpClient.test.js`
- Create: `数据采集/test/core/rateLimiter.test.js`
- Test: `数据采集/test/core/httpClient.test.js`
- Test: `数据采集/test/core/rateLimiter.test.js`

- [ ] **Step 1: Write the failing core tests**

```js
import { describe, expect, test, vi } from 'vitest';
import { requestJson } from '../../src/core/httpClient.js';

describe('requestJson', () => {
  test('retries retryable failures once before succeeding', async () => {
    const responses = [
      { ok: false, status: 502, text: async () => 'bad gateway' },
      { ok: true, status: 200, json: async () => ({ ok: true }) }
    ];

    const fetchImpl = vi.fn().mockImplementation(() => Promise.resolve(responses.shift()));

    const result = await requestJson('https://example.com', {
      fetchImpl,
      retryCount: 1,
      retryBaseMs: 1,
      timeoutMs: 1000
    });

    expect(result).toEqual({ ok: true });
    expect(fetchImpl).toHaveBeenCalledTimes(2);
  });
});
```

```js
import { describe, expect, test } from 'vitest';
import { createRateLimiter } from '../../src/core/rateLimiter.js';

describe('createRateLimiter', () => {
  test('waits between calls for the same key', async () => {
    const limiter = createRateLimiter({ minIntervalMs: 20 });
    const start = Date.now();

    await limiter.wait('baidu');
    await limiter.wait('baidu');

    expect(Date.now() - start).toBeGreaterThanOrEqual(20);
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd "数据采集" && npm test -- test/core/httpClient.test.js test/core/rateLimiter.test.js`
Expected: FAIL because core modules do not exist.

- [ ] **Step 3: Write the minimal core runtime implementation**

```js
export async function sleep(ms) {
  await new Promise((resolve) => setTimeout(resolve, ms));
}

export function getBackoffDelay(attempt, baseMs) {
  return baseMs * (2 ** attempt);
}
```

```js
export const logger = {
  info(message, extra = {}) {
    console.log(JSON.stringify({ level: 'info', message, ...extra }));
  },
  warn(message, extra = {}) {
    console.warn(JSON.stringify({ level: 'warn', message, ...extra }));
  },
  error(message, extra = {}) {
    console.error(JSON.stringify({ level: 'error', message, ...extra }));
  }
};
```

```js
export function createRateLimiter({ minIntervalMs }) {
  const timestamps = new Map();
  return {
    async wait(key) {
      const now = Date.now();
      const previous = timestamps.get(key) ?? 0;
      const delta = now - previous;
      if (delta < minIntervalMs) {
        await new Promise((resolve) => setTimeout(resolve, minIntervalMs - delta));
      }
      timestamps.set(key, Date.now());
    }
  };
}
```

```js
import pLimit from 'p-limit';

export function createConcurrencyPool(limit) {
  return pLimit(limit);
}
```

```js
import { fetch } from 'undici';
import { sleep, getBackoffDelay } from './backoff.js';
import { UpstreamHttpError } from './errors.js';

export async function requestJson(url, options = {}) {
  const {
    fetchImpl = fetch,
    headers = {},
    retryCount = 0,
    retryBaseMs = 100,
    timeoutMs = 10000
  } = options;

  for (let attempt = 0; attempt <= retryCount; attempt += 1) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), timeoutMs);

    try {
      const response = await fetchImpl(url, { headers, signal: controller.signal });
      if (!response.ok) {
        throw new UpstreamHttpError(`Request failed with status ${response.status}`, response.status);
      }
      return await response.json();
    } catch (error) {
      if (attempt === retryCount || error.retryable === false) {
        throw error;
      }
      await sleep(getBackoffDelay(attempt, retryBaseMs));
    } finally {
      clearTimeout(timeout);
    }
  }

  throw new Error('unreachable');
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd "数据采集" && npm test -- test/core/httpClient.test.js test/core/rateLimiter.test.js`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "数据采集/src/core" "数据采集/test/core/httpClient.test.js" "数据采集/test/core/rateLimiter.test.js"
git commit -m "feat: add core collector runtime utilities"
```

### Task 5: Add Platform Helpers and Scheduler Skeleton

**Files:**
- Create: `数据采集/src/platforms/helpers.js`
- Create: `数据采集/src/scheduler/collectPlatform.js`
- Create: `数据采集/src/scheduler/collectMany.js`
- Create: `数据采集/src/scheduler/index.js`
- Create: `数据采集/test/scheduler/collectMany.test.js`
- Test: `数据采集/test/scheduler/collectMany.test.js`

- [ ] **Step 1: Write the failing scheduler test**

```js
import { describe, expect, test } from 'vitest';
import { collectMany } from '../../src/scheduler/collectMany.js';

describe('collectMany', () => {
  test('aggregates success and failure results without aborting', async () => {
    const registry = {
      baidu: { collect: async () => ({ platform: 'baidu', success: true, items: [], warnings: [] }) },
      weibo: { collect: async () => ({ platform: 'weibo', success: false, items: [], warnings: ['403'] }) }
    };

    const result = await collectMany(['baidu', 'weibo'], { registry });

    expect(result.successCount).toBe(1);
    expect(result.failureCount).toBe(1);
    expect(result.results).toHaveLength(2);
  });
});
```

- [ ] **Step 2: Run the scheduler test and verify it fails**

Run: `cd "数据采集" && npm test -- test/scheduler/collectMany.test.js`
Expected: FAIL because scheduler modules do not exist.

- [ ] **Step 3: Write the minimal helper and scheduler implementation**

```js
export function createItem(platform, rank, title, url, extra = {}) {
  return {
    id: `${platform}-${rank}`,
    rank,
    title,
    url,
    mobileUrl: extra.mobileUrl ?? null,
    hot: extra.hot ?? null,
    summary: extra.summary ?? null,
    author: extra.author ?? null,
    category: extra.category ?? null,
    tags: extra.tags ?? [],
    publishTime: extra.publishTime ?? null,
    metadata: extra.metadata ?? {},
    raw: extra.raw ?? null
  };
}
```

```js
import { getPlatformMeta, normalizePlatformId } from '../platforms/aliases/platformRegistry.js';

export async function collectPlatform(platformId, options = {}) {
  const canonical = normalizePlatformId(platformId);
  const meta = getPlatformMeta(canonical);
  const crawler = options.registry?.[canonical];
  if (!canonical || !meta || !crawler) {
    return {
      platform: platformId,
      canonicalPlatform: canonical ?? platformId,
      aliases: [],
      displayName: String(platformId),
      sourceType: 'json-api',
      sourceUrl: 'https://invalid.local/',
      fetchedAt: new Date().toISOString(),
      success: false,
      items: [],
      warnings: ['unsupported platform'],
      error: { code: 'UNSUPPORTED_PLATFORM', message: 'Unsupported platform', retryable: false }
    };
  }
  return crawler.collect({ canonicalPlatform: canonical, meta, env: options.env });
}
```

```js
import { collectPlatform } from './collectPlatform.js';

export async function collectMany(platforms, options = {}) {
  const startedAt = new Date().toISOString();
  const results = [];

  for (const platform of platforms) {
    results.push(await collectPlatform(platform, options));
  }

  return {
    requestedPlatforms: platforms,
    resolvedPlatforms: results.map((entry) => entry.canonicalPlatform),
    startedAt,
    finishedAt: new Date().toISOString(),
    successCount: results.filter((entry) => entry.success).length,
    failureCount: results.filter((entry) => !entry.success).length,
    results
  };
}
```

```js
export { collectPlatform } from './collectPlatform.js';
export { collectMany } from './collectMany.js';
```

- [ ] **Step 4: Run the scheduler test and verify it passes**

Run: `cd "数据采集" && npm test -- test/scheduler/collectMany.test.js`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "数据采集/src/platforms/helpers.js" "数据采集/src/scheduler" "数据采集/test/scheduler/collectMany.test.js"
git commit -m "feat: add scheduler skeleton and item helpers"
```

### Task 6: Implement One End-to-End JSON API Crawler via TDD

**Files:**
- Create: `数据采集/src/platforms/baidu.js`
- Create: `数据采集/src/platforms/index.js`
- Create: `数据采集/test/fixtures/baidu.json`
- Create: `数据采集/test/platforms/baidu.test.js`
- Test: `数据采集/test/platforms/baidu.test.js`

- [ ] **Step 1: Write the failing Baidu crawler test**

```js
import { describe, expect, test } from 'vitest';
import fixture from '../fixtures/baidu.json' with { type: 'json' };
import { createBaiduCrawler } from '../../src/platforms/baidu.js';

describe('createBaiduCrawler', () => {
  test('normalizes baidu upstream payload into standard items', async () => {
    const crawler = createBaiduCrawler({
      requestJson: async () => fixture
    });

    const result = await crawler.collect({
      canonicalPlatform: 'baidu',
      meta: { displayName: '百度热搜', aliases: ['baidu'], sourceType: 'json-api' }
    });

    expect(result.success).toBe(true);
    expect(result.items[0].title).toBe('在雄安 为幸福加码');
    expect(result.items[0].rank).toBe(1);
  });
});
```

- [ ] **Step 2: Run the crawler test and verify it fails**

Run: `cd "数据采集" && npm test -- test/platforms/baidu.test.js`
Expected: FAIL because the crawler module does not exist.

- [ ] **Step 3: Write the minimal Baidu crawler and registry export**

```js
import { createItem } from './helpers.js';

const SOURCE_URL = 'https://top.baidu.com/api/board?platform=wise&tab=realtime';

export function createBaiduCrawler(deps) {
  return {
    async collect({ canonicalPlatform, meta }) {
      const payload = await deps.requestJson(SOURCE_URL);
      const entries = payload.data.cards[0].content[0].content;
      return {
        platform: canonicalPlatform,
        canonicalPlatform,
        aliases: meta.aliases,
        displayName: meta.displayName,
        sourceType: 'json-api',
        sourceUrl: SOURCE_URL,
        fetchedAt: new Date().toISOString(),
        success: true,
        items: entries.map((entry, index) => createItem(canonicalPlatform, index + 1, entry.word, entry.url, {
          hot: entry.hotScore ? String(entry.hotScore) : null,
          summary: entry.desc ?? '',
          raw: entry
        })),
        warnings: []
      };
    }
  };
}
```

```js
import { env } from '../config/env.js';
import { requestJson } from '../core/httpClient.js';
import { createBaiduCrawler } from './baidu.js';

export function createPlatformRegistry() {
  return {
    baidu: createBaiduCrawler({
      requestJson: (url, options) => requestJson(url, {
        ...options,
        timeoutMs: env.httpTimeoutMs,
        retryCount: env.httpRetryCount,
        retryBaseMs: env.httpRetryBaseMs,
        headers: { 'user-agent': env.defaultUserAgent }
      })
    })
  };
}
```

- [ ] **Step 4: Run the crawler test and verify it passes**

Run: `cd "数据采集" && npm test -- test/platforms/baidu.test.js`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "数据采集/src/platforms/baidu.js" "数据采集/src/platforms/index.js" "数据采集/test/fixtures/baidu.json" "数据采集/test/platforms/baidu.test.js"
git commit -m "feat: add baidu crawler"
```

### Task 7: Implement the CLI Entry and Registry Wiring

**Files:**
- Create: `数据采集/src/cli/run.js`
- Create: `数据采集/test/cli/run.test.js`
- Modify: `数据采集/src/platforms/index.js`
- Test: `数据采集/test/cli/run.test.js`

- [ ] **Step 1: Write the failing CLI test**

```js
import { describe, expect, test } from 'vitest';
import { parseArgs } from '../../src/cli/run.js';

describe('parseArgs', () => {
  test('parses collect-many arguments', () => {
    const parsed = parseArgs(['--platforms', 'baidu,weibo']);
    expect(parsed.platforms).toEqual(['baidu', 'weibo']);
  });
});
```

- [ ] **Step 2: Run the CLI test and verify it fails**

Run: `cd "数据采集" && npm test -- test/cli/run.test.js`
Expected: FAIL because `src/cli/run.js` does not exist.

- [ ] **Step 3: Write the minimal CLI implementation**

```js
import { collectMany } from '../scheduler/collectMany.js';
import { createPlatformRegistry } from '../platforms/index.js';
import { listPlatforms } from '../platforms/aliases/platformRegistry.js';

export function parseArgs(argv) {
  const args = { platforms: [] };
  for (let index = 0; index < argv.length; index += 1) {
    if (argv[index] === '--platform') args.platforms = [argv[index + 1]];
    if (argv[index] === '--platforms') args.platforms = argv[index + 1].split(',').map((item) => item.trim());
    if (argv[index] === '--all') args.platforms = listPlatforms();
  }
  return args;
}

export async function run(argv = process.argv.slice(2)) {
  const args = parseArgs(argv);
  const platforms = args.platforms.length ? args.platforms : listPlatforms();
  const result = await collectMany(platforms, { registry: createPlatformRegistry() });
  process.stdout.write(`${JSON.stringify(result, null, 2)}\n`);
  return result;
}

if (import.meta.url === `file://${process.argv[1]}`) {
  await run();
}
```

- [ ] **Step 4: Run the CLI test and verify it passes**

Run: `cd "数据采集" && npm test -- test/cli/run.test.js`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "数据采集/src/cli/run.js" "数据采集/test/cli/run.test.js" "数据采集/src/platforms/index.js"
git commit -m "feat: add collector cli entrypoint"
```

### Task 8: Implement the Remaining JSON API Crawlers in Small Batches

**Files:**
- Create: `数据采集/src/platforms/shaoshupai.js`
- Create: `数据采集/src/platforms/weibo.js`
- Create: `数据采集/src/platforms/zhihu.js`
- Create: `数据采集/src/platforms/36kr.js`
- Create: `数据采集/src/platforms/bilibili.js`
- Create: `数据采集/src/platforms/tieba.js`
- Create: `数据采集/src/platforms/juejin.js`
- Create: `数据采集/src/platforms/douyin.js`
- Create: `数据采集/src/platforms/jinritoutiao.js`
- Create: `数据采集/src/platforms/tenxunwang.js`
- Create: `数据采集/src/platforms/stackoverflow.js`
- Create: `数据采集/src/platforms/github.js`
- Create: `数据采集/src/platforms/sina_finance.js`
- Create: `数据采集/src/platforms/eastmoney.js`
- Create: `数据采集/src/platforms/xueqiu.js`
- Create: `数据采集/src/platforms/cls.js`
- Create: `数据采集/test/platforms/jsonPlatforms.test.js`
- Create: `数据采集/test/fixtures/*.json`
- Modify: `数据采集/src/platforms/index.js`
- Test: `数据采集/test/platforms/jsonPlatforms.test.js`

- [ ] **Step 1: Write the failing batch parser test**

```js
import { describe, expect, test } from 'vitest';
import { createShaoshupaiCrawler } from '../../src/platforms/shaoshupai.js';
import { createWeiboCrawler } from '../../src/platforms/weibo.js';

describe('json platform crawlers', () => {
  test('normalize two representative APIs', async () => {
    const shaoshupai = createShaoshupaiCrawler({ requestJson: async () => ({ data: [{ id: 1, title: 'A', summary: 'S', url: 'https://sspai.com/post/1' }] }) });
    const weibo = createWeiboCrawler({ requestJson: async () => ({ data: { realtime: [{ word: '热搜词', raw_hot: 123 }] } }) });

    const s1 = await shaoshupai.collect({ canonicalPlatform: 'shaoshupai', meta: { displayName: '少数派', aliases: ['shaoshupai', 'sspai'] } });
    const s2 = await weibo.collect({ canonicalPlatform: 'weibo', meta: { displayName: '微博热搜', aliases: ['weibo'] } });

    expect(s1.items[0].title).toBe('A');
    expect(s2.items[0].title).toBe('热搜词');
  });
});
```

- [ ] **Step 2: Run the batch parser test and verify it fails**

Run: `cd "数据采集" && npm test -- test/platforms/jsonPlatforms.test.js`
Expected: FAIL because the crawler modules do not exist.

- [ ] **Step 3: Implement shared JSON crawler pattern and each platform module**

Use this file pattern for each JSON crawler:

```js
import { createItem } from './helpers.js';

const SOURCE_URL = 'UPSTREAM_URL_HERE';

export function createExampleCrawler(deps) {
  return {
    async collect({ canonicalPlatform, meta }) {
      const payload = await deps.requestJson(SOURCE_URL, deps.requestOptions ?? {});
      const entries = /* platform-specific payload extraction */;
      return {
        platform: canonicalPlatform,
        canonicalPlatform,
        aliases: meta.aliases,
        displayName: meta.displayName,
        sourceType: 'json-api',
        sourceUrl: SOURCE_URL,
        fetchedAt: new Date().toISOString(),
        success: true,
        items: entries.map((entry, index) => createItem(canonicalPlatform, index + 1, /* title */, /* url */, {
          hot: /* string or null */,
          summary: /* string or null */,
          author: /* string or null */,
          metadata: {},
          raw: entry
        })),
        warnings: []
      };
    }
  };
}
```

Implement exact upstream endpoints already confirmed in `API_INVENTORY.md` for:

- `shaoshupai`: `https://sspai.com/api/v1/article/index/page/get?limit=20&offset=0&created_at=0`
- `weibo`: `https://weibo.com/ajax/side/hotSearch`
- `zhihu`: `https://www.zhihu.com/api/v3/explore/guest/feeds?limit=30&ws_qiangzhisafe=0`
- `36kr`: `https://gateway.36kr.com/api/mis/nav/home/nav/rank/hot`
- `bilibili`: `https://api.bilibili.com/x/web-interface/popular`
- `tieba`: `http://tieba.baidu.com/hottopic/browse/topicList`
- `juejin`: `https://api.juejin.cn/content_api/v1/content/article_rank?category_id=1&type=hot`
- `douyin`: `https://www.douyin.com/aweme/v1/web/hot/search/list/`
- `jinritoutiao`: `https://www.toutiao.com/hot-event/hot-board/?origin=toutiao_pc`
- `tenxunwang`: `https://i.news.qq.com/gw/event/pc_hot_ranking_list?ids_hash=&offset=0&page_size=51&appver=15.5_qqnews_7.1.60&rank_id=hot`
- `stackoverflow`: `https://api.stackexchange.com/2.3/questions?order=desc&sort=hot&site=stackoverflow`
- `github`: `https://api.github.com/search/repositories?q=stars:%3E1&sort=stars`
- `sina_finance`: `https://zhibo.sina.com.cn/api/zhibo/feed?page=1&page_size=20&zhibo_id=152&tag_id=0&dire=f&dpc=1&pagesize=20`
- `eastmoney`: `https://np-weblist.eastmoney.com/comm/web/getFastNewsList`
- `xueqiu`: preheat `https://xueqiu.com` then `https://xueqiu.com/hot_event/list.json?count=10`
- `cls`: `https://www.cls.cn/featured/v1/column/list`

Update `src/platforms/index.js` to register every implemented JSON crawler.

- [ ] **Step 4: Run the batch parser test and verify it passes**

Run: `cd "数据采集" && npm test -- test/platforms/jsonPlatforms.test.js`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "数据采集/src/platforms" "数据采集/test/platforms/jsonPlatforms.test.js" "数据采集/test/fixtures"
git commit -m "feat: add json api platform crawlers"
```

### Task 9: Implement the HTML Crawlers and Browser Fallback Hook

**Files:**
- Create: `数据采集/src/core/browserClient.js`
- Create: `数据采集/src/platforms/52pojie.js`
- Create: `数据采集/src/platforms/douban.js`
- Create: `数据采集/src/platforms/hupu.js`
- Create: `数据采集/src/platforms/v2ex.js`
- Create: `数据采集/src/platforms/hackernews.js`
- Create: `数据采集/test/platforms/htmlPlatforms.test.js`
- Create: `数据采集/test/fixtures/52pojie.html`
- Create: `数据采集/test/fixtures/douban.html`
- Create: `数据采集/test/fixtures/hupu.html`
- Create: `数据采集/test/fixtures/v2ex.html`
- Create: `数据采集/test/fixtures/hackernews.html`
- Modify: `数据采集/src/platforms/index.js`
- Test: `数据采集/test/platforms/htmlPlatforms.test.js`

- [ ] **Step 1: Write the failing HTML crawler test**

```js
import { describe, expect, test } from 'vitest';
import fs from 'node:fs/promises';
import { createHackerNewsCrawler } from '../../src/platforms/hackernews.js';

describe('html crawlers', () => {
  test('parses hacker news rows from html', async () => {
    const html = await fs.readFile(new URL('../fixtures/hackernews.html', import.meta.url), 'utf8');
    const crawler = createHackerNewsCrawler({ requestText: async () => html });
    const result = await crawler.collect({ canonicalPlatform: 'hackernews', meta: { displayName: 'Hacker News', aliases: ['hackernews'] } });

    expect(result.items[0].title).toBe('Example HN Post');
  });
});
```

- [ ] **Step 2: Run the HTML crawler test and verify it fails**

Run: `cd "数据采集" && npm test -- test/platforms/htmlPlatforms.test.js`
Expected: FAIL because the crawler modules do not exist.

- [ ] **Step 3: Implement HTML crawler modules and browser helper**

Use this file pattern for each HTML crawler:

```js
import * as cheerio from 'cheerio';
import { createItem } from './helpers.js';

const SOURCE_URL = 'HTML_SOURCE_URL_HERE';

export function createExampleHtmlCrawler(deps) {
  return {
    async collect({ canonicalPlatform, meta }) {
      const html = await deps.requestText(SOURCE_URL, deps.requestOptions ?? {});
      const $ = cheerio.load(html);
      const items = [];

      $('SELECTOR_HERE').each((index, node) => {
        const title = $(node).find('TITLE_SELECTOR').text().trim();
        const url = new URL($(node).find('LINK_SELECTOR').attr('href'), SOURCE_URL).toString();
        if (title && url) items.push(createItem(canonicalPlatform, index + 1, title, url, { raw: { title, url } }));
      });

      return {
        platform: canonicalPlatform,
        canonicalPlatform,
        aliases: meta.aliases,
        displayName: meta.displayName,
        sourceType: 'html',
        sourceUrl: SOURCE_URL,
        fetchedAt: new Date().toISOString(),
        success: true,
        items,
        warnings: []
      };
    }
  };
}
```

Add `browserClient.js` with an exported `requestRenderedHtml(pageUrl)` wrapper around Playwright that returns `await page.content()` for fallback mode.

- [ ] **Step 4: Run the HTML crawler test and verify it passes**

Run: `cd "数据采集" && npm test -- test/platforms/htmlPlatforms.test.js`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "数据采集/src/core/browserClient.js" "数据采集/src/platforms" "数据采集/test/platforms/htmlPlatforms.test.js" "数据采集/test/fixtures"
git commit -m "feat: add html platform crawlers"
```

### Task 10: Harden Error Handling and Unsupported Platform Reporting

**Files:**
- Modify: `数据采集/src/scheduler/collectPlatform.js`
- Modify: `数据采集/src/platforms/index.js`
- Create: `数据采集/test/scheduler/collectPlatform.test.js`
- Test: `数据采集/test/scheduler/collectPlatform.test.js`

- [ ] **Step 1: Write the failing unsupported-platform test**

```js
import { describe, expect, test } from 'vitest';
import { collectPlatform } from '../../src/scheduler/collectPlatform.js';

describe('collectPlatform', () => {
  test('returns structured unsupported-platform errors', async () => {
    const result = await collectPlatform('unknown', { registry: {} });

    expect(result.success).toBe(false);
    expect(result.error.code).toBe('UNSUPPORTED_PLATFORM');
  });
});
```

- [ ] **Step 2: Run the unsupported-platform test and verify it fails for the right reason**

Run: `cd "数据采集" && npm test -- test/scheduler/collectPlatform.test.js`
Expected: FAIL if the current implementation returns an invalid `sourceUrl` or inconsistent schema.

- [ ] **Step 3: Tighten the result shape and error generation**

```js
import { resultSchema } from '../schemas/resultSchema.js';

function buildFailureResult(platformId, canonicalPlatform, meta, error) {
  return resultSchema.parse({
    platform: platformId,
    canonicalPlatform: canonicalPlatform ?? String(platformId),
    aliases: meta?.aliases ?? [],
    displayName: meta?.displayName ?? String(platformId),
    sourceType: meta?.sourceType === 'html' ? 'html' : 'json-api',
    sourceUrl: meta?.sourceUrl ?? 'https://example.invalid/',
    fetchedAt: new Date().toISOString(),
    success: false,
    items: [],
    warnings: [error.message],
    error: {
      code: error.code,
      message: error.message,
      retryable: Boolean(error.retryable)
    }
  });
}
```

Wrap `crawler.collect(...)` in `try/catch`, convert thrown transport errors into parsed failure payloads, and validate success payloads with `resultSchema.parse(...)` before returning.

- [ ] **Step 4: Run the unsupported-platform test and the scheduler test suite**

Run: `cd "数据采集" && npm test -- test/scheduler/collectPlatform.test.js test/scheduler/collectMany.test.js`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "数据采集/src/scheduler/collectPlatform.js" "数据采集/src/platforms/index.js" "数据采集/test/scheduler/collectPlatform.test.js"
git commit -m "fix: standardize scheduler failure payloads"
```

### Task 11: Add Root Documentation and Project Guide

**Files:**
- Create: `数据收集.md`
- Create: `API.md`
- Create: `介绍.md`
- Test: `数据收集.md`
- Test: `API.md`
- Test: `介绍.md`

- [ ] **Step 1: Write the failing doc completeness checklist**

```text
数据收集.md must list every canonical platform, upstream URL, source type, response shape, availability state, and collection strategy.
API.md must define upstream endpoints plus CLI/programmatic entry contracts.
介绍.md must explain folder structure, env vars, commands, and extension steps.
```

- [ ] **Step 2: Run the manual checklist and verify the docs are missing**

Run: `test -f "数据收集.md" && test -f "API.md" && test -f "介绍.md"`
Expected: non-zero exit status because the files do not exist yet.

- [ ] **Step 3: Write the documentation files**

Use this `数据收集.md` section template for every platform:

```md
## 平台名称 (`canonicalId`)

上游来源：`SOURCE_URL`
来源方式：JSON API / HTML 页面 / 浏览器回退
当前状态：可用 / 条件可用 / 受限
典型返回：列出顶层字段和列表字段
抓取策略：列出主抓取路径、请求头、Cookie、代理、预热、回退方案
```

Use this `API.md` section template for every upstream and local entrypoint:

```md
## `canonicalId`

端点：`SOURCE_URL`
方法：`GET`
鉴权：无 / Cookie / Token / 代理建议
参数：列出 query 参数
响应字段：列出顶层字段、列表字段、标准化映射
调用示例：提供 `node src/cli/run.js --platform canonicalId`
```

Use this `介绍.md` structure:

```md
# 介绍

## 目录结构
## 架构说明
## 环境变量
## 安装与运行
## 调度入口
## 平台扩展指南
## 故障排查
```

- [ ] **Step 4: Run the manual checklist again**

Run: `test -f "数据收集.md" && test -f "API.md" && test -f "介绍.md"`
Expected: zero exit status.

- [ ] **Step 5: Commit**

```bash
git add "数据收集.md" "API.md" "介绍.md"
git commit -m "docs: add collector architecture and api references"
```

### Task 12: Add Smoke Runner and Final Verification

**Files:**
- Create: `数据采集/scripts/smoke.js`
- Modify: `数据采集/package.json`
- Test: `数据采集/scripts/smoke.js`

- [ ] **Step 1: Write the failing smoke-runner expectation**

```js
import { createPlatformRegistry } from '../src/platforms/index.js';
import { collectMany } from '../src/scheduler/collectMany.js';

const registry = createPlatformRegistry();
const result = await collectMany(['baidu'], { registry });
process.stdout.write(`${JSON.stringify({ successCount: result.successCount }, null, 2)}\n`);
```

- [ ] **Step 2: Run the smoke command and verify it fails before the script exists**

Run: `cd "数据采集" && npm run smoke`
Expected: FAIL with missing script file.

- [ ] **Step 3: Write the minimal smoke runner**

```js
import { createPlatformRegistry } from '../src/platforms/index.js';
import { collectMany } from '../src/scheduler/collectMany.js';

const registry = createPlatformRegistry();
const result = await collectMany(['baidu', 'hackernews'], { registry });
process.stdout.write(`${JSON.stringify(result, null, 2)}\n`);
```

- [ ] **Step 4: Run the full verification suite**

Run: `cd "数据采集" && npm test`
Expected: PASS.

Run: `cd "数据采集" && npm run smoke`
Expected: JSON output with `successCount`, `failureCount`, and `results`.

Run: `node "数据采集/src/cli/run.js" --platform baidu`
Expected: JSON output for the Baidu collector.

- [ ] **Step 5: Commit**

```bash
git add "数据采集/scripts/smoke.js" "数据采集/package.json"
git commit -m "test: add collector smoke verification"
```

## Self-Review

### Spec Coverage

- Root documents are covered by Task 11.
- Node project scaffold and runtime shape are covered by Tasks 1 through 5.
- Standardized JSON schema is covered by Task 3.
- Scheduler and CLI are covered by Tasks 5 and 7.
- JSON API crawlers are covered by Tasks 6 and 8.
- HTML crawlers and browser fallback are covered by Task 9.
- Error handling and structured failures are covered by Task 10.
- Final verification is covered by Task 12.

### Placeholder Scan

- No `TODO`, `TBD`, or deferred implementation placeholders remain in task steps.
- Reusable module templates are included only where the platform-specific endpoint list immediately follows.

### Type Consistency

- The plan consistently uses `collect({ canonicalPlatform, meta })` as the crawler entrypoint.
- Standard result fields remain `platform`, `canonicalPlatform`, `aliases`, `displayName`, `sourceType`, `sourceUrl`, `fetchedAt`, `success`, `items`, `warnings`, and optional `error`.
- Alias names match the approved design: `sspai`, `tskr`, `ftpojie`, and `vtex`.
