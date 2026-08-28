# PostHog Event Capture Audit Report

## Summary

This audit covers the posthog.com website (a Gatsby React SPA with server-side API routes), auditing both client-side `posthog-js` capture calls and server-side `posthog-python` captures. The fix lens surfaced one error and two warnings around dynamic event naming, mixed naming conventions, and PII leakage; the optimize lens flagged six under-utilized captured events and confirmed that 100% of current event volume originates from `localhost`, indicating the live PostHog project is being used as a development environment.

**Counts**

- **Errors**: 1 (must fix)
- **Warnings**: 4 (should fix)
- **Suggestions**: 0 (nice to have / cost savings)
- **Passes**: 2

**Problematic items** _(only `error`, `warning`, `suggestion` — no passes)_

| Severity | Area | Check | File | Details |
|----------|------|-------|------|---------|
| `error` | Event Capture | Static event names | `src/components/Link/index.tsx:141` | `posthog.capture(event)` uses a dynamic variable as the event name instead of a static string literal. |
| `warning` | Event Capture | Naming standardization | `src/components/Squeak/hooks/useQuestion.tsx:138` | Three conventions in use (space-separated, snake_case, Title Case); own-compliance 33%, PostHog-compliance 32% — adopt snake_case throughout. |
| `warning` | Event Capture | Quality context review | `src/components/DuckDBWaitlistSurvey/index.tsx:37` | Raw email passed as `$survey_response` on `survey sent` event (PII in event properties). |
| `warning` | Event Capture — Optimize | Event usage coverage | `src/components/NotFoundPage/BlueScreenOfDeath.tsx:17` | 6 of 7 captured events are not referenced by any PostHog insight, dashboard, cohort, or experiment. |
| `warning` | Event Capture — Optimize | Environment pollution | _(tenant-side)_ | 100% of $pageview events over the last 7 days originate from `localhost:8010` — the production PostHog project is receiving only local dev traffic. |

## Recommended actions

1. **Event Capture · Static event names** — `src/components/Link/index.tsx:141` calls `posthog.capture(event)` where `event` is a dynamic prop, making it impossible to know what event names are actually sent at runtime. _Why it matters:_ Dynamic event names break event schemas, make PostHog's autocomplete and event definitions unreliable, and prevent safe refactoring of instrumentation. _Fix:_ In `src/components/Link/index.tsx:141`, replace the dynamic `event` prop with a static string literal at each call site, or remove the generic `Link` capture and add explicit `posthog.capture('link_clicked', {...})` calls where needed. See [PostHog best practices](https://posthog.com/docs/product-analytics/best-practices).

2. **Event Capture · Quality context review** — `src/components/DuckDBWaitlistSurvey/index.tsx:37` passes the user's raw email as the `$survey_response` property on the `survey sent` event. _Why it matters:_ Any PostHog user with event-read access can see email addresses in the events table and exported data, creating a data-privacy risk and violating the principle of keeping PII in person properties only. _Fix:_ Remove the `$survey_response: email` property from the `posthog.capture('survey sent', ...)` call at `src/components/DuckDBWaitlistSurvey/index.tsx:37` — the email is already set via `posthog.setPersonProperties` two lines later, so no data is lost. See [PostHog best practices](https://posthog.com/docs/product-analytics/best-practices).

3. **Event Capture · Naming standardization** — The codebase uses three incompatible event-name conventions (space-separated lowercase, snake_case, and Title Case with spaces), with only 33% of events following a consistent pattern. _Why it matters:_ Mixed conventions make it impossible to write reliable HogQL queries, break dashboard consistency, and slow down new instrumentation because there is no agreed-upon standard to follow. _Fix:_ Adopt snake_case throughout — starting with the highest-friction examples: rename `'squeak error'` → `squeak_error` (`src/components/Squeak/hooks/useQuestion.tsx:138`), `'Played video'` → `video_played` (`src/components/WistiaCustomPlayer/index.tsx:469`), and `'Opened MaxAI chat'` → `max_ai_chat_opened` (`src/components/AskMax/index.tsx:47`). See [PostHog best practices](https://posthog.com/docs/product-analytics/best-practices).

4. **Event Capture — Optimize · Environment pollution** — All 8,809 events captured in this PostHog project over the last 7 days originate from `localhost:8010`, meaning 100% of event volume is local development traffic with zero production signal. _Why it matters:_ Any dashboard, funnel, or insight built on this project reflects only developer activity, not real user behavior; the project is effectively unusable for production analytics as configured. _Fix:_ Create a separate PostHog project for production and configure the production deploy to use its own API key — then gate the current key's usage on `NODE_ENV !== 'production'` or only use it in local dev. See [PostHog cutting costs](https://posthog.com/docs/product-analytics/cutting-costs).

5. **Event Capture — Optimize · Event usage coverage** — 6 of 7 captured events (`page_404`, `survey sent`, `docs_page_review`, `docs_page_feedback`, `Played video`, `$pageleave`) are captured in source code but not referenced by any saved insight, dashboard, cohort, or experiment. _Why it matters:_ Captured events that are never queried add to ingestion volume and billing cost without delivering analytical value. _Fix:_ For each captured-only event, either create a PostHog insight that uses it (to validate its value) or remove the `posthog.capture(...)` call — starting with `page_404` at `src/components/NotFoundPage/BlueScreenOfDeath.tsx:17`. See [PostHog cutting costs](https://posthog.com/docs/product-analytics/cutting-costs).

## Full audit

### Event Capture

This area covers correctness and quality of `posthog.capture()` call sites: that event names are static strings, that naming follows a consistent convention, that there are no duplicate or kitchen-sink events, and that the captures themselves don't leak PII, high-cardinality values, JSON-stringified blobs, or fire from hot paths.

| Check | Status | File | Details |
|-------|--------|------|---------|
| Static event names | `error` | `src/components/Link/index.tsx:141` | `posthog.capture(event)` uses a dynamic variable 'event' as the event name instead of a static string literal. |
| Naming standardization | `warning` | `src/components/Squeak/hooks/useQuestion.tsx:138` | detected_convention: mixed; own_compliance_pct: 33; posthog_compliance_pct: 32; recommendation: adopt-a-convention; bad_examples: `squeak error` → `squeak_error`, `Played video` → `video_played`, `Opened MaxAI chat` → `max_ai_chat_opened` |
| Duplicates and bloat | `pass` | `src/components/WistiaCustomPlayer/index.tsx:469` | No exact duplicates, semantic duplicates, or bloat events found. |
| Quality context review | `warning` | `src/components/DuckDBWaitlistSurvey/index.tsx:37` | PII: raw email in `$survey_response` on `survey sent` event. Additional naming drift: `survey sent` uses space-separated mixed case; `Played video` uses title case — both should be snake_case. |

#### Assumptions and blind spots

The static scan only covers files with direct `posthog.capture(` calls (7 call sites across 6 files) and does not analyze dynamic dispatch via the `Link` component's `event` prop — the full set of event names passed through that prop is unknown without runtime tracing. Server-side captures via `posthog-python` in `src/api/` route handlers were scanned but contained no `capture()` calls, only `posthog.init`; if those files delegate to shared utilities not matched by the grep pattern, those captures would be missed. The dynamic naming finding at `src/components/Link/index.tsx:141` could reflect intentional flexibility (e.g. the link is a reusable primitive and callers always pass valid static strings), but without confirming all call sites pass literals, the risk cannot be ruled out. To confirm the PII finding, check the PostHog events table for the `survey sent` event and verify whether `$survey_response` contains real email strings in production data.

### Event Capture — Optimize

This area covers cost-side event capture health: whether captured events are actually used in downstream PostHog artifacts (insights, dashboards, cohorts, experiments), whether pageview / pageleave defaults are dominating event volume on high-traffic SPAs, and whether dev / staging environments are leaking events into the production project. Optimize checks use the PostHog API/MCP to read the operator's tenant; rows showing `mcp_skipped: true` in `details` indicate MCP was unavailable.

| Check | Status | File | Details |
|-------|--------|------|---------|
| Event usage coverage | `warning` | `src/components/NotFoundPage/BlueScreenOfDeath.tsx:17` | captured_count: 7; captured_only: page_404, survey sent, docs_page_review, docs_page_feedback, Played video, $pageleave; heavily_used: $pageview; mcp_skipped: false |
| Pageview defaults | `pass` | `gatsby/onPreBootstrap.ts:20` | capture_pageview_setting: false; capture_pageleave_setting: true; pageview_share_pct: 3.76; recommendation: keep; mcp_skipped: false |
| Environment pollution | `warning` | _(tenant-side)_ | chosen_event: $pageview; polluting_share_pct: 100; top_polluting_hosts: localhost:8010; top_polluting_libs: web, posthog-python; recommendation: use-separate-project-keys; mcp_skipped: false |

#### Assumptions and blind spots

The event-usage coverage check queried `system.insights`, `system.cohorts`, and `system.experiments` for event name references — it does not check actions, CDP functions, or data warehouse exports, so some events flagged as `captured-only` may be referenced in those channels. The environment pollution finding (100% localhost traffic) is consistent with the PostHog host being `http://host.docker.internal:8000`, which confirms this is a local Docker-based PostHog instance with no production traffic; the "pollution" finding is technically expected here rather than a true leakage problem, but it means this project cannot be used for production analytics without a separate project key. The pageview-defaults pass relies on `gatsby/onPreBootstrap.ts` being the sole init site — if any dynamically loaded plugin also calls `posthog.init`, its config could override this. To confirm usage coverage findings, open each captured-only event in PostHog's event explorer and verify whether volume exists and whether any team members query these events ad-hoc (outside saved insights).

## About this audit

This audit ran the PostHog `audit-events` skill — a focused, read-only check of event capture health across two lenses: **fix** (correctness and quality) and **optimize** (cost). Fix checks scan the project source; optimize checks additionally query the PostHog project via MCP in read-only mode (and gracefully skip when MCP is unavailable).

- `error` items break correctness now (dynamic event names, broken contracts). Fix first.
- `warning` items work today but cause subtle bugs, data-quality problems, or noticeably elevated cost. Fix when convenient.
- `suggestion` items are best-practice improvements or cost-savings opportunities with measurable upside.

Re-run `posthog-wizard audit-events` after applying fixes to refresh the ledger.
