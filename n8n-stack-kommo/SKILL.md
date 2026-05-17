---
name: n8n-stack-kommo
description: |
  Expert knowledge for Kommo CRM integration in n8n workflows. Covers webhook events,
  lead pipeline management, custom fields, deduplication, and API patterns.
  Use when: building or reviewing workflows that connect to Kommo CRM.
---

# n8n Stack: Kommo

## Overview

Kommo (formerly amoCRM) is a CRM with lead-based pipeline management. This skill covers webhook events, API patterns, custom fields, and n8n-specific implementation.

## Authentication

| Method | How | Where |
|--------|-----|-------|
| OAuth 2.0 | Authorization code flow | n8n Credentials (Kommo) |
| API Key | `X-Domain` + `Authorization: Bearer {token}` | HTTP Request headers |

**Base URL:** `https://{subdomain}.kommo.com/api/v4/`

## Webhook Events

### Lead Events

| Event | Webhook action | Key fields |
|-------|---------------|------------|
| Lead created | `leads.add` | `id`, `name`, `status_id`, `pipeline_id`, `price`, `custom_fields` |
| Lead updated | `leads.update` | `id`, `old_status_id` (if status changed), `updated_at` |
| Lead deleted | `leads.delete` | `id`, `status_id`, `pipeline_id` |
| Lead restored | `leads.restore` | Full lead object |
| Status changed | `leads.status` | `old_status_id`, `status_id` |
| Responsible changed | `leads.responsible` | `old_responsible_user_id`, `responsible_user_id` |

### Contact Events

| Event | Webhook action | Key fields |
|-------|---------------|------------|
| Contact created | `contacts.add` | `id`, `name`, `custom_fields` (phone, email), `type: "contact"` |
| Contact updated | `contacts.update` | `id`, `updated_at`, `custom_fields` |
| Contact deleted | `contacts.delete` | `id`, `type: "contact"` |
| Responsible changed | `contacts.responsible` | `old_responsible_user_id` |

### Company Events

Same structure as contacts but with `type: "company"` and `linked_leads_id`.

### Task Events

| Event | Webhook action | Key fields |
|-------|---------------|------------|
| Task created | `task.add` | `id`, `element_id`, `element_type`, `text`, `complete_till` |
| Task updated | `task.update` | `status`, `old_text`, `result` |
| Task deleted | `task.delete` | `id` |

### Incoming Messages & Conversations

| Event | Webhook action | Key fields |
|-------|---------------|------------|
| Message received | `message.add` | `chat_id`, `contact_id`, `text`, `type: "incoming"`, `origin` |
| Conversation created | `talk.add` | `talk_id`, `entity_id`, `is_in_work`, `is_read` |
| Conversation updated | `talk.update` | `is_read`, `is_in_work` (0 = closed, 1 = active) |

### Lead de Entrada (Unsorted)

| Event | Webhook action | Key fields |
|-------|---------------|------------|
| New incoming lead | `unsorted.add` | `uid`, `category`, `source_data`, `data.leads[]` |
| Updated | `unsorted.update` | `uid`, `data.leads[].id` |
| Deleted/Accepted/Declined | `unsorted.delete` | `action: "accept"` or `"decline"`, `accept_result` / `decline_result` |

## Webhook Response Requirements

**Critical:** Kommo expects HTTP 200-299 within **2 seconds**.

If webhook fails:
- 2nd retry: 5 minutes
- 3rd retry: 15 minutes
- 4th retry: 15 minutes (for 4xx/5xx errors)
- 5th retry: 1 hour (for 5xx errors)

**Auto-disable:** After 100 invalid responses in 2 hours, webhook is disabled.

## Custom Fields Reference

Common custom field types in Kommo:

| Type | Code | Values Format |
|------|------|---------------|
| Text | varies | `{ "value": "text" }` |
| Numeric | varies | `{ "value": "123" }` |
| Select/Radio | varies | `{ "value": "2", "enum": "908107" }` |
| Date | varies | `"1729717200"` (timestamp) |
| Money | varies | `{ "value": "200" }` |
| Phone | `PHONE` | `{ "value": "1234567890", "enum": "552476" }` |
| Email | `EMAIL` | `{ "value": "mail@gmail.com", "enum": "552488" }` |

## Lead Deduplication Pattern

Standard team pattern for preventing duplicates:

```
Webhook (Lead created)
  → Get contacts from lead (via API)
  → Extract phone/email
  → Query leads by phone/email
  → IF (existing lead found)
    → Merge data (update existing, delete new)
    → Log merge action
  → ELSE
    → Process new lead normally
```

## Pipeline & Status IDs

Always work with IDs, not names:
- `pipeline_id`: Pipeline identifier
- `status_id`: Stage within pipeline

**Mapping convention:** Store pipeline/status names in a lookup table (Set node or DB) for human-readable logging.

## n8n Implementation Patterns

### Inbound Lead Processing
```
Webhook (Kommo: lead.add)
  → IF (validate secret/token)
  → Function (normalize payload)
  → Deduplication check
  → IF (duplicate)
    → Merge/update existing
  → ELSE
    → Create in external system (Z-API, Email, etc.)
    → Update lead with external ID (optional)
```

### Status Change → Action
```
Webhook (Kommo: leads.status)
  → Extract old_status_id + status_id
  → Lookup status names (for logging)
  → IF (status_id == "QUALIFIED")
    → Trigger qualified workflow
    → Send welcome message via Z-API
    → Create task for sales
  → IF (status_id == "LOST")
    → Log reason
    → Send feedback survey
```

### Conversation Sync
```
Webhook (Kommo: message.add)
  → Check origin (telegram, whatsapp, etc.)
  → IF (from external channel AND not fromMe)
    → Forward to appropriate handler
    → Mark as read in Kommo (update talk)
```

## Team Naming Convention

| Prefix | Use |
|--------|-----|
| `kommo-inbound-{entity}` | Webhook receiving flows |
| `kommo-outbound-{action}` | API write flows |
| `kommo-sync-{system}` | Bidirectional sync workflows |
| `kommo-dedup` | Deduplication logic (reusable) |

## API Limits

| Plan | Rate Limit |
|------|-----------|
| Base | 100 requests/minute |
| Advanced | 200 requests/minute |
| Pro/Enterprise | Higher limits |

## Common Gotchas

| Issue | Cause | Fix |
|-------|-------|-----|
| Webhook disabled | 100+ failures in 2h | Check endpoint; re-enable in settings |
| Custom field not showing | Wrong field ID | Verify field ID via API or settings |
| Lead not updating | Wrong pipeline/status ID | Use IDs, not names |
| Duplicate leads | No dedup logic | Implement phone/email check |
| Message not syncing | Wrong origin filter | Check `origin` field in payload |
| Task not created | `element_type` wrong | 1=contact, 2=lead, 3=company |

## Sub-Workflows to Reuse

**`kommo-dedup-v2`:**
- Input: `{ phone, email, lead_id }`
- Output: `{ isDuplicate: true/false, existingLeadId: "..." }`
- Steps:
  1. Query leads by phone
  2. Query leads by email
  3. Return first match

**`kommo-error-handler`:**
- Input: `{ error, webhookPayload }`
- Actions:
  1. Log to database
  2. If retryable: queue for retry
  3. If auth error: alert admin
  4. Track failure count (disable webhook if >50/hour)
