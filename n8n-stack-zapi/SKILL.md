---
name: n8n-stack-zapi
description: |
  Expert knowledge for Z-API integration in n8n workflows. Covers webhook patterns, 
  retry policies, rate limits, payload validation, and connection-specific gotchas.
  Use when: building or reviewing workflows that connect to Z-API (WhatsApp Business API).
---

# n8n Stack: Z-API

## Overview

Z-API is a WhatsApp Business API provider. This skill covers patterns, pitfalls, and best practices for integrating Z-API with n8n.

## Authentication

| Method | How | Where |
|--------|-----|-------|
| Instance Token | `token` + `security-token` headers | HTTP Request node headers |
| Instance ID | Part of URL path | `https://api.z-api.io/instances/{instanceId}/...` |

**Critical:** Never hardcode tokens. Use n8n Credentials or environment variables.

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| Send message | ~20/min per instance (varies by plan) |
| Status check | ~60/min |
| Webhook setup | ~10/min |

**Pattern:** Always implement exponential backoff when hitting 429 responses.

## Retry Policy (Team Standard)

```
Max retries: 3
Backoff: 2s → 4s → 8s
On 429: wait 60s then retry
On 401/403: fail immediately (auth issue)
On 500/502/503: retry with backoff
```

Implement in n8n:
- HTTP Request node → On Error → Wait → Loop back
- Or use Error Workflow with retry logic

## Webhook Patterns

### Receiving Messages (Inbound)

Z-API sends webhooks to your endpoint when:
- Message received
- Message status update (delivered, read, failed)
- Instance connection status changes

**Webhook payload structure (message received):**
```json
{
  "event": "ONMESSAGE",
  "instanceId": "YOUR_INSTANCE_ID",
  "data": {
    "messageId": "...",
    "phone": "5511999999999",
    "message": "Hello",
    "type": "text",
    "timestamp": 1234567890,
    "fromMe": false,
    "chatId": "5511999999999@c.us"
  }
}
```

### Sending Messages (Outbound)

**Text message:**
```
POST https://api.z-api.io/instances/{instanceId}/token/{token}/send-text
{
  "phone": "5511999999999",
  "message": "Hello from n8n"
}
```

**Image/Document:**
```
POST https://api.z-api.io/instances/{instanceId}/token/{token}/send-image
{
  "phone": "5511999999999",
  "image": "base64_or_url",
  "caption": "Optional caption"
}
```

## Common Gotchas

| Issue | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Token expired or wrong instance | Check token in Z-API dashboard |
| 429 Too Many Requests | Rate limit hit | Implement backoff; check plan limits |
| Message not delivered | Phone not on WhatsApp | Validate phone before sending |
| Webhook not firing | Instance disconnected | Check instance status; reconnect if needed |
| Double messages | Webhook retry + no dedup | Store messageId in DB; skip if exists |
| Encoding issues | Special chars in message | URL-encode or use base64 for media |

## n8n Implementation Pattern

### Standard Inbound Flow
```
Webhook (Z-API incoming)
  → IF (validate webhook token)
  → Function (parse payload)
  → IF (check if messageId already processed)
    → YES: Stop
    → NO: Process → Store messageId → Continue
```

### Standard Outbound Flow
```
Trigger (CRM update / scheduled)
  → Get data
  → IF (phone valid?)
    → HTTP Request (Z-API send)
    → IF (success)
      → Log sent
    → IF (429)
      → Wait (exponential) → Retry
    → IF (fail)
      → Log error → Notify admin
```

## Validation Rules

**Phone number validation:**
- Must be E.164 format: `+{country}{number}`
- Remove non-digits before sending
- Brazil: `+55` + 11 digits (DDD + number)
- Strip `+`, spaces, `-`, `()` before API call if needed

**Message size limits:**
- Text: 4096 chars
- Image: 16MB
- Video: 16MB
- Document: 100MB

## Team Naming Convention

| Prefix | Use |
|--------|-----|
| `zapi-inbound-{action}` | Webhook receiving flows |
| `zapi-outbound-{action}` | Sending message flows |
| `zapi-status-{type}` | Connection monitoring |
| `zapi-retry-{context}` | Retry/error handling sub-workflows |

## Error Handling Sub-Workflow

Reuse across all Z-API workflows:
- Name: `zapi-error-handler`
- Input: `{ error, context, retryCount }`
- Actions:
  1. Log to database/Notion
  2. If retryCount < 3: Wait → Re-trigger
  3. If retryCount >= 3: Notify Slack/Email admin
  4. Update instance status in monitoring

## Monitoring

Check instance health:
```
GET https://api.z-api.io/instances/{instanceId}/token/{token}/status
```

Expected response:
```json
{
  "connected": true,
  "qrCodeNeeded": false,
  "batteryLevel": 85
}
```

**Alert when:**
- `connected: false` for >5 minutes
- `qrCodeNeeded: true` (needs re-authentication)
- `batteryLevel` < 20 (if applicable)
