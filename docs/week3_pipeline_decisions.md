# Week 3 — V1 Data Pipeline Decisions, Stage Contracts & Traceability — UniNet Recommender
Version: 1.0

Notebook: [`notebooks/week3_pipeline.ipynb`](../notebooks/week3_pipeline.ipynb) · Diagram: [`week3_pipeline_diagram.png`](week3_pipeline_diagram.png) (source: [`week3_pipeline_diagram.drawio`](week3_pipeline_diagram.drawio)) · Inventory: [`week3_data_inventory.md`](week3_data_inventory.md)

## 1. Summary

| Item | Decision |
|------|----------|
| **Topic 2 requirement IDs** | **FR2** (semantic text for cold-start ranking), **RAI3** (no personal identifiers), **RAI2** (club and language exposure evidence) |
| **V1 architecture component** | *Event Catalogue* (green), which feeds the *Text Embedding Capability* (Week 5) and the *Nightly Model Build* |
| **Selected source** | S01: UniNet `events` collection, read through the public Firestore REST list endpoint and frozen as one JSON snapshot (63 events, 2026-11-16T01:08Z) |
| **Pathway** | **D — JSON / API export**. This is a single-modality project, so no other pathway is run. |
| **Chosen configuration** | Decode Firestore typed values → flatten the Lexical description (same walk as UniNet's `lexical.ts`) → mask phone numbers and emails → keep only public fields → exclude unlisted or report-hidden events → sort by `created_at`, `event_id` |
| **Output location** | Part B: `data/processed/prepared_json.jsonl` and `data/week3_manifest.json`. Part C: `data/pipeline_v1/json/prepared_json.jsonl` and `pipeline_manifest.json`. The raw snapshot stays in `data/raw/` and is git-ignored. |
| **ETL / ELT** | **ETL** |
| **Batch / streaming** | **Batch** (snapshot) |

## 2. Part B comparison and evidence (Step B3)

Both configurations were run on the same snapshot.

| Approach / Configuration A | Approach / Configuration B (selected) | Observed evidence | Selected approach and trade-off |
|----------------------------|---------------------------------------|-------------------|---------------------------------|
| **Decode only.** Firestore wrappers are removed, every field is kept as stored, and the description stays as Lexical JSON. | **Decode + flatten + mask + whitelist + exclude** | Records: 63 vs 63. Median description length: 4,180 characters of Lexical JSON vs 847 characters of readable text. Embedding-ready text: no vs yes. Records with personal fields: 63 vs 0. Records with a phone number or email in the text: 5 vs 0. Size: 421.1 KB vs 180.4 KB. Example: A starts `{"root":{"children":[{"children":[{"detail":0,…`, while B starts `Hey everyone! 👋 Here's an opportunity for students…` | **B.** *Benefit:* every event gets readable, embedding-ready text (FR2), and nothing committed or used for training carries a personal identifier or contact detail (RAI3). *Accepted limitation:* flattening drops paragraph and list structure and emphasis, and the poster image (which sometimes holds the event date) is not used. This is acceptable because FR2 embeds short event text. It will be revisited if retrieval quality in Week 5 is poor. |

**Settings kept deliberately unchanged:** `type` and `category` values are stored as they are, including `none` / `All Events`. Empty deadlines become `null` and are not imputed. Comment counts stay `null` where they are missing. No language label is invented. Each of these is listed for Topic 4 in the inventory.

**Exclusion rule (FR1):** the pipeline drops `visibility == "unlisted"` events and events with 5 or more reports, the same threshold the platform uses. No event matched either rule in this snapshot, so 0 were excluded. The rule stays because the catalogue will be re-snapshotted.

## 3. ETL vs ELT

**ETL: transform before the prepared data is stored.** The raw documents contain user IDs, personal names and photos, reporter IDs and contact details. RAI3 and the README's rule against committing user data mean those fields must be gone **before** anything reaches a shared or persistent store (GitHub now, and the nightly model package later). An ELT design would load the raw documents first and transform them inside the store, which would put identifiers exactly where they must not be. The only untransformed copy is the local or Colab landing file in `data/raw/`, and it is never committed.

## 4. Batch vs streaming

**Batch.** The catalogue is small (63 documents in a single 300-document page) and changes at human pace, with a handful of events posted per day. The V1 architecture already does shared work once a night (ADR1, Nightly Model Build). A frozen snapshot also makes reruns repeatable, because Cell 9 compares SHA-256 fingerprints. Streaming would add moving parts and no benefit at this scale. ADR2's "embed on publish" trigger is a later, event-driven addition on top of the nightly batch, not a replacement for it.

## 5. Stage contracts

| Stage | Contract / evidence | Failure response |
|-------|---------------------|------------------|
| **Ingest** | The endpoint returns HTTP 200 on every page, `documents` is a list, pagination finishes within `MAX_PAGES`, and there is at least one document. The access basis (public read) is recorded in the snapshot header. | **Stop.** Raise an error and leave the previous snapshot untouched, because it is written to a `.tmp` file and only replaces the old one after the whole download succeeds (**F1**). |
| **Inspect** | The source parses as JSON, the `documents` key is present, the required envelope keys (`name`, `fields`) and fields (`title`, `description`, `type`, `createdAt`) are present, and every Firestore value type is known | **Flag or stop.** Records missing required keys are counted and excluded. An unknown Firestore value type means the schema changed, so the run stops rather than guessing. |
| **Prepare** | Every kept record has a non-empty `embedding_text`, the Lexical flattening matches UniNet's `lexical.ts`, and the exclusion rule is applied | **Flag or stop.** If a description flattens to nothing, the record is kept with `description_empty = true` and `embedding_text` falls back to the title, so the event stays rankable (FR2). If no records remain, the run stops. |
| **Validate** | The output exists and is non-empty, the read-back equals the prepared records, there are 0 personal fields, 0 raw user IDs and 0 phone or email patterns in the output, and SHA-256 values are recorded | **Stop.** Missing or empty output fails the run. A personal identifier in the output stops the export before it is written (**F2**). |
| **Record lineage** | The source links to the exact output with matching SHA-256 values in `week3_manifest.json` (Cell 6B re-verifies them). The run manifest records the configuration, the inspection and relative paths. The rerun matches, and the Part B and Part C outputs are byte-identical (Cell 9). | **Stop and report** a missing mapping, a checksum mismatch or a repeatability difference |

## 6. Failure points shown in the diagram

| ID | Where | Plausible failure | Response |
|----|-------|-------------------|----------|
| **F1** | Ingest | The Firestore endpoint is unreachable, returns an error or quota response, or pagination is cut off. The result would be an empty or partial catalogue that silently drops new events, which is exactly the FR2 cold-start case. | Stop the run, keep the last good snapshot, and record the HTTP status. Do not publish a partial catalogue. |
| **F2** | Prepare / Validate | The platform adds a new field (for example an attendee list) or changes the description format. A personal identifier would then leak into the prepared data, or descriptions would flatten to empty text. | Allow only whitelisted output fields (new fields are dropped by default). Scan the output for raw user IDs and contact patterns, and stop the export if any are found. Flag empty-text records instead of dropping them. |

## 7. Provenance and access caveats

- **Version:** the snapshot `fetched_at_utc` value, plus each record's Firestore `updateTime`. A Colab run fetches a new snapshot, so its SHA-256 values will differ from the committed ones. That is expected, and the manifests record which snapshot produced which output.
- **Access:** the data is read through the platform's own public-read rule with no credentials. The raw snapshot is used only for this project and is not redistributed. The prepared file contains only fields that are shown on the public event pages, and contact details are masked.
- **What the checks do not prove:** a SHA-256 match shows that files are unchanged. It says nothing about data quality, representativeness or legal permission. Quality and fitness for purpose are reviewed in Topic 4.
- **Practice data:** the dummy dataset pack was **not** used. All results come from the real UniNet catalogue.

## 8. Traceability update

| Objective | Requirement | Architecture element (Week 2) | Week 3 evidence |
|-----------|-------------|-------------------------------|-----------------|
| O1 | FR2 / NFR3 | Event Catalogue → Text Embedding Capability | `embedding_text` for 63/63 events in `data/processed/prepared_json.jsonl`. B3 comparison (Lexical vs flattened text). |
| O2 | RAI3 | Event Catalogue (and later Interaction & Impression Log) | B3 contracts: 0 personal fields, 0 raw user IDs, 0 contact patterns. Raw snapshot is git-ignored. |
| O3 | RAI2 | Exposure, Latency & Cost Monitoring | Baseline catalogue mix in the inventory: 32/63 events under one club, 20 member-hosted, 5/63 with Malay markers (heuristic) |
| O3 | NFR2 / NFR6 | Nightly Model Build | Repeatable V1 pipeline: `data/pipeline_v1/json/pipeline_manifest.json`, Cell 9 PASS |

**Next steps:** Topic 4 runs a data-quality and RAI review of P01 (language detection, exposure baseline, uncategorised events). Week 4 needs the interaction & impression log (S02) for the baseline offline evaluation. Week 5 embeds `embedding_text` with the multilingual model (ADR2).
