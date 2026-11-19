# HARlens: Design (Phase 2)

Status: Approved v1.0 (2026-11-20)
Based on: [01-requirements.md](01-requirements.md) v1.0

---

## 1. Architecture overview

HARlens is a **static single-page app**. There is no backend. Vercel only serves the built files (HTML, JS, CSS). All processing happens in the user's browser.

The code is split into two layers:

| Layer | Folder | What it does | Depends on React? |
|-------|--------|--------------|-------------------|
| Core | `src/core/` | Parse HAR, run detectors, mask secrets, build reports | **No** |
| UI | `src/ui/` | Show results, handle clicks, filters and routing | Yes |

**Why this split:** all the logic worth testing (parsing, detection, masking) is plain TypeScript functions. Each test is input in, output out, with no browser or rendering needed. It also makes it easy to move parsing into a Web Worker later (stretch goal S-06).

### Data flow

```
 .har file (user's disk)
      |
      v
 FileReader  ->  text  ->  JSON.parse
      |
      v
 parseHar()          validate structure, normalise every entry
      |
      v
 ParsedHar           { meta, entries: ParsedEntry[] }
      |
      v
 analyze()           runs every detector, builds the summary
      |
      v
 Analysis            { meta, entries, findings, summary }
      |
      v
 React state  ->  Summary / Findings / Request list / Detail views
      |
      v
 Exports             escalation report, cURL  (always pass through redact())
```

Nothing in this flow makes a network request. The sample HAR (US-03) is bundled into the app at build time, so loading it does not need `fetch` either.

---

## 2. Key technical decisions

| # | Decision | Recommendation | Why | Alternative |
|---|----------|----------------|-----|-------------|
| D1 | HAR validation | Hand-written type guards and a normaliser | We must be **lenient**: fill defaults for missing fields instead of rejecting the file. A normaliser does exactly that, with no extra dependency. | Zod: strict schemas, an extra dependency, and we would still need the defaulting logic |
| D2 | State management | React `useReducer` + Context | Data is read-only after load. Only filters and selection change. A library would be overkill. | Zustand or Redux |
| D3 | Routing | React Router, two routes: `/` and `/guide` | The guide needs its own link that support agents can send (US-14 AC1). | Separate static HTML page |
| D4 | Styling | CSS Modules + CSS variables | Built into Vite, no setup. Plain CSS is easy to read and review line by line. Variables make dark mode (S-09) simple. | Tailwind: faster to write, but long class strings are harder to review |
| D5 | Large lists | `@tanstack/react-virtual` | Renders only the visible rows, so 5,000+ entries stay smooth (US-10 AC6). | Pagination |
| D6 | Masking enforcement | One `redact` module, plus a `Masked` TypeScript type for exports | Export functions only accept masked values, so the compiler stops raw secrets from reaching a report. | Convention only (easy to forget) |
| D7 | Privacy enforcement | Content Security Policy in `vercel.json` with `connect-src 'none'` | The browser itself blocks any network request from the app, even if a bug or a dependency tries one. | Trust the code only |
| D8 | File size | Main-thread parsing in MVP; warning above 50 MB | Simple first. Web Worker parsing is stretch S-06. | Worker from day one |

---

## 3. Folder structure

```
harlens/
  .github/
    workflows/
      ci.yml                  lint, type check, test, build
  docs/
    01-requirements.md
    02-design.md
  public/
    favicon.svg
  src/
    core/                     pure TypeScript, no React
      har/
        harTypes.ts           raw HAR 1.2 types (as the browser exports them)
        parseHar.ts           validate + normalise -> ParsedHar
        parseHar.test.ts
      model.ts                ParsedEntry, Finding, Analysis types
      analyze.ts              runs all detectors, builds the summary
      analyze.test.ts
      detectors/
        index.ts              list of all detectors
        failedRequests.ts     US-05
        auth.ts               US-06
        cors.ts               US-07
        network.ts            US-07
        slow.ts               US-08
        redirects.ts          US-09
        *.test.ts             one test file per detector
      redact/
        redact.ts             masking rules (US-12)
        redact.test.ts
      export/
        escalationReport.ts   Markdown report (US-13)
        curl.ts               Copy as cURL (US-11)
        *.test.ts
      filters/
        presets.ts            filter presets (US-10)
        presets.test.ts
      explain/
        statusCodes.ts        plain-English text per status code
    ui/
      App.tsx                 routes
      state/
        AnalysisContext.tsx   holds Analysis, filters, selection
      pages/
        HomePage.tsx          drop zone, sample button, guide link
        AnalyzerPage.tsx      main results view
        GuidePage.tsx         how to capture a HAR (US-14)
      components/
        DropZone.tsx
        SummaryPanel.tsx
        FindingsPanel.tsx
        FilterBar.tsx
        RequestTable.tsx
        RequestDetail.tsx
        MaskedValue.tsx       shows masked value + reveal + "why masked?"
        CopyButton.tsx
      styles/
        tokens.css            colours, spacing, fonts as CSS variables
    samples/
      sample.har              synthetic demo file (US-03)
    main.tsx                  app entry point
  test/
    fixtures/                 small synthetic HAR files, one per scenario
      chrome-basic.har
      chrome-sanitized.har
      firefox-basic.har
      failed-requests.har
      sso-redirect-loop.har
      oauth-error.har
      expired-jwt.har
      cors-preflight-fail.har
      slow-requests.har
      invalid-not-json.txt
      empty-entries.har
  index.html
  package.json
  tsconfig.json
  vite.config.ts
  vercel.json                 SPA rewrite + security headers
  .gitignore
  README.md
```

**Conventions**
- Unit tests sit next to the file they test (`parseHar.ts` + `parseHar.test.ts`).
- Shared test HAR files live in `test/fixtures/`. All are synthetic, using `example.com`, `example.org` and `idp.example.net` domains.
- `core/` must never import from `ui/`. An ESLint rule enforces this.

---

## 4. Data model

### 4.1 ParsedEntry (one request)

The parser turns each raw HAR entry into a clean, predictable object. Missing values become `null` instead of `-1`, `undefined` or absent keys.

```ts
type ResourceType =
  | 'fetch' | 'xhr' | 'document' | 'script' | 'stylesheet'
  | 'image' | 'font' | 'media' | 'websocket' | 'eventsource' | 'other';

interface NameValue {
  name: string;          // original casing, for display
  value: string;
}

interface Timings {      // all in milliseconds; null when the browser reported -1
  blocked: number | null;
  dns: number | null;
  connect: number | null;
  ssl: number | null;
  send: number | null;
  wait: number | null;   // server wait time, used as TTFB in US-08
  receive: number | null;
}

interface ParsedEntry {
  id: number;                    // position in the HAR, stable key
  startedAt: number;             // epoch ms
  method: string;
  url: string;
  host: string;                  // '' if the URL cannot be parsed
  path: string;
  queryParams: NameValue[];
  isCrossOrigin: boolean;        // compared with the page origin
  resourceType: ResourceType;    // from _resourceType, else inferred
  status: number;                // 0 = no response (blocked, failed, cancelled)
  statusText: string;
  requestHeaders: NameValue[];
  responseHeaders: NameValue[];
  requestBody: { mimeType: string; text: string | null } | null;
  responseBody: {
    mimeType: string;
    size: number | null;
    text: string | null;         // null when not captured
    isBase64: boolean;
  };
  redirectUrl: string | null;    // from redirectURL or the Location header
  browserError: string | null;   // Chrome's _error, e.g. 'net::ERR_FAILED'
  totalTime: number;             // entry.time in ms
  timings: Timings;
  transferSize: number | null;
}
```

### 4.2 Finding (one detected problem)

```ts
type FindingCategory = 'failed' | 'auth' | 'cors' | 'network' | 'slow' | 'redirect';
type Severity = 'critical' | 'high' | 'medium' | 'low' | 'info';
type Confidence = 'high' | 'medium' | 'low';

interface Finding {
  id: string;                    // unique, e.g. 'slow:42'
  ruleId: string;                // which rule fired, e.g. 'slow.total.critical'
  category: FindingCategory;
  severity: Severity;
  confidence: Confidence;        // 'medium' for inferred findings such as CORS
  title: string;                 // 'Login redirect loop detected'
  explanation: string;           // plain English: what this means
  likelyCauses: string[];
  nextStep: string;
  entryIds: number[];            // requests that are evidence for this finding
}
```

Every field in the requirements' "Explainability" rule maps to a field here.

### 4.3 Analysis (the whole result)

```ts
interface HarMeta {
  creator: string;               // e.g. 'WebInspector' (Chrome) or 'Firefox'
  pageOrigin: string | null;     // used for cross-origin checks
  looksSanitized: boolean;       // no cookies or auth headers anywhere (US-04 AC5)
  entryCount: number;
}

interface Summary {
  totalRequests: number;
  failedRequests: number;
  totalTransferBytes: number;
  durationMs: number;            // first request start to last request end
  findingsByCategory: Record<FindingCategory, number>;
  slowest: number[];             // entry ids, top 5
  domains: { host: string; requests: number; failures: number }[];
}

interface Analysis {
  meta: HarMeta;
  entries: ParsedEntry[];
  findings: Finding[];
  summary: Summary;
}
```

### 4.4 Detector contract

Every detector has the same shape:

```ts
type Detector = (entries: ParsedEntry[], meta: HarMeta) => Finding[];
```

- One file per detector, registered in `detectors/index.ts`.
- Pure function: same input, same output, no side effects.
- Adding a new check means adding one file and one line, without changing existing code.

### 4.5 Masking contract

```ts
type Masked = string & { readonly __brand: 'Masked' };

function redactValue(fieldName: string, value: string): Masked;
function isSensitive(fieldName: string): boolean;
```

- `escalationReport()` and `toCurl()` only accept `Masked` values, so passing a raw token is a **compile error**.
- The UI shows `Masked` values by default. Reveal reads the original value from the in-memory entry and never stores it anywhere else (US-12).

---

## 5. Wireframes

### 5.1 Home page (`/`)

```
+--------------------------------------------------------------------+
| HARlens                                     How to capture a HAR -> |
+--------------------------------------------------------------------+
|                                                                    |
|          Find out what went wrong in a HAR file, in seconds.       |
|                                                                    |
|   +------------------------------------------------------------+   |
|   |                                                            |   |
|   |         Drop a .har file here, or [ Choose file ]          |   |
|   |                                                            |   |
|   +------------------------------------------------------------+   |
|                                                                    |
|          [ Load sample HAR ]   (synthetic data)                    |
|                                                                    |
|   Your file never leaves this browser. Nothing is uploaded.        |
|   Why? ->                                                          |
+--------------------------------------------------------------------+
```

### 5.2 Analyzer page (after loading)

```
+----------------------------------------------------------------------------+
| HARlens   customer-login.har  (1,284 requests)   [Copy escalation report]   |
|                                                  [Load another file]        |
+----------------------------------------------------------------------------+
| ! This HAR looks sanitized: cookies and auth headers were removed on       |
|   export. Some auth checks cannot be verified.                     (US-04) |
+----------------------------------------------------------------------------+
| SUMMARY                                                                    |
|  1,284 requests | 37 failed | 18.4 MB | 42.1 s                             |
|  Failed 12   Auth 3   CORS/Network 5   Slow 9   Redirects 1                |
+----------------------------------------------------------------------------+
| FINDINGS (sorted by severity)                                              |
|  [CRITICAL] Login redirect loop (6 hops)                    view ->        |
|  [HIGH]     401 Unauthorized on GET /api/users/{id} (x8)    view ->        |
|  [MEDIUM]   Likely CORS block on api.example.org (x3)       view ->        |
+----------------------------------------------------------------------------+
| [API calls] [Page loads] [Auth/SSO] [Failed] [CORS/Network] [Slow]         |
| [Redirects] [Static assets] [All]       Search: [___________]  [Filters v] |
+-------------------------------------------+--------------------------------+
| REQUESTS                                  | DETAIL                         |
| St  Method  URL                Time       | GET /api/users/123             |
| 200 GET     /api/me            120 ms     | 401 Unauthorized | 340 ms      |
| 401 GET     /api/users/123     340 ms  <  |                                |
| 0   POST    /api/events        --      !  | Findings                       |
| 302 GET     /sso/start         80 ms      |  - Token expired 2 min before  |
| ...                                       |    this request                |
|                                           | Headers                        |
|                                           |  Authorization: Bearer eyJh... |
|                                           |    [masked] [Reveal] [Why?]    |
|                                           | Timings                        |
|                                           |  wait 310 ms (dominant)        |
|                                           | [Copy as cURL]                 |
+-------------------------------------------+--------------------------------+
```

- Clicking a finding filters the request list to its evidence (`entryIds`).
- Status is shown as text and an icon, never colour only (accessibility).

### 5.3 Guide page (`/guide`)

```
+--------------------------------------------------------------------+
| HARlens                                                  Analyze -> |
+--------------------------------------------------------------------+
| How to capture a HAR file                                          |
|                                                                    |
| [ Chrome ] [ Edge ] [ Firefox ] [ Safari ]      (tabs)             |
|                                                                    |
| 1. Open the page where the problem happens.                        |
| 2. Press F12 to open Developer Tools.                              |
| 3. Click the Network tab.                                          |
| 4. Tick "Preserve log" and "Disable cache".                        |
| 5. Reproduce the problem.                                          |
| 6. Click the download icon (Export HAR).                           |
|                                                                    |
| Sanitized or full export?  ->                                      |
| What to send with your HAR (checklist)  ->                         |
+--------------------------------------------------------------------+
```

---

## 6. Security headers (`vercel.json`)

```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "Content-Security-Policy",
          "value": "default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:; connect-src 'none'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'" },
        { "key": "Referrer-Policy", "value": "no-referrer" },
        { "key": "X-Content-Type-Options", "value": "nosniff" }
      ]
    }
  ]
}
```

- `connect-src 'none'` blocks `fetch`, XHR and WebSockets from the app. This is the strongest proof that no data is uploaded.
- The rewrite makes `/guide` work when opened directly from a link.

---

## 7. Testing approach (detailed in Phase 5)

| What | How |
|------|-----|
| Parser | Fixture files for Chrome, Firefox, sanitized and broken HARs |
| Detectors | One fixture per scenario; check exact findings, severity and ruleId |
| Masking | Every default sensitive field; check exports never contain the raw value |
| Report | Snapshot test of the Markdown output |
| UI | React Testing Library for drop zone, filters and reveal behaviour |
| Privacy | Test that loading a file makes no `fetch` / XHR call |

---

## 8. Points to review closely in generated code

- **`redact.ts`**: the most security-critical file. Check that every name in US-12 is covered and that matching is case-insensitive.
- **`parseHar.ts`**: `-1` timings must become `null`, not `0`. A `0` would hide missing data and skew the slow detection.
- **Cross-origin check**: URLs such as `data:`, `blob:` and `chrome-extension:` must not crash `new URL()` or count as cross-origin.
- **Slow detector**: WebSockets, `text/event-stream` and long-polling must be excluded (US-08 AC3).
- **Redirect loops**: URL comparison should ignore fragments (`#...`) and query parameter order, or loops will be missed.

---

## 9. Decisions log

| # | Decision | Status |
|---|----------|--------|
| D1 | Hand-written parser instead of Zod | Approved |
| D2 | `useReducer` + Context for state | Approved |
| D3 | React Router for `/guide` | Approved |
| D4 | CSS Modules instead of Tailwind | Approved |
| D5 | `@tanstack/react-virtual` for large lists | Approved |
| D6 | `Masked` type to block raw secrets in exports | Approved |
| D7 | CSP with `connect-src 'none'` | Approved |
| D8 | Main-thread parsing in MVP, 50 MB warning | Approved |
