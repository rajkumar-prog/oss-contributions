# PR #2737 — Fix wrong TP in FactualCorrectness recall and F1 modes
**Repo:** ragas (explodinggradients → now vibrantlabsai)  
**PR:** https://github.com/vibrantlabsai/ragas/pull/2737  
**Issue:** #2693 — `FactualCorrectness(mode="recall")` returns precision score instead of recall

---

## Before — What was the world like?

**Ragas** is a library for evaluating LLM applications. One of its most important metrics is `FactualCorrectness` — it measures how factually accurate an AI's response is compared to a reference answer.

To measure factual accuracy, ragas uses **NLI (Natural Language Inference)** — a technique where you take two pieces of text and ask: "Does one support the other?"

For example:
- Text A: "The Eiffel Tower is in Paris"
- Text B: "The Eiffel Tower is located in France"
- Question: Does B support A? → **Yes** (Paris is in France)

`FactualCorrectness` runs NLI in **two directions**:

| Variable | Direction | What it checks |
|---|---|---|
| `reference_response` | Response → Reference | For each claim in the response, is it supported by the reference? |
| `response_reference` | Reference → Response | For each claim in the reference, is it supported by the response? |

These two directions measure different things:
- **Precision:** Are the AI's claims accurate? (Use `reference_response`)
- **Recall:** Did the AI cover everything in the reference? (Use `response_reference`)

This distinction is critical. Precision and recall are fundamentally different measures.

---

## What was the Problem?

The bug was in `_single_turn_ascore`, the function that computes the final score. Here's the original code:

```python
tp = sum(reference_response)      # TRUE POSITIVES — response claims verified by reference
fp = sum(~reference_response)     # FALSE POSITIVES — response claims NOT in reference

if self.mode != "precision":
    fn = sum(~response_reference) # FALSE NEGATIVES — reference claims NOT covered by response
else:
    fn = 0

if self.mode == "recall":
    score = tp / (tp + fn + 1e-8)  # ← BUG HERE
```

See it? In recall mode, `tp` is still `sum(reference_response)` — the **precision TP**. But recall should be computed from `response_reference` — how many reference claims are covered.

**Concrete example from the issue:**

| | Response | Reference |
|---|---|---|
| Total claims | 4 | 3 |
| Supported by the other | 3 of 4 supported by reference | 2 of 3 supported by response |

- **True precision TP** = 3 (response claims verified by reference)
- **True recall TP** = 2 (reference claims covered by response)

With the bug:
```
recall = tp / (tp + fn) = 3 / (3 + 1) = 0.75   ← WRONG (uses precision TP)
```

Correct recall:
```
recall = tp_recall / (tp_recall + fn) = 2 / (2 + 1) = 0.667   ← CORRECT
```

The bug made the recall score **inflated** — it looked better than it actually was. Users were getting falsely optimistic recall numbers.

The same bug affected F1 mode too, since F1 uses both precision and recall.

---

## The Fix — What We Changed

We renamed `tp` to `tp_precision` and added `tp_recall` as a separate variable:

```python
# Two separate TPs — one for each direction
tp_precision = sum(reference_response)   # response claims verified by reference
fp = sum(~reference_response)

if self.mode != "precision":
    tp_recall = sum(response_reference)  # reference claims covered by response
    fn = sum(~response_reference)
else:
    tp_recall = 0
    fn = 0

if self.mode == "recall":
    score = tp_recall / (tp_recall + fn + 1e-8)  # now uses the RIGHT TP
```

And we updated `fbeta_score` to accept both TPs so the F1 calculation uses the right one for each leg:

```python
# Before
def fbeta_score(tp, fp, fn, beta):
    precision = tp / (tp + fp)
    recall = tp / (tp + fn)  # ← also used precision TP for recall

# After
def fbeta_score(tp_precision, fp, tp_recall, fn, beta):
    precision = tp_precision / (tp_precision + fp)
    recall = tp_recall / (tp_recall + fn)  # ← now uses recall TP
```

---

## The Tests — 9 Unit Tests Added

We added tests in `tests/unit/test_factual_correctness_scores.py`. The most important one is the **regression test**:

```python
def test_recall_uses_reference_claims():
    reference_response = np.array([True, True, True, False])  # 3 of 4 correct
    response_reference = np.array([True, True, False])         # 2 of 3 covered

    score = _compute_score(reference_response, response_reference, mode="recall")

    # Explicitly check that the OLD WRONG value is not returned
    assert score != pytest.approx(0.75, abs=1e-3), "Bug has not been fixed"
    assert score == pytest.approx(2/3, rel=1e-3)   # correct: 0.667
```

This test will fail if the bug ever comes back — it explicitly checks for the old wrong answer.

---

## What Happens Next

- If merged, all users of `FactualCorrectness(mode="recall")` will get correct recall scores
- People who built dashboards or benchmarks using ragas recall scores may see their numbers drop — this is expected (the old numbers were wrong)
- The `fbeta_score` signature change is a breaking change — code that calls it with 3 positional args will break, but it's an internal function so this is acceptable

---

## Key Lesson

**Naming variables clearly prevents bugs.** The original code had one variable `tp` that silently meant "precision TP" in all modes. By renaming to `tp_precision` and `tp_recall`, the bug becomes obvious — you can *see* that the recall formula is using the wrong variable.

Precision and recall are two sides of a trade-off:
- **High precision, low recall:** AI only says things it's sure about, but misses a lot
- **Low precision, high recall:** AI covers everything but includes wrong things
- **F1:** Balance between the two

Measuring them incorrectly gives a false picture of how the AI is performing.
