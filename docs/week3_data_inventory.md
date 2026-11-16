# Week 3 — Data Inventory — UniNet Recommender
Version: 1.0 · Snapshot taken 2026-11-16T01:08:00Z

Notebook: [`notebooks/week3_pipeline.ipynb`](../notebooks/week3_pipeline.ipynb) · Lineage: [`data/week3_manifest.json`](../data/week3_manifest.json) · Decisions: [`week3_pipeline_decisions.md`](week3_pipeline_decisions.md)

## 1. Requirement → data need (Step A1)

| Requirement ID | Needed information / feature | Candidate source | Why usable / constraint |
|----------------|------------------------------|------------------|-------------------------|
| **FR2** — rank new events by the meaning of their title and description, in Malay and English | Title and readable description text for every listed event, including events with no interactions | UniNet `events` collection (S01) | Authoritative: it is the live catalogue that the feed ranks. It covers every listed event. Public read is allowed by the platform's Firestore rules. Descriptions are stored as Lexical editor JSON and have to be flattened before they can be embedded. |
| **RAI3** — no personal identifiers in profiles or training data | Event records **without** poster user ID, personal name, photo, reporter IDs or contact details | Same source; personal fields removed during preparation | Every raw document carries `userId`, `organizer`, `organizerPhoto` and `reportedBy`, and some descriptions include a phone number or email. Because of that, the raw snapshot is never committed. |
| **RAI2** — exposure not concentrated on popular clubs or English-language events | Club identity, club vs member host, type/category, language evidence | Same source (`clubId`, `clubName`, `advancedFilters`) | Club and type fields exist. Language is not stored, so it is only estimated during inspection, and proper language detection is carried to Topic 4. |

This uses a single dataset. The lab allows one dataset if it is justified, and here the Event Catalogue is the one V1 data component that FR2 depends on and that exists today. Inspection used two subsets of it (see S01-a and S01-b below).

## 2. Inventory

| Source ID | Filename / URL | Type and purpose | Version / date / owner | Access / rights | Inspection evidence |
|-----------|----------------|------------------|------------------------|-----------------|---------------------|
| **S01** | Raw: `data/raw/uninet_events/events_snapshot.json` (git-ignored). Endpoint: `https://firestore.googleapis.com/v1/projects/uninet-477707/databases/(default)/documents/events` | Semi-structured JSON (Firestore REST). This is the event catalogue that feeds FR2 text embeddings and the nightly candidate build. | Snapshot fetched 2026-11-16T01:08:00Z: 63 documents in 1 page. Each record keeps its own Firestore `updateTime`. Owner: UniNet platform (project developer). | Public read (`firestore.rules`: `allow read: if true`), with no key or login used. The raw file contains personal identifiers, so it stays local or in Colab and is **not committed**. It is not redistributed. | Keys: `documents[] → {name, fields, createTime, updateTime}`. Values are wrapped in typed keys (`stringValue`, `timestampValue`, `arrayValue`, `mapValue`). Category, flairs, universities and deadline are nested in `advancedFilters`. Required fields are present on 63/63. 63/63 descriptions are Lexical JSON (median 4,180 characters raw). |
| **S01-a** (subset 1) | S01, documents that have `clubId` | Club-hosted events. Used to check the club fields that RAI2 and FR6 rely on. | Same snapshot | Same | 43 events across only 4 clubs. 32 of them are under the platform's own "UniNet" club, which is a concentration signal for RAI2. `clubName` and `clubVerified` are present on all 43. |
| **S01-b** (subset 2) | S01, documents without `clubId` | Member-hosted events posted by individuals | Same snapshot | Same | 20 events. `organizer` is a **person's name** here, which is why it is dropped. They have no club fields, so `host_kind = "member"`. |
| **P01** (prepared) | `data/processed/prepared_json.jsonl` (Part B) and `data/pipeline_v1/json/prepared_json.jsonl` (Part C). The two files are byte-identical. | JSONL, one public event per line, with `embedding_text` ready for the Week 5 embedding step | SHA-256 recorded in both manifests. Rebuilt from S01 by the notebook. | Identifier-free and committed. Every field also appears on the public event page. Phone numbers and emails are masked. | 63 records. Checks found 0 personal fields, 0 of 12 raw user IDs and 0 phone or email patterns. Median description is 847 characters of readable text. |
| S02 (not ingested) | UniNet interaction & impression log | Implicit feedback labels for NFR2 learned ranking | Does not exist yet: impression logging is introduced by ADR4 | Private and pseudonymous, never committed | Deferred. It is needed from Week 4 (baseline) and Week 7 (learning to rank). |
| S03 (not ingested) | MIND — https://msnews.github.io/ | Public benchmark for supplementary offline evaluation | Microsoft Research licence | Licence to be checked before use | Deferred. It is not needed by FR2, RAI2 or RAI3 this week. |

## 3. Raw vs prepared storage

| Layer | Location | In GitHub? | Why |
|-------|----------|------------|-----|
| Raw | `data/raw/uninet_events/events_snapshot.json` (local), or `/content/week3_project/data/raw/...` in Colab | **No** (`.gitignore`) | Contains user IDs, personal names and photos, reporter IDs and contact details. README and RAI3 forbid committing user data. Re-create it by running Cell 2. |
| Prepared (Part B) | `data/processed/prepared_json.jsonl` | Yes | Identifier-free public event fields |
| Part B lineage | `data/week3_manifest.json` | Yes | Source and output SHA-256, processing note, snapshot version |
| Part C run | `data/pipeline_v1/json/` (`prepared_json.jsonl`, `pipeline_manifest.json`) | Yes | Executed pipeline output and run manifest with relative paths only |

## 4. Observations carried to Topic 4 (data quality and RAI review)

1. **Language is unlabelled (RAI2).** A word-list heuristic flags only 5/63 events with ≥ 3 Malay marker words, and mixed Malay–English text is common. A real language-detection step is needed before the Malay-to-English exposure ratio can be measured.
2. **Exposure concentration (RAI2).** About half of the catalogue (32/63) is posted under one club ("UniNet"), and 20 events have no club at all. Catalogue coverage and per-club exposure need to be reported against this baseline mix.
3. **Uncategorised events.** 3 events have `type = "none"` and `category = "All Events"`. They are kept as stored, not relabelled.
4. **Missing deadlines.** 4 events have an empty `registrationDeadline`, now stored as `null`. Nothing was imputed.
5. **Audience university unused.** No event sets `universities`, so the "campus" signal in FR7 cannot come from this field yet.
6. **Short history.** Events expire about 30 days after posting (oldest `createdAt` 2026-10-16), and one event was already past `expiresAt` at snapshot time. Offline replay for NFR2 and NFR3 needs snapshots taken over time or the interaction log, not a single catalogue snapshot.
7. **Event date is not a field.** The event's own date appears only in the description prose or on the poster. Freshness can only use `created_at`, `expires_at` and `registration_deadline`.
