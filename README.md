# Semantic Caching and Cost-Aware Routing for LLMs

Two popular ways to cut LLM cost and latency: reuse answers for questions that mean the same thing (a **semantic cache**), and send easy questions to a cheap model and hard ones to a strong model (a **router**). This project builds both, measures what they actually save, and checks that the answers stay correct.

Built in Google Colab with Groq (`openai/gpt-oss-20b` as the cheap model, `openai/gpt-oss-120b` as the strong model) and sentence-transformers.

## What it does

- **Semantic cache:** each question is turned into an embedding. If a new question is similar enough to an earlier one (cosine similarity above a threshold), the saved answer is returned without calling the model.
- **Number guard:** blocks a cache hit when the numbers in the two questions differ, so "15% of 250" cannot get the answer to "15% of 240".
- **Router:** a simple complexity score (question length and words like "explain", "compare", "step by step") sends the question to the cheap or the strong model. If the cheap answer looks bad, it escalates to the strong model and logs the reason.
- **Workload:** 39 questions (13 groups, each asked 3 different ways). Three pairs are deliberate traps with almost identical wording and different correct answers: France vs Germany, 15% of 240 vs 250, Australia vs Austria.
- **Four setups compared:** A always strong, B router only, C router + cache, D router + cache + number guard.
- **Quality check:** every answer, including cached ones, must contain the expected keyword. Cost is reported per correct answer.
- **Threshold sweep:** tests six similarity thresholds without extra API calls.

## Results from the run

| Setup | Total cost | Avg latency | Correct answers | Cache hits | False hits |
|---|---|---|---|---|---|
| A: always strong | $0.00363 | 1.32 s | 39 of 39 | 0 | 0 |
| B: router only | $0.00367 (+1.1%) | 1.14 s (-14%) | 39 of 39 | 0 | 0 |
| C: router + cache | $0.00174 (-52%) | 0.56 s (-57%) | 39 of 39 | 23 | 0 |
| D: router + cache + number guard | $0.00160 (-56%) | 0.54 s (-59%) | 39 of 39 | 23 | 0 |

Cost per correct answer fell from $0.000093 (always strong) to $0.000041 (setup D).

**Threshold sweep** (cache hits / false hits): 0.70: 28 / 2, 0.75: 27 / 1, 0.80: 27 / 1, **0.85: 23 / 0**, 0.90: 20 / 0, 0.95: 5 / 0. With the number guard, false hits drop to zero from 0.75 upward, so a lower threshold becomes safe and more hits are kept.

## What the evaluation showed

- **The cache did almost all the saving**, with zero false hits and no loss in answer quality. Below a 0.85 threshold, look-alike questions started sharing wrong answers, which is exactly what the trap pairs were built to expose.
- **A number guard lets you lower the threshold safely** (0.75 instead of 0.85), which keeps more hits.
- **The router saved nothing with this model pair.** Cost went up 1.1% and latency improved 14%. The cheap and strong models are close in price (about 1.5x), and I had planned to use `llama-3.1-8b-instant` for a much bigger price gap, but it returned a 404 (not available on this account), so the notebook fell back to `gpt-oss-20b`. Routing only pays off with a large price gap.
- **Honest flaw in my escalation rule:** all 4 escalations in setup B were flagged "too short", meaning correct short answers such as numbers were treated as bad. The conclusion about the router does not change (the extra cost from those 4 calls is small), but a better rule would only escalate on empty answers or refusals.

## Limitations

- **The cache hit rate is high by design:** 20 of the 39 questions are paraphrases of earlier ones. A real workload would have fewer repeats.
- **Only 39 questions.** Treat all percentages as a demo, not a benchmark.
- **The quality check is a keyword match**, which is lenient. An LLM judge would be stricter (see my LLM Evaluation Platform project).
- **Prices:** the two gpt-oss prices come from Groq's pricing page. Prices change, so check groq.com/pricing and edit the table in the notebook.
- p95 latency barely improved (about 2.8 to 3.0 seconds in every setup), because the slowest calls are the uncached strong-model calls.

## Tech stack

Groq API, sentence-transformers (`all-MiniLM-L6-v2`), pandas, matplotlib.

## How to run

1. Open the notebook in Google Colab.
2. Run the cells from top to bottom.
3. When asked, paste your own Groq API key (free at console.groq.com). It is hidden while typing and is never saved in the notebook.

It makes about 140 API calls and takes several minutes.

## Files the notebook creates

- `routing_cache_log.csv`: every query in every setup (model used, cost, latency, hit, escalation reason, answer)
- `routing_cache_summary.csv`: the comparison table
- `routing_cache_threshold_sweep.csv`: hits and false hits by threshold

---

Built by **Akshat Kesharwani** | [GitHub](https://github.com/akshatkesharwani-info) | [Portfolio](https://akshatkesharwani-info.github.io/)
