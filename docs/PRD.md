# PRD — SentryAPI

## Problem

Developers and small teams often ship APIs without checking them against common security issues (OWASP API Top 10 style problems) — broken access control, missing rate limiting, leaky error messages, weak auth. Manually testing for these takes security expertise most small teams don't have in-house, and commercial scanning tools are expensive or overkill for a single API.

## What it does

SentryAPI takes an API endpoint you own (or have explicit permission to test), runs it through a set of automated security checks, and generates a report of what it found. Each finding is passed to an AI model, which explains the risk in plain English, rates its severity, and suggests a concrete fix — turning raw scan output into something a developer can act on immediately, not just a wall of technical flags.

## Target users

- Solo developers and small teams who want a quick security sanity-check on an API before shipping
- Anyone learning API security who wants plain-English explanations of *why* something is a risk, not just a pass/fail flag

## Core features (v1)

What makes SentryAPI worth using, not just another CRUD app:

- **Automated OWASP-style security scan** — runs 6 real checks against a live endpoint: Broken Access Control, Security Misconfiguration, Injection, Authentication Failures, Mishandling of Exceptional Conditions, Rate Limiting
- **AI-assisted findings** — every failed check comes back with a plain-English explanation of the risk, a severity rating (Low/Medium/High/Critical), and a concrete suggested fix — not just a pass/fail flag
- **Non-blocking scans** — a scan runs in the background; the user isn't stuck waiting on a slow request
- **Security score per report** — a single at-a-glance number summarizing an endpoint's health, tracked across the last 5 scans so a user can see if things are improving or getting worse

Supporting functionality (necessary, but not the point of the product): account signup/login with a one-time authorization confirmation, and manually registering the endpoint URL(s) to be scanned.

## Authorization & legal safety

This tool is for **authorized testing only** — you must own the API you're testing, or have explicit written permission to test it.

- On signup, users must confirm once: *"I confirm I own this API or have explicit written permission to test it, and I am solely responsible for how I use this tool."*
- That confirmation covers the account going forward — users aren't asked to re-confirm every time they add a new endpoint, since that would just be repeated friction without adding real safety.
- Full terms are in the [LICENSE](LICENSE.txt) file — by using this software, responsibility for authorized use rests entirely with the user, not the author.

## Out of scope for v1

- Automatic API/endpoint discovery or crawling
- Support for authentication types beyond what's needed to demo the checks (e.g. no OAuth provider integrations)
- More than 5 stored reports per endpoint
- Team/multi-user accounts (single-user accounts only for v1)

## Success criteria

- A user can sign up, add an endpoint, run a scan, and see a readable AI-generated report end-to-end without errors
- All 6 checks run correctly against a deliberately-vulnerable test API
- Full flow demoable live in under 5 minutes
