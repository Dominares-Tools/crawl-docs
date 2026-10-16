# Changelog

## 1.2.1 — 2026-10-04

Fixes from review. Nothing is removed.

### Fixed
- **A crawl that reads no pages no longer wipes your knowledge base.** Before, one failed
  scheduled run (or one `webcrawl --embed-cache` run while the site was down) replaced
  `pages.json` and `chunks.jsonl` with empty ones, and `prune` then dropped every cached
  embedding. Now both the command and `webcrawl-schedule` leave the last good files alone
  and report the failure (exit 1), and `precompute_chunk_embeddings(prune=True)` never
  prunes against an empty corpus.
- `HybridResult.recovered` counted chunks the result cut off at `limit`. It now counts only
  the recovered hits actually returned.
- The HTTP API reported version 0.2.0 in `/docs` and `/openapi.json`. It now reports the
  package version.
- **Social schedules store their time zone.** `webcrawl-social schedule --times 09:00
  --tz America/Los_Angeles` saves the zone (`SocialStore.save_schedule(timezone_name=...)`);
  `run-due` and `status` use it unless `--tz` is given (then the stored zone beats
  `WEBCRAWL_TZ`). Before, fixed times fired in UTC unless every call passed a zone. An
  unknown zone name is rejected. Old `sources.json` files load with no zone (UTC).
- `webcrawl-social schedule --every` outside 15 to 1440 is rejected (exit 2) instead of
  being silently changed.
- A card no longer reads "Account: None" or "Source: None" when a fetcher leaves those out.
- README: `--search-mode` lists `hybrid`; the time zone is documented.

## 1.2.0 — 2026-10-04

Social sources. Nothing existing changed.

### Added
- **`webcrawl.social`** and the `webcrawl-social` command. Watch YouTube, X and Facebook
  accounts; each scan fetches recent posts, skips ones already seen, and writes one short
  source-grounded summary card per new post (extractive by default, or from any
  `summarizer` function). Cards are `chunks.jsonl`-shaped, so the hybrid search and the
  embedding cache work on them. The raw post text is kept beside each card.
- Schedules for scans: every N minutes, or at fixed times of day on chosen weekdays in a
  named time zone. `run_if_due` re-checks inside the scan lock, so two workers scan once.
- YouTube is read from the public uploads feed (no key for a channel ID). Video words are
  not read; a `transcriber` function can supply them. X needs a bearer token, Facebook a
  Page access token. Error messages are scrubbed of anything shaped like a token or key,
  and are never built from request URLs.
- The fetchers are tested against recorded response shapes through a mock transport, not
  against the live services.

## 1.1.0 — 2026-10-04

Hybrid retrieval and scheduled crawls. Nothing existing changed; the package version now
matches the release number (it read 0.2.0 before, so `webcrawl --version` was wrong).

### Added
- **Hybrid search** (`webcrawl.hybrid`). An embedding cache keyed by a hash of each chunk's
  text, built once at crawl time (`precompute_chunk_embeddings`: skips cached chunks, saves
  after every batch, optional pruning). `hybrid_search` embeds only the query, re-ranks the
  keyword hits by cosine similarity, and appends chunks keyword search missed when they are
  close in meaning (`semantic_recall_merge`). When the semantic layer cannot run (query
  embed failed or timed out, no cache, cache from a different embedder or vector size) it
  returns the keyword result with `mode="lexical"` and a `reason`, and logs a warning.
  Also `hybrid_rerank`, `cache_coverage`, `excerpt_around_match`, `EmbeddingCache`.
- **Scheduled crawls** (`webcrawl.schedule`, command `webcrawl-schedule`). A
  `<out_dir>/schedule.json` per crawl; `claim_due` checks due, advances `next_run_at` and
  takes a lease in one locked step so exactly one worker runs each due crawl; a lease that
  expires is recorded as `interrupted`; `run_due`, `run_forever`, `start_scheduler`,
  `public_status`.
- Command line: `--embed-cache PATH` and `--search-mode hybrid`.
- A small cross-process file lock (stdlib only) shared by both.

## 1.0.5 — 2026-10-04

Bonus pack only. The crawl code is unchanged.

- The bonus pack's `apple-notes-exporter` is now 1.1.1. It reads each note's HTML body as well as its
  plaintext, so a table in an Apple Notes note is kept as one `a | b` line per row instead of being
  reduced to a single placeholder character, and a line Notes splits into styled runs stays one line.
  Its README and changelog also state exactly what the export does and does not cover.

## 1.0.4 — 2026-10-04

Documentation only. The code is unchanged.

- The README now opens with a one-line description and a three-line example you can paste, and the
  licence and access notes moved to the end.

## 1.0.3 — 2026-10-02

Licence and contact details only. The code is unchanged.

- Licence text updated (the commercial licence, version 1.3).
- One public contact address, in the licence and the README.

## 1.0.2 — 2026-09-27

Licence only. The code is unchanged.

- Licence text updated (the commercial licence, version 1.2): it now covers every tool
  in the catalogue, not only crawl.

## 1.0.1 — 2026-09-13

### Fixed
- **Table cells no longer run together.** Adjacent `<td>`/`<th>` cells were joined
  with no separator, so a row read as `ToolWhat it does` and keyword search could
  not find words that were plainly visible in a table. Cells are now separated by
  a space, each row stays one paragraph, and `<caption>`/`<tfoot>` are paragraph
  breaks. Covered by `tests/test_extract.py::TestExtract::test_table_cells_do_not_run_together`.
- **Honest search wording.** The docs, CLI help and demos described the no-key
  local index as search "by meaning". Its `HashingEmbedder` matches shared words,
  not meaning; search by meaning needs `--openai-embeddings` (`OpenAIEmbedder`) or
  an embedder you plug in. The README, `DEMOS/`, examples and `--help` now say so,
  the browser demo's mode is labeled "Vector", and
  `webcrawl --search-mode semantic` prints that caveat when it runs without real
  embeddings.
- **The `DEMOS/` launchers work on Homebrew Python.** They used to `pip install`
  into the system interpreter, which Homebrew and other externally managed
  Pythons refuse, so the demos crashed with `No module named 'httpx'`. Each
  launcher now creates a private environment in `DEMOS/.venv` on first run
  (shared setup in `DEMOS/_venv.sh`) and runs from it.
- **`DEMOS/run_tests.command` runs the real suite.** It ran `unittest` with only
  `httpx` installed, which errored on the pytest-based tests. It now installs
  pytest and every optional extra the tests exercise and runs pytest, with
  nothing skipped.
- **The API launchers print a `curl` example that works.** `start_api.command` and
  `make api` started the server with no key, so the printed example got `401`.
  They now use `WEBCRAWL_API_KEY` if set, otherwise a random key for the session,
  print it, and include the `X-API-Key` header in the example. Both run
  `webcrawl.service:app`.
- **Installing webcrawl no longer adds a top-level `service` module** to
  site-packages. The root `service.py` shim still serves `uvicorn service:app`
  from a checkout; installed code uses `webcrawl.service:app` or `webcrawl-serve`.
- **`pip install -e ".[dev]"` now runs the full test suite.** The `dev` extra
  lacked `fastapi`, `uvicorn`, `pdfminer.six`, `openai` and `psycopg`, so the API
  and PDF tests were skipped. It now includes them.

## 1.0.0 — 2026-09-13

First stable release.

### Fixed
- **Link discovery no longer follows a page's own resources.** `extract_links`
  skipped nothing, so every crawl fetched each `<link rel="stylesheet">`, icon,
  font preload and manifest, then reported each one as a
  `skipped content-type` error. Those `<link>` rels are now ignored; page links
  such as `rel="next"` and `rel="alternate"` are still followed. Covered by
  `tests/test_extract.py::test_links_skip_subresources_but_keep_page_links`.
- **The host blocklist can no longer be dodged with a port or `user@` prefix.**
  Blocklist checks compared `host_key`, which keeps both, so
  `https://x.com:443/` or `https://anyone@x.com/` got past a blocklisted
  `x.com` in link discovery, redirects and the incremental re-crawl seed list.
  They now compare the bare hostname (`extract.blocklist_host`).
- **A redirect to a blocklisted or off-scope host is reported as that even when
  the target answers 4xx/5xx.** The status check used to run first, so it showed
  up as `HTTP 403`. The redirect test now uses the local test server instead of
  a live twitter.com redirect (which now lands on x.com).
- **README no longer lists a `--max-depth` CLI flag**, which never existed; link
  depth is set from Python with `CrawlConfig(max_depth=...)`.

### Added
- **Browser demo** (`DEMOS/html_demo.py` + `DEMOS/run_html_demo.command`): a
  zero-setup, offline-first local web page. Double-click the launcher — it
  crawls the built-in fake site, writes the portable KB files, builds a local
  vector index (`HashingEmbedder`, which matches shared words), and opens a
  browser to a self-contained page (inline CSS/JS, no external assets) showing
  the crawled pages + extracted text with live keyword and vector-index search.
  An optional box crawls a caller-supplied URL,
  SSRF-screened exactly like the FastAPI service (`assert_public_url` +
  `block_private_ips`, robots and same-host scope forced on). Stdlib
  `http.server` only — no FastAPI, no API key. Covered by `tests/test_html_demo.py`.

## 0.2.0

Correctness-first hardening + feature-depth release. The engine, KB pipeline,
CLI, and FastAPI service were each pinned by new regression tests, the service
was hardened for untrusted callers, and the optional feature set was rounded
out. Quality gates (ruff/black/isort/mypy/pytest-cov) now gate CI. The suite is
203 tests across Python 3.10–3.13. Items below are grouped by area; everything
ships in one release.

### Breaking changes
- `Page` gains a `truncated: bool` field (default `False`) and `Page.to_dict()`
  now emits a `"truncated"` key. Consumers that assert an exact key set on the
  page dict must account for the new key. A page whose body was clipped at
  `max_bytes` now reports `truncated=true` and is never collapsed to
  `status="cached"` on an incremental re-crawl.
- **Chunk id width**: `page_to_chunks` now uses 16-hex-char hash components
  (`f"{sha1(url)[:16]}-{content_hash[:16] or 'nohash'}-{i}"`), widened from 8,
  to lower birthday-bound collision risk. Any store keyed on the old 8-char
  ids must be rebuilt; ids from 0.1.0 will not match.
- **CLI exit code & stdout:** `webcrawl` now exits non-zero when a crawl yields
  zero pages (`ok:false`), and stdout carries ONLY the JSON summary — all
  progress and `--search` output moved to stderr. Scripts that parsed mixed
  stdout or relied on exit 0 must adjust.
- **Service: `CrawlRequest` dropped `respect_robots` and `same_host_only`.** The
  service now FORCES `respect_robots=True`, `same_host_only=True`, and
  `block_private_ips=True` on every crawl; clients can no longer relax robots or
  scope. Retained request fields: `url`, `extra_seeds`, `max_pages`,
  `per_host_delay_seconds`, `include_chunks`.
- **Service: API-key auth is ON by default.** Protected routes (`POST /crawl`,
  `GET /jobs`, `GET/DELETE /jobs/{id}`) require header `X-API-Key` matching env
  `WEBCRAWL_API_KEY`. With no key set, protected routes return `401` unless
  `WEBCRAWL_ALLOW_NO_AUTH=1` is set (explicit local-dev bypass). `GET /healthz`
  stays unauthenticated.
- **Service: per-client rate limit.** `POST /crawl` is bounded to
  `RATE_LIMIT_PER_MINUTE` (default 30) requests per client identity (API key, else
  client IP); over-limit returns `429`.
- **Service: the app moved to `webcrawl.service`.** Import/run as
  `webcrawl.service:app` (or the new `webcrawl-serve` console script); the root
  `service.py` remains a thin shim so `uvicorn service:app` still works.
- **Internal crawl pipeline (Phase 5):** `fetch_one` now returns a 5-tuple
  `(url, depth, page, links, error)` and the work queue holds `(url, depth)`
  tuples instead of bare URL strings. These are private engine internals; the
  public `crawl()` / `CrawlConfig` / `Page` / `CrawlResult` surface is unchanged
  except for the additive new `CrawlConfig` fields below.
- **PDF responses are no longer skipped (Phase 5):** an `application/pdf`
  response is now routed through the PDF extractor instead of being rejected on
  content-type. A PDF with no extractable text yields a
  `"pdf had no extractable text"` error rather than a content-type skip.

### Added (Phase 5 — refactor + feature depth)
- **Max-depth crawling**: `CrawlConfig.max_depth: int | None = None` caps hops
  from a seed (seeds are depth 0; `None` = unlimited). The queue now carries depth.
- **Sitemap-index + gzip**: `discover_sitemap` follows `<sitemapindex>` documents
  (bounded recursion) and transparently decompresses gzipped (`*.xml.gz`) sitemaps.
- **Crawl-delay**: the engine honors robots `Crawl-delay`; the effective per-host
  delay is `max(per_host_delay_seconds, robots_crawl_delay)`.
- **Charset sniffing**: when a response's Content-Type header carries no charset,
  the engine reads `<meta charset>` / `<meta http-equiv>` before decoding (no more
  latin-1 mojibake on header-less pages).
- **Truncation reconciliation**: the engine byte cap (`CrawlConfig.max_bytes`) is the
  single governing truncation cap — the body is decoded once within the byte budget,
  and `extract.MAX_HTML_CHARS` is a documented redundant backstop (kept `==` the
  default `max_bytes`) so multibyte content is never truncated twice at inconsistent
  byte/char boundaries.
- **PDF extraction** (`[pdf]` extra, `pdfminer.six`): `webcrawl.pdf.extract_pdf_text`;
  the engine routes `application/pdf` responses through it into the same pipeline.
- **`SqliteVectorStore`** (stdlib `sqlite3`): same interface as
  `LocalJsonVectorStore`, but an incremental DELETE+INSERT per push (no O(n²)
  whole-file rewrite).
- **Real vector-DB connectors**: `PineconeConnector` (httpx REST, no new dep) and
  `PgVectorConnector` (`[pgvector]` extra, lazy `psycopg`), both implementing the
  `Connector` protocol.
- **Render hardening**: `CrawlConfig.render_timeout_seconds` (wall-clock ceiling
  per render) and `CrawlConfig.max_rendered_html_bytes` (rendered-HTML size cap
  before extraction); the `render_js` SSRF trust boundary is documented in-code.
- New extras `[pdf]` and `[pgvector]`; `[all]` updated to the union.

### Changed (Phase 5)
- `fetch_one` decomposed into small pure helpers (`_should_retry_status`,
  `_redirect_rejection`, `_read_capped_body`, plus render/charset/sitemap
  helpers); the unreachable `"unknown fetch state"` return was removed.
- `PlaywrightRenderer.close()` is now lock-guarded and idempotent (safe to call
  concurrently with use); the `close()` contract is documented.

### CLI (Phase 3)
- New flags: `--version`, `--search-mode {keyword,semantic}`, `--openai-api-key`,
  `--embed-model`.
- `--search-mode semantic` now actually queries the `--vector-store` embeddings
  (previously `--search` only did keyword ranking even with a vector store).
- `--openai-embeddings` preflights the `openai` package + key and fails fast with
  a clear message instead of a late stack trace.

### Service hardening (Phase 4)
- `_seed_is_public` delegates to `webcrawl.netfilter` (one SSRF screen), and the
  engine re-screens post-redirect IPs via forced `block_private_ips=True`.
- Job records gain `created_at`; terminal jobs are TTL-evicted
  (`JOB_TTL_SECONDS=3600`); retained result payloads are byte-capped
  (`MAX_JOB_RESULT_BYTES`, over-cap trimmed with `truncated_result`).
- New routes `GET /jobs` (list) and `DELETE /jobs/{id}`; `POST /crawl` returns 202.
- `webcrawl-serve` console script launches uvicorn (host/port/reload via
  `WEBCRAWL_HOST`/`WEBCRAWL_PORT`/`WEBCRAWL_RELOAD`).

### Pipeline correctness (Phase 2)
- **Manifest integrity:** `write_portable` now records sha256 `hashes` of
  `pages.json` and `chunks.jsonl` in the manifest, written last as the bundle
  commit point.
- `load_chunks_jsonl` skips + logs a corrupt line instead of discarding the
  whole file.
- **Phrase search:** lookaround phrase regex (so `c++` / trailing-`?` match) and
  no double-counting of a matched phrase's own words.
- **Semantic search:** zero-norm (empty) embeddings are skipped so empty chunks
  can't crowd top-k; `LocalJsonVectorStore.search` re-infers dim from the records
  it scores rather than a stale cache.

### Engine correctness (Phase 1)
- **SSRF:** new `CrawlConfig.block_private_ips` (default `False`) re-resolves and
  rejects any post-redirect host that maps to a private/loopback/link-local/
  reserved/multicast/unspecified IP. Shared screen lives in `webcrawl/netfilter.py`
  (`host_is_public`, `assert_public_url`).
- **Aux-fetch SSRF / robots spoofing:** `robots.txt` and `sitemap.xml` fetches now
  re-validate their post-redirect target against scope (and IP when blocking); an
  off-scope robots redirect is treated as policy-unknown → conservative deny, so a
  malicious host can't 302 robots.txt to a spoofed allow-all file.
- **Duplicate canonical page:** two aliases that redirect to one canonical URL now
  yield exactly one `Page` (post-redirect key deduped before append).
- **`follow_links=False` on incremental:** a seeds-only resume no longer re-enqueues
  previously-known URLs; it fetches only the seeds.
- **Sitemap `<loc>`:** entity-escaped (`&amp;`) and CDATA-wrapped URLs are now decoded
  and enqueued correctly.
- **Robots cache key:** keyed by `host_key` (lowercased, `www.`-stripped), so
  `www.`/case variants of one host share a single robots fetch.
- **HTTP 429:** treated as a transient rate-limit (retry with backoff honoring
  `Retry-After`), not a permanent 4xx.

## 0.1.0

Initial standalone extraction of the knowledge-sync website crawler, plus a
security/correctness hardening pass driven by an adversarial code review.

### Core
- BFS crawler (`crawl`) with parallel fetching, incremental re-crawl
  (content-hash + `If-Modified-Since`), same-host scoping, social/CMS filtering.
- HTML → clean text + link discovery; paragraph-boundary-aware chunking.
- Portable output (`pages.json` + `chunks.jsonl` + `manifest.json`) any KB can import.
- Optional keyword search, embeddings (provider-agnostic), and a local JSON vector store.
- Optional FastAPI service and optional Playwright JS-rendering.

### Hardening (27 issues found by adversarial review, all fixed + regression-tested)
**Security**
- Redirects are re-validated against scope, host blocklist, and robots.txt, so a
  redirect can't escape scope or reach internal hosts (SSRF/scope-escape).
- `robots.txt` returning 5xx / unreachable now means *disallow* (conservative
  convention), not allow-all.
- HTML parsing moved to stdlib `html.parser`, removing an O(n²) ReDoS on
  pathological `<` runs (1 MB now parses in ~0.5 s instead of minutes).
- Responses are streamed with a Content-Type allowlist and a `max_bytes` cap, so
  binary/huge resources are never fully downloaded.
- The FastAPI service validates seeds (http(s) only; blocks private/loopback/
  link-local IPs), caps concurrent crawls (429), clamps page count, and evicts old jobs.

**Correctness / data integrity**
- `crawl_url_key` now includes the query string and port, so paginated/faceted
  pages (`?page=2`) are no longer silently deduped away.
- Chunk ids incorporate the page URL, so distinct pages with identical content no
  longer overwrite each other in an id-keyed store.
- Vector-store `push` evicts a page's prior chunks on re-crawl (no stale orphans),
  re-reads under a lock (no lost updates), and rejects embedding-dimension mismatches.
- `split_chunks` no longer drops sub-`min_chars` trailing content, and a short
  whole page still yields one chunk.
- Paragraph-based chunking actually fires now (extraction preserves blank lines).
- Commented-out `<title>` and unencoded `>` in attributes no longer corrupt text.
- Keyword search matches on word boundaries (no "art" ⊂ "party" false positives).
- `WebhookConnector` retries transient failures and reports partial-delivery state.

**Politeness / ergonomics**
- The crawl issues no more real requests than `max_pages` (dispatch budget fix).
- `max_pages=0` crawls nothing; `extract_links` accepts `www.`-prefixed allowed hosts.
- JS-render fallback decides on extracted-text length (not HTML byte size).
- The Playwright renderer runs all browser ops on one owning thread (clean shutdown).
- CLI reports vectors pushed this run vs. cumulative store total.

### Second hardening round (regressions/incompletions from the fixes above, all fixed)
A second adversarial review of the fixes caught 8 issues the new code introduced:
- **Body loss regression**: `<head>` was treated as a skip region, but `</head>`
  is optional in HTML5 — pages omitting it lost all body text. Removed `head` from
  the skip set.
- **Robots bypass on same-host redirect**: the redirect robots re-check was gated
  behind a host change; robots is path-sensitive, so it now runs on ANY redirect.
- **robots.txt / sitemap.xml fetches** are now streamed + byte-capped (they had
  escaped the body-size guard).
- **Service semaphore leak**: a failed thread start now releases the in-flight permit
  (no permanent 429 wedge).
- **Service job store**: eviction now drops only terminal jobs (never a running one)
  and won't resurrect an evicted record over the cap.
- **Webhook retries**: only transport errors / 5xx are retried; 4xx fails fast.
- **Vector-store dim check**: validated against the freshly re-read on-disk dimension.
- **Renderer reuse-after-close**: now raises instead of hanging; waits are bounded.

### Third hardening round (2 regressions from round-2 fixes, all fixed)
A third focused review of the round-2 diffs caught 2 more:
- **Renderer false timeouts**: the round-2 bounded wait measured wall-clock from
  enqueue, so renders queued behind others on the single serializer thread timed
  out and silently dropped SPA content. Replaced with a liveness check (waits
  while the owning thread is alive; raises only if it has died).
- **Service job leak**: on thread-start failure the orphaned `queued` job could
  never be evicted (terminal-only eviction), growing JOBS unbounded. The failure
  path now reaps the orphan record.

Review converged 27 → 8 → 2 findings across three adversarial rounds; the final
state has 56 regression tests covering every fix.
