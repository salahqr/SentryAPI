# ERD — SentryAPI

## Entities and relationships

- **users** (1) → (many) **endpoints** — one user can register multiple endpoints
- **endpoints** (1) → (many) **reports** — one endpoint can have multiple reports over time (capped at 5, oldest deleted when a new one is added)

```
users ──< endpoints ──< reports
```

## Design notes

- The database only stores what needs to last: user accounts (with a one-time authorization confirmation), the endpoints they've registered, and each endpoint's most recent reports.
- The AI's full formatted output — findings, severities, explanations, suggested fixes, and the overall score — is stored as-is in one JSON column on `reports`, since it already arrives structured from the AI API. No separate table is needed for it; it's never queried field-by-field, only read back as a whole.
- `authorization_confirmed` lives on `users`, not `endpoints` — confirmed once at signup, covering all endpoints going forward, instead of asking again every time (unnecessary friction with no real added safety).
- No in-progress/status tracking is stored in the database. A scan's live state (started, which check is running) is transient and would mean repeated writes for something that disappears once the scan finishes — that's tracked in application logs and briefly in Redis instead. A `reports` row is only ever inserted once, after a scan is fully complete.

## Schema

```sql
CREATE TABLE users (
    id                       INT AUTO_INCREMENT PRIMARY KEY,
    email                    VARCHAR(255) UNIQUE NOT NULL,
    password_hash            VARCHAR(255) NOT NULL,
    authorization_confirmed  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at               TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE endpoints (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    user_id         INT NOT NULL,
    name            VARCHAR(256) NOT NULL,
    url             VARCHAR(2048) NOT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE TABLE reports (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    endpoint_id     INT NOT NULL,
    report_data     JSON,       -- full AI-formatted report, e.g.:
                                 -- {security_score, findings: [{check_type, passed, severity, explanation, suggested_fix}, ...]}
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (endpoint_id) REFERENCES endpoints(id) ON DELETE CASCADE
);
```

## Report retention

The 5 most recent reports per endpoint are kept; older reports are automatically removed as new ones are generated.
