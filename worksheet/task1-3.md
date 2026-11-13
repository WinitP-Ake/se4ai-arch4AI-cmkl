# Week 6 worksheet: serving patterns

**Name:** Winit Pongsripian

**Student ID:** 01326171

This is **individual work**: each student completes and submits their own copy, in their own fork.

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in English.

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large |509 |605 |93.5% |100% |
| t0-small |11 |25 |72.5% |0% |

**Q0.** What does the large model buy, and what does it cost?

>The large model gains about 21 points of accuracy (93.5% vs 72.5%), because it reads phrases like “not cold”. It costs roughly 45x the latency (509 ms vs 11 ms p50) and a paid large-model call on every request.

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before |26.2 |226 |364 |18 |0 |93.4% |
| t1-after |9.2 |405 |1008 |0 | 23|100% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

>p50 is the typical request. The hung calls are only 10% of traffic, so they don’t move the median, and some of them fail or time out at the client. The errors column (18, about 9% of 200) tells the real story, along with the 26 s run time.

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

>A hung call holds a worker and connection for up to 20 s in the review service and the gateway, so waiting requests pile up and block healthy ones. It also holds one of the large model’s 4 concurrent slots for 20 s (QR5), so the other 90% of calls have to queue behind it. One slow dependency then drags down the whole system, which is a cascading failure.

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

>The README says a batch job labels pending reviews later, so the topology is batch prediction, meaning offline labelling of the saved reviews when the large model is healthy. Pending reviews are kept (FR4) and retried by that job, so no review is lost.

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

>At 0.1 s the timeout is below the large model’s normal latency (about 200 ms per call, more when 4 calls are already running). Nearly every call would time out and become pending, so you’d pay for the model and still get no labels. At 10 s, a hung call keeps its resources for 10 s. That breaks QR2 (the review service waits at most 1 s) and QR1 (95% of reviews within 1 s), and it lets the pools fill up.

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 |5 |540 |91.5% |38% |
| t2-0.7 | 0.7 |390 |567 |93.0% |53% |
| t2-0.8 | 0.8 |434 |600 |93.5% |64% |
| t2-0.9 | 0.9 |480 |599 |93.5% |77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

>I would deploy 0.7. It gives 93.0% accuracy (QR3 needs at least 90%) and sends 47% fewer reviews to the large model than the large-only baseline (QR4), and p95 is 567 ms (QR1). The quality you give up is about 0.5 accuracy points, on reviews the small model is confident about but wrong. 0.6 also passes QR3 on paper (91.5%) and is cheaper, but with only 200 reviews a 1.5-point margin is thin, and 0.8 and 0.9 cost much more for little gain.

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

>About half the requests are answered by the small model alone (about 11 ms), so the median falls among those fast requests. The p95 is set by the escalated requests, which pay for the small model and the large one (about 500+ ms). Those are far more than 5% of traffic, so the tail barely moves.

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

>The small model can be confidently wrong on sarcasm, a negation it misreads, or new slang. The wrong label is accepted without escalation. For users, a real complaint may be labelled positive, so the restaurant owner is never alerted to an unhappy customer.

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache |5 |537 |93.0% |38% |35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

>Repeated or near-identical reviews benefit, such as short generic ones (“good food”, “yummy”) or duplicate posts, since the cache ignores case and extra spaces. Unique, long free-text reviews never get a hit.

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

>The cached answers come from the old model, so they must be invalidated. Put the model version in the cache key (hash(normalised text) + model_version), so the new model starts with an empty cache and old entries age out through the LRU eviction or a TTL.

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

>Fallback trades latency for availability. The lecture only says systems need “fallbacks”. In our runs, t1-after removed all errors but raised p50 (226 → 405 ms) and p95 (364 → 1008 ms), and left 23 reviews pending. Its accuracy of 100% covered only the reviews that were labelled.