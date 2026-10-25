# Requirements Specification — UniNet Recommender: Enhanced Hybrid Event Recommendation
Version: 1.0

## Section 0 — Project Framing & Application Scope Checkpoint

| Framing element | Details |
|-----------------|---------|
| **Problem Evidence** | **1. Event cold start.** Events are short-lived and usually have little or no feedback before they happen, so collaborative filtering alone cannot rank them well (Macedo, Marinho & Santos, 2015, ACM RecSys, https://doi.org/10.1145/2792838.2800187).<br>**2. Interest drift.** User preferences change over time, and modelling this improved recommendation accuracy on the Netflix Prize dataset (Koren, 2009, ACM KDD, https://doi.org/10.1145/1557019.1557072).<br>**3. Observable evidence in UniNet.** The current engine scores events with about ten hand-tuned weights, describes events using only 4 categories and 14 flairs, keeps interest profiles that never decay, and gives users no way to reject a recommendation (UniNet source code, `netlify/shared/recommend.ts`). |
| **Existing Approaches** | **1. Item-to-item collaborative filtering** (Linden, Smith & York, 2003): recommends items that are often interacted with together, and scales well. It is already the basis of UniNet's "similar events" signal.<br>**2. Two-stage deep recommender** (Covington, Adams & Sargin, 2016): separates candidate generation from ranking and learns the ranker from implicit feedback. UniNet already follows this two-stage structure.<br>**3. Context-aware event recommendation** (Macedo et al., 2015): combines content, social, location and time signals with learning-to-rank for Meetup events, and addresses the new-event problem.<br>**4. UniNet's current engine:** a working hybrid of tag-based content matching, item-to-item CF on co-registrations, and context signals (university, club, popularity, deadline, freshness). |
| **Project / Engineering Gap** | Item-to-item CF cannot rank new events, the YouTube approach needs data and infrastructure far beyond a university platform, and the Meetup study relies on English content and social group signals. UniNet's own engine has the same cold-start weakness, and its weights are set by hand, matching is tag-only, profiles never decay and recommendation quality is never measured. The practical gap is a **lightweight, data-driven and measurable** ranking approach for a small, bilingual (Malay and English) university platform with short-lived events. This is a practical engineering gap, not a claim of scientific novelty. |
| **Objective O1** | To design an enhanced hybrid recommendation engine for UniNet that adds semantic event matching, time-decayed interest profiles, learned ranking, explicit negative feedback and diversity-aware re-ranking to the existing two-stage architecture. |
| **Objective O2** | To develop and integrate the enhanced engine into UniNet's "For You" feed, "More like this" section and weekly picks email, within the platform's existing latency, cost and privacy constraints. |
| **Objective O3** | To evaluate the enhanced engine against the current engine using offline ranking metrics (Precision@K, NDCG@K, coverage, diversity) and, where feasible, an online A/B test of click-through and registration rates. |
| **Intended Users** | **Primary:** university students in Malaysia who use UniNet to discover and register for technical events.<br>**Secondary:** event organisers and university clubs, who benefit from better exposure of new events.<br>**Operator:** the UniNet developer/administrator, who trains, evaluates and monitors the engine. |
| **Supported Tasks** | 1. Rank open events in a signed-in student's "For You" feed.<br>2. Show related events ("More like this") on an event page.<br>3. Select events for the weekly picks email sent to inactive users.<br>4. Let a student mark a recommended event as "Not Interested".<br>5. Provide non-personalised recommendations to new users and users who turned personalisation off. |
| **Information / Knowledge Domain** | Technical university events on UniNet (programming competitions, workshops, talks, meetups) described by title, description (Malay and/or English), category, flairs, audience university, organising club and registration deadline, together with implicit interaction signals (views, scroll depth, register clicks, completed registrations). |
| **Operating Context** | A live, responsive web application used on phones and desktop browsers. A nightly serverless job builds the shared model, and each user's browser ranks the candidate pool when they open the feed. Users are mainly in Malaysia (UTC+8). |
| **Important Constraints** | **Data:** a small user base with sparse interactions, so models must work with limited training data.<br>**Technical:** serverless function time limits, a 1 MB limit on the Firestore document that holds the shared model, and no GPU.<br>**Budget:** free or low-cost tiers of Firebase, Netlify and Upstash Redis, so no paid model APIs.<br>**Policy:** Malaysia's PDPA 2010 and the platform's existing personalisation opt-out. User data must not be committed to this public repository.<br>**Business rule:** partner (collaborator) clubs are promised a ranking boost, which must be kept.<br>**Time:** one semester. |
| **Explicit Out-of-Scope Tasks** | The system shall not register users for events or decide anything on their behalf, approve or moderate events, recommend people or clubs, or rank non-event content. It shall not use sensitive personal attributes (gender, race, religion, age) as ranking features. Real-time model retraining and native mobile apps are also out of scope. |
| **Evaluation Focus** | Does the enhanced engine rank the events students actually engage with higher than the current engine (O3), give new events fair exposure, and stay diverse, while meeting UniNet's latency, cost and privacy constraints? |

**Traceability check:** O1 leads to FR2–FR4 and FR6 and to the architecture decisions. O2 leads to FR1, FR5, FR7, NFR1, NFR4 and NFR5. O3 leads to NFR2, NFR3 and RAI1–RAI2, and to the evaluation criteria in `week2_evaluation_criteria.md`.

---

## Section 1 — Functional Requirements

| # | Requirement — The system SHALL | Scope Boundary — SHALL NOT (where relevant) |
|---|-------------------------------|---------------------------------------------|
| FR1 | Generate a personalised, ranked list of events in the "For You" feed for each signed-in student, based on their onboarding preferences and interaction history. | Include events the student has already registered for, or events that have been unlisted or removed through moderation. |
| FR2 | Rank newly published events that have no interaction history using the meaning of their title and description, in both Malay and English. | Exclude an event from recommendations only because it has no interactions yet. |
| FR3 | Give a student's recent interactions more influence on their interest profile than older interactions. | Store a student's individual event-by-event browsing history in order to build the profile. |
| FR4 | Allow a student to mark a recommended event as "Not Interested", remove that event from their recommendations, and lower the ranking of similar events. | Hide the event from search, the "Recent" tab or other non-personalised discovery pages. |
| FR5 | Show up to three related events ("More like this") on each event page. | Show the event currently being viewed, or events the student has already registered for. |
| FR6 | Spread the top of the feed across different clubs and event types, so that no single club or type fills a run of consecutive positions. | Remove relevant events from the feed entirely in order to meet the diversity goal. |
| FR7 | Provide non-personalised recommendations (popularity, freshness and campus) to new users and to users who have turned personalisation off. | Use the behavioural data of users who have turned personalisation off, for ranking or for model training. |

---

## Section 2 — Non-Functional Requirements

| # | Quality Attribute | Requirement | Measurement Method | Metric | Threshold |
|---|-------------------|-------------|--------------------|--------|-----------|
| NFR1 | Performance | On-device ranking of the candidate pool shall not noticeably delay the feed. | Time the ranking step in the browser over 100 feed loads on a mid-range phone and a laptop. | p90 ranking time | < 100 ms |
| NFR2 | Correctness (ranking quality) | The enhanced engine shall rank the events students go on to engage with higher than the current engine. | Offline replay on a time-based hold-out set (the most recent weeks of interactions), comparing the enhanced engine with the current engine. | NDCG@10, Precision@10 | NDCG@10 ≥ 10% relative improvement over the baseline; Precision@10 not lower than the baseline |
| NFR3 | Correctness (cold start) | Newly published events shall reach the students who later engage with them. | Offline replay restricted to hold-out interactions on events less than 7 days old with no prior registrations. | Hit Rate@10 on new events | ≥ 15% relative improvement over the baseline |
| NFR4 | Cost | The enhanced engine shall run within the platform's existing free or low-cost cloud tiers, with no paid model API. | Record Firestore reads, function run time and Redis commands over a 7-day run, and compare with the current engine. | Additional monthly cost; Firestore reads per feed open | RM 0 additional cost; reads per feed open ≤ current engine + 1 |
| NFR5 | Scalability | The shared model shall still fit its storage and processing limits when the event catalogue doubles. | Build the model with a synthetic catalogue twice the current live size, and time the nightly job. | Model document size; nightly job duration | < 800 KB (20% headroom under 1 MB); < 50% of the function time limit |
| NFR6 | Maintainability | The ranking model shall be retrainable and replaceable without changing the client application. | Retrain the model on new data and deploy the new version, then confirm the feed uses it without a client release. | Client code changes per model update; time to deploy a retrained model | 0 client changes; < 15 min |

*Thresholds for NFR2 and NFR3 are initial targets. They will be confirmed once the baseline is measured on real UniNet data.*

---

## Section 3 — Responsible AI Requirements

| # | Responsible AI Risk | Requirement | Measurement Method | Metric | Threshold |
|---|---------------------|-------------|--------------------|--------|-----------|
| RAI1 | Filter bubble / lack of diversity | Each student's feed shall include a variety of event types and clubs, not only the kind they engaged with before. | Generate top-10 feeds offline for all active users, and compare the enhanced engine with the baseline. | Intra-list diversity (average pairwise semantic distance within the top 10); share of users whose top 10 contains ≥ 3 event types | Diversity ≥ baseline; ≥ 90% of users |
| RAI2 | Fairness / exposure bias | Exposure shall not be concentrated on popular clubs or on English-language events, and the ranking shall not use sensitive personal attributes. | Count top-10 impressions per event, club and language across simulated feeds. Audit the list of ranking features. | Catalogue coverage (% of open events shown to at least one user); Malay-to-English exposure ratio (relative to catalogue share); sensitive features used | Coverage ≥ 80%; exposure ratio ≥ 0.8; 0 sensitive features |
| RAI3 | Privacy | Profiles shall be stored as aggregated interests, not event histories, and data from users who turned personalisation off shall not be used for training. | Inspect the stored profile schema. Check the training dataset against the list of opted-out users and for personal identifiers. | Event-level history fields in profiles; opted-out users' records in training data; personal identifiers in training data | 0; 0; 0 (pseudonymous IDs only) |
| RAI4 | User control / accountability | A "Not Interested" choice shall always be honoured for that student. | Scripted test: mark 20 different recommended events as Not Interested, then reload the feed and "More like this" over 7 simulated days. | Reappearance rate of rejected events; change in average rank of similar events | 0% reappearance; similar events rank lower on average |

**Reflection — which RAI requirement is hardest to measure or enforce?**
RAI2 (fairness and exposure) is the hardest. Exposure bias builds up through a feedback loop: events that are ranked higher get more clicks, and those clicks become the training data for the next model. Offline replay only measures exposure on interactions that the old ranking already produced, so it cannot fully show what would have happened under a fairer ranking. Language is also hard to label reliably, because many UniNet event descriptions mix Malay and English. Measuring RAI2 properly will need language detection, and ideally an online test.
