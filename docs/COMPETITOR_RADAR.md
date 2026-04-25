# Competitor Change Radar - Project Summary

## 2026-04-25

## What This Is
A product that monitors competitor landing pages for content changes (pricing, positioning, messaging). Built with Next.js frontend + n8n backend. Users add targets → n8n fetches, hashes, and tracks diffs → alerts on significant changes.

## Architecture
- **Frontend**: Next.js (not yet built)
- **Backend**: n8n Cloud workflows (MCP-managed)
- **Persistence**: n8n Data Tables (built-in)
- **API Pattern**: Next.js acts as BFF (calls n8n webhooks), not client→n8n direct

## State: Core API Working, Persistence Blocked

### n8n Workflow (Core)
- **ID**: `h2UW7crrPF9flxDX`
- **Name**: Competitor Change Radar API
- **Webhook**: `POST https://rznies.app.n8n.cloud/webhook/competitor-radar`
- **6 nodes** (core only, without persistence nodes)

### Nodes (working core)
1. `Competitor Radar API` (Webhook trigger)
2. `Prepare Targets` (Code → parse payload)
3. `Fetch Target HTML` (HTTP Request → fetch HTML)
4. `Combine Target And Response` (Merge → join target + response)
5. `Analyze Per Target` (Code → normalize, hash, compare)
6. `Aggregate Response` (Code → return API response)

### Data Tables Created (schema ready, persistence blocked)
| Table | ID | Columns |
|-------|-----|--------|
| radar_targets | `0Wf7dtv4ck3jBbvd` | target_id, url, is_active, last_hash, last_excerpt, last_checked_at, last_status_code, last_severity, updated_at |
| radar_snapshots | `Zs85C1KnOYIxPiCq` | target_id, url, action, fetched_at, status_code, content_hash, content_length, title, excerpt, changed, similarity, severity, recommendation, ok |
| radar_events | `Jq56Lk4WXewUBZBX` | target_id, url, event_type, severity, changed, content_hash, previous_hash, similarity, detected_at, message |

### Contract: `radar.v1`
**Request**:
```json
{
  "action": "baseline" | "check",
  "targets": [{ "id": "string", "url": "https://...", "previousHash": "sha256", "previousText": "normalized text" }]
}
```

**Response**:
```json
{
  "ok": true,
  "contractVersion": "radar.v1",
  "action": "baseline" | "check",
  "totalTargets": 1,
  "successCount": 1,
  "failureCount": 0,
  "changedCount": 0,
  "results": [{
    "id": "string",
    "url": "https://...",
    "ok": true,
    "action": "...",
    "fetchedAt": "ISO8601",
    "statusCode": 200,
    "contentHash": "sha256",
    "contentLength": 142,
    "title": "Page Title",
    "excerpt": "first 400 chars...",
    "changed": false,
    "similarity": 1.0,
    "severity": "none" | "low" | "medium" | "high",
    "recommendation": "Store this hash as baseline."
  }]
}
```

### How It Works
1. **baseline** → fetch + hash + store (no compare)
2. **check** + previousHash → fetch + hash + compare → if different, mark changed + calculate similarity + severity

### Severity Rules
| Similarity Drop | Severity |
|----------------|----------|
| check with same hash | none |
| < 55% similar | high (investigate immediately) |
| 55-80% similar | medium (review within 24h) |
| > 80% similar | low (track only) |

## Test Results (Working)
```
BASELINE:
{
  "ok": true,
  "contractVersion": "radar.v1",
  "action": "baseline",
  "totalTargets": 1,
  "successCount": 1,
  "failureCount": 0,
  "changedCount": 0,
  "results": [{
    "id": "example",
    "url": "https://example.com",
    "ok": true,
    "action": "baseline",
    "fetchedAt": "2026-04-25T15:57:12.961Z",
    "statusCode": 200,
    "contentHash": "d003f90bc10db991b76e6fb480123cfce2cbb2b2784abe687fccccfa7ecacad8",
    "contentLength": 142,
    "title": "Example Domain",
    "excerpt": "Example Domain Example Domain This domain is ...",
    "changed": false,
    "similarity": null,
    "severity": "none",
    "recommendation": "Store this hash as baseline."
  }]
}

CHECK (same target, no change):
{
  "ok": true,
  "contractVersion": "radar.v1",
  "action": "check",
  "totalTargets": 1,
  "successCount": 1,
  "failureCount": 0,
  "changedCount": 0,
  "results": [{
    "id": "example",
    "url": "https://example.com",
    "ok": true,
    "action": "check",
    "statusCode": 200,
    "contentHash": "d003f90bc10db991b76e6fb480123cfce2cbb2b2784abe687fccccfa7ecacad8",
    "changed": false,
    "similarity": 1,
    "severity": "none",
    "recommendation": "No action needed."
  }]
}
```

## Blocked: Persistence
- **Issue**: n8n MCP SDK has schema mismatches for Data Table node `columns` parameter and Set node `assignments` parameter
- Tried 2 approaches: direct `columns` JSON expression (failed), Set node → Data Table (schema errors)
- **Solution needed**: Manually configure persistence nodes in n8n UI (not MCP), or wait for SDK fix

## What's Next
1. **Workaround**: Manually add persistence nodes in n8n UI editor
2. **Test full change detection**: Use a target that actually changes between baseline/check
3. **Add scheduler workflow**: Cron → fetch all active targets → check → alert
4. **Add alerting**: Telegram/Email when severity != none
5. **Build Next.js dashboard**

## Quick Test Commands
```bash
# Test baseline
curl -X POST https://rznies.app.n8n.cloud/webhook/competitor-radar \
  -H "Content-Type: application/json" \
  -d '{"action":"baseline","targets":[{"id":"example","url":"https://example.com"}]}'

# Test check (use hash from baseline response)
curl -X POST https://rznies.app.n8n.cloud/webhook/competitor-radar \
  -H "Content-Type: application/json" \
  -d '{"action":"check","targets":[{"id":"example","url":"https://example.com","previousHash":"HASH_FROM_BASELINE","previousText":"EXCERPT_FROM_BASELINE"}]}'
```

## Key Contacts
- n8n Cloud: https://rznies.app.n8n.cloud
- Workflow ID: `h2UW7crrPF9flxDX`
- Project ID: `9Mx4dv9Bp8GIR8ET` (personal)