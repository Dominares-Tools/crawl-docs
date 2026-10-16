# crawl — Turn a website into clean text you can search

**Paid tool: $49 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/crawl**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

Follows a site's links, pulls the readable text out of every page and saves passages any knowledge base can import. It can also re-crawl on a schedule and turn recent YouTube, X and Facebook posts into short cards.

## Get it

Buy crawl at https://dominares.org/tools/crawl: one payment of $49, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python library, the webcrawl command and the HTTP API.

Requirements: The base install needs only httpx. Optional extras add OpenAI embeddings, the HTTP API, JavaScript-rendered pages, PDFs and pgvector.

This page documents version 1.2.1. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### What it does

- **Crawl** a site (or several seeds): robots.txt, same-host scoping, rate limits, sitemap discovery, PDF text, optional JavaScript rendering, and a private-address (SSRF) guard.
- **Extract** clean text, **chunk** it into passages, and **detect what changed** so a re-crawl skips unchanged pages (`--resume`).
- **Hand it to any knowledge base**: portable files, a local JSON or SQLite vector index, a webhook, pgvector or Pinecone.
- **Search** the result: keyword ranking, vector similarity, or **hybrid** (new in 1.1): keyword search re-ranked and extended by a chunk-keyed embedding cache that is built once per chunk, not once per query. See [Hybrid search](#hybrid-search).
- **Pull social posts** (new in 1.2): recent YouTube uploads (no key needed), X posts and Facebook Page posts become short, source-grounded summary cards, each post once. See [Social sources](#social-sources).
- **Run on a schedule** (new in 1.1): daily, weekly or every N minutes, with a lease so exactly one worker runs each due crawl even with several workers. See [Scheduled crawls](#scheduled-crawls).

Base dependency: `httpx` only. Everything else is the standard library or an optional extra.

**Limits.** It reads pages; it does not log in or fill forms. A crawl that reads no pages (site down, DNS failure) exits 1 and leaves the files from the last good run, and the embedding cache, untouched. An unreachable or erroring `robots.txt` is treated as "disallow" (the conservative convention), so an outage can show up as "blocked by robots.txt" in the error list. JavaScript rendering needs the `js` extra and a Playwright browser. Search "by meaning" needs a real embedding model (`--openai-embeddings`, or your own `Embedder`): the built-in `HashingEmbedder` matches shared words, not meaning, and the README and the command line say so wherever it is used. The local vector index is for small and medium sites, not a replacement for a vector database. Scheduling uses a file lock, so all workers must see the same file (one machine, or a filesystem with working `flock`). The social fetchers are tested against recorded response shapes, not against the live services, which change their APIs and terms; use your own credentials and follow each platform's terms.

### Install

*The install commands are in the README of the private repository you are invited to when you buy. The notes on them are kept here.*


### Use it — CLI

```bash
# Crawl → writes pages.json + chunks.jsonl + manifest.json into ./kb
python -m webcrawl https://example.com --out ./kb --max-pages 100 --per-host-delay 0.3

# Incremental re-crawl (skips pages that haven't changed since last run)
python -m webcrawl https://example.com --out ./kb --resume

# Also build a local vector index (no API key; the built-in HashingEmbedder
# matches shared words, not meaning)
python -m webcrawl https://example.com --out ./kb --vector-store ./kb/vectors.json

# Crawl + quick keyword search
python -m webcrawl https://example.com --out ./kb --search "contact hours"

# Crawl + build a vector index with OpenAI embeddings, then search it by meaning
# (needs the [embeddings] extra and an OpenAI API key)
python -m webcrawl https://example.com --out ./kb --vector-store ./kb/vectors.json \
    --openai-embeddings --search "how do I get in touch" --search-mode semantic
```

Useful flags: `--no-robots`, `--no-follow`, `--sitemap`, `--render-js`,
`--openai-embeddings`, `--concurrency`, `--timeout`, `--user-agent`,
`--openai-api-key`, `--embed-model`, `--version`. To limit link depth, use
`CrawlConfig(max_depth=...)` from Python.

**Search modes:** `--search "<query>"` runs a keyword lookup by default;
add `--search-mode semantic` (with `--vector-store <path>`) to rank by vector
similarity instead. That is search by meaning only with a real embedding model:
`--openai-embeddings`, or any `Embedder` you plug in from Python. Without it the
index uses the built-in `HashingEmbedder`, which only matches shared words.
`--search-mode {keyword,semantic,hybrid}` defaults to `keyword` (`hybrid` is described below).

**Scripting contract:** stdout carries ONLY the JSON run summary; all
human-readable and search-result text goes to stderr. The process exits
non-zero (and the summary's `ok` is `false`) when the crawl yields 0 pages,
so `python -m webcrawl ... || echo "crawl failed"` works in scripts.

### Use it — library

```python
from webcrawl import crawl, CrawlConfig, write_portable

result = crawl(["https://example.com"], CrawlConfig(max_pages=50, per_host_delay_seconds=0.3))
write_portable(result, "./kb")        # portable files any KB can import

# Vector index in a local file (no external service). HashingEmbedder needs no
# API key but only matches shared words; for search by meaning pass
# OpenAIEmbedder() (the [embeddings] extra) or any object with
# embed(list[str]) -> list[list[float]].
from webcrawl import LocalJsonVectorStore, HashingEmbedder, iter_chunks
store = LocalJsonVectorStore("./kb/vectors.json", HashingEmbedder())
store.push(iter_chunks(result))
for score, rec in store.search("how do I get in touch", k=3):
    print(score, rec["url"], rec["text"][:80])
```

### Hybrid search

Keyword search is exact but blind to synonyms ("donate" never finds a page that says "contribution"). Embedding search finds it but costs a call per chunk. Hybrid search embeds each chunk **once**, at build time, and at query time embeds only the query:

```bash
# build: crawl, then embed every chunk that has no vector yet (re-crawls embed only what changed)
python -m webcrawl https://example.com --out ./kb --embed-cache ./kb/embeddings.json --openai-embeddings
# search: keyword hits re-ranked by meaning, plus close chunks the keywords missed
python -m webcrawl https://example.com --out ./kb --embed-cache ./kb/embeddings.json \
    --openai-embeddings --search "how do I donate" --search-mode hybrid
```

```python
from webcrawl import OpenAIEmbedder, load_chunks_jsonl, precompute_chunk_embeddings, hybrid_search

chunks = load_chunks_jsonl("kb/chunks.jsonl")
emb = OpenAIEmbedder()
precompute_chunk_embeddings(chunks, "kb/embeddings.json", emb, prune=True)   # at build time
res = hybrid_search("how do I donate", chunks, "kb/embeddings.json", emb)
print(res.mode, res.reason, res.coverage)          # "hybrid" / None / how many chunks have vectors
for score, chunk in res.hits:
    print(round(score, 3), chunk["url"])
```

How it behaves, and what holds:

- **Keyword hits stay on top.** The keyword hits are re-ranked by `0.6 * cosine + 0.4 * keyword rank`; chunks keyword search missed entirely are appended below them if their cosine is at least 0.35 (up to 5). Recovered chunks fill only the places the keyword hits leave free under `limit`.
- **It says when it is not working.** If the query embedding fails or times out (5 s), the cache is missing or empty, covers none of your chunks, or was built by a different embedder or vector size, you get the keyword result with `mode="lexical"` and a `reason`, and a WARNING is logged. A broken semantic layer never looks like a working one. `cache_coverage` reports how many chunks have a vector: a chunk added after the last build has none and ranks by keyword only until the next build.
- **The cache is one JSON file** (`{chunk_key: vector}` plus a header naming the embedder and vector size). It saves after every batch (a crash on a big corpus keeps its progress), it merges under a file lock so two builders do not lose each other's vectors, and `prune=True` drops vectors of chunks that no longer exist (never when the corpus is empty, so a failed crawl cannot wipe the cache).
- `excerpt_around_match(text, query, max_chars)` returns the window of a long chunk around where the query matches, instead of its start.
- Pure steps are exported too: `semantic_recall_merge`, `hybrid_rerank`, `chunk_key`.

Tested with a small concept-based embedder (so synonym recovery is proven offline) and with `HashingEmbedder` (plumbing works; it does not understand synonyms). Not exercised against the live OpenAI API in the test suite.

### Scheduled crawls

A schedule is one small file, `<out_dir>/schedule.json`: the seed URLs, how often, and the state of the last run. Any number of workers can call `run_due`; each due crawl is claimed by exactly one.

```bash
webcrawl-schedule set ./kb https://example.com --every daily --max-pages 100
webcrawl-schedule run-due ./kb          # one pass: run it from cron every few minutes
webcrawl-schedule status ./kb           # last/next run, last error, last 10 runs (JSON)
webcrawl-schedule loop ./kb --tick 300  # or stay running instead of using cron
```

```python
from webcrawl.schedule import set_schedule, run_due, start_scheduler

set_schedule("./kb", ["https://example.com"], "daily")           # off, hourly, daily, weekly, 90m, 6h, 2d
run_due(["./kb"], on_event=lambda e: print(e))                   # e["type"] is crawl_ok or crawl_failed
handle = start_scheduler(["./kb"], tick_seconds=300); handle.stop()
```

- **One claim per slot.** Checking that a crawl is due, advancing `next_run_at` and taking a lease happen in one locked step. Tested with six processes racing one due slot: exactly one wins.
- **Crash-tolerant.** A worker killed mid-crawl leaves a lease that expires; the next claim records that run as `interrupted` and the crawl runs again at its next slot.
- **Incremental.** A due run crawls the seeds with `previous_pages` from `pages.json`, so unchanged pages are skipped, and rewrites `pages.json`, `chunks.jsonl` and `manifest.json` in place. Pass your own `crawl_fn(out_dir, schedule_doc) -> {"ok", "pages_count", "error"}` to `run_due` to do something else (for example also refresh an embedding cache).
- **Limits.** The lease (`--lease-seconds`, default 2 hours) is not renewed during a crawl, so set it above your slowest crawl or a second worker can claim the same crawl late. A slot missed while every worker was down runs once when one returns; missed slots are not replayed. The minimum interval is 60 seconds. Failure alerts are the `on_event` callback (the command prints them to stderr and `run-due` exits 1).

### Social sources

Watch accounts; every scan fetches their newest posts, skips the ones already seen, and writes one short summary card per new post.

```bash
webcrawl-social add youtube https://www.youtube.com/channel/UCxxxxxxxxxxxxxxxxxxxxxx   # or @handle
webcrawl-social add facebook SomePageName          # needs FACEBOOK_PAGE_ACCESS_TOKEN
webcrawl-social add x SomeHandle                   # needs X_BEARER_TOKEN
webcrawl-social scan                               # cards go to ./social/cards.jsonl
webcrawl-social schedule --every 60                # or: --times 09:00 17:30 --tz America/Los_Angeles  (and --days 0 1 2 3 4)
webcrawl-social run-due                            # scan only if the schedule says one is due; cron it
webcrawl-social status
```

```python
from webcrawl.social import SocialStore, FileCardSink, scan_sources, OpenAISummarizer

store = SocialStore("./social")
store.add_source(platform="youtube", account="@some_channel")
report = scan_sources(
    store,
    sink=FileCardSink("./social/cards.jsonl"),   # or any function: card dict -> None | {"id": ...}
    summarizer=OpenAISummarizer(),               # optional; any function: prompt -> str
    transcriber=None,                            # optional; any function: video url -> its spoken words
)
print(report["added"], report["skipped_seen"], report["errors"])
```

- **What a scan does.** For each enabled source: fetch recent posts, skip any `external_id` already in the dedupe index (capped at the newest 5,000), summarize each new post, save the card through your `sink`, keep the post's raw text in `transcripts/` (capped at 100,000 characters), record it as seen. A post is marked seen only after its card was saved, so a failing sink means a retry, not a lost post. State is written after every source.
- **The cards** are `chunks.jsonl`-shaped lines (`id`, `url`, `title`, `text`, plus `tags`, `platform`, `published_at`), so `load_chunks_jsonl`, `precompute_chunk_embeddings` and `hybrid_search` work on them unchanged. Card ids are stable per post, so writing the same post twice writes it once.
- **Summaries.** Without a `summarizer` (or if it fails or returns nothing) the card carries an extractive summary: the post's own first distinct sentences, marked `extractive`. With one, it is marked `ai`. The prompt contains only the post's text and tells the model to use only that.
- **Failures are per source.** An expired token or a missing account is reported in `errors` (and `partial` is true if other sources still added cards); the other sources carry on. Error text is scrubbed of anything shaped like `access_token=...`, `key=...` or `Bearer ...`, and the fetchers never build an error from a request URL, because Facebook and YouTube authenticate in the query string.
- **Schedule.** Every N minutes (15 to 1440), or at fixed `HH:MM` times on chosen weekdays in the time zone you give `schedule --tz` (stored with the schedule; with no zone the times are UTC, and `--tz` or `WEBCRAWL_TZ` on `run-due` or `status` overrides the stored one; daylight saving handled; a slot missed while nothing ran fires once). `--every` outside 15 to 1440 is rejected. `run_if_due` re-checks inside the scan lock, so two workers calling it together scan once.
- **Credentials** come from the environment (`YOUTUBE_API_KEY`, `X_BEARER_TOKEN`, `FACEBOOK_PAGE_ACCESS_TOKEN`) or a `Credentials` object, are never stored, and `credential_status` reports presence only.

Limits, stated plainly:

- **YouTube** is read from the channel's public uploads feed: title, description, date and link of the newest videos (the feed holds about 15). It does **not** read what is said in a video; YouTube no longer serves captions to servers. Pass a `transcriber` (a function from video URL to text) if you have a source for the words, and it is called once per new video; otherwise the card is built from the title and description and says so. A channel ID works with no key; an `@handle` is resolved through the YouTube Data API if you set `YOUTUBE_API_KEY`, else from the channel page.
- **X** needs an API plan that allows reading an account's posts (a bearer token); X has paywalled this before and a depleted plan is reported as such. **Facebook** needs a Page access token that can read the page.
- Network fetchers are tested with recorded response shapes and a mock transport, **not against the live services**. The parsing, dedupe, scheduling, credential handling and error scrubbing are what the tests prove.

### Use it — HTTP API

The FastAPI app lives at `webcrawl.service:app`. After `pip install -e ".[api]"`
you can also launch it via the `webcrawl-serve` console script. (A `service.py`
shim in the repository root keeps `uvicorn service:app` working when run from
this folder; it is not installed as a package module.)

```bash
export WEBCRAWL_API_KEY="choose-a-secret"   # auth is ON by default
webcrawl-serve                              # or: uvicorn webcrawl.service:app --reload

# All crawl routes require the key; /healthz does not.
# POST /crawl   {"url":"...","max_pages":50}  -> {"job_id"}     (header: X-API-Key)
# GET  /jobs/{id}                              -> job status + result
# GET  /jobs                                   -> {"jobs":[{"job_id","status"}...]}
# DELETE /jobs/{id}                            -> 204 (or 404 if unknown)
```

The server forces `respect_robots` and `same_host_only` on, re-screens every
post-redirect host against private/loopback/link-local/metadata IPs
(SSRF defense), and evicts terminal jobs after `JOB_TTL_SECONDS`. For local
dev without a key, set `WEBCRAWL_ALLOW_NO_AUTH=1` (protected routes return
401 otherwise).

### Project layout

```
webcrawl/
  __init__.py     public API            __main__.py   `python -m webcrawl`
  crawler.py      BFS engine (the heart) — fetch, follow, incremental, robots, rate-limit, max-depth
  extract.py      HTML → title/text + link discovery
  chunk.py        split text into retrieval-friendly passages
  output.py       portable files (pages.json/chunks.jsonl/manifest+hashes) + connectors
  search.py       keyword ranking            (scope Level 2)
  embed.py        embeddings (provider-agnostic)   (scope Level 3)
  vectorstore.py  local JSON + sqlite vector index  (scope Level 3, no external service)
  connectors.py   pgvector + Pinecone vector-DB push connectors   (scope Level 3)
  pdf.py          application/pdf text extraction   (optional `[pdf]` extra)
  netfilter.py    SSRF guard — reject private/loopback/link-local hosts
  render.py       optional Playwright JS rendering for SPAs
  hybrid.py       embedding cache + hybrid (keyword + meaning) search        (new in 1.1)
  schedule.py     scheduled, lease-claimed crawls                            (new in 1.1)
  schedule_cli.py `webcrawl-schedule` command                                (new in 1.1)
  _lock.py        cross-process file lock (stdlib)
  social/         YouTube / X / Facebook posts -> summary cards              (new in 1.2)
                  accounts.py fetchers.py summarize.py store.py scan.py cli.py
  cli.py          terminal entry point
  service.py      FastAPI wrapper (background-job crawl API; `webcrawl.service:app`)
examples/         run_example.py, crawl_to_vectorstore.py
DEMOS/            click-to-run launchers (run_html_demo = browser demo)
tests/            453 tests, pytest (run: `make test`)
```

### Test

```bash
make test                 # or: pytest
```

Nothing is skipped with the `[dev]` extra installed. Without it, the API and PDF
tests skip. `DEMOS/run_tests.command` does the same in its own `DEMOS/.venv`.

---

Docs mirror of the private repository Dominares-Tools/crawl; updated with each release.
