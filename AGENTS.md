# AGENTS.md

Guidance for AI coding agents (Claude Code and others) working in this
repository. Note: this repo's `.gitignore` excludes `CLAUDE.md`/`.claude/`
by deliberate maintainer choice, so this file is `AGENTS.md` instead —
Claude Code and other agents should treat it as the canonical guidance file
here.

## What This Is

**Crawl4AI** (PyPI: `Crawl4AI`, import name `crawl4ai`) is an open-source,
async, LLM-friendly web crawler and scraper: it drives a real browser
(Playwright, with an alternate Patchright/CDP-based path for stealth) or a
lightweight HTTP strategy, then turns pages into clean Markdown, structured
JSON, or other typed output for RAG/agent pipelines. Python >=3.10. This repo
(`RJHuey73/crawl4ai`) is a fork of `unclecode/crawl4ai`. There is also a
Docker-deployable REST/MCP API server (`deploy/docker/`) with its own
security-hardened trust boundary — see Gotchas below.

## Layout

| Path | Purpose |
|------|---------|
| `crawl4ai/` | The installable package. |
| `crawl4ai/async_webcrawler.py` | `AsyncWebCrawler` — the main entry point (`arun`, `arun_many`). |
| `crawl4ai/async_configs.py` | The three core config objects: `BrowserConfig` (how the browser launches), `CrawlerRunConfig` (per-`arun` behavior), `LLMConfig` (model/provider for LLM-backed strategies), plus `HTTPCrawlerConfig`, `ProxyConfig`, `GeolocationConfig`, `SeedingConfig`, `DomainMapperConfig`. Also owns the untrusted-input deserialization allowlists (see Gotchas). |
| `crawl4ai/async_crawler_strategy.py`, `browser_manager.py`, `browser_adapter.py`, `browser_profiler.py` | Browser lifecycle: launching/pooling Playwright/Patchright contexts, profiles, stealth. |
| `crawl4ai/content_scraping_strategy.py` | `ContentScrapingStrategy` ABC + `LXMLWebScrapingStrategy` (default) — turns raw HTML into structured scrape results (links, media, text). |
| `crawl4ai/extraction_strategy.py` | `ExtractionStrategy` ABC and implementations: `LLMExtractionStrategy`, `CosineStrategy`, `JsonCssExtractionStrategy`, `JsonXPathExtractionStrategy`, `JsonLxmlExtractionStrategy`, `RegexExtractionStrategy`. |
| `crawl4ai/markdown_generation_strategy.py` | `MarkdownGenerationStrategy` ABC + `DefaultMarkdownGenerator`. |
| `crawl4ai/content_filter_strategy.py` | `PruningContentFilter`, `BM25ContentFilter`, LLM-based filters — decide what content survives into the markdown. |
| `crawl4ai/table_extraction.py`, `chunking_strategy.py` | Table extraction and text-chunking strategies (same ABC + implementations pattern). |
| `crawl4ai/deep_crawling/` | Multi-page crawling: `base_strategy.py` (`DeepCrawlStrategy` ABC), `bfs_strategy.py`/`dfs_strategy.py`/`bff_strategy.py` (traversal orders), `filters.py`, `scorers.py`. |
| `crawl4ai/models.py` | Typed Pydantic result models: `CrawlResult` (the central return type), `MarkdownGenerationResult`, `Media`/`Links`, `DispatchResult`, etc. |
| `crawl4ai/async_dispatcher.py` | Concurrency/memory-aware dispatch for `arun_many`. |
| `crawl4ai/adaptive_crawler.py` | Adaptive/exploratory crawling on top of deep-crawl strategies. |
| `crawl4ai/async_url_seeder.py`, `domain_mapper.py` | URL discovery/seeding and domain-to-URL mapping. |
| `crawl4ai/proxy_strategy.py` | `ProxyRotationStrategy`, `RoundRobinProxyStrategy`. |
| `crawl4ai/cache_context.py`, `async_database.py`, `cache_validator.py` | Local SQLite-backed crawl cache. |
| `crawl4ai/script/` | The "C4A script" DSL — `c4ai_script.py` (parser), `c4a_compile.py` (compiles to executable page actions), `c4a_result.py`. |
| `crawl4ai/crawlers/` | Purpose-built crawlers (e.g. `google_search/`, `amazon_product/`) layered on the core strategies. |
| `crawl4ai/cli.py` | The `crwl` console script (crawl/deep-crawl/LLM-extract from the command line). |
| `crawl4ai/legacy/` | Superseded sync-era code (`web_crawler.py`, `crawler_strategy.py`, old CLI) kept for backward compatibility — don't build new features here. |
| `crawl4ai/install.py`, `migrations.py`, `model_loader.py` | Back `crawl4ai-setup`, `crawl4ai-migrate`, `crawl4ai-download-models` console scripts. |
| `deploy/docker/` | The FastAPI-based Docker server: `server.py`/`api.py` (routes), `auth.py`/`auth_gate.py` (auth), `egress_broker.py`/`egress_proxy.py` (network trust boundary), `crawler_pool.py`, `job.py`/`work_queue.py`, `mcp_bridge.py` (MCP server), `monitor.py`. See `deploy/docker/ARCHITECTURE.md` and `SECURITY-VERIFY.md`. |
| `tests/` | Pytest suite; see Testing below for the directory breakdown. |
| `docs/md_v2/` | The published documentation source (mkdocs); `docs/examples/` holds runnable example scripts; `docs/blog/` holds per-release notes (source of truth for what shipped in each version). |
| `docs/codebase/` | Short internal architecture notes (`browser.md`, `cli.md`). |

## Commands

```bash
# Install for development (editable) + browser binaries
pip install -e .
crawl4ai-setup                              # post-install: browser install + home dir (~/.crawl4ai) setup
crawl4ai-doctor                             # environment diagnostics

# Run a crawl
crwl https://example.com -o markdown
crwl https://docs.crawl4ai.com --deep-crawl bfs --max-pages 10

# Tests (pytest; no pytest.ini/pyproject [tool.pytest] config — run directly)
pytest tests/                               # full suite (slow: launches real browsers, hits real URLs)
pytest tests/regression/ -v -m "not network" --tb=short -q   # regression suite, local-server tests only
pytest tests/unit -v                        # narrower/fast subset
pytest path/to/test_file.py::test_name -v   # single test

# Formatting (per CONTRIBUTING.md; no pinned config file in this repo)
black crawl4ai/ tests/
```

There is no `ruff`/`flake8`/`mypy` config or pinned dev-dependency group in
`pyproject.toml` — `black` for formatting and `pytest` for tests are the only
tooling CONTRIBUTING.md asks for. CI (`.github/workflows/`) does **not** run
the general test suite on PRs; the only automated test gates are Docker
security suites (`security.yml`, matching `deploy/docker/tests/test_security_*.py`
and `test_security_default_posture.py`). Treat `pytest` as something you must
run yourself before proposing a change, not something CI will catch for you.

### Docker server

```bash
cd deploy/docker
docker compose up --build          # see deploy/docker/README.md for full options/env vars
```

## Architecture

- **Everything is an injectable strategy behind an ABC.** Scraping
  (`ContentScrapingStrategy`), extraction (`ExtractionStrategy`), markdown
  generation (`MarkdownGenerationStrategy`), content filtering, chunking,
  table extraction, deep-crawl traversal (`DeepCrawlStrategy`), and proxy
  rotation (`ProxyRotationStrategy`) are all `ABC` + concrete subclasses,
  passed into `CrawlerRunConfig` (or `BrowserConfig` for proxy/browser-level
  concerns). Adding a new capability almost always means adding a new
  strategy subclass, not branching inside `AsyncWebCrawler`.
- **Three-config pattern.** `BrowserConfig` governs how the browser process
  itself is launched (once, at `AsyncWebCrawler` construction);
  `CrawlerRunConfig` governs a single `arun()`/`arun_many()` call (cache mode,
  strategies, JS to run, screenshot/PDF capture, session id, ...); `LLMConfig`
  is passed into whichever strategy needs an LLM. Don't conflate the two —
  e.g. proxy/session/JS-execution settings belong on `CrawlerRunConfig`, not
  `BrowserConfig`, for per-request overrides.
- **`CrawlResult` (in `models.py`) is the shared return contract.** Every
  crawl path (single `arun`, `arun_many`, deep crawl, adaptive crawl)
  ultimately produces `CrawlResult` objects (wrapped in
  `CrawlResultContainer` for the batch/streaming cases) — markdown, extracted
  content, media/links, screenshots, and status all live here rather than in
  ad hoc return shapes.
- **Two browser automation backends.** Playwright is the default; Patchright
  (a stealth-patched Playwright fork) is an alternate backend selected via
  `BrowserConfig`/`browser_adapter.py`, primarily to reduce bot-detection
  fingerprinting — don't assume Playwright APIs are the only ones in play in
  `browser_manager.py`/`browser_adapter.py`.
- **`crawl4ai/legacy/` is frozen, not deprecated-but-touchable.** It's kept
  only for backward compatibility with the pre-`AsyncWebCrawler` sync API.
  New features go in the async-first modules at the package root.
- **The Docker server is a separate trust domain from the SDK.** See
  Gotchas — `deploy/docker/` code must go through the untrusted-config
  allowlists in `async_configs.py` rather than assuming SDK-level trust.

## Testing

- Pytest, `pytest-asyncio` style (`@pytest.mark.asyncio` on async tests,
  no `pytest.ini`/`[tool.pytest.ini_options]` in this repo — markers like
  `network` are registered ad hoc per test subtree via that subtree's
  `conftest.py`, e.g. `tests/regression/conftest.py`).
- `tests/` is organized by concern, not 1:1 with `crawl4ai/`: `unit/`,
  `integration/`, `regression/`, `adversarial/`, `async/`, `browser/`
  (with `browser/docker/`, `browser/manager/` subfolders), `cache_validation/`,
  `deep_crawling/` (also note the pre-existing typo'd sibling
  `deep_crwaling/` — check both when working on deep-crawl tests),
  `mcp/`, `docker/`, `cli/`, `proxy/`, `loggers/`, `memory/`, `profiler/`,
  `releases/`, plus many top-level `test_issue_<N>_...py` / `test_bug_...py`
  regression files named after the GitHub issue/bug they pin down.
- `tests/regression/conftest.py` spins up a real local HTTP server with
  crafted fixture pages (no mocking) for deterministic tests, and registers
  the `network` marker for tests that hit real external URLs — run
  `pytest tests/regression/ -m "not network"` to skip those.
- `deploy/docker/tests/` has its own `conftest.py` and is the one directory
  actually gated in CI (`security.yml` runs
  `pytest deploy/docker/tests/test_security_*.py` and
  `test_security_default_posture.py`).
- `.claude/commands/c4ai-check.md` documents this repo's actual verification
  loop for a code change: write adversarial tests into
  `tests/regression/test_tmp_changes.py` (real browser crawls, no mocking),
  run them, then run the full `tests/regression/` suite with
  `-m "not network"` to catch regressions, then delete the temp file. Prefer
  this over inventing an ad hoc verification process.

## Conventions

- Follow `CONTRIBUTING.md`'s branch model: `main` is release-tagged and
  stable (no direct PRs); `develop` is the integration branch PRs target;
  `next` is the lead maintainer's experimental branch. In this fork, treat
  the fork's default branch as the effective integration point unless told
  otherwise.
- Format with `black` before committing (per CONTRIBUTING.md); there's no
  enforced pre-commit hook or CI check for it in this repo, so it's on you.
- New strategy classes should subclass the relevant ABC and implement its
  abstract methods fully — check a sibling implementation (e.g.
  `JsonCssExtractionStrategy` next to `JsonXPathExtractionStrategy`) for the
  expected shape before adding a new one.
- Release notes live in `docs/blog/release-vX.Y.Z.md`, written in the
  maintainer's first-person voice with code examples and migration notes —
  `README.md`'s "Recent Updates" section links out to these; don't hand-edit
  version numbers without checking `crawl4ai/__version__.py`,
  `mkdocs.yml`'s `site_name`, and the Docker version args CONTRIBUTING.md
  lists for a release.

## Gotchas

- **Untrusted-input deserialization is allowlisted, not inferred.**
  `async_configs.py` defines `Provenance` (`TRUSTED` for SDK/in-process
  construction vs. `UNTRUSTED` for a Docker-server request body),
  `UNTRUSTED_ALLOWED_TYPES` (a strict subset of constructible config/strategy
  classes — notably excluding anything LLM-backed, proxy-related, or
  deep-crawl-related), `UNTRUSTED_FORBIDDEN_FIELDS`, and
  `UNTRUSTED_FIELD_ALLOWLIST`. A field not explicitly allowlisted is silently
  dropped from an untrusted body (forward-compatible); a forbidden field
  raises `UntrustedConfigError`, which the Docker layer maps to HTTP 400. If
  you add a new `BrowserConfig`/`CrawlerRunConfig` field that a Docker
  request body should be able to set, you must add it to the relevant
  allowlist — omitting it does not "just work," it gets silently dropped.
  SDK/library callers are unaffected (`TRUSTED` by default).
- **`crawl4ai-setup` and `setup.py` both write to `~/.crawl4ai/`** (or
  `$CRAWL4_AI_BASE_DIRECTORY` if set), including deleting and recreating its
  `cache/` subfolder — be aware a plain `pip install -e .` (which runs
  `setup.py`) has this side effect on the machine it's run on.
- **Two CLIs exist**: `crwl` (`crawl4ai/cli.py`, current) and a legacy one
  under `crawl4ai/legacy/cli.py`. Only extend the current one unless
  specifically working on backward-compat.
- **`tests/deep_crwaling/` (typo) sits alongside `tests/deep_crawling/`** —
  both exist in this repo; check which one a given deep-crawl test actually
  lives in rather than assuming the correctly-spelled directory is the only
  one.
- **CI does not run the general pytest suite.** `.github/workflows/main.yml`
  is Discord notifications only; `security.yml` only runs the Docker
  security tests. A PR can merge without the broader `tests/` suite having
  been run by CI — run it yourself.
- **`LXMLWebScrapingStrategy` is the default scraper; `WebScrapingStrategy`
  is kept only as a backward-compatible alias** for it (see
  `content_scraping_strategy.py`) — prefer the `LXML...` name in new code.
- Browser-dependent tests are genuinely slow and require Playwright/Patchright
  browser binaries to be installed (`crawl4ai-setup` or
  `python -m playwright install --with-deps chromium`); don't assume a bare
  `pip install -e .` without that step is enough to run most of `tests/`.
