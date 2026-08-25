---
name: api-readiness-check
description: Systematically test all APIs required by cron jobs or services. Produces a status table showing what's working, what's blocked, and what needs fixing. Includes known API quirks for common services.
version: 1.1.0
author: Hermes Agent
category: devops
metadata:
  hermes:
    tags: [api, testing, readiness, devops, cron]
triggers:
  - "test the API"
  - "is the API working"
  - "check API readiness"
  - "test all APIs"
  - "API status"
  - "are the APIs working"
---

# API Readiness Check

Systematic testing of all external APIs used by cron jobs and services. Tests connectivity, authentication, and endpoint functionality. Produces an actionable status table.

## Workflow

### 1. Discover What to Test
- List all cron jobs: `cronjob(action='list')`
- Load the credentials/config file (e.g. a JSON file with API keys per service)
- Identify API dependencies per job (marketplace APIs, ERP, proxy services, social platforms, etc.)

### 2. Test Each API
Use a Python script that tests each API. Follow these rules:

**Always check the response body, not just HTTP status.** Many APIs return HTTP 200 with error messages in the body (e.g. marketplace APIs returning `AppWhiteIpLimit` or similar body-level errors).

**Handle these known quirks:**

| API | Quirk | Fix |
|-----|-------|-----|
| **ERPNext** | HTTP 417 on some endpoints | Remove `Expect` header from requests; use custom opener. Ping `/api/method/ping` first. |
| **ERPNext** | Resources with spaces (e.g. "Sales Order") | URL-encode resource names: `urllib.parse.quote("Sales Order", safe='')` |
| **Lazada** | HTTP 200 but body has `AppWhiteIpLimit` | Check `body.code` field, not just HTTP status |
| **Lazada** | HTTP 200 but `InvalidApiPath` after IP fix | App needs API permissions authorized in the marketplace Open Platform console (API Authorization tab) |
| **Lazada** | HMAC-SHA256 signing | Sort params alphabetically; concat k+v; sign with app_secret; uppercase hex digest |
| **Proxy services (Maton-style)** | Domain is `api.<name>.ai` NOT `api.<name>.com` | Test DNS resolution first if unsure |
| **Proxy services** | URL format: `/app-name/native-api-path` | e.g. `/airtable/v0/meta/bases`, `/notion/v1/users/me` |
| **Proxy services** | Connection status: ACTIVE vs PENDING | PENDING connections return 400: "Your connections for `{app}` are either PENDING or FAILED" |
| **Proxy services** | List all connections | `GET /connections` returns all connections with status, metadata, creation time |
| **Google Drive via proxy** | Shared drive diagnosis: 4-step checklist | 1. `GET /about?fields=user` -> shows connected email. 2. `GET /drives` -> empty `drives:[]` means no shared drive access. 3. `GET /files?q=name+contains+'keyword'` -> empty means files aren't visible. 4. `GET /files/{fileId}?supportsAllDrives=true` -> 404 on known IDs confirms wrong-account issue. |
| **Google Drive via proxy** | `/about` returns personal email but all file IDs return 404 | The connected account is a personal Gmail, not the Workspace account that owns the shared drive. Fix: re-authorize with the correct account. |
| **Google Drive via proxy** | Downloaded file is tiny (e.g. 342 bytes) containing JSON error | Not a real download - the file is invisible. Check the response body. A valid download is MB-sized; a small JSON response means `"error": {"code": 404, "message": "File not found: ..."}` |
| **Terminal security scanner** | Blocks `curl \| python3` pipes | Save to file first: `curl ... -o /tmp/out.json` then `python3 ... /tmp/out.json`. The scanner refuses pipes from download commands to interpreters. Always use intermediate files or Python's `requests` library instead of chaining with pipes. |
| **Airtable** | Write records | `POST /v0/{baseId}/{tableId}` with `{"records": [{"fields": {...}}]}` body |
| **Airtable** | Delete records | `DELETE /v0/{baseId}/{tableId}/{recordId}` |
| **Airtable** | Field name mismatch | Unknown field names return HTTP 422 - get field names from a read first |
| **Airtable** | PAT needs `schema.bases:read` scope | Without it, `/v0/meta/bases` returns 403 |
| **TikTok Shop** | Auth endpoint: `auth.tiktok-shops.com/api/v2/token/get` | Sign with HMAC-SHA256 of app_key+timestamp |
| **Facebook Graph** | Token may be in env vars or config files | Check env vars (`FB_ACCESS_TOKEN`, `FACEBOOK_ACCESS_TOKEN`) and config paths |
| **Google OAuth** | HTTP 400 `invalid_grant` is expected without refresh token | Means client_id/secret are valid but no stored token yet |

### 3. Test DNS Resolution for Unknown Services
If an API domain doesn't resolve, try variations:
```python
import socket
candidates = ['api.example.com', 'api.example.org', 'app.example.com']
for domain in candidates:
    try:
        ip = socket.gethostbyname(domain)
        print(f"{domain} -> {ip} OK")
    except Exception:
        print(f"{domain} -> NOT FOUND")
```

### 4. Produce Status Table
After all tests, produce a table with:
- API name
- Status: WORKING / BLOCKED / BROKEN / MISSING / PENDING
- What needs to be fixed
- Impact: which cron jobs are affected

### 5. If Credentials Are Truncated
If a credential in the config file shows a truncated value (e.g. `"v2.7ZD...6VlI"`), the user may provide the full key. Use Python string replacement to update the file, then re-test.

### 6. Verify Full CRUD for Data APIs
When a proxy shows a service as ACTIVE, test full read/write/delete:
1. `GET /.../records?maxRecords=1` - verify reads work
2. `POST /...` with a test record - verify writes work
3. `DELETE /.../{recordId}` - clean up test record
4. If POST returns 422, the field names are wrong - use field names from the GET response

## Rate-Limit Counter Pattern

When using a limited free tier (e.g. 10,000 calls/month):
1. Create counter state file (e.g. `counter.json`)
2. Create pre-check script that injects the API key and blocks if capped
3. Create increment script to call after each API call
4. Wire pre-check script into cron job with `cronjob(action='update', script='counter.py')`
5. Update job prompt to instruct calling the increment script after each API call
6. Cap at 90% of the limit (buffer), auto-reset monthly

## Pitfalls
- **HTTP 200 != Working**: Many APIs return 200 with error messages in the body
- **DNS failures != dead service**: Try alternative domains before concluding
- **Truncated creds**: The patch tool may fail on truncated strings containing `...` - use Python's `str.replace()` instead
- **ERPNext 417**: Only affects some endpoints (e.g. `get_user_info`); core endpoints like `ping`, `Sales Order`, `Item` work fine with the header fix
- **Cron job scripts must use relative paths**: Scripts must be in the scripts dir and referenced by filename only
- **Cron job `last_status: ok` does not mean the job produced output**: The agent may report success even when the referenced script doesn't exist. See the cron-prompt-template skill for the full failure-mode table.
- **Security scanners block `curl | python3` pipes**: The scanner detects pipes from download commands to interpreters and blocks. Always save curl output to a temp file first, then process with a separate call or use Python's `requests` library. This affects API test scripts - don't use one-liner pipes for API testing.
