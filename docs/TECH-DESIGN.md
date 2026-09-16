# Technical Design — SentryAPI

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Frontend | React | Matches required stack; dashboard for scan results and history |
| Backend | PHP (Laravel) | Matches required stack; mature queue/job system fits the async scan flow |
| Database | MySQL | Simple, well-supported by Laravel/Eloquent; JSON column type covers the one flexible field needed (`report_data`) |
| Queue & scratch storage | Redis | Backs Laravel Queues (async scans) and doubles as a fast temporary store for in-progress findings — see note below |
| AI | OpenAI or Claude API | Turns raw pass/fail scan results into plain-English explanations, severity ratings, and suggested fixes |
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
User submits endpoint + confirms authorization
            │
            ▼
Laravel dispatches a Queue Job (nothing written to DB yet)
            │
            ▼
Queue Worker runs the Scan Service:
  - runs each registered security check against the endpoint
  - logs progress (and writes each result to Redis as it finishes)
            │
            ▼
Once all checks are done, worker pulls every result back from Redis
            │
            ▼
All failed checks are sent to the AI API together, in a single batched
request → explanation + severity + suggested fix returned for each
            │
            ▼
Worker inserts ONE final report row (raw findings + AI analysis) into
MySQL, clears the scan's temporary Redis keys — oldest report beyond
the last 5 for that endpoint is deleted
            │
            ▼
React dashboard displays: security score, findings ranked by severity,
AI explanations, suggested fixes, and scan history
```

## Key decisions

**Why Redis, not Kafka:** Redis is used as a fast, temporary scratchpad — one background job writes findings as it goes, then reads them back moments later within that same scan. It's a short-lived, single-consumer need. Kafka's strength is a durable, replayable event log for multiple independent consumers over time — that's a different problem than this project has, so using Redis is the more correctly-scoped choice here.

**Why one batched AI call instead of one call per finding:** fewer API calls means faster overall processing, lower cost, and avoids rate-limit issues that come from firing many small requests back to back.

**Why no in-progress status in the database:** a scan's live state (started, which check is running) is transient and would mean repeated writes for something that disappears once the scan finishes. That's tracked in application logs (and briefly in Redis) instead — the database only stores the final, completed result.

**AI's role is strictly explanatory:** the AI never runs its own checks or invents findings — it only explains and suggests fixes for what the scanner engine actually detected. The scanning logic stays deterministic and testable; AI is used purely for the "explain it to a human" layer.

## Code architecture

Each security check is implemented as its own class following a shared interface (Strategy pattern), so adding a new check type doesn't require touching existing ones. A central Scan Service runs all registered checks against a target endpoint, writing each result to Redis as it completes. Once every check has finished, the Scan Service pulls all results back from Redis and hands the failed ones to the AI Analysis Service, which sends them to the AI API in a single batched request and stores the resulting explanations/fixes as the final report.


## API map
 
| Method | Path | Purpose | Auth required |
|---|---|---|---|
| POST | `/auth/register` | Create account (includes one-time authorization confirmation) | No |
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

## Security check implementation details

| Check | How it's implemented |
|---|---|
| **Broken Access Control** | Logs in as one user, tries swapping IDs in requests to access another user's data. Tries hitting admin-only routes as a regular user. |
| **Security Misconfiguration** | Sends malformed input and inspects error responses for stack traces or internal details. |
| **Injection** | Sends common SQL injection payloads into input fields and checks for anomalous responses. |
| **Authentication Failures** | Attempts brute-force login, checks for lockout protection. Tests whether a token still works after logout. |
| **Mishandling of Exceptional Conditions** | Sends missing/malformed fields to see if errors are handled cleanly, leak information, or leave the system in an inconsistent state (e.g. an interrupted multi-step transaction that isn't rolled back). |
| **Rate Limiting** | Sends rapid repeated requests (up to ~1,000) to confirm the API throttles or rejects excess traffic. |
