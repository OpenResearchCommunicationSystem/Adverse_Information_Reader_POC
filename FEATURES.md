# POC Adverse Information Reader — Feature Reference

**Edition 2.0.** This guide describes implemented screens and actions. Paths and labels follow [neutral reference conventions](README.md#naming-conventions); they do not rename a deployed application.

## Authentication and navigation

The React app uses wouter. Its base comes from the Vite base path. A sign-in gate shows login when no user is loaded and a password-change dialog when the account requires it. Administrator recovery has a separate entry flow. The shared dataset context preserves the current selection across local workspace screens.

The sidebar provides dashboard, emails, threads, entities, identity cards, upload, crypto detections, and Research Reader. A dataset selection is relevant to local views; it does not transform source-site browsing into a query over a locally complete copy.

Authentication UI calls login/current-user/password-change endpoints. An API-key management control supports protected integration use: status, generate/rotate, one-time display, revoke. No key appears in this publication.

**Boundary:** a UI gate is not backend authorization. See the [architecture access matrix](ARCHITECTURE.md#5-authentication-and-authorization).

## Dashboard · `/`

Displays counts for selected/all datasets, message timeline, leading senders, language labels, and entity categories.

Calls: `GET /api/stats`, `/stats/timeline`, `/stats/top-senders`, `/stats/languages`, `/stats/entity-types`, plus dataset selection.

No model calls occur merely to display these charts. They reflect stored rows, including duplicates where not filtered. Script-based language labels and legacy identity counts should not be interpreted as validated coverage metrics.

## Email Explorer · `/emails`

### Find

The list supports dataset, search, sender, language, date range, attachment-presence, and junk-state filters. The backend text search includes saved body translations and attachment text/translations. It uses SQL substring matching, not a general ranked full-text or Boolean engine.

Cards show message metadata, preview, and selected derivative-match indicators. Opening a card fetches the complete stored message rather than relying on its preview.

### Read and enrich

The detail panel displays sender/recipient/date information, body and stored translation, extracted entities, attachments, and available source trace. Actions can:

- Translate a body.
- Translate eligible entity values in a batch.
- Suggest or manually edit an entity translation.
- Process a supported local attachment and translate its saved text.
- Retrieve a retained local file where bytes exist.

Reads do not themselves generate a summary. Translation and OCR actions call the backend; model calls are server-side. No general “analyze this one email” endpoint equivalent to thread analysis is implemented.

Source trace can include reply/reference headers, original filename/member information, upstream identifiers, attachment relationship evidence, and derivative metadata. Presence of a derivative reference is not proof every byte was read by a model.

### Endpoint mapping

`GET /api/emails`, `GET /api/emails/:id`, `POST /api/emails/:id/translate`, `POST /api/emails/:id/entities/translate`, `PATCH /api/entities/:id`, `POST /api/entities/:id/suggest`, `POST /api/attachments/:id/ocr`, `POST /api/attachments/:id/ocr/translate`, `GET /api/attachments/:id/file`.

## Threads · `/threads`

List entries show subject, counts, and dates. Opening a thread displays chronological messages, participants, stored representations, attachment panels, deduplicated entities, and existing analysis.

### Preparation controls

Translation, attachment processing, and attachment-text translation call `POST /api/threads/:id/enrich` with corresponding flags. Counts/toasts report completed/skipped operations and available errors. This is request-bound processing, not a job queue.

Identical attachment hashes suppress duplicate processing/display text in the applicable thread paths. The source message relationships remain separate.

### Analysis controls

The user chooses original body, body translation, attachment text, and attachment translation, then calls `POST /api/threads/:id/summarize`. Output displays summary, people, topics, and proposed leads.

Selecting a representation does not create it. The model receives a bounded subset: first 20 messages plus character cutoffs, not necessarily everything visible in the thread. The analysis uses a domain-framed prompt and replaces saved output; it is not a factual determination or versioned analysis record.

List search matches subject only. Grouping uses header relationships with subject fallback, not a guarantee that every related historical message has been discovered.

Calls: `GET /api/threads`, `GET /api/threads/:id`, and the two POST actions above.

## Entities · `/entities`

Paginated list with dataset/type/search filters, type badges, raw/translated values, source-email references, confidence display, and copy controls. This page is a list, not an interactive relationship graph. Value translation editing/suggestion is handled from email detail.

Calls: `GET /api/entities`, dataset listing. No model call is necessary to read this page.

General mention extraction supports six types. A filter offering another label does not prove the general model helper detects it. Current references point to emails, not exact text spans.

## Identity cards · `/identities`

Grid of editable research profiles with canonical name, aliases, address/contact fields, affiliations, notes, associated messages, and legacy risk labels.

Actions:

- Create a record.
- Edit a record.
- Extract proposed records with AI for the selected dataset.
- Open associated sender-matched emails.

Calls: `GET /api/identities`, `POST /api/identities`, `PATCH /api/identities/:id`, `POST /api/identities/extract`, `GET /api/identities/:id/emails`.

AI extraction is sampled/truncated; matching/upsert is not a full resolution engine. Associated messages are sender-based and capped, not all possible appearances of the person. The stored count can have different meanings depending on creation path.

No user-facing identity deletion, cross-dataset merge, or network-graph capability is claimed. The separate identity backfill API deletes/rebuilds records and must not be confused with ordinary editing or extraction.

## Crypto detections · `/crypto`

Displays stored address-pattern groups, occurrence metadata, type summaries, keyword hits, and dataset breakdowns. A scan action invokes `POST /api/crypto/scan`.

The scanner is regex-based and runs separately from general model entity extraction. It does not query a blockchain or establish ownership/payment activity.

Calls: `GET /api/crypto/wallets`, `/crypto/summary`, `/crypto/by-dataset`, `POST /api/crypto/scan`. Several reads aggregate across datasets even when another screen has a selected dataset; use actual endpoint filters rather than assuming global context applies everywhere.

These calls use handwritten fetch/query logic rather than assuming every endpoint has a generated hook.

## Upload · `/upload`

Supports selection/drop of EML, MBOX, and ZIP files, up to ten at 200 MiB each. Users choose an existing dataset or a new dataset name. The browser submits repeated `files` multipart fields and displays ingestion/duplicate/thread counts.

Calls: `GET /api/datasets`, `POST /api/upload`. Ordinary ingestion can trigger model entity extraction per eligible message; OCR and translation are not automatically performed on every attachment.

The upload action can run for a long time. The returned duplicate field name does not mean duplicate rows were not stored. Parse failures can be skipped without a complete per-member failure list.

The upload parser is distinct from restricted inspection and does not have the same archive containment.

## Research Reader · `/research`

Administrator-only and environment-enabled. Shows selected pages from `<SampleWebSite>` through normalized text and metadata, not an embedded active version of the site.

Implemented interface elements include:

- Home/search/collection/archive/article/document navigation.
- Search terms and supported source/type controls.
- Back/forward navigation state.
- Browser-local bookmarks.
- Source facets, including lower-bound counts and disabled/selected states.
- Result links, pagination, file references, and relationship information.
- Fetch/cache metadata, warnings, loading/error states.

Calls: `GET /api/research/status`, `POST /api/research/read`, with explicit authenticated requests.

Original download links remain blocked in this reading path. Images/scripts/forms/cookies are not forwarded. A bookmark saves a destination, not an immutable capture. Parsing is site-specific and bounded, so the rendered page is not a promise of complete upstream content.

## Inspection and relay: API capabilities versus screens

The backend exposes selected-record relay jobs, inspection jobs, and protected derivative retrieval. The inspected Research Reader page is not a general job-submission/polling console for these APIs. Do not turn an API capability into an invented UI feature.

A developer can build such a client from [API-REFERENCE.md](API-REFERENCE.md). Existing endpoint availability does not mean one current button automatically fetches, converts, translates, imports, and analyzes every linked record.

## Representation distinctions to preserve

| Shown material | Meaning |
|---|---|
| Source body | Parsed stored correspondence |
| Translation | Generated reading aid, possibly prefix-limited |
| Local OCR/text | Extracted/AI-processed text, sometimes including analysis |
| Thread analysis | Generated summary and proposals over selected inputs |
| Contained page | Normalized subset of fetched source HTML |
| Controlled derivative | Restricted conversion product with its own lineage and access checks |

These are implementation distinctions, not generic claims about the identities mentioned in the data. [INGEST-PIPELINE.md](INGEST-PIPELINE.md) explains the actual transformations.
