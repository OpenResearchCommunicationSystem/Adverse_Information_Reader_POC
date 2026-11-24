# POC Adverse Information Reader — API Reference

**Edition 2.0.** Contracts below follow mounted HTTP handlers inspected on November 24, 2026, not an assumed complete OpenAPI inventory. Apply [neutral aliases](README.md#naming-conventions). All examples are synthetic; the only example host is `https://example.invalid`.

## Conventions

- Prefix: `/api`. A gateway can add a deployment prefix before it.
- JSON unless described as multipart or streamed bytes. Dates serialize to ISO strings or null; optional fields can be absent.
- Most paginated local lists use `{items, total, page, limit, pages}`. Arrays from datasets, statistics, and associated-email endpoints are not wrapped that way.
- Auth labels below: **public**, **JWT**, **admin JWT**, **JWT/key**, or **legacy**. Legacy means the inspected handler has no route-level authentication, **not** that unauthenticated exposure is recommended.
- `Authorization: Bearer <token-or-key>` is required for protected endpoints. API-key management itself requires a JWT; an inspection key cannot mint another key.
- Common explicit errors use `{ "error": "Description" }`. Some add `outcome`, `retryable`, `retry_after_seconds`. Unhandled legacy failures can use Express's default non-JSON error response; there is no universal application error envelope.
- Numeric parsing and body validation are uneven in older routes. Documented types are intended contracts, not a claim that every malformed value is rejected cleanly.

## Route inventory

Every mounted route is listed here. Detailed contracts follow.

| Method | Path after `/api` | Access |
|---|---|---|
| GET | `/healthz` | public |
| POST | `/auth/login` | public |
| POST | `/auth/founder-reset` | public + recovery token/throttle |
| POST | `/auth/change-password` | JWT |
| GET | `/auth/me` | JWT |
| GET | `/auth/api-key` | JWT |
| POST | `/auth/api-key` | JWT + mutation throttle |
| DELETE | `/auth/api-key` | JWT + mutation throttle |
| GET | `/datasets` | legacy |
| POST | `/upload` | legacy |
| GET | `/emails` | legacy |
| GET | `/emails/:id` | legacy |
| POST | `/emails/:id/translate` | legacy |
| GET | `/threads` | legacy |
| GET | `/threads/:id` | legacy |
| POST | `/threads/:id/enrich` | legacy |
| POST | `/threads/:id/summarize` | legacy |
| GET | `/entities` | legacy |
| PATCH | `/entities/:id` | legacy |
| POST | `/entities/:id/suggest` | legacy |
| POST | `/datasets/:id/backfill-entities` | legacy |
| POST | `/emails/:id/entities/translate` | legacy |
| GET | `/identities` | legacy |
| POST | `/identities/extract` | legacy |
| POST | `/datasets/:id/backfill-identities` | legacy |
| POST | `/identities` | legacy |
| PATCH | `/identities/:id` | legacy |
| GET | `/identities/:id/emails` | legacy |
| GET | `/emails/:emailId/attachments` | legacy |
| GET | `/attachments/:id/file` | legacy |
| POST | `/attachments/:id/ocr` | legacy |
| POST | `/attachments/:id/ocr/translate` | legacy |
| POST | `/crypto/scan` | legacy |
| GET | `/crypto/wallets` | legacy |
| GET | `/crypto/summary` | legacy |
| GET | `/crypto/by-dataset` | legacy |
| GET | `/stats` | legacy |
| GET | `/stats/timeline` | legacy |
| GET | `/stats/top-senders` | legacy |
| GET | `/stats/languages` | legacy |
| GET | `/stats/entity-types` | legacy |
| GET | `/research/status` | admin JWT |
| POST | `/research/read` | admin JWT + feature flag |
| GET | `/relay/health` | JWT/key + inspection throttle |
| POST | `/relay/inspect` | JWT/key + inspection throttle |
| GET | `/relay/jobs/:id` | JWT/key + owner + inspection throttle |
| POST | `/inspection/jobs` | JWT/key + inspection throttle |
| GET | `/inspection/jobs/:jobId` | JWT/key + owner + inspection throttle |
| GET | `/inspection/derivatives/:derivativeId` | JWT/key + owner + inspection throttle |

## Health and authentication

### `GET /api/healthz`

`200`: `{"status":"ok"}`. This is a service check, not a test of every dependency or ingestion path.

### `POST /api/auth/login`

Body: `{"username":"researcher","password":"<user-supplied-password>"}`.

`200`: `{"token":"<jwt>","mustChangePassword":true,"username":"researcher"}`.

`400` missing fields; `401` invalid credentials. Successful login signs a seven-day token. The client must retain it and explicitly attach it where required.

### `GET /api/auth/me`

`200`: `{"id":1,"username":"researcher","mustChangePassword":false}`. `401` missing/invalid/stale token.

### `POST /api/auth/change-password`

Body: `{"newPassword":"<new-user-supplied-password>"}`. Minimum six characters in the current handler. Returns the login response shape with a fresh token and `mustChangePassword:false`; `400` invalid password; `401` invalid authentication.

### `POST /api/auth/founder-reset`

Body: `{"resetToken":"<configured-recovery-token>","newPassword":"<new-password>","confirmPassword":"<new-password>"}`.

`200`: `{"ok":true}`. `400` malformed/nonmatching/short password; `401` unauthorized reset or consumed token; `429` throttled; `503` recovery unconfigured/unavailable.

Recovery consumes a token hash once, changes the administrator password, invalidates prior administrator JWTs through credential-version rotation, and revokes active administrator API keys. It is not an email password-reset service.

### `GET|POST|DELETE /api/auth/api-key`

GET reports status:

```json
{"active":false,"prefix":null,"created_at":null,"last_used_at":null}
```

POST creates/rotates and returns the same status plus `key`, shown only once. DELETE revokes and returns status without the key. Mutation throttle: 10/hour/user, in process. `401` invalid JWT; `429` throttle. Key format uses the neutral prefix alias `reader_live_`; do not hardcode an example credential.

## Datasets and upload

### `GET /api/datasets`

Returns an array of `{id,name,description,email_count,thread_count,created_at}`, ordered by creation time. Thread count is based on thread rows' dataset IDs; it is not an exhaustive cross-dataset conversation count.

### `POST /api/upload`

`multipart/form-data`, repeated **`files`** field; up to ten files, 200 MiB each. Optional `dataset_id` appends to an existing dataset; otherwise `dataset_name` or uploaded filenames name a new dataset.

```sh
curl -X POST 'https://example.invalid/api/upload' \
  -F 'files=@sample.eml' -F 'files=@sample-archive.zip' \
  -F 'dataset_name=Example collection'
```

Supported upload dispatch: ZIP containing EML entries, MBOX, and individual mailparser-compatible messages. It is not the inspection service's general format list.

`200`:

```json
{
  "dataset_id": 1,
  "emails_ingested": 2,
  "duplicates_skipped": 1,
  "threads_created": 1,
  "message": "Synthetic result description"
}
```

Important: duplicate rows are inserted and marked, despite the historical response name `duplicates_skipped`. The ingested count excludes flagged duplicates. `400`: no files, invalid ZIP, no valid messages, bad/missing target dataset. Multer limit failures are not normalized by this handler into a guaranteed JSON response.

## Emails

### `GET /api/emails`

| Query | Behavior |
|---|---|
| `dataset_id`, `thread_id` | Numeric filters |
| `search` | SQL `ILIKE '%term%'` over subject, body, sender, saved translation, attachment text/translation |
| `sender` | Sender substring |
| `language` | Exact language code |
| `date_from`, `date_to` | Inclusive date bounds |
| `has_attachments` | Only string `"true"` applies a positive filter |
| `is_junk` | Strings `"true"` / `"false"` |
| `page`, `limit` | Defaults 1 / 50; limit capped 200 |

Rows ordered by email date ascending. `search` is not Boolean parsing; SQL wildcard characters retain pattern semantics.

Item fields: `id,message_id,subject,from_address,from_name,to_addresses,date,has_attachments,attachment_count,language,thread_id,dataset_id,is_junk,body_preview,matched_in`.

`body_preview` is up to 200 characters; `matched_in` is an array containing applicable `translation`, `ocr`, `ocr_translation` labels, not an exhaustive explanation of every matched field. Whole response uses the pagination envelope.

### `GET /api/emails/:id`

Returns stored message fields plus attachments, entities, and trace:

```json
{
  "id": 42,
  "message_id": "<demo-42@example.invalid>",
  "subject": "Document revision",
  "from_address": "sender@example.invalid",
  "from_name": "Example Sender",
  "to_addresses": ["recipient@example.invalid"],
  "cc_addresses": [],
  "date": "2026-01-02T09:00:00.000Z",
  "body_text": "Please review the attached revision.",
  "body_html": null,
  "body_translation": null,
  "language": "en",
  "has_attachments": true,
  "attachments": [{
    "id": 5, "filename": "sample.pdf", "content_type": "application/pdf",
    "size_bytes": 2048, "content_hash": null, "source_file_id": "FILE_DEMO_01",
    "source_member_path": "sample.eml", "is_duplicate": false,
    "ocr_text": null, "ocr_translation": null
  }],
  "entities": [],
  "thread_id": 7,
  "dataset_id": 1,
  "is_junk": false,
  "content_hash": null,
  "trace": {
    "message_id": "<demo-42@example.invalid>",
    "in_reply_to": null,
    "references": [],
    "source_filename": "sample.eml",
    "source_member_path": "sample.eml",
    "source_document_id": "EMAIL_DEMO_01",
    "attachment_edges": [],
    "derivatives": []
  }
}
```

Attachment-edge fields: `file_id,email_document_id,filename,display_path,evidence_basis,confidence`. Derivative trace fields: `id,filename,upstream_file_id,source_member_path`. The trace is metadata; it is not an owner-authorized derivative download in itself. The local message route lacks universal authentication.

Entity objects use `{id,entity_type,value,confidence,email_id,value_translation}`. `400` invalid ID; `404` no message.

### `POST /api/emails/:id/translate`

No required body. Returns `{email_id,original_language,translation}`. Reuses a nonempty saved translation; otherwise translates up to 4,000 body characters and stores the result. English bypasses the model. `400` invalid ID, `404` missing message. This is not a general single-email analysis endpoint.

## Threads

### `GET /api/threads`

Query: `dataset_id`, `search`, `page=1`, `limit=50` (max 200). Search matches **subject only**. Dataset filtering includes thread rows bearing the dataset ID or thread IDs referenced by that dataset's messages. It does not construct new cross-dataset threads.

Pagination envelope; item fields `id,subject,email_count,participant_count,first_date,last_date,dataset_id,summary`. Ordered by last date ascending.

### `GET /api/threads/:id`

Thread metadata plus `analysis_result`, `correlation_evidence`, chronological `emails`, `participants` (sender/to address union), and deduplicated `entities` (type/value key).

Embedded message fields: `id,message_id,subject,from_address,from_name,to_addresses,date,has_attachments,attachment_count,language,thread_id,dataset_id,body_text,body_translation,attachments`.

Attachment fields: `id,filename,content_type,ocr_text,ocr_translation,is_duplicate,content_hash`. Later appearances of an identical nonempty content hash get null text/translation in this response and a duplicate flag. Stored rows remain separate. `400` invalid ID; `404` missing thread.

### `POST /api/threads/:id/enrich`

```json
{"translate_emails":true,"ocr_attachments":false,"translate_ocr":false}
```

All flags default false. Sequentially fills selected missing derivatives; does not create a durable background job. Hash-identical attachments are processed once in the request.

Response: `{translated_emails,ocr_attachments,translated_ocr,skipped_emails,skipped_attachments,errors}`. `errors` is an array of per-item strings, so HTTP success does not imply every item succeeded. Body translation skips non-null existing fields (including empty strings), which differs from the single-message route's nonempty-cache test. Recomputed OCR clears its saved translation.

### `POST /api/threads/:id/summarize`

```json
{
  "include_original": true,
  "include_translation": false,
  "include_ocr": false,
  "include_ocr_translation": false
}
```

Defaults shown. Returns `{thread_id,summary,key_people,key_topics,investigative_leads}` and replaces saved thread analysis. Repeated requests rerun analysis. Selecting translations does not create missing translations.

Coverage: first 20 ascending-date messages; each original/translated body prefix 1,200 characters, each attachment text/translation prefix 800; final assembled packet prefix 12,000. Ordinal email labels are not stable passage citations. Both thread POST routes use `400` invalid ID and `404` missing thread.

## Entities

### `GET /api/entities`

Query: `dataset_id`, `entity_type` exact match, `search` raw or translated value substring, `page=1`, `limit=100` capped 500. Returns pagination envelope of entity objects defined above.

### `PATCH /api/entities/:id`

Body: `{"value_translation":"Example translated value"}`. Returns updated entity. Does not edit raw `value`. `404` missing entity.

### `POST /api/entities/:id/suggest`

No required body. Requests one value translation and returns the entity. Some helper errors return the unchanged entity with `200`; unchanged text is not proof a new suggestion succeeded. `404` missing entity.

### `POST /api/emails/:id/entities/translate`

Translates eligible untranslated person/organization/location values, then returns `{"entities":[...]}` for that email. Existing values are reused; no new mentions are extracted. `404` missing email.

### `POST /api/datasets/:id/backfill-entities`

Extracts from nonduplicate messages with body length over ten and **no existing entity rows of any type**. Returns `{emails_processed,entities_created}`. A prior wallet entity can therefore make a message ineligible for this backfill. Errors can be suppressed; processed counter is not a success counter. `400` invalid dataset ID.

## Identity records

### `GET /api/identities`

Query: `dataset_id`, `search` (canonical name/address-array text), `risk_level`, `page=1`, `limit=50` capped 200. Descending `email_count`; pagination envelope.

Identity object: `{id,dataset_id,canonical_name,aliases,email_addresses,email_count,phone_numbers,physical_addresses,organizations,risk_level,notes}`. Organizations are `{name,role}` pairs. Labels are editable/proposed values, not validated findings.

### `POST /api/identities`

Accepts those fields except generated `id`/`email_count`; optional dataset ID, names/notes nullable, arrays default empty. `201` created object. Sets `email_count` to address-array length, not number of matching messages.

### `PATCH /api/identities/:id`

Partial update of the same editable fields. Supplying `email_addresses` also recalculates `email_count` from that array's length. Returns updated object; `400` invalid ID; `404` missing record.

### `POST /api/identities/extract`

Body `{"dataset_id":1}` required. Selects newest 500 messages; helper samples first 300 and truncates body/packet text. Proposes at most 50 records in the prompt. Upserts by exact canonical name within the dataset. Returns `{created,updated}`; `400` missing dataset ID. It is not an exhaustive person-resolution or merge endpoint.

### `POST /api/datasets/:id/backfill-identities`

**Destructive rebuild:** deletes existing identities for that dataset before model extraction. A failure/empty result can leave previous manual work gone; the full delete/rebuild is not wrapped as one atomic replacement.

Returns `{deleted_existing:true,identities_created}`; `400` invalid ID; `404` missing dataset. Document this separately from the nondestructive-looking extract action.

### `GET /api/identities/:id/emails`

Array of message previews, matched by exact sender membership in `email_addresses`, newest first, maximum 200. No dataset restriction is added by this lookup. Not a search of all recipient/body mentions. `400` invalid ID; `404` missing identity; empty address array returns `[]`.

## Local attachments

### `GET /api/emails/:emailId/attachments`

Returns `{"attachments":[...]}` with attachment metadata and saved text fields; no binary data. `400` invalid parent ID.

### `GET /api/attachments/:id/file`

Streams stored `file_data` with content type and filename headers. `400` invalid ID; `404` missing row/binary. This is a legacy original-file route, not the controlled-derivative endpoint and not protected by its owner check.

### `POST /api/attachments/:id/ocr`

Reads supported available local bytes, stores returned `ocr_text`, clears `ocr_translation`. Response uses **camelCase**: `{"id":5,"ocrText":"..."}`.

`400` invalid ID; `404` no attachment; `422` missing bytes, unsupported type, or no extractable text. PDFs use embedded-text extraction plus model cleanup, not universal scanned-page OCR. Images use the local vision helper.

### `POST /api/attachments/:id/ocr/translate`

Requires saved text; returns **camelCase** `{"id":5,"ocrTranslation":"..."}` and saves it. `400` invalid ID; `404` missing attachment; `422` no stored text. It is separate from extraction.

## Address-pattern scanning

### `POST /api/crypto/scan`

Optional `{"dataset_id":1}`; omission scans all messages. Uses deterministic patterns, not a model or chain query. Returns **camelCase** `{emailsScanned,walletCount,keywordCount}`. The scan replaces previously stored crypto-typed detections within its selected message scope; it is not an append-only detector-run log.

### `GET /api/crypto/wallets`

Optional `type` exact entity-type filter. Response `{wallets,keywords}`; not paginated and not dataset-filtered in this handler.

Wallet group: `{address,type,confidence,cases,occurrences}`. `cases` holds dataset names; occurrences use `{emailId,subject,fromAddress,date,datasetId,datasetName}`. Keyword entries carry source-message metadata. A recognized string does not establish checksum validity, ownership, activity, or a transfer.

### `GET /api/crypto/summary`

`{byType:[{type,count}],uniqueAddresses}` across all stored crypto entities, with keyword marker excluded from unique address count.

### `GET /api/crypto/by-dataset`

Array of `{datasetId,datasetName,totalEmails,uniqueAddresses,keywordHits,byType,topAddresses}` for nonempty datasets. Field counts summarize stored detections, not externally verified addresses.

## Statistics

All accept optional `dataset_id`.

| Endpoint | Additional query | Response |
|---|---|---|
| `GET /api/stats` | none | `{total_emails,total_threads,total_entities,total_identities,duplicate_emails,total_attachments,languages_detected}` |
| `GET /api/stats/timeline` | `granularity=day|week|month`, default month | `[{date,count}]` |
| `GET /api/stats/top-senders` | `limit=20`, max 100 | `[{address,name,count}]` |
| `GET /api/stats/languages` | none | `[{language,count}]` |
| `GET /api/stats/entity-types` | none | `[{entity_type,count}]` |

`languages_detected` counts distinct non-English, non-null labels, not every language category displayed. Counts include stored duplicate messages unless a query explicitly filters them.

## Contained source reader

### `GET /api/research/status`

Admin JWT required even when disabled. Returns `{enabled,founder_only,upstream_origin,max_response_bytes,minimum_fetch_interval_seconds,cache_ttl_seconds,allowed_page_types,containment}`.

### `POST /api/research/read`

Admin JWT and enabled environment required. Strict request:

```json
{"path":"/search?tabType=documents&dataTypes[0]=emails&sources[0]=sample","query":"invoice"}
```

The `sources[0]` name is a reference alias. Request `path` may contain only supported origin/path/query combinations. Search query is limited to 160 characters. Unknown upstream query keys are dropped rather than forwarded.

Response field sketch, with empty arrays used only for this example:

```json
{
  "path": "/search?query=invoice",
  "query": "invoice",
  "page_type": "search",
  "title": "Example results",
  "description": null,
  "fetched_at": "2026-01-02T09:00:00.000Z",
  "cached": false,
  "sections": [],
  "results": [],
  "attachments": [],
  "file_email_edges": [],
  "source_facets": [],
  "pagination": {"current_page":1,"next_path":null,"previous_path":null,"page_paths":[]},
  "links": [],
  "warnings": []
}
```

Nested types:

- Section: `{heading,body}`.
- Result: `{title,excerpt,path,kind,tags,blocked_download,source_identifier,source_dataset,attachments}`. Neutral kinds: `document,email,article,collection,page`.
- Identifier: `{original_value,canonical_value}`. Hex-like IDs can be lowercased; opaque IDs must not be assumed to be local content hashes.
- Attachment: `{file_id,filename,href,path,display_path,source_page,source_document_id,source_identifier,source_dataset}`.
- File/email edge: `{file_id,email_document_id,display_path,evidence_basis,confidence}`.
- Facet: `{source,label,count,count_text,is_lower_bound,selected,disabled}`. A displayed lower bound is not an exact count.
- Link: `{label,path,kind,blocked,source_identifier,source_dataset}`; optional `file_id`. Link kinds `page` and `blocked_download`.
- Pagination entry: `{label,path}`.

Read errors: `400` invalid/unsupported request; `401` invalid token; `403` nonadministrator/disabled; `413` response too large; `429` pacing with `Retry-After`; `502` retrieval/parsing failure. The route validates the normalized response and persists source relationship metadata. It does not create local emails from every page.

## Older selected-record relay

### `GET /api/relay/health`

JWT/key and inspection request throttle. Returns `{status,source,configured,limits}`. Source discriminator is `sample_site` in this reference.

### `POST /api/relay/inspect`

```json
{"source":"sample_site","record_type":"emails","record_id":"DEMO_RECORD_01"}
```

Types: `emails|files|csv`; ID 8–160 letters, digits, underscore, hyphen. Returns `202` `{job_id,status,cached,created_at,status_url}`. A cache hit still creates a job response under this contract.

### `GET /api/relay/jobs/:id`

Owner-scoped. `{job_id,status,request,created_at,started_at,completed_at,result,error}`. Request echoes snake-case fields; result uses `{source,recordType,recordId,sourceUrl,fetchedAt,archiveBytes,documents,warnings}`.

Document fields: required `filename,kind,sizeBytes,warnings`, plus email `subject,from,fromName,to,cc,date,bodyText,attachments`, or plain `text,mediaType` when applicable. Attachment metadata uses `{filename,contentType,sizeBytes}`. Kinds: `email,text,unsupported`. Status: `queued,processing,complete,failed`. Do not substitute inspection status spellings.

`400` invalid request; `401` bad credentials; `404` missing/not-owned job; `429` request throttle. Worker/upstream failures appear on the job. The result cache and queue are process-local, not durable.

## Inspection service

### `POST /api/inspection/jobs`

One source, with optional persistence:

```json
{
  "source_url":"https://example.invalid/attachments/collections/sample/sample.zip",
  "persist_derivatives":false
}
```

Alternative bodies: `{"email_id":42}` or `{"attachment_id":5}`. Local records must belong to a dataset explicitly listed in `INSPECTION_ALLOWED_DATASET_IDS`. Source fields can also be under `record`; positive email ID takes precedence over attachment ID, then URL resolution. Prefer sending only one selector.

Supported URL helpers include `source_url`/`sourceUrl`, an HTTPS URL in `source_id`, an `attachments/` relative source ID, or `dataset_slug` plus relative `source_id`. Host/path allowlists still apply. Neutral URL prefixes are aliases, not a live public-file contract.

`persist_derivatives:true` is administrator-only. Optional top-level `dataset_id` targets an existing dataset; otherwise optional `dataset_name` names a new one. Stored outputs are controlled derivatives and supported parsed email records, not original archives.

New work returns `202`; reusable result returns `200`. Both use:

```json
{
  "job_id":"demo-job",
  "status":"queued",
  "source_id":"email:42",
  "created_at":"2026-01-02T09:00:00.000Z",
  "started_at":null,
  "completed_at":null,
  "result":null,
  "error":null,
  "outcome":null,
  "retryable":false
}
```

### `GET /api/inspection/jobs/:jobId`

Same public shape; owner-scoped, with unexpired persisted fallback. Status: `queued,running,completed,failed`.

Result: `{source,inspected_at,files,warnings,stats,outcome,ingestion?}`.

- Source: `source_id,source_type` plus applicable `url,dataset_slug,local_id`. Neutral source types: `source_url,email_record,attachment_record`.
- File: `{source_id,filename,media_type,text,ocr,metadata,warnings,outcome,reason}`. Per-file outcomes `processed,unsupported,safety_rejected`.
- Stats: `{files_inspected,bytes_processed}`. Bytes can include archive and expanded-member processing, not just network transfer.
- Result outcome: `processed` or `unsupported`. A processed aggregate can coexist with unsupported/rejected members; inspect every file.
- Optional ingestion: `{dataset_id,emails_ingested,derivatives_stored}`.
- Persistence metadata can add `derivative_id,derivative_url,dataset_id,email_id`. Image conversion bytes are not left as base64 in public job metadata.

Job failure outcomes, with neutral aliases where applicable: `upstream_refused,upstream_policy_denied,safety_rejected,unsupported,temporary_failure`. Refusal by the publisher is distinct from rejection by the local parser. `retryable` is generally true only for temporary failure, with a restart-transition caveat in [INGEST-PIPELINE.md](INGEST-PIPELINE.md#inspection-jobs-and-restart-behavior).

Errors include `400` invalid source, `401` authentication, `403` unapproved record/nonadministrator persistence, `404` missing/not-owned job or record, `413` size rejection, `429` active-job/pacing/throttle, and upstream/service failures. Controlled creation errors include `{error,outcome,retryable}` and optional retry timing.

### `GET /api/inspection/derivatives/:derivativeId`

Owner-scoped processed derivative stream. Sets content type, inline filename, `Cache-Control: private, no-store, max-age=0`, `X-Content-Type-Options: nosniff`, and `Content-Security-Policy: sandbox; default-src 'none'`.

`400` invalid ID; `401` invalid credentials; `404` missing/not-owned derivative; `502` object read failure before streaming. Once streaming starts, later failures can terminate the connection rather than return JSON.

Inspection request throttle is 30/minute/user, shared across creation, polling, derivative reads, and older relay requests **within the API process**. Upstream scans additionally use shared pacing/locking. Avoid polling fast enough to consume the entire request budget.

## What is not an API capability here

No general single-email analysis endpoint, universal browser/crawler endpoint, model-driven tool execution, whole-corpus download service, exact-passage citation service, cross-dataset identity merge, general transaction ledger, or translated-PDF layout endpoint is implemented by the routes above.
