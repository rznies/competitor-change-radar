# Product Requirements Document: Competitor Change Radar

**Version:** 1.0  
**Date:** 2026-04-25  
**Status:** Draft

---

## Problem Statement

Marketing teams need to monitor competitor websites for messaging and positioning changes, but currently this is done manually by checking competitor pages periodically or relying on ad-hoc observations. This approach has three critical problems:

1. **Reactive not proactive** — Teams only discover changes after they've been live for days or weeks, losing first-mover advantage
2. **Unscalable** — Manual monitoring doesn't scale across dozens of competitors and dozens of landing pages per competitor
3. **No prioritization** — Even when changes are noticed, there's no systematic way to assess their significance (pricing change vs. minor copy tweak)

Marketing teams need an automated system that continuously monitors competitor pages, detects changes with severity classification, and alerts them immediately when significant changes occur.

---

## Solution

Competitor Change Radar is an automated monitoring service that:

1. **Fetches competitor landing pages** on a scheduled basis
2. **Computes content fingerprints** (SHA-256 hashes of normalized HTML text)
3. **Compares fingerprints** against previously stored baselines to detect changes
4. **Classifies severity** using Jaccard similarity (term overlap) to differentiate major vs. minor changes
5. **Sends email alerts** when changes exceed the configured severity threshold
6. **Maintains change history** so users can review what changed over time

---

## User Stories

### Target Management
1. As a marketing team member, I want to add a competitor URL with a friendly name, so that I can track multiple pages easily
2. As a marketing team member, I want to pause or delete a target, so that I can temporarily stop monitoring pages I'm no longer interested in
3. As a marketing team member, I want to see all my active targets in one list, so that I can review what I'm monitoring

### Manual Scanning
4. As a marketing team member, I want to trigger a baseline scan on a target URL, so that I can establish a new reference point
5. As a marketing team member, I want to trigger a check scan on a target URL, so that I can see the current state and compare it to baseline

### Automated Monitoring
6. As a marketing team member, I want the system to automatically check all my targets on a schedule (every 6 hours), so that I don't have to manually trigger scans
7. As a marketing team member, I want to configure the check frequency per target, so that high-priority pages are checked more often

### Change Detection
8. As a marketing team member, I want the system to detect content changes by comparing hash fingerprints, so that even subtle text changes are caught
9. As a marketing team member, I want severity classification (high/medium/low/none) based on how much content changed, so that I can prioritize my response
10. As a marketing team member, I want the system to show me a similarity score (0-1), so that I understand how radical the change is

### Alerting
11. As a marketing team member, I want to receive email alerts when changes are detected, so that I know immediately without checking the dashboard
12. As a marketing team member, I want to configure alert threshold (only high, or high+medium), so that I'm not overwhelmed with low-priority notifications
13. As a marketing team member, I want email alerts to include the target name, URL, severity, and a brief excerpt of what changed, so that I can quickly assess if it's relevant

### History & Analytics
14. As a marketing team member, I want to see a timeline of all changes for a target, so that I can understand the change history
15. As a marketing team member, I want to see a summary of how many changes were detected this month, so that I can report on competitor activity
16. As a marketing team member, I want to click into a specific change event to see details (what text changed, similarity score), so that I can understand the impact

### Settings & Configuration
17. As a user, I want to configure my alert email address, so that alerts go to the right inbox
18. As a user, I want to choose my default alert threshold, so that I only get notified for changes I care about
19. As a user, I want to set my check frequency (hourly, 6-hourly, daily), so that I can balance coverage and resource usage

### Onboarding & Support
20. As a new user, I want guided setup to add my first competitor target, so that I can get started quickly
21. As a new user, I want to see sample data to understand how the product works, so that I can verify it's right for my needs

---

## Implementation Decisions

### Architecture

**Stack:**
- Frontend: Next.js 14 with App Router
- Styling: Tailwind CSS + ShadCN/UI components
- Backend: n8n Cloud (workflows for automation)
- Persistence: PostgreSQL (Neon) via Next.js API routes
- Authentication: NextAuth.js with email magic links

**Pattern:**
- User-facing app runs on Next.js
- Next.js API routes act as BFF (Backend for Frontend)
- API routes call n8n webhooks for heavy lifting (fetching, hashing, comparison)
- n8n workflows are stateless; state stored in PostgreSQL via Next.js

### Modules

1. **Targets Module**
   - `POST /api/targets` — Create target
   - `GET /api/targets` — List targets
   - `GET /api/targets/:id` — Get target details
   - `PATCH /api/targets/:id` — Update target
   - `DELETE /api/targets/:id` — Delete target
   - `POST /api/targets/:id/baseline` — Trigger baseline scan
   - `POST /api/targets/:id/check` — Trigger check scan

2. **Scans Module**
   - `POST /api/scans` — Trigger scan (called by n8n or frontend)
   - `GET /api/targets/:id/scans` — Scan history for target

3. **Events Module**
   - `GET /api/events` — List change events (with filters)
   - `GET /api/events/:id` — Event details

4. **Scheduler Module**
   - n8n workflow: Scheduled Trigger (every 6 hours by default)
   - Queries active targets → calls `/api/targets/:id/check` → stores result → triggers alert if severity > threshold

5. **Alerting Module**
   - n8n workflow: On change detected, send email via SendGrid/Postmark
   - Alert threshold configurable (high only, or high+medium)

### API Contract

**Request Schema (v1):**
```json
{
  "action": "baseline" | "check",
  "targets": [
    {
      "id": "string (optional, auto-generated if omitted)",
      "url": "string (required)",
      "previousHash": "string (optional, for check action)",
      "previousText": "string (optional, for similarity calculation)"
    }
  ]
}
```

**Response Schema (v1):**
```json
{
  "ok": boolean,
  "contractVersion": "radar.v1",
  "action": "baseline" | "check",
  "totalTargets": number,
  "successCount": number,
  "failureCount": number,
  "changedCount": number,
  "results": [
    {
      "id": "string",
      "url": "string",
      "ok": boolean,
      "action": "string",
      "fetchedAt": "ISO8601",
      "statusCode": number,
      "contentHash": "string",
      "contentLength": number,
      "title": "string | null",
      "excerpt": "string (first 400 chars)",
      "changed": boolean,
      "similarity": "number (0-1, null for baseline)",
      "severity": "none" | "low" | "medium" | "high",
      "recommendation": "string"
    }
  ],
  "generatedAt": "ISO8601"
}
```

### Severity Classification Rules

| Similarity Score | Severity | Alert Behavior |
|---------------|----------|--------------|
| 1.0 (no change) | none | No alert |
| > 0.80 | low | No alert (by default) |
| 0.55 - 0.80 | medium | Alert if threshold includes medium |
| < 0.55 | high | Always alert |

### Database Schema

**Table: targets**
```sql
id          UUID PRIMARY KEY,
user_id     UUID NOT NULL REFERENCES users(id),
name        VARCHAR(255) NOT NULL,
url         VARCHAR(2048) NOT NULL,
is_active   BOOLEAN DEFAULT TRUE,
check_freq  VARCHAR(20) DEFAULT '6h',
alert_threshold VARCHAR(20) DEFAULT 'medium',
last_hash   VARCHAR(64),
last_checked_at TIMESTAMP,
created_at  TIMESTAMP DEFAULT NOW(),
updated_at  TIMESTAMP DEFAULT NOW()
```

**Table: snapshots**
```sql
id          BIGSERIAL PRIMARY KEY,
target_id   UUID NOT NULL REFERENCES targets(id),
action      VARCHAR(20) NOT NULL,
fetched_at   TIMESTAMP NOT NULL,
status_code INTEGER,
content_hash VARCHAR(64),
content_length INTEGER,
title       VARCHAR(500),
excerpt     TEXT,
changed    BOOLEAN,
similarity  NUMERIC(4,3),
severity    VARCHAR(20),
ok         BOOLEAN,
created_at  TIMESTAMP DEFAULT NOW()
```

**Table: events**
```sql
id          BIGSERIAL PRIMARY KEY,
target_id   UUID NOT NULL REFERENCES targets(id),
target_url VARCHAR(2048),
event_type  VARCHAR(50),
severity   VARCHAR(20),
changed   BOOLEAN,
content_hash VARCHAR(64),
previous_hash VARCHAR(64),
similarity  NUMERIC(4,3),
detected_at TIMESTAMP NOT NULL,
message    TEXT,
created_at TIMESTAMP DEFAULT NOW()
```

### n8n Workflow Design

**Workflow 1: Radar API (Webhook)**
- Trigger: Webhook (POST /webhook/radar)
- Nodes: Prepare → Fetch → Merge → Analyze → Return JSON
- Purpose: Serves API requests from Next.js

**Workflow 2: Scheduled Monitor (Cron)**
- Trigger: Schedule (every 6h default)
- Nodes: Get Active Targets → For Each → Call Radar API (check) → Filter (changed) → Send Email Alert
- Purpose: Automated monitoring

### Alert Email Content

```
Subject: [Competitor Radar] {severity}: {target_name} changed

{severity} change detected on {target_name}

Target: {url}
Severity: {severity}
Similarity: {similarity_score}%
Detected: {detected_at}

Excerpt:
{excerpt}

View details: {dashboard_url}/targets/{target_id}
```

### Out of Scope

- SMS / WhatsApp / Slack alerts (v1)
- Multi-user teams with role-based access
- Browser extension
- Mobile app
- White-label / reselling
- Competitor pricing extraction (structured data)
- AI-generated insights / summary of changes
- Webhook integrations for third-party tools
- API rate limiting / quotas for free tier
- Billing / subscription management (future)
- SSO / SAML authentication

### Further Notes

- **Freemium Model:** Free tier includes 3 targets, 6-hourly checks, email alerts for high severity only. Paid tier (future) includes unlimited targets, custom check frequency, multi-channel alerts, history retention.

- **Change Detection Approach:** Normalize HTML by stripping `<script>`, `<style>`, `<noscript>`, comments, and all tags. Compute Jaccard similarity on tokenized text. This captures meaningful content changes while ignoring HTML structure changes.

- **Why Next.js + n8n:** n8n handles the unpredictable automation work (scheduled jobs, webhooks, external APIs) very well with low code. Next.js provides the user-facing app with proper auth, database, and custom UI. Using n8n Data Tables is blocked by SDK issues, so PostgreSQL via Next.js is the viable path.

- **Verification:** The core API already works and has been tested. The working endpoint: `POST https://rznies.app.n8n.cloud/webhook/competitor-radar`