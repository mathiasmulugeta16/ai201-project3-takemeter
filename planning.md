# Community: What community did you choose and why? Why is this community a good fit for a classification task — what makes the discourse varied enough to be interesting?
R/ SOCCER: I used this community cause it has a lot of measureable post like 
anaylitics, hot takes, reactions and such. Aside from that it also is something i connect with and can understand outputs provided by the llm more.

# Labels: What are your 2–4 labels? Define each in a complete sentence. Include 2 example posts per label.
I used the provided examples as they resonate with my chosen community. I replaced `reaction` with `debate` because pure emotional reactions are rare as standalone r/soccer posts — they mostly live in comments — while GOAT threads, player rankings, and comparison posts are extremely common and easy to collect.

- **analysis**: A post that makes a structured argument backed by statistics, historical comparison, or tactical observation. Evidence is specific and verifiable.
  - Example 1: "City's high press under Guardiola is unsustainable — their PPDA has dropped from 7.2 to 9.8 over the last 3 seasons as the squad ages."
  - Example 2: "Rodri's passing map from last night shows how he shifts between a pivot and a 6 depending on whether Bernardo is in the half-space. Thread with stats below."

- **hot_take**: A bold, confident opinion stated without supporting evidence. The claim might be true, but the post asserts rather than argues.
  - Example 1: "Haaland is not a top 5 player in the world. He only scores because he's fed by elite creators."
  - Example 2: "The Premier League is massively overrated. Serie A in the 90s had better football, full stop."

- **debate**: A post that invites comparison or community discussion — GOAT arguments, player/club rankings, hypothetical matchups. It poses a question or comparison rather than asserting one or arguing with evidence.
  - Example 1: "Who was the better player: peak Ronaldinho or peak Zidane? Genuine debate."
  - Example 2: "Rate this all-time r/soccer XI — tried to be balanced across eras. Drop your changes below."

- **match_reaction**: A post made during or immediately after a match reacting to the result, a key moment, or the scoreline. The focus is the live or just-finished event, not a broader argument or discussion prompt.
  - Example 1: "IT'S 3-2 IN THE 94TH MINUTE. ABSOLUTE SCENES."
  - Example 2: "FT: Liverpool 4-0 United. That second half was embarrassing, couldn't believe what I was watching."



# Hard edge cases: What type of post will be genuinely ambiguous between two labels? How will you handle it when you encounter it during annotation?
Two boundary pairs are genuinely ambiguous:

- **analysis vs hot_take**: Some posts cite a single stat or comparison purely for rhetorical punch without building a real argument. Decision rule: if the post presents one data point to assert a conclusion rather than to examine it, label it `hot_take`.
- **hot_take vs debate**: A post can state a bold opinion and then ask "do you agree?" Decision rule: if the post includes an explicit invitation for others to weigh in (a question, a poll, "change my mind"), label it `debate` — the invitation to discuss is the defining feature.
- **match_reaction vs hot_take**: A match reaction post can sound like a hot take ("that goalkeeper is terrible"). Decision rule: if the post is clearly anchored to a specific live or just-finished match (score mentioned, match referenced, time-sensitive language), label it `match_reaction`. A timeless bold claim with no match anchor is `hot_take`.


# Data collection plan: Where will you collect examples? How many per label? What will you do if a label is underrepresented after 200 examples?
I plan on collecting data from reddit. A total of 200 posts and around 50 per label across the four labels: analysis, hot_take, debate, and match_reaction.


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




# Definition of success: What performance would make this classifier genuinely useful? What would you accept as "good enough" for deployment in a real community tool?

 I'll consider it good enough for deployment if weighted F1 ≥ 0.75 and no single label scores below F1 = 0.65. If reaction falls below that floor, I'll collect more examples before deploying."

# AI Tool Plan 

## Label stress-testing
I will give Claude (claude-sonnet-4-6) my three label definitions (analysis, hot_take, debate) along with the edge case description — posts that use stats for rhetorical punch, and posts that state an opinion but also invite discussion. I'll ask it to generate 8–10 r/soccer posts that sit at the boundary between analysis/hot_take and hot_take/debate. If I cannot confidently assign a single label to a generated post, that means my definitions are ambiguous and I'll tighten the decision rules. This stress-test happens before I annotate a single real example.

## Annotation assistance
Yes — I used Gemini to pre-label posts before reviewing them myself. I provided it my four label definitions (analysis, hot_take, debate, match_reaction) and my edge case decision rules, and asked it to assign one label per post with a one-sentence rationale. I then reviewed every pre-assigned label myself and corrected any I disagreed with. All pre-labeled examples are marked with a `prelabeled: true` column in the CSV. I did not skip or skim the review — every label in the final dataset reflects my own judgment after seeing Gemini's suggestion.

## Failure analysis
After training, I'll collect all misclassified posts (predicted label ≠ true label) and give the full list to Claude with the prompt: "Identify patterns in why these posts were mislabeled — look for recurring linguistic features, ambiguous phrasing, or label definition gaps." I'll specifically look for: (1) posts where emotional language co-occurs with a stat (analysis/hot_take confusion), (2) posts that state an opinion and invite replies, mislabeled as hot_take instead of debate, and (3) short comparative posts that straddle hot_take and debate. I'll verify each pattern Claude surfaces by manually reading at least 5 examples of that pattern myself before accepting it as real.