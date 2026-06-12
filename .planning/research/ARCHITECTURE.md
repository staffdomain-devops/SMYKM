# Architecture Research: SMYKM Pipeline

**Researched:** 2026-06-12
**Overall confidence:** HIGH — all major claims verified against official GitHub Actions docs, Anthropic structured outputs docs, and primary sources.

---

## Component Map

### The Five Scripts and Their Ownership

Each script owns exactly one external API boundary. No script touches two external systems.

| Script | Owns | Reads | Writes | Fails gracefully? |
|--------|------|-------|--------|-------------------|
| `fetch_hubspot.py` | HubSpot API calls | workflow inputs: `contact_id`, `contact_email` | `$RUNNER_TEMP/hubspot_contact.json` | No — fatal if contact unreachable |
| `fetch_chorus.py` | Chorus AI API calls | `hubspot_contact.json` (for `chorus_conversation_ids`) | `$RUNNER_TEMP/chorus_transcripts.json` | Yes — silent fallback to `[]` on 404/401/timeout |
| `compute_campaign_tokens.py` | Runtime token calculation | System clock, env vars | `$RUNNER_TEMP/campaign_tokens.json` | No — pure computation, cannot fail unless bugs |
| `generate_campaign.py` | Anthropic Claude API | All three JSON files above | `$RUNNER_TEMP/campaign_output.json` | No — fatal if Claude call fails |
| `write_hubspot.py` | HubSpot write-back | `campaign_output.json` | HubSpot contact properties + engagement | Partially — note creation non-fatal |

**Boundary principle:** Each script is independently testable by swapping its input JSON with a fixture file. No script should import from another script. The YAML workflow is the only coordinator.

### Component Responsibilities (What Each Script Must NOT Do)

- `fetch_hubspot.py` must not call Chorus — even if Chorus IDs are visible in the data it fetches. It extracts IDs and writes them to JSON; `fetch_chorus.py` uses them.
- `fetch_chorus.py` must not read from `campaign_tokens.json` — it has no dependency on timing context.
- `compute_campaign_tokens.py` must not make any external HTTP calls. It is pure Python logic.
- `generate_campaign.py` must not write to HubSpot. It writes to `campaign_output.json` only.
- `write_hubspot.py` must not call Claude. It is a pure translation layer between `campaign_output.json` and HubSpot's API.

---

## Data Flow

### Why `$RUNNER_TEMP` JSON Files Are the Right Pattern Here

`$RUNNER_TEMP` is the correct mechanism for this pipeline. Here is the reasoning with alternatives ruled out:

**`$GITHUB_OUTPUT` (step outputs via `echo "key=value" >> $GITHUB_OUTPUT`):**
- Hard platform limit of approximately 1MB per file.
- Values are flat strings; nested JSON must be escaped, making them fragile.
- Not suitable for multi-KB payloads like contact properties, email history, or call transcripts.
- Source: GitHub community reports of truncation above ~1MB in GITHUB_OUTPUT.

**`$GITHUB_ENV` (environment variable injection):**
- Same file format as GITHUB_OUTPUT, same size constraints.
- Designed for simple key-value pairs, not structured payloads.

**`$RUNNER_TEMP` JSON files:**
- No documented size limit (runner disk is ~14GB on `ubuntu-latest`).
- Cleaned up automatically at job end — no manual cleanup needed.
- Files are shared across all steps in the same job (a job runs on a single runner).
- Full JSON structure preserved — no escaping required.
- Readable by Python with `json.load(open(path))` — no parsing gymnastics.

**Security note:** `$RUNNER_TEMP` is scoped to the job — not shared between concurrent runs. API keys and secrets must still flow via GitHub Secrets and environment variables, never via the JSON files. Contact data (name, email, transcript snippets) in `$RUNNER_TEMP` is not a secret leakage risk because it does not leave the runner and is wiped after the job.

### Data Format at Each Boundary

```
workflow_dispatch inputs
  contact_id: "12345"           (string, HubSpot numeric ID)
  contact_email: "name@co.com"  (string, for DLQ records)
        |
        v
$RUNNER_TEMP/hubspot_contact.json
{
  "contact_properties": {
    "firstname": "...", "lastname": "...", "email": "...",
    "jobtitle": "...", "company": "...", "industry": "...",
    "num_employees": "...", "city": "...", "country": "...",
    "website": "...", "hubspot_owner_id": "...",
    "job_title_posted": "...", "job_post_link": "...",
    "job_description": "..."
  },
  "deal_stage": "...",
  "owner_firstname": "...",
  "email_history": [
    {"date": "...", "direction": "OUTGOING|INCOMING", "subject": "...", "body_text": "..."}
  ],
  "meeting_engagements": [
    {"date": "...", "title": "...", "notes": "...", "attendees": [...]}
  ],
  "chorus_conversation_ids": ["abc123", "def456"]
}
        |
        v
$RUNNER_TEMP/chorus_transcripts.json
[
  {
    "conversation_id": "abc123",
    "date": "...",
    "duration_minutes": 42,
    "utterances": [
      {"speaker": "...", "text": "...", "start_time_seconds": 0}
    ]
  }
]
(Empty array [] if Chorus fails or no IDs found)
        |
        v
$RUNNER_TEMP/campaign_tokens.json
{
  "current_date": "2026-06-12",
  "eofy_timing_context": "pre_eofy_full|pre_eofy_compressed|post_eofy",
  "days_to_eofy": 18
}
        |
        v (generate_campaign.py assembles all three files)
$RUNNER_TEMP/campaign_output.json
{
  "reasoning": { ... },
  "email_1": { "subject": "...", "body": "..." },
  ...
  "email_7": { "subject": "...", "body": "..." },
  "sdr_call_notes": { ... }
}
        |
        v
HubSpot contact properties (write_hubspot.py)
  subject_1 through subject_7 (single-line text)
  email_1 through email_7 (multi-line text, up to 65k chars each)
  smykm_sdr_notes (multi-line text)
  smykm_generated_date (date)
  + new HubSpot note (engagement) on the contact record
```

---

## Build Order

### Canonical Sequence

```
Step 1: fetch_hubspot.py
Step 2: fetch_chorus.py          ← depends on Step 1 (needs chorus_conversation_ids)
Step 3: compute_campaign_tokens  ← independent of Steps 1 and 2
Step 4: generate_campaign.py     ← depends on Steps 1, 2, 3
Step 5: write_hubspot.py         ← depends on Step 4
```

### Can Any Steps Run in Parallel?

`compute_campaign_tokens.py` (Step 3) has zero data dependencies on Steps 1 and 2. It only reads the system clock and environment. In theory it could run in parallel with Step 1.

**Recommendation: Do not parallelise. Keep all steps in a single sequential job.**

Rationale:
- Parallelism in GitHub Actions requires separate jobs with `needs:` dependencies. Separate jobs run on separate runner instances, which means `$RUNNER_TEMP` files from one job are inaccessible in another. You would need artifact upload/download between jobs, which adds ~20-30 seconds of overhead and complexity.
- `compute_campaign_tokens.py` takes under 1 second. There is no wall-clock benefit to parallelising it.
- A single job means a single runner filesystem — all `$RUNNER_TEMP` files are always accessible.
- Error isolation (see below) is simpler to reason about in a linear step sequence.
- Billing: a single job is billed as one runner-minute allocation, not two.

`fetch_hubspot.py` and `fetch_chorus.py` cannot run in parallel because Chorus needs the `chorus_conversation_ids` extracted from HubSpot meeting notes. If Chorus IDs were always provided externally (via the `INPUT_CHORUS_IDS` override), they could theoretically run in parallel — but that is an edge case and not worth the complexity.

### Dependency Rationale

| Step | Hard dependency | Soft dependency |
|------|----------------|-----------------|
| fetch_hubspot | workflow inputs only | none |
| fetch_chorus | hubspot_contact.json must exist | none |
| compute_tokens | none | none |
| generate_campaign | all three JSON files must exist | chorus can be empty [] |
| write_hubspot | campaign_output.json must be valid JSON with required keys | none |

---

## Error Isolation Patterns

### The Core Design: One Fatal Tier, One Non-Fatal Tier

Not all failures are equal. The architecture should distinguish:

**Fatal steps** (failure halts pipeline, DLQ record written, notification sent):
- `fetch_hubspot.py` — if we cannot fetch the contact, we have nothing to work with
- `generate_campaign.py` — if Claude fails, there is no output to write
- `write_hubspot.py` — if the write fails, the output is lost (no value delivered)

**Non-fatal steps** (failure is absorbed, pipeline continues with degraded input):
- `fetch_chorus.py` — absent transcripts mean a less personalised email, but the pipeline can still run
- HubSpot note creation inside `write_hubspot.py` — the emails are written; the note is supplementary

### Implementing Non-Fatal Chorus Failure

The `fetch_chorus.py` script uses `continue-on-error: true` at the step level in the YAML, and always writes its output file — even on failure it writes `[]` to `chorus_transcripts.json`.

```yaml
- name: Fetch Chorus transcripts
  id: fetch_chorus
  continue-on-error: true
  env:
    CHORUS_API_TOKEN: ${{ secrets.CHORUS_API_TOKEN }}
    INPUT_CONTACT_EMAIL: ${{ inputs.contact_email }}
    RUNNER_TEMP: ${{ runner.temp }}
  run: python scripts/fetch_chorus.py

- name: Warn if Chorus unavailable
  if: steps.fetch_chorus.outcome == 'failure'
  run: echo "::warning::Chorus fetch failed — proceeding with empty transcripts"
```

This pattern uses `steps.<id>.outcome` (reports `failure`) vs `steps.<id>.conclusion` (reports `success` when `continue-on-error: true` — because the job continues). By checking `outcome`, the downstream generate step can log a warning without blocking.

**Critical detail:** `fetch_chorus.py` must write `chorus_transcripts.json` as `[]` in its exception handler before re-raising or exiting. `generate_campaign.py` must treat a missing or empty `chorus_transcripts.json` as valid input, not an error.

### Implementing Fatal Failure + DLQ

For fatal steps, the pattern is:

1. Step fails normally (non-zero exit code) — GitHub Actions marks the job as failed.
2. The script writes `$RUNNER_TEMP/failed_contacts.json` before raising the exception (DLQ record).
3. Two final steps run under `if: failure()`:

```yaml
- name: Upload failed contact record
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: failed-contacts-${{ github.run_id }}
    path: failed_contacts.json
    retention-days: 30

- name: Notify Teams on failure
  if: failure()
  env:
    TEAMS_WEBHOOK_URL: ${{ secrets.TEAMS_WEBHOOK_URL }}
  run: |
    python -c "
    import os, json, requests
    requests.post(os.environ['TEAMS_WEBHOOK_URL'], json={
      'text': 'SMYKM pipeline failed',
      'attachments': [{'text': 'Contact: ${{ inputs.contact_email }} | Run: ${{ github.run_id }}'}]
    })
    "
```

**Important note on `if: failure()`:** When you write a custom `if:` condition on a step, the implicit `if: success()` is dropped. The `if: failure()` steps will only run when the job is in a failed state. Steps with no `if:` condition have an implicit `if: success()` applied and will be skipped after any failure.

### The `continue-on-error` Trap

`continue-on-error: true` at the **job level** behaves differently from step level: "the job reports success but the overall workflow reports failure." Do not use it at the job level — only at the step level for Chorus.

### Step Outcome Reference

| Scenario | `steps.X.outcome` | `steps.X.conclusion` |
|----------|------------------|---------------------|
| Step succeeded | `success` | `success` |
| Step failed, no `continue-on-error` | `failure` | `failure` |
| Step failed, `continue-on-error: true` | `failure` | `success` |

Use `outcome` (not `conclusion`) when you want to detect "this step failed even though the job continued."

---

## Context Assembly Pattern

### The Problem

`generate_campaign.py` must assemble a prompt context from three sources with different sizes and relevance levels:
- HubSpot contact properties (small, ~1-2KB, always present)
- Chorus call transcripts (large, potentially 10-50KB per call, zero to many calls)
- Email/meeting history (medium, ~2-10KB)
- Campaign tokens (tiny, ~0.1KB)

The Claude Sonnet 4.6 context window is 200K tokens, but `max_tokens=16384` is reserved for output. The practical input budget is approximately 180K tokens — more than enough for this use case, but transcripts can be large and must be managed.

### Recommended Assembly Architecture: Budget-First Truncation

Assign each component a token budget. Measure tokens with `tiktoken`. Truncate per-component before assembling, not after.

```python
import tiktoken

ENCODING = tiktoken.get_encoding("cl100k_base")  # Good approximation for Claude

# Token budgets (total input target: ~150K tokens, leaving 30K headroom)
BUDGET = {
    "system_prompt":     2_000,   # Fixed — the methodology instructions
    "prompt_template":  15_000,   # The main prompt scaffolding
    "contact_props":     2_000,   # All contact properties
    "email_history":     8_000,   # Past 12 months emails
    "meeting_history":   4_000,   # Past 12 months meetings
    "transcripts":      60_000,   # Chorus transcripts (large but bounded)
    "campaign_tokens":     500,   # Timing context etc.
    "output_reserve":   16_384,   # Reserved for Claude's response
}
TOTAL_BUDGET = 150_000

def count_tokens(text: str) -> int:
    return len(ENCODING.encode(text))

def truncate_to_budget(text: str, budget: int, label: str) -> str:
    tokens = ENCODING.encode(text)
    if len(tokens) <= budget:
        return text
    truncated = ENCODING.decode(tokens[:budget])
    print(f"WARNING: {label} truncated from {len(tokens)} to {budget} tokens")
    return truncated + f"\n\n[... {label} truncated to fit context window ...]"
```

### Activity History Assembly Order (Priority: Newest First)

Within each source type, assemble with the most recent items first. This ensures that if truncation is needed, older/less-relevant content is cut, not recent context.

```python
def build_activity_history(
    email_history: list,
    meeting_engagements: list,
    chorus_transcripts: list,
) -> str:
    sections = []

    # --- CALL TRANSCRIPTS (highest signal for personalisation) ---
    if chorus_transcripts:
        for t in chorus_transcripts:
            transcript_text = f"## Call Transcript: {t['date']}\n"
            for u in t.get('utterances', []):
                transcript_text += f"{u['speaker']}: {u['text']}\n"
            sections.append(("transcript", t['date'], transcript_text))

    # --- MEETING NOTES ---
    for m in meeting_engagements:
        text = f"## Meeting: {m['date']} — {m.get('title', '')}\n{m.get('notes', '')}"
        sections.append(("meeting", m['date'], text))

    # --- EMAIL HISTORY ---
    for e in email_history:
        text = f"## Email ({e['direction']}): {e['date']}\nSubject: {e['subject']}\n{e['body_text']}"
        sections.append(("email", e['date'], text))

    # Sort all items newest-first across all types
    sections.sort(key=lambda x: x[1], reverse=True)

    # Assemble with budget
    assembled = []
    tokens_used = 0
    for source_type, date, text in sections:
        t = count_tokens(text)
        if tokens_used + t > BUDGET["transcripts"] + BUDGET["email_history"] + BUDGET["meeting_history"]:
            assembled.append("[... older activity truncated to fit context window ...]")
            break
        assembled.append(text)
        tokens_used += t

    return "\n\n---\n\n".join(assembled)
```

### Token Substitution in the Prompt Template

The `{{token.name}}` pattern is the right choice for this use case. Assessment:

**Why `{{token.name}}` over alternatives:**

| Approach | Pros | Cons | Verdict |
|----------|------|------|---------|
| `{{token.name}}` custom regex | Simple, readable, no deps, safe | No conditionals or loops | Best for this use case |
| Python f-strings | Native, fast | Crashes on `{` or `}` in prompt text (common in JSON examples); must escape all braces | Dangerous |
| Jinja2 | Full conditionals, loops, filters | Security risk if templates are untrusted; adds a dependency; overkill for token substitution | Overkill |
| Python `.format()` | Native | Same brace-escaping problem as f-strings | Dangerous |

**Implementation:**

```python
import re

def substitute_tokens(template: str, tokens: dict) -> str:
    """
    Replaces {{token.name}} and {{token.sub.name}} with values from a nested dict.
    Raises KeyError if a placeholder has no corresponding token.
    """
    def replacer(match):
        key_path = match.group(1).split('.')
        value = tokens
        for key in key_path:
            if not isinstance(value, dict) or key not in value:
                raise KeyError(f"Token not found: {match.group(0)}")
            value = value[key]
        return str(value)

    return re.sub(r'\{\{([a-zA-Z0-9_.]+)\}\}', replacer, template)

# Usage in generate_campaign.py:
tokens = {
    "contact": contact_properties,          # dict from hubspot_contact.json
    "crm": {
        "full_activity_history": activity_history_string,
    },
    "campaign": campaign_tokens_dict,        # from campaign_tokens.json
}

prompt = substitute_tokens(template_text, tokens)
```

**Limitations of this approach:**
- No conditional sections (cannot include a block only if transcripts exist). Workaround: set the token value to an empty string or a fallback string (e.g., `"No call history available."`).
- No loops in template. All lists must be pre-formatted as strings before substitution.
- Template tokens must be defined before substitution runs — a missing token raises an error at render time, which is the right behaviour (fail fast, don't silently omit content).

---

## Claude Output Schema Design

### Recommended Schema

The current schema in the blueprint is well-designed. Key principles:

1. **Flat email envelope pattern** (`email_N.subject` + `email_N.body`) is better than a flat list because it allows `write_hubspot.py` to reference `campaign_output["email_1"]["body"]` directly without index arithmetic.

2. **Reasoning block first** — Claude reasons better when it writes reasoning before output. The `reasoning` object should be the first key in the schema. This is a known prompting pattern (chain-of-thought before structured output).

3. **Required keys validation before write-back** — `write_hubspot.py` should validate the schema before touching HubSpot. A partial write (e.g., 3 of 7 email subjects) is worse than no write.

### Using Claude Structured Outputs (official API feature, GA as of late 2025)

Claude Sonnet 4.6 supports structured outputs via `output_config.format`. This guarantees schema compliance at the token generation level — Claude literally cannot produce JSON that violates the schema.

```python
import anthropic
from pydantic import BaseModel
from typing import Optional

class EmailBlock(BaseModel):
    subject: str
    body: str

class ReasoningBlock(BaseModel):
    drop_reason_classification: str
    eofy_timing_context: str
    meeting_evidence_check: str
    personalisation_signals: str

class SdrCallNotes(BaseModel):
    quick_brief: str
    the_hook: str

class CampaignOutput(BaseModel):
    reasoning: ReasoningBlock
    email_1: EmailBlock
    email_2: EmailBlock
    email_3: EmailBlock
    email_4: EmailBlock
    email_5: EmailBlock
    email_6: EmailBlock
    email_7: EmailBlock
    sdr_call_notes: SdrCallNotes

client = anthropic.Anthropic()
response = client.messages.parse(
    model="claude-sonnet-4-6",
    max_tokens=16384,
    system=system_prompt,
    messages=[{"role": "user", "content": assembled_prompt}],
    output_format=CampaignOutput,   # SDK handles schema extraction from Pydantic
)
output = response.parsed_output  # Fully typed CampaignOutput instance
```

**No beta header required** — structured outputs moved out of beta. The old `anthropic-beta: structured-outputs-2025-11-13` header still works for a transition period but is deprecated.

**Key constraint:** `additionalProperties: false` is required and enforced automatically by the SDK. The schema cannot use recursive types or external `$ref` URLs. All the email/SDR fields in this project are flat strings or simple objects — no constraints are hit.

**Trade-off vs raw JSON prompting:**
- Structured outputs: guaranteed schema compliance, no parsing errors, cleaner code. Adds ~100ms latency on first request (grammar compilation), cached for 24 hours.
- Raw JSON prompting: more flexible (can return freeform reasoning text outside the schema), but requires defensive parsing and post-processing. The blueprint's current approach of stripping markdown fences and validating keys is a reasonable fallback if structured outputs introduce unexpected issues.

**Recommendation:** Use structured outputs for the main email/SDR fields. Keep `reasoning` inside the schema (not free-text) to force structured reasoning that can be logged and audited.

### Validation Before HubSpot Write

Even with structured outputs, validate before writing:

```python
REQUIRED_KEYS = [f"email_{i}" for i in range(1, 6)] + ["sdr_call_notes"]
REQUIRED_EMAIL_KEYS = ["subject", "body"]

def validate_campaign_output(data: dict) -> None:
    for key in REQUIRED_KEYS:
        if key not in data:
            raise ValueError(f"Missing required key in campaign output: {key}")
    for i in range(1, 6):
        email = data[f"email_{i}"]
        for ek in REQUIRED_EMAIL_KEYS:
            if ek not in email:
                raise ValueError(f"email_{i} missing key: {ek}")
        if len(email["body"]) > 65_000:
            raise ValueError(f"email_{i}.body exceeds HubSpot 65k char limit")
```

---

## Make.com Integration

### What Make.com Sends

Make.com calls the GitHub REST API to dispatch the workflow:

```
POST https://api.github.com/repos/{owner}/{repo}/actions/workflows/campaign.yml/dispatches
Authorization: Bearer {GITHUB_PAT}
Accept: application/vnd.github+json
Content-Type: application/json

{
  "ref": "main",
  "inputs": {
    "contact_id": "12345",
    "contact_email": "prospect@company.com"
  }
}
```

This is a standard HTTP module in Make.com with:
- URL: the dispatches endpoint
- Method: POST
- Headers: Authorization and Accept as above
- Body: JSON with `ref` and `inputs`

### GitHub PAT Permissions

Use a fine-grained Personal Access Token scoped to the specific repository with:
- **Actions: Read and Write** — required to trigger workflow_dispatch
- **Contents: Read and Write** — required by the API endpoint
- **Metadata: Read** — enabled by default on fine-grained tokens

Do **not** use a classic PAT with `repo` scope if avoidable — fine-grained tokens limit blast radius if compromised.

Store the PAT as a Make.com connection secret (not hardcoded in the scenario).

### What the Workflow Inputs Declare

```yaml
on:
  workflow_dispatch:
    inputs:
      contact_id:
        description: "HubSpot contact numeric ID"
        required: true
        type: string
      contact_email:
        description: "Contact email (for DLQ records and notifications)"
        required: true
        type: string
```

Both are `type: string` — HubSpot numeric IDs can have leading zeros in edge cases, so string is safer than number.

### Make.com Webhook Payload Checklist

The payload Make.com sends must include exactly `contact_id` and `contact_email`. No other inputs are needed — the pipeline fetches everything else from APIs. This keeps the Make.com scenario simple (it is a pure trigger, not a data carrier).

**Anti-pattern:** Do not pass the full contact JSON as a workflow input. Workflow inputs are logged in GitHub's run summary and are not treated as secrets. Contact data should be fetched inside the workflow via authenticated API calls, not passed in as inputs.

---

## Suggested Build Order for Implementation

### Phase 1: Skeleton and Infrastructure

Build the structural scaffolding with no business logic. Goal: a working pipeline that runs end-to-end with stub data.

1. Create `requirements.txt` with all dependencies pinned
2. Create `.github/workflows/campaign.yml` with all 5 steps, both `if: failure()` steps, and `continue-on-error: true` on fetch_chorus step
3. Implement the `RUNNER_TEMP` file path helper (used by all scripts):
   ```python
   def temp_path(filename: str) -> str:
       return os.path.join(os.environ.get("RUNNER_TEMP", "/tmp"), filename)
   ```
4. Implement the DLQ writer (used by 4 of 5 scripts)
5. Implement the tenacity retry decorator (used by 3 of 5 scripts)
6. Smoke test: create stub scripts that write hardcoded JSON to RUNNER_TEMP and verify the file is accessible in the next step

### Phase 2: Data Ingestion Scripts

Build `fetch_hubspot.py` and `fetch_chorus.py` against real APIs.

1. `fetch_hubspot.py` — properties, email engagements, meeting engagements, CRM meetings via v4 API, owner name resolution
2. Test with a real contact ID against the HubSpot sandbox
3. `fetch_chorus.py` — conversation search by company name, transcript fetch, silent fallback
4. Test with both a valid Chorus ID and an invalid one (verify `[]` output)

### Phase 3: Token Computation and Prompt Assembly

Build `compute_campaign_tokens.py` and the context assembly logic in `generate_campaign.py`.

1. `compute_campaign_tokens.py` — EOFY timing logic and any other runtime tokens
2. Build `build_activity_history()` in `generate_campaign.py` with tiktoken budget management
3. Build `substitute_tokens()` with the `{{token.name}}` regex pattern
4. Unit test token substitution with a fixture template and fixture JSON

### Phase 4: Claude Integration

Wire up the actual Claude API call.

1. Implement structured outputs with the Pydantic schema for `CampaignOutput`
2. Implement the system prompt (Australian English, SMYKM methodology constraints)
3. Test with a real contact + fixture transcripts
4. Validate output schema, verify character counts, verify banned punctuation stripping

### Phase 5: HubSpot Write-Back

Implement `write_hubspot.py`.

1. Map `campaign_output.json` keys to HubSpot property names
2. Implement contact property batch update (single PATCH call with all 16 properties)
3. Implement HubSpot engagement (note) creation with HTML body
4. Implement non-fatal note failure (log warning, continue)
5. End-to-end test: run full pipeline on a test contact, verify properties appear in HubSpot

### Phase 6: Failure Path Testing

Deliberately break each step and verify the failure path works.

1. Revoke Chorus token temporarily — verify pipeline continues, emails are less detailed but succeed
2. Pass an invalid contact_id — verify DLQ record is written and Teams notification fires
3. Pass a valid contact but malformed template — verify generate step fails with DLQ
4. Verify artifact upload captures `failed_contacts.json`

---

## Sources

- GitHub Actions — passing information between jobs: https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/passing-information-between-jobs
- GitHub Actions — contexts (step outputs): https://docs.github.com/en/actions/learn-github-actions/contexts
- GitHub Actions — error handling: https://www.kenmuse.com/blog/how-to-handle-step-and-job-errors-in-github-actions/
- GitHub Actions — RUNNER_TEMP: https://nesin.io/blog/temp-directory-path-github-actions
- GitHub Actions — single job vs multiple jobs: https://github.com/orgs/community/discussions/52927
- Anthropic — structured outputs: https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- GitHub — fine-grained PAT permissions for workflow dispatch: https://github.com/orgs/community/discussions/68302
- Ackama — passing values between steps: https://www.ackama.com/articles/values-github-actions/
- GITHUB_OUTPUT size limit (~1MB): https://github.com/google-github-actions/run-gemini-cli/issues/479
- HubSpot API rate limits: https://developers.hubspot.com/docs/developer-tooling/platform/usage-guidelines
