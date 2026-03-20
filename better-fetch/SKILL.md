---
name: better-fetch
description: "Fetch and convert web pages to clean Markdown with automatic failover (markdown.new → Jina Reader). Supports single-page fetch and multi-page site crawl."
---

# BetterFetch

Fetch and convert web pages to clean Markdown. Supports single-page fetch and multi-page crawl, with automatic failover between providers.

## Provider Chain

1. **Primary — markdown.new**: `https://markdown.new/{URL}`
2. **Fallback — Jina Reader**: `https://r.jina.ai/{URL}`
3. **Last resort**: Fetch the original URL directly.

Failover triggers: HTTP 403/429, empty response, bot-detection pages, timeout, or any non-200 error from the current provider. When falling back, briefly note which provider failed and why.

### Jina Reader Authentication (optional)

Jina Reader works without authentication but has stricter rate limits. To increase quota, pass a bearer token via the `Authorization` header:

```
Authorization: Bearer jina_<your_token>
```

Get a free API key at https://jina.ai/reader/.

## Input

The user provides URLs and options, optionally followed by a prompt or question.

## Mode Detection

- **Single-page mode** (default): No flags, or `--method=browser|ai|auto`
- **Crawl mode**: When `--crawl` flag is present, or the user explicitly says "crawl" / "crawl the entire site"

## Single-Page Instructions

1. Parse the input: extract URL(s) and any user prompt.
2. For each URL, try the provider chain in order:
   - **markdown.new**: Fetch `https://markdown.new/{URL}` with the instruction "Return the full markdown content as-is. Do not summarize."
   - **Jina Reader** (on failure): Fetch `https://r.jina.ai/{URL}` with the same instruction. Jina Reader returns Markdown directly.
   - **Direct fetch** (on failure): Fetch the original URL and note the fallback.
3. If the user provided a prompt/question, process the returned markdown to answer it. Otherwise, present the full markdown content.

## Crawl Instructions

Crawl is only available via markdown.new. If the crawl API itself is blocked, inform the user and suggest single-page mode with Jina Reader as an alternative.

1. Parse the input: extract the target URL and any crawl options.
2. **Start crawl** by sending a POST request:
   ```
   POST https://markdown.new/crawl
   Content-Type: application/json

   {"url": "<TARGET_URL>", "limit": <LIMIT>, "depth": <DEPTH>}
   ```
   Defaults: limit=50, depth=5. Override with `--limit=N`, `--depth=N`.
3. **Extract jobId** from the JSON response.
4. **Poll status** every 10 seconds (max 30 attempts):
   ```
   GET https://markdown.new/crawl/status/<JOB_ID>?format=json
   ```
   Check if the job is complete. If still running, show progress and wait.
5. **Retrieve results** once complete:
   ```
   GET https://markdown.new/crawl/status/<JOB_ID>
   ```
   This returns the combined Markdown of all crawled pages.
6. If the user provided a prompt, process the combined markdown. Otherwise, present it.

### Crawl Options

| Flag | Description | Default |
|------|-------------|---------|
| `--limit=N` | Max pages (1-500) | 50 |
| `--depth=N` | Max link depth (1-10) | 5 |
| `--render` | Enable JS rendering for SPAs | off |
| `--source=sitemaps\|links\|all` | Discovery method | all |
| `--include=pattern` | URL wildcard allowlist | auto |
| `--exclude=pattern` | URL wildcard blocklist | none |

## Examples

```
# Single page fetch
https://example.com

# Single page with a question
https://docs.python.org/3/library/asyncio.html What are the main asyncio primitives?

# Multi-page crawl
--crawl https://docs.example.com

# Crawl with options
--crawl --limit=20 --depth=3 https://docs.example.com Summarize the API reference

# Crawl a JS-rendered SPA
--crawl --render https://spa-docs.example.com
```
