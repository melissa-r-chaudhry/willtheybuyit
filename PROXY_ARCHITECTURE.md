# Server-Side Proxy Architecture

`index.html` in this repo is the real, deployed client for a small public
tool: paste a product idea, get a live-web-search-backed verdict on
whether there's real demand for it. The interesting engineering problem
isn't the idea-validation logic — it's that the page needed to call an
LLM API **without ever holding a spendable API key in the browser.**

## The problem, found before shipping

The tool was originally supplied as a single static HTML file that called
the LLM API directly from client-side JavaScript, with the key embedded in
that same file. That means:
- Anyone opening dev tools gets a live, spendable API key.
- Anyone who finds the key can run unlimited paid requests on someone
  else's account, with no rate limit and no way to revoke access short of
  rotating the key (which breaks the page for everyone until redeployed).

This was caught and fixed before the tool went live — not something a
user or reviewer found after the fact.

## The fix: same-origin proxy, server-decided parameters

```
Browser → POST /api/<tool-name>  (same origin — zero CORS needed)
            │
            ▼
        reverse proxy (path rewrite only)
            │
            ▼
        workflow engine, server-side
          ├─ per-IP rate limiting (in-memory, rolling window)
          ├─ real API key, never exposed to any client
          └─ model / max-tokens / search-budget: ALL decided here,
             never trusted from the client request body
            │
            ▼
        LLM API call → response passed back to the browser
```

**Key design decisions:**

1. **Same-origin, not cross-origin.** The public page and the proxy
   endpoint share a hostname (`/api/...` on the same domain as the page
   itself), reached via a path-based reverse-proxy rewrite rather than a
   separate API subdomain. This was a deliberate choice specifically to
   avoid needing any CORS configuration at all — fewer moving parts, fewer
   ways to misconfigure an origin check.

2. **Rate limiting with no new infrastructure.** Rather than standing up a
   database table or external rate-limit service for a single counter,
   the proxy keeps an in-memory `{ip: [timestamps]}` map inside the
   workflow engine's own persisted execution state, filtered to a rolling
   window, with a small-probability sweep on each request to prune stale
   entries so it never grows unbounded. Verified this actually persists
   across separate invocations (not assumed) before relying on it.

3. **Every cost-relevant parameter is pinned server-side.** Model choice,
   max output tokens, and search-tool call budget are all hardcoded in the
   proxy, never read from the client's request — even though the original
   client code technically sent fields like `maxTokens`. This closes two
   different failure modes at once: a malicious caller can't request a
   more expensive model or an unbounded search budget, and a benign client
   bug (see below) can't silently under-serve real users either.

## Two real bugs found in production, same day as launch

**1. Silently wrong HTTP status codes.** The workflow engine's
webhook-response node accepted a `responseCode` parameter, but setting it
at the top level of the response config was silently ignored — the
endpoint always answered `200`, even when the response body correctly
said "rate limited." Verified with real requests: `200` with the
original config shape, `429` once the code was nested under the engine's
`options` block instead. Worth knowing if you build on a similar
workflow-engine webhook response node: **test the actual HTTP status with
curl, don't just trust that the response body looks right.**

**2. A client-side parameter that could never actually vary.** While
raising the output-token ceiling to fix truncated responses, found that
the ceiling computation on the server (`min(request value, server cap)`)
never actually used the request value — every real client call site sent
the *same* fallback number for that field, so the "fallback" path was
actually the only path that ever ran, and the real ceiling was always far
lower than intended. Fixed by having the server own that value outright
instead of deferring to a client-supplied one the client never actually
varied — which is also the more defensible choice on security grounds,
consistent with every other server-pinned parameter above.

## Client-side resilience (visible in `index.html`)

- **Tolerant JSON extraction, not a bare `JSON.parse`.** The model
  occasionally prefixes its JSON response with a sentence of context
  (e.g. explaining it hit its search-call budget) instead of confining
  that note to the field the prompt asked for. Rather than special-casing
  that one failure, the client extracts the `{...}` span out of the
  response text before parsing — robust to any stray prose around the
  JSON, not just the specific case that surfaced it.
- **A real, previously-invisible click-handler bug**, found via live
  cross-browser testing: a button's click listener was wired as
  `addEventListener('click', runCheck)` instead of
  `addEventListener('click', () => runCheck())` — passing the function
  reference directly meant a real click invoked the handler with the
  browser's `MouseEvent` as its first argument, which the function then
  tried to call `.trim()` on, throwing inside an event handler with
  nothing downstream to catch it. It happened to work by accident in one
  browser (a separate, correctly-wired keyboard-shortcut path fired on the
  first test there) and silently failed in another — a bug that looked
  like a browser-compatibility issue but wasn't one.
