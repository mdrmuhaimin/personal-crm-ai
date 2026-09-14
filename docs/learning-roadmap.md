# Learning roadmap: evidence before expansion

All boxes below are **planned**, not implemented or authorized for automatic execution. The human selects one ID at a time. This documentation task does not start any of them. Read [current architecture](architecture.md) and [AGENTS.md](../AGENTS.md) before implementation.

## Handoff contract for the next development agent

For the selected card, inspect Graphify first when its graph exists, then the named files. Confirm acceptance criteria; deliver the Learning Step before coding. Spawn a fresh implementer with no subagents; use the smallest working diff and meaningful failing tests. Inspect its changes, give the Implementation Update, and use a separate verifier. On PASS, report exact evidence, teach the checkpoint, update both learning-history files, and stop. On FAIL, correct only that card and independently verify again.

Each card targets approximately 30–90 minutes of focused implementation; data collection, model downloads, and human labeling can take longer. If the stated deliverable cannot fit, split it before coding. Existing paths below identify inspection targets; paths marked **new** are proposed. No card permits extra packages, graph nodes, migrations, or product features beyond its scope. Each card inherits this **stop rule: satisfy only its acceptance criteria, obtain independent verification, update history, and wait for the next human selection**.

Recommended order: 01 → 02–04; 05–08 and 09–10 can be independent learning tracks; 10 → 11 → 12 → 13 → 14 → 15 → 16. Reliability/presentation cards 17–24 can be selected independently where their dependency allows. No agent should execute this as a batch plan.

## Foundations and reproducibility

### 01 — Trace one success and three failures (45 minutes)

- **Depends on:** nothing. **Goal/concept:** explain graph control flow, guards, and side effects from code.
- **Files:** `crm/graph.py`, `crm/state.py`, `tests/test_graph.py`, `tests/test_verify.py`; **new** `docs/graph-walkthrough.md`.
- **Scope:** document card-only success, invalid image, failed verification, and embedding failure; distinguish an executed guarded node from a skipped edge. No new graph nodes.
- **Acceptance:** each trace lists visited nodes, status transitions, provider calls, and possible committed writes; matches actual graph invocation.
- **Verify:** focused existing tests plus small recording fakes only where traces lack evidence; confirm failed verification never embeds.
- **Graph change:** none. **Learning question:** why can an error result still have a saved contact ID?

### 02 — Record a reproducible dependency baseline (60 minutes)

- **Depends on:** 01. **Goal/concept:** reproducibility versus an unbounded dependency list.
- **Files:** `pyproject.toml`, `how_to_run.md`; **new** a single chosen dependency lock/constraints artifact.
- **Scope:** resolve the current supported Python environment and document one repeatable install. Do not upgrade unrelated packages or promise unsupported OS coverage.
- **Acceptance:** a fresh virtual environment installs the same resolved versions and runs offline tests; document native SQLite extension prerequisites.
- **Verify:** fresh install and full offline suite; show an actionable install failure instead of masking it.
- **Graph change:** none. **Learning question:** why is a passing developer environment insufficient to reproduce a result?

### 03 — Add offline GitHub Actions CI (60 minutes)

- **Depends on:** 02. **Goal/concept:** independently repeat regression evidence on pushes/PRs.
- **Files:** `pyproject.toml`; **new** `.github/workflows/tests.yml`.
- **Scope:** one supported Python version initially, locked install, offline pytest; no deployments or provider secrets.
- **Acceptance:** workflow runs the offline suite and fails on failed tests; live tests remain excluded; no real contact artifacts uploaded.
- **Verify:** inspect trigger/install/test commands, run equivalent commands locally, then record actual hosted result when available; do not claim green CI before a run exists.
- **Graph change:** none. **Learning question:** what does green CI prove about real-card extraction?

### 04 — Explain workflow versus autonomous agents (30 minutes)

- **Depends on:** 01. **Goal/concept:** explain intentional bounded orchestration in an interview.
- **Files:** `docs/architecture.md`, `crm/graph.py`; **new** `docs/design-decisions.md`.
- **Scope:** a short decision record comparing predefined transitions with model-selected tools; explain why this CRM needs no planning agent.
- **Acceptance:** identify deterministic routers, probabilistic nodes, termination, and one scenario where dynamic planning might be justified; explicitly keep it out of scope.
- **Verify:** cross-check every example against code and answer the walkthrough questions without adding features.
- **Graph change:** none. **Learning question:** does using LangGraph automatically make an application autonomous?

## Real interpretation evidence

### 05 — Define and label a small card dataset (60–90 minutes plus collection)

- **Depends on:** 01. **Goal/concept:** ground truth, consent, and held-out evaluation.
- **Files:** `eval/dataset.json`, `crm/schemas.py`; **new** `eval/real_cards/manifest.json`, `eval/real_cards/README.md`.
- **Scope:** specify 10–20 consented or purpose-created realistic card images; identify synthetic versus real provenance; label visible fields and absent fields. Keep sensitive media private unless sharing is authorized.
- **Acceptance:** stable IDs, media references/checksums, human-reviewed labels, layout/blur/missing-field categories, and a frozen development/held-out split. If media is unavailable, stop with a reviewed manifest specification, explicitly incomplete collection.
- **Verify:** validate schema, unique IDs, disjoint splits, valid media references, no accidental personal-data publication.
- **Graph change:** none. **Learning question:** why must an absent field be explicitly labeled rather than omitted?

### 06 — Implement extraction scoring (60 minutes)

- **Depends on:** 05 schema/labels. **Goal/concept:** measure correctness without rewarding all-null output.
- **Files:** `crm/eval.py`, `tests/test_eval.py`; **new** `crm/eval_metrics.py`, `tests/test_eval_metrics.py` if separation is needed.
- **Scope:** normalized populated-field accuracy, whole-card success, absent-field unsupported prediction rate, and provider/schema failure counts. Keep existing fake metrics clearly named/documented; no live calls.
- **Acceptance:** publish normalization and denominators; missing outputs cannot count as success; display raw counts and per-field results.
- **Verify:** hand-computed fixtures for perfect, wrong, missing, all-null, invented, and failed outputs; zero-opportunity metrics report N/A.
- **Graph change:** none. **Learning question:** how can null preservation be perfect while extraction is useless?

### 07 — Add an opt-in real-card evaluation runner (60–90 minutes)

- **Depends on:** 05 complete data, 06. **Goal/concept:** separate experiments from the user's CRM.
- **Files:** `crm/providers/groq.py`, `crm/eval.py`; **new** `crm/eval_live.py`, `tests/test_eval_live.py`.
- **Scope:** run extraction on the frozen manifest and emit raw predictions/errors plus metric inputs; no production DB writes; explicit live mode and optional upload only.
- **Acceptance:** record model, prompt version, dataset checksum, timestamp, durations, and provider errors; never silently substitute fakes; failures remain scored; cost/token usage only when actually available.
- **Verify:** fake transport tests for timeouts, invalid JSON, missing key/media, no DB writes and no default uploads; live execution is separately opt-in.
- **Graph change:** none; evaluation calls interpretation separately. **Learning question:** why should an experiment avoid a personal contact database?

### 08 — Run and publish an honest quality report (45–60 minutes plus live run)

- **Depends on:** 07; human-provided media/API budget. **Goal/concept:** turn measurements into an evidence-backed claim.
- **Files:** `README.md`; **new** `eval/reports/card-baseline.md` and sanitized result artifact.
- **Scope:** run once on the frozen split; report counts, per-field quality, unsupported fields, errors, latency, and three failure examples. No tuning during held-out evaluation.
- **Acceptance:** distinguish development and held-out results; disclose sample size and limitations; link exact configuration/results; report unrun cases as unrun, never fabricate scores.
- **Verify:** recompute aggregates from saved results and inspect redaction before sharing.
- **Graph change:** none. **Learning question:** what claim can 15 cards support, and what claim would overreach?

## Retrieval and component depth

### 09 — Create relevance judgments (60–90 minutes)

- **Depends on:** 01. **Goal/concept:** test meaning beyond shared words.
- **Files:** `crm/search.py`, `tests/test_embeddings.py`; **new** `eval/retrieval/dataset.json`.
- **Scope:** small fixed synthetic/consented contact corpus and 15–25 human-labeled queries: synonyms with little overlap, exact names, multiple relevant contacts, misleading overlap, and no-answer cases. No model choice yet.
- **Acceptance:** stable contact/query IDs, full relevant-ID sets, category labels, development/held-out split, and documented relevance rubric; freeze labels before comparing models.
- **Verify:** unique IDs, referential integrity, disjoint splits; manually review ambiguous labels.
- **Graph change:** none. **Learning question:** why does “warehouse consulting” versus “warehouse modernization” provide weak semantic evidence?

### 10 — Implement retrieval metrics and hash baseline (60–90 minutes)

- **Depends on:** 09. **Goal/concept:** retrieval relevance is measurable independently of generated answers.
- **Files:** `crm/search.py`, `crm/providers/embeddings.py`; **new** `crm/eval_retrieval.py`, `tests/test_eval_retrieval.py`, baseline result artifact.
- **Scope:** precision@k, recall@k, MRR@k for answerable queries; separate unanswerable false-positive results; report current hash baseline. No threshold tuning or runtime change.
- **Acceptance:** use fixed k precision, count missing slots as nonrelevant, deduplicate IDs, handle zero relevant labels separately, record dataset and embedding configuration.
- **Verify:** hand-computed rankings for multiple hits, missing results, duplicates, wrong first result, empty index, no-answer; recompute aggregate baseline.
- **Graph change:** none. **Learning question:** when can precision rise while recall falls?

### 11 — Define an explicit embedding contract (60 minutes)

- **Depends on:** 10. **Goal/concept:** vector compatibility is more than length.
- **Files:** `crm/providers/base.py`, `crm/providers/embeddings.py`, `tests/test_groq_provider.py`; **new** focused contract tests.
- **Scope:** define model identity/revision, dimension, normalization and document-format version; validate nonempty finite vectors of the declared length. Explicitly label the hash provider; do not choose a learned runtime default.
- **Acceptance:** selected provider errors cannot silently change embedding spaces; same-dimension different-model identity stays distinguishable; clear error on invalid vectors.
- **Verify:** empty/NaN/infinite/wrong-dimension outputs, missing model, stable metadata, existing injected fake compatibility.
- **Graph change:** no nodes; stricter provider boundary. **Learning question:** why can two 384-dimensional vectors be incompatible?

### 12 — Benchmark candidate learned models (60–90 minutes plus downloads)

- **Depends on:** 10, 11. **Goal/concept:** choose a model based on measured tradeoffs.
- **Files:** `crm/eval_retrieval.py`, `pyproject.toml`; **new** a minimal experiment-only Sentence Transformers adapter and benchmark report.
- **Scope:** compare hash baseline with `all-MiniLM-L6-v2` and `multi-qa-MiniLM-L6-cos-v1` candidates on the same labels/hardware. Document model license, dimensions, truncation, language assumptions, cold/warm latency, memory/download footprint. No default runtime switch.
- **Acceptance:** pinned model revisions/configuration, per-category retrieval metrics and raw rankings, reproducible command; decide from development results then evaluate once on held-out data. State if no candidate improves the required use cases.
- **Verify:** mocked adapter error/shape tests; real benchmark artifacts separately, no download during default tests.
- **Graph change:** none. **Learning question:** why might a QA-tuned model differ from a general sentence model for contact notes?

### 13 — Persist and enforce index compatibility (60–90 minutes)

- **Depends on:** 11. **Goal/concept:** derived data has a schema and provenance.
- **Files:** `crm/db.py`, `crm/search.py`, `crm/graph.py`, `tests/test_embeddings.py`.
- **Scope:** store index model/revision/dimension/normalization/document version; refuse mismatched writes and queries with a rebuild instruction. Never drop an existing index automatically.
- **Acceptance:** legacy index is explicitly recognized as unknown/incompatible; equal dimension but wrong model is rejected; contact rows remain readable.
- **Verify:** fresh/legacy/matching/mismatched databases, equal-dimension mismatch, schema-write failure; no contact loss.
- **Graph change:** no new nodes; compatibility checks at index boundaries. **Learning question:** which metadata change requires re-embedding all contacts?

### 14 — Build a safe explicit index rebuild command (60–90 minutes)

- **Depends on:** 13. **Goal/concept:** repair replaceable derived data without replaying business writes.
- **Files:** `crm/db.py`, `crm/cli.py`, `crm/search.py`; **new** `crm/reindex.py`, `tests/test_reindex.py` if needed.
- **Scope:** rebuild vectors from contact rows into a staged index; validate counts/metadata before switching. Back up or preserve the old index on failure. No extraction/transcription or note append.
- **Acceptance:** explicit target DB/model, idempotent repeat run, all expected contacts indexed, interruption/provider failure leaves old usable state, successful swap updates metadata atomically where supported.
- **Verify:** inject failure midway and at swap; compare original contact rows byte-for-byte or field-for-field; rerun yields same coverage with no duplicate notes.
- **Graph change:** capture unchanged; separate `read contacts → embed → validate staged index → swap` command.
- **Learning question:** why is rebuilding an index safer than replaying capture?

### 15 — Select and wire the measured runtime provider (60 minutes)

- **Depends on:** 12, 14. **Goal/concept:** configuration must preserve the experimental decision.
- **Files:** `crm/providers/`, `crm/cli.py`, `crm/discord_bot.py`, `crm/graph.py`, `README.md`, `pyproject.toml`.
- **Scope:** use the benchmark-selected provider in both capture and query, including Discord; explicitly configure hash baseline only as a diagnostic option. No silent fallback, no migration on startup.
- **Acceptance:** consistent identity/configuration across entry points, actionable missing-model/download errors, documented installation/rebuild path; key requirements reflect selected provider.
- **Verify:** adapter configuration parity and failures with fakes; offline suite; one opt-in local learned-model smoke test on temporary data.
- **Graph change:** same graph, provider substitution. **Learning question:** what breaks if capture and query use different models?

### 16 — Calibrate no-answer behavior (60 minutes)

- **Depends on:** 15. **Goal/concept:** nearest does not mean relevant.
- **Files:** `crm/search.py`, CLI/Discord formatters, `crm/eval_retrieval.py`, relevant tests.
- **Scope:** evaluate a simple threshold for the selected normalized model on development labels; document it as model-specific. No answer generation or LLM relevance judge.
- **Acceptance:** empty/unrelated queries can abstain; report relevant-query recall loss and unanswerable false positives on held-out data; if no useful threshold exists, document failure rather than claim success.
- **Verify:** empty index/query, unrelated and borderline queries, top-k limits, model-metadata mismatch.
- **Graph change:** capture unchanged; query `rank → threshold → hits/no matches`. **Learning question:** why cannot vector distance be displayed as percentage confidence?

## Reliability and presentation

### 17 — Make partial index failure explicit (60 minutes)

- **Depends on:** 01, 14. **Goal/concept:** successful business writes and failed derived work can coexist.
- **Files:** `crm/state.py`, `crm/graph.py`, `crm/cli.py`, `crm/discord_bot.py`, `tests/test_embeddings.py`.
- **Scope:** report contact write success and index failure distinctly, with a reindex recovery instruction. Preserve existing failure status compatibility unless explicitly documented.
- **Acceptance:** no false “fully complete” result; both adapters show saved ID and failed indexing; recovery does not ask users to recapture.
- **Verify:** embed and vector-store failures after create/update, unchanged contact contents, successful reindex recovery.
- **Graph change:** existing nodes/status output clarified; no retry loop. **Learning question:** why is one success boolean insufficient here?

### 18 — Specify capture retry identity (45 minutes)

- **Depends on:** 17. **Goal/concept:** duplicate contact matching is not request idempotency.
- **Files:** `crm/db.py`, `crm/discord_bot.py`, `crm/cli.py`; **new** a short retry decision record.
- **Scope:** choose stable request-ID ownership/lifetime for CLI and Discord, distinguish legitimate repeated notes from retries, define storage and failure transaction. Specification only.
- **Acceptance:** concrete same-ID/same-payload, same-ID/different-payload, and new-ID/same-card outcomes; minimal schema/transaction brief for 19.
- **Verify:** walk through timeout before/after commit, restart, and index failure; no ambiguous note behavior.
- **Graph change:** none yet. **Learning question:** why is hashing the note text alone an unsafe idempotency rule?

### 19 — Implement the selected retry contract (60–90 minutes)

- **Depends on:** 18 accepted. **Goal/concept:** repeat requests without duplicate side effects.
- **Files:** only files selected in 18; focused store/adapter tests.
- **Scope:** persist request identity with contact write using one transaction; do not add queues, generalized event sourcing, or autonomous retries.
- **Acceptance:** replay returns previous contact outcome without appending notes; mismatched payload for existing ID fails; new intentional capture remains possible; index recovery uses 14.
- **Verify:** duplicate delivery, conflicting request ID, transaction failure, restart replay; note counts and returned ID remain correct.
- **Graph change:** follow 18's explicit minimal route, document it before coding; split if more than one bounded transition is needed.
- **Learning question:** where must the request record commit relative to the contact write?

### 20 — Restrict Discord to an explicit owner (60 minutes)

- **Depends on:** 01. **Goal/concept:** a DM channel does not provide data authorization.
- **Files:** `crm/discord_bot.py`, `tests/test_discord.py`, `how_to_run.md`.
- **Scope:** configured owner allowlist checked before attachment download, capture, or query in both message and slash paths. No multi-tenant database redesign.
- **Acceptance:** missing owner configuration fails closed; unauthorized users cannot retrieve or write contacts; authorized owner flow still works.
- **Verify:** unauthorized message/slash/download/query/capture calls are blocked; no leaked contact details; guild rejection remains intact.
- **Graph change:** graph unchanged; authorization at adapter entry. **Learning question:** why is pending state keyed by user ID insufficient for privacy?

### 21 — Bound attachment and sensitive-log retention (60–90 minutes)

- **Depends on:** 20. **Goal/concept:** lifecycle ownership includes failure cleanup.
- **Files:** `crm/discord_bot.py`, `crm/tracing.py`, `tests/test_discord.py`, run guide.
- **Scope:** clean owned temporary attachments on success/failure/replacement and define pending-session expiry; remove raw DM/query text from routine logs; document trace opt-in data exposure. Split expiry into a separate selected card if needed.
- **Acceptance:** only adapter-owned files are removed; pending files remain while needed; failure paths clean up; no card content in routine logs.
- **Verify:** success/error/replaced pending/expired pending, unrelated-file preservation, captured log redaction; tracing behavior tested without uploads.
- **Graph change:** none. **Learning question:** who owns a temporary file while a capture waits for voice?

### 22 — Add transcription evidence separately (60–90 minutes plus labeling)

- **Depends on:** 06, 07. **Goal/concept:** good card extraction does not prove good voice understanding.
- **Files:** `crm/providers/groq.py`, evaluation metrics/runner; **new** consented voice manifest and report.
- **Scope:** small labeled voice set, WER and business-entity error examples; frozen transcripts, duration/language/noise categories. Split collection from runner implementation if media is unavailable.
- **Acceptance:** explicit empty-reference policy, failed calls counted, model/version recorded, privacy-reviewed report; no new summarizer.
- **Verify:** hand-computed WER including insertions/deletions/substitutions and empty reference; mocked provider failure; opt-in real run.
- **Graph change:** none. **Learning question:** why can low WER still misrepresent a critical company name?

### 23 — Package a reproducible portfolio demonstration (60 minutes)

- **Depends on:** 03, 08, 15; may document outstanding evidence explicitly if not all complete.
- **Files:** `README.md`; **new** `docs/demo.md` and sanitized demo assets.
- **Scope:** script a 2–3 minute recording: capture, note, verified row, duplicate behavior, meaningful query, one failure, evidence report. Use synthetic/consented contacts.
- **Acceptance:** reproducible commands and expected outputs; real recording link only once recorded; honest implemented/planned labels; no fabricated CI badge or quality number.
- **Verify:** rehearse with a temporary database, follow links, inspect recording for credentials/private contacts.
- **Graph change:** none. **Learning question:** which part of the demo proves orchestration, and which proves model quality?

### 24 — Polish repository metadata and generated artifacts (45–60 minutes)

- **Depends on:** 23. **Goal/concept:** discoverability without erasing engineering history.
- **Files:** `.gitattributes`, `.gitignore`, `README.md`; GitHub About/topics; license file only after user choice.
- **Scope:** inspect actual metadata/language statistics; draft an accurate description/topics; choose generated-artifact handling that preserves Graphify availability and required teaching files. No broad directory deletion.
- **Acceptance:** present concrete metadata and license options for the human; do not select a legal license on their behalf; apply external metadata only when selected task authorization covers it. Existing Graphify query/update instructions still work.
- **Verify:** review tracked file diff, links, Graphify regeneration/query, and post-change GitHub metadata if changed.
- **Graph change:** none. **Learning question:** why is a license decision different from a repository description?

## Optional later work — separate briefs required

These are discovery cards, not authorization to add the listed features. Select one only after the core evidence and access boundaries are satisfactory.

### L1 — Define a RAG answer contract (45 minutes)

- **Depends on:** 16, 20 and adequate measured retrieval. **Files:** new design brief plus relevance dataset references.
- **Goal/concept:** evidence-grounded generation. **Scope:** specify contact-ID citations, abstention, answer fields, prompt-injection handling, and a labeled claim-support rubric; no generation implementation yet.
- **Acceptance/verification:** walk through supported, unsupported, contradictory, and empty-context examples; distinguish retrieval relevance from answer faithfulness; define human-audited hallucination denominators.
- **Proposed graph change:** separate query workflow `retrieve → relevance gate → generate with citations → validate/abstain`; capture unchanged.
- **Learning question:** can an answer be faithful to irrelevant context and still be wrong? **Stop:** deliver the reviewed contract; future implementation needs its own card.

### L2 — Define draft-only follow-up behavior (45 minutes)

- **Depends on:** privacy boundary and reliable retrieval. **Files:** new follow-up brief.
- **Goal/concept:** separate probabilistic drafting from consequential actions. **Scope:** grounded draft with contact/context attribution and human editing; no sending tool.
- **Acceptance/verification:** examples for missing address, ambiguous contact, unsupported claims, and incomplete context; drafts never promise unsupported dates or commitments.
- **Proposed graph change:** separate `select verified contact → draft → human review`; no send edge.
- **Learning question:** which claims must be grounded before a draft is useful? **Stop:** brief only.

### L3 — Investigate LinkedIn tracking feasibility (45–60 minutes)

- **Depends on:** explicit user selection. **Files:** new feasibility/decision note.
- **Goal/concept:** external data access, identity, provenance, and freshness. **Scope:** inspect current official permitted access methods, consent and account capabilities; define what profile identity and change events would mean. No scraping, automation, credentials use, or polling implementation.
- **Acceptance/verification:** source-backed access constraints and unresolved decisions; distinguish a profile URL from verified identity; define refresh costs and stale-data behavior.
- **Graph change:** none until an access method and product contract are approved.
- **Learning question:** how would you prevent an unrelated profile from updating a CRM contact? **Stop:** feasibility note; no integration started.
