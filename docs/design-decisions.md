# Design decisions — workflow versus autonomous agents (card 04)

Depends on: card 01 (`docs/graph-walkthrough.md` exists, Status DONE 2026-09-16).
Source of truth: `crm/graph.py` `build_graph` (L350–449); node/router defs cited below.
No code, nodes, packages, or features were added by this record.

## 1. Classification — every node and router in exactly one bucket

Authorship rule: the model interprets data inside nodes; only plain-Python
routers choose the next step. The model never returns a branch name.

| Bucket | Member | Code ref | Why this bucket |
| --- | --- | --- | --- |
| Interpretation / probabilistic | `extract_card` | `crm/graph.py:L79–101`, protocol `CardExtractor` (`crm/providers/base.py:L12`), Groq impl (`crm/providers/groq.py`) | Image → dict via vision model; output varies, wrapped as `ExtractorError` on failure |
| Interpretation / probabilistic | `transcribe_voice` | `crm/graph.py:L142–162`, protocol `VoiceTranscriber` (`crm/providers/base.py:L16`), Groq Whisper impl | Audio → transcript; model output, never branching or writing |
| Interpretation / probabilistic | `create_embedding` | `crm/graph.py:L307–324`, protocol `EmbeddingProvider` (`crm/providers/base.py:L24`) | Search text → vector via provider; probabilistic/remote, wrapped as `EmbedderError` |
| Deterministic transformation | `load_input` | `crm/graph.py:L37–56` | Resets state, sets `status="loaded"`; no model, no I/O |
| Deterministic transformation | `validate_input` | `crm/graph.py:L59–76` | File-existence checks → `valid`/`invalid`; pure Python |
| Deterministic transformation | `validate_extraction` | `crm/graph.py:L104–129` | Pydantic `ContactEvidence` shape check → `valid`/`invalid`; validates shape, not card truth |
| Deterministic transformation | `merge_context` | `crm/graph.py:L165–178` | Concatenates `typed_notes` + `voice_transcript` → `conversation_notes`; string logic only |
| Deterministic transformation | `normalize_contact` | `crm/graph.py:L187–190` via `crm/normalize.py` | Lower/strip normalization; deterministic |
| Deterministic transformation | `build_search_document` | `crm/graph.py:L300–304` via `crm/search.py:build_search_document` | Joins full name, company, job title, notes; deterministic |
| Deterministic transformation | `finalize` | `crm/graph.py:L339–347` | `valid` + evidence + `contact_id` + `verified_contact` → `complete`, else unchanged; no I/O |
| Side effect | `search_crm` | `crm/graph.py:L193–201` | Read: `ContactStore.find_match`; never writes |
| Side effect | `create_contact` | `crm/graph.py:L221–228` | Write: `store.create`; sets `contact_id`, `crm_action="created"` |
| Side effect | `update_contact` | `crm/graph.py:L231–242` | Write: `store.update`; sets `contact_id`, `crm_action="updated"` |
| Side effect | `verify_write` | `crm/graph.py:L252–291` | Read-back: `store.get` + field/notes comparison → `error` on mismatch; never trusts INSERT/UPDATE return alone |
| Side effect | `store_embedding` | `crm/graph.py:L327–336` | Write: `store.upsert_embedding`; guarded no-op on `invalid`/`error` |
| Routing / deterministic | `voice_present` | `crm/graph.py:L132–139`, wired L408–415 | Plain Python on `status` + `voice_path`; returns `transcribe_voice` or `merge_context` |
| Routing / deterministic | `persistable` | `crm/graph.py:L181–184`, wired L417–424 | `invalid`/`error` → `finalize`, else `normalize_contact` |
| Routing / deterministic | `match_found` | `crm/graph.py:L204–211`, wired L426–434 | `matched_contact_id` → `update_contact`, else `create_contact` (or `finalize` on `invalid`/`error`) |
| Routing / deterministic | `write_ok` | `crm/graph.py:L294–297`, wired L437–444 | `valid` + `verified_contact` → `write_ok` (`build_search_document`), else `write_failed` (`finalize`) |

Count: 15 nodes + 4 routers, each in exactly one row above. CREATE-versus-UPDATE
is decided by `match_found` (deterministic match order: email, then phone, then
name+company in `crm/db.py`); the model never writes SQL and never branches.

## 2. Termination paths

`build_graph` (`crm/graph.py:L388–449`) is a DAG with no cycles: every path ends
at `finalize → END` (L448). Bounded steps: at most 15 node visits per run
(13 on the longest success path, fewer on short-circuits).

- Success: `... → verify_write → write_ok → build_search_document → create_embedding → store_embedding → finalize → END`.
- Invalid/error short-circuit after context: `merge_context → persistable → finalize` (L417–424), skipping `normalize_contact`/`search_crm`/writes.
- Invalid/error at match: `search_crm → match_found → finalize` (L426–434).
- Failed verification: `verify_write → write_ok → write_failed → finalize` (L437–444), so embedding nodes never run.
- Guarded nodes (present in trace, do no work) versus skipped edges (absent from
  trace) are defined in `docs/graph-walkthrough.md` ("Guarded node vs skipped edge").

## 3. Actual trace — walkthrough (a), card-only success

From `docs/graph-walkthrough.md` trace (a), cross-checked against
`crm/graph.py:build_graph` (L350–449) and `tests/test_graph.py`:

`load_input` → `validate_input` → `extract_card` → `validate_extraction` →
`merge_context` → `normalize_contact` → `search_crm` → `create_contact` →
`verify_write` → `build_search_document` → `create_embedding` →
`store_embedding` → `finalize` (13 nodes).

Statuses: `pending` → `loaded` (`load_input`) → `valid` (`validate_input`) →
`extracted` (`extract_card`) → `valid` (`validate_extraction` through
`store_embedding`) → `complete` (`finalize`, L339–347).
Provider calls: extractor 1, transcriber 0 (no voice — `voice_present` →
`merge_context`), embedder 1. Store calls: `find_match` → `create` → `get`
(verify) → `upsert_embedding`. Writes: 1 contact row, vector present;
`contact_id=1`, `verified_contact` set.

## 4. Hypothetical model-planned alternative (NOT implemented)

Multi-source enrichment with unknown step order: after capture, the model is
given tools (re-read card image when unsure, web/company lookup, fuzzy matcher
choice, vector re-index) and decides per run which tools to call, in what order,
and when to stop — e.g. "card is blurry, re-read with a different prompt, then
pick the matcher, then decide whether one more lookup is worth it."

Extra testing/control burden this would add (reason it is rejected here):

- Non-deterministic paths: same input can traverse different tool sequences, so
  trace (a)-style node-order assertions no longer hold; every test needs
  path-tolerance or per-trajectory oracles.
- Unbounded steps: the stop condition is a model judgment, requiring a
  step budget, timeout, and cost ceiling plus tests proving they always fire.
- Per-trajectory write-safety proofs: because the model chooses match/write
  order, each trajectory must prove no duplicate, overwrite, or unindexed write;
  today's single `verify_write` read-back plus fixed create/update order would
  not cover model-chosen sequences.
- Replay/audit difficulty: reproducing a decision requires the full prompt,
  model revision, tool outputs, and stop rationale — not just the code revision
  and fakes used today.

## 5. Verdict and out-of-scope planning case

Verdict: the predefined workflow is correct for capture. Card → deterministic
validation → optional transcription → deterministic match → guarded write →
read-back verification → indexing covers the task with bounded steps and a
single write path; no planning agent is needed.

Explicitly out of scope: the multi-source enrichment case above (uncertain card
+ external lookups + model-chosen matcher/stop) is the one scenario where
dynamic planning might be justified — open-ended evidence gathering with no
fixed order. It is kept out of scope: no lookup tools, no planner node, no
ordering model. Do not implement it under this card.

## 6. Learning question

Does using LangGraph automatically make an application autonomous? No.
LangGraph only runs the graph the author wrote (`add_node` / `add_edge` /
`add_conditional_edges` in `build_graph`, L388–449). Autonomy comes from who
authors the transitions: here four plain-Python routers (`voice_present`,
`persistable`, `match_found`, `write_ok`) choose every branch and the model
never selects a tool or next step, so this is an intentional bounded workflow
(see `docs/architecture.md` "Workflow understanding and deployment boundary,"
L80–84) — not an autonomous agent.
