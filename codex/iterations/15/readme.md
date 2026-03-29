# Iteration 15 — Remote Knowledge Ingestion

## Overview

Iteration 15 adds a **remote acquisition layer** to `codex_context_engine`.

The goal is not to make the engine depend on live web access during retrieval.
Instead, this iteration allows the engine to **ingest documentation from URLs**, convert that content into **local, reproducible artifacts**, and then feed those artifacts into the **existing local knowledge pipeline**.

This keeps the engine aligned with its core philosophy:

> **remote acquisition, local reasoning**

---

## Why this iteration exists

Until now, the most reliable way to feed knowledge into mods has been to place local files directly inside the mod and let the engine process them.

That works well, but it introduces friction:

- the user has to find the right documentation page
- download it manually
- store it in the correct folder
- then run the processing pipeline

Iteration 15 removes that friction by letting the engine:
- register URLs as sources
- fetch them
- snapshot them locally
- extract normalized text
- materialize canonical local documents
- reuse the current processing stages

The result is a smoother path from **official documentation URL** to **usable local knowledge**.

---

## Core principles

### 1. Remote acquisition, local reasoning
URLs are only used during the ingestion phase.
The engine must not depend on live web access during retrieval.

### 2. Reproducible local snapshots
Every fetched source must become a local artifact with metadata.
The engine should always know:
- what URL was fetched
- when it was fetched
- what type it was
- where the raw and extracted outputs are stored

### 3. Reuse the existing knowledge pipeline
This iteration does not replace notes, summaries, indices, or manifests.
It feeds those stages by emitting canonical local documents into the mod’s normal flow.

### 4. Conservative scope
Iteration 15 is not a crawler.
It should support direct URL ingestion cleanly before attempting recursive or large-scale web acquisition.

---

## What this iteration adds

Inside each mod:

```text
.codex_library/mods/<mod_name>/
  remote_sources/
    manifest.json
    raw/
    snapshots/
    extracted/
```

### New capabilities
- register URL-based sources under a mod
- fetch registered sources
- snapshot content locally
- extract normalized text from HTML, PDF, Markdown, and TXT
- emit canonical inbox documents for the existing learning pipeline

### New CLI commands
```bash
codex-context mod add-source <mod_name> --url <url> [--type html|pdf|md|txt|auto] [--tag <tag>]...
codex-context mod fetch-sources <mod_name> [--source-id <id>] [--force]
```

---

## Directory structure

### `remote_sources/manifest.json`
Registry of remote URL sources for the mod.

### `remote_sources/raw/`
Stores the raw fetched payloads exactly as retrieved or in minimally modified form.

Examples:
- `.html`
- `.pdf`
- `.md`
- `.txt`

### `remote_sources/snapshots/`
Stores per-fetch metadata records so each acquisition can be traced and reproduced.

### `remote_sources/extracted/`
Stores normalized text extracted from the raw sources.
These files are intermediates, not final retrieval files.

### `inbox/`
Successful remote ingestions must emit a canonical local document into the existing `inbox/` so the rest of the engine can continue to work without needing a second processing system.

---

## Manifest contract

Suggested shape:

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

### Status values
- `pending`
- `fetched`
- `processed`
- `failed`

---

## Snapshot contract

Each fetch should create a snapshot metadata record.

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

---

## Canonical local document

Each successful ingestion must produce a local document in `inbox/`.

Recommended default format: Markdown.

Suggested template:

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

[Normalized extracted content]
```

This document becomes the bridge between URL ingestion and the rest of the engine.

---

## Supported input formats

### HTML
The extractor should:
- remove obvious boilerplate when practical
- preserve headings, paragraphs, lists, tables when possible, and code blocks
- prefer semantic structure over raw DOM dumps

### PDF
The extractor should:
- use text extraction only
- avoid OCR by default
- fail clearly if extraction is not possible

### Markdown
Preserve content mostly as-is, normalizing line endings and encoding.

### TXT
Preserve content mostly as-is, with basic normalization.

---

## URL handling

### Canonicalization
Use conservative canonicalization:
- lowercase scheme and host
- remove fragments
- trim trailing slash when safe
- avoid aggressive query stripping unless clearly safe

### Source IDs
Generate stable filesystem-safe IDs derived from URL structure where possible.
If collisions occur, append a short hash.

---

## Error handling

Failures must be recorded in the manifest and should not collapse the whole batch.

Examples:
- invalid URL
- timeout
- unsupported type
- non-200 response
- extraction failure
- empty extraction output

On failure:
- set status to `failed`
- update `last_error`
- continue processing other sources when possible

---

## CLI behavior

### Add a source
```bash
codex-context mod add-source android_core --url https://developer.android.com/guide --tag android --tag official-docs
```

Expected result:
- URL validated
- canonical URL stored
- source ID generated
- source entry written to manifest
- duplicate URLs rejected

### Fetch sources
```bash
codex-context mod fetch-sources android_core
```

Expected result:
- sources loaded from manifest
- raw content fetched
- snapshot metadata written
- normalized text extracted
- canonical inbox document emitted
- manifest updated

Optional scope:
```bash
codex-context mod fetch-sources android_core --source-id android_guide
```

---

## Example workflow

```bash
codex-context mod add-source android_core --url https://developer.android.com/guide --tag android --tag official-docs
codex-context mod fetch-sources android_core
codex-context mod learn android_core
```

Flow:
1. register source
2. fetch source
3. snapshot locally
4. extract normalized text
5. emit canonical inbox document
6. process with existing learning pipeline

---

## Future improvements

These are explicitly out of scope for the first implementation, but should remain compatible with the design:

- deduplication across fetched sources
- refresh policies
- incremental updates
- richer extraction heuristics
- optional recursive crawling with strict boundaries
- optional external fetch integrations

---

## Summary

Iteration 15 does not make the engine “browse the web during retrieval”.
It makes the engine better at **turning remote documentation into local knowledge**.

That is the right level of scope:
- easier knowledge ingestion
- preserved local-first reasoning
- reproducible artifacts
- minimal disruption to the current architecture