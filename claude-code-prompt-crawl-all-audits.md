# Claude Code Prompt: Add Cloudflare Browser Rendering Crawl Across LawFirmAudits.com Audit Suite

## Context

I run lawfirmaudits.com — a hub for 6 diagnostic audits mapped to the law firm client acquisition journey, all powered by Rankings.io. Each audit is a separate Vercel-deployed Next.js app. I want to integrate Cloudflare's new Browser Rendering `/crawl` endpoint as a shared crawl layer that feeds richer website data into whichever audits benefit from it.

### The 6 Audits (and their repos/deploy URLs):

1. **Awareness Audit** — `law-firm-awareness.vercel.app` — Social footprint, traditional media, sponsorships, community presence, brand recall
2. **Perception Audit (LegalBrandGrader)** — `legalbrandgrader.vercel.app` — Reputation, sentiment, review quality, website feel, messaging clarity, trust signals. Scores across 8 brand dimensions benchmarked against exemplar firms.
3. **Consideration Audit** — Coming soon — SEO rankings, paid search, LSAs, content depth, CRO
4. **Client Acquisition Audit** — `client-acquisition-grader.vercel.app` — Intake process, CRM signals, speed to lead, call tracking, follow-up, conversion readiness
5. **AI Readiness Audit** — `ai-readiness-grader-zeta.vercel.app` — AI discoverability, structured data, content clarity, llms.txt, schema markup
6. **Content Audit** — Coming soon — Blog strategy, topical authority, keyword coverage, content depth, conversion copy

### What Cloudflare Browser Rendering `/crawl` Does

New endpoint (open beta, March 10 2026). You POST a starting URL, it crawls the entire site using a headless browser (captures JS-rendered content, chat widgets, dynamic elements), returns pages as HTML, Markdown, or structured JSON. Async — you get a job ID and poll for results. Honors robots.txt. Free on Workers plan.

Docs: https://developers.cloudflare.com/browser-rendering/rest-api/crawl-endpoint/

```bash
# Start crawl
curl -X POST 'https://api.cloudflare.com/client/v4/accounts/{account_id}/browser-rendering/crawl' \
  -H 'Authorization: Bearer <apiToken>' \
  -H 'Content-Type: application/json' \
  -d '{
    "url": "https://example-lawfirm.com/",
    "limit": 75,
    "maxDepth": 3,
    "formats": ["markdown", "html"],
    "filterPatterns": { "exclude": ["/privacy-policy", "/terms-*", "/wp-admin/*"] }
  }'

# Poll results
curl -X GET 'https://api.cloudflare.com/client/v4/accounts/{account_id}/browser-rendering/crawl/{job_id}' \
  -H 'Authorization: Bearer <apiToken>'
```

## What I Need You To Build

### Step 1: Shared Crawl Service (build this first)

Create a shared utility module or package that any of the 6 audit apps can import. This module should:

- Accept a firm URL, kick off a Cloudflare crawl, poll for completion (exponential backoff: 5s start, 30s max, 10 min timeout)
- Return structured crawl results with markdown content per page
- Cache results locally (or in Vercel KV / filesystem) keyed by domain + date so we don't re-crawl the same site across multiple audits in the same session
- Use env vars: `CLOUDFLARE_ACCOUNT_ID` and `CLOUDFLARE_BR_API_TOKEN`
- Default crawl config for law firm sites:
  - `limit`: 75 pages
  - `maxDepth`: 3
  - `formats`: ["markdown", "html"]
  - `exclude`: ["/privacy-policy", "/terms-*", "/wp-admin/*", "/wp-login*", "/feed/*", "/tag/*", "/author/*", "/cart/*", "/checkout/*"]
- Export helper functions like `getPagesByPattern(crawlData, "/practice-areas/*")`, `findPagesContaining(crawlData, "testimonial")`, `countCTAs(crawlData)`, etc.

### Step 2: Integrate Into Each Audit

Look at each audit's existing codebase to understand how it currently fetches/analyzes website data, then enhance it with crawl data. Here's what the crawl unlocks for each:

#### 1. Awareness Audit
- **Current approach**: Likely scanning the homepage + social links
- **Crawl enhancement**: Crawl the full site to find ALL social media links (not just header/footer), press/news pages, community involvement pages, sponsorship mentions, media logos ("as seen in"), podcast/video embed pages
- **New signals from crawl**: Count of unique social platforms linked, presence of dedicated press/media page, community involvement page depth, event sponsorship mentions

#### 2. Perception Audit (LegalBrandGrader)
- **Current approach**: Probably fetching homepage + maybe a few key pages, running through Claude for brand scoring
- **Crawl enhancement**: This is the BIG winner. Full site crawl lets you analyze brand consistency ACROSS pages, not just the homepage. Feed the full markdown of key pages (home, about, practice areas, case results, attorney bios) into Claude for much richer scoring.
- **New signals from crawl**:
  - Messaging consistency (is the brand voice the same on every page?)
  - Trust signal density (testimonials count, case result dollar amounts across the site, awards/badges)
  - Visual/UX proxy signals (CTA count per page, contact form presence on practice area pages, chat widget detection)
  - Attorney bio quality (credentials, experience years, headshot presence via img tags)
  - Differentiation scoring (unique selling propositions, taglines, competitive positioning language)

#### 3. Consideration Audit (coming soon — build this in from the start)
- **Crawl provides**: Full site content for analyzing on-page SEO signals:
  - Title tag and H1 analysis per page
  - Internal linking structure and depth
  - Content length per practice area page
  - Schema/structured data presence in HTML
  - Page count by section (how deep is the blog? how many practice area pages?)
  - Duplicate/thin content detection
  - Local SEO signals (city/state mentions, office location pages, Google Maps embeds)

#### 4. Client Acquisition Audit
- **Current approach**: Scanning for conversion readiness signals
- **Crawl enhancement**: Headless browser rendering means we actually capture JS-rendered elements that a basic fetch misses:
  - Live chat widget detection (Drift, Intercom, LiveChat, etc.)
  - Click-to-call phone number visibility (is it on every page or just homepage?)
  - Contact form analysis (how many fields? is it on practice area pages or just /contact?)
  - After-hours messaging/scheduling tools
  - CTA placement and frequency across the full site
  - Speed-to-lead signals: is there a form on every practice area page? is there a sticky header CTA?

#### 5. AI Readiness Audit
- **Current approach**: Checking for llms.txt, structured data, content clarity
- **Crawl enhancement**: HTML format from crawl gives you raw access to:
  - Full schema.org markup across ALL pages (not just homepage)
  - JSON-LD structured data (LocalBusiness, Attorney, LegalService, FAQPage, etc.)
  - Content clarity scoring on practice area pages (is the content written for humans/AI or is it keyword-stuffed?)
  - Internal linking to authoritative content
  - llms.txt presence (check root URL)
  - robots.txt AI bot directives (is the site blocking AI crawlers?)
  - OpenGraph and meta description quality across pages

#### 6. Content Audit (coming soon — build this in from the start)
- **Crawl provides the entire content corpus**:
  - Blog post count, publish frequency, recency of last post
  - Practice area page count and depth (thin vs. comprehensive)
  - FAQ page presence and question count
  - Video embed count across the site
  - Case results/case studies page depth
  - Content length distribution (histogram of word count per page)
  - Topical coverage map (which PI sub-topics have dedicated pages vs. which are missing?)
  - Internal linking from blog posts to practice area pages
  - Author attribution and expertise signals (E-E-A-T)

### Step 3: Crawl-Once, Use-Many Architecture

Since a user might run multiple audits on the same firm, implement this:

- First audit to run kicks off the crawl and caches results
- Subsequent audits check cache first (cache valid for 24 hours)
- Each audit only reads the slices of crawl data it needs
- If the firm site is already cached, skip the crawl and go straight to analysis
- This keeps the "results in 60 seconds" promise realistic — the crawl might take 30-90 seconds the first time, but subsequent audits are instant

### Step 4: Fallback Gracefully

- If Cloudflare crawl fails or times out, fall back to the existing fetch/analysis approach (don't break the audits)
- If crawl returns fewer pages than expected (robots.txt blocking, etc.), note it in results but don't penalize the firm
- If the API token isn't configured, skip the crawl silently and use existing logic

## Important

- Read each audit's existing codebase FIRST before modifying anything
- Match existing code style, patterns, and conventions in each repo
- This is additive — don't break any current functionality
- Start with the shared crawl module, then integrate into LegalBrandGrader first (it's the most mature), then the others
- For the two "coming soon" audits (Consideration and Content), just set up the integration hooks so the crawl data is ready when I build them out
