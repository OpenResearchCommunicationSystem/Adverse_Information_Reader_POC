# POC Adverse Information Reader — Data Model

**Edition 2.0.** PostgreSQL with Drizzle ORM. Names subject to normalization follow the [naming conventions](README.md#naming-conventions). This is a reconstruction schema reference, not a database dump.

## Relationship map

```text
datasets ----< emails >---- email_threads
   |              |
   |              +----< attachments
   |              +----< entities
   +----< identities

users ----< api_keys
  +-------< inspection_jobs ----< safe_derivatives >---- datasets / emails
  +-------< source_file_email_edges ----< source_local_edge_matches
                                              |              |
                                            emails       attachments

inspection_relay_state             shared singleton pacing state
founder_password_reset_tokens      consumed-token hash registry
founder_password_reset_rate_limits shared recovery throttling
```

The active schema index exports **15 tables** described below. Separate unused conversation/message scaffold files are not exported through that index and are not part of this workflow.

Notation: `?` nullable, `[]` JSONB array, `PK` primary key, `FK` declared foreign key. Timestamps without a stated timezone are ordinary PostgreSQL timestamps in these definitions. Application JSON uses ISO date strings where serialized by Express or explicit conversion.

## 1. `datasets`

| Columns | Type / constraints |
|---|---|
| `id` | serial PK |
| `name` | text, required |
| `description` | text? |
| `created_at` | timestamp, required, default now |

A named local collection. Upload can create one or append to an existing ID. Inspection persistence can similarly select or create a dataset. A dataset does not generally have an owner column or an upstream freshness policy.

## 2. `emails`

| Columns | Type / constraints |
|---|---|
| `id` | serial PK |
| `dataset_id` | integer, required FK datasets |
| `thread_id` | integer? FK email_threads |
| `message_id`, `in_reply_to` | text? |
| `references` | JSONB string[], required, default [] |
| `subject`, `from_address`, `from_name` | text? |
| `to_addresses`, `cc_addresses` | JSONB string[]?, default [] |
| `date` | timestamp? |
| `body_text`, `body_html`, `body_translation`, `language` | text? |
| `has_attachments` | boolean, required, default false |
| `attachment_count` | integer, required, default 0 |
| `content_hash` | text? |
| `is_duplicate`, `is_junk` | boolean, required, default false |
| `source_filename`, `source_member_path` | text? |
| `source_document_id`, `source_attachment_id` | text?; neutral aliases |
| `created_at` | timestamp, required, default now |

Indexes cover dataset, thread, sender, content hash, date, message ID, and upstream document ID. None of those indexes makes a content hash globally unique.

The ingestion hash is SHA-256 of `messageId || subject || bodyText || fromAddress`, using literal `||` separators between values and empty strings for absent values. It is not a hash of the original RFC message bytes. Existing hashes are loaded per dataset. Duplicate rows are still inserted and flagged; the returned duplicate counter does not mean those rows were omitted.

`body_html` is retained data, not an instruction to render untrusted HTML. Source identifiers are independent of the local digest.

## 3. `email_threads`

| Columns | Type / constraints |
|---|---|
| `id` | serial PK |
| `dataset_id` | integer? FK datasets |
| `subject` | text? |
| `email_count`, `participant_count` | integer, required, default 0 |
| `first_date`, `last_date` | timestamp? |
| `summary` | text? |
| `analysis_result` | JSONB? |
| `correlation_evidence` | JSONB string[], required, default [] |
| `created_at` | timestamp, required, default now |

No explicit secondary indexes in the table definition. Message-side thread indexing supports joins.

Grouping uses eligible unthreaded messages in a dataset: normalized Message-ID/reply/reference relationships and normalized-subject fallback. Evidence values include `in_reply_to`, `references`, `normalized_subject`. They describe the grouping method, not a calibrated probability.

Stored analysis convention:

```json
{
  "summary": "Synthetic summary",
  "key_people": ["Example Contact"],
  "key_topics": ["Document revision"],
  "investigative_leads": ["Review the revised attachment."]
}
```

Reanalysis replaces this JSON and summary. No analysis-run table or exact input manifest is currently attached.

## 4. `attachments`

| Columns | Type / constraints |
|---|---|
| `id` | serial PK |
| `email_id` | integer, required FK emails |
| `filename`, `content_type` | text? |
| `size_bytes` | integer? |
| `content_hash` | text?; SHA-256 of available file bytes |
| `source_file_id`, `source_member_path` | text?; source-file name is an alias |
| `is_duplicate` | boolean, required, default false |
| `ocr_text`, `ocr_translation` | text? |
| `file_data` | bytea? via a custom Buffer mapping |

Ordinary ingestion inserts attachment metadata/hash but does **not** populate `file_data`. Other/legacy paths can populate it. OCR requires binary data or a supported stored-text path; metadata alone does not make a file readable.

Within a thread response, repeated content hashes suppress repeated text fields and mark duplicate display occurrences. This does not merge all database attachment rows.

An email has attachment rows, while separate source edges allow a single upstream file identifier to link to multiple upstream emails. Those two relationships must not be collapsed.

## 5. `entities`

| Columns | Type / constraints |
|---|---|
| `id` | serial PK |
| `email_id` | integer, required FK emails |
| `entity_type`, `value` | text, required |
| `confidence` | real? |
| `value_translation` | text? |

Indexes: email, type, raw value. General model extraction asks for `PERSON`, `ORG`, `EMAIL`, `PHONE`, `LOCATION`, `DATE`. `entity_type` is unconstrained text, not a database enum. The separate scanner uses `CRYPTO_ETH`, `CRYPTO_TRX`, `CRYPTO_XMR`, `CRYPTO_BTC_LEGACY`, `CRYPTO_BTC_BECH32`, `CRYPTO_LTC`, `CRYPTO_XRP`, `CRYPTO_DOGE`, `CRYPTO_BCH`, `CRYPTO_SOL`, and `CRYPTO_KEYWORD`.

There is no source-span field or dedicated extraction-run outcome here. Empty results can mean no model findings or a parse failure. Confidence values are detector/model outputs, not measured identity probabilities.

## 6. `identities`

| Columns | Type / constraints |
|---|---|
| `id` | serial PK |
| `dataset_id` | integer? FK datasets |
| `canonical_name` | text? |
| `aliases`, `email_addresses`, `phone_numbers`, `physical_addresses` | JSONB string[]?, default [] |
| `email_count` | integer, required, default 0 |
| `organizations` | JSONB `{name, role}`[]?, default [] |
| `risk_level`, `notes` | text? |

Index: dataset. These are editable research records, not enforced unique real-world persons. AI upsert matches exact canonical name within a dataset; no cross-dataset merge feature is implied.

Counter caveat: the lightweight sender pass counts messages; create/edit and AI upsert use address-array length for `email_count`. Do not treat the field as a consistently computed message total across creation paths. The associated-emails endpoint matches sender addresses, not all recipient mentions or narrative references.

## 7. `users`

| Columns | Type / constraints |
|---|---|
| `id` | serial PK |
| `username` | required unique text |
| `password_hash` | required text; bcrypt |
| `must_change_password` | required boolean, default true |
| `credential_version` | required integer, default 0 |
| `founder_password_bootstrapped_at` | timestamp? |
| `founder_password_reset_token_hash` | text? |
| `created_at` | required timestamp, default now |

Active JWT verification compares the token's version with the current row. Administrator recognition is implemented by a fixed account-name comparison, not a roles table.

## 8–9. Recovery tables

`founder_password_reset_tokens`: serial `id` PK; required unique text `token_hash`; required `used_at` timestamp default now. Stores consumed-token hashes to prevent reuse.

`founder_password_reset_rate_limits`: text `client_key_hash` PK; required timestamp `window_started_at` default now; required integer `attempt_count` default 0. Stores shared throttle state, not raw reset credentials.

## 10. `api_keys`

| Columns | Type / constraints |
|---|---|
| `id` | serial PK |
| `owner_user_id` | required integer FK users |
| `key_hash` | required unique text |
| `key_prefix` | required text |
| `created_at` | required timestamp, default now |
| `revoked_at`, `last_used_at` | timestamp? |

Index on owner and partial unique index on owner where `revoked_at IS NULL`: one active key per owner. The raw generated key is returned only at creation; lookup hashes the supplied key.

## 11. `inspection_jobs`

| Columns | Type / constraints |
|---|---|
| `job_id` | text PK |
| `owner_user_id` | required integer FK users |
| `source_key` | required text |
| `source` | required JSONB object |
| `status` | required text |
| `outcome`, `error` | text? |
| `retryable` | required boolean, default false |
| `result` | JSONB object? |
| `created_at` | required timestamp, default now |
| `started_at`, `completed_at` | timestamp? |
| `expires_at` | required timestamp |

Indexes: owner, source key, expiration. Source object variants carry URL, local email ID, or local attachment ID plus their source identifier; optional persistence options are included in persisted source metadata.

Status is execution state; outcome describes processing/refusal/failure. Results contain per-file outcomes and may be partially useful. TTL is 30 minutes for job visibility in this implementation, with five-minute result reuse. Persisted metadata does not imply resumable job execution.

## 12. `inspection_relay_state`

`id` smallint PK, required `next_fetch_at` timestamp default now, required `failure_count` integer default 0. A singleton row coordinates upstream pacing alongside a PostgreSQL advisory lock. This is not a durable queue table.

## 13. `safe_derivatives`

The name denotes controlled conversions, not a general malware-free guarantee.

| Columns | Type / constraints |
|---|---|
| `id` | serial PK |
| `job_id` | required text FK inspection_jobs.job_id |
| `owner_user_id` | required integer FK users |
| `dataset_id`, `email_id` | integer? FKs datasets/emails |
| `source_id` | required text |
| `upstream_file_id`, `source_page_url`, `package_filename`, `source_member_path` | text? |
| `filename`, `kind`, `media_type` | required text |
| `object_path` | required unique text |
| `size_bytes` | required integer |
| `content_hash` | required text |
| `outcome` | required text |
| `reason` | text? |
| `warnings` | required JSONB string[], default [] |
| `formula_neutralized` | required boolean, default false |
| `created_at` | required timestamp, default now |

Indexes: owner, job, dataset, email, upstream file. Binary derivative data resides in private object storage, not in this row. Retrieval checks owner and processed outcome. Formula-neutralization metadata records when CSV-oriented output changed.

## 14. `source_file_email_edges` (alias)

| Columns | Type / constraints |
|---|---|
| `id` | serial PK |
| `owner_user_id` | required integer FK users |
| `email_document_id`, `file_id` | required text, upstream IDs |
| `filename`, `upstream_href` | text? |
| `source_page`, `display_path`, `evidence_basis`, `confidence` | required text |
| `created_at` | required timestamp, default now |

Unique combination: owner, email document ID, file ID, source page, evidence basis. Secondary indexes: file ID and email document ID.

Reader edge evidence is `document_attachment`/`confirmed` for directly observed relationships, or `identifier_search_result`/`candidate` for inferred search hits. “Confirmed” describes the observed link, not file ownership or factual correctness.

## 15. `source_local_edge_matches` (alias)

| Columns | Type / constraints |
|---|---|
| `id` | serial PK |
| `edge_id` | required integer FK source_file_email_edges |
| `email_id` | required integer FK emails |
| `attachment_id` | integer? FK attachments |
| `evidence_basis`, `confidence` | required text |
| `created_at` | required timestamp, default now |

Unique combination: edge, email, attachment. Secondary indexes: email, attachment. PostgreSQL nullable uniqueness permits multiple null attachment values; do not assume the index alone eliminates every email-only duplicate.

Matching may use upstream file identity or exact filename, recording their different evidence strength. Exact filename is not a content digest. Re-reading source pages and later ingestion can add associations without downloading the original again.

## Representation and lifecycle decisions

- JSONB arrays simplify whole-record updates; they do not provide element-level keys or automatic resolution.
- Most foreign keys do not declare cascading deletion. Dataset deletion and comprehensive retention controls are not available as general REST workflows.
- Original attachment bytes, extracted text, AI-cleaned text, translation, and inspection derivatives have different semantics and lifetimes.
- Ordinary OCR fields and analysis JSON are mutable. Append-only transformation runs and exact passage references are extensions.
- A content hash identifies bytes or a specified input string, not authorship, event truth, or independent corroboration.
- Keep inspection owner checks independent from local data authorization. The schema does not make every imported record tenant-private.

See [API-REFERENCE.md](API-REFERENCE.md) for public shapes, which do not expose all database columns, and [INGEST-PIPELINE.md](INGEST-PIPELINE.md) for how each path writes them.
