# Iteration 15 — Remote Knowledge Ingestion (Execution-Ready Prompt)

You are implementing **Iteration 15 — Remote Knowledge Ingestion** for `codex_context_engine`.

Your job is to extend the engine so it can ingest remote documentation sources through URLs, materialize them as reproducible local artifacts, and then pass those artifacts through the **existing local knowledge pipeline**.

This iteration must **not** turn retrieval into live web browsing.
It must only add a **remote acquisition layer** before the existing processing flow.

---

## Core intent

The engine currently works best with local documents placed inside mods and then processed into notes, summaries, indices, and manifests.

This iteration adds the ability to:

1. register a URL as a source for a mod
2. fetch and snapshot the source locally
3. extract normalized text from the fetched content
4. convert the result into a canonical local document
5. feed that document into the existing processing pipeline

The final retrieval path must remain:
- deterministic
- local
- reproducible

---

## Non-goals

Do **not** implement:
- open-ended web crawling
- recursive site exploration
- live retrieval from the web during context recovery
- JS-heavy rendering pipelines unless already trivial to support
- semantic ranking changes unrelated to ingestion
- broad MCP orchestration beyond what is strictly needed here

---

## Architectural principle

**Remote acquisition, local reasoning.**

All URL-based sources must be converted into local artifacts before they become part of engine knowledge.

---

## Implementation scope

Implement the following capabilities:

### 1. Source registration
The engine must support registering URL sources under a mod.

New command shape:

```bash
codex-context mod add-source <mod_name> --url <url> [--type html|pdf|md|txt|auto] [--tag <tag>]...
```

Expected behavior:
- create the mod if it does not exist only if the current engine conventions already support that; otherwise fail clearly
- persist the source entry into a remote source manifest
- avoid duplicate entries for the same canonicalized URL
- default type is `auto`
- multiple `--tag` flags are allowed

### 2. Source fetching
New command shape:

```bash
codex-context mod fetch-sources <mod_name> [--source-id <id>] [--force]
```

Expected behavior:
- fetch all registered sources for the mod unless `--source-id` is provided
- materialize a local snapshot
- detect content type
- store raw payload
- extract usable text
- emit canonical local documents into the mod inbox
- update metadata and fetch status
- skip unchanged sources unless `--force` is set and the implementation can reasonably detect unchanged content

### 3. Canonical local materialization
Every fetched source must produce:
- a raw artifact
- a snapshot manifest
- an extracted normalized text artifact
- a canonical inbox document to be consumed by the existing learning pipeline

### 4. Format support
Support these input types:
- HTML
- PDF
- Markdown
- TXT

Behavior:
- `auto` attempts best-effort detection from headers, extension, and content sniffing
- if unsupported, mark source as failed with a clear error message

### 5. Local-first downstream compatibility
Do not replace the existing learning pipeline.
Instead:
- generate canonical local documents
- place them where the current processing pipeline can consume them
- preserve traceability back to the remote URL

---

## Directory layout

Inside each mod, add:

```text
.codex_library/mods/<mod_name>/
  remote_sources/
    manifest.json
    raw/
    snapshots/
    extracted/
```

### Folder semantics

- `manifest.json`
  - registry of remote sources for the mod

- `raw/`
  - raw fetched payloads, such as original html, pdf, md, or txt

- `snapshots/`
  - per-fetch metadata records, one per fetch event or latest-per-source depending on implementation choice
  - must include enough information for reproducibility

- `extracted/`
  - normalized text artifacts produced from raw content
  - these are intermediate artifacts, not final retrieval files

The existing mod folders such as `inbox/`, `processed/`, `notes/`, `summaries/`, `indices/`, and `manifests/` must continue to work as before.

---

## Data contracts

### A. Remote sources manifest

Create:

```text
.codex_library/mods/<mod_name>/remote_sources/manifest.json
```

Suggested schema:

```json
{
  "version": 1,
  "sources": [
    {
      "id": "android_guide",
      "url": "https://developer.android.com/guide",
      "canonical_url": "https://developer.android.com/guide",
      "declared_type": "auto",
      "detected_type": null,
      "tags": ["android", "official-docs"],
      "status": "pending",
      "created_at": "2026-03-29T00:00:00Z",
      "updated_at": "2026-03-29T00:00:00Z",
      "last_fetched_at": null,
      "last_successful_snapshot_id": null,
      "last_error": null
    }
  ]
}
```

Rules:
- `id` must be stable and filesystem-safe
- `canonical_url` should normalize trivial URL differences when reasonable
- `status` allowed values:
  - `pending`
  - `fetched`
  - `processed`
  - `failed`

### B. Snapshot manifest

Each fetch should create a snapshot record under `snapshots/`.

Suggested shape:

```json
{
  "snapshot_id": "android_guide_20260329T101530Z",
  "source_id": "android_guide",
  "url": "https://developer.android.com/guide",
  "canonical_url": "https://developer.android.com/guide",
  "fetched_at": "2026-03-29T10:15:30Z",
  "declared_type": "auto",
  "detected_type": "html",
  "content_type_header": "text/html; charset=utf-8",
  "http_status": 200,
  "checksum_sha256": "abc123...",
  "raw_path": "raw/android_guide_20260329T101530Z.html",
  "extracted_path": "extracted/android_guide_20260329T101530Z.md",
  "inbox_path": "../inbox/remote_android_guide_20260329T101530Z.md",
  "title": "Android Developers Guide",
  "etag": null,
  "last_modified": null,
  "word_count": 1234,
  "extraction_notes": [
    "Removed navigation boilerplate",
    "Preserved headings",
    "Preserved code blocks when detected"
  ]
}
```

### C. Canonical inbox document

Each successful fetch must generate a canonical local document in `inbox/`.

Suggested format: Markdown with frontmatter-like metadata block or JSON if the current engine strongly prefers JSON.

Default recommendation: Markdown.

Suggested content template:

```md
---
source_kind: remote_url
source_id: android_guide
snapshot_id: android_guide_20260329T101530Z
source_url: https://developer.android.com/guide
canonical_url: https://developer.android.com/guide
detected_type: html
fetched_at: 2026-03-29T10:15:30Z
title: Android Developers Guide
tags:
  - android
  - official-docs
---

# Android Developers Guide

[Normalized extracted content goes here]
```

Requirement:
- preserve traceability
- be easy for the existing learning pipeline to consume
- avoid inventing a second downstream processing system

---

## Extraction rules

### HTML
For HTML extraction:
- remove obvious navigation, cookie banners, footers, and repeated boilerplate when practical
- preserve:
  - document title
  - headings
  - paragraphs
  - lists
  - tables if reasonably representable as text
  - code blocks
- prefer semantic structure over raw DOM dump
- do not attempt full browser rendering unless your current stack already makes it trivial

### PDF
For PDF extraction:
- extract text in reading order as well as reasonably possible
- preserve page boundaries only if useful
- do not OCR by default
- if text extraction fails, mark the source as failed with a clear reason

### Markdown / TXT
For Markdown or TXT:
- preserve content mostly as-is
- normalize line endings
- ensure UTF-8 handling

### Normalization rules
Across all formats:
- normalize whitespace without flattening all structure
- preserve headings where possible
- preserve fenced code blocks if detected
- preserve source title
- record extraction notes

---

## Source ID generation

Implement deterministic source IDs.

Preferred strategy:
1. derive from hostname + meaningful path slug when possible
2. sanitize to lowercase filesystem-safe text
3. if collision occurs, append a short hash suffix

Example:
- `https://developer.android.com/guide/topics/providers/content-providers`
- becomes
- `developer_android_com_guide_topics_providers_content_providers`

Shorter IDs are acceptable if stable and readable.

---

## URL canonicalization

Implement lightweight canonicalization:
- lowercase scheme and host
- remove default ports
- trim trailing slash when safe
- remove fragments
- keep query params only when they are likely content-significant
- avoid over-aggressive normalization that merges different docs incorrectly

If full canonicalization is too risky, keep it conservative.

---

## CLI behavior details

### `add-source`
Must:
- validate URL syntax
- load or create manifest file
- canonicalize URL
- refuse duplicates by canonical URL
- persist source entry
- print a clear success message with source ID

### `fetch-sources`
Must:
- load manifest
- optionally filter by source ID
- fetch each source
- store raw file using extension based on detected type
- create snapshot record
- create extracted normalized artifact
- create canonical inbox document
- update source status and metadata
- continue processing other sources if one fails
- return non-zero exit only if all selected sources fail or if there is a fatal setup error

### Suggested optional future command
Do **not** implement unless trivial, but structure for future support:
```bash
codex-context mod refresh-source <mod_name> --source-id <id>
```

---

## Error handling

Errors must be explicit and persisted.

Examples:
- invalid URL
- unsupported content type
- fetch timeout
- non-200 HTTP status
- PDF extraction failure
- extraction produced empty content

On failure:
- set source status to `failed`
- update `last_error`
- do not crash the entire batch unless setup itself is broken

---

## Observability

Add lightweight logs for:
- source registered
- source skipped
- source fetched
- snapshot created
- extraction completed
- inbox document emitted
- failure reason

Avoid noisy logs.

---

## Integration constraints

This iteration must fit the current engine design.
Prefer:
- small, composable functions
- explicit manifests
- local files as system-of-record
- compatibility with future knowledge processing stages

Do not refactor unrelated parts of the engine unless required to integrate this feature cleanly.

---

## README updates required

Update the global README so the engine clearly documents:
1. what Iteration 15 adds
2. the new remote ingestion directory layout
3. the new CLI commands
4. the principle that retrieval remains local and reproducible
5. examples of using official docs as mod sources

---

## Acceptance criteria

Iteration 15 is complete when:

1. A user can register a URL source under a mod.
2. The source is persisted in a manifest.
3. Fetching sources materializes raw artifacts locally.
4. Fetching sources produces extracted normalized text.
5. Successful fetches emit canonical inbox documents for the existing pipeline.
6. Failures are recorded without collapsing the whole batch.
7. README documentation is updated.
8. The design clearly preserves local-first retrieval.

---

## Example happy path

```bash
codex-context mod add-source android_core --url https://developer.android.com/guide --tag android --tag official-docs
codex-context mod fetch-sources android_core
codex-context mod learn android_core
```

Expected result:
- URL registered
- HTML fetched
- raw artifact saved
- normalized markdown extracted
- canonical document emitted into inbox
- existing learning pipeline processes it

---

## Deliverables

Produce:
1. implementation code
2. tests where appropriate
3. README updates
4. any minimal helper modules required for fetch/extract/materialize flow

Keep the implementation practical, minimal, and aligned with the engine’s existing style.