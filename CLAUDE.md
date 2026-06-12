# Staff Domain SMYKM Campaign Pipeline

## Project

GitHub Actions pipeline that generates personalised 5-email SMYKM cold outreach sequences for Staff Domain's sales team. Triggered by Make.com → HubSpot list membership → GitHub Actions `workflow_dispatch`. Fetches HubSpot contact data + Chorus AI transcripts, calls Claude API, writes emails + SDR notes back to HubSpot.

**Core value:** Each prospect gets a hyper-personalised email sequence grounded in their specific job ad and company context — at scale, zero manual writing.

## File Structure

```
SMYKM/
├── scripts/
│   ├── fetch_hubspot.py         # HubSpot contact + engagement fetch
│   ├── fetch_chorus.py          # Chorus AI transcript fetch (silent fallback)
│   ├── compute_campaign_tokens.py  # Runtime campaign tokens (dates, context)
│   ├── generate_campaign.py     # Prompt assembly + Claude call + validation
│   └── write_hubspot.py         # HubSpot property write + note creation
├── prompt_template.md           # SMYKM prompt with {{token.name}} substitution
├── requirements.txt
└── CLAUDE.md

.github/workflows/
└── campaign.yml                 # GitHub Actions workflow
```

## Key Technical Decisions

- **Inter-step data:** `$RUNNER_TEMP` JSON files (not `$GITHUB_OUTPUT` — too small for structured payloads)
- **Single sequential job:** All 5 steps in one job; `$RUNNER_TEMP` is not accessible across separate jobs
- **Retry:** All API calls use shared tenacity decorator; Anthropic SDK `max_retries=0` (no double-retry)
- **Claude output:** Use `client.messages.parse()` with Pydantic schema (Structured Outputs, GA). Still check `stop_reason` first
- **Token budget:** `client.messages.count_tokens()` — not tiktoken (undercounts Claude by 30-50%). Hard 100K input limit
- **Prompt substitution:** Custom `re.sub(r'\{\{([a-zA-Z0-9_.]+)\}\}', ...)` regex — not f-strings (crash on `{`/`}` in prompt)
- **Chorus auth:** `Authorization: Token XXXXXXXX` (not Bearer)
- **HubSpot pagination:** Always loop on `paging.next.after` — silent truncation at 100 records otherwise

## Required Library Versions (requirements.txt)

```
hubspot-api-client>=12.0.0   # v12: object_id is string in v4 associations (was int)
requests>=2.34.0             # 2.32.0-2.32.1 yanked (CVE-2024-35195)
beautifulsoup4>=4.12.0
lxml>=5.2.0                  # Required — BS4 silently falls back without explicit lxml
anthropic>=0.50.0            # Stable prompt caching + structured outputs
tiktoken>=0.7.0              # Relative sizing only; count_tokens() for budget enforcement
tenacity>=9.0.0
```

## GitHub Actions

- `actions/upload-artifact@v4` (v3 was removed Dec 2024)
- Artifact names include `${{ github.run_attempt }}` suffix
- Concurrency limit: max 5 simultaneous runs
- `timeout-minutes: 10` on each step
- Failure path: `if: failure()` for DLQ artifact + Teams notification
- Campaign output: `if: always()` for 7-day artifact

## GitHub Secrets Required

- `HUBSPOT_API_KEY` — HubSpot Private App token (contacts + engagements read/write, owners read)
- `CHORUS_API_TOKEN` — Raw token value (header: `Authorization: Token {value}`)
- `ANTHROPIC_API_KEY` — Anthropic API key
- `TEAMS_WEBHOOK_URL` — Teams (or Slack) incoming webhook URL

## HubSpot Custom Properties

Must exist in HubSpot before first run (Settings > Properties > Contact):
- `job_title_posted`, `job_post_link`, `job_description` — inputs (Single-line text)
- `email_1` through `email_5` — Multi-line text (65K limit)
- `subject_1` through `subject_5` — Single-line text
- `smykm_sdr_notes` — Multi-line text
- `smykm_generated_date` — Date

## GSD Workflow

`.planning/` contains full project context. Follow this order:

```
/gsd-plan-phase 0   → Plan infrastructure
/gsd-execute-phase 0 → Execute
/gsd-verify-work 0  → Verify
/gsd-plan-phase 1   → Next phase
```

**Current state:** See `.planning/STATE.md`
**Requirements:** See `.planning/REQUIREMENTS.md` (49 v1 requirements)
**Roadmap:** See `.planning/ROADMAP.md` (6 phases)
**Research:** See `.planning/research/SUMMARY.md`

## Email Sequence (SMYKM)

| Email | Principle | Length |
|-------|-----------|--------|
| 1 | SMYKM cold open — hyper-specific hook + market reality + objection pre-empt | 130-180w |
| 2 | 48hr bump — no new pitch, one resource link | 70-110w |
| 3 | Peer proof — case study story + normalised objection | 130-170w |
| 4 | Commercial reframe — three specific meeting times (no Calendly) | 180-230w |
| 5 | Gracious exit — dignified close, door open | 110-150w |

## Known Pitfalls

- Chorus 404/401/timeout → must write `[]` and exit 0 (never fail the pipeline)
- HubSpot engagement list silently truncates at 100 → always paginate
- `stop_reason: max_tokens` or `refusal` → non-retryable, do not retry
- Context saturation at ~50% window fill → enforce 100K token hard cap
- Resource hallucination → validate Email 3 case study against known resource list
- `job_title_posted` missing → Email 1 hook degrades; add fallback logic
