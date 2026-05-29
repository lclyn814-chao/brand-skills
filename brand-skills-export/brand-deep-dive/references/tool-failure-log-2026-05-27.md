# Tool Failure Log — 2026-05-27 (Fabrique deep dive)

Session attempting brand-deep-dive for Fabrique (设计师服装品牌). Documents every approach tried and its outcome.

## Approaches that FAILED

### 1. delegate_task + toolsets=["web"] — RESULTS TRUNCATED
**Attempted:** 6+ times with different configurations
- 2-subagent split (口碑 vs 营销)
- 3-leaf-agent parallel with single-query focus
- Single agent with explicit "write ALL results to file" instruction
- Single agent with "DO NOT summarize" directive

**Result every time:** Subagent makes web_search calls (visible in tool_trace), but the summary returned to orchestrator strips out all actual search result data. Summary only shows the function call signature, never the returned snippets.

**Evidence:** tool_trace shows `web_search` with `result_bytes: 93` and `status: ok` — 93 bytes is an error message or empty response, not real search results.

### 2. Browser navigate to Google — TIMEOUT
`browser_navigate("https://www.google.com/search?q=...")` → "Command timed out after 60 seconds"

### 3. Terminal curl | python3 — SECURITY BLOCK
`curl https://html.duckduckgo.com/html/?q=... | python3 -c "..."` → blocked by tirith security scan: `[HIGH] Pipe to interpreter: curl | python3`

### 4. Terminal curl -o (DuckDuckGo) — TIMEOUT
`curl -sL -A "Mozilla/5.0" "https://html.duckduckgo.com/html/?q=..." -o /tmp/file.html` → "Command timed out after 15s"

### 5. execute_code terminal curl (SearX) — TIMEOUT
`curl -sL --max-time 10 "https://search.sapti.me/search?q=..."` → exit code 28 (timeout)

### 6. Direct curl to Baidu Baike — CAPTCHA
`curl -sL "https://baike.baidu.com/item/Fabrique"` → 2501 bytes of captcha JS, no content

### 7. Direct curl to Sohu article — 404
`curl -sL "https://www.sohu.com/a/818163782_121124361"` → 404 page, article removed/URL wrong

### 8. Direct curl to Fashionista — DATADOME BLOCK
`curl -sL "https://fashionista.com/2025/04/fabrique-collective-designer-brand"` → 775 bytes of DataDome JS challenge

### 9. Browser navigate to Fashionista — DATADOME BLOCK
`browser_navigate("https://fashionista.com/2025/04/fabrique-collective-designer-brand")` → "Access is temporarily restricted" (DataDome)

### 10. Browser navigate to Zhihu article — EMPTY PAGE
`browser_navigate("https://zhuanlan.zhihu.com/p/407595609")` → snapshot shows only a checkbox element (JSON-rendered page, no content extracted)

### 11. Bing CN query deduplication — ALL 10 QUERIES RETURNED IDENTICAL RESULTS
Confirmed: every single query to `cn.bing.com` (regardless of keywords) returned the exact same 10 results:
1. fabrique.cn (brand site)
2. baike.baidu.com (encyclopedia)
3. jd.com (JD store)
4. sohu.com (PR article)
5. fulemi.com (3rd party store)
6. qq.com (PR article)
7. fabrique.co (Thailand site)
8. taobao.com (Taobao store)
9. zhihu.com (brand promo)
10. fashionista.com (English article)

This is Bing CN's server-side caching/fingerprinting, not actual lack of content. Switching to "国际版" toggle didn't help. Adding `&cc=us&setlang=en` still redirected to `cn.bing.com`.

## What ACTUALLY worked

### 1. browser_navigate to Bing CN → scroll → snapshot
This rendered the search results page and the snapshot captured all 10 results with titles, URLs, and snippet text. This is the ONLY reliable data extraction path.

### 2. execute_code + Python requests to Bing
`requests.get("https://www.bing.com/search?q=...", headers=...)` → returned HTML with the same 10 snippets extractable via regex. Works as a backup when browser is unavailable.

### 3. Synthesizing from limited data
With only 10 results (mostly brand site + PR + encyclopedia + shopping), the best strategy is:
- Extract hard facts from Baidu Baike snippet (company name, registration, founder)
- Extract price points from Taobao/JD snippets
- Extract timeline from PR article dates
- Be honest about gaps — mark sections as "无数据" rather than fabricating
- Use the "推断" label for reasonable inferences based on available facts

## Key lessons

1. **Never use delegate_task for web search.** It's a dead end — results always get truncated.
2. **Bing CN is the only working search engine via browser, but it deduplicates aggressively.** Accept the 10 results and extract what you can.
3. **Chinese content sites (Baidu Baike, Zhihu, Sohu) block headless access.** Don't waste time on direct page visits.
4. **For niche brands, be prepared to produce a "low confidence" report.** The template should be filled with "无数据" where appropriate, not left blank or fabricated.
