# Features Research: SMYKM Pipeline

**Domain:** AI-powered B2B cold outreach email automation pipeline (GitHub Actions + HubSpot + Chorus AI + Anthropic Claude)
**Researched:** 2026-06-12
**Methodology:** SMYKM (Sam McKenna's "Show Me You Know Me") — hyper-personalised 5-email sequences for Australian offshore staffing (Staff Domain)

---

## Table Stakes

Must-have features. Missing any of these and the system is broken, untrustworthy, or unsafe for production use.

### Data Fetching and Enrichment

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| HubSpot contact property fetch (full set) | Pipeline has nothing to personalise without it | Low | `firstname`, `lastname`, `jobtitle`, `company`, `industry`, `num_employees`, `city`, `country`, `website`, `hubspot_owner_id`, job ad properties — all required |
| HubSpot email engagement history (12 months) | Reveals prior touchpoints; prevents re-pitching already-contacted prospects | Medium | Strip HTML from bodies; summarise to token budget |
| HubSpot meeting engagement history (12 months) | Meeting notes and internal notes carry SDR discovery context | Medium | Fetch both v1 engagements and v4 CRM meeting objects; extract Chorus conversation IDs via regex |
| HubSpot owner name resolution | Personalised "from" context and SDR call notes need owner first name, not raw ID | Low | Resolve via `owners_api.get_by_id()` |
| Chorus transcript fetch (per conversation ID) | Call transcripts are the richest personalisation signal available; transforms generic AI output into genuinely contact-specific content | High | Silent 404/401/timeout fallback is mandatory; pipeline must continue with empty transcripts |
| Chorus conversation ID discovery (multiple sources) | Meeting notes may not always contain IDs; company-name search fallback prevents silent data gaps | Medium | Merge + dedup from: (1) HubSpot meeting note regex, (2) `INPUT_CHORUS_IDS` env override, (3) Chorus v3 engagements search by company |
| Job posting context pass-through | `job_title_posted`, `job_post_link`, `job_description` are the anchor for the SMYKM "Show Me You Know Me" opener — the whole Email 1 hook depends on them | Low | Already HubSpot custom properties; must be nil-safe (empty string if unset, not crash) |

### Retry and Error Handling

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Exponential backoff with jitter on 429 and 5xx | HubSpot, Chorus, and Anthropic all rate-limit; without retry the pipeline fails on burst load | Low | Use `tenacity` with `wait_random_exponential(multiplier=1, min=1, max=60)` + `stop_after_attempt(6)` + `stop_after_delay(60)` |
| `Retry-After` header honoring | Anthropic and HubSpot can return explicit wait times; ignoring them causes immediate re-bans | Low | Custom `RetryAfterWait` callable reads `response.headers.get('Retry-After')` |
| Hard fail on 4xx (except 429) | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found are permanent — retrying them wastes quota and masks bugs | Low | `retry_if_exception(lambda e: _is_retryable(e))` filters to 429 + 5xx only |
| Anthropic SDK `max_retries=0` | Anthropic SDK has built-in retry that conflicts with tenacity; disable it so tenacity owns all retry logic | Low | Pass `max_retries=0` to `anthropic.Anthropic()` constructor |
| DLQ record written on unrecovered failure | Enables manual re-processing; without it failed contacts silently disappear | Low | Write `failed_contacts.json` to `$RUNNER_TEMP` before re-raising; must include `contact_id`, `contact_email`, `failed_step`, `error_message[:2000]`, `timestamp`, `retry_count` |
| Chorus silent fallback | Chorus may be unavailable or contact may have no calls; pipeline must generate emails with whatever context it has | Low | Catch all exceptions in `fetch_chorus.py`; write empty array to `chorus_transcripts.json` and log warning |
| Note creation failure as non-fatal | HubSpot engagement note is audit trail, not core data; failure to create it must not fail the run | Low | Wrap note POST in try/except; log warning and continue |

### Observability and Logging

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Structured step-level logging | GitHub Actions log is the only debugger; unstructured print statements make post-mortems impossible | Low | `logging` module with timestamps; log at script entry/exit, each API call attempt, each retry, each fallback |
| Teams/Slack failure notification with context | SDR team needs immediate visibility when a contact fails; GitHub Actions URL alone is insufficient | Low | POST webhook on `if: failure()` with `contact_email`, `failed_step`, error excerpt (truncated), and direct link to run log |
| DLQ artifact upload on failure | Allows ops team to identify and re-trigger failed contacts without digging through logs | Low | `actions/upload-artifact` with `failed-contacts` name; same `if: failure()` condition |
| Campaign output artifact upload (every run) | Provides 7-day audit trail of what was generated; also useful for manual QA of email quality | Low | `actions/upload-artifact` with `campaign-output` name; 7-day retention |
| Per-step exit codes propagated | GitHub Actions must stop on first script failure; silent partial success is worse than explicit failure | Low | Each Python script `sys.exit(1)` on unrecovered error; `continue-on-error: false` on all steps |
| Token/prompt size logging | Prompt overrun causes truncation; knowing when tiktoken estimate is near 16384 limit allows pre-emptive truncation | Low | Log estimated token count before API call; warn if >14,000 |

### Output Validation Before HubSpot Write

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Required key presence check | Claude occasionally omits keys when context is ambiguous; writing partial output corrupts HubSpot data | Low | Validate all of: `email_1`–`email_5`, `subject_1`–`subject_5`, `sdr_call_notes` present and non-empty |
| Raw JSON parse before access | Claude prompt instructs raw JSON output; any markdown fence, preamble, or explanation causes `json.loads()` to throw | Low | Post-process: strip leading/trailing whitespace; if first char not `{`, attempt to extract JSON block; fail loudly if still invalid |
| HubSpot property character limit enforcement | Multi-line text properties cap at 65,000 chars; exceeding silently truncates or throws 400 | Low | Truncate email bodies to 64,900 chars before write with clear warning log |
| Banned punctuation strip | Em dash and en dash as separators are disallowed by brand voice guidelines; Claude sometimes uses them regardless | Low | Post-process output with `str.replace()` before writing; log when substitution occurs |
| `smykm_generated_date` write | Date stamp enables CRM reporting on campaign coverage and re-generation frequency | Low | Write ISO date on every successful run; fail silently only if this single property write errors |

### Failure Modes Requiring Dedicated Handling

| Failure Mode | Handling Required |
|---|---|
| Contact not found in HubSpot (404) | Hard fail with DLQ entry; do not attempt generation with empty contact |
| Job posting fields missing (`job_title_posted` empty) | Soft fallback: continue generation with `[no job posting available]` substitution; log warning prominently |
| Chorus API completely unavailable (all IDs 404/timeout) | Silent fallback to empty transcripts; must not halt pipeline |
| Claude returns non-JSON or malformed JSON | Retry Claude call once with explicit system prompt reminder; if still invalid, hard fail with DLQ |
| Claude returns valid JSON missing required keys | Hard fail with DLQ; do not partial-write to HubSpot |
| HubSpot write 400 (property doesn't exist) | Hard fail with DLQ; surface property name in error message for ops fix |
| HubSpot write 429 (rate limit) | Retry with backoff per standard retry policy |
| GitHub Actions `$RUNNER_TEMP` missing between steps | Hard fail; each script must validate input file exists before reading |
| `contact_id` not passed (malformed trigger) | Hard fail at step 1 entry with clear error: "contact_id required" |

---

## Differentiators

Features that separate a generic AI email generator from a genuinely high-quality SMYKM pipeline. These are not required for the pipeline to function, but they are the difference between mediocre output and output that SDRs actually trust and send.

### Prompt Engineering Techniques That Improve Email Quality

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Reasoning block in JSON output | Claude articulates why it chose specific personalization angles before writing; surfaces when evidence is weak (e.g. no transcript, no job ad) | Low | `reasoning` key with `drop_reason_classification`, `meeting_evidence_check`, `personalization_source` sub-keys forces Claude to "think before writing" |
| Explicit banned phrases list in system prompt | Prevents boilerplate phrases ("I hope this email finds you well", "touch base", "circle back") that signal AI generation to experienced B2B buyers | Low | Embed as flat list in system prompt; Claude compliance is high with explicit enumeration |
| Hyper-specificity filter instruction | Prompts Claude to test each sentence: "could this sentence appear in an email to a different company?" — if yes, rewrite | Low | Borrowed directly from SMYKM methodology; dramatically reduces generic output |
| Show Up to Solve filter | Every email must open with a prospect problem, not a vendor claim; forces Claude away from feature-lead openers | Low | Explicit instruction in system prompt with good/bad examples |
| Better Than the Weather opener instruction | Banish weather/holiday/generic openers; opening must reference something specific to this prospect | Low | Named filter in prompt; easy to verify in output review |
| Vendor vs. Partner tone instruction | "We work with you" vs "we provide for you" — consistently shifts Claude's framing toward partnership language | Low | One-line instruction with 2-3 examples sufficient |
| Australian English enforcement | Spelling, idiom, and formality must match AU market; US English reads as offshore-generated to Australian buyers | Low | System prompt: "Write in Australian English. Use Australian spelling (e.g. 'organisation' not 'organization')." |
| Resource selection priority rules | Case study matching by industry/role/problem, YouTube video matching by topic, website page matching by need — deterministic rules prevent Claude hallucinating resource URLs | Medium | Embed full lookup table in prompt; Claude selects from enumerated options only |
| Forthcoming Objection pre-emption | Emails 2-5 anticipate the most likely "why not now" objection and address it indirectly | Medium | Requires prompt to specify which objection maps to which email in the sequence arc |
| Stewardship CTA instruction | CTAs request a specific micro-commitment (15 min, a question, a review) rather than generic "let me know if interested" | Low | Named in prompt with examples; high compliance with Claude |

### Context Enrichment Beyond Basic Contact Data

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Chorus call transcript context | Transcripts reveal: hiring pain, team structure, previous objections, decision-maker language/vocabulary, and what was already promised — impossible to fake with scraped data | High | Already in architecture; the single highest-value enrichment source for re-engagement sequences |
| Activity history assembly (email + meeting + transcript) | Shows Claude the full relationship history in one labelled block; prevents re-pitching what was already discussed, and enables "I know you spoke with [owner] about X" references | Medium | Combine into `crm.full_activity_history` token with clear section headers per source type |
| Campaign timing tokens (EOFY context) | Provides temporal relevance cue; "with June 30 approaching" lands differently than a generic send | Low | `compute_campaign_tokens.py` outputs `eofy_timing_context`, `days_to_eofy` — drives urgency framing in emails |
| Owner name personalisation | "Sarah mentioned..." reads as internal knowledge, not cold outreach | Low | Resolve owner first name in `fetch_hubspot.py`; inject as `{{contact.owner_firstname}}` |
| Company website as signal | Job posting link and job description are the sharpest available signal of current pain — hiring for a role = they have a gap | Low | Already fetched as `job_post_link`, `job_description`; Claude instructed to anchor Email 1 hook on this |
| Deal stage awareness | If prospect has an existing deal in HubSpot, the email arc must shift from cold open to re-engagement | Medium | `deal_stage` fetched; prompt must have conditional instruction block based on whether deal exists |

### What Makes the Difference Between Generic and Genuinely Personalised AI Output

The core differentiator is evidence depth. Token-swapping `{{company_name}}` into a generic template is detectable by experienced B2B buyers in Australia immediately. Genuine SMYKM personalisation requires:

1. **A specific hook only this prospect would recognise** — drawn from their job posting, Chorus transcript, or LinkedIn activity. If none of these are available, the prompt must acknowledge the limitation in the `reasoning` block rather than fabricate specificity.
2. **Language mirroring** — Claude should be instructed to note vocabulary the prospect used in calls or emails and mirror it back in the email copy (e.g. if they said "we're drowning in admin," the email references "admin overhead" not "operational inefficiency").
3. **Sequence arc coherence** — each email must build on the previous one; Email 2 (48hr Bump) references Email 1 directly; Email 3 (Peer Proof) cites a case study matched to the specific hiring pain in their job ad. Disconnected emails read as template spam.
4. **Evidence-gated personalization** — if transcript data is absent, the prompt must fall back to job-posting anchors, not invent specificity. Hallucinated personalisation is worse than generic.

### Prompt Versioning and Iteration

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Prompt as version-controlled file in Git | `prompt_template.md` in repository means every change is audited, diffable, and rollback-able without a separate system | Low | Git history IS the version history; no external prompt management tool needed at this scale |
| `{{token.name}}` substitution syntax | Separates prompt structure from variable content; allows prompt iteration without touching Python code | Low | Already in architecture; critical discipline: ALL variable content must be tokens, no hard-coded prospect data in Python |
| Reasoning block output for QA | `reasoning` JSON key lets reviewers verify Claude's logic without reading the full email; enables fast quality checks during prompt iteration | Low | Most valuable during early prompt iteration; SDR team can spot-check reasoning vs email quality |
| Manual `INPUT_CHORUS_IDS` override | Allows testing specific transcript inputs without triggering via Make.com; accelerates prompt iteration | Low | Env var already in architecture; document clearly as the primary testing interface |
| GitHub Actions manual trigger (`workflow_dispatch`) | Allows on-demand re-generation when prompt is updated, without waiting for Make.com list event | Low | Already in architecture; document that re-generating overwrites HubSpot properties (intentional) |

---

## Anti-Features

What to explicitly exclude from v1, and why.

| Anti-Feature | Why Avoid | What to Do Instead |
|---|---|---|
| A/B testing framework | Requires outcome tracking (reply rates, open rates) which needs email send platform integration; doubles complexity with no v1 signal yet | Single best sequence per contact; iterate prompt based on SDR qualitative feedback first |
| Real-time / synchronous processing | GitHub Actions cold start is 30-60s; synchronous user-facing latency is unacceptable. But this is already async — Make.com fires and forgets | Keep async; Make.com doesn't need a response, just a 204 |
| Multi-language support | Australian English only; adding even NZ English variants adds prompt branching with no current prospect need | Hard-code AU English in system prompt |
| Web UI for campaign management | GitHub Actions is the interface; a web UI requires auth, hosting, and maintenance for zero additional capability | Surface run status via Teams notifications + GitHub Actions UI |
| Email send integration (sending directly from pipeline) | Sending from the pipeline bypasses HubSpot sequences, spam warming, and deliverability infrastructure; and creates CAN-SPAM/Spam Act 2003 compliance risks | Write to HubSpot properties only; SDR or HubSpot sequence handles send |
| Prospect research scraping (LinkedIn, company website) | Web scraping is brittle, slow, legally grey in AU, and usually blocked by LinkedIn; adds fragile dependency for marginal gain when job posting data already provides strong hooks | Use HubSpot custom properties (`job_description`, `job_post_link`) as the research anchor |
| Sentiment analysis on transcripts | Adds complexity without actionable output — Claude reads the full transcript and extracts relevant context directly without a pre-processing sentiment layer | Trust Claude to extract relevant signals from raw transcript text |
| Batch processing of multiple contacts per run | Each contact is independent; batching adds failure isolation complexity (one bad contact shouldn't fail others) and complicates artifact naming | Keep 1 contact per workflow dispatch; Make.com handles parallelism by firing N individual dispatches |
| Automatic re-queue / self-healing DLQ | Re-queuing a failed contact risks duplicate emails if the failure occurred after a partial HubSpot write; manual re-trigger is safer | DLQ artifact surfaces the failure; ops team re-triggers manually after verifying state |
| Prompt caching or LLM response caching | Each contact requires unique context assembly; caching prompt responses risks serving stale output for a different contact's data | No caching; cost is acceptable at current volume (1 API call per contact) |
| Output scoring / self-critique loop | Multi-turn critique loops double token cost and latency; quality control should happen via prompt instruction + human SDR review | Strong system prompt + reasoning block output + SDR spot-check is sufficient for v1 |
| CRM other than HubSpot | Not needed; all prospects are in HubSpot; abstracting the CRM layer adds interfaces with no current consumer | HubSpot-specific implementation throughout |
| Logging to external observability platform (Datadog, etc.) | GitHub Actions native logs + Teams notification is sufficient at current volume; external log routing adds infra overhead | Use `logging` module to stdout; GitHub Actions captures and retains for 90 days |

---

## Dependencies Between Features

```
Make.com trigger (workflow_dispatch)
  └── fetch_hubspot.py
        ├── contact_properties          → token substitution in generate_campaign.py
        ├── email_history               → crm.full_activity_history token
        ├── meeting_engagements         → crm.full_activity_history token
        │     └── chorus_conversation_ids  → fetch_chorus.py
        └── owner_name resolution       → contact.owner_firstname token

fetch_chorus.py
  └── chorus_transcripts               → crm.full_activity_history token (silent empty on failure)

compute_campaign_tokens.py
  └── campaign tokens (eofy_context, days_to_eofy, current_date)  → campaign.* tokens

generate_campaign.py
  ├── REQUIRES: hubspot_contact.json from fetch_hubspot.py
  ├── REQUIRES: chorus_transcripts.json from fetch_chorus.py (may be empty array)
  ├── REQUIRES: campaign_tokens.json from compute_campaign_tokens.py
  ├── REQUIRES: prompt_template.md (version-controlled)
  ├── PRODUCES: campaign_output.json (validated JSON)
  └── VALIDATES: required keys present before writing output file

write_hubspot.py
  ├── REQUIRES: campaign_output.json (validated) from generate_campaign.py
  ├── WRITES: email_1–email_7, subject_1–subject_7, smykm_sdr_notes, smykm_generated_date
  └── CREATES: HubSpot note engagement (non-fatal on failure)

DLQ path (if: failure())
  ├── READS: $RUNNER_TEMP/failed_contacts.json (written by any failing script)
  ├── UPLOADS: failed-contacts artifact
  └── POSTS: Teams/Slack webhook notification

Artifact path (always)
  └── UPLOADS: campaign_output.json as campaign-output artifact (7-day retention)
```

**Critical dependency note:** `generate_campaign.py` must validate all three input JSON files exist before assembling the prompt. A missing `campaign_tokens.json` (e.g. `compute_campaign_tokens.py` errored silently) would cause empty token substitution — `{{campaign.eofy_timing_context}}` would appear literally in the prompt sent to Claude.

**Prompt template coupling:** Every `{{token.name}}` placeholder in `prompt_template.md` must have a corresponding key in the `tokens` dict in `generate_campaign.py`. Adding a new token to the prompt without updating the Python dict silently produces literal placeholder text in the Claude prompt. This is the most common maintenance failure mode.

---

## Sources

- SMYKM methodology: [Sam McKenna's SMYKM Cheat Sheet (Apollo)](https://www.apollo.io/academy/sams-perfect-email-smykm-cheat-sheet.pdf) | [ContentGrip: Sam McKenna's cold email method](https://www.contentgrip.com/cold-email-method-sam-mckenna/) | [Outplay: Show Me You Know Me](https://outplay.ai/blog/show-me-you-know-me)
- Retry patterns: [Tenacity docs](https://tenacity.readthedocs.io/) | [Retry strategies for LLM API calls](https://callsphere.ai/blog/retry-strategies-llm-api-calls-exponential-backoff-jitter-tenacity) | [Fault-tolerant AI agent pipelines](https://mightybot.ai/blog/fault-tolerant-ai-agent-pipelines/)
- Claude structured outputs: [Anthropic structured outputs docs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs) | [Anthropic structured outputs launch](https://techbytes.app/posts/claude-structured-outputs-json-schema-api/)
- Prompt versioning: [Prompt versioning guide (Agenta)](https://agenta.ai/blog/prompt-versioning-guide) | [Git-native prompt management](https://medium.com/@jision/i-built-promptops-git-native-prompt-management-for-production-llm-workflows-ae49d1faa628)
- AI cold email quality: [AI cold email pitfalls (earezki.com)](https://earezki.com/ai-news/2026-03-29-why-most-ai-cold-emails-go-to-spam-and-how-to-fix-it/) | [Predictability as the real problem](https://medium.com/@tanin.reachoutly/your-cold-emails-arent-failing-because-of-ai-they-re-failing-because-they-re-predictable-6a61d0cae8b5)
- HubSpot enrichment: [HubSpot data enrichment guide](https://generect.com/blog/hubspot-data-enrichment/) | [HubSpot engagements API reference](https://developers.hubspot.com/docs/guides/api/crm/engagements/engagement-details)
- Australian B2B outreach: [Mastering cold email outreach for Australian B2B](https://captivateclick.com/blog/mastering-cold-email-outreach-for-australian-b2b-success) | [Email sequences for cold outreach 2026](https://salesleadagent.com/blog/email-sequences-cold-outreach-2026)
- GitHub Actions observability: [Production-grade pipeline patterns](https://medium.com/@harsh.vaghela.work/ci-cd-with-github-actions-part-3-security-observability-production-grade-pipeline-patterns-c89db981d2ad) | [OpenTelemetry tracing for GitHub Actions](https://www.dash0.com/guides/github-actions-observability-opentelemetry-tracing)
