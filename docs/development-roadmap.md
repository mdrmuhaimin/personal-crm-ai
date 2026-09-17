# Development roadmap: current strengths and implementation plan

Reviewed against the checkout on **2026-09-15**. This is the development handoff for a working personal CRM: what is built, which design choices to preserve, and how to close its evidence and reliability gaps. Read [current architecture](architecture.md) and [AGENTS.md](../AGENTS.md) before implementation.

Navigate: [completed strengths](#completed-foundation-what-is-done-and-done-right) · [priorities/status](#priorities-and-milestone-exit-evidence) · [engineering contracts](#principal-engineering-plan-proposed-contracts-not-current-code) · [agent handoff](#handoff-contract-for-the-next-development-agent) · [foundations 01–04](#foundations-and-reproducibility) · [interpretation 05–08](#real-interpretation-evidence) · [retrieval 09–16](#retrieval-and-component-depth) · [reliability/presentation 17–24](#reliability-and-presentation) · [later discovery](#optional-later-work--separate-briefs-required).

## Completed foundation: what is done and done right

| Capability/status | Implementation and evidence | Why preserve it / remaining limit |
| --- | --- | --- |
| DONE — explicit capture workflow | [Graph](../crm/graph.py), [state](../crm/state.py), [graph tests](../tests/test_graph.py) | Named nodes, guards, and conditional edges expose side effects and termination. No autonomous planning is needed; card 01 documents failure traces. |
| DONE — card extraction and separate context | [Groq adapters](../crm/providers/groq.py), [schema](../crm/schemas.py), [extraction tests](../tests/test_extract.py), [voice tests](../tests/test_voice.py) | Direct providers behind protocols isolate uncertain interpretation; voice/typed notes do not overwrite identity. Real quality remains unmeasured (05–08, 22). |
| DONE — deterministic storage and matching | [ContactStore](../crm/db.py), [normalization](../crm/normalize.py), [CRM tests](../tests/test_crm.py) | Python chooses CREATE/UPDATE using the strongest available key; SQLite rows remain authoritative. Contact matching does not prevent retry note duplication (18–19). |
| DONE — post-write verification | [Verifier node](../crm/graph.py), [verification tests](../tests/test_verify.py) | Re-reading the row checks actual persistence before indexing. It does not establish extraction truth or roll back a later indexing failure (17). |
| DONE — vector indexing and contact retrieval | [Search](../crm/search.py), [embedder](../crm/providers/embeddings.py), [embedding tests](../tests/test_embeddings.py) | A replaceable vector index points to canonical contact IDs. Current default hashes offer lexical matching, not demonstrated semantic retrieval (09–16). |
| DONE — CLI and Discord interfaces | [CLI](../crm/cli.py), [Discord](../crm/discord_bot.py), [adapter tests](../tests/test_discord.py) | Adapters reuse graph/search behavior. Shared storage lacks owner authorization and attachment lifecycle controls (20–21). |
| DONE — offline evaluation and optional traces | [Evaluation](../crm/eval.py), [tracing](../crm/tracing.py), [evaluation tests](../tests/test_eval.py) | Fakes isolate orchestration correctness; tracing exposes execution. Neither measures real-model quality. |
| DONE — current README and architecture explanation | [README](../README.md), [architecture](architecture.md), [history](../progress_so_far.md) | Claims distinguish implemented behavior from evidence. `.gitignore` already ignores `graphify-out/`; public metadata/license checks and demo remain planned (23–24). |

Last recorded full offline test evidence is **2026-09-16: 102 passed, 2 deselected, 142 warnings**, re-verified identically across cards 02–03 implementation, verification, and both commit gates (see [progress history](../progress_so_far.md)). This roadmap edit did not introduce new test behavior. Historical live smoke checks are not a labeled quality benchmark. No current claim establishes production readiness or generated RAG-answer quality; queries return contact records.

## Priorities and milestone exit evidence

| Priority/milestone | Cards and dependencies | Evidence needed to close it |
| --- | --- | --- |
| P0 — explain and reproduce | 01 → 02 → 03; 01 → 04 | Source-backed traces, reproducible install, actual hosted CI result, workflow decision record |
| P0 — protect personal use | 01 → 20 → 21 | Unauthorized access denied before downloads/queries; owned-file cleanup and redacted routine logs |
| P1 — measure interpretation | 01 → 05 → 06 → 07 → 08; 06/07 → 22 | Frozen labels, checked scorers, saved real predictions and honest quality report |
| P1 — establish semantic retrieval | 01 → 09 → 10 → 11 → 12; 11 → 13 → 14; 12/14 → 15 → 16 | Hash baseline versus learned models, enforced compatibility, tested rebuild, measured relevance/abstention |
| P1 — recover without duplicate writes | 01/14 → 17 → 18 → 19 | Explicit partial success and crash/replay evidence for one request transaction |
| P2 — present the evidence | 03/08/15 → 23 → 24 | Reproducible demonstration, linked reports, accurate metadata |
| Later discovery only | L1–L3 with their stated dependencies | Reviewed product/access contracts; no feature implementation authorized |

The human selects **one card or labeled substep** at a time; these priorities are guidance, not automatic execution authorization. Implementation cards **01–04: DONE** (see per-card status lines below); **05–24: NOT STARTED**. Discovery cards **L1–L3: NOT STARTED**. The completed foundation above is separate from these future cards.

For each selected card, add a status line beneath its heading: `Status: IN PROGRESS | owner: … | started: …`. Use `BLOCKED — reason/evidence` if inputs are unavailable; retain unfinished acceptance criteria. Change to `DONE — verifier PASS | date | commit/diff | evidence path` only after independent verification and history updates. For split cards track each substep; the parent stays incomplete until all substeps pass. A written plan or mocked live call is not a completed live evaluation.

## Principal engineering plan: proposed contracts, not current code

Keep Python, the explicit capture graph, provider protocols, SQLite and `sqlite-vec`. Do not add PostgreSQL, LangChain, a web UI, a second application agent, or generated answers in these cards. The improvements strengthen boundaries already present. The following shapes are design targets for the named cards, not claims about existing interfaces.

### Evaluation contracts — cards 05–10 and 22

The manifest owns ground truth; scorers own deterministic normalization/counts; runners own provider calls and timing; reports consume saved results. Keep these responsibilities separable so reports can be recomputed without spending API calls. Example proposed JSON shapes (ellipsis placeholders must be replaced with full validated values during implementation):

```json
{"schema_version":1,"dataset_id":"cards-v1","examples":[{"id":"card-001","split":"dev","media":{"path":"private/card-001.jpg","sha256":"…"},"provenance":"purpose-created","sharing":"private","categories":["missing-phone"],"expected":{"full_name":"Ada Example","company":"Example Ltd","job_title":null,"email":null,"phone":null,"website":null,"address":null}}]}
{"schema_version":1,"run_id":"…","dataset_sha256":"…","git_revision":"…","model":"…","model_revision":null,"prompt_version":"card-v1","started_at":"…","results":[{"id":"card-001","status":"ok","prediction":{"full_name":"Ada Example"},"error":null,"duration_ms":123,"usage":null}]}
{"schema_version":1,"dataset_id":"retrieval-v1","contacts":[{"id":"c01","full_name":"Ada Example","company":"Example Ltd","job_title":null,"notes":"Helps firms migrate legacy databases"}],"queries":[{"id":"q01","text":"Who can modernize our data platform?","split":"dev","category":"synonym","relevant_ids":["c01"]}]}
```

Card 05 must define all seven expected fields, including explicit nulls, and define illegible/ambiguous-field handling before labeling; do not coerce unknown labels into absent fields. Card 07 records one result per example with `ok`, `provider_error`, `schema_error`, or `input_error`; missing results are evaluation failures. Store raw responses privately when needed, publish sanitized artifacts separately, and never overwrite the user's CRM. If a provider exposes no immutable revision or usage, record `null` plus the limitation, not an invented version/cost. Voice manifests add transcript, language, duration and noise categories; shared result metadata stays consistent.

Card 06 defines field normalization once, uses populated-reference fields as the extraction denominator, and counts failed calls as incorrect there. Report absent-field false positives separately with failure counts so all-null/errors cannot look successful. Card 10 keeps human relevance IDs as truth: fixed-k precision, answerable-only recall/MRR, no-answer false positives separately. Record rankings and distances as well as aggregates. L1 would add answer claim support/citations later; today's retrieval cannot measure generated-answer hallucinations.

### Embedding and index contracts — cards 11–16

Today `EmbeddingProvider` exposes only `dimension` and `embed(text)`; the store has one fixed `contact_embeddings` table and no model identity. Card 11 owns one immutable identity definition, used by capture, query and evaluation. Proposed fields:

```text
EmbeddingIdentity(provider, model, revision, dimension, normalization,
                  document_version, query_format_version)
example: (sentence_transformers, selected-model, pinned-revision, measured-dimension,
          l2, contact-v1, query-v1)
```

Keep `embed(text)` unless the selected model demonstrably requires separate query/document encoding; any prefixes belong in versioned shared formatting, not independently in adapters. Validate positive declared dimension, actual vector length, finite numbers and normalization policy; define rejection/handling of zero-norm output explicitly. Selected-model errors must never switch to hashes. Card 12 owns comparison and decision evidence; card 15 owns the shared runtime construction used by CLI, Discord and graph defaults. No downloaded model enters offline tests.

Card 13 owns additive index metadata and compatibility checks in `ContactStore`; `query_contacts` and graph indexing must supply the same identity. Proposed ordinary tables: `index_generations(generation_id, table_name, identity_json, status)` and a singleton `active_index(generation_id)`. Internally generated table names such as `contact_embeddings_g17` must be validated; user strings never become SQL identifiers. Missing legacy metadata is **unknown**, never inferred as compatible by dimension. Preserve legacy vectors and contact rows; fail incompatible vector reads/writes with an explicit rebuild command. A brand-new empty database may bootstrap an empty compatible active generation so capture works before card 14; a database with existing contacts or legacy vectors must use explicit rebuild.

Card 14 owns rebuild from stored rows using the existing `build_search_document`, never capture. For this personal tool, use an exclusive application maintenance lock for the selected database, respected by **all CLI/Discord capture and query entry points**; fail busy with a retry message. The lock must release on process death and cover the entire rebuild/read/write operation. Direct third-party DB writers are unsupported during maintenance; document this boundary. Stop Discord or reject its work while rebuilding. Do not add distributed locking or a background scheduler.

Build a new generation while retaining the old table and pointer. Validate complete contact-ID coverage (including duplicates/missing IDs), vector shape/identity and a sample read. In one ordinary SQLite transaction mark the new generation ready and update the singleton pointer; readers/writers resolve that pointer under the same maintenance contract. Test supported sqlite3/APSW paths: do not assume virtual-table rename is atomic. Crash before pointer commit leaves the old generation active; crash after commit leaves the fully validated new generation active. Incomplete staging tables are ignored and a subsequent explicit rebuild may discard only identified staging generations. Keep the previous generation for rollback, never drop contact data. A first-ever build has no old index: interruption must leave contacts readable and queries explicitly unavailable. Repeated rebuilds must preserve contact fields/timestamps/notes and produce the same ID coverage, not necessarily byte-identical vectors.

### Recovery and retry contracts — cards 17–19

Card 17 exposes two facts in results: contact write verified, index ready/failed. Keep the saved contact ID on index failure and instruct reindex, not recapture. Proposed additive outcome fields: `write_status=verified|failed`, `index_status=ready|failed|not_attempted`; retain existing `status` behavior until adapters/tests explicitly adopt a change.

Card 18 fixes request identity before card 19 adds code. Recommended minimal contract: CLI accepts an explicit reusable `--request-id` and prints a generated ID when omitted; Discord derives an ID from the save/voice event, with the pending card/text/voice snapshot bound to it. Freeze canonical input (media-content checksums, typed notes, relevant input options), hash that payload, and check replay **before** rerunning providers. Same ID/same hash returns the stored contact-write outcome; same ID/different hash fails; new ID/same input is a deliberate new capture and may append notes. Do not derive identity solely from note text or an attachment path. A CLI user must retain the ID to retry after a lost response; no claim of transparent retry otherwise.

Proposed record: `capture_requests(request_id PRIMARY KEY, payload_sha256, contact_id, action, committed_at)`. Card 19 must move request check, contact match/write and request-record insert onto **one connection and transaction**; today's separate store calls do not provide that boundary. Initial execution still verifies by rereading after commit; indexing follows verification outside this transaction. A pre-commit crash leaves no request or contact mutation; a post-commit crash replays the committed-write acknowledgement without appending notes. Concurrent equal IDs resolve to one committed write; conflicting payloads fail.

With this minimal record, replay checks current row existence and returns `replayed` plus the committed contact ID/action; it must explicitly say original field verification and index completion are **not established by replay**. Do not invoke `verify_write` with empty intended state and call that proof. A later legitimate capture may have updated the contact, so present values are not necessarily the first request's values. A missing row is a replay error; a present row is not automatically `complete`. If 18 instead requires original-value verification or replay of final completion, it must add an intended-write snapshot and durable verification/completion outcome with a policy for later updates before 19 begins. The minimal proposal deliberately records committed writes, not completed indexing; use reindex for index recovery.

## Handoff contract for the next development agent

For the selected card, inspect Graphify first when its graph exists, then the named files. Confirm acceptance criteria; deliver the Learning Step before coding. Spawn a fresh implementer with no subagents; use the smallest working diff and meaningful failing tests. Inspect its changes, give the Implementation Update, and use a separate verifier. On PASS, report exact evidence, teach the checkpoint, update both learning-history files, and stop. On FAIL, correct only that card and independently verify again.

Each card targets approximately 30–90 minutes of focused implementation; data collection, model downloads, and human labeling can take longer. If the stated deliverable cannot fit, split it before coding. Existing paths below identify inspection targets; paths marked **new** are proposed. No card permits extra packages, graph nodes, migrations, or product features beyond its scope. Each card inherits this **stop rule: satisfy only its acceptance criteria, obtain independent verification, update history, and wait for the next human selection**.

Recommended order: 01 → 02–04; 05–08 and 09–10 can be independent learning tracks; 10 → 11 → 12 → 13 → 14 → 15 → 16. Reliability/presentation cards 17–24 can be selected independently where their dependency allows. No agent should execute this as a batch plan.

Copy/paste handoff (replace the selected ID; do not leave it open-ended):

```text
Implement ONLY development roadmap card <ID>, substep <if present>.
Read AGENTS.md and docs/development-roadmap.md, including current strengths,
proposed contracts, dependencies and this card. Inspect current code with
Graphify first when graphify-out/graph.json exists, then read affected files.
Confirm dependency evidence; if absent, report the missing prerequisite without
claiming this card completed or starting another card. Give the Learning Step.
Prepare a focused brief naming interfaces, failures, tests and excluded scope.
Use a fresh implementer and separate verifier as AGENTS.md requires; small diff,
failing tests where practical. Do not execute any other roadmap card.
After implementation, give Implementation Update. Independently verify every
acceptance criterion, including failure behavior and architectural boundaries.
On PASS update this card's status, progress_so_far.md and understandable_so_far.md;
run graphify update . when crm/ changed. Report Evaluation and learning checkpoint.
Stop and wait. Do not fabricate unavailable media, API results or hosted CI proof.
```

Evidence checklist for every handoff:

- [ ] Each acceptance criterion maps to a test, inspected artifact, or explicitly outstanding input.
- [ ] Commands, dates, exact results, dataset/model identity and offline/live distinction are recorded as applicable.
- [ ] Failure/recovery tests demonstrate preserved contact data and no unplanned side effects.
- [ ] Independent verifier verdict, important design decisions, links and learning exercise are recorded.
- [ ] Roadmap status and both history documents agree; only the selected scope is marked complete.

## Foundations and reproducibility

### 01 — Trace one success and three failures (45 minutes)

Status: DONE — verifier PASS | 2026-09-16 | docs/graph-walkthrough.md NEW | focused 25 passed, full 102 passed, 2 deselected

1. Read `build_graph`, routers and node guards; make a table of visited nodes versus provider/store calls for the four scenarios.
2. Run recording fakes or existing focused tests and capture the observed statuses/side effects; write the walkthrough with source links and reconcile diagram differences.

- **Depends on:** nothing. **Goal/concept:** explain graph control flow, guards, and side effects from code.
- **Files:** `crm/graph.py`, `crm/state.py`, `tests/test_graph.py`, `tests/test_verify.py`; **new** `docs/graph-walkthrough.md`.
- **Scope:** document card-only success, invalid image, failed verification, and embedding failure; distinguish an executed guarded node from a skipped edge. No new graph nodes.
- **Acceptance:** each trace lists visited nodes, status transitions, provider calls, and possible committed writes; matches actual graph invocation.
- **Verify:** focused existing tests plus small recording fakes only where traces lack evidence; confirm failed verification never embeds.
- **Graph change:** none. **Learning question:** why can an error result still have a saved contact ID?

### 02 — Record a reproducible dependency baseline (60 minutes)

Status: DONE — verifier PASS | 2026-09-16 | requirements-lock.txt NEW | fresh venv 102 passed, 2 deselected

1. Record Python/platform and installed dependency versions; choose one lock/constraints mechanism compatible with the current editable install.
2. Resolve once, install into a new temporary virtual environment, run the offline suite and document the exact repeatable commands and extension prerequisite.

- **Depends on:** 01. **Goal/concept:** reproducibility versus an unbounded dependency list.
- **Files:** `pyproject.toml`, `how_to_run.md`; **new** a single chosen dependency lock/constraints artifact.
- **Scope:** resolve the current supported Python environment and document one repeatable install. Do not upgrade unrelated packages or promise unsupported OS coverage.
- **Acceptance:** a fresh virtual environment installs the same resolved versions and runs offline tests; document native SQLite extension prerequisites.
- **Verify:** fresh install and full offline suite; show an actionable install failure instead of masking it.
- **Graph change:** none. **Learning question:** why is a passing developer environment insufficient to reproduce a result?

### 03 — Add offline GitHub Actions CI (60 minutes)

Status: DONE — verifier PASS | 2026-09-16 | hosted run 35061475950 success (offline 30s, headSha 7335862) | local 102 passed, 2 deselected

1. Add push/PR triggers, one supported Python setup and the locked install command from 02; use minimal read-only repository permissions.
2. Run offline pytest with tracing/uploads disabled and no provider secrets; inspect local equivalence, then record the hosted run URL/result when available.

- **Depends on:** 02. **Goal/concept:** independently repeat regression evidence on pushes/PRs.
- **Files:** `pyproject.toml`; **new** `.github/workflows/tests.yml`.
- **Scope:** one supported Python version initially, locked install, offline pytest; no deployments or provider secrets.
- **Acceptance:** workflow runs the offline suite and fails on failed tests; live tests remain excluded; no real contact artifacts uploaded.
- **Verify:** inspect trigger/install/test commands, run equivalent commands locally, then record actual hosted result when available; do not claim green CI before a run exists.
- **Graph change:** none. **Learning question:** what does green CI prove about real-card extraction?

### 04 — Explain workflow versus autonomous agents (30 minutes)

Status: DONE — verifier PASS | 2026-09-16 | docs/design-decisions.md NEW | 8 code refs spot-checked, 0 wrong

1. Classify each existing node/router as interpretation, deterministic transformation, side effect or routing; identify termination paths.
2. Write the decision record using one actual capture trace and one hypothetical model-planned alternative; explain the additional testing/control burden without implementing it.

- **Depends on:** 01. **Goal/concept:** explain intentional bounded orchestration in an interview.
- **Files:** `docs/architecture.md`, `crm/graph.py`; **new** `docs/design-decisions.md`.
- **Scope:** a short decision record comparing predefined transitions with model-selected tools; explain why this CRM needs no planning agent.
- **Acceptance:** identify deterministic routers, probabilistic nodes, termination, and one scenario where dynamic planning might be justified; explicitly keep it out of scope.
- **Verify:** cross-check every example against code and answer the walkthrough questions without adding features.
- **Graph change:** none. **Learning question:** does using LangGraph automatically make an application autonomous?

## Real interpretation evidence

### 05 — Define and label a small card dataset (60–90 minutes plus collection)

1. Adopt the proposed manifest shape above; specify absent versus unreadable labels and review consent/provenance before adding media references.
2. Collect and human-review all field labels, compute checksums and freeze the split; validate every ID/path/field and record collection gaps explicitly.

- **Depends on:** 01. **Goal/concept:** ground truth, consent, and held-out evaluation.
- **Files:** `eval/dataset.json`, `crm/schemas.py`; **new** `eval/real_cards/manifest.json`, `eval/real_cards/README.md`.
- **Scope:** specify 10–20 consented or purpose-created realistic card images; identify synthetic versus real provenance; label visible fields and absent fields. Keep sensitive media private unless sharing is authorized.
- **Acceptance:** stable IDs, media references/checksums, human-reviewed labels, layout/blur/missing-field categories, and a frozen development/held-out split. If media is unavailable, stop with a reviewed manifest specification, explicitly incomplete collection.
- **Verify:** validate schema, unique IDs, disjoint splits, valid media references, no accidental personal-data publication.
- **Graph change:** none. **Learning question:** why must an absent field be explicitly labeled rather than omitted?

### 06 — Implement extraction scoring (60 minutes)

1. Write tiny hand-scored fixtures first; specify normalization and count formulas including absent fields and failed calls.
2. Implement pure scoring functions over saved labels/results, return numerator/denominator plus per-field counts, and verify N/A handling without providers.

- **Depends on:** 05 schema/labels. **Goal/concept:** measure correctness without rewarding all-null output.
- **Files:** `crm/eval.py`, `tests/test_eval.py`; **new** `crm/eval_metrics.py`, `tests/test_eval_metrics.py` if separation is needed.
- **Scope:** normalized populated-field accuracy, whole-card success, absent-field unsupported prediction rate, and provider/schema failure counts. Keep existing fake metrics clearly named/documented; no live calls.
- **Acceptance:** publish normalization and denominators; missing outputs cannot count as success; display raw counts and per-field results.
- **Verify:** hand-computed fixtures for perfect, wrong, missing, all-null, invented, and failed outputs; zero-opportunity metrics report N/A.
- **Graph change:** none. **Learning question:** how can null preservation be perfect while extraction is useless?

### 07 — Add an opt-in real-card evaluation runner (60–90 minutes)

1. Validate manifest/media before provider calls; require explicit live execution and an output location separate from CRM data.
2. Call the injected extractor per example, validate predictions and write the proposed result envelope including errors; score saved results through 06 and test uploads remain opt-in.

- **Depends on:** 05 complete data, 06. **Goal/concept:** separate experiments from the user's CRM.
- **Files:** `crm/providers/groq.py`, `crm/eval.py`; **new** `crm/eval_live.py`, `tests/test_eval_live.py`.
- **Scope:** run extraction on the frozen manifest and emit raw predictions/errors plus metric inputs; no production DB writes; explicit live mode and optional upload only.
- **Acceptance:** record model, prompt version, dataset checksum, timestamp, durations, and provider errors; never silently substitute fakes; failures remain scored; cost/token usage only when actually available.
- **Verify:** fake transport tests for timeouts, invalid JSON, missing key/media, no DB writes and no default uploads; live execution is separately opt-in.
- **Graph change:** none; evaluation calls interpretation separately. **Learning question:** why should an experiment avoid a personal contact database?

### 08 — Run and publish an honest quality report (45–60 minutes plus live run)

1. Freeze model/prompt and dataset checksums; run the 07 command only with available consented media and authorized API budget.
2. Recompute metrics from saved predictions, review failures/redaction and publish the report with denominators, limitations and configuration links.

- **Depends on:** 07; human-provided media/API budget. **Goal/concept:** turn measurements into an evidence-backed claim.
- **Files:** `README.md`; **new** `eval/reports/card-baseline.md` and sanitized result artifact.
- **Scope:** run once on the frozen split; report counts, per-field quality, unsupported fields, errors, latency, and three failure examples. No tuning during held-out evaluation.
- **Acceptance:** distinguish development and held-out results; disclose sample size and limitations; link exact configuration/results; report unrun cases as unrun, never fabricate scores.
- **Verify:** recompute aggregates from saved results and inspect redaction before sharing.
- **Graph change:** none. **Learning question:** what claim can 15 cards support, and what claim would overreach?

## Retrieval and component depth

### 09 — Create relevance judgments (60–90 minutes)

1. Define the relevance rubric and corpus using the proposed retrieval JSON shape; include difficult low-overlap and no-answer queries.
2. Human-review the complete relevant-ID sets, validate references and freeze split/checksum before inspecting candidate rankings.

- **Depends on:** 01. **Goal/concept:** test meaning beyond shared words.
- **Files:** `crm/search.py`, `tests/test_embeddings.py`; **new** `eval/retrieval/dataset.json`.
- **Scope:** small fixed synthetic/consented contact corpus and 15–25 human-labeled queries: synonyms with little overlap, exact names, multiple relevant contacts, misleading overlap, and no-answer cases. No model choice yet.
- **Acceptance:** stable contact/query IDs, full relevant-ID sets, category labels, development/held-out split, and documented relevance rubric; freeze labels before comparing models.
- **Verify:** unique IDs, referential integrity, disjoint splits; manually review ambiguous labels.
- **Graph change:** none. **Learning question:** why does “warehouse consulting” versus “warehouse modernization” provide weak semantic evidence?

### 10 — Implement retrieval metrics and hash baseline (60–90 minutes)

1. Implement pure ranking scorers against hand-computed examples; deduplicate IDs before taking k and document the rule consistently.
2. Populate a temporary store from the labeled corpus using the current hash provider, save rankings/configuration, and compute per-category and aggregate baseline scores.

- **Depends on:** 09. **Goal/concept:** retrieval relevance is measurable independently of generated answers.
- **Files:** `crm/search.py`, `crm/providers/embeddings.py`; **new** `crm/eval_retrieval.py`, `tests/test_eval_retrieval.py`, baseline result artifact.
- **Scope:** precision@k, recall@k, MRR@k for answerable queries; separate unanswerable false-positive results; report current hash baseline. No threshold tuning or runtime change.
- **Acceptance:** use fixed k precision, count missing slots as nonrelevant, deduplicate IDs, handle zero relevant labels separately, record dataset and embedding configuration.
- **Verify:** hand-computed rankings for multiple hits, missing results, duplicates, wrong first result, empty index, no-answer; recompute aggregate baseline.
- **Graph change:** none. **Learning question:** when can precision rise while recall falls?

### 11 — Define an explicit embedding contract (60 minutes)

1. Add the immutable identity contract above alongside the existing protocol and adapt injected fakes; define zero-vector and normalization validation policies.
2. Label hashes explicitly, remove selected-model fallback and test invalid output plus same-dimension/different-model identities without network calls.

- **Depends on:** 10. **Goal/concept:** vector compatibility is more than length.
- **Files:** `crm/providers/base.py`, `crm/providers/embeddings.py`, `tests/test_groq_provider.py`; **new** focused contract tests.
- **Scope:** define model identity/revision, dimension, normalization and document-format version; validate nonempty finite vectors of the declared length. Explicitly label the hash provider; do not choose a learned runtime default.
- **Acceptance:** selected provider errors cannot silently change embedding spaces; same-dimension different-model identity stays distinguishable; clear error on invalid vectors.
- **Verify:** empty/NaN/infinite/wrong-dimension outputs, missing model, stable metadata, existing injected fake compatibility.
- **Graph change:** no nodes; stricter provider boundary. **Learning question:** why can two 384-dimensional vectors be incompatible?

### 12 — Benchmark candidate learned models (60–90 minutes plus downloads)

1. Verify candidates' current official model cards and pin revisions; create an experiment-only adapter with explicit normalization/truncation configuration.
2. Run identical corpus/query splits against hashes and both candidates; save raw rankings, resource measurements and a decision based on development data, then evaluate held-out once.

- **Depends on:** 10, 11. **Goal/concept:** choose a model based on measured tradeoffs.
- **Files:** `crm/eval_retrieval.py`, `pyproject.toml`; **new** a minimal experiment-only Sentence Transformers adapter and benchmark report.
- **Scope:** compare hash baseline with `all-MiniLM-L6-v2` and `multi-qa-MiniLM-L6-cos-v1` candidates on the same labels/hardware. Document model license, dimensions, truncation, language assumptions, cold/warm latency, memory/download footprint. No default runtime switch.
- **Acceptance:** pinned model revisions/configuration, per-category retrieval metrics and raw rankings, reproducible command; decide from development results then evaluate once on held-out data. State if no candidate improves the required use cases.
- **Verify:** mocked adapter error/shape tests; real benchmark artifacts separately, no download during default tests.
- **Graph change:** none. **Learning question:** why might a QA-tuned model differ from a general sentence model for contact notes?

### 13 — Persist and enforce index compatibility (60–90 minutes)

1. Add the proposed ordinary metadata tables and internal-generation name validation; preserve and label the existing legacy table unknown.
2. Route embedding read/write entry points through identity checks and active generation lookup; test additive schema changes and mismatch failures on temporary fresh/legacy databases.

- **Depends on:** 11. **Goal/concept:** derived data has a schema and provenance.
- **Files:** `crm/db.py`, `crm/search.py`, `crm/graph.py`, `tests/test_embeddings.py`.
- **Scope:** store index model/revision/dimension/normalization/document version; refuse mismatched writes and queries with a rebuild instruction. Never drop an existing index automatically.
- **Acceptance:** legacy index is explicitly recognized as unknown/incompatible; equal dimension but wrong model is rejected; contact rows remain readable.
- **Verify:** fresh/legacy/matching/mismatched databases, equal-dimension mismatch, schema-write failure; no contact loss.
- **Graph change:** no new nodes; compatibility checks at index boundaries. **Learning question:** which metadata change requires re-embedding all contacts?

### 14 — Build a safe explicit index rebuild command (60–90 minutes)

Select one substep at a time; **14a → 14b → 14c**, each approximately 30–60 minutes. The total card can exceed one session.

1. **14a — Maintenance boundary:** implement a process-death-safe exclusive per-database application lock across capture/query/rebuild entry points; test busy rejection and release. Add explicit DB/model command arguments without rebuilding yet.
2. **14b — Staged generation:** enumerate rows under that lock, use shared document formatting, embed into a new generation, and validate ID coverage/identity. Inject mid-provider failure; old active pointer and contacts must remain intact.
3. **14c — Activation/recovery:** commit readiness and active pointer together in an ordinary SQLite transaction; test crashes before/after commit, first build without prior index, repeated rebuild and orphan-stage handling as specified above.

- **Depends on:** 13. **Goal/concept:** repair replaceable derived data without replaying business writes.
- **Files:** `crm/db.py`, `crm/cli.py`, `crm/search.py`, `crm/discord_bot.py`; **new** `crm/reindex.py`, `tests/test_reindex.py` if needed.
- **Scope:** rebuild vectors from contact rows into a staged index; validate counts/metadata before switching. Back up or preserve the old index on failure. No extraction/transcription or note append.
- **Acceptance:** explicit target DB/model, repeatable rebuild, all expected IDs indexed; interruption preserves the old active generation (or explicit unavailable index on first build); readiness/pointer change commit together with verified crash behavior. All application entry points respect maintenance exclusion.
- **Verify:** inject failure midway and at swap; compare original contact rows byte-for-byte or field-for-field; rerun yields same coverage with no duplicate notes.
- **Graph change:** capture unchanged; separate `read contacts → embed → validate staged index → swap` command.
- **Learning question:** why is rebuilding an index safer than replaying capture?

### 15 — Select and wire the measured runtime provider (60 minutes)

1. Promote the selected adapter/configuration from 12 through one shared construction function; update CLI, Discord, graph defaults and query to use identical identity/formatting.
2. Document install and explicit rebuild commands, test entry-point parity and missing-model errors, then run a separately opt-in learned-model smoke check on temporary contacts.

- **Depends on:** 12, 14. **Goal/concept:** configuration must preserve the experimental decision.
- **Files:** `crm/providers/`, `crm/cli.py`, `crm/discord_bot.py`, `crm/graph.py`, `README.md`, `pyproject.toml`.
- **Scope:** use the benchmark-selected provider in both capture and query, including Discord; explicitly configure hash baseline only as a diagnostic option. No silent fallback, no migration on startup.
- **Acceptance:** consistent identity/configuration across entry points, actionable missing-model/download errors, documented installation/rebuild path; key requirements reflect selected provider.
- **Verify:** adapter configuration parity and failures with fakes; offline suite; one opt-in local learned-model smoke test on temporary data.
- **Graph change:** same graph, provider substitution. **Learning question:** what breaks if capture and query use different models?

### 16 — Calibrate no-answer behavior (60 minutes)

1. Sweep a small documented distance-threshold range on development rankings; choose from false-positive rate versus relevant-query recall loss, not cosmetic demo behavior.
2. Add deterministic query filtering/empty-input handling, check CLI/Discord no-match formatting and report held-out results for the frozen model-specific threshold.

- **Depends on:** 15. **Goal/concept:** nearest does not mean relevant.
- **Files:** `crm/search.py`, CLI/Discord formatters, `crm/eval_retrieval.py`, relevant tests.
- **Scope:** evaluate a simple threshold for the selected normalized model on development labels; document it as model-specific. No answer generation or LLM relevance judge.
- **Acceptance:** empty/unrelated queries can abstain; report relevant-query recall loss and unanswerable false positives on held-out data; if no useful threshold exists, document failure rather than claim success.
- **Verify:** empty index/query, unrelated and borderline queries, top-k limits, model-metadata mismatch.
- **Graph change:** capture unchanged; query `rank → threshold → hits/no matches`. **Learning question:** why cannot vector distance be displayed as percentage confidence?

## Reliability and presentation

### 17 — Make partial index failure explicit (60 minutes)

1. Add the proposed separate write/index outcome fields, setting them at verification/index boundaries without adding retry nodes.
2. Update both adapter outputs with saved ID and reindex instructions; inject embedding/store errors after create and update, then verify recovery changes no contact notes.

- **Depends on:** 01, 14. **Goal/concept:** successful business writes and failed derived work can coexist.
- **Files:** `crm/state.py`, `crm/graph.py`, `crm/cli.py`, `crm/discord_bot.py`, `tests/test_embeddings.py`.
- **Scope:** report contact write success and index failure distinctly, with a reindex recovery instruction. Preserve existing failure status compatibility unless explicitly documented.
- **Acceptance:** no false “fully complete” result; both adapters show saved ID and failed indexing; recovery does not ask users to recapture.
- **Verify:** embed and vector-store failures after create/update, unchanged contact contents, successful reindex recovery.
- **Graph change:** existing nodes/status output clarified; no retry loop. **Learning question:** why is one success boolean insufficient here?

### 18 — Specify capture retry identity (45 minutes)

1. Trace CLI and Discord intake lifetimes; finalize the proposed request ID, canonical payload/hash and persisted record contract with worked replay/conflict/new-capture examples.
2. Draw the pre-provider replay check, one-transaction write boundary and post-commit verification/index path; identify precisely which existing store methods 19 must refactor.

- **Depends on:** 17. **Goal/concept:** duplicate contact matching is not request idempotency.
- **Files:** `crm/db.py`, `crm/discord_bot.py`, `crm/cli.py`; **new** a short retry decision record.
- **Scope:** choose stable request-ID ownership/lifetime for CLI and Discord, distinguish legitimate repeated notes from retries, define storage and failure transaction. Specification only.
- **Acceptance:** concrete same-ID/same-payload, same-ID/different-payload, and new-ID/same-card outcomes; minimal schema/transaction brief for 19.
- **Verify:** walk through timeout before/after commit, restart, and index failure; no ambiguous note behavior.
- **Graph change:** none yet. **Learning question:** why is hashing the note text alone an unsafe idempotency rule?

### 19 — Implement the selected retry contract (60–90 minutes)

1. Write duplicate/conflict/crash tests against 18's contract; add durable request lookup before providers and same-connection request/contact transaction logic.
2. Wire request IDs through the selected adapter/state fields and explicit replay route; verify restart replay acknowledges the stored write and checks current row existence without note append, never treating unestablished indexing/verification as full success.

- **Depends on:** 18 accepted. **Goal/concept:** repeat requests without duplicate side effects.
- **Files:** only files selected in 18; focused store/adapter tests.
- **Scope:** persist request identity with contact write using one transaction; do not add queues, generalized event sourcing, or autonomous retries.
- **Acceptance:** replay returns previous contact outcome without appending notes; mismatched payload for existing ID fails; new intentional capture remains possible; index recovery uses 14.
- **Verify:** duplicate delivery, conflicting request ID, transaction failure, restart replay; note counts and returned ID remain correct.
- **Graph change:** follow 18's explicit minimal route, document it before coding; split if more than one bounded transition is needed.
- **Learning question:** where must the request record commit relative to the contact write?

### 20 — Restrict Discord to an explicit owner (60 minutes)

1. Parse one configured owner/allowlist at startup and fail closed on absent/invalid configuration; add a small shared authorization check.
2. Apply it before every message/slash capture/query/download entry, then assert unauthorized handlers never call downstream dependencies or disclose data.

- **Depends on:** 01. **Goal/concept:** a DM channel does not provide data authorization.
- **Files:** `crm/discord_bot.py`, `tests/test_discord.py`, `how_to_run.md`.
- **Scope:** configured owner allowlist checked before attachment download, capture, or query in both message and slash paths. No multi-tenant database redesign.
- **Acceptance:** missing owner configuration fails closed; unauthorized users cannot retrieve or write contacts; authorized owner flow still works.
- **Verify:** unauthorized message/slash/download/query/capture calls are blocked; no leaked contact details; guild rejection remains intact.
- **Graph change:** graph unchanged; authorization at adapter entry. **Learning question:** why is pending state keyed by user ID insufficient for privacy?

### 21 — Bound attachment and sensitive-log retention (60–90 minutes)

Select one substep at a time; **21a → 21b → 21c**, each approximately 30–60 minutes.

1. **21a — File ownership:** record adapter-owned temporary paths in pending intake, clean them on completion/error/replacement, and test that arbitrary user paths are never removed.
2. **21b — Pending expiry:** choose/document a short configurable TTL, inject a clock and expire pending sessions on defined adapter activity; if periodic expiry is required, specify that trigger explicitly. Test exact boundary, replacement and cleanup behavior.
3. **21c — Logs/traces:** remove raw content from routine logs and test captured messages; document opt-in trace exposure and test local configuration without uploading data.

- **Depends on:** 20. **Goal/concept:** lifecycle ownership includes failure cleanup.
- **Files:** `crm/discord_bot.py`, `crm/tracing.py`, `tests/test_discord.py`, run guide.
- **Scope:** clean owned temporary attachments on success/failure/replacement and implement the documented pending-session expiry; remove raw DM/query text from routine logs; document trace opt-in data exposure. Substeps define separate learning sessions, not new card IDs.
- **Acceptance:** only adapter-owned files are removed; pending files remain while needed; failure paths clean up; no card content in routine logs.
- **Verify:** success/error/replaced pending/expired pending, unrelated-file preservation, captured log redaction; tracing behavior tested without uploads.
- **Graph change:** none. **Learning question:** who owns a temporary file while a capture waits for voice?

### 22 — Add transcription evidence separately (60–90 minutes plus labeling)

Select one substep at a time; **22a → 22b → 22c**, each approximately 30–60 minutes plus collection/live calls.

1. **22a — Labels:** adopt the voice manifest extension above, human-review consented reference transcripts and entity spellings, and freeze language/noise categories and split. Missing media keeps collection incomplete.
2. **22b — Scorer/runner:** define tokenization and empty-reference policy; implement hand-tested WER and an injected transcriber runner reusing result provenance/error conventions from 07.
3. **22c — Evidence:** execute only opt-in real calls, inspect entity mistakes separately from WER, recompute sanitized report aggregates and record unrun/failed cases.

- **Depends on:** 06, 07. **Goal/concept:** good card extraction does not prove good voice understanding.
- **Files:** `crm/providers/groq.py`, evaluation metrics/runner; **new** consented voice manifest and report.
- **Scope:** small labeled voice set, WER and business-entity error examples; frozen transcripts, duration/language/noise categories. The labeled substeps keep collection, code and evidence distinct.
- **Acceptance:** explicit empty-reference policy, failed calls counted, model/version recorded, privacy-reviewed report; no new summarizer.
- **Verify:** hand-computed WER including insertions/deletions/substitutions and empty reference; mocked provider failure; opt-in real run.
- **Graph change:** none. **Learning question:** why can low WER still misrepresent a critical company name?

### 23 — Package a reproducible portfolio demonstration (60 minutes)

1. Write the demo as exact commands against disposable synthetic data, showing a verified write, deliberate duplicate/update, relevant query and recoverable failure.
2. Rehearse with documented dependencies, record only when ready, inspect privacy and link actual reports/recording; label any missing quality or CI evidence plainly.

- **Depends on:** 03, 08, 15; may document outstanding evidence explicitly if not all complete.
- **Files:** `README.md`; **new** `docs/demo.md` and sanitized demo assets.
- **Scope:** script a 2–3 minute recording: capture, note, verified row, duplicate behavior, meaningful query, one failure, evidence report. Use synthetic/consented contacts.
- **Acceptance:** reproducible commands and expected outputs; real recording link only once recorded; honest implemented/planned labels; no fabricated CI badge or quality number.
- **Verify:** rehearse with a temporary database, follow links, inspect recording for credentials/private contacts.
- **Graph change:** none. **Learning question:** which part of the demo proves orchestration, and which proves model quality?

### 24 — Polish repository metadata and generated artifacts (45–60 minutes)

1. Inspect tracked generated files and current official GitHub metadata; preserve the existing `graphify-out/` ignore and required teaching history while drafting accurate description/topics and license options.
2. Apply only selected authorized presentation changes, verify regeneration/query and links, and record actual external results when metadata changes are in scope.

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
