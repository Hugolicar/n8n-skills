---
name: n8n-review-workflow
description: |
  Analyze and review existing n8n workflows for quality, performance, stack compliance,
  and best practices. Checks against team standards (Z-API, Kommo, Railway patterns).
  Use when: reviewing a workflow before deployment, debugging, refactoring, or onboarding review.
---

# n8n Review Workflow

## Purpose

Analyze existing n8n workflows for:
- Stack compliance (Z-API, Kommo, Railway patterns)
- Performance optimization opportunities
- Security vulnerabilities
- Code quality and maintainability
- Error handling completeness

## Review Dimensions

### 1. Architecture Review

| Check | Pass Criteria | Priority |
|-------|-------------|----------|
| Node count | < 50 nodes (split into sub-workflows if larger) | High |
| Sub-workflow usage | Reusable logic extracted to sub-workflows | Medium |
| Trigger diversity | Single trigger per workflow (except error handlers) | High |
| Circular references | No loops without exit conditions | Critical |
| Dead nodes | All nodes reachable from trigger | Medium |

### 2. Stack Compliance (Hugo's Team)

#### Z-API Compliance
- [ ] Phone validation before sending
- [ ] Retry pattern implemented (3x with backoff)
- [ ] Message deduplication (store messageId)
- [ ] Rate limit awareness (429 handling)
- [ ] Error workflow linked

#### Kommo Compliance
- [ ] Webhook responds within 2 seconds (HTTP 200)
- [ ] Deduplication check for leads/contacts
- [ ] Custom field IDs verified (not hardcoded names)
- [ ] Pipeline/status IDs used (not names)
- [ ] Webhook failure handling (retries, disable prevention)

#### Railway Compliance
- [ ] `DATABASE_URL` from env vars only
- [ ] SSL enabled for PostgreSQL
- [ ] Parameterized queries (no SQL injection)
- [ ] Connection pooling configured
- [ ] Healthcheck endpoint responding

### 3. Error Handling Review

| Check | What to verify |
|-------|---------------|
| Error workflow configured | Workflow settings → Error Workflow is set |
| Per-node error handling | Continue/Retry on error configured where appropriate |
| Fallback paths | IF nodes have else branches for failure cases |
| Notification on failure | Admin notified on critical errors |
| Logging | Errors logged with context (execution ID, node, payload) |
| Circuit breaker | Repeated failures trigger alternative path |

### 4. Performance Review

| Check | Threshold | Action if failing |
|-------|-----------|-------------------|
| Execution time | < 30 seconds for webhook workflows | Optimize slow nodes, add caching |
| Node count per path | < 20 nodes in critical path | Extract to sub-workflow |
| Database queries | < 10 queries per execution | Batch operations, cache results |
| HTTP requests | < 5 sequential requests | Parallelize where possible |
| Memory usage | No Code node processing >10MB arrays | Stream/paginate large datasets |

### 5. Naming & Documentation

| Check | Standard |
|-------|----------|
| Workflow name | `system-action-detail` (e.g., `zapi-inbound-message`) |
| Node names | Descriptive, no defaults (e.g., "Validate Phone" not "IF") |
| Notes on complex nodes | Description explaining logic |
| Workflow tags | `production`, `dev`, `zapi`, `kommo`, `railway` |
| Documentation | README or notes for non-obvious logic |

## Review Report Format

```markdown
## Workflow Review: {workflow-name}

### Overall Score: {X}/100

### Architecture: {score}/20
- ✅ Strengths: ...
- ⚠️ Issues: ...

### Stack Compliance: {score}/25
- Z-API: {pass/fail} - {details}
- Kommo: {pass/fail} - {details}
- Railway: {pass/fail} - {details}

### Error Handling: {score}/20
- ✅ Coverage: ...
- ⚠️ Gaps: ...

### Performance: {score}/20
- Execution time: {X}ms
- Bottlenecks: ...

### Documentation: {score}/15
- Naming: ...
- Notes: ...

### Critical Issues
1. **[SEVERITY]** {issue} @ {node}
   - Impact: ...
   - Fix: ...

### Recommendations
- {prioritized list}

### Approved for: {dev/staging/production/needs-work}
```

## Common Anti-Patterns

| Anti-Pattern | Why it's bad | Fix |
|-------------|-------------|-----|
| Giant IF chain | Hard to maintain, slow | Use Switch node or sub-workflows |
| No error path | Silent failures | Add Continue on error + notification |
| Hardcoded credentials | Security risk | Use Credentials node or env vars |
| Recursive webhook call | Infinite loops, rate limits | Store processed IDs, check before calling |
| Loading entire table | Memory/timeout | Use pagination, filters, LIMIT |
| Synchronous wait in webhook | Blocks response | Use queue pattern or split workflow |
| No input validation | Injection, crashes | Validate required fields at entry |
| Missing webhook response | Kommo/Z-API retries | Always return HTTP 200 quickly |

## Sub-Workflows Checklist

When reviewing, verify these reusable sub-workflows exist (or should exist):

- [ ] `zapi-error-handler` - Z-API specific error handling
- [ ] `kommo-dedup-v2` - Lead/contact deduplication
- [ ] `railway-db-log` - Database logging standard
- [ ] `generic-retry` - Generic retry with backoff
- [ ] `validate-phone` - Phone number validation (Z-API)
- [ ] `webhook-auth` - Webhook authentication check

## Action Priority

| Priority | When to block deployment |
|----------|------------------------|
| 🔴 Critical | Security vulnerability, infinite loop, data loss risk |
| 🟡 High | Missing error handling, stack non-compliance, performance issue |
| 🟢 Medium | Naming issues, missing documentation, minor optimization |
| 🔵 Low | Style preference, optional refactoring |

## Quick Commands

When asked to review a workflow:
1. Ask for workflow JSON or ID
2. Run through all 5 dimensions
3. Score each dimension
4. List critical issues first
5. Provide actionable fixes with node-level precision
6. Flag for re-review after fixes
