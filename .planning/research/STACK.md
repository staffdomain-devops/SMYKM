# Stack Research: SMYKM Pipeline

**Project:** Staff Domain SMYKM Campaign Pipeline
**Researched:** 2026-06-12
**Scope:** Python 3.12 pipeline running in GitHub Actions — HubSpot, Chorus AI, Anthropic Claude

---

## Recommended Stack

| Layer | Library / Tool | Pinned Version | Rationale | Confidence |
|-------|---------------|----------------|-----------|------------|
| Python runtime | Python | 3.12 | Confirmed in PROJECT.md; supported by all libraries below | HIGH |
| HubSpot CRM client | `hubspot-api-client` | `>=12.0.0` | Latest stable (12.0.0, May 2025); v4 associations API in this release series; private app token auth | HIGH |
| HTTP client (Chorus + webhooks) | `requests` | `>=2.34.0` | Latest stable 2.34.2 (May 2026); prior 2.32.0–2.32.1 yanked due to CVE-2024-35195; `>=2.34.0` avoids all yanked/patched versions safely | HIGH |
| HTML parsing (job descriptions) | `beautifulsoup4` | `>=4.12.0` | Latest stable 4.15.0 (Jun 2026); 4.12 baseline fine, `>=4.12.0` picks up any release; pair with `lxml` parser for speed | HIGH |
| Anthropic API | `anthropic` | `>=0.50.0` | Latest stable 0.109.1 (Jun 2026); 0.50+ includes stable prompt caching without beta headers; `>=0.30.0` is too conservative — bump floor | HIGH |
| HTML-to-text parsing | `lxml` | `>=5.2.0` | Required as BS4 parser backend for reliable, fast HTML parsing of job descriptions; not in original spec but needed | HIGH |
| Retry logic | `tenacity` | `>=9.0.0` | Latest stable 9.1.4 (Feb 2026); `wait_exponential_jitter`, `wait_exception` (reads Retry-After header), `retry_if_exception_type` — all in 9.x | HIGH |
| Token counting | `tiktoken` | `>=0.7.0` | Latest stable 0.13.0 (May 2026); use for pre-flight token budget checks before sending to Claude; `>=0.7.0` baseline is fine | HIGH |
| GitHub Actions runtime | `ubuntu-latest` | — | Maps to ubuntu-24.04 as of mid-2025; standard for Python pipelines | HIGH |
| GitHub Actions artifact | `actions/upload-artifact` | `v4` | v3 deprecated Dec 2024; v4 is current, immutable artifacts, 90% faster uploads | HIGH |
| Failure notifications | `curl` via `run:` step | — | Direct `curl` POST to Teams/Slack Incoming Webhook URL; no third-party action dependency needed for a JSON POST | HIGH |
| Secrets management | GitHub Encrypted Secrets | — | `secrets.HUBSPOT_TOKEN` etc.; pass as `${{ secrets.X }}` env vars into Python steps; never write to GITHUB_ENV | HIGH |

---

## Validation Notes

### `hubspot-api-client` — current: 12.0.0

**Version history:** 9.0.0 (Mar 2024) → 10.0.0 (Oct 2024) → 11.0.0 (Nov 2024) → 12.0.0 (May 2025)

**Breaking changes in 12.0.0 relevant to this project:**
- `crm.associations.v4.basic_api.archive()`, `create()`, and `create_default()` — parameter types for `object_id` and `to_object_id` changed from `int` to `string`. Any hardcoded integer IDs passed to v4 association methods must be strings.
- `crm.associations.v4.basic_api.get_page()` — `object_id` changed from `int` to `string`.
- `crm.associations.v4.models.LabelsBetweenObjectPair` — `to_object_id` and `from_object_id` changed to `string`.
- `crm.associations.v4.schema.definitions_api.delete()` renamed to `.archive()`.
- `hs_note_body` is the correct property key for note body (not `content`). The note create endpoint is `POST /crm/v3/objects/notes`.

**HubSpot v4 Associations pattern (confirmed from official docs):**
```python
from hubspot.crm.associations.v4 import BatchInputPublicAssociationMultiPost

result = client.crm.associations.v4.batch_api.create(
    from_object_type="contacts",
    to_object_type="notes",
    batch_input_public_association_multi_post=BatchInputPublicAssociationMultiPost(
        inputs=[{
            "from": {"id": "contact_id_string"},  # must be string in v12+
            "to": {"id": "note_id_string"},
            "types": [{
                "associationCategory": "HUBSPOT_DEFINED",
                "associationTypeId": 202  # contact-to-note
            }]
        }]
    )
)
```

**Note creation pattern (confirmed from HubSpot developer docs):**
```python
note = client.crm.objects.basic_api.create(
    object_type="notes",
    simple_public_object_input_for_create=SimplePublicObjectInputForCreate(
        properties={
            "hs_note_body": "<p>SMYKM pipeline run completed.</p>",
            "hs_timestamp": "2026-06-12T10:00:00Z",
            "hubspot_owner_id": "owner_id"
        },
        associations=[{
            "to": {"id": contact_id},
            "types": [{
                "associationCategory": "HUBSPOT_DEFINED",
                "associationTypeId": 202
            }]
        }]
    )
)
```

---

### `anthropic` — current: 0.109.1

**Spec says `>=0.30.0`. Recommended: bump to `>=0.50.0`** — prompt caching without beta headers was stabilised in 0.40+. The `>=0.30.0` floor predates stable caching support and will install a version that requires `anthropic-beta: prompt-caching-2024-07-31` headers manually.

**Model string:** `claude-sonnet-4-6` — confirmed as valid API ID (dateless, pinned snapshot, not an alias). Context window: 1M tokens. Max output: 64k tokens. Pricing: $3/$15 per MTok input/output.

**SDK retry behaviour (from source):**
The SDK retries by default on 408, 409, 429, 5xx with `DEFAULT_MAX_RETRIES = 2`. The project decision to set `max_retries=0` and use tenacity for unified retry logic is correct and well-founded — this gives a single retry surface across HubSpot, Chorus, and Anthropic calls with consistent logging.

**Prompt caching best practice for this pipeline:**
- claude-sonnet-4-6 minimum cacheable block: 1,024 tokens
- Place the full system prompt (SMYKM methodology, voice guidelines, banned phrases, case studies, resources list) as a static `cache_control: ephemeral` block in `system[]`
- Place per-contact dynamic content (contact properties, transcript) in the `messages` array without cache_control
- 5-minute TTL (default, free refreshes) is sufficient for batch processing; the system prompt will be cached across all contacts processed within a 5-minute window
- No beta header required for 5-minute or 1-hour TTL as of SDK 0.40+

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16384,
    max_retries=0,  # tenacity handles retries
    system=[
        {
            "type": "text",
            "text": STATIC_SYSTEM_PROMPT,  # full methodology, guidelines, resources
            "cache_control": {"type": "ephemeral"}  # cached, re-read free within 5min
        }
    ],
    messages=[
        {
            "role": "user",
            "content": dynamic_user_message  # contact data + transcripts — not cached
        }
    ]
)
```

**Error types to catch explicitly:**
```python
from anthropic.types import RateLimitError, OverloadedError
from anthropic import APIStatusError, APIConnectionError
```

---

### `requests` — current: 2.34.2

**Versions 2.32.0 and 2.32.1 were yanked** due to CVE-2024-35195 (SSRF mitigation). Patch landed in 2.32.2; recommend pinning `>=2.34.0` to stay clear of yanked range.

`requests` is the correct choice over `httpx` for this pipeline. The pipeline is synchronous (sequential Python scripts, not async), and `requests` has cleaner error handling for simple REST calls. The Anthropic SDK uses `httpx` internally; adding another HTTP/2 client for Chorus would be unnecessary complexity.

---

### `beautifulsoup4` — current: 4.15.0

The `>=4.12.0` floor is fine. Add `lxml` as an explicit dependency — BS4 will fall back to Python's stdlib `html.parser` silently if lxml is not installed, which is slower and less forgiving of malformed HTML (job descriptions are often malformed). Explicit parser selection in code: `BeautifulSoup(html, "lxml")`.

`lxml>=5.2.0` is the recommended version — Python 3.12 wheels available, actively maintained.

---

### `tenacity` — current: 9.1.4

`>=9.0.0` is the correct floor. Key patterns for this pipeline:

**Canonical retry decorator for HubSpot/Chorus/Anthropic:**
```python
from tenacity import (
    retry, stop_after_attempt, wait_exponential_jitter,
    retry_if_exception_type, before_sleep_log
)
import logging

logger = logging.getLogger(__name__)

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential_jitter(multiplier=1, max=60, jitter=5),
    retry=retry_if_exception_type((requests.HTTPError, requests.Timeout, requests.ConnectionError)),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True
)
def call_with_retry(fn, *args, **kwargs):
    return fn(*args, **kwargs)
```

For Anthropic calls, catch `anthropic.RateLimitError` and `anthropic.APIStatusError` with appropriate status code filtering.

For HubSpot calls with `hubspot-api-client`, the SDK raises `ApiException` — inspect `status` attribute for 429/5xx.

---

### `tiktoken` — current: 0.13.0

`>=0.7.0` is the right floor. Use for pre-flight token budget estimation before constructing the Claude request. Note: claude-sonnet-4-6 uses Anthropic's tokenizer (not GPT-4 tokenizer), so tiktoken gives an approximation, not an exact count. Use it with a safety margin (e.g., multiply tiktoken count by 1.1 or compare against response `usage.input_tokens` empirically). The primary use case here is preventing prompt assembly that would exceed max_tokens=16384 for output.

---

## GitHub Actions Patterns

### workflow_dispatch with Make.com

Make.com fires `workflow_dispatch` via the GitHub API. As of December 2025, `workflow_dispatch` supports **up to 25 inputs** (raised from 10). The pipeline only needs `contact_id` and `contact_email`, so this is not a constraint.

```yaml
on:
  workflow_dispatch:
    inputs:
      contact_id:
        description: "HubSpot Contact ID"
        required: true
        type: string
      contact_email:
        description: "Contact email for logging"
        required: true
        type: string
```

Make.com HTTP module: `POST https://api.github.com/repos/{owner}/{repo}/actions/workflows/{workflow_file}/dispatches` with `Authorization: Bearer {PAT}` and body `{"ref": "main", "inputs": {...}}`.

---

### Secrets Handling

- All API tokens (HubSpot, Chorus, Anthropic, Teams/Slack webhook) stored as **repository secrets** under Settings → Secrets and Variables → Actions.
- Pass into Python steps as environment variables, not as workflow inputs:
```yaml
env:
  HUBSPOT_TOKEN: ${{ secrets.HUBSPOT_TOKEN }}
  CHORUS_TOKEN: ${{ secrets.CHORUS_TOKEN }}
  ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
  TEAMS_WEBHOOK_URL: ${{ secrets.TEAMS_WEBHOOK_URL }}
```
- Never echo secrets or write them to `$GITHUB_ENV` or `$RUNNER_TEMP` files. JSON data files in `$RUNNER_TEMP` should contain only contact data — not tokens.
- OIDC is not applicable here (OIDC is for cloud provider auth like AWS/Azure; HubSpot/Chorus/Anthropic use static API tokens).

---

### Inter-Step Data (RUNNER_TEMP JSON Files)

`$RUNNER_TEMP` is the correct approach — isolated per-job, automatically cleaned up after the job completes, not accessible outside the runner VM. Confirmed secure for ephemeral data.

Pattern:
```yaml
- name: Fetch HubSpot data
  env:
    HUBSPOT_TOKEN: ${{ secrets.HUBSPOT_TOKEN }}
  run: |
    python scripts/01_fetch_hubspot.py \
      --contact-id "${{ inputs.contact_id }}" \
      --output "$RUNNER_TEMP/hubspot_data.json"

- name: Fetch Chorus transcripts
  run: |
    python scripts/02_fetch_chorus.py \
      --contact-email "${{ inputs.contact_email }}" \
      --output "$RUNNER_TEMP/transcripts.json"
```

---

### Artifact Upload

Use `actions/upload-artifact@v4` — v3 was deprecated December 2024 and removed from GitHub.com in early 2025.

```yaml
- name: Upload campaign output
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: smykm-${{ inputs.contact_id }}-${{ github.run_id }}
    path: ${{ runner.temp }}/campaign_output.json
    retention-days: 7
    if-no-files-found: warn
```

Key v4 notes:
- Artifact names must be unique per workflow run (use `contact_id` + `run_id` in name).
- Artifacts are immutable once created.
- `${{ runner.temp }}` is the YAML expression equivalent of `$RUNNER_TEMP` in shell.

---

### Failure Notification (Teams/Slack)

Use a terminal step with `if: failure()` sending a `curl` POST. No third-party action needed — avoids supply chain risk for a simple JSON POST.

```yaml
- name: Notify failure
  if: failure()
  env:
    TEAMS_WEBHOOK_URL: ${{ secrets.TEAMS_WEBHOOK_URL }}
  run: |
    curl -s -X POST "$TEAMS_WEBHOOK_URL" \
      -H "Content-Type: application/json" \
      -d "{
        \"type\": \"message\",
        \"attachments\": [{
          \"contentType\": \"application/vnd.microsoft.card.adaptive\",
          \"content\": {
            \"type\": \"AdaptiveCard\",
            \"body\": [{
              \"type\": \"TextBlock\",
              \"text\": \"SMYKM pipeline failed for contact ${{ inputs.contact_id }} — run ${{ github.run_id }}\"
            }]
          }
        }]
      }"
```

For Slack, the payload is simpler:
```yaml
-d "{\"text\": \"SMYKM pipeline failed: contact=${{ inputs.contact_id }} run=${{ github.run_id }}\"}"
```

Teams is strict about Adaptive Card JSON format — a stray quote or missing comma silently rejects the POST. Test the webhook payload independently before relying on it.

---

### DLQ (Dead Letter Queue) Pattern

Write a DLQ record to a JSON file before uploading as an artifact with a separate `dlq-` prefixed name and longer retention (30 days):

```yaml
- name: Write DLQ record
  if: failure()
  run: |
    python -c "
    import json, os, datetime
    dlq = {
      'contact_id': '${{ inputs.contact_id }}',
      'contact_email': '${{ inputs.contact_email }}',
      'run_id': '${{ github.run_id }}',
      'failed_at': datetime.datetime.utcnow().isoformat(),
      'workflow': '${{ github.workflow }}'
    }
    with open(os.environ['RUNNER_TEMP'] + '/dlq.json', 'w') as f:
        json.dump(dlq, f)
    "

- name: Upload DLQ artifact
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: dlq-${{ inputs.contact_id }}-${{ github.run_id }}
    path: ${{ runner.temp }}/dlq.json
    retention-days: 30
```

---

### New GitHub Actions Features Worth Using (2025-2026)

1. **25 workflow_dispatch inputs** (Dec 2025) — no longer need to pack multiple values into a single JSON string input.
2. **Repository custom property claims in OIDC tokens** (Apr 2026, GA) — not applicable here (no cloud provider auth), but relevant if the project later integrates with AWS S3 for long-term artifact storage.
3. **Artifact immutability (v4)** — already captured above. Critical: if you re-run a failed job that already uploaded an artifact, the upload will fail unless the artifact name includes the run attempt number (`${{ github.run_attempt }}`).

---

## What NOT to Use

| Category | Avoid | Why |
|----------|-------|-----|
| HTTP client | `httpx` (for this pipeline) | Async overhead is wasted — pipeline is synchronous scripts; `requests` is cleaner for straightforward REST calls |
| HTTP client | `aiohttp` | Same reason — no async required |
| Retry | Anthropic SDK built-in retry (`max_retries>0`) | Breaks unified retry logic; Anthropic SDK uses different backoff than tenacity; PROJECT.md decision explicitly disables this |
| Token counting | tiktoken as an exact counter for Claude | tiktoken uses OpenAI tokenizer; Claude has its own — use tiktoken for estimation only, add safety margin |
| HubSpot auth | `hapikey` parameter | Deprecated since hubspot-api-client 5.1.0; use Private App token only |
| Notifications | `slackapi/slack-github-action` action | Adds a third-party action dependency for what is a single `curl` POST; direct `curl` is simpler and has no supply chain risk |
| Artifact upload | `actions/upload-artifact@v3` | Deprecated and removed from GitHub.com (Dec 2024 / early 2025) |
| Data passing | `$GITHUB_ENV` for inter-step JSON | Env vars are length-limited and logged; `$RUNNER_TEMP` JSON files are the correct pattern for structured data between steps |
| HubSpot engagement API | Legacy `/engagements/v1/` endpoint | Deprecated; use `/crm/v3/objects/notes` with `hs_note_body` property |
| HTML parsing | `html.parser` (default BS4) | Slower, less forgiving of malformed HTML; always specify `lxml` explicitly |
| BS4 alternative | `selectolax` | Faster but minimal community adoption, no justification for the complexity swap in this pipeline |

---

## Final requirements.txt Recommendation

```text
# HubSpot CRM
hubspot-api-client>=12.0.0

# HTTP (Chorus AI, webhooks)
requests>=2.34.0

# HTML parsing (job descriptions)
beautifulsoup4>=4.12.0
lxml>=5.2.0

# Anthropic Claude
anthropic>=0.50.0

# Token estimation
tiktoken>=0.7.0

# Retry logic
tenacity>=9.0.0
```

Python version constraint: `python_requires>=3.12` (or `python-version: '3.12'` in `actions/setup-python`).

---

## Sources

- HubSpot Python API client PyPI: https://pypi.org/project/hubspot-api-client/
- HubSpot v12 breaking changes — associations v4: https://github.com/HubSpot/hubspot-api-python/releases
- HubSpot Notes create endpoint: https://developers.hubspot.com/docs/api-reference/latest/crm/activities/notes/guide
- Anthropic Python SDK PyPI: https://pypi.org/project/anthropic/
- Anthropic models overview (canonical API IDs): https://platform.claude.com/docs/en/about-claude/models/overview
- Anthropic prompt caching docs: https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- Anthropic SDK retry source: https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/_base_client.py
- tenacity PyPI / GitHub: https://github.com/jd/tenacity
- requests version history (yanked versions): https://requests.readthedocs.io/en/latest/community/updates/
- beautifulsoup4 latest: https://www.crummy.com/software/BeautifulSoup/bs4/doc/
- tiktoken PyPI: https://pypi.org/project/tiktoken/
- GitHub Actions upload-artifact v4: https://github.com/actions/upload-artifact
- GitHub Actions workflow_dispatch 25 inputs (Dec 2025): https://github.blog/changelog/2025-12-04-actions-workflow-dispatch-workflows-now-support-25-inputs/
- GitHub Actions secrets best practices: https://docs.github.com/en/actions/reference/security/secure-use
- Teams notification in GitHub Actions: https://techresolve.blog/2025/12/16/alerting-on-github-actions-workflow-failures-via-m/
