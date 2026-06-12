# Project State: Staff Domain SMYKM Campaign Pipeline

## Current Phase
Phase 0 — Infrastructure & Config

## Status
Not started

---

## Project Reference
See: .planning/PROJECT.md (updated 2026-06-12)

**Core value:** Each prospect receives a hyper-personalised 5-email SMYKM sequence that could only have been written for them — at scale, with zero manual writing effort from the SDR team.
**Current focus:** Phase 0 — Infrastructure & Config

---

## Phase Progress

| Phase | Name | Status | Completed |
|-------|------|--------|-----------|
| 0 | Infrastructure & Config | Pending | - |
| 1 | Skeleton Pipeline + Shared Utilities | Pending | - |
| 2 | Data Ingestion | Pending | - |
| 3 | Prompt Assembly & Claude Generation | Pending | - |
| 4 | HubSpot Write-Back | Pending | - |
| 5 | End-to-End Hardening | Pending | - |

Progress: 0/6 phases complete `[----------]`

---

## Performance Metrics

| Metric | Value |
|--------|-------|
| Phases total | 6 |
| Requirements total | 49 |
| Requirements complete | 0 |
| Plans complete | 0 |

---

## Accumulated Context

### Key Decisions
- GitHub Actions as pipeline runtime (zero infra overhead, native secrets, artifacts, audit trail)
- `$RUNNER_TEMP` for inter-step JSON (ephemeral, secure, uncapped size vs `$GITHUB_OUTPUT` ~1MB limit)
- tenacity for all retries with `max_retries=0` on Anthropic SDK (unified retry logic across all three APIs)
- HubSpot engagement note created fresh each run (preserves history; note failure is non-fatal)
- 5-email sequence in v1; 7 output slots reserved for future extension without schema change
- `client.messages.count_tokens()` for token budget enforcement (tiktoken undercounts Claude by 30-50%)
- Claude Structured Outputs (`client.messages.parse()` with Pydantic schema) for guaranteed JSON compliance

### Critical Pitfalls (from research)
- HubSpot pagination silently truncates at 100 — must loop on `paging.next.after` cursor
- Chorus auth header is `Authorization: Token XXXXXXXX` (not `Bearer`)
- HubSpot Private App requires 8 specific scopes — validate with startup-time test calls in Phase 0
- Claude JSON may be markdown-wrapped — three-layer defence: system prompt, `{` prefill, structured outputs
- `stop_reason` must be checked before parsing Claude output (catches truncation + refusal)
- `actions/upload-artifact@v4` required (v3 removed from GitHub.com)
- `requests>=2.34.0` required (2.32.x yanked, CVE-2024-35195)
- `anthropic>=0.50.0` required (structured outputs GA, stable prompt caching)

### Open Questions (to resolve during build)
- Chorus API endpoint paths and rate limits — need empirical first-run test
- HubSpot custom property names (`job_title_posted`, `job_post_link`, `job_description`, output properties) — verify exist in portal before Phase 4
- Email sequence length: confirm 5 emails consistent across Pydantic schema, template, and HubSpot properties
- HubSpot portal tier — determines rate limit (100 or 190 req/10s) and whether startup delay needed for concurrent runs
- Make.com poll interval — determines max concurrent run count in production

### Blockers
None

### TODOs
- [ ] Verify all 8 required HubSpot Private App scopes before Phase 0 completion
- [ ] Confirm email count (5 vs 7) and align Pydantic schema + prompt template + HubSpot properties before Phase 3

---

## Session Continuity

**Last session:** 2026-06-12 — project initialized, research complete, roadmap created
**Next action:** `/gsd-plan-phase 0` — plan Phase 0 (Infrastructure & Config)
**Resume note:** Start with HubSpot Private App scope verification and repo scaffold before wiring Make.com

---

*Last updated: 2026-06-12 after roadmap creation*
