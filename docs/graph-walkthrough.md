# Graph walkthrough (card 01)

Depends on: nothing. Goal: explain graph control flow, guards, and side effects from code.

Method: `graph.stream` node order (as in `tests/test_graph.py:test_stream_node_order_valid_and_invalid`),
recording fakes (`FakeExtractor`/`FakeTranscriber`/`FakeEmbedder` in `tests/helpers.py`)
plus a `RecordingStore` wrapper (temporary, not committed) counting
`find_match`/`create`/`update`/`get`/`upsert_embedding`.
Existing focused tests already cover all four paths, so no new committed tests were needed.

Focused evidence run: `.venv/bin/python -m pytest -q tests/test_graph.py tests/test_verify.py tests/test_embeddings.py` → 25 passed.

## Overview

| Scenario | Visited nodes (in order) | Provider calls | Store calls | Committed writes |
| --- | --- | --- | --- | --- |
| (a) card-only success | `load_input` → `validate_input` → `extract_card` → `validate_extraction` → `merge_context` → `normalize_contact` → `search_crm` → `create_contact` → `verify_write` → `build_search_document` → `create_embedding` → `store_embedding` → `finalize` | extractor 1, transcriber 0, embedder 1 | `find_match`, `create`, `get`, `upsert_embedding` | contact row yes, vector yes |
| (b) invalid image | `load_input` → `validate_input` → `extract_card` → `validate_extraction` → `merge_context` → `finalize` | extractor 0, transcriber 0, embedder 0 | none | contact row no, vector no |
| (c) failed verification | `load_input` → `validate_input` → `extract_card` → `validate_extraction` → `merge_context` → `normalize_contact` → `search_crm` → `create_contact` → `verify_write` → `finalize` | extractor 1, transcriber 0, embedder 0 | `find_match`, `create`, `get` | contact row yes, vector no |
| (d) embedding failure | `load_input` → `validate_input` → `extract_card` → `validate_extraction` → `merge_context` → `normalize_contact` → `search_crm` → `create_contact` → `verify_write` → `build_search_document` → `create_embedding` → `store_embedding` → `finalize` | extractor 1, transcriber 0, embedder 1 (raises) | `find_match`, `create`, `get` (no `upsert_embedding`) | contact row yes, vector no |

Vector-store failure variant of (d) visits the same 13 nodes, but `create_embedding` succeeds and
`store_embedding` raises: store calls `find_match`, `create`, `get`, `upsert_embedding`; contact row yes, vector no.

## Traces

### (a) Card-only success

- Nodes: 13 listed above. Observed via `graph.stream`, matching
  `tests/test_graph.py:test_stream_node_order_valid_and_invalid` (`expected_valid`) and
  `tests/test_verify.py:test_create_stream_includes_verify_write`.
- Status: `pending` → `loaded` (`load_input`, `crm/graph.py:L37`) → `valid` (`validate_input`)
  → `extracted` (`extract_card`) → `valid` (`validate_extraction` through `store_embedding`)
  → `complete` (`finalize`, `crm/graph.py:L339-L347`).
- Providers: `FakeExtractor.calls` 1 image path; `FakeTranscriber.calls` 0 (no voice);
  `FakeEmbedder.calls` 1 search document (`Sarah Khan NexaTech Solutions Product Manager`).
- Store: `find_match` → `create` → `get` (verify) → `upsert_embedding`.
- Writes: 1 contact row, vector present. Final `status=complete`, `contact_id=1`, `verified_contact` set.

### (b) Invalid image

- Nodes: 6 listed above, matching `expected_invalid` in `tests/test_graph.py:test_stream_node_order_valid_and_invalid`
  and `tests/test_crm.py:test_invalid_or_error_does_not_write_db`.
- Status: `pending` → `loaded` → `invalid` (`validate_input`, `crm/graph.py:L59-L76`)
  → stays `invalid` through `extract_card`, `validate_extraction`, `merge_context`, `finalize`.
- Providers: 0/0/0. `extract_card` guard (`crm/graph.py:L79-L81`) returns state unchanged when
  `status != valid`, so the extractor is never called (`fake.calls == []` in existing tests).
- Store: none. `persistable` router (`crm/graph.py:L181-L184`, `crm/graph.py:L417-L424`)
  sends `merge_context` → `finalize`, skipping `normalize_contact`/`search_crm`/writes.
- Writes: 0 rows, no vector. `contact_id=None`.

### (c) Failed verification

- Nodes: 10 listed above. Reproduce with `store.get = lambda _cid: None`
  (see `tests/test_verify.py:test_missing_record_fails`,
  `tests/test_embeddings.py:test_failed_verify_skips_embedding`).
- Status: `pending` → `loaded` → `valid` → `extracted` → `valid` up to `create_contact`
  → `error` at `verify_write` (`verification failed: contact not found`, `crm/graph.py:L252-L291`)
  → `error` at `finalize` (guard in `crm/graph.py:L339-L347` only completes fully verified writes).
- Providers: extractor 1, transcriber 0, embedder 0. `write_ok` router
  (`crm/graph.py:L294-L297`, `crm/graph.py:L437-L444`) takes `write_failed` → `finalize`,
  so `build_search_document`/`create_embedding`/`store_embedding` are never visited.
  This is the proof failed verification never embeds: `embedder.calls == []` and
  `raw.get_embedding(contact_id) is None`.
- Store: `find_match`, `create`, `get`. No `upsert_embedding`.
- Writes: contact row yes (the `create` committed before the re-read failed), vector no.
  Result keeps `contact_id=1` with `verified_contact=None`, `status=error`.

### (d) Embedding failure

- Nodes: same 13 as (a). Reproduce with `FakeEmbedder(error=EmbedderError("embed down"))`
  (see `tests/test_embeddings.py:test_embed_error_is_not_complete`) and
  `store.upsert_embedding = boom` (see `tests/test_embeddings.py:test_store_embedding_error_is_not_complete`).
- Status (embedder error): `valid` through `verify_write` and `build_search_document`
  → `error` at `create_embedding` (`crm/graph.py:L307-L324`)
  → stays `error` through `store_embedding` and `finalize`.
- Status (vector-store error): `valid` through `create_embedding`
  → `error` at `store_embedding` (`crm/graph.py:L327-L336`) → `error` at `finalize`.
- Providers: extractor 1, transcriber 0, embedder 1 (called with the search document; raises in case 1).
- Store (embedder error): `find_match`, `create`, `get`; `upsert_embedding` never called because
  `store_embedding` guard returns unchanged on `error`.
  Store (vector-store error): `find_match`, `create`, `get`, `upsert_embedding` (called, raises).
- Writes: contact row yes, vector no in both variants. Result keeps `contact_id` and
  `verified_contact`, `status=error`.

## Guarded node vs skipped edge

- Executed guarded node = present in `graph.stream` output but returns state unchanged because of a
  status guard. Example (b): `extract_card` and `validate_extraction` appear in the 6-node trace,
  but do no work: `extract_card` early-returns when `status != valid` (`crm/graph.py:L79-L81`);
  `validate_extraction` early-returns on `invalid`/`error` (`crm/graph.py:L104-L107`).
  Example (d embedder error): `store_embedding` appears in the 13-node trace but skips its write
  via `if status in ("invalid", "error"): return state` (`crm/graph.py:L327-L329`).
- Skipped edge = absent from `graph.stream` output because a conditional edge was not taken.
  Example (b): `transcribe_voice`, `normalize_contact`, `search_crm`, `create_contact`,
  `update_contact`, `verify_write`, `build_search_document`, `create_embedding`, `store_embedding`
  (persistence/indexing branch) never run: `voice_present` → `merge_context`
  (`crm/graph.py:L132-L139`, `crm/graph.py:L408-L415`) and `persistable` → `finalize`
  (`crm/graph.py:L181-L184`, `crm/graph.py:L417-L424`).
  Example (c): `build_search_document`, `create_embedding`, `store_embedding` never run:
  `write_ok` → `write_failed` → `finalize` (`crm/graph.py:L294-L297`, `crm/graph.py:L437-L444`).
  `match_found` → `finalize` on `invalid`/`error` (`crm/graph.py:L204-L211`, `crm/graph.py:L426-L434`)
  is the same skipped-edge mechanism for the create/update branch.

## README reconciliation

`README.md` workflow diagram and `crm/graph.py:build_graph` (`crm/graph.py:L350-L449`) agree.
Short diagram labels map to code routers: `VP` = `voice_present`, `P` = `persistable`,
`Q` = `match_found`, `O` = `write_ok`. Edge targets use the same node names; code return strings
(`transcribe_voice`/`merge_context`, `normalize_contact`/`finalize`, `update_contact`/`create_contact`/`finalize`,
`write_ok`/`write_failed`) are the diagram's branch labels. The note
"Invalid input still traverses guarded extraction nodes without calling providers.
Embedding failure still traverses `store_embedding`, which skips its write."
matches traces (b) and (d) above. Querying (`query → embed → nearest vector rows → read contacts`)
is correctly shown as a separate path outside the capture graph.

## Learning question: why can an error result still have a saved contact ID?

Because contact writes commit before indexing, and verification/indexing failures do not roll back
the row. In (d), `create_contact` commits, `verify_write` succeeds (`verified_contact` set),
then `create_embedding`/`store_embedding` fails — the error result keeps the verified `contact_id`
with a missing/stale vector; recovery is reindex, not recapture
(see `crm/graph.py:L221-L242`, `crm/graph.py:L252-L291`, `crm/graph.py:L307-L336`,
`docs/architecture.md` failure semantics). In (c), `create` commits before the re-read fails,
so the row exists even though `status=error` and `verified_contact=None`.
