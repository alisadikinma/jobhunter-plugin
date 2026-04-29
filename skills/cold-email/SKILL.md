---
name: cold-email
description: Draft an initial cold email plus two follow-up variants for an application, personalised against the company + variant + tailored CV context. ALSO push the initial draft into Gmail via the Gmail MCP server (mcp__claude_ai_Gmail__create_draft) so the operator can review + attach CV + send from the Gmail UI directly. Returns subject + body per email + Gmail draft IDs. Calls back to FastAPI via X-Callback-Secret. Invoked by the FastAPI subprocess spawner with --api-url, --api-token, --job-id.
---

# `/cold-email` — draft 3-email sequence per application

**Model:** Sonnet
**Reference:** `refs/refs-email.md`

Produces one initial email + 2 follow-ups for a given application,
tuned to the application's matched variant.

## Arguments

Shared from `CLAUDE.md`: `--api-url`, `--api-token`, `--job-id`.
No additional flags — the `reference_id` on the agent_job is the
`application_id`.

## Process

1. **Fetch context** — `GET /api/callbacks/context/{job-id}`. Expected:
   ```json
   {
     "job_type": "cold_email",
     "reference_id": 42,
     "payload": {
       "application_id": 42,
       "job": {...},
       "company": {..., "enriched_context": {"markdown": "..."}},
       "master_cv_summary": {...basics...},
       "variant_used": "vibe_coding"
     }
   }
   ```

2. **Trigger enrichment if missing** — if `company.enriched_context` is
   null, call `POST /api/enrichment/company/{company_id}` first so the
   personalized opener has real material to draw from. (The enrichment
   endpoint uses the user JWT, not the callback secret — the skill's
   `--api-token` works for callbacks only; enrichment must be
   pre-triggered by the caller.)

3. **Apply variant tone** from `refs/refs-email.md`. If `variant_used`
   is null or "no_match", fall back to the `vibe_coding` tone
   (priority order).

4. **Draft all 3 emails** obeying the 5-part structure:
   - Initial: 75-150 words
   - Follow-up 1 (Day 5): 40-60 words, one new data point
   - Follow-up 2 (Day 10): 30-50 words, flip to new idea/resource

5. **Stream progress** after each draft.

6. **Push the initial draft into Gmail** via the Gmail MCP server.
   Skip this step if `application.contact_email` is null/missing — log
   `gmail_skipped: "no recipient (run /jobhunter:find-contact first)"`
   in the result and continue with backend persistence only.

   ```
   tool: mcp__claude_ai_Gmail__create_draft
   args: {
     to: [{"email": "<application.contact_email>", "name": "<application.contact_name>"}],
     subject: "<initial.subject>",
     body: "<initial.body>",
     # Optional CV attachment if you have the cv_id (the FastAPI
     # backend's /api/cv/{id}/download/docx URL works in MCP if the
     # Gmail server fetches via authenticated GET; if not, leave
     # attachment to the operator).
   }
   ```

   Capture the returned `draft.id` and the `https://mail.google.com/mail/u/0/#drafts/<id>`
   URL so the operator can jump straight to the Gmail UI.

   **Do NOT auto-send.** This skill always leaves a draft. The operator
   reviews, attaches CV (if not auto-attached), and clicks Send manually.
   Auto-send is out of scope by design — see `refs/refs-email.md` §
   "no auto-send" for rationale.

   If Gmail MCP isn't available in the runtime (e.g. running outside
   Claude Code with a stripped MCP config), record
   `gmail_drafts: []` and `gmail_status: "mcp_unavailable"` and continue
   normally — backend dispatch still saves the draft to `email_drafts`.

7. **Post completion**:
   ```
   PUT {api-url}/api/callbacks/complete/{job-id}
   Header: X-Callback-Secret: {api-token}
   Body: {
     "status": "completed",
     "result": {
       "initial": {"subject": "...", "body": "..."},
       "follow_up_1": {"subject": "...", "body": "..."},
       "follow_up_2": {"subject": "...", "body": "..."},
       "strategy": "vibe_coding / velocity-first",
       "personalization_notes": [...],
       "gmail_drafts": [
         {"email_type": "initial", "draft_id": "<gmail-id>", "url": "https://mail.google.com/..."}
       ],
       "gmail_status": "drafted" | "mcp_unavailable" | "no_recipient" | "failed"
     }
   }
   ```

## Failure modes

- Missing company enrichment → emails will feel generic. Skill may
  still succeed with a weaker opener; note `strategy: "fallback/no_enrichment"`.
- JD too short (<200 chars) after enrichment attempt → fail with
  `error_message: "insufficient JD content"`.
- Master CV summary_variants[variant] missing → fail; Phase 8 seed
  didn't complete properly.
- Gmail MCP `create_draft` rate-limited or auth-broken → record
  `gmail_status: "failed"` with the error in `personalization_notes`
  but DO complete the callback successfully — the body+subject still
  flow into `email_drafts` table and the operator can paste manually.
