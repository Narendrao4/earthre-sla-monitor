# EarthRe SLA Monitor

A deployed-first SLA monitoring workspace that turns messy, multi-agent CSV health checks into an auditable availability record. The interface is designed for two readers at once: on-call engineers need to find failures quickly, while billing and support teams need to understand why the resulting percentage can be trusted.

> ## Live Demo

🌐 https://earthre-sla-monitor.vercel.app/

The application is deployed on Vercel with Neon PostgreSQL as the persistent database.

## Architecture

```text
Browser upload + dashboard (Next.js on Vercel)
                 │
                 ▼ multipart CSV
Vercel Function: POST /api/uploads
  parse → validate → normalize → consolidate → quality receipt
                 │ atomic transaction
                 ▼
Neon serverless Postgres
  upload_batches + service_checks
                 │ filtered SQL
                 ▼
Vercel Functions: /api/dashboard and /api/logs
                 │ JSON
                 ▼
Single-screen dashboard (collapsible stats + filterable logs)
```

- **Next.js 16 App Router on Vercel** keeps the UI and API in one small deployable project. Each route handler under `app/api` becomes an actual Vercel Function in production.
- **TypeScript, Tailwind CSS, and shadcn/ui** provide the frontend system; the availability and latency views use shadcn's chart composition over Recharts 3.
- **Neon Postgres** is durable, relational, serverless-friendly, and available through Vercel's Marketplace free plan. A dataset and all its cleaned checks are written in one transaction.
- **No ORM** is used. The query surface is small, parameterized SQL makes the aggregation decisions visible, and the Neon HTTP driver avoids holding traditional connections open in serverless executions.
- The database schema is created idempotently on first use. The equivalent DDL is also checked in at `db/schema.sql` for review.

## What the dashboard answers

- Overall and per-service observed availability against the 99.9% SLA threshold
- Failure counts and affected services, without hiding non-standard failure sentinels
- P95 response latency, excluding absent readings rather than coercing them to zero
- Data coverage against the stated 15-minute cadence, kept separate from availability
- A daily failure pulse for locating incident windows
- Daily availability plotted against the 99.9% threshold, plus per-service P95 latency
- A visible cleaning receipt: timestamps normalized, seconds converted, duplicates consolidated, missing values, sentinels, and rejected rows
- The cleaned underlying checks, with single-day or inclusive date-range filters, service/state filters, quality flags, and pagination
- Clickable quality details for every check: flagged rows explain each cleaning condition and SLA impact; clean rows show the validations they passed
- A searchable dataset picker with upload date, observed range, and cleaned-row count

## Data findings and handling

All five supplied datasets were profiled with the same code used by the upload function:

| File | Input → clean | Epoch timestamps | Seconds → ms | Blank latency | Duplicates | Conflicting duplicates | `999` | Rejected |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 9 days / seed 101 | 4,672 → 4,319 | 70 | 924 | 56 | 352 | 345 | 1 | 1 |
| 12 days / seed 505 | 6,230 → 5,759 | 93 | 1,229 | 74 | 470 | 460 | 1 | 1 |
| 14 days / seed 202 | 7,269 → 6,719 | 109 | 1,435 | 87 | 549 | 540 | 1 | 1 |
| 21 days / seed 303 | 10,904 → 10,079 | 163 | 2,151 | 130 | 824 | 807 | 1 | 1 |
| 30 days / seed 404 | 15,577 → 14,399 | 233 | 3,095 | 186 | 1,177 | 1,156 | 1 | 1 |

Every file contains exactly one negative latency (`-223` to `-342 ms`), which is the rejected row shown above. Duplicate counts are computed *after* timestamp normalization, so they include collisions between Unix and ISO representations that a raw-string comparison misses. The original order is shuffled in every file.

| Finding | Why it matters | Handling |
| --- | --- | --- |
| Timestamps mix ISO-8601 strings with Unix epoch seconds | Lexical sorting and date filters would be wrong | Parse both, require an explicit timezone for strings, and store UTC `timestamptz` |
| Latency mixes `ms` and `s` | Percentiles would be off by 1,000× | Convert every valid value to milliseconds |
| Some latency values are blank, and one per file is negative | Treating blank as zero would improve latency dishonestly; a negative duration is impossible | Retain blank as `NULL` with `missing_latency`; reject negative values |
| The same service/15-minute interval can be reported twice by different agents | Counting both inflates the denominator and changes the SLA | Consolidate on normalized `(service_id, timestamp)`; keep a failed observation over a successful one, then the slower result as a conservative tie-break; preserve all reporter names |
| Status `999` is not an HTTP status | Dropping it could hide monitoring failures | Preserve it as a non-standard sentinel, classify it unavailable, and add `non_standard_status` |
| Rows are shuffled rather than chronological | First/last file row cannot define the period | Sort on normalized UTC timestamps and derive the range from the data |
| Dataset length varies (9, 12, 14, 21, and 30 days) | Hard-coded ranges produce false coverage | Infer every dataset's range and expected 15-minute intervals independently |

Rows with missing identities, invalid/tz-less timestamps, malformed or negative latency, unsupported units, or unusable status values are rejected and counted by reason. Up to eight source-row examples are returned in the processing receipt for diagnosis.

## Assumptions and decisions

1. **Availability means HTTP 2xx or 3xx.** Redirects show the endpoint served a valid HTTP response; 4xx, 5xx, and sentinels are unavailable.
2. **Availability is based on observed, cleaned intervals.** Missing intervals are not silently counted as success or failure. Coverage is shown alongside availability so a sparse feed cannot look trustworthy by percentage alone.
3. **A duplicate interval is one SLA observation.** The conservative merge rule prevents a second agent's success from erasing a simultaneous failure.
4. **P95 ignores missing latency.** The missing count stays prominent; substituting zero or an arbitrary penalty would distort the latency distribution.
5. **Dates are UTC.** A start date alone means one UTC calendar day. Supplying an end date creates an inclusive UTC date range.
6. **Each upload is an immutable dataset.** Prior uploads remain queryable via the dataset selector, which makes processing results reproducible without requiring accounts or tenants.
7. **The browser/function upload cap is 4 MB.** Vercel Functions have a 4.5 MB request/response limit; 4 MB leaves room for multipart overhead and covers every supplied file.

## Local development

Prerequisites: Node.js 20.9+ and a Postgres database (a free Neon development branch works well).

```bash
npm install
cp .env.example .env.local
# Set DATABASE_URL in .env.local
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). Tables and indexes are created on the first dashboard or upload request.

Verification:

```bash
npm test
npm run lint
npm run build
```

`npm test` includes both cleaning unit tests and a Postgres-engine integration suite (PGlite) that executes the checked-in schema, performs an atomic-style batch/check insert, verifies SLA aggregation and UTC date filtering, and confirms the database uniqueness constraint.

To reproduce the data-quality table against a directory of CSV files:

```bash
npm run profile -- /path/to/csv-directory
```

## Vercel + Neon deployment

1. Import this repository as a new Vercel project (framework preset: Next.js).
2. In the project, open **Storage / Marketplace**, add **Neon**, and choose its free plan.
3. Connect the Neon resource to Production and Preview so Vercel injects `DATABASE_URL`.
4. Redeploy. The first request creates the schema automatically.
5. Upload a supplied CSV, confirm the quality receipt and an incident window, then add the live URL above.

No authentication, multi-tenancy, or CI pipeline is included because those are explicitly out of scope.

## API surface

| Route | Purpose |
| --- | --- |
| `POST /api/uploads` | Accept a `file` multipart field, clean it, and persist one immutable dataset |
| `GET /api/dashboard?uploadId=` | Dataset list, overall/per-service stats, daily signal, and quality receipt |
| `GET /api/logs?uploadId=&from=&to=&service=&state=&page=` | Filtered, newest-first cleaned log records |

## What I would do with more time

- Move files larger than 4 MB through direct object-storage upload and queue chunked ingestion.
- Add explicit SLA policy configuration (success-code ranges, maintenance windows, regional quorum, and monthly credit bands).
- Materialize per-day/per-service rollups after ingestion for very large datasets.
- Add contract tests against an ephemeral Postgres branch and browser-level accessibility tests.
- Export the cleaning receipt and filtered logs as a signed audit artifact.
- Add retention controls and deletion once authentication/authorization enters scope.
