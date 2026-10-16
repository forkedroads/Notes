
# Summary: Cloudflare and OpenAI's AI Discovery Pilot

**Source:** [Cloudflare and OpenAI: Making AI Discovery Better for Everyone](https://blog.cloudflare.com/cloudflare-openai-discovery-agentic-web/) — Cloudflare Blog, October 15, 2026

---

## The Core Idea

Cloudflare sits as a reverse proxy in front of roughly 20% of the web, giving it a real-time vantage point into traffic, content changes, and page health that no external crawler naturally has. In a research pilot launched in July 2026 (covering ~1% of Cloudflare's network as a test cohort), Cloudflare began feeding **metadata signals** — not page content — to OpenAI's search crawler.

The goal: help ChatGPT's search system discover new, useful pages (especially obscure ones that few other sites link to, like a product manual or a restaurant's menu) and keep already-known pages fresh, **without wasting crawl resources** on dead links, redirects, duplicates, or pages that haven't changed.

Early results: the pilot helped OpenAI discover and index **hundreds of millions of new pages** during the trial.

---

## Who Benefits, and How

### 🤖 The LLM Provider (OpenAI)
- **More efficient use of crawl budget** — crawl resources go toward pages likely to be new or updated, rather than being spent blindly re-checking unchanged pages.
- **Better discovery of long-tail content** — pages with few inbound links (hard to find via traditional link-following crawlers) become discoverable through Cloudflare's traffic-level visibility.
- **Fresher answers** — getting notified close to real-time when a page changes means ChatGPT's citations and answers can reflect current information faster.
- **Safer index** — signals flagging compromised pages (e.g., a fake login form injected into an otherwise legitimate site) help keep malicious content out of search results without penalizing the whole domain.
- OpenAI still independently evaluates, ranks, and decides what to index/surface — Cloudflare's signals only affect *discovery/efficiency*, not quality judgment.

### 🌐 The Cloudflare Customer (Website/Content Owner)
- **Reduced server load** — fewer wasted crawler requests hitting the origin server for pages that haven't changed, don't exist, or just redirect.
- **Faster discoverability** — new or updated content can be surfaced to ChatGPT more quickly than waiting for a traditional crawl schedule.
- **Protection without overreach** — a single compromised page can be flagged and excluded without hurting the reputation/indexing of the rest of the site.
- **Retained control** — a "signal" about a page doesn't override the site owner's access permissions; Cloudflare's **AI Crawl Control** tooling still governs what crawlers can actually access.

### 🙋 The End User (ChatGPT User)
- **More relevant, specific answers** — because the system can surface very specific pages (the exact appliance manual, the current passport fee schedule) instead of only broad, heavily-linked sources.
- **More current information** — faster change-detection means answers are less likely to be based on stale content.
- **Safer results** — flagged/compromised pages are less likely to appear as sources.
- **More paths back to original sources** — ChatGPT Search links back to the originating page/publisher, letting users explore further.

---

## How This Might Actually Work (Signals) — Some Speculation Flagged

The blog post itself is fairly high-level and doesn't give a full technical spec. Based on Cloudflare's existing, publicly documented infrastructure (particularly its 2021-era **Crawler Hints** feature and the **IndexNow** protocol it already uses with Bing/Yandex), the following is a *plausible* technical picture — explicitly inferred/speculative where noted:

### 1. Filtering signals (pages that don't exist or redirect)
- Cloudflare, as the reverse proxy, directly observes **HTTP status codes** returned by origin servers: `404`/`410` (not found), `301`/`302` (redirects).
- It can also spot **patterns** across many requests — e.g., a site that redirects *every* unknown URL to its homepage, or bots repeatedly requesting files that never existed — and flag these as low-value paths before a crawler wastes a fetch on them.

### 2. Change signals (is a page worth revisiting?)
- **Likely mechanism (inferred):** Cloudflare's edge caches a copy of each page. When a new request comes in, if the origin's response differs from the cached version, that's a **cache miss indicating real content change** — a reliable, near real-time signal.
- Supporting signals could include `ETag`/`Last-Modified` HTTP headers, content hashing, and cache-purge events triggered by the site owner.
- This is conceptually the same mechanism behind Cloudflare's existing **Crawler Hints** → **IndexNow** pipeline, which has already pushed over 600 billion "this page changed" notifications to search engines like Bing and Yandex since 2021. The OpenAI pilot plausibly extends this same plumbing with richer signal types.
- **Importantly:** this is a **push model, not a replacement for crawling.** OpenAI's crawler still has to actually fetch and read the page to index its content — Cloudflare doesn't hand over page content, only metadata. What changes is *when* OpenAI bothers to crawl: instead of blindly re-checking pages on a fixed schedule (where, per Cloudflare's own data, more than half of such re-checks find nothing changed), OpenAI can wait for a "this changed" signal before spending crawl resources on a revisit.

### 3. Harmful/compromised-page signals
- **Likely mechanism (inferred):** This probably reuses Cloudflare's existing **WAF and bot/threat detection infrastructure** — the same systems that already protect customer sites from attacks — repurposed to flag single compromised URLs (e.g., an injected fake login form) rather than entire domains.
- Early results suggest these signals can help keep such pages out of the index.

### What's Explicitly *Not* Shared
- **No page content** is shared for model training — only metadata about pages (change status, existence, traffic patterns, safety flags).
- A signal about a page **does not grant crawling permission** — site owners' existing access controls (robots.txt, Cloudflare's AI Crawl Control, block/allow settings) still govern whether a crawler can actually fetch a page.

---

## Open Questions / What's Not Yet Public
- No detailed technical specification (exact data format, transmission protocol, or API) for this specific pilot has been published by either company.
- It's unclear how a site outside the ~1% pilot cohort could opt in, or when/if the program expands.
- No measurement yet on how many of the newly discovered pages are actually cited in ChatGPT answers versus just added to the index.
- The harmful-page detection is described as still in a "testing" phase.

## References
- http://indexnow.org/faq
- https://developers.cloudflare.com/cache/advanced-configuration/crawler-hints/
