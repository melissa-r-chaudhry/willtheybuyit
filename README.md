# WillTheyBuyIt

**What it is:** A small public AI product-idea validator with a server-side proxy architecture.
**Why it existed:** The originally-supplied tool called an LLM API directly from the browser with an embedded key — a real security hole, found and fixed before launch.
**What I owned:** The proxy redesign (same-origin, server-pinned parameters, per-IP rate limiting) and two real bugs found in production the same day as launch.
**Stack:** n8n (proxy/backend), vanilla JS (client), Claude API.
**Production status:** Live, public.

---

A small public tool: paste a product idea, get a live-web-search-backed
GREEN/YELLOW/RED verdict on whether real demand signals exist for it, then
(if green/yellow) a draft outline and pricing/channel suggestions.

Single-page, no account required, no tracking beyond the server-side rate
limiter described below.

See **[PROXY_ARCHITECTURE.md](./PROXY_ARCHITECTURE.md)** for the real
engineering story: this started as a static page calling an LLM API
directly from the browser with an embedded key, and was reworked into a
same-origin proxy with server-pinned parameters and per-IP rate limiting
before it ever went live — plus two real bugs found in production the
same day, both fixed with verified before/after tests.

`index.html` in this repo is the actual deployed client code.
