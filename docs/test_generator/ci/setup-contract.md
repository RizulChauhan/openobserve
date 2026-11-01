# Test Setup Contract: Traces Open Result in New Tab  (area: Traces)

Concrete data + navigation preconditions for `traces-15477-open-trace-new-tab.spec.js`
(`tests/ui-testing/playwright-tests/RegressionSet/Traces/`). Every item below copies an
EXISTING helper — do not invent a new ingestion/setup path.

## Streams / data the spec must establish

- **`e2e_15477_<random>`** **[per-test]** — a traces stream seeded with 5 single-span ERROR traces,
  each followed by an OK span (so `span_status` filtering and page boundaries are checkable).
  Fields present per hit (spans mode): `trace_id`, `span_id`, `operation_name`, `start_time`,
  `end_time`, `http_status_code`, `span_status`, `service_name`, `duration`, `spans`.
  Why: every test reads the result rows' link/new-tab behavior; the stream is per-test (unique
  name) and deleted in `afterEach`, so tests never contend for it.
  - NOTE: the current spec switches to **spans mode**, so the href also carries `span_id=`.
    If a future test asserts trace-mode behavior, it must stay in Traces mode (no `span_id`).

## How to create it (copy these EXACT patterns — do NOT invent setup)

- **Ingest** (from the existing spec `traces-15477-open-trace-new-tab.spec.js:17-19`):
  ```js
  stream = 'e2e_15477_' + Math.random().toString(36).slice(2, 7);
  const ingest = await ingestSequencedErrorSpans(page, stream, 5);
  expect(ingest.status, 'Precondition: seeded spans must ingest').toBe(200);
  ```
  `ingestSequencedErrorSpans(page, streamName, count)` lives at
  `tests/ui-testing/playwright-tests/utils/trace-ingestion.js:325` and POSTs
  `/api/{org}/v1/traces` with header `stream-name`.
- **Navigate to the search page with the stream selected**:
  `await pm.tracesPage.navigateToTracesUrlWithStream(stream);` —
  `tests/ui-testing/pages/tracesPages/tracesPage.js:885` (builds
  `/web/traces?org_identifier=<ORGNAME>&stream=<stream>`; a full goto re-fetches the stream list
  so a stream ingested after page load is selectable).
- **Time range** (so the just-ingested spans are in window):
  `await pm.tracesPage.setTimeRange('15m');` — `tracesPage.js:583`.
- **Spans mode**: `await pm.tracesPage.switchToSpansMode();` — `tracesPage.js:515`
  (clicks `traces-search-mode-spans-btn` and asserts `data-state="on"`).
- **Wait for rows to hydrate** before asserting on links:
  `await expect(pm.tracesPage.traceRowLinks().first()).toBeVisible({ timeout: 30000 });`
  (`traceRowLinks()` → `[data-test="trace-row-link"]`, `tracesPage.js:898`).

## Test actions / new-tab plumbing (copy these EXACT helpers)

- `traceRowLinks()` → `page.locator('[data-test="trace-row-link"]')` — `tracesPage.js:898`.
- `firstRowNonLinkCell()` → first `o2-table-row-*` `td[data-test^="o2-table-cell-"]:not(:has(a))`
  — used to exercise the row's own middle-click handler rather than the browser's native link
  behavior — `tracesPage.js:903`.
- `trackNewPages(context)` → `const opened = []; context.on('page', p => opened.push(p))` —
  `tracesPage.js:911`.
- `expectOneNewPage(opened)` → polls `opened.length === 1`, then waits 1000ms and asserts exactly
  1 — `tracesPage.js:918`.
- Assert the new tab URL: `await expect(opened[0]).toHaveURL(new RegExp(DETAILS_PATH));`
  where `DETAILS_PATH = '/traces/trace-details'`.
- Cleanup: `await deleteStream(page.request, stream);` —
  `tests/ui-testing/playwright-tests/utils/traces-api.js:63`.

## Preconditions / toggles

- Auth/org: `navigateToTracesUrlWithStream` uses `process.env["ORGNAME"] || 'default'` and
  `process.env["ZO_BASE_URL"]`. The worker auth state / login is established by
  `navigateToBase(page)` from `utils/enhanced-baseFixtures.js` (already in the spec).
- Non-SQL / default mode: the spec does NOT toggle SQL mode; the default (non-SQL) query editor is
  assumed. Do not switch to SQL mode.
- Spans mode MUST be active before asserting `trace_id=` + `span_id=` hrefs (see Edge Case 2).

## Gotchas (so the Healer/Engineer don't rediscover them)

- **Href is mode-dependent.** In spans mode the query includes `span_id=`; in traces mode it does
  not. `toContain('trace_id=')` is safe in both; `toContain('span_id=')` only in spans mode.
- **The link is a real `<a>`**, so Ctrl/Cmd-click on the LINK is native browser behavior (no
  `window.open`); a popup blocker would only matter for the non-link-cell `window.open` paths
  (Playwright's default context does not block popups).
- **A second tab opens asynchronously** — `expectOneNewPage` includes a fixed 1s settle wait after
  the `page` event; don't remove it or the "exactly one" assertion can race.
- **Middle-click target must NOT contain an `<a>`.** `firstRowNonLinkCell()` filters
  `:not(:has(a))` so the row's `@auxclick` path (`window.open`) is what fires. Middle-clicking the
  link would instead trigger the browser's native link middle-click.
- **Stream schema/rows arrive async** after ingestion; always wait on
  `traceRowLinks().first()` visible (30s timeout) before touching rows — a search that runs before
  ingestion commits returns 0 hits and renders the empty state (`traces-search-result-not-found-text`).
- **Ingestion is asserted 200** — if `ingest.status !== 200` the whole suite should fail at the
  precondition, not downstream, so keep that `expect`.
