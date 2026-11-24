# Adverse Information Reader POC

## Technical documentation · 2.0

This seven-document series describes an existing proof of concept for searching imported communications, reading selected public source pages, and inspecting selected records from an **Activist Adverse Information Site**. The sample publisher is called **`<SampleWebSite>`** throughout.

The intended reader is a developer or AI-assisted IDE building a comparable process. This is documentation, not a runnable distribution, source-code release, stable upstream specification, or deployment approval. The 2.0 designation identifies this documentation edition and the sample site's current configuration; it is not a versioned public API offered by the publisher.

Descriptions were checked against the current implementation on November 24, 2026. No new production load test, full browser regression, or security assessment was performed for this publication.

## Read in this order

| Guide | Purpose |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Service boundaries, runtime, configuration, authentication, model orchestration |
| [DATA-MODEL.md](DATA-MODEL.md) | Fifteen exported tables, relationships, source identifiers, derivative lineage |
| [API-REFERENCE.md](API-REFERENCE.md) | Mounted route inventory, contracts, errors, and synthetic examples |
| [INGEST-PIPELINE.md](INGEST-PIPELINE.md) | Upload, parsing, normalization, threading, source reading, relay, inspection |
| [FEATURES.md](FEATURES.md) | Current screens and user-triggered actions |
| [EXTENDING.md](EXTENDING.md) | Reconstruction sequence and source/processing/UI extension patterns |

## Naming conventions

These are neutral **reference-design aliases**, not guarantees of byte-for-byte compatibility with an undistributed application:

| Reference name | Meaning |
|---|---|
| `<SampleWebSite>` | The external publisher, never the local application |
| `https://example.invalid` | Reserved, nonoperational origin for all URL examples |
| `artifacts/reader` / `@workspace/reader` | Frontend directory/package alias |
| `sample-site.ts` | Source-specific normalization helper alias |
| `source-lineage.ts` | Source relationship schema module alias |
| `source_document_id`, `source_file_id`, `source_attachment_id` | Upstream identifiers stored on local records |
| `source_file_email_edges`, `source_local_edge_matches` | Source relationship table aliases |
| `source_facets`, `sources[0]` | Normalized source-facet property and upstream filter-key aliases |
| `/collections`, `/collection/:slug` | Collection listing/detail route aliases |
| `/attachments/collections/:slug/:member`, `/files/collections/:slug/:member` | Approved public-file path aliases |
| `sample_site` | Source discriminator used in relay examples |
| `source_url`, `upstream_refused`, `upstream_policy_denied` | Neutral source-type and outcome values where source-branded values exist internally |
| `READER_PRIVATE_RESEARCH_ENABLED`, `READER_FOUNDER_PASSWORD`, `READER_AGENT_PASSWORD`, `READER_FOUNDER_RESET_TOKEN` | Source-neutral configuration aliases |
| `reader_token`, `reader-research-bookmarks`, `reader_live_` | Browser storage names and generated-key prefix aliases |

When reconstructing the design, choose these names or your own consistently across schema, API, adapter, and client. Match a publisher's actual routes and field names inside its adapter; do not assume the aliases are reachable upstream. Unmarked endpoint paths and ordinary fields describe current behavior. Example records, counts, and dates are synthetic, not measurements or real correspondence. All URL examples use the reserved domain; none identifies a live publisher or service.

## Four distinct workflows

1. **Local workspace:** import supported email material; store messages and relationships; search, translate, extract entities, and review threads.
2. **Contained source reader:** fetch bounded allowlisted HTML server-side; return normalized JSON text and metadata rather than active upstream HTML.
3. **Selected-record relay:** request a public record package by type and opaque ID; receive bounded email/text results through an in-memory job.
4. **Inspection service:** inspect an approved public URL or allowlisted local record using restricted conversion; report per-file outcomes; optionally persist controlled derivatives.

These workflows do not share identical limits, supported formats, authorization, retention, or execution models. In particular, the local upload path is not the restricted inspection pipeline.

## Implemented functions and boundaries

| Function | Current capability | Boundary |
|---|---|---|
| Email search | Subject, body, sender, saved translation, attachment text/translation; filters and pagination | SQL substring matching, not a general Boolean or ranked search engine |
| Thread reconstruction | Message/reply/reference headers plus normalized-subject fallback | Limited to eligible unthreaded records within a dataset |
| Thread enrichment | Body translation, supported attachment processing, translation of stored attachment text | Sequential request-bound work |
| Thread analysis | Summary, people, topics, proposed leads from selected representations | First 20 dated messages, per-field and total character cutoffs |
| Entities | Six general mention types, translations, separate address-pattern scanner | Email-level references; no exact-span extraction or complete resolution engine |
| Identity cards | Editable names, aliases, contacts, affiliations, notes | AI input is sampled; no cross-dataset merging feature |
| PDF reading | Embedded-text extraction and AI cleanup in the local attachment helper | No general scanned-PDF raster OCR or translated-layout reconstruction |
| Source reading | Allowed pages, filters, pagination, file references, lineage | Text-only normalized view; original downloads blocked |
| Inspection | Restricted deterministic parsing/conversion with explicit outcomes | Supported formats only; not malware certification or exhaustive archive processing |

## Reconstructing a comparable process

Start with local persistence and retrieval, then add threading and relationships, then source adapters, bounded inspection, and optional AI enrichment. See the ordered implementation guide in [EXTENDING.md](EXTENDING.md).

The inspected implementation uses React 19, Vite 7, TanStack Query, wouter, Tailwind/shadcn components, Express 5, Node.js/TypeScript, PostgreSQL/Drizzle, and server-side OpenAI-compatible chat-completion calls. API clients and Zod schemas are generated from OpenAPI with Orval, but not every route or call is generated.

The frontend sign-in gate is **not universal backend authorization**. Several local data and mutation endpoints lack route-level authentication. Reconstructors should apply consistent authorization before exposing data, rather than copy this limitation as a recommended pattern.

## Scope and packaging

This publication contains only these seven Markdown guides. It does not include records, binaries, application source, runtime credentials, repository history, or an upstream dataset catalogue. It does not assert that all published information has the same handling requirements or that any source-specific adapter is universally compatible. No new licensing choice is made by these documents.
