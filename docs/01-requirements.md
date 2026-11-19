# HARlens: Requirements (Phase 1)

Status: Approved v1.0 (2026-11-20)
Changes in v0.2: reveal toggle with security explanation (US-12), researched slow thresholds (US-08), HAR capture guide (US-14), filter presets by issue type (US-10).

---

## 1. Problem

Support engineers get HAR files from customers almost every day. To find the cause of a failure, they scroll through hundreds or thousands of requests by hand in DevTools or a generic HAR viewer. Common problems include failed API calls, broken SSO redirects, expired tokens, CORS blocks and slow endpoints.

Today this is:
- **Slow**: no tool points to "what went wrong", only "what happened".
- **Risky**: HAR files contain cookies, tokens and SAML assertions. Pasting them into online analyzers can leak credentials.
- **Inconsistent**: every engineer writes a different escalation note.
- **Back-and-forth**: customers often send incomplete HARs (recorded too late, no "Preserve log", wrong tab), so support has to ask again and resolution takes longer.

**HARlens** is a browser-only tool. It teaches customers how to capture a useful HAR, parses the file locally, flags likely problems in plain English, and produces a masked, Jira-ready escalation report.

---

## 2. Users

| ID | Persona | Goal |
|----|---------|------|
| P1 | Support engineer (primary) | Triage a customer HAR in minutes and escalate with evidence |
| P2 | Developer / QA | Debug their own app's network failures |
| P3 | Escalation receiver (engineer reading Jira) | Get a clean, consistent summary without opening the raw HAR |
| P4 | End user / customer (non-technical) | Capture and send a correct HAR the first time, so their ticket is resolved faster |

---

## 3. Constraints and assumptions

- **Format**: HAR 1.2. Must handle exports from Chrome, Edge, Firefox and Safari. Chrome adds custom fields starting with `_` (for example `_error`, `_resourceType`, `_initiator`). These are optional and must never be required.
- **Sanitized exports**: Recent Chrome versions export HARs **without** cookies and auth headers by default. The tool must not report "missing token" just because headers were stripped on export. It should detect this and say so.
- **Missing data**: Response bodies may be absent, truncated or base64-encoded. Timings may be `-1`. The parser must tolerate this.
- **Client-only**: No backend. File contents never leave the browser.
- **Data**: Synthetic or public test data only.
- **Detection is heuristic**: A HAR does not record "this was a CORS error". Some findings can only be inferred, so the UI labels them as "likely" with a confidence level.

---

## 4. Scope

### MVP (must ship)

| ID | Story | Epic |
|----|-------|------|
| US-01 | Load a HAR file | Load |
| US-02 | Handle invalid files | Load |
| US-03 | Try with a sample HAR | Load |
| US-04 | See a summary | Overview |
| US-05 | See failed requests, grouped and explained | Diagnostics |
| US-06 | See auth and SSO problems | Diagnostics |
| US-07 | See CORS and network errors | Diagnostics |
| US-08 | See slow requests | Diagnostics |
| US-09 | See redirect loops and long chains | Diagnostics |
| US-10 | Filter and search requests, with presets by issue type | Explore |
| US-11 | View request details | Explore |
| US-12 | Sensitive data is masked, with an explained reveal option | Privacy |
| US-13 | Copy an escalation report | Export |
| US-14 | Learn how to capture a useful HAR | Guide |

### Stretch goals (after MVP)

| ID | Idea | Why it is useful |
|----|------|------------------|
| S-01 | Waterfall / timeline chart | Visualise blocking and parallel requests |
| S-02 | SAML response decoder (status, issuer, audience, NotOnOrAfter) | Explains SSO failures in depth |
| S-03 | JWT claims viewer (no signature check) | Shows scopes, audience, expiry |
| S-04 | Compare two HARs (working vs broken) | Spot the difference quickly |
| S-05 | Download a sanitized copy of the HAR | Safe sharing with third parties |
| S-06 | Parse in a Web Worker | Keeps UI responsive for 50 MB+ files |
| S-07 | Download report as `.md` file | Alternative to clipboard |
| S-08 | Custom masking rules | Company-specific header names |
| S-09 | Dark mode | Polish |
| S-10 | Configurable slow thresholds | Different apps, different baselines |
| S-11 | Per-endpoint outlier detection (for example 3x the median of the same endpoint) | Catches "slow for this endpoint" even under the fixed thresholds |
| S-12 | HAR quality check ("this capture looks incomplete because...") | Tells support immediately if they need to ask for a new HAR |

### Out of scope

- Any backend, accounts, or file storage
- Live traffic capture
- AI or LLM-based analysis (rule-based on purpose: deterministic, testable, and nothing leaves the browser)
- Mobile-optimised layout (responsive is fine, desktop is the target)

---

## 5. User stories and acceptance criteria

### US-01 Load a HAR file
As a support engineer, I want to drop a `.har` file into the page, so that I can start analysing it immediately.

- AC1: I can drag and drop a file onto the page, or pick it with a file picker.
- AC2: `.har` and `.json` files are accepted.
- AC3: A loading state is shown while parsing.
- AC4: After parsing, the summary view opens automatically.
- AC5: No network request is made with the file contents (verified by test).

### US-02 Handle invalid files
As a user, I want a clear message when a file cannot be read, so that I know what to fix.

- AC1: Non-JSON file: "This file is not valid JSON."
- AC2: Valid JSON without `log.entries`: "This does not look like a HAR file."
- AC3: HAR with zero entries: "This HAR file has no requests. It may have been exported before the page loaded." Links to the capture guide (US-14).
- AC4: The app never crashes or shows a blank screen on bad input.

### US-03 Try with a sample HAR
As a first-time visitor (for example a recruiter), I want to load a sample file with one click, so that I can see what the tool does without my own HAR.

- AC1: A "Load sample" button is visible on the start screen.
- AC2: The sample is synthetic and contains at least one of each finding type.
- AC3: The sample is clearly labelled as synthetic.

### US-04 See a summary
As a support engineer, I want an overview of the capture, so that I understand its scope at a glance.

- AC1: Shows total requests, failed requests, total transfer size, and capture duration.
- AC2: Shows the number of findings by category (failed, auth, CORS/network, slow, redirects).
- AC3: Shows the top 5 slowest requests.
- AC4: Shows domains involved, with request count and failure count per domain.
- AC5: If the HAR looks sanitized (no cookies or auth headers anywhere), a notice explains this.

### US-05 See failed requests, grouped and explained
As a support engineer, I want failed requests grouped with a plain-English explanation, so that I can spot the root cause instead of reading every row.

- AC1: A request counts as failed if status is 400 to 599, or status is 0 / has a browser error.
- AC2: Failures are grouped by status code and endpoint pattern (for example `GET /api/users/{id}` with IDs collapsed).
- AC3: Each group shows: count, a plain-English meaning of the status, common causes, and a suggested next step.
- AC4: Clicking a group lists its requests; clicking a request opens the detail view.

### US-06 See auth and SSO problems
As a support engineer, I want auth failures highlighted, so that I can debug login and token issues quickly.

- AC1: 401 and 403 responses are flagged, with the `WWW-Authenticate` header value if present.
- AC2: Redirects to common identity provider paths (for example `/authorize`, `/oauth2/`, `/saml`, `/sso`) are recognised as an SSO flow.
- AC3: An OAuth error in a redirect or callback URL (`error=`, `error_description=`) is flagged with its decoded description.
- AC4: An SSO flow that ends in a 4xx/5xx or loops is flagged.
- AC5: If a bearer token is a JWT, its `exp` is compared to the request time; expired tokens are flagged. The token itself is never displayed unmasked by default.
- AC6: If auth headers are absent because the HAR is sanitized, the tool says "cannot verify" instead of "missing".

### US-07 See CORS and network errors
As a support engineer, I want blocked and failed network calls flagged, so that I can tell browser-level failures from server errors.

- AC1: Requests with status 0 or a browser error (for example `net::ERR_*`) are listed with the error text.
- AC2: A request is flagged as "likely CORS" if: an `OPTIONS` preflight failed, or a cross-origin request has status 0, or a cross-origin response lacks `Access-Control-Allow-Origin`.
- AC3: Each CORS finding states its confidence (high / medium) and the rule that triggered it.
- AC4: Cancelled or aborted requests are shown separately and marked as usually harmless.

### US-08 See slow requests
As a support engineer, I want slow requests listed with the reason they are slow, so that I can tell a slow server from a slow network or a large download.

**Research basis for the thresholds**
- Google's web.dev guidance rates server response time (Time to First Byte) as good at 800 ms or less and poor above 1800 ms. Source: web.dev/articles/ttfb. It is defined for page loads, so HARlens borrows it as a per-request heuristic.
- Nielsen Norman Group's response-time limits: about 1 s keeps the user's flow of thought, and about 10 s is the limit for keeping attention. Source: nngroup.com/articles/response-times-3-important-limits.

**Rules**

| Check | Measured on | Warning | Slow | Critical |
|-------|-------------|---------|------|----------|
| Server wait (TTFB) | `timings.wait` | > 800 ms | > 1800 ms | n/a |
| Total time | `time` | > 1 s | > 3 s | > 10 s |

- AC1: Requests are flagged using the table above. The highest level reached is shown.
- AC2: Each slow request shows its dominant phase and a plain-English cause:
  - `wait` dominates: "The server took a long time to respond" (backend issue).
  - `receive` dominates: "Large or slow download" (check response size).
  - `blocked` / `dns` / `connect` / `ssl` dominate: "Network or connection setup delay" (often the user's network, proxy or VPN).
- AC3: Long-lived connections are excluded, so they are not false positives: WebSockets, Server-Sent Events (`text/event-stream`), and long-polling requests.
- AC4: Slow requests are sorted by total time, slowest first.
- AC5: Each finding shows which rule and threshold triggered it.

### US-09 See redirect loops and long chains
As a support engineer, I want redirect loops detected, so that I can catch broken SSO and routing setups.

- AC1: Redirect chains are rebuilt by following `Location` / `redirectURL` across 3xx responses.
- AC2: A chain is a loop if the same URL appears twice.
- AC3: A chain longer than 10 hops is flagged as excessive.
- AC4: The full chain is shown step by step with status codes.

### US-10 Filter and search requests, with presets by issue type
As a support engineer, I want one-click filter presets for common issue types, so that I see the requests that matter for the problem I am debugging.

**Presets**

| Preset | Shows | Use when the issue is... |
|--------|-------|--------------------------|
| API calls (Fetch/XHR) | `fetch` and `xhr` requests | Data not loading, buttons failing, "something went wrong" errors. Most useful in practice. |
| Page loads (Doc) | `document` requests | Blank page, page not found, wrong page shown |
| Auth / SSO | Requests with auth findings, IdP domains, 401/403, OAuth or SAML redirects | Login fails, user logged out, access denied |
| Failed | 4xx, 5xx, status 0 | Any error |
| CORS / Network | Status 0, `net::ERR_*`, failed preflights | "Failed to fetch", blocked requests, works on one network but not another |
| Slow | Requests with slow findings | Page or feature is slow, timeouts |
| Redirects | 3xx and chains | Redirect loops, wrong landing page |
| Static assets | JS, CSS, images, fonts | Broken layout, missing images, outdated version after a deploy |

- AC1: Presets are shown as one-click chips, each with a short "use when" hint.
- AC2: Default view on load is **API calls (Fetch/XHR)**, with one click to show all requests.
- AC3: Resource type comes from `_resourceType` when present, otherwise it is inferred from MIME type and initiator.
- AC4: Text search across URL and method.
- AC5: Manual filters: status class, method, domain, resource type, and "has finding". They combine with presets and clear with one click.
- AC6: The list stays responsive with 5,000 entries.

### US-11 View request details
As a user, I want a detail view for a request, so that I can see the evidence behind a finding.

- AC1: Shows method, URL, status, timings, request and response headers, query params, and body preview (if present).
- AC2: Shows any findings attached to this request, with explanations.
- AC3: Sensitive values are masked (see US-12).
- AC4: A "Copy as cURL" action is available, with masked values.

### US-12 Sensitive data is masked, with an explained reveal option
As a support engineer, I want secrets masked everywhere, and I want to understand why, so that I never leak customer credentials in screenshots or tickets.

**Why values are masked (shown in the app)**
- Cookies, bearer tokens and SAML assertions are effectively **keys to the user's session**. Anyone who has one can act as that user until it expires, without a password or MFA.
- Screenshots, Jira tickets and Slack messages are shared widely and kept for a long time. A token pasted there is exposed to everyone with access.
- Masking by default means the safe action is the easy action.

**Security constraints of the reveal option**
- Reveal works per field, in the detail view only, and needs a deliberate click.
- Revealed values exist only in browser memory. They are never written to localStorage, sessionStorage, IndexedDB, the URL or logs.
- Values are re-masked automatically when you open another request or reload the page.
- Exports (escalation report, cURL) **always** use masked values, even if a field is revealed on screen.
- Closing the tab removes all data. Nothing is kept between sessions.

**Acceptance criteria**
- AC1: Masked headers by default: `Authorization`, `Proxy-Authorization`, `Cookie`, `Set-Cookie`, `X-API-Key`, `X-Auth-Token`, `X-CSRF-Token`, and similar.
- AC2: Masked query and body fields by default: `access_token`, `id_token`, `refresh_token`, `code`, `client_secret`, `password`, `api_key`, `SAMLResponse`, `SAMLRequest`.
- AC3: Masked format keeps a short prefix for context, for example `Bearer eyJh…[masked]`.
- AC4: Each masked field has a "Why is this masked?" link that opens the explanation above.
- AC5: Reveal follows every constraint above, and each constraint has a test.
- AC6: Exports are always masked, with no option to disable.
- AC7: A unit test covers every default masked field.

### US-13 Copy an escalation report
As a support engineer, I want a one-click Markdown report, so that I can paste it into a Jira ticket.

- AC1: "Copy escalation report" copies Markdown to the clipboard and confirms success.
- AC2: The report contains: capture summary, key findings ordered by severity, the top failing requests (method, masked URL, status, time), and redirect chains if any.
- AC3: All sensitive values are masked.
- AC4: The report renders correctly in Jira and GitHub Markdown.
- AC5: A snapshot test locks the report format.

### US-14 Learn how to capture a useful HAR
As a customer, I want simple step-by-step instructions for recording a HAR, so that I send support the right file the first time and my issue is resolved faster.

- AC1: A "How to capture a HAR" page with its own URL (for example `/guide`), so support agents can send the link to customers.
- AC2: Step-by-step instructions for Chrome, Edge, Firefox and Safari, written for non-technical users.
- AC3: Covers the steps that are most often missed:
  - Open DevTools **before** reproducing the issue.
  - Turn on **Preserve log** (needed for login and redirect flows).
  - Turn on **Disable cache**.
  - Reproduce the problem fully, then export.
  - Record in the tab where the problem happens.
- AC4: Explains sanitized vs full exports: use the default (sanitized) export unless support asks for full data, and send full exports only through the official support channel.
- AC5: A "What to send with your HAR" checklist: what you were trying to do, the time it happened, your timezone, the expected vs actual result, and a screenshot of the error.
- AC6: Reachable from the start screen and from the "no requests" error (US-02).
- AC7: Written in short sentences with numbered steps. Does not depend on screenshots, so it stays correct when browser UIs change.

---

## 6. Non-functional requirements

| Area | Requirement |
|------|-------------|
| Privacy | No file data in any network request. No third-party analytics or scripts. Strict Content Security Policy. No file data in browser storage. |
| Performance | 10 MB HAR (about 2,000 entries) parsed and shown in under 2 s on a mid-range laptop. Request list virtualised. |
| Robustness | Tolerates missing optional fields, `-1` timings, absent bodies, browser-specific fields. |
| Accessibility | Keyboard navigation, WCAG 2.1 AA contrast, status never shown by colour alone. |
| Readability | Plain English, short sentences, consistent labels. Explanations avoid jargon or define it. |
| Code quality | TypeScript strict mode, ESLint and Prettier, no `any` in parsing or detection code. |
| Testing | At least 80% line coverage on parser, detectors, and masking. CI must pass before merge. |
| Browsers | Latest Chrome, Edge, Firefox, Safari (desktop). |
| Explainability | Every finding has: title, severity, plain-English meaning, likely causes, next step, and the rule that triggered it. |

---

## 7. Stack

React + TypeScript + Vite, tested with Vitest and React Testing Library, linted with ESLint and Prettier, deployed on Vercel. Styling and state management are decided in Phase 2.

---

## 8. Definition of Done (per story)

- Acceptance criteria met
- Unit tests written and passing
- Lint, type check and build pass in CI
- Merged via pull request from a feature branch, with conventional commits
- No real customer data in code, fixtures or screenshots

---

## 9. Decisions log

| # | Decision | Status |
|---|----------|--------|
| 1 | Per-field reveal in detail view only, with an explanation of why values are masked; exports always masked | Approved |
| 2 | Tiered slow thresholds based on web.dev TTFB and Nielsen limits, instead of one 3 s cut-off | Approved |
| 3 | Sample HAR in MVP, plus a customer-facing HAR capture guide (US-14) | Approved |
| 4 | Stack confirmed with ESLint, Prettier, React Testing Library | Approved |
| 5 | Filter presets by issue type; default view is API calls (Fetch/XHR) | Approved |
