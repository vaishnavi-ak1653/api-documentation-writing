# Sentinel API Reference

The Sentinel REST API lets you start dependency scans, retrieve findings, and manage suppressions programmatically. Use it to integrate Sentinel into CI/CD pipelines, build custom dashboards, or automate remediation workflows.

**Base URL:** `https://api.sentinel-cli.dev/v1`

---

## Contents

- [Authentication](#authentication)
- [Versioning](#versioning)
- [Rate limits](#rate-limits)
- [Endpoints](#endpoints)
  - [Scans](#scans)
  - [Findings](#findings)
  - [Suppressions](#suppressions)
- [Errors](#errors)
- [Examples](#examples)

---

## Authentication

All requests require an API key passed in the `Authorization` header:

```
Authorization: Bearer <your-api-key>
```

Generate an API key in **Settings → API keys**. Keys are scoped to a workspace and expire after 90 days by default.

**Scopes**

| Scope | Description |
|---|---|
| `scans:read` | Read scan results and findings |
| `scans:write` | Start and cancel scans |
| `suppressions:write` | Create and delete suppressions |

---

## Versioning

The API version is included in the URL path (`/v1`). Breaking changes are introduced in new versions with at least 90 days notice. Non-breaking additions (new fields, new endpoints) may be added at any time.

---

## Rate limits

| Plan | Requests per minute |
|---|---|
| Free | 30 |
| Pro | 300 |
| Enterprise | Custom |

Responses include these headers:

```
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 287
X-RateLimit-Reset: 1718800000
```

When the limit is exceeded, the API returns `429 Too Many Requests`. Wait until `X-RateLimit-Reset` (Unix timestamp) before retrying.

---

## Endpoints

### Scans

#### Start a scan

```
POST /scans
```

Starts a new dependency scan for a project. Sentinel detects the manifest type automatically, or you can specify it explicitly.

**Request body**

| Field | Type | Required | Description |
|---|---|---|---|
| `project_id` | string | Yes | The ID of the project to scan |
| `ref` | string | No | Git ref to scan (branch, tag, or commit SHA). Defaults to the project's default branch |
| `manifest` | string | No | Manifest type: `npm`, `pip`, `go`, `nuget`. Auto-detected if omitted |
| `profile` | string | No | Scan profile ID. Applies custom severity thresholds and exclusions |

**Example request**

```bash
curl -X POST https://api.sentinel-cli.dev/v1/scans \
  -H "Authorization: Bearer $SENTINEL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": "proj_4kR9mNx2",
    "ref": "main",
    "manifest": "npm"
  }'
```

**Example response** — `202 Accepted`

```json
{
  "id": "scan_7vQpLw3X",
  "status": "queued",
  "project_id": "proj_4kR9mNx2",
  "ref": "main",
  "manifest": "npm",
  "created_at": "2025-06-20T09:14:32Z",
  "links": {
    "self": "/v1/scans/scan_7vQpLw3X",
    "findings": "/v1/scans/scan_7vQpLw3X/findings"
  }
}
```

---

#### Get a scan

```
GET /scans/{scan_id}
```

Returns the current status and summary of a scan.

**Path parameters**

| Parameter | Type | Description |
|---|---|---|
| `scan_id` | string | The ID of the scan |

**Scan status values**

| Status | Description |
|---|---|
| `queued` | Scan is waiting to start |
| `running` | Scan is in progress |
| `completed` | Scan finished successfully |
| `failed` | Scan encountered an error |
| `cancelled` | Scan was cancelled by the user |

**Example request**

```bash
curl https://api.sentinel-cli.dev/v1/scans/scan_7vQpLw3X \
  -H "Authorization: Bearer $SENTINEL_API_KEY"
```

**Example response** — `200 OK`

```json
{
  "id": "scan_7vQpLw3X",
  "status": "completed",
  "project_id": "proj_4kR9mNx2",
  "ref": "main",
  "manifest": "npm",
  "packages_scanned": 134,
  "summary": {
    "critical": 3,
    "high": 8,
    "medium": 12,
    "low": 47
  },
  "created_at": "2025-06-20T09:14:32Z",
  "completed_at": "2025-06-20T09:15:18Z",
  "links": {
    "self": "/v1/scans/scan_7vQpLw3X",
    "findings": "/v1/scans/scan_7vQpLw3X/findings"
  }
}
```

---

#### List scans

```
GET /scans
```

Returns a paginated list of scans for the authenticated workspace.

**Query parameters**

| Parameter | Type | Description |
|---|---|---|
| `project_id` | string | Filter by project |
| `status` | string | Filter by status: `queued`, `running`, `completed`, `failed` |
| `limit` | integer | Number of results per page. Default: `20`, max: `100` |
| `cursor` | string | Pagination cursor from a previous response |

**Example request**

```bash
curl "https://api.sentinel-cli.dev/v1/scans?project_id=proj_4kR9mNx2&status=completed&limit=5" \
  -H "Authorization: Bearer $SENTINEL_API_KEY"
```

**Example response** — `200 OK`

```json
{
  "data": [
    {
      "id": "scan_7vQpLw3X",
      "status": "completed",
      "project_id": "proj_4kR9mNx2",
      "summary": { "critical": 3, "high": 8, "medium": 12, "low": 47 },
      "created_at": "2025-06-20T09:14:32Z",
      "completed_at": "2025-06-20T09:15:18Z"
    }
  ],
  "pagination": {
    "next_cursor": "cur_Yw9mRn4K",
    "has_more": true
  }
}
```

---

#### Cancel a scan

```
POST /scans/{scan_id}/cancel
```

Cancels a scan that is `queued` or `running`. Has no effect on completed or already-cancelled scans.

**Example request**

```bash
curl -X POST https://api.sentinel-cli.dev/v1/scans/scan_7vQpLw3X/cancel \
  -H "Authorization: Bearer $SENTINEL_API_KEY"
```

**Response** — `200 OK` with the updated scan object.

---

### Findings

#### List findings for a scan

```
GET /scans/{scan_id}/findings
```

Returns all findings from a completed scan.

**Query parameters**

| Parameter | Type | Description |
|---|---|---|
| `severity` | string | Filter by severity: `critical`, `high`, `medium`, `low` |
| `status` | string | Filter by status: `open`, `suppressed`, `fixed` |
| `limit` | integer | Default: `50`, max: `200` |
| `cursor` | string | Pagination cursor |

**Example request**

```bash
curl "https://api.sentinel-cli.dev/v1/scans/scan_7vQpLw3X/findings?severity=critical" \
  -H "Authorization: Bearer $SENTINEL_API_KEY"
```

**Example response** — `200 OK`

```json
{
  "data": [
    {
      "id": "find_2mBvKq9Z",
      "scan_id": "scan_7vQpLw3X",
      "severity": "critical",
      "status": "open",
      "package": {
        "name": "lodash",
        "version": "4.17.15",
        "ecosystem": "npm"
      },
      "vulnerability": {
        "id": "CVE-2021-23337",
        "title": "Command injection via template",
        "description": "Lodash versions prior to 4.17.21 are vulnerable to command injection via the template function when untrusted data is passed to the sourceURL option.",
        "cvss_score": 7.2,
        "references": [
          "https://nvd.nist.gov/vuln/detail/CVE-2021-23337"
        ]
      },
      "fix": {
        "available": true,
        "recommended_version": "4.17.21",
        "is_breaking_change": false
      },
      "created_at": "2025-06-20T09:15:18Z"
    }
  ],
  "pagination": {
    "next_cursor": null,
    "has_more": false
  }
}
```

---

#### Get a finding

```
GET /findings/{finding_id}
```

Returns a single finding by ID, including full vulnerability detail and fix information.

**Example request**

```bash
curl https://api.sentinel-cli.dev/v1/findings/find_2mBvKq9Z \
  -H "Authorization: Bearer $SENTINEL_API_KEY"
```

**Response** — `200 OK` with the finding object (same schema as above).

---

### Suppressions

Use suppressions to mark a finding as intentionally ignored. Suppressed findings are excluded from future scan summaries until the suppression is removed.

#### Create a suppression

```
POST /findings/{finding_id}/suppress
```

**Request body**

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | Yes | Why this finding is suppressed. One of: `false_positive`, `accepted_risk`, `not_applicable` |
| `note` | string | No | Free-text explanation. Visible to teammates and in audit logs |
| `expires_at` | string | No | ISO 8601 datetime. The suppression lifts automatically after this time |

**Example request**

```bash
curl -X POST https://api.sentinel-cli.dev/v1/findings/find_2mBvKq9Z/suppress \
  -H "Authorization: Bearer $SENTINEL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "reason": "not_applicable",
    "note": "We do not use the template function with untrusted input.",
    "expires_at": "2025-12-31T00:00:00Z"
  }'
```

**Example response** — `201 Created`

```json
{
  "id": "supp_9nRkWm1P",
  "finding_id": "find_2mBvKq9Z",
  "reason": "not_applicable",
  "note": "We do not use the template function with untrusted input.",
  "created_by": "user_5jXqAb2N",
  "created_at": "2025-06-20T10:02:44Z",
  "expires_at": "2025-12-31T00:00:00Z"
}
```

---

#### Delete a suppression

```
DELETE /findings/{finding_id}/suppress
```

Removes the suppression. The finding returns to `open` status and appears in future scan summaries.

**Example request**

```bash
curl -X DELETE https://api.sentinel-cli.dev/v1/findings/find_2mBvKq9Z/suppress \
  -H "Authorization: Bearer $SENTINEL_API_KEY"
```

**Response** — `204 No Content`

---

## Errors

All errors follow a consistent shape:

```json
{
  "error": {
    "code": "resource_not_found",
    "message": "No scan found with ID scan_7vQpLw3X.",
    "request_id": "req_3KmNpQ8v"
  }
}
```

Include `request_id` when contacting support — it identifies the exact request in Sentinel's logs.

**HTTP status codes**

| Status | Meaning |
|---|---|
| `400 Bad Request` | The request body is malformed or missing required fields |
| `401 Unauthorized` | The API key is missing, invalid, or expired |
| `403 Forbidden` | The API key does not have the required scope for this action |
| `404 Not Found` | The requested resource does not exist or is not accessible with this key |
| `409 Conflict` | The action conflicts with the current state (e.g., cancelling a completed scan) |
| `422 Unprocessable Entity` | The request is well-formed but contains invalid values |
| `429 Too Many Requests` | Rate limit exceeded. See [Rate limits](#rate-limits) |
| `500 Internal Server Error` | Sentinel encountered an unexpected error. Retry with exponential backoff |

**Error codes**

| Code | Description |
|---|---|
| `invalid_api_key` | The API key is malformed or has been revoked |
| `insufficient_scope` | The API key lacks permission for this action |
| `resource_not_found` | The specified resource does not exist |
| `scan_not_complete` | Attempted to retrieve findings for a scan that has not yet completed |
| `suppression_exists` | A suppression already exists for this finding |
| `rate_limit_exceeded` | Too many requests. Check `X-RateLimit-Reset` to know when to retry |

---

## Examples

### Poll for scan completion

Scans are asynchronous. This example starts a scan and polls until it completes:

```python
import time
import requests

API_KEY = "your-api-key"
BASE_URL = "https://api.sentinel-cli.dev/v1"
HEADERS = {"Authorization": f"Bearer {API_KEY}"}

# Start the scan
resp = requests.post(f"{BASE_URL}/scans", headers=HEADERS, json={
    "project_id": "proj_4kR9mNx2",
    "ref": "main"
})
scan_id = resp.json()["id"]
print(f"Scan started: {scan_id}")

# Poll until complete
while True:
    resp = requests.get(f"{BASE_URL}/scans/{scan_id}", headers=HEADERS)
    scan = resp.json()
    status = scan["status"]
    print(f"Status: {status}")

    if status in ("completed", "failed", "cancelled"):
        break

    time.sleep(5)

if status == "completed":
    summary = scan["summary"]
    print(f"Done — {summary['critical']} critical, {summary['high']} high")
```

---

### Fail a CI build on critical findings

```bash
#!/bin/bash
set -e

SCAN_ID=$(curl -s -X POST "$BASE_URL/scans" \
  -H "Authorization: Bearer $SENTINEL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"project_id": "'"$PROJECT_ID"'", "ref": "'"$GIT_SHA"'"}' \
  | jq -r '.id')

echo "Scan ID: $SCAN_ID"

# Wait for completion (use exponential backoff in production)
while true; do
  STATUS=$(curl -s "$BASE_URL/scans/$SCAN_ID" \
    -H "Authorization: Bearer $SENTINEL_API_KEY" \
    | jq -r '.status')
  [ "$STATUS" = "completed" ] && break
  sleep 5
done

CRITICAL=$(curl -s "$BASE_URL/scans/$SCAN_ID" \
  -H "Authorization: Bearer $SENTINEL_API_KEY" \
  | jq '.summary.critical')

if [ "$CRITICAL" -gt 0 ]; then
  echo "Build failed: $CRITICAL critical vulnerabilities found."
  exit 1
fi

echo "Scan passed."
```