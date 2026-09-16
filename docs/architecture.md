# Architecture and evidence guide

Reviewed against the local repository on 2026-09-14. Current implementation is described here; completed strengths and proposed implementation contracts are in [the development roadmap](development-roadmap.md).

## Assessment of the supplied review

I agree with the central assessment: this is a substantial end-to-end AI workflow with useful deterministic boundaries and regression coverage. Portfolio ratings are subjective; missing evidence matters more than a promised increase from 8 to 9.

| Review claim | Assessment from local evidence |
| --- | --- |
| End-to-end capture, notes, storage, verification, retrieval, tracing, Discord | Agree: implemented in `crm/graph.py`, `crm/search.py`, `crm/tracing.py`, `crm/discord_bot.py` |
| Strong separation of AI and deterministic software | Agree: injectable provider protocols; graph/normalization/store own writes and routing |
| 102 tests pass | Confirmed offline; the two live tests were deselected |
| README is behind code | Agree for the previous README; updated in this task |
| Default embedding is not semantic | Agree: hashes and vocabulary-overlap tests cannot establish synonym understanding |
| Perfect fake evaluation scores do not measure real AI quality | Agree: `crm/eval.py` uses `tests.helpers` fakes and placeholder media |
| No CI or license | Neither a workflow configuration nor a license file was found in this checkout |
| GitHub description/topics/HTML language share need polish | Public GitHub metadata was not verified here; generated Graphify HTML exists locally. Inspect metadata before claiming or changing it |
| Learning artifacts need cleanup | Qualify: the tutoring harness requires learning history. Organize generated output intentionally without deleting that history |
| Replace hashes with Sentence Transformers first | Directionally agree, but establish labels, provider contract, and index compatibility before switching the runtime |

## Boundaries and state

| Boundary | Files | Responsibility |
| --- | --- | --- |
| Adapters | `crm/cli.py`, `crm/discord_bot.py` | Input conversion and output formatting; queries bypass capture |
| Orchestration | `crm/graph.py`, `crm/state.py` | Nodes, conditional edges, status/error propagation |
| Probabilistic interpretation | `crm/providers/groq.py`, `crm/providers/base.py` | Image → dictionary; audio → transcript; wrapped provider errors |
| Deterministic validation | `crm/schemas.py`, `crm/normalize.py` | Nonblank full name, optional blanks, matching forms |
| Persistence | `crm/db.py` | SQLite match/create/update/get; authoritative contact rows |
| Index/retrieval | `crm/providers/embeddings.py`, `crm/search.py`, `crm/db.py` | Document/vector creation, replacement, nearest neighbors, contact hydration |
| Evaluation/observability | `crm/eval.py`, `crm/tracing.py`, `eval/`, `tests/` | Regression scores and optional external traces/results |

The model never decides CREATE versus UPDATE and never writes SQL. Voice notes remain context rather than overwriting card identity. Pydantic validates shape, not whether a field was visible on the card. `CRMState` is a `TypedDict`, not a runtime Pydantic validation boundary.

State evolves through inputs → raw `extracted_card` → `contact_evidence` → separate `voice_transcript`/`conversation_notes` → `normalized_contact` → match/write IDs → `verified_contact` → search document/vector. `load_input` resets intermediates. After extraction validation, success remains `valid` until finalization changes it to `complete`; errors remain errors. No durable checkpointing or autonomous planning loop is configured.

Follow the [README graph](../README.md#workflow) alongside `build_graph`. Invalid input still visits guarded extraction nodes, then skips voice/persistence. Search errors route directly to finalization. Write errors traverse a guarded verifier then the `write_failed` edge. Embedding errors traverse a guarded store node then finalization. Explain both routing and guards when teaching the execution model.

## Data and failure semantics

`contacts` stores `id`, seven contact fields (`full_name`, `company`, `job_title`, `email`, `phone`, `website`, `address`), notes, source, and timestamps. `contact_embeddings` is a `vec0` virtual table containing vectors with rowid equal to contact ID. Search documents contain only full name, company, job title, and notes. Vectors are derived data; contacts are the source of truth.

Matching scans rows ordered by ID. Email, when present, is the exclusive key; otherwise phone; otherwise name + company. A changed email with the same phone can therefore create a new contact. Conflicting keys are not reconciled and the first match wins. Incoming nonblank fields replace existing values; blank values preserve old fields; notes append. This is not conflict review or concurrency-safe identity resolution.

Write verification compares nonblank intended contact fields and checks submitted notes occur within stored notes. It does not prove exact whole-row equality, audit all old values, or verify card truth. Contact writes commit before indexing. An embedding failure can return `error` after a contact was saved, with a missing/stale vector. Retrying capture can append identical notes again. Recovery should work from the saved row instead of repeating AI interpretation and writes.

SQLite uses `sqlite3` when extension loading is available, with an APSW compatibility wrapper otherwise. PostgreSQL/pgvector are not implemented; do not introduce them just for portfolio size.

## Embedding mechanics

`GroqEmbedder` advertises 768 dimensions. By default it lowercases/splits text, adds MD5-derived token counts, and L2-normalizes. Its first two digest bytes only address bins 0–255, so the nominal 768 coordinates are not fully used. This is a lexical baseline with collisions, not a trained representation.

`CRM_EMBED_MODEL` attempts `Groq(...).embeddings.create`; selected missing-model errors warn then fall back to hashing. No model/revision/document-version metadata accompanies the vectors. Equal dimensions do not imply compatible vector spaces, and remote output length may disagree with the advertised dimension. This review does not establish a live supported embedding configuration.

Vector distance is not confidence or probability. For unit vectors, squared Euclidean distance equals `2 - 2*cosine_similarity`; the remote branch does not enforce normalization. Queries return nearest rows without a relevance threshold, potentially including unrelated contacts.

Compare the hash baseline with candidates such as `all-MiniLM-L6-v2` and `multi-qa-MiniLM-L6-cos-v1`; neither is selected yet. Measure query/contact relevance, language coverage, query/document asymmetry, truncation, normalization, dimensions, license, latency, download size, and local resource use on the same corpus and hardware. See the [official Sentence Transformers model guide](https://sbert.net/docs/sentence_transformer/pretrained_models.html). Freeze development and held-out splits before tuning.

## Evaluation definitions

Offline tests ask whether the workflow behaves correctly. Real labeled media/query sets ask whether interpretation and retrieval are useful. Generated-answer faithfulness applies only after answer generation exists; today's contact lookup is not a RAG answer pipeline.

Current `extraction_correctness` averages exact equality over reference keys, including nulls. Correct absent fields can hide weak populated-field extraction. `unsupported_field_hallucination` is actually expected-null preservation, so 1.0 is good; missing output can score well on this metric alone. Pair it with extraction success and completeness. CREATE/UPDATE and duplicate scores use seeded fake scenarios. The committed weakest-case explanation explicitly acknowledges perfect fake scores. No committed score establishes real transcription accuracy.

| Proposed measurement | Definition |
| --- | --- |
| Extraction accuracy | Normalized exact match per populated field; report whole-card success separately; failed calls stay in denominators |
| Unsupported-field rate | Unsupported nonnull predictions / annotated absent-field opportunities; lower is better; pair with coverage and error rate |
| Transcription | Word error rate `(substitutions + deletions + insertions) / reference words`; define empty-reference handling; report business-entity errors |
| Precision@k | Relevant unique IDs in first k / fixed k; missing slots count as nonrelevant |
| Recall@k | Relevant unique IDs in first k / all labeled relevant IDs for the query |
| MRR@k | Reciprocal rank of first relevant result within k, or zero; mean across answerable queries |
| No-answer behavior | Separate unanswerable-query abstention success and false-positive retrieval rate; no recall division by zero |
| Context relevance | Human relevance labels for returned contact text against the query; report categories and failures |
| Future generated answers | Claim support in retrieved evidence, citation correctness, answer correctness/completeness, unsupported-claim rate; audit any LLM judge against human labels |

Record counts, dataset/split version, model/revision, prompt/document version, environment, latency, and observed costs. Ten to twenty consented cards provide starter evidence, not broad production validity. Include glare, blur, missing fields, varied layouts, and difficult cases. Never tune on the held-out set and describe it as unseen evidence.

## Workflow understanding and deployment boundary

This is a predefined workflow: models interpret data within controlled transitions. An autonomous agent typically lets a model choose actions/tools or the next step dynamically. More autonomy is not required here. Learn state, nodes, conditional routes, termination, guarded errors, and side effects first. See the official [workflows versus agents guide](https://docs.langchain.com/oss/python/langgraph/workflows-agents) and [graph API concepts](https://docs.langchain.com/oss/python/langgraph/graph-api). The application uses provider SDKs directly, not a LangChain application abstraction.

Discord pending intake is keyed by user ID, but persisted contacts and queries share a database without ownership filters. Pending state is lost on restart. Downloaded temporary attachments lack a cleanup lifecycle, and logs/traces may expose personal information. The roadmap separates a single-owner access boundary, retention, and recovery into bounded tasks; no multi-user service is claimed.
