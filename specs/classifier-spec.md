# Classifier Spec — Pod Classifier

Complete this spec **before** writing any code for Milestone 2.

Use Plan or Ask mode to think through each blank field. When you're done,
your answers here become the blueprint for `build_few_shot_prompt()` and
`classify_episode()` in `classifier.py`.

---

## build_few_shot_prompt(labeled_examples, description)

### What it does
Constructs a prompt string for the LLM that includes the task instructions,
all labeled training examples, and the new episode description to classify.

### Inputs

| Parameter | Type | Description |
|---|---|---|
| `labeled_examples` | `list[dict]` | Each dict has `"title"`, `"description"`, `"label"` (and others). These are the examples you labeled in Milestone 1. |
| `description` | `str` | The episode description to classify. |

### Output

| Return value | Type | Description |
|---|---|---|
| prompt | `str` | A complete prompt string ready to send to the LLM. |

---

### Spec fields — fill these in before writing code

**Task instruction (what should the LLM know about the task?):**

```
You are classifying podcast episodes by their format. Classify the episode
into exactly one of these four labels:

- interview: a conversation between a host and one or more guests
- solo: a single host speaking from memory, experience, or opinion — no guests,
  no assembled external sources
- panel: multiple guests with roughly equal speaking time, often debating or
  discussing a topic together
- narrative: a story assembled from external sources — interviews, archival
  audio, reporting — with a clear narrative arc

Return only the label and your reasoning. Do not explain the taxonomy.
```

---

**How should labeled examples be formatted in the prompt?**

```
Each example should include the episode title, a brief excerpt or the full
description, and the correct label. Separate examples with a blank line or
a delimiter like "---". Include all fields that help the model see why the
label was applied — title and description are both useful; other fields
(like episode ID) are not needed.
```

---

**Example block sketch (write one concrete example):**

```
Title: {title}
Description: {description}
Label: {label}
```

---

**How should the new episode (to be classified) be presented?**

```
Present it in the same format as the labeled examples, but omit the Label
line and replace it with an instruction to classify. For example:

Title: {title}
Description: {description}
Label: ?

Then add a line like: "Classify the episode above. Return your answer in
the format below:" followed by the output format you chose.
```

---

**What output format should you request from the LLM?**

```
Request this exact two-line format:

  Label: <label>
  Reasoning: <one sentence explanation>

Tradeoffs:
- A bare label on one line is the simplest to parse, but you lose reasoning
  and can't tell whether the model was confident or guessing.
- JSON is the most machine-readable, but LLMs sometimes emit prose before or
  after the JSON block, requiring more defensive parsing.
- The "Label: X / Reasoning: Y" two-line format is the best balance: easy to
  parse with a single startswith("Label:") check, and the reasoning line is
  always present for debugging and the Implementation Notes section.

Include this instruction verbatim at the end of the prompt:
  "Respond in exactly this format:
  Label: <one of: interview, solo, panel, narrative>
  Reasoning: <one sentence>"
```

---

**Edge cases to handle in the prompt:**

```
1. labeled_examples is empty: the prompt becomes zero-shot — still valid, just
   omit the "Examples:" block entirely. The LLM can still classify from the
   task instruction and label definitions alone.

2. Description is very short (e.g., one sentence or a title only): include it
   as-is. Don't pad or skip it; a short description is still a real signal.
   The LLM may return "unknown" or a low-confidence label — that's fine and
   will show up in evaluate.py results.

3. Description contains special characters or newlines: no escaping needed
   since the prompt is plain text, but strip leading/trailing whitespace before
   inserting so the format stays consistent.
```

---

## classify_episode(description, labeled_examples)

### What it does
Classifies a single podcast episode description using the few-shot LLM classifier.
Returns a dict with a label and reasoning.

### Inputs

| Parameter | Type | Description |
|---|---|---|
| `description` | `str` | The episode description to classify. |
| `labeled_examples` | `list[dict]` | Labeled training examples from `load_labeled_examples()`. |

### Output

| Return value | Type | Description |
|---|---|---|
| result | `dict` | Must have keys `"label"` and `"reasoning"`. `"label"` must be one of `VALID_LABELS` or `"unknown"`. |

---

### Spec fields — fill these in before writing code

**Step 1 — Build the prompt:**

```
Call build_few_shot_prompt(labeled_examples, description) and store the
returned string in a variable (e.g., prompt). Pass through both arguments
exactly as received — no modification needed before calling.
```

---

**Step 2 — Send to the LLM:**

```
Call _client.chat.completions.create() with:
  - model: the model name from config (LLM_MODEL)
  - messages: a list with one dict — {"role": "user", "content": prompt}
    (system-design.md shows an optional system message too — either shape works)
  - max_tokens: a reasonable limit (e.g., 200–300) to keep responses concise

Extract the response text from:
  response.choices[0].message.content
```

---

**Step 3 — Parse the response:**

```
Given the "Label: X\nReasoning: Y" format, iterate over the response lines:

  label = "unknown"
  reasoning = ""
  for line in raw_text.strip().splitlines():
      if line.lower().startswith("label:"):
          label = line.split(":", 1)[1].strip().lower()
      elif line.lower().startswith("reasoning:"):
          reasoning = line.split(":", 1)[1].strip()

Use split(":", 1) (not split(":")) so a colon inside the reasoning text doesn't
get cut off. Call .strip().lower() on the label so "  Interview\n" → "interview".
```

---

**Step 4 — Validate the label:**

```
After parsing, check:

  if label not in VALID_LABELS:
      label = "unknown"

Do NOT try to fuzzy-match (e.g., mapping "interviews" → "interview"). A
non-exact label means the model ignored the format instruction; "unknown" is
the correct signal so evaluate.py can count format failures separately.
Keep reasoning unchanged — it still contains useful debugging information.
```

---

**Step 5 — Handle errors gracefully:**

```
Wrap the entire function body in a try/except:

  try:
      # Steps 1–4 here
      return {"label": label, "reasoning": reasoning}
  except Exception as e:
      return {"label": "unknown", "reasoning": f"Error: {e}"}

What can go wrong:
- Network / timeout error from the API call → caught by Exception
- API key missing or rate-limit hit → same
- Response is empty or the "Label:" line is missing → parsing produces
  label = "unknown" naturally (Step 3 initializes it that way)
- Response is valid but label is not in VALID_LABELS → Step 4 handles it

Returning {"label": "unknown", ...} on every failure lets evaluate.py's
loop complete all 20 calls and count errors as a distinct outcome rather
than crashing mid-run.
```

---

### Return value structure

```python
{
    "label": str,      # one of VALID_LABELS, or "unknown" if invalid/error
    "reasoning": str,  # brief explanation from the LLM
}
```

---

## Notes on label quality

The classifier is only as good as your labels. If your training examples have
inconsistent or ambiguous labels, the LLM will learn the wrong pattern.

Before implementing the classifier, re-read `data/taxonomy.md` and double-check
any labels you're unsure about. Annotation quality is part of the lab.

---

## Implementation Notes

*Fill this in after implementing and testing both functions.*

**Test: what does the raw LLM response look like for one episode?**

```
Episode tested: [title]
Raw response text: [paste it here]
```

**How did you parse the label out of the response?**

```
[describe the string operations — strip, split, lower, etc.]
```

**Did any episodes return `"unknown"`? If so, why?**

```
[yes / no — if yes, what did the raw response look like?]
```

**One thing about the output format that surprised you:**

```
[your answer here]
```
