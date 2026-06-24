# Community: What community did you choose and why? Why is this community a good fit for a classification task — what makes the discourse varied enough to be interesting?
R/ SOCCER: I used this community cause it has a lot of measureable post like 
anaylitics, hot takes, reactions and such. Aside from that it also is something i connect with and can understand outputs provided by the llm more.

# Labels: What are your 2–4 labels? Define each in a complete sentence. Include 2 example posts per label.
I used the provided examples are they resonate with my chosen community, the labels are:
- analysis: the posts makes a strctured argument backed by statistics, historical comparision or tacticatl observation. Evidence is specific and verifiyable.
- hot_take: a bold, confident opinion stated without supporting evidence. The claim might be true, but the post asserts rather than argues.
- reaction: an immeditate emotional response to a specific event. Little to no argument. The post is expressing a feeling in the moment.



# Hard edge cases: What type of post will be genuinely ambiguous between two labels? How will you handle it when you encounter it during annotation?
I'd say some posts use comparasion or stats to provoke a reaction or make a hot_take. For that I plan on making a decision rule.


# Data collection plan: Where will you collect examples? How many per label? What will you do if a label is underrepresented after 200 examples?
I plan on collection data from reddit. A total of 200 posts and around 40 per label


# valuation metrics: Which metrics will you use to evaluate your model and why are those the right ones for this specific task? (Accuracy alone is not enough — explain what else you need and why.)

Evaluation checklist:
- **Per-class precision:** Fraction of predicted positives that are correct — controls false alarms.
- **Per-class recall:** Fraction of true positives found — controls missed items.
- **Per-class F1; Macro & Weighted F1:** F1 balances precision and recall. Use **macro F1** to treat labels equally and **weighted F1** to account for class imbalance.
- **Confusion matrix:** Shows which labels are confused with which — guides label-definition fixes and targeted improvements.
- **Class support (counts):** Report counts per label so metric estimates are interpretable (low support → high variance).
- **AUC-PR / Precision-Recall curves (one-vs-rest):** Prefer over ROC AUC for imbalanced classes and when positive-class performance matters.
- **Calibration / reliability checks:** If you use predicted probabilities or thresholds, verify the model is well-calibrated.
- **Annotation agreement (Cohen's kappa or Krippendorff's alpha):** Measures labeler consistency and sets an approximate upper bound for model performance.
- **Qualitative error analysis:** Manual review of typical failure cases to identify patterns and guide label or data fixes.

Practical note: report per-class precision/recall/F1 plus macro and weighted F1, show the confusion matrix and supports, and run focused failure-analysis on the most common confusions.



# Definition of success: What performance would make this classifier genuinely useful? What would you accept as "good enough" for deployment in a real community tool?


# AI Tool Plan 

# Label stress-testing: Give the AI your label definitions and edge case description, and ask it to generate 5–10 posts that sit at the boundary between two labels. If it produces posts you can't classify cleanly, your definitions need tightening — do that now, before you annotate 200 examples.

# Annotation assistance: Decide whether you'll use an LLM to pre-label a batch of examples before reviewing them yourself. If yes, note which tool you'll use and how you'll track which examples were pre-labeled (for disclosure in your AI usage section).


# Failure analysis: Plan to give your list of wrong predictions to an AI tool and ask it to identify patterns before you write up your evaluation. Note what you'll look for and how you'll verify the patterns yourself.