---
name: geektime-article-summarizer
description: Use when extracting or summarizing already-logged-in Geekbang article pages on time.geekbang.org from a URL or the current tab, including title,正文,评论区, or moving to the next article in the column.
---

# Geekbang Article Summarizer

## Overview

Use the current logged-in Chrome session through `chrome-local` whenever possible. Accept either a Geekbang article URL or the current selected Geekbang tab. Prefer the real Chrome DOM snapshot first. Use the in-app browser / MCP_DOCKER surface only as a quick negative check; if it returns `HTTP ERROR 451`, `chrome-error://chromewebdata/`, a login page, or incomplete article content, switch to the logged-in Chrome extension backend instead of retrying the same failing surface.

## Chrome Runtime

1. Read and follow `chrome-local` when the real Chrome browser, existing tabs, or logged-in sessions are needed.
2. Bootstrap the installed Chrome plugin from its absolute `chrome/latest/scripts/browser-client.mjs` path. The current runtime returns the agent; do not expect it to assign a global `agent` object:

```js
const { setupBrowserRuntime } = await import("<plugin root>/scripts/browser-client.mjs");
const agent = await setupBrowserRuntime();
const chrome = await agent.browsers.get("chrome");
nodeRepl.write(await chrome.documentation());
```

3. Name the session, then call `chrome.user.openTabs()` and claim only the matching GeekTime tab. Treat discovery as read-only.
4. Do not patch `browser-client.mjs`. If bootstrap reports a missing runtime method or a missing paired Browser module, treat it as an installed-plugin/cache version mismatch: update or reinstall Codex's Chrome integration through **Settings → Computer use**, then restart Codex and Chrome before retrying.

## Quick Workflow

1. If the user gave a URL, open or select it in the same logged-in Chrome session. If no URL is given, use the current selected Geekbang tab.
2. Confirm the final page URL is on `time.geekbang.org`, is not `chrome-error://chromewebdata/`, and is not a login or purchase-block page.
3. Read title, outline,正文, and visible comments from `tab.playwright.domSnapshot()`.
4. If正文或评论区被折叠、截断或延迟加载，use read-only `tab.playwright.evaluate(...)` in the same logged-in page.
5. Use `/serv/v1/article`, `/serv/v1/column/articles`, and `/serv/v4/comment/list` only after confirming same-origin login state and page-side `fetch` availability.

## Retrieval Pattern

Use this only as a logged-in same-origin fallback, not as the primary path:

```js
await fetch('/serv/v1/article', {
  method: 'POST',
  credentials: 'include',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ id, include_neighbors: true, is_freelyread: true })
})
```

Use the returned `article_title`, `article_content`, `article_summary`, `neighbors`, and comment data to build the summary.

## Output Format

- 标题
- 正文核心论点
- 结构化要点
- 评论区高频观点 / 争议点
- 可选：上一篇 / 下一篇
- 默认不贴长段原文，只保留必要短引述

## Guardrails

- Do not paste long verbatim passages.
- Do not use public web search for this task.
- If the page is not logged in, not on `time.geekbang.org`, or resolves to `chrome-error://chromewebdata/`, say so and stop.
- Treat default IAB / MCP_DOCKER `HTTP ERROR 451` as a failed surface, not as article evidence.
- Keep summaries high-density and aimed at experienced developers.
- If the user asked for full text, provide the best extractable正文 plus a compact summary, not a literal mirror of the page.

## Common Mistakes

- Using the legacy bootstrap that assumes `setupBrowserRuntime()` populates a global `agent`.
- Patching bundled browser-client files instead of reconciling an outdated plugin cache.
- Staying on IAB / MCP_DOCKER after it returns `451` or incomplete content.
- Assuming the API fallback works before confirming same-origin logged-in page context and `fetch` availability.
- Relying only on visible DOM when the article body is partially loaded.
- Ignoring the column API when asked for the next article.
- Mixing正文、评论和个人解读 without clear section labels.
- Returning too much raw text instead of a compact summary.

