# Contact discovery rubric — variant-aware target persona

Used by `/jobhunter:find-contact` to pick the highest-response-rate
person to cold-email per company.

## Why hiring manager > recruiter > HR

Cold email industry baselines (per Apollo/Lemlist 2024 data):

| Role tier | Reply rate | Decision authority | Inbox load |
|---|---|---|---|
| **Hiring manager** (the role's direct boss) | 20-40% | Can override formal filters | Low |
| **Internal recruiter** | 10-20% | Process gatekeeper, KPI-driven | High |
| **C-suite / founder** (small co) | 20-30% | Final say; small co only | High, spam-flagged |
| **HRD / HR Manager** | 5-10% | Forwards-only; doesn't pick candidates | Very high |
| **Generic `careers@<domain>`** | 1-3% | ATS routing | N/A — black hole |

**Rule:** Always prefer the human decision-maker. Variant determines
which decision-maker.

## Priority by variant

### `vibe_coding`

Target: someone who runs an engineering team that ships product fast.

| Tier | Title patterns (case-insensitive substring match) |
|---|---|
| 0 (best) | `vp engineering`, `vp of engineering`, `vp eng` |
| 1 | `head of engineering`, `head of eng`, `chief engineer` |
| 2 | `director of engineering`, `engineering director` |
| 3 | `engineering manager`, `eng manager`, `tech lead` (only if company <100 people) |
| 4 | `cto`, `chief technology officer` (only if company <50 people) |
| 5 | `founder`, `co-founder`, `ceo` (only if company <20 people) |
| 6 (last-human) | `tech recruiter`, `technical recruiter`, `engineering recruiter` |
| 7 (fallback) | generic `careers@<domain>` |

Avoid: `talent partner`, `people operations`, `hr business partner` — these
route through HRD funnel.

### `ai_automation`

Target: someone responsible for operational efficiency or business automation.

| Tier | Title patterns |
|---|---|
| 0 | `head of operations`, `head of ops`, `vp operations`, `vp ops` |
| 1 | `coo`, `chief operating officer` |
| 2 | `automation lead`, `head of automation`, `director of automation` |
| 3 | `director of operations`, `ops director` |
| 4 | `cto`, `chief technology officer` (small/medium co) |
| 5 | `founder`, `ceo` (small co only) |
| 6 (last-human) | `talent acquisition`, `recruiter` |
| 7 (fallback) | generic `careers@<domain>` |

Avoid: `office manager`, `hr coordinator`.

### `ai_video`

Target: someone who owns content quality or creative output.

| Tier | Title patterns |
|---|---|
| 0 | `creative director`, `head of creative` |
| 1 | `head of content`, `vp content`, `content director` |
| 2 | `marketing director`, `head of marketing`, `vp marketing` |
| 3 | `head of brand`, `brand director` |
| 4 | `founder`, `ceo` (only if creative agency <50 people) |
| 5 (last-human) | `creative recruiter`, `talent acquisition` |
| 6 (fallback) | generic `careers@<domain>`, `hello@<domain>`, `contact@<domain>` |

Avoid: `social media manager`, `community manager` — too junior to make hiring decisions.

## Email pattern fallbacks (when name found but email not)

When a likely name is found (e.g. "Sarah Chen, VP Engineering") but no
direct email visible on the page, try these patterns in order:

1. `firstname.lastname@<domain>` (e.g. `sarah.chen@everfield.com`)
2. `firstnamelastname@<domain>` (e.g. `sarahchen@everfield.com`)
3. `firstname@<domain>` (e.g. `sarah@everfield.com`)
4. `f.lastname@<domain>` (e.g. `s.chen@everfield.com`)
5. `firstname-lastname@<domain>` (e.g. `sarah-chen@everfield.com`)

Mark these as `confidence: 0.5–0.7` since they're guesses. Confidence
boosts to `0.8+` only when the email is literally visible on the page.

Do NOT include both `firstname.lastname@` AND `firstname@` in the result —
pick one (#1 by default unless the company website uses a different
pattern visibly).

## Confidence scoring

| Score | Means |
|---|---|
| 0.9–1.0 | Real name + email both visible on company website at priority-tier title |
| 0.7–0.9 | Real name + title at priority tier; email guessed via top-pattern |
| 0.5–0.7 | Title-only match (no name); email guessed via secondary pattern |
| 0.3–0.5 | Generic inbox (`careers@`, `hello@`) — last resort |
| <0.3 | Skip — caller should not draft cold email yet |

## Privacy + ethics guardrails

- Capture professional contact info only — no phone numbers, no home addresses, no personal email domains (`@gmail.com`, `@outlook.com`).
- Audit trail required: every saved `contact_email` MUST have `source_url` set so the operator can re-verify provenance later.
- If a company explicitly opts out of cold outreach (e.g. `/contact` page says "no recruitment emails please"), abort with `status: failed` and `error_message: "company opted out of outbound contact"`.
