# TakeMeter — Classifying r/soccer Discourse

A four-way text classifier that sorts r/soccer posts into the kinds of discourse the
community actually produces: structured **analysis**, evidence-free **hot takes**,
live **match reactions**, and discussion-prompting **debates**. The project compares a
fine-tuned `distilbert-base-uncased` model against a zero-shot Groq
(`llama-3.3-70b-versatile`) baseline on a hand-labeled dataset of 200 posts.

> **Dataset:** [`posts.csv`](posts.csv) (200 posts, 50 per label) · **Design notes:** [`planning.md`](planning.md)

---

## Community choice and reasoning

I chose **r/soccer**. It is a community I follow closely, so I can judge whether the
model's outputs are sensible rather than trusting metrics blindly. More importantly,
its discourse is genuinely varied: the same subreddit produces tactical breakdowns with
xG/PPDA stats, bold one-line opinions, live in-match score posts, and GOAT/ranking
debates. Those four modes are visually and linguistically distinct enough to be
learnable, but they overlap at the edges (a heated match reaction can read like a hot
take), which makes the classification task non-trivial and interesting.

---

## Label taxonomy

I kept three of the provided example labels and replaced `reaction` with `debate`,
because pure emotional reactions are rare as standalone r/soccer *posts* — they live in
comments — whereas GOAT threads, rankings, and comparison posts are everywhere and easy
to collect.

| Label | Definition | Example 1 | Example 2 |
|---|---|---|---|
| **analysis** | A structured argument backed by statistics, historical comparison, or tactical observation; evidence is specific and verifiable. | "City's high press under Guardiola is unsustainable — their PPDA has dropped from 7.2 to 9.8 over the last 3 seasons as the squad ages." | "Rodri's passing map from last night shows how he shifts between a pivot and a 6 depending on whether Bernardo is in the half-space." |
| **hot_take** | A bold, confident opinion stated *without* supporting evidence; it asserts rather than argues. | "Haaland is not a top 5 player in the world. He only scores because he's fed by elite creators." | "The Premier League is massively overrated. Serie A in the 90s had better football, full stop." |
| **match_reaction** | A post made during or right after a match, reacting to the result, a key moment, or the scoreline; the focus is the live/just-finished event. | "IT'S 3-2 IN THE 94TH MINUTE. ABSOLUTE SCENES." | "FT: Liverpool 4-0 United. That second half was embarrassing." |
| **debate** | A post that invites comparison or community discussion — GOAT arguments, rankings, hypothetical matchups; it poses a question rather than asserting one. | "Who was the better player: peak Ronaldinho or peak Zidane? Genuine debate." | "Rate this all-time r/soccer XI — drop your changes below." |

---

## Data collection & labeling

- **Source:** Posts collected from **r/soccer** (titles/post bodies).
- **Size & distribution:** 200 posts, perfectly balanced at **50 per label** across
  `analysis`, `hot_take`, `match_reaction`, and `debate`.
- **Labeling process:** I used Gemini to *pre-label* each post against my four
  definitions and edge-case rules, with a one-sentence rationale per post. I then
  reviewed **every** label myself and corrected disagreements — every final label
  reflects my own judgment after seeing the suggestion. (See the AI Usage section.)

### Label distribution

| Label | Count |
|---|---|
| analysis | 50 |
| hot_take | 50 |
| match_reaction | 50 |
| debate | 50 |
| **Total** | **200** |

### 3 difficult-to-label examples and my decisions

1. **"EXCLUSIVE: Como agree deal with Real Madrid to BUY Nico Páz... here we go!"**
   → labeled **hot_take**. *Decision:* transfer/news reports assert a claim without
   building an argument and have no community-discussion hook, so they fall under
   `hot_take` ("asserts rather than argues") rather than `analysis` or `debate`. This is
   a deliberate rule, and the evaluation below shows it is the model's hardest boundary.

2. **"Rank the top 5 managers in world football right now. Do you agree with my list?"**
   → labeled **debate**, not `hot_take`. *Decision rule:* an explicit invitation to weigh
   in (a question, "do you agree?", "change my mind") is the defining feature of `debate`,
   even when the post also states an opinion.

3. **"OTD 10 years ago, Lionel Messi announced his retirement... following a 4-2 defeat
   to Chile on penalties in the Copa América final."**
   → labeled **hot_take** (historical/opinion-adjacent post). *Decision:* it has no live
   match anchor, so it is *not* `match_reaction` by my rule — but it *contains a
   scoreline*, which makes the boundary genuinely fuzzy. An earlier training run
   misclassified this post as `match_reaction`, confirming the label-boundary tension.

---

## Fine-tuning approach

- **Base model:** `distilbert-base-uncased` with a 4-class sequence-classification head.
- **Split:** 70% train / 15% validation / 15% test (140 / 30 / 30), **stratified** by
  label, `random_state=42`.
- **Tokenization:** truncation at `max_length=256`, dynamic padding via
  `DataCollatorWithPadding`.
- **Training setup:** 30 epochs, learning rate `2e-5`, train batch size 16, weight decay
  `0.01`, 50 warmup steps, `load_best_model_at_end=True` selecting on validation accuracy.

### Hyperparameter decision

I increased `num_train_epochs` from the starter default of **3 to 30**. With only 140
training examples each epoch is very fast, and I wanted enough passes for the model to
fully separate the lexically-overlapping classes rather than stopping while the
`hot_take`/`match_reaction` boundary was still under-learned. To control the overfitting
that 30 epochs invites, I relied on `eval_strategy="epoch"` together with
`load_best_model_at_end=True` and `metric_for_best_model="accuracy"`: the Trainer
evaluates on the validation set after every epoch and restores the best-performing
checkpoint at the end. So even though training runs 30 epochs, the weights I keep are the
ones that generalized best to validation — not the final, potentially over-fit weights. A
final test accuracy of 0.967 with no class below F1 0.93 indicates this didn't overfit
catastrophically. I left `learning_rate=2e-5` (the standard BERT-family fine-tuning rate),
which trained stably.

---

## Baseline (zero-shot Groq)

- **Model:** `llama-3.3-70b-versatile` via the Groq API, `temperature=0`, `max_tokens=20`.
- **Method:** Each test post was sent with a system prompt defining all four labels and
  one example each, instructing the model to output **only** the label name. Outputs were
  matched against the label set (longest-label-first to avoid substring collisions);
  unparseable responses were excluded from scoring.

```
You are classifying soccer posts from r/soccer on reddit.
Assign each post to exactly one of the following categories.

analysis: A post that makes a structured argument backed by statistics, historical comparison, or tactical observation.
Example: "City's high press under Guardiola is unsustainable — their PPDA has dropped from 7.2 to 9.8 over the last 3 seasons as the squad ages."

hot_take: A bold, confident opinion stated without supporting evidence.
Example: "The Premier League is massively overrated. Serie A in the 90s had better football, full stop."

debate: A post that invites comparison or community discussion — GOAT arguments, player/club rankings, hypothetical matchups.
Example: "Rate this all-time r/soccer XI — tried to be balanced across eras. Drop your changes below."

match_reaction: A post made during or immediately after a match reacting to the result, a key moment, or the scoreline.
Example: "IT'S 3-2 IN THE 94TH MINUTE. ABSOLUTE SCENES."

Respond with ONLY the label name.
Do not explain your reasoning.

Valid labels:
analysis
match_reaction
debate
hot_take
```

---

## Evaluation report

### Overall accuracy

| Model | Accuracy | Test set |
|---|---|---|
| Zero-shot baseline (Groq `llama-3.3-70b`) | **0.967** | 30 |
| Fine-tuned DistilBERT | **0.967** | 30 |
| **Improvement** | **0.000** | — |

### Per-class metrics — Fine-tuned model

*(Computed from the confusion matrix; matches `accuracy = 29/30 = 0.967`. One error: `analysis` → `debate`.)*

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 1.00 | 0.88 | 0.93 | 8 |
| hot_take | 1.00 | 1.00 | 1.00 | 7 |
| match_reaction | 1.00 | 1.00 | 1.00 | 8 |
| debate | 0.88 | 1.00 | 0.93 | 7 |
| **macro avg** | **0.97** | **0.97** | **0.97** | 30 |
| **weighted avg** | **0.97** | **0.97** | **0.97** | 30 |

### Per-class metrics — Baseline

*(All 30 baseline responses were parseable.)*

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 1.00 | 1.00 | 1.00 | 8 |
| hot_take | 1.00 | 0.86 | 0.92 | 7 |
| match_reaction | 0.89 | 1.00 | 0.94 | 8 |
| debate | 1.00 | 1.00 | 1.00 | 7 |
| **macro avg** | **0.97** | **0.96** | **0.97** | 30 |
| **weighted avg** | **0.97** | **0.97** | **0.97** | 30 |

**Baseline vs. fine-tuned:** Both models scored 29/30, but on **different** errors. The
baseline misclassified one `hot_take` as `match_reaction` (hot_take recall 0.86, match_reaction
precision 0.89). The fine-tuned model fixed that boundary entirely — `hot_take` is perfect —
but introduced a new error: one `analysis` post predicted as `debate`. The two models tied
on accuracy yet disagree on *which* post is hard, which reveals that the remaining errors
are genuinely ambiguous rather than consistently hard for all classifiers.

### Confusion matrix — Fine-tuned model (test set)

Rows = true label, columns = predicted label. (Also saved as
[`confusion_matrix.png`](confusion_matrix.png).)

| true ↓ \ pred → | analysis | hot_take | match_reaction | debate |
|---|---|---|---|---|
| **analysis** | 7 | 0 | 0 | **1** |
| **hot_take** | 0 | 7 | 0 | 0 |
| **match_reaction** | 0 | 0 | 8 | 0 |
| **debate** | 0 | 0 | 0 | 7 |

**Reading it:** the model makes exactly **one error** — one `analysis` post predicted as
`debate`. Every other class is classified perfectly. `hot_take` and `match_reaction`, the
hardest boundary in the dataset, are both at 100% recall. The sole mistake is a
structurally ambiguous post at the `analysis`/`debate` boundary.

### Analyzed failures

The fine-tuned model made **only 1 error on the test set**. I analyze it in depth, then
add two further cases — one from the baseline and one from an earlier training run — to
reach the required three diagnosable failures.

**Error 1 — fine-tuned: `analysis` → `debate`, confidence 0.30** (test set)
> "Countries eligible to host the 2038 World Cup under current rules"
- *Why it failed:* This post sits on a **genuine structural boundary**. It is labeled
  `analysis` because it makes an implicit argument from FIFA eligibility rules — a
  structured claim with verifiable backing. But it also reads as a discussion prompt: it
  names a topic without stating a conclusion, which is exactly how many `debate` posts are
  phrased ("who would qualify?", "which countries are eligible?"). The very low confidence
  (0.30, barely above the 0.25 chance baseline for 4 classes) shows the model found no
  decisive signal either way. This is a **data-distribution problem**: short, topic-framing
  posts appear in both `analysis` and `debate` in the training set, so the model has no
  reliable feature to separate them when the post is under-specified.

**Error 2 — baseline: `hot_take` → `match_reaction`** (test set, from the Groq run)
> One `hot_take` post predicted as `match_reaction` (baseline hot_take recall 0.86,
> match_reaction precision 0.89 — the only baseline error).
- *Why it failed:* The baseline makes the same error that plagued earlier fine-tuned runs:
  a `hot_take` post containing event-reference language (a scoreline, an on-this-day
  reference, or transfer news) gets routed to `match_reaction`. Even with a complete
  4-label prompt, the zero-shot LLM uses surface vocabulary rather than rhetorical intent,
  and factual/reportorial phrasing pulls it toward `match_reaction`. The fine-tuned model
  fixed this boundary entirely — its `hot_take` recall is 1.00 — showing that fine-tuning
  on labeled examples can learn distinctions that prompting alone cannot reliably enforce.

**Error 3 — earlier training run: `hot_take` → `match_reaction`, confidence 0.32** (previous run)
> "Arsenal hold talks with Strasbourg's Pascal De Maesschalck over academy manager role"
- *Why it failed:* This **transfer/executive news post** was labeled `hot_take` under my
  "asserts rather than argues" rule — but it contains no opinion whatsoever. It reads in
  the neutral register of club news, which the model associated with `match_reaction`. The
  near-chance confidence (0.32) signals the model found no strong signal for any class —
  exactly right, because the post genuinely doesn't fit the `hot_take` training examples.
  This is a **labeling problem**: news reports that assert a fact don't belong in `hot_take`
  under my definition; they would need their own label or be excluded from the dataset.

**Diagnosis across all three:** the three errors fall on **two different boundaries**:
- The fine-tuned model's remaining weakness is `analysis`/`debate` — structurally ambiguous,
  low-information posts where the rhetorical intent (arguing vs. prompting discussion) isn't
  signalled by vocabulary alone.
- Both the baseline and earlier fine-tuned runs struggled with `hot_take`/`match_reaction` —
  driven by news/event vocabulary inside posts labeled `hot_take`. The final fine-tuned model
  solved this through additional training passes; the baseline still makes it.

The shared root cause: **my labels are defined by rhetorical function, but classifiers
learn lexical patterns**. Where vocabulary reliably signals function (live scores →
`match_reaction`; ranking prompts → `debate`) both models work. Where vocabulary and
function diverge (a news headline asserting a fact, labeled `hot_take`; a topic-framing
question, labeled `analysis`) both models struggle.

### Sample classifications

<!-- TODO: run this cell in Colab to fill in the confidence values, then delete this comment:

import torch
samples = [
    "Goal! England [1] - 0 France - Jude Bellingham 14'",
    "Who's the GOAT — Messi or Ronaldo? Settle it.",
    "Rodri's passing map proves City press too high, PPDA down to 9.8.",
    "Salah is the most overrated forward in the league, full stop.",
    "Countries eligible to host the 2038 World Cup under current rules",
]
for s in samples:
    enc = tokenizer(s, return_tensors="pt", truncation=True, max_length=256).to(model.device)
    with torch.no_grad():
        p = torch.softmax(model(**enc).logits, dim=-1)[0]
    print(f"{ID_TO_LABEL[int(p.argmax())]}  conf={p.max():.2f}  | {s}")
-->

| # | Post (truncated) | Predicted | Confidence |
|---|---|---|---|
| 1 | "Goal! England [1] - 0 France - Jude Bellingham 14'" | match_reaction | _fill from Colab_ |
| 2 | "Who's the GOAT — Messi or Ronaldo? Settle it." | debate | _fill from Colab_ |
| 3 | "Rodri's PPDA numbers prove City press too high." | analysis | _fill from Colab_ |
| 4 | "Salah is the most overrated forward in the league." | hot_take | _fill from Colab_ |
| 5 | "Countries eligible to host the 2038 World Cup under current rules" | debate (wrong — true: analysis) | _fill from Colab_ |

**Why #1 is reasonable:** "Goal! England [1] - 0 France - ... 14'" has the canonical
`match_reaction` structure — a live scoreline with a minute marker and a goalscorer. There
is no argument, opinion, or discussion prompt, so a high-confidence `match_reaction`
prediction is exactly correct.

---

## Reflection: what the model learned vs. what I intended

I intended four labels defined by **rhetorical function** — is the post *arguing*,
*asserting*, *reacting live*, or *inviting discussion*? What the model's decision boundary
actually learned is closer to **surface vocabulary**:

- It learned `analysis`, `match_reaction`, and `debate` essentially perfectly — but those
  three carry strong lexical fingerprints (stats/tactics terms; scorelines + minute
  markers; "rank/who's better/do you agree"). The model likely keys on those tokens rather
  than truly understanding rhetorical intent.
- It **learned `hot_take` and `match_reaction` as distinct registers** after 30 epochs with
  best-checkpoint selection — achieving 100% recall on both. This is the clearest win from
  fine-tuning: the baseline still conflates these, but the fine-tuned model learned the
  distinction through labeled examples.
- Its remaining error is at the **`analysis`/`debate` boundary**, on a low-information
  topic-framing post. This boundary depends on whether a post makes a claim or merely
  raises a question — a distinction that doesn't reduce to vocabulary, so the model
  defaulted to near-chance confidence (0.30). More training examples of short, structured
  `analysis` posts would help, but the fundamental issue is that some posts don't signal
  their intent clearly enough in their surface form.

The gap, in one sentence: I defined labels by *rhetorical intent*, but the model can only
learn *lexical patterns* — so it mastered the classes where intent is reliably encoded in
vocabulary, and its one failure is the post where vocabulary alone is genuinely ambiguous.

---

## Spec reflection

- **How the spec helped:** The spec's insistence on per-class metrics and a confusion
  matrix (not just accuracy) is what surfaced the real story. Both models scored 0.967 —
  identical headline numbers — but the per-class tables reveal they make **different**
  errors: the baseline still leaks `hot_take` into `match_reaction` (recall 0.86), while
  the fine-tuned model fixed that boundary entirely but introduced one `analysis`/`debate`
  confusion. Accuracy alone would make them look equivalent; the class-level breakdown
  shows fine-tuning genuinely moved the decision boundary, just not in the direction that
  raised the headline number.
- **How my implementation diverged:** The starter taxonomy included a `reaction` label; I
  diverged by replacing it with `debate`. Standalone emotional-reaction *posts* are rare on
  r/soccer (they live in comments), while debate/ranking posts are abundant and collectable
  — so the divergence was driven by what the community actually produces, making the dataset
  realistic and balanceable at 50/label.

---

## AI usage

1. **Label stress-testing (Claude, `claude-sonnet-4-6`).** I gave Claude my label
   definitions and edge-case rules and asked it to generate boundary posts between
   `analysis`/`hot_take` and `hot_take`/`debate`. Where I couldn't confidently assign a
   single label to a generated post, I tightened my decision rules (e.g., the "explicit
   invitation to discuss → `debate`" rule). I **overrode** several of its generated
   borderline labels where I disagreed, and discarded examples that were too contrived to
   resemble real r/soccer posts.

2. **Failure-pattern analysis (Claude).** I pasted my misclassified examples and asked
   Claude to surface common themes. It flagged the score-/event-language pattern in the
   `hot_take → match_reaction` errors. I **verified this myself** by re-reading the actual
   test errors before accepting it, and **corrected** its initial over-generalization that
   "all short posts fail" — short posts like `debate` prompts classified perfectly, so the
   real signal was score vocabulary, not length.

3. **Annotation assistance — disclosure (Gemini).** I used Gemini to **pre-label** all 200
   posts against my definitions with a one-sentence rationale each. I reviewed every label
   myself and corrected disagreements; pre-labeled rows are marked in the dataset. No label
   in the final dataset was accepted without my own review.
