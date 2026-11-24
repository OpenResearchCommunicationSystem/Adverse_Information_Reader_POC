# POC Adverse Information Reader — Ingestion and Processing

**Edition 2.0.** This guide distinguishes four implemented paths. They are not interchangeable stages of one universal crawler. Apply the [neutral naming conventions](README.md#naming-conventions).

## Local upload

```text
multipart files
  -> temporary disk files
  -> ZIP/EML/MBOX parsing
  -> normalized ParsedEmail[]
  -> existing or new dataset
  -> hash + insert messages and attachment metadata
  -> optional inline entity extraction
  -> header/subject thread reconstruction
  -> lightweight sender identity pass
  -> JSON counts + temporary-upload cleanup
```

### Receive and dispatch

The upload handler accepts repeated `files` fields: ten files maximum, 200 MiB per file. Multer writes uploads to temporary disk under sanitized generated names, not an in-memory 100-MB buffer. A `finally` block attempts to unlink the upload files.

- `.zip`: enumerate ZIP entries ending in `.eml`, buffer and parse each using mailparser.
- `.mbox`: read UTF-8, split on lines beginning `From `, parse each segment.
- Other filenames: inspect the first five text bytes for an MBOX separator; otherwise try mailparser on the file.

This is a pragmatic MBOX split, not an exhaustive implementation of every MBOX variant. Entry parse failures can be suppressed. If no valid emails result, return `400`. ZIP parse failure also returns `400`.

**Important boundary:** this upload path does not inherit the inspection service's ZIP-entry, expansion-ratio, worker, or no-network controls. Do not recommend arbitrary untrusted archives on this endpoint merely because inspection is separately restricted. The upload parser collects normalized messages in memory after disk receipt.

### Normalize

The source helper normalizes addresses/names, recipient arrays, dates, and a known exporter metadata prefix. It extracts available attachment/source identifiers separately from message text. Exchange-style directory names and placeholder addresses require source-specific treatment.

The neutral reconstruction input is:

```typescript
interface ParsedEmail {
  messageId?: string;
  inReplyTo?: string;
  references?: string[];
  subject?: string;
  fromAddress?: string;
  fromName?: string;
  toAddresses: string[];
  ccAddresses: string[];
  date?: Date;
  bodyText?: string;
  bodyHtml?: string;
  attachments: Array<{
    filename?: string;
    contentType?: string;
    sizeBytes?: number;
    content?: Buffer;
  }>;
  sourceFilename?: string;
  sourceMemberPath?: string;
  sourceDocumentId?: string;    // neutral alias
  sourceAttachmentId?: string;  // neutral alias
}
```

`References` values are normalized from header representations into a deduplicated string array. Body and HTML remain separate. ZIP upload currently supplies the member basename as the source name/path, so it does not preserve every archive directory component. Do not confuse that with the inspection service's explicit member-path metadata.

### Select a dataset

If `dataset_id` is supplied, validate it against existing rows and append. Otherwise create a dataset using `dataset_name` or joined filenames. Dataset creation and all subsequent processing are not one atomic transaction: partial work can remain after a later failure.

### Hash and insert

1. Load existing message hashes for the target dataset into a set.
2. Hash `messageId`, subject, body text, sender address with literal `||` separators.
3. Mark duplicate if already present; add new hash to the set.
4. Insert the email **even when duplicate**.
5. Insert available attachment metadata, digest, upstream file identity, and source member information.
6. Match available upstream relationships to local records, preserving evidence strength.

The ordinary attachment insert does not save binary content in `file_data`. File hashes can exist even when the bytes are not retained. Nonduplicate and duplicate counters describe inserted row categories; the API's historical `duplicates_skipped` field name is misleading.

Language detection uses Unicode ranges. Core Arabic-script text is checked as Persian first, so Persian/Arabic labels are not a reliable classifier; Cyrillic and CJK checks likewise infer script-based labels rather than establish language or locale.

### Optional entity extraction

For nonduplicate body text longer than ten characters, ingestion normally calls the general model helper inline. It sends only the first 2,000 characters and stores proposed entity type/value/confidence rows.

`runAi:false` skips these calls. Inspection persistence uses this option, keeping deterministic parsing and import independent of model availability. Ordinary upload uses the default model-enabled behavior.

Parsing/model errors can be suppressed, so a successfully inserted message can have no entities because it was empty, not eligible, failed extraction, or produced zero results. The current entity table alone does not distinguish those cases.

### Thread reconstruction

The current implementation is **not subject-only**:

1. Fetch eligible unthreaded messages for the dataset.
2. Normalize message identifiers and build an ID lookup.
3. Use union-find to connect resolvable `In-Reply-To` and `References` identifiers.
4. For messages without recorded header-correlation evidence, apply normalized-subject fallback: remove recognized reply/forward prefixes, trim, lowercase, take the first 100 characters.
5. Build grouped thread rows with message/participant counts, date ranges, and correlation-evidence labels; assign thread IDs to messages.

Only currently eligible unthreaded messages participate. A later import is not guaranteed to join every pre-existing thread, and subject collisions remain possible. The thread listing can find cross-associated rows; that is not proof of automatic global reconstruction.

### Lightweight identities

A deterministic pass groups sender addresses and observed names and creates previously unseen sender records within the dataset. This is distinct from the model-based identity extraction action. The latter samples recent messages and can propose richer attributes.

### Execution and output

Upload ingestion, entity calls, threading, and sender updates run within the HTTP request. There is no durable upload queue or interruption recovery. Response counts can arrive after long processing; temporary-file cleanup does not roll back inserted rows.

## On-demand local enrichment

### Message translation

The individual-message endpoint returns an existing nonempty translation or generates one from the first 4,000 body characters. English returns the source text without a model call. Original and translated fields remain distinct.

### Thread enrichment

The thread action can translate missing bodies, process supported local attachments, and translate their stored text. Operations run sequentially. Existing body translation is skipped when non-null; attachment text and translations have their own presence tests. Content hashes prevent processing identical attachments repeatedly in the same request.

The response reports translated, processed, skipped counters and per-item errors. It does not persist an enrichment job/run manifest.

### Local PDF and image helper

```text
local attachment bytes
  PDF -> embedded text via pdf-parse -> first 6,000 chars -> model cleanup
  image -> model vision transcription plus requested analysis
  existing-text fallback -> selected cleanup paths
      |
      v
  mutable ocr_text -> optional translation -> mutable ocr_translation
```

The PDF path does not render all scanned pages for OCR. Empty extracted text can fall back to existing text; otherwise there is no usable extraction. Raw embedded-text extraction is not separately retained by this workflow.

The image prompt also requests an analysis section; its output is not purely verbatim OCR. The helper has legacy tiny-JPEG placeholder handling with existing text. Non-JPEG image inputs are labeled PNG by the vision payload builder, so broad MIME acceptance should not be read as verified format correctness.

Re-running OCR replaces processed text and clears translation. OCR translation takes the first 6,000 stored characters. There is no translated PDF preserving layout, bilingual page alignment, or immutable OCR/translation version history.

### Thread analysis

The summarization action assembles selected stored representations, then calls a domain-framed analysis prompt. The action does not automatically enrich missing inputs.

It selects the first 20 messages by ascending date; keeps 1,200 characters each for original body/translation and 800 each for attachment text/translation; then truncates the combined packet to 12,000 characters. A subject is provided separately. Attachment hashes suppress duplicates.

Output fields are summary, people, topics, proposed leads. Reanalysis overwrites previous output. Malformed JSON can be saved as raw summary text with empty lists. Coverage and source-span manifests are not implemented.

## Contained source reading

This path reads the sample site's current HTML configuration. It is not dependent on a stable publisher-supplied dataset API.

### Request normalization

The reader accepts allowlisted home, search, collection listing/detail, archive, article, and document display paths. Origin is restricted to configured-in-code source origins. Downloads, original file URLs, and unsupported paths are blocked.

Supported search controls include:

- `query`, trimmed and limited to 160 characters.
- `tabType`: `all|documents|articles`.
- `dataTypes[0]`: `emails|files|csv`.
- `sources[0]`, `sources[1]`, etc., using the neutral upstream key alias; source indexes/values are checked.
- Pagination keys `page`, `p`, `offset`, `cursor`, with bounded values.

Unsupported parameters are not carried through arbitrarily. Search requires a term or supported selected sources. A normalized source identifier may replace its original spelling in an identifier query.

### Fetch and parse

| Bound | Current value |
|---|---|
| Upstream response | 5 MiB |
| Fetch timeout | 25 seconds |
| Minimum fetch interval | 10 seconds |
| Result cache | 5 minutes, up to 100 cache entries |
| Results / links / sections | 40 / 80 / 18 |
| General text bound | 16,000 characters |
| Attachment references / source facets | 100 / 64 |

The backend validates redirects and reads bounded HTML. Parsing extracts visible markup and selected embedded data using source-specific routines, not a complete browser DOM/JavaScript execution environment. Output contains structured text, results, attachments, facets, pagination, links, and warnings.

No remote scripts, forms, cookies, embeds, images, or file downloads are forwarded. Missing visual content or truncated sections must not be presented as a complete reproduction.

### Linkage

Document pages can yield directly observed file/email relationships. Identifier search can yield candidate relationships. Both retain evidence basis and confidence labels; no original file acquisition is needed to store an observed relationship.

Edges persist under the requesting owner. Local matching can associate later imported messages/attachments using upstream IDs or weaker exact-filename evidence. Local SHA-256 hashes remain separate from upstream IDs, even when those IDs look hexadecimal.

### Search semantics

The reader forwards a bounded supported subset of source controls; it does not supply its own exhaustive remote index. A count displayed as a lower bound is not exact, and HTML search cannot be assumed to support Boolean expressions, full attachment text, all pages, or reproducible corpus-wide totals.

Bookmarks retain browser-local navigation state, not immutable captures. The current reader fetches on demand with short caches; it is not a continuous synchronization service.

## Older selected-record relay

Input is a source discriminator, record type, and opaque ID. The adapter constructs a download URL under the selected record route; the public origin is represented as `https://example.invalid`.

A process-local queue serializes jobs and reuses an in-memory result cache. Fetching is paced (default 90 seconds). It parses supported EML/text representations and attachment metadata in a worker thread, then discards original package bytes when no longer referenced.

| Bound | Older relay |
|---|---|
| Download | 50 MiB |
| ZIP entries | 50 |
| Single expanded entry | 25 MiB |
| Expanded total | 50 MiB |
| Text | 120,000 characters |
| Fetch / parser timeout | 45 / 20 seconds |
| Stored job-map target | 100 jobs, pruning finished jobs |

Do not substitute the inspection service's archive limits or restricted-process claims here. The relay is a narrower earlier mechanism, not a durable universal inspection engine.

## Inspection jobs and restart behavior

### Source approval

Accept an approved HTTPS public-file URL or an approved local email/attachment ID. URL validation restricts host, path, credentials, alternate ports, traversal, and redirects. DNS checks reject recognized private-address destinations. Local-record datasets require an explicit allowlist.

These checks are implemented controls, not a comprehensive assurance about DNS races, all address classes, or all parser vulnerabilities.

### Scheduling

New work acquires a process-local active-job slot; URL work also acquires a PostgreSQL advisory lock and shared pacing lease. It writes a job row, schedules execution in the API process, and returns a job identifier.

There is no independent durable worker broker. Completed result reuse can use process memory or an unexpired persisted result. Public URL result reuse uses a source key; local-record keys include owner identity. Created job handles and derivative retrieval remain owner-scoped.

Restart cleanup removes stale temporary directories and marks queued/running persisted jobs failed. It does not rerun them. That update does not comprehensively rewrite every outcome/retryability field, so inspect execution status and error together rather than assuming every failed job is automatically retryable.

### Bounded acquisition and conversion

| Bound | Inspection |
|---|---|
| Download / fetch timeout | 50 MiB / 30 seconds |
| ZIP entries | 1,000 |
| Single expanded entry / expanded total | 10 MiB / 100 MiB |
| Compression ratio | 100 |
| Files inspected | 25 |
| Worker / job time budget | 20 / 120 seconds |
| V8 heap setting | 128 MiB |
| Additional process limits | CPU, process count, output file size, RSS setting |
| Upstream pacing / cache | 90 seconds / 5 minutes |
| Job TTL | 30 minutes |

The 128-MiB value is a V8 heap setting, not a hard guarantee on total worker/system memory. OS resource-limit semantics and namespace availability depend on the host. Worker code also bounds read/output representations. Parent-enforced timeouts and file limits remain necessary.

ZIP checks reject encrypted entries, symbolic links, nested ZIPs, unsafe paths, excessive expansion, and oversized members. Only supported members up to the processing cap are inspected. Unrecognized member extensions are skipped without individual result rows, so the result is not a complete archive-member manifest. Unsupported or skipped members do not become processed merely because the job completed.

### Deterministic worker outputs

| Input | Conversion |
|---|---|
| EML | Parse headers/body and attachment metadata without forwarding active content |
| MSG | Explicitly unsupported in the restricted worker |
| PDF | Extract embedded text; no general raster-page OCR |
| TXT, CSV, JSON, XML | Bounded textual representation |
| HTML/HTM | Textual output with warnings; not active page replay |
| JPEG, PNG, WebP | Decode and re-encode to a metadata-free JPEG |
| Other formats | Unsupported outcome |

Extension recognition does not imply successful conversion. A ZIP can admit an MSG candidate and then report it unsupported. Image re-encoding is not model OCR. Inspection itself does not call a model.

### Outcomes and persistence

Per-file outcomes distinguish processed, unsupported, and safety-rejected content. Job failures separately distinguish upstream refusal/policy decisions from local rejection and temporary service errors. Aggregate success can contain rejected members.

If persistence is requested by the administrator:

1. Resolve/create the target dataset.
2. Normalize supported parsed EML records and ingest with `runAi:false`.
3. Store processed text, CSV with formula neutralization where applicable, or converted JPEG derivatives in private object storage.
4. Insert lineage rows containing owner, job, source/member identifiers, derivative type, hash, size, and warnings.
5. Add authorized derivative references to file metadata; remove temporary inline image bytes from the public result.

A conversion can succeed but yield no storable derivative, changing the file outcome. Failed derivative writes attempt to delete stored objects and derivative rows, but dataset/email writes are not one distributed transaction with object storage and can remain.

Temporary original packages/member files are removed in `finally`; restart cleanup addresses abandoned inspection directories. This retention policy does not apply to optional legacy local attachment binaries or to every other ingestion utility.

## Building against this design

Preserve the separate acquisition paths and representation types. A comparable implementation should expose whether an action reads stored data, fetches a source, converts bytes, or calls a model. [EXTENDING.md](EXTENDING.md) describes recommended improvements without mislabeling them as current behavior.
