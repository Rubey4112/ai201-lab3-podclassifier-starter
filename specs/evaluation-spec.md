# Evaluation Spec — Pod Classifier

Complete this spec **before** writing any code for Milestone 3.

Use Plan or Ask mode to think through each blank field. When you're done,
your answers here become the blueprint for `compute_accuracy()` and
`compute_per_class_accuracy()` in `evaluate.py`.

---

## Background: What is evaluation?

After building a classifier, we need to know how well it works. Evaluation answers:
- **Overall:** What fraction of episodes did we classify correctly?
- **Per-class:** Are we better at some labels than others?

Both functions take the same inputs: a list of predicted labels and a list of
ground-truth labels, in the same order.

---

## compute_accuracy(predictions, ground_truth)

### What it does
Returns the fraction of predictions that exactly match the ground truth.

### Inputs

| Parameter | Type | Description |
|---|---|---|
| `predictions` | `list[str]` | Labels predicted by `classify_episode()`, one per episode. |
| `ground_truth` | `list[str]` | The correct labels, in the same order as `predictions`. |

### Output

| Return value | Type | Description |
|---|---|---|
| accuracy | `float` | A value between 0.0 and 1.0. |

---

### Spec fields — fill these in before writing code

**Formula:**

```
accuracy = (number of positions where predictions[i] == ground_truth[i]) / len(predictions)

A prediction is "correct" when it exactly matches the ground-truth label at the same
index. Divide the count of correct predictions by the total number of predictions to
get a float between 0.0 and 1.0.
```

---

**Step-by-step logic:**

```
1. If len(predictions) == 0, return 0.0 immediately (edge case).
2. Count how many indices i satisfy predictions[i] == ground_truth[i].
3. Divide that count by len(predictions).
4. Return the result as a float.
```

---

**Edge case — what if both lists are empty?**

```
Return 0.0.

There are no predictions to evaluate, so accuracy is undefined. 0.0 is the safe
default — it avoids a ZeroDivisionError and signals "no data" rather than "perfect".
```

---

**Worked example:**

```
predictions  = ["interview", "solo", "panel", "interview"]
ground_truth = ["interview", "solo", "solo",  "narrative"]

index 0: "interview" == "interview"  ✓
index 1: "solo"      == "solo"       ✓
index 2: "panel"     != "solo"       ✗
index 3: "interview" != "narrative"  ✗

correct = 2, total = 4
compute_accuracy() returns 2 / 4 = 0.5
```

---

## compute_per_class_accuracy(predictions, ground_truth)

### What it does
Returns accuracy broken down by each label. For each label in `VALID_LABELS`,
reports how many episodes with that ground-truth label were classified correctly.

### Inputs

| Parameter | Type | Description |
|---|---|---|
| `predictions` | `list[str]` | Labels predicted by `classify_episode()`. |
| `ground_truth` | `list[str]` | Correct labels, in the same order. |

### Output

A `dict` keyed by label. Each value is a dict with three keys:

```python
{
    "interview": {"correct": int, "total": int, "accuracy": float},
    "solo":      {"correct": int, "total": int, "accuracy": float},
    "panel":     {"correct": int, "total": int, "accuracy": float},
    "narrative": {"correct": int, "total": int, "accuracy": float},
}
```

---

### Spec fields — fill these in before writing code

**What does "correct" mean for a given class?**

```
An episode counts as correctly classified for class X when BOTH of the following
are true:
  1. ground_truth[i] == X   (the episode actually belongs to class X)
  2. predictions[i]  == X   (we predicted X)

For example, an episode counts as "correct" for the "interview" class only when
its true label is "interview" AND we predicted "interview". If we predicted
"interview" but the truth is "solo", that does NOT count as correct for either class.
```

---

**What does "total" mean for a given class?**

```
"total" for class X is the number of episodes whose ground-truth label is X —
regardless of what was predicted for them. It is NOT the overall total number of
predictions.

For example, if 5 out of 20 test episodes truly belong to "interview", then
total["interview"] == 5.
```

---

**Step-by-step logic:**

```
1. Initialize a dict with an entry for each label in VALID_LABELS:
     { label: {"correct": 0, "total": 0, "accuracy": 0.0} }

2. Loop over each index i in range(len(predictions)):
     predicted = predictions[i]
     truth     = ground_truth[i]

3. For each pair:
     a. Increment stats[truth]["total"] by 1  (this episode belongs to class `truth`)
     b. If predicted == truth, also increment stats[truth]["correct"] by 1

4. After the loop, for each label in VALID_LABELS:
     if stats[label]["total"] > 0:
         stats[label]["accuracy"] = stats[label]["correct"] / stats[label]["total"]
     else:
         stats[label]["accuracy"] = 0.0

5. Return the stats dict.
```

---

**Edge case — what if a class has no examples in ground_truth (total == 0)?**

```
Set accuracy to 0.0.

Dividing by zero is undefined, and a missing class means there is nothing to
evaluate. 0.0 is the safe default — it matches what the docstring in evaluate.py
specifies and prevents a ZeroDivisionError at runtime.
```

---

**Worked example:**

```
predictions  = ["interview", "interview", "solo", "panel", "panel"]
ground_truth = ["interview", "solo",      "solo", "panel", "narrative"]

Trace through each pair:
  i=0: truth="interview", pred="interview"  → interview: total+1, correct+1
  i=1: truth="solo",      pred="interview"  → solo:      total+1  (no correct)
  i=2: truth="solo",      pred="solo"       → solo:      total+1, correct+1
  i=3: truth="panel",     pred="panel"      → panel:     total+1, correct+1
  i=4: truth="narrative", pred="panel"      → narrative: total+1  (no correct)

label       correct  total  accuracy
----------  -------  -----  --------
interview       1      1      1.0
solo            1      2      0.5
panel           1      1      1.0
narrative       0      1      0.0
```

---

## Reflection questions (discuss at the checkpoint)

1. Your overall accuracy might be decent even if one class has very low accuracy.
   Why is per-class accuracy a more informative metric than overall accuracy alone?

2. If `panel` episodes consistently get misclassified as `interview`, what does
   that tell you about your training labels or your prompt?

3. You labeled 20 training episodes and evaluated on 20 test episodes (5 per class).
   How might the evaluation results change if you had labeled 100 training episodes?
   What if you had 200 test episodes?
