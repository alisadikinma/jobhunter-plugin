---
name: scrape-company-careers
description: Pull a target company's `/careers` page directly via self-hosted Firecrawl, extract structured job listings, and POST them as `scraped_jobs` rows. Higher signal than aggregator boards because the listings are first-party and rarely redundant. Useful when you've identified a specific company worth applying to (e.g. via Apollo / Crunchbase / a LinkedIn discovery skill) and want their full open-role list rather than just whatever bubbled up on RemoteOK. Calls back to FastAPI via X-Callback-Secret. Invoked by the FastAPI subprocess spawner with --api-url, --api-token, --job-id and a --company-domain or --company-id arg.
---

# `/jobhunter:scrape-company-careers` — first-party career-page scraper

**Model:** Sonnet (extraction)
**Reference:** `refs/refs-scoring.md` (re-uses the variant rubric for tagging)
**Pairs with:** `companies` table (`backend/app/models/company.py`)

Solves the "aggregator boards never list every role" gap. When Ali finds
a company worth pursuing — via Apollo, LinkedIn, recommendation, or just
seeing one role on RemoteOK — this skill pulls EVERY currently-open role
on that company's careers page and writes them as `scraped_jobs` so they
flow into the same scoring + tailoring pipeline.

## Arguments

Shared (from CLAUDE.md): `--api-url`, `--api-token`, `--job-id`.

Required (one of):
- `--company-id <int>` — existing `companies` row; reads `domain` from DB.
- `--company-domain <domain>` — e.g. `everfield.com`. Skill creates a
  `companies` row if one doesn't exist.

Optional:
- `--firecrawl-url` (default `http://firecrawl-api:3002`).
- `--max-roles` (default 30) — cap to keep cost down.

## Why first-party > aggregator

| Aspect | Aggregator boards (RemoteOK, WWR, etc.) | Company careers page |
|---|---|---|
| Latency | Days behind (companies post here last) | Live |
| Coverage | Top-tier roles only; senior/exec rarely listed | Everything |
| Description quality | Truncated for board layout | Full text |
| Internal vs contract | Mixed | Often labelled |
| Contact info | None | Sometimes lists hiring manager / recruiter |

The cost: scraping company-specific pages is brittle. Each company has its
own careers UI (Greenhouse, Lever, Ashby, Workable, custom). Firecrawl
absorbs that variability.

## Process

1. **Resolve company.**
   ```
   GET {api-url}/api/callbacks/context/{job-id}
   Header: X-Callback-Secret: {api-token}
   ```
   The callback returns `{company: {id, name, domain}}`. If only
   `--company-domain` was passed, the FastAPI dispatcher creates a
   stub `companies` row before the GET.

2. **Discover careers URL.**
   Try in order, take the first that returns a non-404:
   - `https://<domain>/careers`
   - `https://<domain>/jobs`
   - `https://<domain>/career`
   - `https://<domain>/work-with-us`
   - `https://<domain>/join`
   - `https://<domain>/about/careers`

   Common ATS subdomains as a fallback (these are nearly always public):
   - `https://jobs.lever.co/<slug>` (slug usually = company name lowercased)
   - `https://boards.greenhouse.io/<slug>`
   - `https://jobs.ashbyhq.com/<slug>`
   - `https://<slug>.workable.com/`

   If none resolve, return `status: failed` with
   `error_message: "no careers page discoverable for <domain>"`.

3. **Scrape via Firecrawl.**
   ```
   POST {firecrawl-url}/v1/scrape
   Body: {"url": "<careers-url>", "formats": ["markdown"], "onlyMainContent": true}
   ```
   Markdown is friendlier for LLM extraction than raw HTML.

4. **Extract roles via Sonnet.**
   For each markdown section that looks like a job listing, emit:
   ```typescript
   {
     title: string;
     location: string | null;     // "Remote", "Berlin · Hybrid", etc.
     team: string | null;         // "Engineering", "AI Research", etc.
     url: string;                 // absolute URL to the role detail
     description_snippet: string; // first 200-500 chars from markdown
     remote: boolean;
   }
   ```

   Be conservative: skip sections that look like blog posts, marketing
   copy, or "Why work at X" pages. Heuristic — a role section usually has
   a title + apply link + location/team.

5. **Variant-tag each role.**
   Run the same 3-variant rubric from `refs/refs-scoring.md` against each
   role (cheap, since titles + snippets are short). Emit
   `suggested_variant: "vibe_coding" | "ai_automation" | "ai_video" | null`.
   `null` = not a target variant (e.g. role is sales / finance / ops).

6. **Stream progress.**
   ```
   PUT {api-url}/api/callbacks/progress/{job-id}
   Body: {
     "progress_pct": <0-100>,
     "current_step": "extracted 5/12 roles",
     "log_message": "discovered: Senior AI Engineer (vibe_coding) at /careers/123"
   }
   ```

7. **POST completion.**
   ```
   PUT {api-url}/api/callbacks/complete/{job-id}
   Body: {
     "status": "completed",
     "result": {
       "company_id": <int>,
       "company_name": "Everfield",
       "careers_url": "https://everfield.com/careers",
       "roles": [
         {
           "title": "Senior AI Engineer",
           "location": "Remote · USA/EU",
           "team": "Engineering",
           "url": "https://everfield.com/careers/senior-ai-engineer",
           "description_snippet": "...",
           "remote": true,
           "suggested_variant": "vibe_coding"
         },
         ...
       ],
       "roles_count": 12,
       "skipped_count": 3
     }
   }
   ```

   Backend dispatcher inserts each role into `scraped_jobs` with
   `source = "company:<domain>"` so they're distinguishable from
   aggregator hits, and runs the deduplicator.

## Failure modes

- **All careers URLs 404** → `status: failed`, `error_message: "no careers page discoverable for <domain>"`. Don't guess emails.
- **Firecrawl unreachable** → retry once after 5s; still down → `status: failed`. No silent degradation to Bash/curl.
- **Markdown contains zero job-shaped sections** → `status: completed` with `roles: []` and `error_message: "page rendered but no roles extracted (possibly JS-only — try Apify alternative)"`. Don't error; let caller decide.
- **Cloudflare / WAF block** → Firecrawl will surface the 403; report as `firecrawl_blocked` with the blocked URL.

## Non-goals

- **No deep-crawl into per-role detail pages.** This skill returns the
  listing-level data. If a follow-up wants the full JD, that's a separate
  scrape via the existing `enrichment_service` (Phase 5.5).
- **No company discovery.** This skill assumes the caller already knows
  WHICH company. Discovery (Apollo / Crunchbase / LinkedIn) is a
  separate skill.
- **No authentication.** Doesn't try logged-in scrapes (some Lever/Greenhouse
  boards require login for full descriptions). If a board needs auth,
  return `status: failed` with `requires_auth: true` so the operator
  can route it through the Apify pool instead.

## Max tokens

Careers pages are ~10-50k tokens markdown after `onlyMainContent`. Sonnet
200k input is plenty. Output is structured JSON — small.
