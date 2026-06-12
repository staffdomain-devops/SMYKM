# Requirements: Staff Domain SMYKM Campaign Pipeline

**Defined:** 2026-06-12
**Core Value:** Each prospect receives a hyper-personalised 5-email SMYKM sequence that could only have been written for them — at scale, with zero manual writing effort from the SDR team.

---

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Infrastructure & Trigger

- [ ] **INFRA-01**: GitHub Actions workflow triggered by `workflow_dispatch` event with `contact_id` (string) and `contact_email` (string) inputs
- [ ] **INFRA-02**: Make.com scenario detects new HubSpot list membership and fires `workflow_dispatch` via GitHub API
- [ ] **INFRA-03**: All secrets (HUBSPOT_API_KEY, CHORUS_API_TOKEN, ANTHROPIC_API_KEY, TEAMS_WEBHOOK_URL) stored as GitHub repository secrets — never in code or logs
- [ ] **INFRA-04**: Workflow enforces concurrency limit (max 5 simultaneous runs) to prevent HubSpot/Chorus rate limit storms
- [ ] **INFRA-05**: Each workflow step has a `timeout-minutes: 10` guard to prevent hung runs
- [ ] **INFRA-06**: Workflow runs on `ubuntu-latest` with working directory `SMYKM/`

### HubSpot Data Fetching

- [ ] **FETCH-01**: `fetch_hubspot.py` fetches contact properties: firstname, lastname, email, jobtitle, company, industry, num_employees, city, country, website, hubspot_owner_id, job_title_posted, job_post_link, job_description
- [ ] **FETCH-02**: Fetches all email engagements from the past 12 months with full pagination loop (cursor-based, no silent truncation at 100)
- [ ] **FETCH-03**: Fetches all meeting engagements from the past 12 months including notes, internal notes, and attendees
- [ ] **FETCH-04**: Fetches CRM meeting objects via v4 associations API (scheduler-created meetings not in legacy engagement API)
- [ ] **FETCH-05**: Extracts Chorus conversation IDs from meeting notes via regex (`chorus.ai/meeting/XXXXXXXX`)
- [ ] **FETCH-06**: Resolves owner first name from owner ID via HubSpot owners API
- [ ] **FETCH-07**: Writes `hubspot_contact.json` to `$RUNNER_TEMP` with all fetched data

### Chorus AI Data Fetching

- [ ] **CHORUS-01**: `fetch_chorus.py` fetches call transcripts using `Authorization: Token XXXXXXXX` format
- [ ] **CHORUS-02**: Sources conversation IDs from: (a) HubSpot meeting notes regex, (b) `INPUT_CHORUS_IDS` env override, (c) Chorus v3/engagements search by company name
- [ ] **CHORUS-03**: Silent fallback to `[]` on any Chorus failure (404, 401, timeout, API error) — pipeline must never fail due to Chorus
- [ ] **CHORUS-04**: Writes `chorus_transcripts.json` to `$RUNNER_TEMP` (may be empty array)

### Campaign Token Computation

- [ ] **TOKEN-01**: `compute_campaign_tokens.py` computes runtime campaign tokens: `current_date`, and any timing context tokens needed by the prompt template
- [ ] **TOKEN-02**: Writes `campaign_tokens.json` to `$RUNNER_TEMP`

### Prompt Assembly & Claude Generation

- [ ] **GEN-01**: `generate_campaign.py` assembles `activity_history` string from emails + meetings + transcripts as labelled blocks
- [ ] **GEN-02**: Substitutes `{{token.name}}` placeholders in `prompt_template.md` using regex — raises error if any placeholder remains unsubstituted after assembly
- [ ] **GEN-03**: Enforces input token budget of 100K tokens using `client.messages.count_tokens()` (not tiktoken); gracefully truncates oldest content first when over budget
- [ ] **GEN-04**: Calls `claude-sonnet-4-6` with `max_tokens=16384`, `max_retries=0` (tenacity handles retry)
- [ ] **GEN-05**: Uses Claude Structured Outputs (`client.messages.parse()` with Pydantic schema) to guarantee JSON compliance
- [ ] **GEN-06**: Checks `stop_reason` before accessing parsed output — handles `max_tokens` (truncated) and `refusal` as non-retryable failures
- [ ] **GEN-07**: Validates all required keys present in output: `email_1` through `email_5`, `sdr_call_notes`, `reasoning`
- [ ] **GEN-08**: Strips banned punctuation from all email bodies (em dash `—`, en dash `–`)
- [ ] **GEN-09**: Detects any banned phrases from the SMYKM prompt in output — logs WARNING but does not fail pipeline
- [ ] **GEN-10**: Validates all case study names and URLs in Email 3 against known resource list — logs WARNING on mismatch
- [ ] **GEN-11**: Writes `campaign_output.json` to `$RUNNER_TEMP`

### HubSpot Write-Back

- [ ] **WRITE-01**: `write_hubspot.py` writes all 5 email subjects and bodies to HubSpot contact properties with `value[:65000]` truncation
- [ ] **WRITE-02**: Writes SDR call notes to HubSpot contact property (`smykm_sdr_notes`)
- [ ] **WRITE-03**: Writes campaign generation date to HubSpot contact property (`smykm_generated_date`)
- [ ] **WRITE-04**: Creates a new HubSpot engagement note on the contact with campaign brief + SDR notes as HTML body
- [ ] **WRITE-05**: Note creation failure is non-fatal — logs warning and continues

### Retry & Error Handling

- [ ] **ERR-01**: All API calls (HubSpot, Chorus, Anthropic) wrapped in shared tenacity decorator: exponential backoff with jitter, max 6 attempts, max 60s total, retry on 429 and 5xx only
- [ ] **ERR-02**: `Retry-After` header honored when present on 429 responses
- [ ] **ERR-03**: On unrecovered failure, each script writes DLQ record to `$RUNNER_TEMP/failed_contacts.json` before re-raising
- [ ] **ERR-04**: DLQ record contains: `contact_id`, `contact_email`, `failed_step`, `error_message` (max 2000 chars), `timestamp`, `retry_count`
- [ ] **ERR-05**: Workflow failure path copies DLQ file from `$RUNNER_TEMP` and uploads as GitHub Actions artifact (30-day retention) via `actions/upload-artifact@v4`
- [ ] **ERR-06**: Workflow failure path sends POST notification to Teams/Slack webhook with: contact_email, failed_step, error excerpt, run log URL

### Artifacts & Observability

- [ ] **OBS-01**: Campaign output uploaded as GitHub Actions artifact (7-day retention) on `if: always()`
- [ ] **OBS-02**: All scripts emit structured log output at entry/exit, each API call, each retry, each fallback
- [ ] **OBS-03**: Artifact names include `${{ github.run_attempt }}` suffix to support re-runs without collision

### Prompt Template (SMYKM Content)

- [ ] **PROMPT-01**: `prompt_template.md` implements full SMYKM methodology: SMYKM hook, Show Up to Solve, Better Than the Weather, Forthcoming Objection, Stewardship CTA, Vendor vs. Partner filters
- [ ] **PROMPT-02**: Prompt enforces Australian English, no em/en dashes, no banned phrases, voice guidelines
- [ ] **PROMPT-03**: Prompt includes full resource library: 13 case studies, 11 YouTube videos, 8 website resources with selection priority rules
- [ ] **PROMPT-04**: Prompt instructs Claude on all 5 email structures with length constraints (Email 1: 130-180w, Email 2: 70-110w, Email 3: 130-170w, Email 4: 180-230w, Email 5: 110-150w)
- [ ] **PROMPT-05**: Prompt requires `reasoning` block in output capturing: company_summary, job_description_insights, buyer_frame, forthcoming_objection, smykm_hook_strategy, resources_selected

---

## v2 Requirements

Deferred to future release.

### Enhanced Personalisation

- **ENRICH-01**: LinkedIn profile data enrichment (company news, recent posts)
- **ENRICH-02**: Deal stage-aware prompt branching (different tone for re-engaged vs cold contacts)
- **ENRICH-03**: Previous campaign detection — skip Email 1 if contact already received a sequence

### Observability & Quality

- **QUAL-01**: Banned phrase hit rate dashboard (alert if >5% across weekly runs)
- **QUAL-02**: Prompt version tracking — tag each HubSpot note with the prompt template git SHA
- **QUAL-03**: Email open/reply rate feedback loop from HubSpot back to prompt iteration
- **QUAL-04**: A/B testing framework for prompt variants

### Operations

- **OPS-01**: Self-service re-trigger UI (HubSpot workflow button → Make.com → GitHub)
- **OPS-02**: Multi-campaign support (parameterised campaign type in workflow_dispatch)

---

## Out of Scope

| Feature | Reason |
|---------|--------|
| Direct email sending from pipeline | Email delivery stays in HubSpot; compliance (AU Spam Act 2003) requires human send control |
| Batch multi-contact processing per run | One run per contact keeps failure isolation clean and retry simple |
| Web UI for campaign management | GitHub Actions is the operational interface for v1 |
| A/B testing | Adds complexity without clear v1 ROI — deferred to v2 |
| Real-time webhooks (sub-minute trigger) | Make.com polling cadence is sufficient for sales outreach volume |
| CRM platforms other than HubSpot | Not needed for current Staff Domain stack |
| Self-healing DLQ (auto-retry) | Manual re-trigger via Make.com or GitHub UI is sufficient |
| Web scraping for additional prospect enrichment | Legal risk (AU Privacy Act), not needed given job posting data quality |

---

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| INFRA-01 | Phase 0 | Pending |
| INFRA-02 | Phase 0 | Pending |
| INFRA-03 | Phase 0 | Pending |
| INFRA-04 | Phase 0 | Pending |
| INFRA-05 | Phase 0 | Pending |
| INFRA-06 | Phase 0 | Pending |
| FETCH-01 | Phase 2 | Pending |
| FETCH-02 | Phase 2 | Pending |
| FETCH-03 | Phase 2 | Pending |
| FETCH-04 | Phase 2 | Pending |
| FETCH-05 | Phase 2 | Pending |
| FETCH-06 | Phase 2 | Pending |
| FETCH-07 | Phase 2 | Pending |
| CHORUS-01 | Phase 2 | Pending |
| CHORUS-02 | Phase 2 | Pending |
| CHORUS-03 | Phase 2 | Pending |
| CHORUS-04 | Phase 2 | Pending |
| TOKEN-01 | Phase 2 | Pending |
| TOKEN-02 | Phase 2 | Pending |
| GEN-01 | Phase 3 | Pending |
| GEN-02 | Phase 3 | Pending |
| GEN-03 | Phase 3 | Pending |
| GEN-04 | Phase 3 | Pending |
| GEN-05 | Phase 3 | Pending |
| GEN-06 | Phase 3 | Pending |
| GEN-07 | Phase 3 | Pending |
| GEN-08 | Phase 3 | Pending |
| GEN-09 | Phase 3 | Pending |
| GEN-10 | Phase 3 | Pending |
| GEN-11 | Phase 3 | Pending |
| WRITE-01 | Phase 4 | Pending |
| WRITE-02 | Phase 4 | Pending |
| WRITE-03 | Phase 4 | Pending |
| WRITE-04 | Phase 4 | Pending |
| WRITE-05 | Phase 4 | Pending |
| ERR-01 | Phase 1 | Pending |
| ERR-02 | Phase 1 | Pending |
| ERR-03 | Phase 1 | Pending |
| ERR-04 | Phase 1 | Pending |
| ERR-05 | Phase 1 | Pending |
| ERR-06 | Phase 1 | Pending |
| OBS-01 | Phase 5 | Pending |
| OBS-02 | Phase 1 | Pending |
| OBS-03 | Phase 5 | Pending |
| PROMPT-01 | Phase 3 | Pending |
| PROMPT-02 | Phase 3 | Pending |
| PROMPT-03 | Phase 3 | Pending |
| PROMPT-04 | Phase 3 | Pending |
| PROMPT-05 | Phase 3 | Pending |

**Coverage:**
- v1 requirements: 49 total
- Mapped to phases: 49
- Unmapped: 0 ✓

---
*Requirements defined: 2026-06-12*
*Last updated: 2026-06-12 after initial definition*
