# POC Adverse Information Reader — Architecture

**Edition 2.0.** Apply the [reference naming conventions](README.md#naming-conventions) to every path, source field, and configuration alias below.

## 1. Components

```text
Browser: React/Vite + TanStack Query + wouter
  |
  | local API requests; explicit Bearer header for protected operations
  v
Express 5 API (/api)
  +-- Local workspace routes ----------> PostgreSQL / Drizzle
  +-- Model helpers -------------------> OpenAI-compatible model service
  +-- Contained source reader ---------> allowed public HTML
  +-- Selected-record relay -----------> bounded public package + worker thread
  +-- Inspection coordinator ---------> temporary files + restricted Node process
                 |                      (no network in conversion worker)
                 +--------------------> job/lineage rows
                 +--------------------> private controlled-derivative storage
```

The normalized source view is not an iframe containing the publisher's page, a general remote-browser session, or a transparent proxy. The backend retrieves a subset of HTML and interprets it with source-specific parsing. The user's browser receives structured JSON.

## 2. Reference workspace

```text
artifacts/
  reader/                 frontend alias
    src/pages/            dashboard, emails, threads, entities, identities,
                          upload, crypto, research, login, founder-reset
  api-server/
    src/
      app.ts              Express setup
      routes/             HTTP contracts
      lib/
        ingest.ts         local email ingestion/threading
        ai.ts             translation, extraction, thread/identity analysis
        ocr.ts            local PDF/image model-assisted reading
        sample-site.ts    source-specific normalization alias
        research-reader.ts
        relay.ts
        inspection.ts
        derivative-storage.ts
        auth.ts
        api-key.ts
      inspection-worker.cjs
  mockup-sandbox/         development component preview, not a research service
lib/
  db/                     schema and PostgreSQL client
  api-spec/               OpenAPI and generation configuration
  api-client-react/       generated query hooks + custom fetch wrapper
  api-zod/                generated validation schemas
scripts/                  import/maintenance utilities
```

This is a reconstruction map, not a claim that these files are distributed with the guides.

## 3. Runtime and routing

Each service reads its assigned `PORT`; the frontend Vite configuration requires `BASE_PATH`, exposed to browser code as `import.meta.env.BASE_URL`. Both bind for proxied access. The API mounts its router under `/api`, with JSON bodies limited to 256 KiB and URL-encoded parsing enabled. Multipart uploads are handled separately. Express enables CORS and trusts one proxy hop.

The frontend derives an API prefix from its configured base path and appends `/api/...`. A deployment gateway must route that path to the API. Do not hardcode a development host into browser code. The public example `https://example.invalid/api` is illustrative; actual gateway mounting is deployment-specific.

An API reconstruction needs PostgreSQL, its schema applied before use, model configuration for AI operations, and private object storage if derivative persistence is enabled. Restricted inspection additionally needs compatible Linux user/network namespace support, `unshare`, `prlimit`, the Node permission model, and installed parsers. Missing containment facilities are failures, not a reason to execute the worker unrestricted.

## 4. Configuration

Names are documented, never populated with real values here.

| Variable | Purpose |
|---|---|
| `PORT` | Assigned listening port per service |
| `BASE_PATH` | Vite base-path configuration |
| `DATABASE_URL` | PostgreSQL connection configuration |
| `JWT_SECRET` or `SESSION_SECRET` | JWT signing configuration; production startup rejects an absent configured secret |
| `READER_AGENT_PASSWORD` | Initial standard-user bootstrap alias |
| `READER_FOUNDER_PASSWORD` | One-time administrator password bootstrap alias |
| `READER_FOUNDER_RESET_TOKEN` | High-entropy, one-use administrator recovery alias |
| `READER_PRIVATE_RESEARCH_ENABLED` | Enables contained reading only when the string equals `true` |
| `AI_INTEGRATIONS_OPENAI_BASE_URL` | OpenAI-compatible service base |
| `AI_INTEGRATIONS_OPENAI_API_KEY` | Server-side model credential |
| `INSPECTION_ALLOWED_DATASET_IDS` | Comma-separated IDs approved for local-record inspection |
| `RELAY_SCAN_INTERVAL_MS` | Older relay fetch interval; default 90 seconds, bounded at 24 hours |
| `DEFAULT_OBJECT_STORAGE_BUCKET_ID` | Private derivative bucket |
| `PRIVATE_OBJECT_DIR` | Private derivative object prefix |

No source-neutral upstream-origin environment variable is implemented: the inspected adapters use hardcoded source origins and paths. Making those configurable is an extension, not an existing generic connector.

Startup includes targeted additive schema initialization, account bootstrap, sample-data logic, and inspection restart cleanup. This is not a replacement for a managed migration and data lifecycle process.

## 5. Authentication and authorization

Login uses bcrypt verification and issues a seven-day JWT. Client storage is localStorage (reference key `reader_token`). Protected operations verify token validity and load the current user to compare `credential_version`, so an old credential version is rejected.

Administrator recovery uses a configured token, stores/consumes its hash atomically, changes the password, increments the credential version, and revokes active API keys. Reset throttling is stored in PostgreSQL. The first-login UI requires password change, but UI state alone does not protect data endpoints.

| Surface | Current enforcement |
|---|---|
| Login, health | Public entry points |
| Recovery | Configured recovery token and shared throttling |
| Current user/password change | Active JWT verification |
| API-key management | Active JWT; mutation rate limit |
| Contained source reader | Active JWT plus fixed administrator-account check; read additionally gated by environment flag |
| Relay and inspection | Active JWT or active API key; per-user request limits |
| Inspection job/derivative retrieval | Owner check; persistence creation restricted to administrator |
| Local datasets/emails/threads/entities/identities/attachments/stats/crypto/upload | No universal route-level authentication applied in the inspected composition |

The administrator check currently compares a fixed account name, not a generalized role/permission model. No real account identifier is reproduced. API keys are generated randomly, shown once, hashed at rest, and limited to one active key per owner by a partial unique index. Rotation revokes the old key transactionally. Prefixes are display aids, not authenticators.

Inspection HTTP throttling is 30 requests/minute per user, including polling; management mutations are 10/hour. Those buckets are in-process, distinct from shared recovery throttling and database-backed upstream inspection pacing.

**Implementation limit:** several data reads and writes lack backend authorization, and local datasets are not generally tenant-owned. Owner-scoped inspection does not retroactively isolate all imported records. Consistent authorization is required before operational exposure.

## 6. Source reader

1. Validate an allowed origin/path, normalize supported search and pagination parameters.
2. Serve a five-minute cached result or enforce a ten-second fetch interval.
3. Fetch bounded HTML with a 25-second timeout and five-MiB response cap; validate redirects.
4. Parse title, sections, results, facets, pagination, attachments, links, and warnings.
5. Derive observed or candidate attachment relationships and persist their lineage for the authenticated owner.
6. Validate the normalized response and return JSON.

No upstream scripts, forms, cookies, images, embeds, or active downloads are forwarded. Collection, document, search, and article parsing depends on the sample site's layout. The service does not supply the entire corpus or implement the publisher's historical API.

See [INGEST-PIPELINE.md](INGEST-PIPELINE.md#contained-source-reading) for normalized fields and parsing limits.

## 7. Two different job systems

### Older selected-record relay

Requests identify a source, record type (`emails`, `files`, `csv`), and opaque record ID. A process-local queue serializes work, applies pacing, fetches a package, and parses supported email/text members in a worker thread. Job states are `queued`, `processing`, `complete`, `failed`. Jobs and result cache are in memory; they are not recovered after restart.

### Inspection service

Requests identify an approved public-file URL or approved local email/attachment. New work uses `queued`, `running`, `completed`, `failed`. The coordinator keeps a process-local active-job guard and uses a PostgreSQL advisory lock plus shared pacing state for upstream URL work.

Job metadata/results are persisted. Restart marks unfinished persisted jobs failed and removes abandoned temporary working directories; it does not resume interrupted execution. Expiration filters job retrieval, not an automatic deletion policy for all stored rows and derivatives.

Conversion runs in a separate Node process with user/network namespaces, filesystem-read allowlists, Node permission controls, V8 heap setting, resource limits, output bounds, and timeouts. These controls reduce exposure; they do not certify arbitrary content as harmless. The older worker-thread relay and regular upload do not have this same process containment.

## 8. Representations and storage

- Local emails retain parsed body/header/source fields. Local attachments may have metadata, optional legacy binary data, and mutable OCR/translation fields.
- Reader lineage stores upstream file/message edges independently of file bytes.
- Inspection originals live temporarily in a restricted working directory and are removed in success/failure cleanup. Supported derivatives may be persisted to private object storage.
- Private derivative URLs require owner authorization and set `private, no-store`, `nosniff`, and restrictive sandbox headers.
- AI enrichment is downstream of acquisition/conversion. Inspection persistence calls email ingestion with AI disabled.

Do not describe a generated translation as an original, a controlled derivative as the source binary, or local mutable AI fields as immutable processing history.

## 9. Model operations

Calls are server-side OpenAI SDK chat completions using an OpenAI-compatible service. No model credentials go to the browser.

| Operation | Model | Input selection | Completion ceiling |
|---|---|---|---|
| Email translation | `gpt-5-mini` | First 4,000 characters; English bypass returns original text | 4,000 tokens |
| Attachment-text translation | `gpt-5-mini` | First 6,000 characters | 4,000 tokens |
| General entity extraction | `gpt-5-mini` | First 2,000 characters | 1,000 tokens |
| Entity-value translation | `gpt-5-mini` | Supplied entity ID/value/type list | 2,000 tokens |
| Thread analysis | `gpt-5` | First 20 ascending-date messages; body/translation 1,200 each, attachment fields 800 each; combined packet 12,000 characters | 2,000 tokens |
| Identity extraction | `gpt-5` | Route selects newest 500; helper samples first 300, 400 body/translation characters each; packet 14,000 characters | 4,000 tokens |
| Local PDF cleanup/image reading | `gpt-5` | PDF text prefix 6,000; image as vision input; selected text fallback paths | 2,000 tokens |

Input limits are characters, not tokens. The helper may truncate a packet after selecting records, excluding later selected material. Prompts include task/domain-specific framing, not a general neutral factual-analysis guarantee. Identity output includes proposed risk labels; they are not independently validated measurements.

JSON is requested by prompts and parsed, not enforced by a model-side output schema. Entity/identity parse failures can return empty collections; thread parse failure can put raw text in the summary. Some ingestion/backfill exceptions are suppressed. Thread reanalysis and local OCR overwrite stored fields rather than append versioned runs.

## 10. Contracts and extension boundary

OpenAPI generates shared clients and validators, while some routes/UI calls use handwritten contracts. The research read route explicitly validates request and response schemas; do not assume every legacy route does. The [API reference](API-REFERENCE.md) is based on mounted handlers, not an old endpoint count or generated spec alone.

For a reconstruction, keep source adapters, retrieval, deterministic conversion, and optional model analysis independently replaceable. Recommended durable queues, passage-level references, versioned transformations, and stronger authorization are identified as extensions in [EXTENDING.md](EXTENDING.md), not represented as completed features.
