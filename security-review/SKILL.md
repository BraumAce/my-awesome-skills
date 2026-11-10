---
name: security-review
description: Security-focused code review. Use when changes touch authentication, user input handling, secrets, API endpoints, payment features, file uploads, or any security-sensitive code. Provides a fast checklist-driven review — detailed code examples are in references/checklists.md.
---

# Security Review

Fast, checklist-driven security review. For detailed code examples and patterns, read `references/checklists.md`.

## When to Activate

Trigger when changed files touch ANY of:
- Authentication or authorization logic
- User input processing or file uploads
- API endpoint definitions or middleware
- Secrets, credentials, or environment variables
- Payment or billing features
- Database queries (especially raw SQL)
- Third-party API integrations
- CORS, CSP, or security header configuration

## Review Process

### 1. Identify Security Surface

```bash
git diff --name-only HEAD | head -30
```

Classify each changed file: auth, input, data, API, config, or other.

### 2. Run Through Checklist

For each changed file, check the applicable items:

#### Secrets Management
- [ ] No hardcoded API keys, tokens, or passwords in source
- [ ] All secrets from environment variables or secret manager
- [ ] `.env*` files in `.gitignore`
- [ ] No secrets in git history (`git log -p --all -S 'password'`)
- [ ] Required secrets validated at startup

#### Input Validation
- [ ] All user input validated with schema (Zod, Pydantic, etc.)
- [ ] File uploads: size limit, type whitelist, extension check
- [ ] No user input directly in queries or commands
- [ ] Whitelist validation, not blacklist
- [ ] Error messages don't leak internals

#### Injection Prevention
- [ ] SQL: parameterized queries only, no string concatenation
- [ ] XSS: user HTML sanitized (DOMPurify or equivalent)
- [ ] Command injection: no user input in shell commands
- [ ] Path traversal: user input not in file paths without sanitization

#### Auth & Session
- [ ] Tokens in httpOnly cookies (not localStorage)
- [ ] Authorization checked before every sensitive operation
- [ ] Session expiration configured
- [ ] CSRF protection on state-changing endpoints
- [ ] RLS enabled (if using Supabase/Postgres)

#### API Security
- [ ] Rate limiting on all endpoints
- [ ] Stricter limits on auth and search endpoints
- [ ] CORS restricted to known origins
- [ ] Security headers set (CSP, X-Frame-Options, HSTS)
- [ ] Error responses generic (no stack traces to client)

#### Data Protection
- [ ] No passwords, tokens, or PII in logs
- [ ] Detailed errors server-side only
- [ ] Sensitive data encrypted at rest and in transit
- [ ] HTTPS enforced in production

### 3. Severity Classification

| Severity | Criteria | Action |
|----------|----------|--------|
| CRITICAL | Exploitable now (SQLi, exposed secrets, auth bypass) | Block. Fix immediately. |
| HIGH | Exploitable with effort (weak validation, missing rate limit) | Block. Fix before merge. |
| MEDIUM | Defense-in-depth gap (missing CSP header, verbose errors) | Warn. Fix when possible. |

### 4. Report Format

```
## Security Review

Scope: [files reviewed]

### Findings

[CRITICAL] <issue>
  File: path:line
  Risk: <what can go wrong>
  Fix: <specific remediation>

### Checklist Summary
- Secrets: pass/fail
- Input validation: pass/fail
- Injection prevention: pass/fail
- Auth & session: pass/fail
- API security: pass/fail
- Data protection: pass/fail
```

## Decision Shortcuts

- Changed only CSS/styles/docs → skip security review
- Changed auth logic → ALWAYS review, no exceptions
- Added new API endpoint → check rate limiting + auth + validation
- Added file upload → check size limit + type whitelist + storage path
- Changed DB queries → check for parameterization

## Reference

For detailed code examples (TypeScript, Python, SQL, etc.), see `references/checklists.md`.
