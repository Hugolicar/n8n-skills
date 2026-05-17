---
name: n8n-audit-security
description: |
  Security audit for n8n workflows. Scans for exposed credentials, hardcoded secrets,
  unvalidated inputs, missing HTTPS, SQL injection risks, and insecure HTTP nodes.
  Use when: reviewing workflows for security compliance, before production deploy,
  or when credentials may have been exposed in workflow JSON.
---

# n8n Audit Security

## Purpose

Perform security audits on n8n workflows to identify vulnerabilities before they reach production.

## Security Checklist

### 1. Credentials & Secrets

| Check | What to look for | Risk |
|-------|-----------------|------|
| Hardcoded API keys | Any `apiKey`, `token`, `password` in node parameters | Critical |
| Basic Auth in URL | `https://user:pass@host.com` | Critical |
| Exposed OAuth tokens | Token values in HTTP Request headers | Critical |
| Database passwords | Plaintext connection strings | Critical |
| n8n credentials node | Verify `n8n-nodes-base.credentials` is used instead | Best practice |

### 2. Input Validation

| Check | What to look for | Risk |
|-------|-----------------|------|
| No validation on webhook inputs | Webhook/Wait node accepts any payload | High |
| Direct SQL concatenation | Code node building SQL with `$json.field` | Critical |
| Unescaped output | Data passed to HTTP/Email without sanitization | Medium |
| File upload paths | User-controlled paths in Write Binary File | High |

### 3. Network Security

| Check | What to look for | Risk |
|-------|-----------------|------|
| HTTP instead of HTTPS | URLs starting with `http://` (not https) | Medium |
| Missing TLS verification | `rejectUnauthorized: false` | High |
| Exposed internal IPs | Calls to `10.x.x.x`, `192.168.x.x`, `127.0.0.1` | Medium |
| Open webhook endpoints | No authentication on Webhook/Wait nodes | High |

### 4. Node-Specific Risks

**Code Node:**
- `eval()` or `new Function()` usage
- `require()` loading untrusted modules
- File system access (`fs.readFile`, `fs.writeFile`)
- Child process execution (`exec`, `spawn`)

**HTTP Request Node:**
- Redirect following without validation
- Custom headers with sensitive data
- Response body parsed as executable code

**Postgres/MySQL Node:**
- Query parameters interpolated directly
- No prepared statements / parameterized queries

**Webhook Node:**
- No API key / token validation
- Missing IP whitelist
- Responds with sensitive data

## Audit Report Format

When auditing a workflow, produce:

```markdown
## Security Audit: {workflow-name}

### Score: {X}/100

### Critical Issues ({count})
1. **[SEVERITY]** {issue} @ {node-name} ({node-id})
   - Details: {explanation}
   - Fix: {recommendation}

### Warnings ({count})
1. **[SEVERITY]** {issue} @ {node-name}
   - Details: {explanation}
   - Fix: {recommendation}

### Recommendations
- {list of best practices to implement}
```

## Quick Fixes

| Issue | Fix |
|-------|-----|
| Hardcoded API key | Move to n8n Credentials store |
| SQL injection | Use parameterized queries or Query node with binding |
| Open webhook | Add API key header validation or Basic Auth |
| HTTP URL | Replace with HTTPS |
| Missing input validation | Add IF node checking required fields |

## Team-Specific Rules (Hugo's Team)

- All external API calls MUST use Credentials node
- Webhook endpoints MUST validate `x-api-key` header
- Database connections MUST use connection pool (not individual connects)
- No Code node with `require('fs')` or `require('child_process')`
- Railway PostgreSQL: use `DATABASE_URL` from environment variables only
