# Roadmap: Staff Domain SMYKM Campaign Pipeline

## Overview

6 phases | 49 requirements | Deliver a GitHub Actions pipeline that generates and writes hyper-personalised 5-email SMYKM sequences to HubSpot — triggered by Make.com, zero manual SDR writing effort.

---

## Phases

- [ ] **Phase 0: Infrastructure & Config** — Repo structure, secrets, GitHub Actions workflow skeleton, Make.com trigger wired
- [ ] **Phase 1: Skeleton Pipeline + Shared Utilities** — Stub scripts run end-to-end through the workflow; retry/error/DLQ/logging utilities in place
- [ ] **Phase 2: Data Ingestion** — HubSpot contact data, Chorus transcripts, and campaign tokens fetched and written to `$RUNNER_TEMP`
- [ ] **Phase 3: Prompt Assembly & Claude Generation** — Prompt template assembled, Claude called, structured output validated and written to `$RUNNER_TEMP`
- [ ] **Phase 4: HubSpot Write-Back** — All 5 email subjects/bodies, SDR notes, generation date, and engagement note written to HubSpot
- [ ] **Phase 5: End-to-End Hardening** — Artifact uploads, failure notification, concurrent-run testing, and full pipeline observability confirmed

---

## Phase Details

### Phase 0: Infrastructure & Config
**Goal**: The GitHub Actions workflow exists, all secrets are loaded, Make.com can fire a run, and a test `workflow_dispatch` reaches the runner — no script logic yet, but the entire trigger-to-runner path is verified.
**Depends on**: Nothing (first phase)
**Requirements**: INFRA-01, INFRA-02, INFRA-03, INFRA-04, INFRA-05, INFRA-06
**Success Criteria** (what must be TRUE):
  1. A manual `workflow_dispatch` from the GitHub Actions UI reaches an `ubuntu-latest` runner and the run appears in the Actions tab with status "Success"
  2. Make.com fires `workflow_dispatch` with a real `contact_id` and `contact_email` and the run appears in GitHub Actions within the expected polling interval
  3. All four secrets (HUBSPOT_API_KEY, CHORUS_API_TOKEN, ANTHROPIC_API_KEY, TEAMS_WEBHOOK_URL) are accessible inside the run via `${{ secrets.X }}` and no secret value is visible in any step log
  4. A sixth concurrent trigger is queued (not failed) when five runs are already active, confirming the concurrency group cap is enforced
  5. Any step that runs for more than 10 minutes is automatically cancelled by the timeout guard
**Plans**: TBD

### Phase 1: Skeleton Pipeline + Shared Utilities
**Goal**: All five Python scripts exist as stubs that pass data through the workflow via `$RUNNER_TEMP` JSON files; the shared retry decorator, DLQ writer, and structured logger are implemented and importable from every script.
**Depends on**: Phase 0
**Requirements**: ERR-01, ERR-02, ERR-03, ERR-04, ERR-05, ERR-06, OBS-02
**Success Criteria** (what must be TRUE):
  1. A full end-to-end workflow run completes with all five stub scripts executing in sequence, each reading its input JSON and writing its output JSON to `$RUNNER_TEMP`
  2. A deliberately injected failure in any stub script causes the DLQ file (`failed_contacts.json`) to appear in `$RUNNER_TEMP` with all six required fields populated before the step exits
  3. Any API call wrapped in the shared tenacity decorator retries on 429/5xx up to 6 times with observable backoff delay, then re-raises — confirmed via unit test with a mock HTTP server
  4. All script log output includes structured entry/exit markers, API call events, retry events, and fallback events — visible in the GitHub Actions step log
**Plans**: TBD

### Phase 2: Data Ingestion
**Goal**: `fetch_hubspot.py`, `fetch_chorus.py`, and `compute_campaign_tokens.py` all run against real APIs, producing validated JSON files in `$RUNNER_TEMP` that downstream steps can consume.
**Depends on**: Phase 1
**Requirements**: FETCH-01, FETCH-02, FETCH-03, FETCH-04, FETCH-05, FETCH-06, FETCH-07, CHORUS-01, CHORUS-02, CHORUS-03, CHORUS-04, TOKEN-01, TOKEN-02
**Success Criteria** (what must be TRUE):
  1. `hubspot_contact.json` contains all 14 required contact properties (nil-safe), owner first name resolved from owner ID, and full engagement history (emails + meetings) for the past 12 months with no silent truncation at 100 records
  2. Chorus conversation IDs extracted via regex from HubSpot meeting notes appear in `chorus_transcripts.json`; when no IDs are found and the API returns 404/401/timeout, the file contains an empty array and the workflow step exits with code 0
  3. `campaign_tokens.json` exists and contains at minimum `current_date` in the expected format
  4. The full data ingestion phase completes for a real contact within the 10-minute per-step timeout
**Plans**: TBD

### Phase 3: Prompt Assembly & Claude Generation
**Goal**: `generate_campaign.py` assembles the full prompt from `prompt_template.md` and fetched data, calls Claude, and writes a validated `campaign_output.json` containing all required keys — with the complete SMYKM methodology, voice, and quality gates enforced.
**Depends on**: Phase 2
**Requirements**: GEN-01, GEN-02, GEN-03, GEN-04, GEN-05, GEN-06, GEN-07, GEN-08, GEN-09, GEN-10, GEN-11, PROMPT-01, PROMPT-02, PROMPT-03, PROMPT-04, PROMPT-05
**Success Criteria** (what must be TRUE):
  1. `campaign_output.json` contains all required keys (`email_1`–`email_5`, `sdr_call_notes`, `reasoning`) and no `{{` placeholder tokens remain in any field
  2. Email bodies contain no em dashes or en dashes; any banned phrase from the SMYKM list triggers a WARNING log entry but does not fail the pipeline
  3. Email 3's case study name and URL are present in the known resource list; a mismatch produces a WARNING log entry
  4. A prompt that would exceed 100K input tokens is gracefully truncated (oldest activity history first) and the run still completes successfully without a token-limit API error
  5. When Claude returns `stop_reason: max_tokens` or `stop_reason: refusal`, the pipeline fails with a clear non-retryable error rather than attempting to parse incomplete output
**Plans**: TBD
**UI hint**: no

### Phase 4: HubSpot Write-Back
**Goal**: `write_hubspot.py` writes all campaign output to HubSpot — contact properties updated and an engagement note created — so the SDR sees the full 5-email sequence and call notes on the contact record immediately after the workflow run.
**Depends on**: Phase 3
**Requirements**: WRITE-01, WRITE-02, WRITE-03, WRITE-04, WRITE-05
**Success Criteria** (what must be TRUE):
  1. After a successful run, all 5 email subjects and bodies are visible on the HubSpot contact record in the correct custom properties (`email_1`–`email_5`, `subject_1`–`subject_5`), truncated to 65,000 characters if needed
  2. `smykm_sdr_notes` and `smykm_generated_date` are populated on the contact record with correct values
  3. A new engagement note appears in the contact's HubSpot activity timeline containing the campaign brief and SDR notes as an HTML body
  4. If the note creation API call fails (any error), the workflow step exits with code 0 and logs a warning — no email or SDR notes write is rolled back
**Plans**: TBD

### Phase 5: End-to-End Hardening
**Goal**: The complete pipeline runs reliably under production conditions — artifacts are uploaded, failure alerts are sent, and 10 concurrent runs complete without rate-limit storms or DLQ collisions.
**Depends on**: Phase 4
**Requirements**: OBS-01, OBS-03
**Success Criteria** (what must be TRUE):
  1. After every run (success or failure), a campaign output artifact named with the `run_attempt` suffix is available in the GitHub Actions UI for at least 7 days
  2. When any step fails, a Teams/Slack webhook notification is received containing the contact email, failed step name, error excerpt, and a direct link to the run log — without requiring any manual action
  3. Re-running a failed job produces a new artifact with an incremented `run_attempt` suffix rather than overwriting the previous artifact
  4. Ten simultaneous runs against distinct contacts all complete (or fail gracefully into DLQ) without any run being cancelled due to rate limit errors propagating across runs
**Plans**: TBD

---

## Progress

| Phase | Name | Plans Complete | Status | Completed |
|-------|------|----------------|--------|-----------|
| 0 | Infrastructure & Config | 0/? | Not started | - |
| 1 | Skeleton Pipeline + Shared Utilities | 0/? | Not started | - |
| 2 | Data Ingestion | 0/? | Not started | - |
| 3 | Prompt Assembly & Claude Generation | 0/? | Not started | - |
| 4 | HubSpot Write-Back | 0/? | Not started | - |
| 5 | End-to-End Hardening | 0/? | Not started | - |

---

## Requirement Coverage

| Requirement | Phase |
|-------------|-------|
| INFRA-01 | Phase 0 |
| INFRA-02 | Phase 0 |
| INFRA-03 | Phase 0 |
| INFRA-04 | Phase 0 |
| INFRA-05 | Phase 0 |
| INFRA-06 | Phase 0 |
| ERR-01 | Phase 1 |
| ERR-02 | Phase 1 |
| ERR-03 | Phase 1 |
| ERR-04 | Phase 1 |
| ERR-05 | Phase 1 |
| ERR-06 | Phase 1 |
| OBS-02 | Phase 1 |
| FETCH-01 | Phase 2 |
| FETCH-02 | Phase 2 |
| FETCH-03 | Phase 2 |
| FETCH-04 | Phase 2 |
| FETCH-05 | Phase 2 |
| FETCH-06 | Phase 2 |
| FETCH-07 | Phase 2 |
| CHORUS-01 | Phase 2 |
| CHORUS-02 | Phase 2 |
| CHORUS-03 | Phase 2 |
| CHORUS-04 | Phase 2 |
| TOKEN-01 | Phase 2 |
| TOKEN-02 | Phase 2 |
| GEN-01 | Phase 3 |
| GEN-02 | Phase 3 |
| GEN-03 | Phase 3 |
| GEN-04 | Phase 3 |
| GEN-05 | Phase 3 |
| GEN-06 | Phase 3 |
| GEN-07 | Phase 3 |
| GEN-08 | Phase 3 |
| GEN-09 | Phase 3 |
| GEN-10 | Phase 3 |
| GEN-11 | Phase 3 |
| PROMPT-01 | Phase 3 |
| PROMPT-02 | Phase 3 |
| PROMPT-03 | Phase 3 |
| PROMPT-04 | Phase 3 |
| PROMPT-05 | Phase 3 |
| WRITE-01 | Phase 4 |
| WRITE-02 | Phase 4 |
| WRITE-03 | Phase 4 |
| WRITE-04 | Phase 4 |
| WRITE-05 | Phase 4 |
| OBS-01 | Phase 5 |
| OBS-03 | Phase 5 |

**Coverage: 49/49 v1 requirements mapped**

---

*Created: 2026-06-12*
