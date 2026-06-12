# Research Summary: SMYKM Pipeline

**Synthesized:** 2026-06-12

---

## Recommended Stack

| Library | Pinned Version | Change from Original | Rationale |
|---------|---------------|----------------------|-----------|
| `hubspot-api-client` | `>=12.0.0` | No change | v12 breaks association `object_id` to string — audit all ID parameters |
| `requests` | `>=2.34.0` | **Bump** | 2.32.0–2.32.1 yanked (CVE-2024-35195); `>=2.34.0` is safe floor |
| `beautifulsoup4` | `>=4.12.0` | No change | Pair with `lxml` explicitly |
| `lxml` | `>=5.2.0` | **New — add** | Required BS4 parser backend; without it BS4 silently falls back to `html.parser` |
| `anthropic` | `>=0.50.0` | **Bump from `>=0.30.0`** | Stable prompt caching without beta headers; structured outputs GA |
| `tiktoken` | `>=0.7.0` | No change | Use for relative sizing only; use `client.messages.count_tokens()` for budget enforcement |
| `tenacity` | `>=9.0.0` | No change | Confirmed correct |
| Python | `3.12` | No change | All libraries have 3.12 wheels |
| `actions/upload-artifact` | `v4` | **Required** | v3 deprecated Dec 2024 and removed from GitHub.com |

**Model:** `claude-sonnet-4-6` (1M context, 64K max output)

**Critical:** Disable Anthropic SDK built-in retry (`max_retries=0`). Use tenacity for all three APIs uniformly.

---

## Table Stakes Features

- Full HubSpot contact property fetch (nil-safe) + engagement history with pagination loop
- HubSpot owner name resolution
- Chorus transcript fetch — silent fallback to `[]` on ANY failure (404, 401, timeout)
- Exponential backoff with jitter on 429/5xx; Retry-After header honoring; hard fail on other 4xx
- DLQ record written to `$RUNNER_TEMP/failed_contacts.json` before any re-raise
- `stop_reason` check before `json.loads()` — catches truncation and refusal
- Required key presence check on Claude output before HubSpot write
- `value[:65000]` truncation guard before every HubSpot property write
- Teams/Slack failure notification with contact_email, failed_step, error excerpt, run URL
- Campaign output artifact upload (7-day), DLQ artifact upload (30-day, on failure)
- Literal placeholder detection — fail if `{{` appears in prompt after token substitution

---

## Key Architecture Decisions

1. **Five scripts, one external API boundary each.** Scripts do not import from each other; workflow YAML is the only coordinator.
2. **`$RUNNER_TEMP` JSON for inter-step data.** `$GITHUB_OUTPUT` is capped at ~1MB — `$RUNNER_TEMP` has no documented limit.
3. **Single sequential job.** `compute_campaign_tokens.py` runs in <1s — parallelising adds 20-30s overhead and loses `$RUNNER_TEMP` access.
4. **Claude Structured Outputs (GA).** Use `client.messages.parse()` with Pydantic schema. Still check `stop_reason` first.
5. **`{{token.name}}` custom regex substitution.** f-strings crash on `{`/`}` in prompt text; Jinja2 is overkill and a security risk.
6. **Token budget via `client.messages.count_tokens()`, not tiktoken.** tiktoken undercounts Claude tokens by 30-50%.
7. **Make.com sends only `contact_id` (string) and `contact_email`.** All data fetched inside workflow. Fine-grained PAT scoped to single repo.

---

## Critical Pitfalls to Avoid

1. **HubSpot pagination silently truncates at 100.** Loop on `paging.next.after` cursor. v4 associations has a bug where `paging` may be absent even when paginated — always check key existence.
2. **Chorus auth is `Token`, not `Bearer`.** `Authorization: Token XXXXXXXX`. Verify before first run.
3. **tiktoken undercounts Claude tokens by 30-50%.** Replace budget enforcement with `client.messages.count_tokens()`.
4. **Claude JSON wrapped in markdown; `stop_reason` not checked.** Three-layer defence: system prompt, `{` prefill, structured outputs. Always check `stop_reason` before parse. Do not retry refusals.
5. **Context saturation degrades email quality.** Hard 100K input token budget. Fixed prompt template may be 15K+ tokens before contact data. Monitor banned phrase hit rate (>5% signals saturation).
6. **HubSpot Private App scope gaps cause silent 403s.** 8 required scopes — validate all with startup-time test calls.
7. **Failure notification step may itself fail.** Use `if: always()` on Teams notification. `continue-on-error: true` on artifact upload.
8. **Resource hallucination.** Claude may invent case study names/URLs. Post-parse validation: extract company names and URLs, check against known resource list.

---

## Phase Implications

| Phase | Focus | Key Risk |
|-------|-------|----------|
| 0 — Infrastructure | Repo, secrets, PAT, HubSpot scopes, Make.com wiring, workflow YAML skeleton | Silent auth failures |
| 1 — Skeleton | Stub scripts end-to-end, verify `$RUNNER_TEMP` passing, failure paths | Workflow YAML bugs |
| 2 — Data Ingestion | `fetch_hubspot.py` + `fetch_chorus.py` + `compute_campaign_tokens.py` | Pagination, Chorus auth |
| 3 — Claude Integration | Token budget, prompt assembly, structured outputs, all quality gates | Token overflow, bad JSON |
| 4 — HubSpot Write-Back | Property writes + note creation | Missing custom properties |
| 5 — Failure + Load | Failure paths, 10 concurrent runs, rate limit testing | Concurrency, DLQ edge cases |

---

## Open Questions

1. **Chorus API endpoint paths and rate limits** — auth format confirmed; paths need empirical first-run testing.
2. **HubSpot custom property names** — `job_title_posted`, `job_post_link`, `job_description`, output properties — must be verified as existing in the portal before Phase 4.
3. **Email sequence length: 5 or 7?** — prompt template defines 5 emails; blueprint references 7 output slots. Must be confirmed and consistent across Pydantic schema, template, and HubSpot properties.
4. **HubSpot portal tier** — determines rate limit (100 or 190 req/10s) and whether startup delay needed for concurrent runs.
5. **Make.com poll interval** — determines max concurrent runs in production batch scenario.
