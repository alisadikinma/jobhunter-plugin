# jobhunter-plugin

Claude Code plugin for the [JobHunter](https://github.com/alisadikinma/jobhunter) automated
job-hunting pipeline. Provides three skills that the FastAPI backend invokes
as subprocess (or that you can run interactively):

| Slash | Model | What it does |
|---|---|---|
| `/jobhunter:job-score` | Sonnet | Batch-score up to 25 unscored jobs against the master CV using the 3-variant rubric (`vibe_coding` / `ai_automation` / `ai_video`) |
| `/jobhunter:cv-tailor` | Opus | Generate a variant-specific tailored CV for one application — rewrites summary, ranks highlights, picks portfolio assets |
| `/jobhunter:cold-email` | Sonnet | Draft initial + 2 follow-up cold emails per application, personalised against company + variant + tailored CV |

## Install

### Via Claude Code marketplace

```
claude /plugin marketplace add github.com/alisadikinma/jobhunter-plugin
claude /plugin install jobhunter
```

### Local clone (for the JobHunter FastAPI subprocess spawner)

```bash
git clone https://github.com/alisadikinma/jobhunter-plugin.git
# Set CLAUDE_PLUGIN_PATH in your jobhunter .env to the absolute path of the clone:
#   CLAUDE_PLUGIN_PATH=/abs/path/to/jobhunter-plugin
```

Production Docker images can clone at build time:

```dockerfile
RUN git clone --branch main --depth 1 \
    https://github.com/alisadikinma/jobhunter-plugin.git \
    /app/claude-plugin
ENV CLAUDE_PLUGIN_PATH=/app/claude-plugin
```

## Callback contract

These skills are not standalone — they call back to a paired FastAPI backend
via `X-Callback-Secret`-protected HTTP routes. The shared shape:

| Step | Method | URL | Purpose |
|---|---|---|---|
| 1 | `GET` | `{api-url}/api/callbacks/context/{job-id}` | Fetch input data (jobs to score, master CV, application context) |
| 2 | `PUT` | `{api-url}/api/callbacks/progress/{job-id}` | Stream `{progress_pct, current_step, log_message}` |
| 3 | `PUT` | `{api-url}/api/callbacks/complete/{job-id}` | POST final `{status, result}` |

All calls send `X-Callback-Secret: <token>` instead of a user JWT — the subprocess
has no user session.

The backend defines the request/response schemas in
[`backend/app/schemas/callbacks.py`](https://github.com/alisadikinma/jobhunter/blob/main/backend/app/schemas/callbacks.py).
Result shape per skill is defined in each `skills/<name>/SKILL.md`.

## Spawn shape (FastAPI subprocess)

```python
subprocess.Popen([
    "claude",
    "--plugin-dir", CLAUDE_PLUGIN_PATH,
    "--print",
    "--output-format", "text",
    "--dangerously-skip-permissions",
    "--model", "claude-sonnet-4-6",  # or claude-opus-4-7 for cv-tailor
    f"/jobhunter:{skill_slug} --api-url {url} --api-token {secret} --job-id {id}",
])
```

Note: `--dangerously-skip-permissions` is required because there's no human
to approve curl/Bash tool calls inside the subprocess. `--allow-dangerously-skip-permissions`
only EXPOSES the option as available; it does not apply it.

## Repo layout

```
jobhunter-plugin/
├── .claude-plugin/
│   ├── plugin.json          ← plugin metadata (name, version, license)
│   └── marketplace.json     ← marketplace install metadata
├── skills/
│   ├── job-score/SKILL.md   ← /jobhunter:job-score
│   ├── cv-tailor/SKILL.md   ← /jobhunter:cv-tailor
│   └── cold-email/SKILL.md  ← /jobhunter:cold-email
├── refs/
│   ├── refs-scoring.md      ← 3-variant classifier rubric
│   ├── refs-cv.md           ← variant-preset bullet ranking + ATS rules
│   └── refs-email.md        ← cold email structure + follow-up cadence
├── CLAUDE.md                ← convention reference (shared args, callback shape, model picks)
├── README.md                ← this file
└── LICENSE
```

## Adding new skills

The plugin is the **AI / orchestration layer** of JobHunter. The FastAPI
backend stays minimal (scraping, storage, kanban, basic ATS); anything
that requires Claude reasoning lives here so it can be processed by the
VPS Claude CLI without backend redeploy.

To add a skill:

1. Create `skills/<slug>/SKILL.md` with YAML frontmatter (`name`, `description`).
2. Document the callback contract in the skill body — what `GET /context/{job_id}`
   should return, what `PUT /complete/{job_id}` body shape your skill emits.
3. If the FastAPI side needs new context endpoints or dispatch handlers,
   wire them in `backend/app/api/callbacks.py` and bump the plugin's
   `version` in `.claude-plugin/plugin.json`.
4. Tag a release (`git tag v0.X.0 && git push --tags`) — VPS pulls the
   matching tag during deploy.

Candidate skills for v0.2:

- `/jobhunter:run-pipeline` — orchestrate score → auto-target ≥85 → tailor → email in one shot. Intended for cron.
- `/jobhunter:enrich-jd` — Firecrawl-based JD enrichment for descriptions <500 chars.
- `/jobhunter:weekly-digest` — Monday 8am summary of the past week's pipeline.
- `/jobhunter:retarget-stale` — re-score jobs sitting unscored / unactioned >7 days.

## Versioning

Semantic versioning. Breaking changes to the callback contract bump major.
Skill rubric / prompt tweaks bump minor or patch.

## License

MIT — see [LICENSE](LICENSE).
