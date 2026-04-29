---
name: find-contact
description: Discover the best person to cold-email at a target company — prefers the hiring manager (VP Engineering, Head of AI, Eng Lead) over generic recruiter or HR. Variant-aware: vibe_coding targets engineering leadership, ai_automation targets ops/COO, ai_video targets creative leads. Uses self-hosted Firecrawl to scrape company About/Team/Contact pages, Sonnet to extract structured contacts, and a fallback chain (hiring manager → recruiter → careers@<domain>). Calls back to FastAPI via X-Callback-Secret. Invoked by the FastAPI subprocess spawner with --api-url, --api-token, --job-id, OR ad-hoc via cron.
---

# `/jobhunter:find-contact` — discover hiring-manager email per application

**Model:** Sonnet (extraction, not creative writing)
**Reference:** `refs/refs-contact.md`

Fills in `applications.contact_name`, `contact_email`, `contact_title` so
that `/jobhunter:cold-email` has a real recipient instead of drafting into
the void.

## Arguments

Shared (from CLAUDE.md): `--api-url`, `--api-token`, `--job-id`.

Optional:
- `--firecrawl-url` (default `http://firecrawl-api:3002`) — internal Firecrawl base URL.
- `--max-pages` (default 5) — cap pages scraped per company to keep cost down.

## Why hiring manager beats recruiter beats HRD

| Role | Response rate | Why |
|---|---|---|
| **Hiring manager** (VP Eng, Head of AI, Eng Lead, Director) | 20-40% | Final decision-maker, can override formal filters |
| **Internal recruiter** | 10-20% | Has 50+ open reqs; your email is 1 of 200 |
| **CTO/founder** (only if company <50 people) | 20-30% | Final say; works only for early-stage |
| **HRD / HR Manager** | 5-10% | Forwards "interesting" only; most filed |
| **Generic `careers@<domain>`** | 1-3% | Black hole — auto-routed to ATS |

**Always prefer the human decision-maker** for the role-cluster — and
that decision-maker is variant-specific (see `refs/refs-contact.md`).

## Variant → target persona priority

| Variant | Priority order |
|---|---|
| `vibe_coding` | VP Engineering > Head of Engineering > Engineering Manager > CTO (small co) > Tech Recruiter |
| `ai_automation` | Head of Operations > COO > Automation Lead > Director of Ops > CTO > Recruiter |
| `ai_video` | Creative Director > Head of Content > Marketing Director > Founder (creative agency) > Recruiter |

Pick the highest-priority person actually present on the company website.
If none of the priority titles are findable, fall back to recruiter, then
to a generic `careers@<domain>`.

## Process

1. **Fetch context.**
   ```
   GET {api-url}/api/callbacks/context/{job-id}
   Header: X-Callback-Secret: {api-token}
   ```
   Returns `{application_id, job, company, variant_used}`. Pull
   `company.domain` and `variant_used` for routing.

2. **Build candidate URL list** (cap at `--max-pages`):
   - `https://<domain>/about` / `/about-us`
   - `https://<domain>/team` / `/people` / `/who-we-are`
   - `https://<domain>/contact` / `/contact-us`
   - `https://<domain>/careers` / `/jobs` (recruiter often listed)
   - `https://<domain>/leadership`

3. **Scrape via Firecrawl** (self-hosted — already in stack):
   ```
   POST {firecrawl-url}/v0/scrape
   Body: {"url": "<candidate>", "formats": ["markdown"]}
   ```
   Stream-fail individual 404s; keep all successful pages.

4. **Extract candidates with Sonnet.**
   For each scraped page, extract any tuple
   `{name, title, email, source_url}`. Be liberal — capture every email
   visible, including obfuscated forms (`name [at] domain [dot] com`).

5. **Score candidates against variant priority.**
   For each candidate, compute `tier` (0 = top of priority list, N = fallback)
   from `refs/refs-contact.md` matching against the title. Pick lowest-tier
   match. Tie-break: prefer real name + email > role-title-only > generic.

6. **Fallback chain** if no good candidate found:
   - Tier `recruiter`: any title containing "recruiter", "talent", "people ops"
   - Tier `careers_inbox`: synthesise `careers@<domain>` (mark `confidence: low`)
   - Last resort: leave fields null and report `status: failed` with
     `error_message: "no contact discoverable on company website"`.

7. **Confidence scoring** (0.0–1.0):
   - `0.9+`: real name + verified-format email + matched priority title
   - `0.7–0.9`: real name + plausible email pattern + matched title
   - `0.5–0.7`: title match only, email guessed via pattern
   - `<0.5`: generic inbox or no match — caller may decide to skip cold-email

8. **Stream progress.**
   ```
   PUT {api-url}/api/callbacks/progress/{job-id}
   Body: {
     "progress_pct": <0-100>,
     "current_step": "scraped 3/5 pages",
     "log_message": "found candidate: Sarah Chen (VP Eng) at /team"
   }
   ```

9. **POST completion.**
   ```
   PUT {api-url}/api/callbacks/complete/{job-id}
   Body: {
     "status": "completed",
     "result": {
       "contact_name": "Sarah Chen",
       "contact_email": "sarah.chen@everfield.com",
       "contact_title": "VP Engineering",
       "source_url": "https://everfield.com/team",
       "tier": "hiring_manager",
       "confidence": 0.85,
       "candidates_evaluated": 4
     }
   }
   ```

   On failure: `{"status": "failed", "error_message": "<why>"}`.

## Result shape (consumed by FastAPI dispatcher)

```typescript
type FindContactResult = {
  contact_name: string | null;
  contact_email: string | null;
  contact_title: string | null;
  source_url: string | null;          // where the data came from (audit trail)
  tier: "hiring_manager" | "recruiter" | "careers_inbox" | "founder";
  confidence: number;                  // 0.0–1.0
  candidates_evaluated: number;        // for telemetry
};
```

Backend dispatcher writes `contact_name`, `contact_email`, `contact_title`
into `applications`, and appends a note to `application.notes`:

```
[2026-04-29] Contact discovered: Sarah Chen (VP Engineering) — sarah.chen@everfield.com
Source: https://everfield.com/team (confidence 0.85)
```

## Failure modes

- Firecrawl unreachable → retry once after 5s; if still down,
  `status: failed` with `error_message: "firecrawl unavailable"`. Don't
  guess emails without source verification.
- Domain returns 0 scrapeable pages (SPA, paywall) → fall through to
  `careers_inbox` tier with `confidence: 0.3`.
- Email validation: format-check only (regex `[a-z]+@[a-z0-9.-]+\.[a-z]+`).
  We don't SMTP-verify — that's a deliverability concern, not a discovery
  concern.
- Privacy: skip social-security-style PII (phone, address). Capture only
  professional contact channels.

## Non-goals

- **No Hunter.io / Apollo paid integration in this skill.** Self-hosted
  free chain only. Optional paid tier-up can live in a separate skill
  (`/jobhunter:verify-email`) if needed later.
- **No outbound email sending.** This skill discovers contacts only;
  `/jobhunter:cold-email` drafts; Ali presses send manually.
- **No LinkedIn scraping** here. The Apify pool is the right mechanism if
  Ali decides he wants it; that would be a separate skill
  (`/jobhunter:find-contact-linkedin`).

## Max tokens

Each scraped page ~5-15k tokens; 5 pages = ~75k. Sonnet 200k input is
plenty. Output is small (single JSON object).
