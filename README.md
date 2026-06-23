# ai201-project3-takemeter
# TakeMeter — NBA Discourse Classifier
A fine-tuned text classifier that categorizes r/nba posts into three discourse types: **analysis**, **hot_take**, and **reaction**. Built for AI201 Project 3.

---

## Community Choice

I chose r/nba because it's one of the most active sports communities on Reddit and produces a genuinely wide range of discourse quality in real time. During any playoff run you'll see all three post types within the same thread — someone citing on/off defensive stats, someone declaring a player overrated with zero evidence, and someone screaming in all-caps about the final buzzer. That variation makes it a strong fit for a classification task: the labels are meaningful to people who actually participate in the community, and the distinctions map onto something real about how NBA fans talk.

The community also produces enough text-heavy posts that labeling is tractable — most posts are 1-4 sentences, long enough to carry signal but short enough to annotate quickly.

---

## Label Taxonomy

**analysis** — The post makes a claim and supports it with specific, verifiable evidence: statistics, historical comparisons, tactical observations, or structured logical reasoning. The evidence must carry the argument, not just decorate it.

> Example 1: "When Chet was on the court opponents shot 57.84% at the rim vs 64.31% when he was off — that on/off gap is larger than Gobert's, which is why the Chet snub is hard to justify."
> Example 2: "The Knicks are only the third team in NBA history to win a championship without a player who received MVP votes, joining the 77-78 Bullets and 67-68 Celtics."

**hot_take** — A bold, confident opinion stated without meaningful supporting evidence. The claim may be plausible but the post asserts rather than argues. Provocative framing designed to invite pushback.

> Example 1: "Jokic fans are in hiding now but in a month they'll say he's better than prime LeBron."
> Example 2: "Top 10 player ever but has been out of the second round twice in 12 years. It's crazy how much leeway this guy gets."

**reaction** — An immediate emotional response to a specific recent event. Little to no argument. Often short, all-caps, or exclamatory. The primary intent is expressing a feeling, not making a point.

> Example 1: "MY GUY, MICHAEL JORDAN CLARKSON IS AN NBA CHAMPION"
> Example 2: "I was crying as a 9 year old kid in 1994. I AM SO FUCKING HAPPY right now."

---

## Data Collection

**Source:** Posts and comments collected manually from r/nba during the 2026 NBA Finals (Knicks vs Spurs) and surrounding playoff threads.

**Process:** I collected 200 raw posts, then cleaned the dataset down to 132 usable examples after removing duplicates, posts too short to label confidently, and non-English posts. Each post was saved with a `text` and `label` column. I used Groq's llama-3.3-70b-versatile to pre-label batches before reviewing every label myself, tracking corrections in a `corrected` column.

**Label distribution:**

| Label | Count |
|---|---|
| reaction | 66 |
| analysis | 37 |
| hot_take | 29 |
| **Total** | **132** |

**Three difficult-to-label examples:**

**1.** *"The Spurs were something like 28 and 0 when they shot 40% from distance. When they're hitting their shots they're literally unbeatable."*
This could be analysis (uses a record stat) or hot_take (the stat is approximate and the conclusion is hyperbolic). I labeled it **analysis** because the stat is being used to build a conditional argument about team identity, not just to punch up an opinion.

**2.** *"Top 10 player ever but has been out of the second round twice in his 12 year career. It's crazy how much leeway this guy gets."*
This has a verifiable stat (second round exits) but no logical chain — it's using the stat rhetorically to support a dismissive claim. I labeled it **hot_take** per my decision rule: one stat with no argument structure = hot_take.

**3.** *"Incredible comebacks by the Knicks all series. Youth and inexperience of Spurs showed time and time again."*
This starts as a reaction to the series result but adds a tacked-on explanation with no supporting evidence. I labeled it **reaction** per my decision rule: reaction + unsupported opinion = still reaction.

---

## Fine-Tuning Approach

**Base model:** distilbert-base-uncased (HuggingFace)

**Training setup:** Fine-tuned using HuggingFace Trainer on Google Colab T4 GPU. Dataset split 70/15/15 (train/val/test), stratified by label.

**Hyperparameter decision:** I kept the default learning rate of 2e-5 and 3 epochs, which are standard starting points for fine-tuning BERT-family models on small datasets. I considered increasing epochs to 5 to give the model more passes over the training data, but with only 92 training examples the risk of overfitting outweighed the potential gain. The validation accuracy plateaued by epoch 2, confirming 3 epochs was appropriate.

---

## Baseline

**Model:** Groq llama-3.3-70b-versatile (zero-shot, no task-specific training)

**Prompt used:**
You are a data annotator for an NBA discourse classification project.

Your job is to assign each r/nba post to exactly one of three categories.
LABEL DEFINITIONS:
analysis: The post makes a claim and supports it with specific, verifiable

evidence — statistics, historical comparisons, tactical observations, or

structured logical reasoning. The evidence must carry the argument, not just

decorate it. A post with one cherry-picked stat used to punch up an opinion

(without building a logical chain) is NOT analysis.
hot_take: A bold, confident opinion stated without meaningful supporting

evidence. The claim may be plausible but the post asserts rather than argues.

Provocative framing, designed to invite pushback. Includes posts with one

isolated stat used rhetorically rather than as part of a real argument.
reaction: An immediate emotional response to a specific recent event — a game,

play, trade, or breaking news. Little to no argument. Often short, all-caps,

or exclamatory. The primary intent is expressing a feeling, not making a point.
DECISION RULES FOR HARD CASES:

If a post has one stat but no logical chain → hot_take
If a post is emotional but builds a real argument with evidence → analysis
If a post starts as a reaction but adds a tacked-on opinion with no support → reaction

Respond with ONLY the label. One word. No punctuation. No explanation.
Valid labels:

analysis

hot_take

reaction

**How results were collected:** Each test example was sent to the Groq API individually using `temperature=0` and `max_tokens=20`. The model's response was stripped, lowercased, and matched against the label list. All 20 responses were parseable — 0 unparseable outputs.

**Baseline result:** 0.600 accuracy on the 20-example test set. The baseline performed strongest on `reaction` (F1 0.90), which makes sense — the zero-shot model could apply the label definitions directly from the prompt and reaction posts have strong stylistic signals. It completely failed on `analysis` (F1 0.00), collapsing most analysis posts into `hot_take`. This tells us the analysis/hot_take boundary is genuinely hard to apply from a text description alone.

---

## Evaluation Report

### Model Performance

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq llama-3.3-70b-versatile) | 0.600 |
| Fine-tuned DistilBERT | 0.400 |

Fine-tuning hurt performance by 0.200 — the baseline outperformed the fine-tuned model on this test set.

---

### Per-Class Metrics

**Baseline (Groq):**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 0.00 | 0.00 | 0.00 | 6 |
| hot_take | 0.30 | 0.75 | 0.43 | 4 |
| reaction | 0.90 | 0.90 | 0.90 | 10 |

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 0.50 | 0.17 | 0.25 | 6 |
| hot_take | 0.31 | 1.00 | 0.47 | 4 |
| reaction | 0.60 | 0.30 | 0.40 | 10 |

---

### Confusion Matrix (Fine-Tuned Model)

| | Predicted: analysis | Predicted: hot_take | Predicted: reaction |
|---|---|---|---|
| **True: analysis** | 1 | 3 | 2 |
| **True: hot_take** | 0 | 4 | 0 |
| **True: reaction** | 1 | 6 | 3 |

---

### Wrong Predictions — Error Analysis

**#1 — analysis predicted as hot_take**
> "Wemby competed against the US team in a great game with the second best player being Nicolas Batum. Wemby is playoff ready. So is Fox, Kornet and Barnes. More importantly the Spurs organization is ready."

This post builds a structured argument using a specific game as evidence, which should make it analysis. The model likely flagged it as hot_take because the conclusion ("Wemby is playoff ready") reads as a bold assertion. The evidence supporting the claim is present but subtle — the model never learned to distinguish decorative confidence from evidence-backed reasoning.

**#2 — reaction predicted as hot_take**
> "IT SHOULD HAVE BEEN IN THE GARDEN F*** YOU *****""

This is a clear emotional reaction — all-caps, expletive, responding to a specific external event. The model predicted hot_take, likely because the post contains a direct accusation, which superficially resembles provocative framing. The model appears to have learned that anger = hot_take rather than distinguishing between emotional venting (reaction) and opinion assertion (hot_take).

**#3 — analysis predicted as hot_take**
> "The Spurs were something like 28 and 0 when they shot 40% from distance. When they're hitting their shots they're literally unbeatable. When they're not you can crash the paint..."

This post uses a specific record stat to build a tactical argument about the Spurs' shooting dependency. The model predicted hot_take, likely because the phrase "literally unbeatable" reads as hyperbolic assertion. This is the exact edge case my decision rules were designed for: a stat used as part of a logical chain should be analysis, not hot_take. The model never learned this boundary.

---

### Sample Classifications

| Post (truncated) | True Label | Predicted | Confidence |
|---|---|---|---|
| "When Chet was on the court opponents shot 57.84% at the rim..." | analysis | analysis | 0.50 |
| "MY GUY, MICHAEL JORDAN CLARKSON IS AN NBA CHAMPION" | reaction | reaction | 0.62 |
| "Jokic fans are in hiding now but in a month they will say he is better than prime LeBron" | hot_take | hot_take | 0.34 |
| "LETS GO" | reaction | hot_take | 0.35 |
| "Top 10 player ever but has been out of the second round twice in his 12 year career" | analysis | reaction | 0.34 |

The correctly predicted reaction post ("MY GUY, MICHAEL JORDAN CLARKSON...") is reasonable — the all-caps format and celebratory tone are strong stylistic signals the model did pick up on consistently.

---

### What the Model Learned vs. What I Intended

I intended the model to learn the structural distinction between three post types: evidence-backed arguments (analysis), unsupported confident opinions (hot_take), and emotional in-the-moment responses (reaction).

What it actually learned was a much shallower pattern: **when uncertain, predict hot_take**. Every wrong prediction has a confidence of ~0.34 — essentially random for a 3-class problem — and the confusion matrix shows hot_take being predicted 13 out of 20 times. The model correctly learned that hot_take is the "middle ground" label but never internalized the boundaries that separate it from analysis (evidence chains) or reaction (emotional framing).

The root cause is data size. With only 92 training examples split across 3 subjective labels, DistilBERT did not have enough signal to learn the nuanced linguistic features that distinguish these categories. The baseline zero-shot model outperformed it because Groq's llama-3.3-70b-versatile could apply the label definitions directly from the prompt — it didn't need to learn the pattern from examples.

---

## Spec Reflection

**One way the spec helped:** The spec's emphasis on writing label definitions precise enough that "two people reading them would agree on most examples" forced me to add the decision rules section to my prompt and planning.md before annotating. Without that pressure I would have labeled the analysis/hot_take boundary inconsistently — posts like the Spurs shooting stat example would have gone either way. Having the explicit rule ("one stat with no logical chain → hot_take") made my annotation more consistent even if the model ultimately couldn't learn that boundary.

**One way implementation diverged:** The spec assumes fine-tuning on 200 examples will produce a model that meaningfully beats the zero-shot baseline. My implementation diverged from that expectation — the fine-tuned model scored 0.40 vs the baseline's 0.60. The divergence happened because my dataset ended up at 132 examples after cleaning, and because the three labels I chose are genuinely hard to distinguish from surface-level text features alone. The harder real problem was whether 132 examples was enough signal for DistilBERT to learn anything at all. It wasn't.

---

## AI Usage

**Instance 1 — Annotation pre-labeling:** I used Groq's llama-3.3-70b-versatile to pre-label batches of posts before reviewing them myself. I provided the model my label definitions and decision rules from planning.md and asked it to assign one label per post. I then reviewed every pre-assigned label and corrected ones I disagreed with, tracking corrections in a `corrected` column in my CSV. The model struggled most on the same boundary my fine-tuned model later failed on: posts with one rhetorical stat that could read as either analysis or hot_take. I overrode roughly 15-20% of pre-labels, mostly in that category.

**Instance 2 — Label stress-testing:** Before finalizing my label definitions I pasted them into Claude and asked it to generate 10 posts that would sit at the boundary between analysis and hot_take. Several of the generated examples I genuinely couldn't classify cleanly, which confirmed that the boundary needed a more explicit decision rule. This directly led to adding "a post with one cherry-picked stat used to punch up an opinion is NOT analysis" to my label definitions. Without that stress-test I would have discovered the ambiguity during annotation instead of before it.

**Instance 3 — Failure pattern analysis:** After getting my wrong predictions from Section 4, I pasted them into Claude and asked it to identify common patterns. It flagged two things I verified myself: (1) the model was over-predicting hot_take across the board, and (2) most reaction misclassifications involved emotional posts that also contained a named person or opinion, which the model treated as opinion-bearing rather than emotionally expressive. Both patterns held up when I re-read the examples manually.

 
 
