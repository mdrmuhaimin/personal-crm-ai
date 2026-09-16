# What You Should Understand So Far

Last updated: 2026-09-15

This is the learning notebook for the AI Conference CRM. It is the accumulated **What You Should Understand Now** after each completed task.

What was built (files, tests, CLI): [progress_so_far.md](progress_so_far.md).  
How to run: [how_to_run.md](how_to_run.md).

After every later task that passes verification, the orchestrator must update **this file** and `progress_so_far.md` before stopping.

---

## The split that runs through everything

| Kind | Examples | Who does it |
| --- | --- | --- |
| Probabilistic interpretation | Read a card image, transcribe voice | Groq, behind a small interface |
| Deterministic | Validate paths, normalize, match, CREATE/UPDATE, verify the row, create default token-hash vectors, KNN lookup, load rows by id | Python + SQLite + sqlite-vec |

Do not ask a model to decide something ordinary code can decide.

Current clarification: the default embedding implementation is token hashing, not a learned semantic model. Learned encoders can also produce deterministic vectors at inference; the useful distinction is learned representation versus lexical hashing. Older task entries below preserve their historical terminology. See [the architecture guide](docs/architecture.md) for the current evidence limits.

---

## Task 1 — LangGraph is state + nodes + edges

LangGraph is not “an agent.” It is a **state object**, named functions, and explicit edges.

If you want to know what the system knows, open `crm/state.py`, not a prompt.

Validation is ordinary Python. An image is required. A typed name is required only when there is no image. With a card image, blank name is fine: `validate_extraction` copies `full_name` from the card. The run still reaches END so you can inspect the result.

**Check:** Why can `finalize` stay on the linear path when input is invalid?

---

## Task 2 — The model does not own the result

The vision model returns a dict. `ContactEvidence.model_validate` decides if that dict is usable.

The graph talks only to `CardExtractor.extract_card(path)`. Groq lives in the provider. Tests inject `FakeExtractor`.

Failure is a state: missing env or a dead API → `error`. Bad JSON → `invalid`.

**Check:** If `validate_input` already set `invalid`, what actually prevents the API call?

---

## Task 3 — Voice is optional context, not identity

A missing `--voice` is a **route**, not an error. `voice_present` skips `transcribe_voice`. Whisper is not called. Notes stay `None`.

Both paths meet at `merge_context`.

```text
business card → ContactEvidence (who)
voice note    → conversation_notes (what we discussed)
```

Voice never overwrites company, email, or other card fields.

**Check:** After `validate_extraction` with no `--voice`, which nodes run, and is Whisper called?

---

## Task 4 — Persistence, keys, and notes

A successful capture is a **row** in SQLite (`data/crm.db`), not only JSON.

Normalize before search: email lowercased, phone as digits, name/company stripped and casefolded.

Match in this order, first hit wins:

1. normalized email
2. else normalized phone
3. else normalized name **and** company

Name alone is not a key. Two Sarahs with `sarah@nexatech.com` and `sarah@other.com` are two people. Same name, different company, no email/phone → two rows.

CREATE vs UPDATE is an explicit graph fork (`match_found`). Failed UPDATE does not CREATE.

Notes append (`old` + blank line + `new`). No new notes → leave old notes. A new `null` field does not erase a stored value.

The LLM does not detect duplicates.

**Check:** Why match on email (or name+company) instead of “they sound like the same person”?

---

## Write verification — proof vs hope

`create()` returning an id is a claim. `store.get(id)` is evidence.

`verify_write` re-reads the row and compares intended non-empty fields. Missing row, wrong values, or a get exception → `error`.

`finalize` sets `complete` only when `contact_id` **and** `verified_contact` are present.

The read goes through `ContactStore` (SQLite today). The graph does not need to know the engine.

**Check:** If create returns `contact_id=7` but `get(7)` is missing, what is `status`? Is the CLI allowed to print `complete`?

---

## Task 6 — The vector is an index, not the record

An embedding is a list of numbers that represents the *meaning* of a short document. Nearby vectors mean similar meaning. The CRM still stores the person in `contacts`. `contact_embeddings` only answers: “which `contact_id`s are closest to this question?”

The searchable document is not the whole row. It is `full_name`, `company`, `job_title`, and `notes`. Email and phone help *find the same person later*. They do not help *“who talked about automation?”*

This is a different job from duplicate matching:

| Job | Tool | Example |
| --- | --- | --- |
| Same person? | Deterministic keys | same email → UPDATE |
| Related conversation? | Vector similarity | “workflow automation” → Sarah’s notes |

The embed path runs only after `write_ok`. A failed verify must not write a vector for a row you do not trust. When notes change, DELETE + INSERT replaces that `contact_id`’s vector so search does not keep the old meaning.

The graph calls `EmbeddingProvider.embed(text)`. Groq HTTP stays in the provider. Tests inject `FakeEmbedder`. This Groq account lists no embedding models, so the live default is a local 768-d token hash. Set `CRM_EMBED_MODEL` only if a host actually offers one.

`crm query` is not a capture-graph node. It embeds the question, asks sqlite-vec for IDs, then `ContactStore.get`.

**Check:** After a notes update, why must the old vector be replaced — and why is that not the same as changing the `contacts` row?

---

## Task 7 — A trace is not a score

Tracing answers “what ran?” Evaluation answers “was it right?”

LangSmith traces (when `LANGSMITH_API_KEY` is set) wrap live Groq extract / transcribe / embed under the graph invoke. They do not add CRM nodes.

The experiment is a **labeled dataset** plus **code evaluators**. Each example has expected `contact_evidence`, `crm_action`, `null_fields`, and `verified`. The target is the real capture graph. Fakes stand in for Groq so default eval needs no key.

All five scores being 1.0 is honest, not success theater: the fake extractor returns the labeled card. The graph’s job is matching, CREATE vs UPDATE, verify, and “voice does not overwrite identity.” That path is deterministic.

The hardest labeled case is still `conflicting-voice`. The voice says she works at Google. The card says Analytical Engines. A live model (or a sloppy merge) could copy the voice onto the card. Our graph must keep card identity and put the voice in notes.

**Check:** If traces appear in LangSmith but `unsupported-fields` invents an email, did evaluation pass? Which evaluator should fail?

---

## Task 8 — Two inputs, one merge; two jobs, two paths

Typed notes and a voice transcript are two sources of the same kind of thing: conversation context. They enter state separately (`typed_notes`, `voice_transcript`). `merge_context` joins them with a blank line. No model rewrites that.

Absence of `--notes` or `--voice` is normal. Text-only never calls Whisper. Voice-only never needs typed text.

Search is a different path:

```text
question → query embedding → sqlite-vec IDs → contacts row → print
```

The question gets a new embedding at query time. That is not the contact’s stored vector. The stored vector was built from the search document after a verified write. Results are CRM rows, not vec-table metadata.

**Check:** You type `--notes` and also pass `--voice`. Which node combines them, and does `crm query` run that node?

---

## Discord — another door, same rooms

The bot does not sit on the graph. A DM is translated into the same payload the CLI builds, or into `query_contacts`.

Pending intake is a dict keyed by Discord user id. The card waits. Voice or `save` is when `build_graph()` runs. Typed DM text is notes, not the contact name. `/query` never does that.

Guild messages are ignored. DMs only.

**Check:** You send a card image in a Discord DM, then `/query who did I meet?` before `save`. Did a contact get written?

---

## Current graph (all tasks)

```text
START
  ↓
load_input (name, image, optional voice, optional typed_notes)
  → validate_input → extract_card → validate_extraction
  ↓
voice_present?
   /        \
 no          yes → transcribe_voice
  \          /
   merge_context   ← typed and/or voice, no LLM
        ↓
   persistable?
      /      \
 finalize  normalize_contact → search_crm → match_found?
                                    /              \
                          update_contact      create_contact
                                    \              /
                                     verify_write
                                          ↓
                                      write_ok?
                                       /        \
                                 fail            ok
                                  |               |
                                  |    build_search_document
                                  |         ↓
                                  |    create_embedding
                                  |         ↓
                                  |    store_embedding
                                   \        /
                                    finalize → END
```

---

## Tooling (how we work, not what the CRM is)

- **Ponytail:** smallest **code**. Does not skip teaching.
- **Graphify-Labs Graphify** (https://github.com/Graphify-Labs/graphify): query `graphify-out/` before exploring files, then teach from what you found.

Required reports after every specified task: Learning Step → Implementation Update → Evaluation → What You Should Understand Now. Then update this file and `progress_so_far.md`.

---

## Portfolio documentation and roadmap — 2026-09-14

### What You Should Understand Now

1. **Tests and model evaluations answer different questions.** Fake providers let us check routing, writes, and failures reliably. Real labeled cards and queries are needed to measure extraction accuracy and retrieval usefulness. Perfect fake scores do not prove either.
2. **Verification has a boundary.** Pydantic checks structure, and `verify_write` checks stored values against intended extraction output. Neither proves the model read the card correctly. A contact can also be saved before indexing fails.
3. **Embedding choice includes evidence and compatibility.** Token hashes mainly capture shared words. A learned model should be compared on labeled queries, latency, and resource use. Capture and query must use compatible model versions and document formats; changing those can require an index rebuild.
4. **A controlled workflow demonstrates engineering depth.** Explicit state, nodes, conditional edges, and failure paths are the right tools for this capture process. Retrieval currently returns stored records; generated RAG answers would need separate relevance, support, citation, and abstention evaluations.

**Exercise:** A run saves the correct extracted row, then embedding creation fails. Explain what `verify_write` proved, what remains unknown about the card, and why replaying capture could duplicate notes.

Use [the development roadmap](docs/development-roadmap.md) to select one small task at a time. No future implementation has started.

## Development roadmap engineering handoff — 2026-09-15

### What You Should Understand Now

1. A development roadmap starts with working capabilities and their evidence. It preserves sound decisions while separating missing functionality from missing quality measurements.
2. An implementable engineering plan names the data contracts, component responsibilities, dependencies, failure behavior, and evidence required for completion. The coding agent should not have to invent those boundaries.
3. Recovery is part of the design: replacing an index must preserve usable data after interruption, and replaying a committed request must not be mistaken for proof of completed verification or indexing.

**Exercise:** Open card 14 in [the development roadmap](docs/development-roadmap.md). Explain which index remains active if rebuilding stops before the active-pointer transaction commits, and why contact notes should remain unchanged.

## Card 01 — Trace success and failures — 2026-09-16

### What You Should Understand Now

1. A guarded node still runs but does nothing when status is wrong; a skipped edge never runs because a router chose another path. `graph.stream` shows the difference: present-but-no-op vs absent.
2. Side effects follow commit order, not final status. `create_contact` commits before `verify_write` re-reads and before any embedding, so `error` can still keep a saved `contact_id` with no vector.
3. Failed verification must never embed. `write_ok` routes `write_failed` straight to `finalize`, so `build_search_document`/`create_embedding`/`store_embedding` are absent and `embedder.calls` stays empty.

**Exercise:** In `docs/graph-walkthrough.md` trace (d), explain why `store_embedding` appears in the 13-node list on embedder failure even though it writes nothing, citing the guard line.

## Card 02 — Reproducible baseline — 2026-09-16

### What You Should Understand Now

1. Unbounded deps (`langgraph`, `pydantic` with no pins) can resolve differently tomorrow. A full `pip freeze` lock records exact resolved versions so fresh envs install identical packages.
2. Lock + editable install are split: lock carries deps, `pip install -e . --no-deps` carries only the package, avoiding double resolution.
3. Native extensions are part of reproducibility. `sqlite-vec`+`apsw` must install or embedding tests must fail loudly — silent skip would hide a broken baseline.

**Exercise:** Why is `102 passed` in your dev `.venv` insufficient to claim reproducibility, and what does the fresh `/tmp` venv proof add?

## Card 03 — Offline CI (local done, hosted pending) — 2026-09-16

### What You Should Understand Now

1. CI repeats the lock-install + offline suite on a machine nobody touched. Read-only perms + no secrets + tracing flags off keep it offline and unable to leak or upload contacts.
2. Green CI proves workflow regression only — `102 passed` with fakes says routing/writes/failures behave, nothing about real card reading or semantic search. That is card 03's learning question answered in advance.
3. A workflow file is a claim until a hosted run exists. Local equivalence (fresh `/tmp` venv, same commands) is necessary but not sufficient; the run URL is the evidence.

**Exercise:** Open `.github/workflows/tests.yml` and point to the three lines that keep live providers out of CI (hint: install source, test flags, env).
