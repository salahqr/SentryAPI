- Landing page
- Login / Register (combined page)
- Reset password (request + confirm — 2 steps, same as the API)
- Dashboard
    - Nav: Home, Logout, + Create New URL button
    - List of URLs (empty state if none yet)
    - Click a URL → its detail page:
        - Name/label for the URL (editable)
        - "Start Scan" button
        - Scan in progress state (while scanning)
        - Scan failed state (if it errors)
        - List of past reports (up to 5) with count
        - Delete URL (with confirmation)
- Report detail page:
    - Security score
    - List of findings (severity badge, AI explanation, suggested fix per finding)
- 404 / not found page

<br>

## Design

Full mockups and page looks: [View on Figma](https://www.figma.com/design/GmSnkewkirVENOB1oBEMI3/SentryAPI)