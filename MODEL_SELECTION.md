# Model selection

## The question
The model predicts whether a customer is going to churn (leave). A false alarm wastes a retention offer on someone who would have stayed; a missed churner is a customer we lose without a chance to react. F1 balances both, so it is my main metric.

## Candidates
Data: `data/churn_v1.csv` (1200 customers, about 25% churn), 80/20 split, the same split for every candidate.

| Candidate | Why I tried it | AUC | F1 | F1 per slice (lowest) | p95 latency (ms) | Size (KB) | MLflow run ID |
|---|---|---|---|---|---|---|---|
| logreg_balanced | Simple and fast baseline; `class_weight="balanced"` because only 25% of customers churn | 0.797 | 0.585 | 0.316 (two-year) | 2.4 | 4 | cdc3e0d2 |
| random_forest | A non-linear model that can capture interactions (100 trees, depth 6, balanced weights) | 0.778 | 0.573 | 0.400 (two-year) | 6.7 | 669 | b5902c0f |
| gradient_boosting | Usually the strongest on tabular data (150 trees, learning rate 0.05, depth 3) | 0.759 | 0.396 | 0.250 (one-year) | 2.7 | 206 | 8682f8fc |

F1 per contract type:

| Candidate | month-to-month | one-year | two-year |
|---|---|---|---|
| logreg_balanced | 0.626 | 0.571 | 0.316 |
| random_forest | 0.591 | 0.615 | 0.400 |
| gradient_boosting | 0.417 | 0.250 | 0.364 |

## The champion
I picked **logreg_balanced**. It has the best F1 (0.585) and the best AUC (0.797), and it is also the fastest (2.4 ms) and the smallest (4 KB). It is the best on four of the five things I compared, so nothing pulls me towards a more complex model.

What I accepted losing: on two-year contracts its F1 is 0.316, lower than the random forest's 0.400. The random forest is better on the weakest slice, but it is only 0.012 better overall, almost three times slower and 150 times bigger. Gradient boosting was worst overall because I did not give it class weights, so it rarely predicts churn.

## My promotion rules
- Latency budget (p95): **20 ms**, because one prediction in an API should feel instant, and the slowest candidate here takes 6.7 ms on my laptop. CI runners are slower than my laptop, so I left about a 3x safety margin.
- Minimum gain to replace the champion: **0.01 F1 points**, because the test set has only 240 customers, so differences smaller than about one point are noise and I do not want to swap models for luck.

## What I would check next
The two-year slice has only about 50 test customers and about 6 of them churn, so its F1 changes a lot when one prediction changes. I would collect more data for it, or use cross-validation, before trusting that logreg is really worse than the random forest there. I would also watch for data drift: if customers change, the data hash tells me the model was trained on older data.
