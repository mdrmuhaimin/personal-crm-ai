# Progress So Far — AI Conference CRM

Last updated: 2026-09-15

This document records completed work. Current strengths and proposed tasks are in [the development roadmap](docs/development-roadmap.md); no next implementation task has been selected.

Lessons (what to understand): [understandable_so_far.md](understandable_so_far.md).  
How to run the current app: see [how_to_run.md](how_to_run.md).

After every later task that passes verification, update **this file** and `understandable_so_far.md` before stopping.

**Agent tooling in this repo:**

- **Ponytail (full)** — `.cursor/rules/ponytail.mdc`. Smallest working **code**. Learning reports still follow [AGENTS.md](AGENTS.md).
- **Graphify-Labs Graphify** — https://github.com/Graphify-Labs/graphify only. Rule: `.cursor/rules/graphify.mdc`. Query `graphify-out/` first, then teach from what you found using the AGENTS.md formats.

---

## What this project is

A personal conference CRM with capture and vector retrieval. For capture, the user provides:

- optionally, a person's name (the card must contain an extractable full name)
- a business-card image
- an optional voice-file path
- optional typed conversation notes

The system is built as an explicit LangGraph. Deterministic Python handles validation, matching, persistence, and write verification. Groq reads card images and transcribes optional voice notes. After a verified write, the default embedder creates local token-hash vectors; learned semantic quality has not been demonstrated. Contacts live in SQLite (`data/crm.db`); `sqlite-vec` stores vectors keyed by contact ID separately from canonical contact rows. Retrieval returns stored contacts, not generated RAG answers.

No PostgreSQL or Telegram yet.

The full current graph (Tasks 1–8) is in [understandable_so_far.md](understandable_so_far.md). Observability wraps that graph; it does not add nodes. Search is a separate CLI path.

---

## Task 1 — LangGraph Skeleton

**Status:** Done. Independent verifier PASS.

**What we built:** The smallest runnable graph and CLI. No LLM.

**Graph at the end of Task 1:**

```text
START → load_input → validate_input → finalize → END
```

**CLI:**

```bash
python -m crm --name "Ada Lovelace" --image /path/to/card.jpg
python -m crm --name "Ada Lovelace" --image /path/to/card.jpg --voice /path/to/note.wav
```

**State (`crm/state.py`):** raw inputs plus `status` and `errors`.

**Validation (ordinary Python):**

- Name is required (blank or whitespace fails).
- Image path is required and must be an existing file.
- Voice path is optional; if provided, it must be an existing file.
- Failures set `status="invalid"` and still reach END. They do not crash.

**What Task 1 did not do:** read the card, call an API, or store a contact.

**Key files:** `crm/state.py`, `crm/graph.py`, `crm/cli.py`, `tests/test_graph.py`, `tests/test_cli.py`

---

## Task 2 — Business Card Extraction

**Status:** Done. Independent verifier PASS.

**What we built:** Two new graph nodes. A Pydantic schema. An isolated AI provider. Fake-extractor unit tests plus one optional live smoke test.

**Graph now:**

```text
START → load_input → validate_input → extract_card → validate_extraction → finalize → END
```

Linear edges only. If input validation already failed, `extract_card` returns the state unchanged and does **not** call Groq.

**Schema (`crm/schemas.py` — `ContactEvidence`):**

- `full_name` (required)
- `company`, `job_title`, `email`, `phone`, `website`, `address` (each may be null)

The model is told to return `null` when a field is not visible on the card. Empty optional strings are normalized to `null`. Invalid model JSON fails schema validation (`status="invalid"`). API/provider failure sets `status="error"`.

**Provider isolation:**

```text
LangGraph extract_card  →  CardExtractor.extract_card(path)  →  Groq
                                    ↑
                           tests: FakeExtractor
```

The graph does not contain Groq HTTP details. Tests inject `FakeExtractor`, so default `pytest` never needs a network or API key.

**Live provider:** official Groq Python client (`from groq import Groq`), following Groq vision docs:

- local image encoded as a base64 `data:` URL
- model `qwen/qwen3.6-27b`
- `response_format={"type": "json_object"}`

**API key:** `GROQ_API_KEY` in a local `.env` file (gitignored) or in the system environment. The app loads `.env` with `python-dotenv`. The key is not committed or hardcoded.

**CLI output** now includes `contact_evidence`. Exit `0` on `complete`, `1` on `invalid` or `error`.

**Key files:** `crm/schemas.py`, `crm/providers/base.py`, `crm/providers/groq.py`, `crm/graph.py`, `tests/test_extract.py`, `tests/test_live_extract.py`

---

## Task 3 — Optional Voice Note Branch and Transcription

**Status:** Done. Independent verifier PASS (`pytest -q` → 30 passed, 2 deselected).

**What we built:** An explicit LangGraph fork after card extraction. Voice is optional conversation context, not identity. Both paths meet at `merge_context`.

**Graph after Task 3** (persist and verify came later):

```text
START → load_input → validate_input → extract_card → validate_extraction
  → voice_present? → (transcribe_voice or skip) → merge_context → finalize
```

The fork is a real `add_conditional_edges` call in `crm/graph.py`, not a hidden `if` inside one node.

**Optional behaviour:**

- Name + image only: skip `transcribe_voice`, leave voice fields `None`, still `complete`. Groq Whisper is not called.
- Name + image + existing `.ogg`: run `transcribe_voice`, store the text, still `complete`.
- Voice supplied but STT fails: `status="error"` (not silently ignored).

**New state fields (`crm/state.py`):**

- `voice_transcript` — raw Whisper text, or `None`
- `conversation_notes` — same text attached as context in `merge_context`, or `None`

**Provider isolation:**

```text
LangGraph transcribe_voice  →  VoiceTranscriber.transcribe(path)  →  Groq Whisper
                                         ↑
                                tests: FakeTranscriber
```

Live impl: official Groq client, `GROQ_API_KEY`, model `whisper-large-v3`. Original `.ogg` (Opus) is passed through. No FFmpeg. Groq accepts `ogg` directly.

**Boundary:**

```text
business card → ContactEvidence (identity)
voice note    → conversation_notes (context)
```

Voice never overwrites `company`, email, or other card fields.

**CLI:**

```bash
python -m crm --name "Sarah Khan" --image input/visiting_card.png
python -m crm --name "Sarah Khan" --image input/visiting_card.png --voice input/6134386456120009929.ogg
```

Telegram is not part of this task. The CLI still takes a local voice-file path.

**Key files:** `crm/graph.py`, `crm/state.py`, `crm/providers/base.py`, `crm/providers/groq.py`, `tests/test_voice.py`, `tests/test_live_transcribe.py`

---

## Live checks

On branch `ft/Core_Engine`, with `GROQ_API_KEY` in `.env`.

**Card only:**

```bash
python -m crm --name "Sarah Khan" --image input/visiting_card.png
```

Result: `status="complete"`. `contact_evidence` included Sarah Khan, NexaTech Solutions, Product Manager, email, phone, website, and address. `voice_transcript` and `conversation_notes` were `None`.

**Card + sample voice** (`input/6134386456120009929.ogg`):

```bash
python -m crm \
  --name "Sarah Khan" \
  --image input/visiting_card.png \
  --voice input/6134386456120009929.ogg
```

Result: `status="complete"`. Card fields unchanged. Voice context:

> So I met this person in an event. She is a very good contact for our CRM project and would like to follow up with her after two weeks.

---

## Tests

Default (no live API):

```bash
pytest -q
```

Last recorded default run after local-default embeddings: **102 passed, 2 deselected**.

Optional live smoke tests (need `GROQ_API_KEY`; card image and/or `input/6134386456120009929.ogg`):

```bash
pytest -q -m live --override-ini addopts=
```

---

## Task 4 — SQLite CRM Persistence, Matching, and Notes

**Status:** Done. Independent verifier PASS (`pytest -q` → 46 passed, 2 deselected).

**What we built:** After merge, normalize and search SQLite. Explicit CREATE vs UPDATE. Notes append. LLM does not match people.

**Graph after Task 4** (verify_write came next):

```text
merge_context
      ↓
persistable?
   /         \
finalize   normalize_contact
                  ↓
             search_crm
                  ↓
            match_found?
             /         \
     update_contact  create_contact
             \         /
              finalize → END
```

**Store:** `crm/db.py` (`sqlite3`, default `data/crm.db`). Tests use a temp file.

**Match order:** normalized email, then phone, then full_name + company.

**Notes:** CREATE may have `NULL`. UPDATE appends with a blank line. No new notes → leave old notes. Null fields do not erase stored values.

**Key files:** `crm/db.py`, `crm/normalize.py`, `crm/graph.py`, `tests/test_crm.py`

---

## Write verification

**Status:** Done. Independent verifier PASS (`pytest -q` → 55 passed, 2 deselected).

**What we built:** After CREATE/UPDATE the graph re-reads the row. `complete` requires `verified_contact` from that read, not from the write return value.

```text
create_contact ↘
                verify_write → write_ok? → finalize → END
update_contact ↗
```

Read-back is `ContactStore.get` (SQLite). PostgreSQL was not added.

**Key files:** `crm/graph.py`, `tests/test_verify.py`

---

## Task 6 — Semantic Contact Retrieval with sqlite-vec

**Status:** Done. Independent verifier PASS (`pytest -q` → 65 passed, 2 deselected).

**What we built:** After a verified write, build a short search document, embed it, and store the vector in `sqlite-vec`. `crm query` finds contacts by meaning, then loads the SQLite rows.

**Graph after a successful write:**

```text
verify_write → write_ok?
   fail → finalize
   ok   → build_search_document → create_embedding → store_embedding → finalize
```

Failed verify skips the embed path. Embed or store failure keeps `status="error"` (not `complete`).

**Storage:** `contacts` table unchanged. Separate virtual table `contact_embeddings` (`vec0`). `rowid` = `contact_id`. Upsert is DELETE then INSERT so updates do not leave a stale vector.

**Search document:** non-blank `full_name`, `company`, `job_title`, `notes` only.

**Provider isolation:**

```text
LangGraph create_embedding  →  EmbeddingProvider.embed(text)  →  Groq
                                         ↑
                                tests: FakeEmbedder
```

Live default: local 768-d token-hash vector (this Groq account has no embedding models). Groq embeddings run only if `CRM_EMBED_MODEL` is set; a 404 falls back to the same local hash. Default tests inject `FakeEmbedder` (dim 8). No key, no network.

**CLI:**

```bash
python -m crm query "Who did I meet regarding AI workflow automation?"
```

Embeds the question, KNN on `contact_embeddings`, loads contacts by id. JSON list with `distance`. Default `--limit` 5.

This Mac’s CPython cannot load SQLite extensions, so `crm/db.py` uses a thin APSW wrapper so `sqlite_vec.load` still works.

**Key files:** `crm/search.py`, `crm/providers/embeddings.py`, `crm/db.py`, `crm/graph.py`, `crm/cli.py`, `tests/test_embeddings.py`

---

## Task 7 — LangSmith Observability and Evaluation

**Status:** Done. Independent verifier PASS (`pytest -q` → 83 passed, 2 deselected).

**What we built:** Optional LangSmith tracing on live model calls. An offline dataset of 10 capture cases. Five deterministic evaluators. `python -m crm.eval` runs the real graph (fakes + temp SQLite) through `langsmith.evaluate` and writes scores.

**Graph change:** none.

**Tracing:** `crm/tracing.py` `enable_tracing()`. If `LANGSMITH_API_KEY` is set: `LANGSMITH_TRACING=true`, project `ai-conference-crm`. No key → no-op. Live Groq `extract_card` / `transcribe` / `embed` are `@traceable`. CLI calls `enable_tracing()` after `load_dotenv`.

**Dataset** (`eval/dataset.json`): complete-card, partial-card, missing-phone, missing-email, existing-contact, new-contact, voice-present, voice-absent, conflicting-voice, unsupported-fields.

**Evaluators** (code, not LLM judges): extraction correctness; unsupported-field hallucination; CREATE vs UPDATE; duplicate avoidance; post-write verification.

**Experiment:** `eval/latest_experiment.json`. All means 1.0 with fakes. `experiment_url` is null (no LangSmith key). Weakest/hardest labeled case: `conflicting-voice`.

**CLI:**

```bash
python -m crm.eval
```

Default pytest does not upload and does not need a LangSmith key.

**Key files:** `crm/eval.py`, `crm/tracing.py`, `eval/dataset.json`, `eval/latest_experiment.json`, `tests/test_eval.py`

---

## Task 8 — Text Notes and Semantic Search Interface

**Status:** Done. Independent verifier PASS (`pytest -q` → 93 passed, 2 deselected).

**What we built:** Optional `--notes` on capture. Typed and voice context merge in ordinary Python. `crm query` prints a short ranked list from SQLite rows. Search does not run the capture graph.

**Graph:** same nodes. `load_input` now copies `typed_notes`. `merge_context` joins typed and/or voice (`"\n\n"` when both). No notes AI node.

**CLI:**

```bash
python -m crm --name "Sarah Khan" --image input/visiting_card.png --notes "Potential consulting lead."
python -m crm query "Who did I speak with about data warehouse consulting?"
```

`--notes` and `--voice` are both optional. Query default `--limit` 5. Results are name, company, title, notes from `contacts`.

**Key files:** `crm/graph.py` (`merge_context`), `crm/cli.py`, `crm/state.py`, `crm/search.py`, `tests/test_voice.py`, `tests/test_crm.py`, `tests/test_cli.py`

---

## Discord DM adapter

**Status:** Done. Independent verifier PASS (`pytest -q` → 101 passed, 2 deselected). Follow-up fix: name comes from the card; Groq embed 404 no longer fails the run.

**What we built:** A Discord DM adapter. Image + optional notes wait in memory. The bot does not ask for a name. Follow-up voice or `save` runs the existing capture graph with `name=None`; `validate_extraction` copies `full_name` from the card. `/query` searches and does not write.

**Graph change:** none.

**CLI:**

```bash
python -m crm.discord_bot
```

Needs `DISCORD_BOT_TOKEN` and Message Content Intent. DMs only.

**Key files:** `crm/discord_bot.py`, `tests/test_discord.py`, `how_to_run.md`

---

## What is intentionally not built yet

These were not specified as later work:

- PostgreSQL / SQLAlchemy / migrations
- pgvector / RAG chat
- Telegram, Slack, or WhatsApp
- Structured reminders / follow-up extraction from the transcript
- A live-vision LangSmith experiment (this task’s experiment uses fakes so it stays offline)

---

## Where to look

| Topic | File |
| --- | --- |
| Graph nodes and edges | `crm/graph.py` |
| Shared graph state | `crm/state.py` |
| Contact schema | `crm/schemas.py` |
| SQLite store | `crm/db.py` |
| Vector index | `crm/db.py` (`contact_embeddings`), `crm/search.py` |
| Write verification | `crm/graph.py` (`verify_write`), `tests/test_verify.py` |
| Normalization | `crm/normalize.py` |
| Groq vision + Whisper | `crm/providers/groq.py` |
| Groq embeddings | `crm/providers/embeddings.py` |
| Provider interfaces | `crm/providers/base.py` |
| CLI (capture + query) | `crm/cli.py` |
| Discord DM adapter | `crm/discord_bot.py` |
| Tracing | `crm/tracing.py` |
| Eval dataset + experiment | `eval/dataset.json`, `crm/eval.py`, `eval/latest_experiment.json` |
| Sample card | `input/visiting_card.png` |
| Sample voice | `input/6134386456120009929.ogg` |
| How to run | `how_to_run.md` |
| What to understand | `understandable_so_far.md` |
| Learning harness | `AGENTS.md` |
| Ponytail rule | `.cursor/rules/ponytail.mdc` |
| Graphify rule | `.cursor/rules/graphify.mdc` |
| Code knowledge graph | `graphify-out/graph.json` |

---

## Portfolio documentation and learning roadmap — 2026-09-14

**Status:** Done. Independent verifier PASS. Documentation only; no future feature was implemented.

**What was built:** Rewrote the README around implemented capture, context, persistence, verification, indexing, retrieval, tracing, and Discord. Added an evidence-backed review assessment and architecture guide. Added 24 core learning cards and three optional discovery briefs, with dependencies, scope, acceptance criteria, verification, graph impact, and learning questions. Aligned the run guide with current matching and retrieval behavior.

**Graph change:** None. Documented the existing capture graph and the separate query path. No `crm/` changes, so no Graphify rebuild was required.

**Important files:** [README.md](README.md), [architecture guide](docs/architecture.md), [development roadmap](docs/development-roadmap.md), [run guide](how_to_run.md), and both learning-history documents.

**Verification:** Independently ran `LANGSMITH_API_KEY='' LANGCHAIN_API_KEY='' LANGSMITH_TRACING=false LANGCHAIN_TRACING_V2=false .venv/bin/python -m pytest -q`: **102 passed, 2 deselected, 142 warnings in 14.26s**. Relative links/anchors, Markdown fences, documentation claims against source, and `git diff --check` passed. No live-model quality evaluation was run.

**Evidence limits:** Default vectors are token hashes; perfect fake-provider experiment scores establish workflow behavior, not model quality. Write verification does not establish extraction truth. Discord contacts share one database. Detailed constraints and proposed improvements are in the architecture guide; older task entries retain their historical terminology.

**Next:** Wait for the human to select one roadmap card. The roadmap is not authorization to execute future tasks automatically.

## Development roadmap engineering handoff — 2026-09-15

**Status:** Done. Independent verifier PASS. Documentation only.

**What changed:** Renamed the roadmap to [docs/development-roadmap.md](docs/development-roadmap.md). Added completed capabilities with source/test evidence, sound design choices to preserve, priority milestones, and proposed evaluation, embedding/index, and retry contracts. All 24 core cards now include numbered implementation steps; cards 14, 21 and 22 have individually selectable substeps. Preserved the three optional discovery briefs and added a copy-paste agent handoff, status rules, and completion evidence checklist. Updated README, architecture, and both history-document links.

**Graph change:** None. No application, configuration, or test code changed.

**Verification:** Independent documentation checks covered 9 authored Markdown files and 65 relative links/anchors with zero issues; stable task IDs, required fields and numbered steps passed. `git diff --check` passed. The application suite was not rerun for this documentation-only change; the last recorded result remains **102 passed, 2 deselected, 142 warnings on 2026-09-14**.

**Next:** The human selects one development card or labeled substep. No future implementation began.

## Card 01 — Trace one success and three failures — 2026-09-16

**Status:** Done. Independent verifier PASS. Documentation only; no `crm/` changes.

**What was built:** New `docs/graph-walkthrough.md` (125 lines) tracing card-only success (13 nodes), invalid image (6 nodes), failed verification (10 nodes), and embedding failure (13 nodes + vector-store variant). Each trace lists visited nodes via `graph.stream`, status transitions, provider calls (extractor/transcriber/embedder), store calls (`find_match`/`create`/`get`/`upsert_embedding`), and committed writes (row yes/no, vector yes/no). Distinguishes executed guarded node (in stream but no-op) from skipped edge (absent from stream). Reconciles README diagram labels VP/P/Q/O with routers and confirms query path is separate. Answers why error can keep saved contact ID: writes commit before indexing, failures don't roll back.

**Graph change:** None. No application, configuration, or test code changed.

**Verification:** Focused `.venv/bin/python -m pytest -q tests/test_graph.py tests/test_verify.py tests/test_embeddings.py` → **25 passed**. Full `.venv/bin/python -m pytest -q` → **102 passed, 2 deselected, 142 warnings**. Verifier independently reproduced invalid-image (6 nodes, 0/0/0 calls), failed-verify (10 nodes, embedder 0 calls), and embed-fail (13 nodes, status error with verified contact) traces; confirmed failed verification never embeds via `tests/test_embeddings.py:test_failed_verify_skips_embedding`. `git status --short crm/ tests/` empty.

**Next:** Wait for human to select next card (02 needs 01, now unblocked; 04 needs 01, now unblocked). No future implementation began.

## Card 02 — Reproducible dependency baseline — 2026-09-16

**Status:** Done. Independent verifier PASS. Config/docs only; no `crm/` changes.

**What was built:** New `requirements-lock.txt` (69 lines, full `pip freeze` pin minus editable line) + `how_to_run.md` §3b with ONE repeatable path: `venv /tmp/...` → `pip install -r requirements-lock.txt` → `pip install -e . --no-deps` → `pytest -q`. Baseline Python 3.13.4 Darwin x86_64 pip 25.1.1. Documents `sqlite-vec`+`apsw` prerequisite and APSW wrapper (`crm/db.py`), with embedding-test failure as actionable install error, not silent skip. No upgrades, no new runtime deps, no CI.

**Graph change:** None.

**Verification:** Repo `.venv/bin/python -m pytest -q` → **102 passed, 2 deselected, 142 warnings**. Verifier fresh venv `/tmp/crm-02-verify-venv` → lock install clean → `pip install -e . --no-deps` ok → **102 passed, 2 deselected, 142 warnings in 12.21s**, identical. `git status crm/` empty, `pyproject.toml` diff empty, no `.github/workflows`.

**Next:** Wait for human. Card 03 (CI, needs 02) now unblocked. No future implementation began.

## Card 03 — Offline GitHub Actions CI — 2026-09-16 (IN PROGRESS, hosted run outstanding)

**Status:** Local part DONE, independent verifier PASS (local). Card stays IN PROGRESS — no hosted run exists yet, per stop rule do not mark DONE before run URL exists.

**What was built:** New `.github/workflows/tests.yml` (25 lines, greenfield): push+PR on `[main, develop]`, `permissions: contents: read`, `setup-python@v5` `"3.13"`, `pip install -r requirements-lock.txt` → `pip install -e . --no-deps` → `pytest -q` with `LANGSMITH_TRACING`/`LANGCHAIN_TRACING_V2` false. No secrets, uploads, matrix, or deploys. `pyproject.toml` untouched (`addopts -m "not live"` already excludes live). No `crm/` changes.

**Verification (local):** Repo `.venv` with tracing flags off → **102 passed, 2 deselected**. Verifier fresh `/tmp/crm-03-verify-venv` exact CI sequence → **102 passed, 2 deselected**. YAML parses; triggers/permissions/Python verified; secret/upload/continue-on-error/matrix/deploys all absent.

**Outstanding:** Commit + push + PR to trigger first hosted run, then record run URL + result in roadmap card 03 status and here. Uncommitted: `.github/` untracked, roadmap 03 IN PROGRESS line.

**Next:** Wait for human (explicit commit/push authorization needed for hosted evidence). No future implementation began.
