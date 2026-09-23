# Technical Design — API Health Checkup

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Frontend | React | Matches required stack; dashboard for scan results and history |
| Backend | PHP (Laravel) | Matches required stack; mature queue/job system fits the async scan flow |
| Database | MySQL | Simple, well-supported by Laravel/Eloquent; JSON column type covers the one flexible field needed (`report_data`) |
| Queue & scratch storage | Redis | Backs Laravel Queues (async scans) and doubles as a fast temporary store for in-progress findings — see note below |
| AI | OpenAI or Claude API | Turns raw pass/fail check results into plain-English explanations, severity ratings, and suggested fixes |
| Architecture | Monolith with async workers | Deliberately simple — no microservices, no event-streaming layer |

## System structure

```
┌─────────────┐     ┌──────────────────┐      ┌───────────┐
│   React     │────>│  Laravel API     │─────>│  MySQL    │
│  Frontend   │     │  (Backend)       │      │ (final    │
└─────────────┘     └───────┬──────────┘      │  reports) │
                            │                 └───────────┘
                            ▼
                     Queue Job dispatched
                            │
                            ▼
                     Queue Worker runs Scan Service
                            │
                            ▼
                     Redis (temp findings store,
                     cleared after scan completes)
                            │
                            ▼
                     AI API (batched request,
                     one call per scan)
```

## How a scan flows end-to-end

```
User submits an API base URL + confirms ownership/permission
            │
            ▼
Laravel dispatches a Queue Job (nothing written to DB yet)
            │
            ▼
Queue Worker runs the Scan Service:
  - runs each of the 8 external, read-only checks against the URL
  - no login or credentials used — every check works like a normal client
  - logs progress (and writes each result to Redis as it finishes)
            │
            ▼
Once all checks are done, worker pulls every result back from Redis
            │
            ▼
All check results are sent to the AI API together, in a single batched
request → the AI returns category scores, an overall score, plain-English
explanations, severity ratings, and suggested fixes
            │
            ▼
Worker inserts ONE final report row (raw findings + AI analysis) into
MySQL, clears the scan's temporary Redis keys — oldest report beyond
the last 5 for that endpoint is deleted
            │
            ▼
React dashboard displays: overall Health Score, category breakdown
(Security, Performance, Reliability, Data Hygiene, Documentation),
findings ranked by severity, AI explanations, suggested fixes, and
scan history
```

## Key decisions

**Why Redis, not Kafka:** Redis is used as a fast, temporary scratchpad — one background job writes findings as it goes, then reads them back moments later within that same scan. It's a short-lived, single-consumer need. Kafka's strength is a durable, replayable event log for multiple independent consumers over time — that's a different problem than this project has, so using Redis is the more correctly-scoped choice here.

**Why one batched AI call instead of one call per finding:** fewer API calls means faster overall processing, lower cost, and avoids rate-limit issues that come from firing many small requests back to back.

**Why no in-progress status in the database:** a scan's live state (started, which check is running) is transient and would mean repeated writes for something that disappears once the scan finishes. That's tracked in application logs (and briefly in Redis) instead — the database only stores the final, completed result.

**AI's role is strictly explanatory:** the AI never runs its own checks or invents findings — it only explains, scores, and suggests fixes for what the scanner engine actually detected. The scanning logic stays deterministic and testable; AI is used purely for the "explain it to a human" layer.

**Why every check is external/read-only:** the tool needs to work against any business's API with zero setup — no code access, no credentials, no login. This also removes the legal/safety complexity of deeper testing (no brute-forcing, no injection payloads, no cross-account ID swapping), since nothing sent could meaningfully damage a live system.

**Confidence-aware findings:** checks that can't be 100% certain (e.g. an endpoint returning data without auth — is it meant to be public?) are flagged as "please confirm" rather than declared bugs outright, to keep reports trustworthy rather than alarmist.

## Code architecture

Each check is implemented as its own class following a shared interface (Strategy pattern), so adding a new check doesn't require touching existing ones. A central Scan Service runs all registered checks against a target URL, writing each result to Redis as it completes. Once every check has finished, the Scan Service pulls all results back from Redis and hands them to the AI Analysis Service, which sends them to the AI API in a single batched request and stores the resulting scores/explanations/fixes as the final report.

## API map

| Method | Path | Purpose | Auth required |
|---|---|---|---|
| POST | `/auth/register` | Create account (includes one-time ownership/permission confirmation) | No |
| POST | `/auth/login` | Log in, returns token | No |
| POST | `/auth/logout` | Invalidate current token | Yes |
| POST | `/auth/token/refresh` | Get a new token without re-login | Yes (refresh token) |
| POST | `/auth/password/reset-request` | Request a password reset (sends email/token) | No |
| POST | `/auth/password/reset-confirm` | Set new password using the reset token | No |
| GET | `/urls` | List the current user's registered URLs | Yes |
| GET | `/urls/{id}` | Get details of a single URL | Yes |
| POST | `/urls` | Register a new URL to scan | Yes |
| DELETE | `/urls/{id}` | Remove a registered URL | Yes |
| POST | `/urls/{id}/scan` | Start a scan on this URL — returns immediately (202 Accepted), actual work runs in the background queue | Yes |
| GET | `/urls/{id}/reports` | Get this URL's reports (last 5) | Yes |
| GET | `/reports/{id}` | Get a single report's full findings/AI output | Yes |

## Check implementation details

| Check | Category | How it's implemented |
|---|---|---|
| **Missing/Weak Authentication** | Security | Calls the endpoint with no auth header/token. If it returns 200 with real data instead of 401/403 — especially on URLs like `/users`, `/admin`, `/account`, or responses containing personal data — flags it as likely missing auth. |
| **Missing Security Headers** | Security | Sends one normal request, checks response headers for `Content-Security-Policy`, `X-Content-Type-Options`, `Strict-Transport-Security`, and CORS config. |
| **Verbose Error Messages** | Security | Sends deliberately malformed input, scans the error response text for leak signals: "Exception," "stack trace," file paths, SQL fragments. |
| **Missing Rate Limiting** | Security | Sends a capped burst of requests (20–50, with small delays) — enough to detect throttling without risking impact on a live system. Flags if no 429 is ever returned. |
| **HTTP Status Code Correctness** | Reliability | Sends test cases with known expected outcomes (nonexistent ID, wrong method) and checks if the correct status code (404, 405) comes back. |
| **Response Time** | Performance | Sends the same request 3–5 times, measures response time, flags if average exceeds a threshold. |
| **Sensitive/Internal Field Leakage** | Data Hygiene | Recursively scans JSON response bodies for suspicious key names (`password`, `token`, `internal_id`, `debug`, `ssn`). |
| **API Spec Detection** | Documentation | Checks common paths (`/swagger.json`, `/openapi.json`, `/docs`) for a 200 response. |

## Testing approach

Checks are built and validated against a self-built, deliberately-flawed test API (a handful of endpoints with known, intentional issues — no auth on one, no rate limiting on another, a verbose error on a third) — not against real third-party accounts or credentials. This gives full control, safe repeated testing, and a clear way to verify each check catches exactly what it should.