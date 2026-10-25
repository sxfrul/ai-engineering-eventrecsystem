# V1 Architecture, Decisions & Traceability — UniNet Recommender
Version: 1.0

Diagram: [`week2_architecture_v1.png`](week2_architecture_v1.png) (editable source: [`week2_architecture_v1.drawio`](week2_architecture_v1.drawio))

## C1 — Simplest Architecture

```
Student opens feed → Application Interface → Ranking Capability → Ranked list of events
```

**Main user interaction:** a student opens the "For You" feed (or an event page, or the weekly email) and expects relevant events at the top.
**AI capability needed:** ranking a set of candidate events for one student.

The simple flow can produce *a* ranked list, but it cannot meet most of the requirements on its own:

- **FR2 / NFR3:** it has no way to understand event text, so new events cannot be matched.
- **FR3:** it has no memory of a student's interests, so nothing can decay.
- **NFR2:** it has no training data and no training step, so the weights stay hand-tuned.
- **FR4, FR7, RAI1, RAI3:** it has no controls for Not Interested, opt-out or diversity.
- **O3:** it has no point where quality can be measured.

Components are added below only where a requirement needs them.

## C2 — Architecture Needs from Requirements

| Requirement ID | What does the requirement need? | Architecture Response |
|----------------|---------------------------------|-----------------------|
| FR1 | A personalised ranking for each student, built from a shared set of live events. | **Candidate Generation** (shared, built nightly) + **Ranking Capability** (per student) |
| FR2 / NFR3 | Understanding of event titles and descriptions in Malay and English, including for events with no interactions. | **Event Catalogue** + **Text Embedding Capability** (multilingual) |
| FR3 | A record of each student's interests in which recent behaviour counts more. | **User Interest Profile** (aggregated, time-decayed) |
| NFR2 / NFR6 | Ranking weights learned from real behaviour, and replaceable without a client release. | **Interaction & Impression Log** + **Ranking Model Training** (offline), published as part of the shared model |
| FR4 / RAI4 | Capture of a student's rejections, and enforcement of them on every surface. | **Feedback Capture** (Not Interested) + **Exclusion Control** |
| FR1 / FR5 | Removal of registered, moderated and currently viewed events from results. | **Exclusion Control** |
| FR6 / RAI1 | A balance between relevance and variety in the top of the list. | **Diversity Re-ranking Control** |
| FR7 / RAI3 | Respect for the personalisation opt-out, with a fallback for new users. | **Consent Control** + non-personalised fallback ranking |
| Business constraint | Keeping the ranking boost promised to partner clubs, separate from what the model learns. | **Partner Boost Rule** |
| NFR1 / NFR4 / NFR5 | Fast ranking with no paid inference, and a shared model small enough to store and download. | Two-stage split: shared work done nightly, per-student ranking done **on the device** |
| O3 / NFR2 / RAI2 | A place to measure ranking quality, diversity, exposure, latency and cost. | **Offline Evaluation**, **A/B Experiment Assignment** and **Exposure, Latency & Cost Monitoring** points |

## C3 — V1 Conceptual Architecture

See [`week2_architecture_v1.png`](week2_architecture_v1.png). Components are labelled by function, using the Week 1 colour convention:

| Colour | Components |
|--------|------------|
| Grey — User interface | Student; Application Interface (For You feed, event page, weekly picks email) |
| Green — Data sources and pipeline | Event Catalogue; Interaction & Impression Log; User Interest Profile; Nightly Model Build (candidate generation) |
| Blue — Model and AI components | Text Embedding Capability; Ranking Model Training; Ranking Capability (on device) |
| Orange — Guardrails and controls | Consent Control; Exclusion Control; Diversity Re-ranking; Partner Boost Rule |
| Purple — Evaluation and monitoring | Offline Evaluation; A/B Experiment Assignment; Exposure, Latency & Cost Monitoring |

**Main flows**
1. **Ranking request.** The student opens the feed. The Application Interface loads the shared model package and the student's interest profile, after Consent Control has checked the opt-out. The Ranking Capability scores the candidates and passes them through Exclusion Control, the Partner Boost Rule and Diversity Re-ranking. The ranked list is shown to the student.
2. **Feedback.** Views, reads, register clicks, registrations, impressions and Not Interested choices go to the Interaction & Impression Log and update the User Interest Profile (with decay).
3. **Nightly build.** Candidate Generation reads the Event Catalogue (with embeddings from the Text Embedding Capability) and the log, and publishes the shared model package with the latest learned ranking weights.
4. **Training and evaluation.** Ranking Model Training learns weights offline from the log. Offline Evaluation compares the candidate model with the baseline before it is published. The A/B assignment and monitoring points watch the live system.

## C4 — Architecture Decision Records (ADR)

| Decision | Requirement / Constraint | Alternatives Considered | Selected Option | Trade-off Accepted |
|----------|--------------------------|-------------------------|-----------------|--------------------|
| **ADR1 — Where ranking runs** | NFR1 latency, NFR4 cost, RAI3 privacy, free-tier budget | (a) Server-side ranking function on every feed open; (b) per-student lists precomputed nightly; (c) **shared nightly candidate generation + on-device ranking** | (c) Keep UniNet's two-stage split: shared work once a night, per-student ranking in the browser | The shared model must stay small enough to store and download (NFR5), which limits model complexity and embedding size. The model is only as fresh as the last nightly build, so events posted later are merged on the device. |
| **ADR2 — How event content is represented** | FR2, NFR3, bilingual content, NFR4 (no paid APIs) | (a) Predefined tags only (current); (b) keyword matching such as TF-IDF; (c) **pre-trained multilingual text-embedding capability, self-hosted**; (d) paid external embedding API; (e) fine-tuned embedding model | (c) Generate an embedding when an event is published or edited, and backfill in the nightly build | A general-purpose model is not tuned to university events, so some matches will be imperfect. Embeddings must be compacted to fit the model size limit, and the nightly job takes longer. Fine-tuning is postponed until there is enough data. |
| **ADR3 — How ranking weights are set** | NFR2, NFR6, sparse training data | (a) Hand-tuned weights (current); (b) **learned linear ranking model**; (c) gradient-boosted learning-to-rank (LambdaMART); (d) deep neural ranker | (b) as the V1 deployed model, with (c) trained and compared offline. (c) is deployed only if it beats (b) on NDCG@10 and still meets NFR1 and NFR5 | A linear model captures fewer patterns than boosted trees. Learning needs impression logging for negative examples (see ADR4). With limited data the model may overfit, so it is regularised and the baseline weights are kept as a fallback. |
| **ADR4 — How training data is collected** | NFR2 needs event-level labels; RAI3 privacy; PDPA 2010; FR3 | (a) Keep only aggregated tag profiles (cannot train a ranker); (b) full event histories linked to accounts; (c) **pseudonymous interaction & impression log with limited retention** | (c) A pseudonymous ID per user, 90-day retention, and opted-out users excluded. Profiles used for live ranking stay aggregated (time-decayed interests). | Less history is available for training, and storage grows. Some re-identification risk remains, which is controlled by access rules and by never committing the data to this public repository. |
| **ADR5 — How the partner boost is applied** | Business promise to partner clubs; RAI2 transparency | (a) Add it as a feature and let the model learn it; (b) **separate business rule applied after model scoring** | (b) A Partner Boost Rule component after the Ranking Capability | Measured NDCG may be slightly lower than the pure model. To keep this visible, evaluation reports results both with and without the rule. |

## Initial Requirements Traceability Matrix

| Objective | Requirement | Evaluation Criterion | Architecture Element | Later Evidence |
|-----------|-------------|----------------------|----------------------|----------------|
| O1 | FR2 / NFR3 | Cold-start Hit Rate@10 on new events | Text Embedding Capability + Event Catalogue | Offline replay results on new events, compared with the baseline |
| O1 | FR3 | NDCG@10 on the latest week, with vs without decay | User Interest Profile (time-decayed) | Offline comparison of decay vs no decay |
| O1 | FR4 / RAI4 | Not Interested reappearance rate | Feedback Capture + Exclusion Control | Scripted 20-event rejection test log |
| O2 | NFR1 | p90 on-device ranking time | Ranking Capability (on device) | Browser timing report from 100 feed loads |
| O2 | NFR4 / NFR5 | Additional cost; model document size | Two-stage split (ADR1) + Nightly Model Build | 7-day usage dashboards; size test at double the catalogue |
| O2 | FR7 / RAI3 | Opted-out records used | Consent Control + Interaction & Impression Log | Training-data and ranking-input audit |
| O3 | NFR2 | NDCG@10, Precision@10; online CTR and registration rate | Ranking Model Training + Offline Evaluation + A/B Experiment Assignment | Offline evaluation report; A/B test results |
| O3 | RAI1 / RAI2 | Diversity, coverage, Malay-to-English exposure ratio | Diversity Re-ranking + Exposure, Latency & Cost Monitoring | Simulated-feed exposure analysis |

## Final Review Checklist

- **Does each major component support a requirement or evaluation need?** Yes. Every component in C2 maps to at least one FR, NFR, RAI or business constraint, and each row of the traceability matrix names its component.
- **Is any important data source, safety control or evaluation point missing?** The event catalogue, interaction log and interest profile cover the data. Consent, exclusion and diversity controls cover FR4, FR7 and RAI1–RAI4. Evaluation points cover every criterion. The one known gap is language labelling for RAI2. A language-detection step will be added once mixed Malay–English text is examined in the data.
- **Have I added any technology or component that is not yet justified?** No specific technologies are committed at V1. Gradient-boosted learning-to-rank is kept as an offline comparison only (ADR3), and the A/B assignment is used only "where feasible".
- **Is there a simpler architecture that could still meet the requirements?** A simpler design without the training pipeline (only adding embeddings and decay to the hand-tuned engine) would meet FR2 and FR3, but not NFR2 or NFR6. Dropping the impression log would remove the only source of negative examples for learning. Both are therefore kept.

**Reflection — which decision involved the most important trade-off?**
ADR4 (training-data collection) involved the most important trade-off. NFR2 needs event-level interaction and impression labels to learn ranking weights, but RAI3 and PDPA 2010 push towards storing as little as possible, which is why UniNet's current profiles hold only aggregated tag counts. The accepted compromise is a pseudonymous log with a 90-day limit that excludes opted-out users. It gives up some training history and adds some re-identification risk (controlled by access rules) in exchange for being able to learn ranking from real behaviour at all.
