---
name: x402geo-skills
description: Use when an agent needs to run GEO/SEO audits through x402geo.com with payment gating, status tracking, and report delivery via MCP or ACP.
---

# x402geo MCP + ACP Skill

## Purpose

Use this skill to run a full user-facing GEO/SEO audit flow on https://x402geo.com:

1. Collect user email + target URL.
2. Create a payment-required audit request.
3. Return payment link and audit redirect/progress URL to the user.
4. Track job progress until completion.
5. Deliver report summary and public report link.

## Integration Modes

Use one of these modes:

- MCP: `POST https://x402geo.com/api/mcp` (JSON-RPC 2.0 tools)
- ACP: `https://x402geo.com/api/acp/*` REST endpoints

If your agent supports tool-calling well, prefer MCP.
If your agent is workflow/REST oriented, prefer ACP.

## Required User Inputs

Collect these first:

- `email`: user email for payment and audit association
- `url`: full website URL to audit (for example `https://example.com`)

## ACP Flow (Recommended for explicit payment lifecycle)

### Step 1: Create payment request

Request:

```http
POST https://x402geo.com/api/acp/jobs
Content-Type: application/json

{
  "email": "user@example.com",
  "requirements": {
    "url": "https://example.com",
    "report_depth": "standard"
  }
}
```

Expected response fields:

- `status` = `payment_required`
- `checkout_url` (payment link)
- `email`
- `target_url`

Agent action:

- Return `checkout_url` to user.
- Tell user to complete payment.
- Tell user this starts the audit flow and leads to an audit tracking URL.

### Step 2: Start paid audit

After payment (or directly in local/dev setups where paywall is disabled), call:

```http
GET https://x402geo.com/api/acp/jobs/start?url=https%3A%2F%2Fexample.com&email=user%40example.com
```

Expected response fields:

- `job_id`
- `progress_url`
- `results_url`
- `report_url`
- `status`

Agent action:

- Return `progress_url` as the audit status URL.
- Keep `job_id` for tracking.
- Return `report_url` as the final public report link placeholder.

### Step 3: Track progress

Poll either endpoint until complete:

- `GET /api/acp/jobs/{jobId}`
- `GET /api/acp/resources/audit-status?jobId={jobId}`

State mapping:

- `pending` / `in_progress`: keep polling
- `completed`: proceed to deliverable fetch
- `failed`: stop and report error_reason to user

### Step 4: Read and return report

Fetch deliverable:

```http
GET https://x402geo.com/api/acp/jobs/{jobId}/deliverable
```

Notes:

- Returns `409` while audit is not complete.
- Returns `deliverable` object when complete.

Agent action:

- Summarize key findings and recommendations for user.
- Provide `report_url` from status/start response as the public report link.

## MCP Flow (Tool-calling agents)

Endpoint:

- `POST https://x402geo.com/api/mcp`

Protocol methods:

- `initialize`
- `tools/list`
- `tools/call`

Primary tools:

- `create_geo_audit` with `url` and optional `email`
- `get_audit_status` with `jobId`
- `get_report_links` with `jobId`

MCP workflow:

1. Call `create_geo_audit`.
2. If `status=payment_required`, return `paymentUrl` to user and ask user to complete payment.
3. After payment, get `jobId` from redirected status URL and call `get_audit_status`.
4. When `done`, call `get_report_links` and return `reportPage`.

## Agent Output Contract

At minimum, send user:

- Payment step: `checkout_url` or `paymentUrl`
- Tracking step: `progress_url` or status page URL
- Completion step: concise summary + public `report_url`/`reportPage`

If processing fails, send:

- current status
- provider error message
- exact next action user should take
