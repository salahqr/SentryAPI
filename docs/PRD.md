# PRD — API Health Checkup

## Problem

Small businesses and indie developers ship APIs without ever really knowing if they're solid. They don't have a security team, don't know what "OWASP" means, and don't have time or budget to audit their own infrastructure. Existing scanning tools are built for security engineers — technical, intimidating, and narrowly focused on vulnerabilities only. There's no simple, non-technical way for a business owner to ask "is my API okay?" and get a straight answer.

## What it does

API Health Checkup takes any API's base URL and runs it through a set of automated, fully external checks — no login, no code access, no setup on the business's end. It generates a plain-English report scored across five categories: Security, Performance, Reliability, Data Hygiene, and Documentation. Each finding is passed to an AI model, which explains the issue in everyday language, rates its severity, and suggests a fix — turning raw check results into something anyone can understand and act on, not just developers.

## Target users

- Small business owners and founders who want to know if their API is safe and reliable, without needing technical expertise
- Solo developers and small teams who want a fast sanity-check before shipping
- Anyone learning API best practices who wants plain-English explanations of *why* something matters, not just a pass/fail flag

## Core features (v1)

- **Fully external scan** — user pastes an API base URL, nothing else required. All checks run like a normal client would (read-only HTTP requests) — no credentials, no code access, no infrastructure changes needed
- **8 automated checks across 5 categories:**
  - *Security:* missing/weak authentication, missing security headers, verbose error messages
  - *Performance:* response time
  - *Reliability:* HTTP status code correctness
  - *Data Hygiene:* sensitive/internal field leakage in responses
  - *Documentation:* public API spec detection (Swagger/OpenAPI/docs)
  - *Security (rate control):* missing rate limiting
- **AI-assisted findings** — every flagged issue includes a plain-English explanation, a severity/confidence rating, and a concrete suggested fix
- **Overall Health Score** — one number summarizing the API's health, broken down by category, tracked across the last 5 scans so a user can see improvement or decline over time
- **Confidence-aware reporting** — checks that can't be 100% certain (like whether an open endpoint is *meant* to be public) are flagged as "please confirm" rather than declared bugs outright, to keep the report trustworthy rather than alarmist

Supporting functionality: simple account signup/login, and registering the API URL(s) to be scanned.

## Authorization & responsible use

Since all checks are external, read-only, and non-invasive (no brute-forcing, no injection attempts, no aggressive exploitation), the risk profile is low — but ownership confirmation is still required:

- On signup, users confirm once: *"I confirm I own this API or have explicit permission to test it."*
- This covers the account going forward — no repeated confirmation per scan
- Rate-limiting checks are capped and gently throttled by design, so they can't meaningfully impact a live system

## Testing approach

Checks will be built and validated against a self-built, deliberately-flawed test API (a handful of endpoints with known, intentional issues — no auth on one, no rate limiting on another, a verbose error on a third), not against real third-party accounts or credentials. This gives full control, safe repeated testing, and a clear way to verify each check catches exactly what it should.

## Out of scope for v1

- Automatic API/endpoint discovery or crawling
- Authentication flows beyond what's needed to demo the checks (no OAuth integrations)
- The 3 harder checks (response consistency, payload size flagging, HTTPS/SSL cert checks) — stretch goals only if time allows
- More than 5 stored reports per endpoint
- Team/multi-user accounts (single-user only for v1)

## Tech stack

React (frontend), Laravel/PHP (backend), MySQL (database), Redis (queue for background scans), AI API (finding explanations)

## Success criteria

- A user can sign up, add an API URL, run a scan, and see a readable AI-generated report end-to-end without errors
- All 8 checks run correctly against the self-built test API with known, intentional flaws
- Full flow demoable live in under 5 minutes