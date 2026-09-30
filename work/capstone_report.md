# Capstone Report — Flagging content decline before it costs traffic

- **Author: Muhammad Faizan**
- **Lane: 2 - Classification**
- **Repo: github.com/MuhammadFaizan0023/FlyRank_ML_internship_repo**
- **Date: September 30, 2026**

## 0. Abstract

SEO teams manage far more pages than they can manually re-check for decline. This project builds decline_score, a rule-derived proxy label (0–17) counting how many of 17 performance signals — search visibility, click-through rate, engagement, and traffic across six acquisition channels, moved in a negative direction between a prior 30-day window and the most recent 30 days. A multi-class classifier is trained to predict this label from prior-window features alone, and is evaluated under a client-holdout split so that no client's content is seen in both training and testing.<br>

A random split was found to inflate the model's macro F1 to 0.582, traced to heavy overlap between train and test content. The honest, client-grouped evaluation puts macro F1 at 0.444 and ROC-AUC at 0.913 — reliable at the extremes of the score range, weaker in the middle. The output is packaged as a ranked, reason-coded action playbook with explicit confidence tiers, intended strictly as decision support for a human reviewer, never as an automated content-change trigger.

## 1. Problem framing

Decision: flag which pages need a content refresh, so a human reviewer knows where to look first. <br>
Who acts: an SEO team or content owner who cannot manually audit every page's analytics on a recurring basis. <br>
Improvement: replaces a full manual sweep with a short, ranked list — a human still inspects each flagged page and decides the fix, but no longer has to find the candidates by hand.
What this can and cannot claim
This is decision support. It ranks and flags; a human with domain expertise still diagnoses the specific problem and decides the fix.
It will never:
Predict Google's core algorithm changes — it observes how a page reacted, not what Google will do next.
Prove refresh causes recovery — it can show traffic dropped alongside a lack of refresh, not that refresh was the only variable.
Guarantee rankings — it can support a case for improvement, not control or predict a specific position.
Auto-refresh content — it outputs a decision signal only, never an action.
Fix SEO issues directly, or identify who visits a page.
Claim certainty — it is trained on a limited time window, not a full year of seasonal behavior.

## 2. Data safety

Release: FlyRank/internship-warehouse on Hugging Face.
Tables used: dim_content, fact_content_daily_performance, merged for feature construction.
Date window: the analysis window spans 2026-04-01 through 2026-06-30 — a 60-day prior period (April–May) compared against the most recent 30 days (June) to build every prev30 / last30 feature pair and the decline_score label itself.
Excluded: dim_client (client identity kept out of the feature set entirely — see Methodology); fact_content_query_90d, which was already feature-engineered upstream and covered far fewer rows than the daily-performance table.
Every feature column is a prev30/last30 window aggregate or a static dim_content field — nothing extends past the last30 window, and no product/QA flag column exists in the feature set

## 3. Baseline
Since the score were calculated from reason codes, and this proxy label was measured from data. It does not corresponds to any input feature directly as this is derived label from the rule. So, only model is applied later on and test output was compared with model output results for comparison.  

## 4. Model / analysis

Your method and why it fits the lane. The exact feature list (and what you left out on
purpose). The target or proxy definition, in one sentence.
Feature list: 'client_hash_id', 'content_hash_id', 'gsc_impressions_prev30d', 'gsc_clicks_prev30d', 'gsc_avg_position_prev30d', 'ga4_pageviews_prev30d', 'ga4_sessions_prev30d', 'ga4_users_prev30d', ''ga4_engaged_session_prev30', 'ga4_total_engagement_sec_prev30', 'sessions_organic_prev30', 'sessions_direct_prev30', 'sessions_referral_prev30', 'sessions_social_prev30', 'sessions_paid_prev30', 'sessions_ai_prev30', 'scroll_events_prev30', 'content_created_date', 'content_updated_date', 'cpc_prev30', 'backlinks_prev30', 'ctr_prev30', 'engagement_rate_prev30', 'scroll_events_rate_prev30', 'days_since_update_prev30', 'gsc_impressions_last30d', 'gsc_clicks_last30d', 'gsc_avg_position_last30d', 'ga4_pageviews_last30d', 'ga4_sessions_last30d', 'ga4_users_last30d',
''ga4_engaged_session_last30', 'ga4_total_engagement_sec_last30', 'sessions_organic_last30', 'sessions_direct_last30', 'sessions_referral_last30', 'sessions_social_last30', 'sessions_paid_last30', 'sessions_ai_last30', 'scroll_events_last30', 'cpc_last30', 'backlinks_last30', 'ctr_last30', 'engagement_rate_last30', 'scroll_events_rate_last30', 'days_since_update_last30', 'decline_score'
Left out: 'client_hash_id', 'content_hash_id', 'content_created_date', 'content_updated_date', 'decline_score' because last one is proxy label and others are just for data grouping and are not involved as features for the training of model.
'decline_score' is a proxy label.

## 5. Evaluation

Split: time split to get signal difference and client-holdout split as train-test split so no two clients happens to be in both train and test split.
Scores: F1, auc-roc
Analysis of Overlap and Score Drop:
The Leakage Mechanism: Under a standard random split, multiple time-series snapshots of the exact same content items (content_hash_id) are split arbitrarily. As a result, approximately 97.38% of the content items in the test set already exist in the training set with highly similar historic features.
The Consequence: This severe leak inflates the model's performance to an unrealistic 0.5821 F1-score. When we transition to an honest grouped split (grouping strictly by content_hash_id or client_hash_id so that unseen items are tested), the model can no longer rely on memorizing specific content behaviors, which drops the F1-score to an honest 0.444.

## 6. Interpretation

What the model/clusters actually found. Feature importances or cluster profiles in plain
words. Surprises and negative results — a well-understood "no effect" is a valid result.

A Random Forest (10 estimators, balanced class weight) was trained on prev30 and last30, only features and evaluated on the held-out client set. Reported here is the single honest run — the client-grouped split, since that is the number that should govern trust in every downstream recommendation.
The leakage mechanism, and its consequence:
Under a standard random split, multiple time-series snapshots of the exact same content items are scattered arbitrarily across train and test. As a result, approximately 97.38% of test-set content already existed in training with highly similar historic features. This inflated macro F1 to an unrealistic 0.5821. Grouping strictly by client (so unseen content is genuinely unseen) drops this to an honest 0.444 — the model can no longer rely on having memorized a specific page's behavior.
Reliability varies sharply by class:
Classes 3 and 4 have almost no training examples (2 and 17 respectively) and score 0 across precision, recall, and F1 — this is a data volume ceiling, not a fixable model defect. The middle of the range (roughly 8–13) is weak for a different reason: the same aggregate score can arise from many different combinations of which specific conditions triggered, so the feature pattern per class is far less consistent even with tens of thousands of examples.

## 7. Recommendation
The queue is sorted by how strongly the available signals agree that a page has declined — not by a predicted content strategy. It resolves into three tiers:
At the extremes of the score range, evaluated accuracy is highest (class 16–17 F1 of 0.70–0.75), so a reviewer can act on those rankings with more confidence. In the middle of the range, the same score can arise from very different combinations of triggered reason codes, and accuracy is weaker (F1 0.39–0.55) — "what to do first" there means look here next, then verify manually, not treat the order as decisive.

## 8. Reproducibility
Open Colab, download the file from baseline_score.ipynb and run all the capstone, and you will get the results. 

## 9. Acknowledgments & data credit
Built on the FlyRank internship data warehouse, provided by flyrank.ai, released as FlyRank/internship-warehouse on Hugging Face.
FlyRank ML Internship Capstone · Muhammad Faizan · 2026

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
