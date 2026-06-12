# Staff Domain SMYKM Campaign Pipeline

## What This Is

A GitHub Actions pipeline that generates personalised AI-powered cold email sequences for Staff Domain's sales outreach. When a contact is added to a HubSpot list in Make.com, the pipeline pulls the contact's HubSpot data and Chorus AI call transcripts, feeds them to Claude, and generates a 5-email SMYKM (Show Me You Know Me) sequence plus SDR call notes — then writes everything back to HubSpot contact properties.

## Core Value

Each prospect receives a hyper-personalised 5-email sequence that could only have been written for them, grounded in their specific job ad and company context — at scale, with zero manual writing effort from the SDR team.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Make.com triggers GitHub Actions workflow_dispatch when contact added to HubSpot list, passing contact_id and contact_email
- [ ] Pipeline fetches full HubSpot contact data (properties, email history, meetings, deals, owner names)
- [ ] Pipeline fetches Chorus AI call transcripts linked to the contact
- [ ] Claude generates a 5-email SMYKM sequence + SDR call notes from contact + transcript context
- [ ] Generated emails and SDR notes written back to HubSpot contact properties
- [ ] HubSpot note (engagement) created on contact record after each run
- [ ] Exponential backoff retry on 429/5xx for all API calls (HubSpot, Chorus, Anthropic)
- [ ] On failure: DLQ record written, artifact uploaded, Teams/Slack webhook notified
- [ ] Campaign output uploaded as GitHub Actions artifact (7-day retention)
- [ ] Prompt template uses token substitution ({{token.name}}) for all variable content
- [ ] Prompt enforces SMYKM methodology, Australian English, voice guidelines, and banned phrases
- [ ] 5-email sequence follows arc: Cold Open → 48hr Bump → Peer Proof → Commercial Reframe → Gracious Exit
- [ ] Resource selection logic: case study (Email 3), YouTube video (Email 4), website resources (Emails 2 & 5)
- [ ] JSON output validated for required keys before writing to HubSpot

### Out of Scope

- Real-time processing (batch/async via GitHub Actions is sufficient)
- Multi-language support — Australian English only for this campaign
- A/B testing framework — single best sequence per contact
- CRM other than HubSpot — not needed for current stack
- Web UI for campaign management — GitHub Actions is the interface

## Context

**Architecture:**
- Trigger: Make.com monitors HubSpot list → fires `workflow_dispatch` to GitHub Actions
- Workflow: `ubuntu-latest`, working directory `SMYKM/`, secrets via GitHub Secrets
- Data flow: temp files in `$RUNNER_TEMP` (JSON) between steps
- 5 Python scripts in `scripts/` + `prompt_template.md` + GitHub Actions YAML

**Tech stack:**
- Python 3.12
- `hubspot-api-client>=12.0.0`, `requests>=2.31.0`, `beautifulsoup4>=4.12.0`
- `anthropic>=0.30.0` (claude-sonnet-4-6), `tiktoken>=0.7.0`, `tenacity>=9.0.0`
- GitHub Actions, Make.com, HubSpot Private App, Chorus AI, Anthropic API

**HubSpot properties used:**
- Contact inputs: firstname, lastname, email, jobtitle, company, industry, num_employees, city, country, website, hubspot_owner_id, job_title_posted, job_post_link, job_description
- Campaign outputs: email_1–email_7 (multi-line text), subject_1–subject_7 (single-line text), smykm_sdr_notes (multi-line text), smykm_generated_date (date)

**Prompt methodology:**
- SMYKM (Show Me You Know Me) by Sam McKenna
- Hyper-specificity filter, Show Up to Solve filter, Better Than the Weather opener
- Forthcoming Objection pre-emption, Stewardship CTA, Vendor vs. Partner tone
- 13 case studies, 11 YouTube videos, 8 website resources with selection priority rules

**Existing reference documents:**
- `rebuild-prompt.md` — architecture blueprint and build instructions
- `staff_domain_SMYKM_prompt_v2.md` — complete prompt template with SMYKM methodology

## Constraints

- **API auth**: HubSpot Private App token (contacts + engagements read/write, owners read), Chorus Token-based auth, Anthropic API key, Teams webhook — all in GitHub Secrets
- **GitHub Actions**: Runs on ubuntu-latest; data passes between steps via `$RUNNER_TEMP` JSON files
- **Prompt output**: Claude must return raw JSON only — no markdown, no code fences; validated before HubSpot write
- **HubSpot limits**: Property values up to 65,000 chars for multi-line text; note body as HTML engagement
- **Chorus fallback**: Silent fallback on 404/401/timeout — pipeline must continue with empty transcripts
- **Token window**: Using claude-sonnet-4-6 with max_tokens=16384; prompt built with tiktoken awareness

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| GitHub Actions as pipeline runtime | Zero infra overhead, native secrets, artifact storage, audit trail | — Pending |
| `$RUNNER_TEMP` for inter-step data | Ephemeral, secure, no cleanup required after run | — Pending |
| tenacity for retry (not SDK built-in) | Anthropic SDK built-in retry disabled (max_retries=0) for unified retry logic | — Pending |
| HubSpot note created new each run | Preserves history; note failure is non-fatal | — Pending |
| 5-email sequence (not 7) | SMYKM methodology; 7 output slots allow future extension without schema change | — Pending |

---
## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-06-12 after initialization*
