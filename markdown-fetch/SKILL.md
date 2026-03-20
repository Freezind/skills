---
name: markdown-fetch
description: Fetch and convert web pages to clean Markdown using markdown.new. Supports single-page fetch and multi-page site crawl.
---

# Web Fetch via markdown.new

Use markdown.new to fetch and convert web pages to clean Markdown. Supports single-page fetch and multi-page crawl.

## Input

The user provides URLs and options, optionally followed by a prompt or question.

## Mode Detection

- **Single-page mode** (default): No flags, or `--method=browser|ai|auto`
- **Crawl mode**: When `--crawl` flag is present, or the user explicitly says "crawl" / "crawl the entire site"

## Single-Page Instructions

1. Parse the input: extract URL(s) and any user prompt.
2. For each URL, fetch `https://markdown.new/{URL}` with the instruction "Return the full markdown content as-is. Do not summarize."
3. If the user provided a prompt/question, process the returned markdown to answer it. Otherwise, present the full markdown content.
4. If markdown.new fails, fall back to fetching the original URL directly and note the fallback.

## Crawl Instructions

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
