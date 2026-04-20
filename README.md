# rehype-external-link-title

[**rehype**][rehype] plugin that fetches the page `<title>` of every external
link in your HTML and writes it to the link's `title` attribute (so users see
the destination's real name when they hover the link), with a pluggable
caching layer.

## Contents

- [What is this?](#what-is-this)
- [When should I use this?](#when-should-i-use-this)
- [Install](#install)
- [Use](#use)
- [API](#api)
  - [`unified().use(rehypeExternalLinkTitle[, options])`](#unifieduserehypeexternallinktitle-options)
  - [`Options`](#options)
  - [`Cache`](#cache)
  - [Built-in caches](#built-in-caches)
- [Examples](#examples)
- [Types](#types)
- [Compatibility](#compatibility)
- [Security](#security)
- [Contributing](#contributing)
- [Releasing](#releasing)
- [License](#license)

## What is this?

This is a [unified][] ([rehype][]) plugin. It walks the [hast][] tree, finds
external `<a>` elements (by default: anchors whose `href` starts with `http://`
or `https://`), fetches each unique URL, parses the `<title>` element from the
response, and stores it on the anchor as a `title` attribute.

To avoid hammering remote servers (and to keep your build times reasonable),
results are persisted to a cache. The cache is **pluggable**: a default
[lowdb][]-backed JSON file is provided out of the box, and you can swap in
your own backend (Redis, KV, in-memory, etc.) by implementing a tiny
two-method interface.

## When should I use this?

Use this plugin if you publish content with many external references — blog
posts, link round-ups, documentation — and you want hover tooltips to display
the actual page title rather than the raw URL.

You probably **shouldn't** use it if:

- Your build runs in a sandbox without outbound network access.
- You don't trust the remote pages and don't want to render their titles
  (consider [`rehype-sanitize`][rehype-sanitize] downstream regardless — see
  [Security](#security)).
- Build performance is more important than hover-over UX (the first build
  fetches every link; subsequent builds are cache hits).

## Install

This package is [ESM only][esm]. In Node.js (version 18+):

```sh
npm install rehype-external-link-title
```

```sh
pnpm add rehype-external-link-title
```

## Use

Say we have the following input HTML:

```html
<p>Read more on <a href="https://example.com">example.com</a>.</p>
```

…and a script `example.js`:

```js
import {unified} from 'unified'
import rehypeParse from 'rehype-parse'
import rehypeStringify from 'rehype-stringify'
import rehypeExternalLinkTitle from 'rehype-external-link-title'

const file = await unified()
  .use(rehypeParse, {fragment: true})
  .use(rehypeExternalLinkTitle)
  .use(rehypeStringify)
  .process('<p><a href="https://example.com">example.com</a></p>')

console.log(String(file))
```

…running `node example.js` yields (assuming the page's title is `Example Domain`):

```html
<p><a href="https://example.com" title="Example Domain" data-title-updated-at="2026-04-19T00:00:00.000Z">example.com</a></p>
```

## API

This package exports the named identifiers `lowdbCache`, `memoryCache`, and the
TypeScript types `Cache`, `CacheEntry`, `FetchOptions`, `LinkPredicate`, and
`Options`. The default export is `rehypeExternalLinkTitle`.

### `unified().use(rehypeExternalLinkTitle[, options])`

Adds page titles to external `<a>` elements as `title` attributes, with caching.

###### Parameters

- `options` ([`Options`](#options), optional) — configuration

###### Returns

Async transform.

### `Options`

Configuration (TypeScript type).

| Field              | Type                                  | Default                              | Description                                                                                                                                            |
| ------------------ | ------------------------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `cache`            | `string \| Cache`                     | built-in `lowdbCache()`              | `undefined`: lowdb at `./db.titles.json`. `string`: lowdb at the given path. `Cache`: your own implementation.                                          |
| `ttl`              | `number`                              | `Infinity`                           | TTL for **successful** entries, in ms.                                                                                                                 |
| `failureTtl`       | `number`                              | `86_400_000` (24 h)                  | TTL for **failed** entries (`title === null`), in ms. Use `0` to never cache failures, `Infinity` to cache forever.                                    |
| `test`             | `(href, node) => boolean`             | `http(s)://...`                      | Predicate deciding which `<a>` elements to process.                                                                                                    |
| `attribute`        | `string`                              | `'title'`                            | Attribute name written on the link element.                                                                                                            |
| `includeUpdatedAt` | `boolean`                             | `true`                               | Whether to also write `data-title-updated-at` (ISO timestamp).                                                                                         |
| `concurrency`      | `number`                              | `8`                                  | Maximum concurrent outbound fetches per transformer invocation.                                                                                        |
| `fetch`            | `FetchOptions`                        | see below                            | Options forwarded to the internal HTTP client (`timeout`, `userAgent`, `signal`).                                                                      |

### `Cache`

The plugin treats the cache as a dumb async key/value store. TTL/staleness
handling is performed by the plugin itself, so cache implementations stay
trivial:

```ts
export interface CacheEntry {
  title: string | null  // `null` = "we tried and got nothing"
  updatedAt: string     // ISO-8601
}

export interface Cache {
  get(url: string): Promise<CacheEntry | undefined> | CacheEntry | undefined
  set(url: string, entry: CacheEntry): Promise<void> | void
  delete?(url: string): Promise<void> | void  // optional
}
```

Both sync and async return values are supported, so a `Map`-backed cache or a
Redis-backed cache are equally easy to write.

### Built-in caches

```ts
import {lowdbCache, memoryCache} from 'rehype-external-link-title/cache'

const persistent = lowdbCache({path: '.cache/titles.json'})
const ephemeral  = memoryCache()
```

- **`lowdbCache(options?: {path?: string})`** — JSON file backed by [lowdb][].
  The file is opened lazily on first use (no top-level I/O).
- **`memoryCache()`** — `Map`-backed; useful for tests or short-lived processes.

## Examples

### Custom cache path

```ts
unified().use(rehypeExternalLinkTitle, {cache: '.cache/external-link-titles.json'})
```

### Refetch every entry older than a week

```ts
unified().use(rehypeExternalLinkTitle, {ttl: 7 * 24 * 60 * 60 * 1000})
```

### Bring your own cache (Redis-style)

```ts
import type {Cache, CacheEntry} from 'rehype-external-link-title'

const redisCache: Cache = {
  async get(url) {
    const raw = await redis.get(`title:${url}`)
    return raw ? (JSON.parse(raw) as CacheEntry) : undefined
  },
  async set(url, entry) {
    await redis.set(`title:${url}`, JSON.stringify(entry))
  },
  async delete(url) {
    await redis.del(`title:${url}`)
  },
}

unified().use(rehypeExternalLinkTitle, {cache: redisCache})
```

### Custom User-Agent

```ts
unified().use(rehypeExternalLinkTitle, {
  fetch: {userAgent: 'MyCoolBlog/1.0 (+https://example.com/about)'},
})
```

### Process only a subset of links

```ts
unified().use(rehypeExternalLinkTitle, {
  test: (href) => href.startsWith('https://en.wikipedia.org/'),
})
```

## Types

This package is fully typed with [TypeScript][]. It exports the additional
types `Options`, `Cache`, `CacheEntry`, `FetchOptions`, and `LinkPredicate`.

## Compatibility

Compatible with maintained versions of Node.js (>=18). Works with `unified`
version 11+.

## Security

This plugin sets the `title` attribute on `<a>` elements based on data fetched
from third-party servers. While `title` is generally not an XSS vector
(browsers do not interpret it as HTML), you should still pair this plugin with
[`rehype-sanitize`][rehype-sanitize] downstream, configured to allow the
`title` attribute on anchors:

```ts
import rehypeSanitize, {defaultSchema} from 'rehype-sanitize'

unified()
  .use(rehypeExternalLinkTitle)
  .use(rehypeSanitize, {
    ...defaultSchema,
    attributes: {
      ...defaultSchema.attributes,
      a: [...(defaultSchema.attributes?.a ?? []), 'title'],
    },
  })
```

The HTML returned by remote servers is sanitized internally with [DOMPurify][]
(stripped down to `<html>`/`<head>`/`<title>` only) before the title is
extracted, so malicious script tags in the source page are discarded before
parsing.

## Contributing

### Local setup

```sh
pnpm install
pnpm test           # run vitest
pnpm test:coverage  # run vitest with v8 coverage (writes ./coverage)
pnpm typecheck      # tsc --noEmit
pnpm build          # tsdown → ./dist
```

### Secret scanning (gitleaks pre-commit hook)

This repo ships a Docker-based [gitleaks][] pre-commit hook in [`.githooks/pre-commit`](./.githooks/pre-commit). It scans the **staged diff only** (so it's fast) and blocks the commit if any secret is detected.

#### Enable the hook (one-time, per clone)

```sh
git config core.hooksPath .githooks
```

> Hooks live inside the repo (under `.githooks/`) instead of `.git/hooks/` so they're versioned and shared. Each contributor must opt in once with the command above — git does not auto-trust in-repo hooks, by design.

#### Requirements

- [Docker][] must be available on `PATH`. The hook pulls/runs the pinned image `zricethezav/gitleaks:v8.30.1` once per commit (no local Go install needed).
- If Docker is missing, the hook prints a warning and lets the commit through, so it doesn't break contributors who haven't installed Docker yet — but CI will still reject committed secrets (see below).

#### Bypassing

For an intentional false-positive bypass on a single commit:

```sh
git commit --no-verify
```

#### Manual full-repo scan

```sh
docker run --rm -v "$(pwd):/repo" -w /repo zricethezav/gitleaks:v8.30.1 git --redact --verbose
```

Project-specific allowlists (e.g. test fixtures that look like keys but aren't) live in [`.gitleaks.toml`](./.gitleaks.toml).

### Continuous integration

[`.github/workflows/ci.yml`](./.github/workflows/ci.yml) runs on every push and pull request:

1. **Build & test** — `pnpm install --frozen-lockfile`, `pnpm typecheck`, `pnpm test:coverage`, `pnpm build`.
2. **SonarQube Cloud scan** — uploads `coverage/lcov.info` plus a static analysis pass. Requires the `SONAR_TOKEN` repository secret (generated at [SonarCloud → My Account → Security][sonarcloud-token]).
3. **Gitleaks scan** — full-history secret scan via [gitleaks/gitleaks-action][gitleaks-action]. No license needed for personal-account repos.

All actions are pinned to commit SHAs (with the human-readable tag in a comment) so the workflow is reproducible and auditable.

## Releasing

Releases are published to npm by [`.github/workflows/publish.yml`](./.github/workflows/publish.yml) on every published GitHub Release, using **npm trusted publishing (OIDC)** — no `NPM_TOKEN` secret is involved.

### One-time setup on npmjs.com

1. Publish version `0.1.0` manually once (so the package exists). Then on subsequent releases the trusted publisher takes over.
2. Visit `https://www.npmjs.com/package/rehype-external-link-title/access` → **Trusted Publisher** → add a new GitHub Actions publisher with:
   - Organization or user: `aripalo`
   - Repository: `rehype-external-link-title`
   - Workflow filename: `publish.yml`
   - Environment: _(leave blank, or set if you want manual approval gating)_
3. Delete any pre-existing `NPM_TOKEN` repository secret — it's no longer needed and is now an unused attack surface.

### Cutting a release

1. Bump the `version` field in `package.json` and commit.
2. Tag and push: `git tag v0.2.0 && git push --tags`
3. Create a GitHub Release for that tag (via the GitHub UI or `gh release create v0.2.0`).
4. The workflow runs typecheck + tests + build, then `npm publish --provenance --access public`. The OIDC token is exchanged with npm for short-lived publish credentials, and a [provenance attestation][provenance] is attached to the published version.

### Anatomy of `package.json` lifecycle scripts

| Script | When it runs | Purpose |
|---|---|---|
| `prepack`            | `npm pack`, `npm publish`, git installs | Builds `dist/` so the tarball is always complete |
| `prepublishOnly`     | `npm publish` only | Runs `pnpm typecheck && pnpm test` as a publish gate |

## License

[MIT][license] © [Ari Palo][author]

[unified]: https://github.com/unifiedjs/unified
[rehype]: https://github.com/rehypejs/rehype
[hast]: https://github.com/syntax-tree/hast
[rehype-sanitize]: https://github.com/rehypejs/rehype-sanitize
[lowdb]: https://github.com/typicode/lowdb
[esm]: https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c
[typescript]: https://www.typescriptlang.org
[dompurify]: https://github.com/cure53/DOMPurify
[gitleaks]: https://gitleaks.io/
[gitleaks-action]: https://github.com/gitleaks/gitleaks-action
[docker]: https://www.docker.com/
[sonarcloud-token]: https://sonarcloud.io/account/security
[provenance]: https://docs.npmjs.com/generating-provenance-statements
[license]: ./LICENSE
[author]: https://aripalo.technology
