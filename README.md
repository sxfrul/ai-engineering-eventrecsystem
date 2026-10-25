# UniNet Recommender: Enhanced Hybrid Event Recommendation
> An enhanced recommendation engine for UniNet that helps university students discover relevant technical opportunities, such as programming competitions, workshops, talks and meetups, through semantic matching and learned ranking.

## Problem Statement
UniNet's "For You" feed recommends events by combining content-based filtering and item-to-item collaborative filtering, but its ranking weights are configured manually, matching depends on predefined tags, and user interests are not updated as they change over time. As a result, students may miss relevant opportunities, and newly published events receive little exposure because they have no collaborative signal yet. AI engineering is the right approach because these issues need machine learning models, including semantic text embeddings and learning-to-rank models trained on real interaction data, built into a live system and evaluated with measurable metrics rather than more hand-written rules.

## Evidence of the Problem
- Event recommendation is a natural new-item cold-start problem: events are short-lived and usually have little or no user feedback before they take place, so collaborative filtering alone cannot rank them well — Macedo, Marinho & Santos (2015), *Context-Aware Event Recommendation in Event-based Social Networks*, ACM RecSys. https://doi.org/10.1145/2792838.2800187
- User preferences drift over time, and modelling these temporal dynamics significantly improved recommendation accuracy on the Netflix Prize dataset — Koren (2009), *Collaborative Filtering with Temporal Dynamics*, ACM KDD. https://doi.org/10.1145/1557019.1557072
- Recommendations drive most user engagement on large platforms, influencing about 80% of hours streamed on Netflix, so the quality of a recommender directly affects how much users find and use — Gomez-Uribe & Hunt (2015), *The Netflix Recommender System: Algorithms, Business Value, and Innovation*, ACM TMIS. https://doi.org/10.1145/2843948
- UniNet's current engine uses about ten hand-tuned ranking weights, describes events using only 4 categories and 14 flairs, and keeps interest profiles that never decay — UniNet source code, [`uninet/netlify/shared/recommend.ts`](uninet/netlify/shared/recommend.ts)

## Initial Related Work / Existing Approaches
- **Amazon item-to-item collaborative filtering** recommends items based on how often they are bought together, and scales well to large catalogues. It is the basis of UniNet's current collaborative filtering. **Limitation:** it needs co-interaction history, so it cannot recommend new items that nobody has interacted with yet — Linden, Smith & York (2003), *Amazon.com Recommendations: Item-to-Item Collaborative Filtering*, IEEE Internet Computing. https://doi.org/10.1109/MIC.2003.1167344
- **YouTube deep neural network recommender** splits recommendation into two stages, candidate generation and ranking, and trains the ranking model on implicit feedback such as watch time. UniNet already follows this two-stage design. **Limitation:** it relies on massive datasets and large-scale infrastructure, which a university-scale platform does not have — Covington, Adams & Sargin (2016), *Deep Neural Networks for YouTube Recommendations*, ACM RecSys. https://doi.org/10.1145/2959100.2959190
- **Context-aware event recommendation** combines content, social, location and time signals with learning-to-rank to recommend Meetup events. **Limitation:** it was evaluated only on English Meetup data, depends on social group signals, and does not handle multilingual content or recommendation diversity — Macedo, Marinho & Santos (2015), ACM RecSys. https://doi.org/10.1145/2792838.2800187

## Project Objectives
- O1: To analyse UniNet's existing recommendation engine and current hybrid recommendation techniques, and identify their limitations.
- O2: To design and develop an enhanced hybrid recommendation engine using multilingual sentence embeddings, time-decayed interest profiles, learned ranking models, a "Not Interested" feedback option and Maximal Marginal Relevance (MMR) re-ranking.
- O3: To evaluate the enhanced engine against the existing one using offline metrics (Precision@K, NDCG@K, coverage and diversity) and, where feasible, online A/B testing of click-through and registration rates.

## Target Users
University students in Malaysia who use UniNet to discover and register for technical events. Event organisers and university clubs also benefit indirectly, because their newly published events get better exposure to relevant students.

## Data Sources
- UniNet event and registration data (Cloud Firestore: events, registrations, onboarding preferences) — https://uninet.my
- UniNet interaction logs (Upstash Redis: views, scroll depth, register clicks and completed registrations) — internal, anonymised
- Pre-trained multilingual embedding model: paraphrase-multilingual-MiniLM-L12-v2 — https://huggingface.co/sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2
- Sentence-BERT multilingual knowledge distillation (Reimers & Gurevych, 2020) — https://arxiv.org/abs/2004.09813

## Weekly Progress
| Week | Focus | Deliverable | Status |
|------|-------|-------------|--------|
| W1 | System map + project scope | README + docs | ✅ |
| W2 | Literature review + analysis of current engine | Related work summary, baseline analysis | ⬜ |
| W3 | Data collection + preprocessing | Anonymised interaction dataset | ⬜ |
| W4 | Baseline offline evaluation | Baseline Precision@K, NDCG@K, coverage, diversity | ⬜ |
| W5 | Multilingual sentence embeddings | Event embeddings in nightly pipeline | ⬜ |
| W6 | Time-decayed interest profiles | Decay function + updated user profiles | ⬜ |
| W7 | Learning-to-rank (Logistic Regression) | Trained model + learned weights | ⬜ |
| W8 | Learning-to-rank (LambdaMART) | Model comparison results | ⬜ |
| W9 | "Not Interested" feedback | Feedback UI + profile update logic | ⬜ |
| W10 | MMR diversity re-ranking | Re-ranking module | ⬜ |
| W11 | Integration into UniNet | Enhanced engine deployed | ⬜ |
| W12 | Offline evaluation of enhanced engine | Comparison against baseline | ⬜ |
| W13 | Online A/B testing | CTR and registration rate results | ⬜ |
| W14 | Documentation + final report | Final README, report and demo | ⬜ |
