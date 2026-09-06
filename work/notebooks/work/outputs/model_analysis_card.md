# Content Action Playbook -- Model / Analysis Card

## What this is
A five-way triage queue (protect / refresh / expand / prune / monitor) for prioritizing content
review, built from FlyRank's 2026-03 warehouse snapshot. Two of the five buckets (protect, refresh)
are ranked using an out-of-fold Random Forest classifier (20.1% base rate,
grouped-split AUC 0.721, precision@50 0.96 on the `is_declining_proxy` label -- see w05/w06). The
other three buckets (expand, prune, monitor) are transparent rule-based heuristics with no
ground-truth validation available in this warehouse.

## What it's for
Helping a content team with limited review hours per cycle decide what to look at first. It is a
prioritization aid, not an auto-execution pipeline -- see Section 3's no-go list.

## Data
- `fact_content_daily_performance`, `dim_content`, `dim_clients` -- FlyRank internship warehouse,
  build v20260703.
- Feature window: 2026-03 only. Comparison window (2026-02) used solely to build the
  `is_declining_proxy` label, never as a feature (verified in w05 Section 3.1 / w06 Section 3).

## Method
- Baseline: transparent AND-gate rule (eligible x stale x CTR-gap), see w04/w05 Section 0.2.
- Model: Random Forest (n_estimators=300, max_depth=8, min_samples_leaf=20, class_weight=balanced),
  5-fold GroupKFold by client_hash_id, out-of-fold predictions only.
- Action assignment: priority-ordered rule set (Section 1.0-1.1) combining `pred_rf` with position,
  visibility, staleness, word count, and age -- full thresholds documented in Section 1.0's code cell.

## Performance (protect/refresh, validated against is_declining_proxy)
See Section 1.2's table for the current run's exact numbers; w05/w06 reported AUC 0.721-0.723 and
precision@50 of 0.96 on this same model spec.

## Changes this cycle (w07)
- Fixed `precision_at_k` to stop padding degenerate/near-all-zero scores out to K with arbitrary ties (affected the baseline column only; see the Changelog at the top of the notebook).
- Fixed the ranked queue to sort by action tier (protect > refresh > expand > monitor > prune) before priority_score, so a heuristic bucket can no longer outrank a validated protect/refresh row on traffic size alone.

## Where it breaks
- expand/prune/monitor are unvalidated heuristics (Section 2).
- Single-month snapshot; no rolling or out-of-time test yet (Section 4).
- `avg_position_mar` inherits a known no-data-day averaging bias (Section 0.3).
- Weaker for clients the model has not seen before (Section 2, citing w06's audit of the source paper).
- Label measures search-visibility decline only, not revenue or conversions.

## Owner / retrain cadence
Monthly, alongside the warehouse's monthly partitions (Section 4).
