1. Community

I chose r/nba, the NBA subreddit, because it is one of the most active sports discourse communities on the internet and its conversations naturally span a wide quality range — from rigorous statistical breakdowns to gut-reaction hype to pure emotional noise. That variance is exactly what makes it interesting for a classification task: unlike a community where all posts sound roughly the same, r/nba produces posts that are genuinely different in structure, intent, and evidentiary grounding.

The community is also a good fit because the distinction between "good take" and "bad take" is something r/nba members themselves actively argue about. Phrases like "this is just a hot take," "where's your data," and "stats don't tell the full story" appear constantly in comment sections. That means my labels aren't invented from the outside — they map onto distinctions the community already makes.

Finally, r/nba generates an enormous volume of public text across game threads, trade discussions, player debates, and weekly free-talk posts, making it easy to collect 200+ diverse examples without scraping private or restricted content.


2. Label Taxonomy

analysis

Definition: The post makes a claim and supports it with specific, verifiable evidence — statistics, historical comparisons, tactical observations, or structured logical reasoning — where the evidence carries the argument rather than just decorating it.

Examples:


"Jokic's assist-to-turnover ratio this playoffs (4.8) is the highest for a center in the last 20 years. He's effectively operating as a point guard and defenses still have no answer because you can't send a big to guard him at the 3-point line without leaving shooters open."
"The Celtics' halfcourt offense ranks 3rd in the league overall but drops to 18th in the last 5 minutes of close games. That's not a roster depth problem — it's a late-clock iso-heavy tendency from Tatum that collapses spacing."



hot_take

Definition: A bold, confident opinion stated without meaningful supporting evidence; the claim may be plausible, but the post asserts rather than argues, and the framing is often provocative or designed to invite pushback.

Examples:


"LeBron is overrated. Always has been. Jordan wins those Finals every single time, full stop."
"The Suns should blow it up entirely. Booker is not a number one on a championship team and everyone knows it."



reaction

Definition: An immediate emotional response to a specific, recent event — a game result, a highlight play, a trade, or breaking news — where the post expresses a feeling in the moment with little to no argument or analysis.

Examples:


"CURRY WHAT THE HELL WAS THAT SHOT oh my god"
"I can't believe they actually traded him. This season is completely over. I'm done."



3. Hard Edge Cases

The stat-supported hot take

The hardest boundary to hold is between analysis and hot_take when a post includes one specific statistic. At first glance it looks like evidence; on closer reading, it's a rhetorical move.

Example ambiguous post:


"LeBron is overrated — his playoff win rate against top-seeded opponents is below .500."



This could be analysis (cites a stat) or hot_take (bold accusatory claim, cherry-picked number).

Decision rule: If a post provides one isolated stat to punch up an opinion but does not build a logical chain from evidence → reasoning → conclusion, label it hot_take. analysis requires the evidence to carry the argument. A single cherry-picked stat that could be removed without weakening an already-declarative statement is decoration, not evidence.

The passionate-but-argued post

A post can be emotionally charged and still be analysis if it builds a real argument. Tone alone does not make something a hot_take.

Decision rule: If the post is angry but provides structured reasoning with specific evidence, label it analysis. Emotion + evidence = analysis. Emotion + assertion = hot_take.

The reaction with a mini-opinion

A game-thread post might start as a pure reaction but tack on a one-sentence opinion at the end.

Decision rule: Label by the primary intent. If most of the post is an immediate emotional response and the opinion is tacked on without support, it's reaction. If the opinion is the main point and the emotional framing is just delivery, apply hot_take or analysis as appropriate.


4. Data Collection Plan

Sources:


r/nba game threads (top-level comments and replies) → primary source for reaction
r/nba "Discussion" and "Analysis" flair posts → primary source for analysis
r/nba "Unpopular opinion," "Hot take," and player debate threads → primary source for hot_take
Top comment sections on controversial trade or MVP posts → mixed source for all three


Collection method: Manual copy-paste into a Google Sheet with columns: text, label, notes. I will read each post in context before labeling. I will not use a scraper so I stay close to the data.

Target distribution:

LabelTarget countanalysis70hot_take70reaction60Total200

If a label is underrepresented after 200 examples: I will specifically seek out posts with that label rather than continuing to collect randomly. For example, if analysis is lagging, I will go directly to pinned analysis threads or player-efficiency discussion posts. I will not proceed to annotation until no single label exceeds 70% of the dataset.

Data I will exclude: Memes, image-only posts, posts under 5 words, posts that are replies-to-replies with no standalone meaning, and any posts from private or locked threads.


5. Evaluation Metrics

Overall accuracy will be reported but is not sufficient on its own for this task. With three labels, a model that always predicts hot_take would achieve 33% accuracy without learning anything — and a mildly imbalanced dataset could push that higher. Accuracy doesn't tell me which label is failing.

Per-class F1 is the primary metric. F1 is the harmonic mean of precision and recall, and it captures the tradeoff between false positives and false negatives for each label independently. I need to know if the model is failing on a specific label (e.g., consistently misclassifying analysis as hot_take) — per-class F1 exposes that where overall accuracy hides it.

Precision and recall per class will be reported separately because the failure modes are asymmetric. For a community moderation tool, false positives on reaction (labeling thoughtful posts as reactions) are more damaging than false negatives, so I need to see precision and recall independently.

Confusion matrix will be included to identify which specific label pairs the model confuses. A directional confusion (e.g., analysis → hot_take more than hot_take → analysis) tells me something specific about which boundary the model failed to learn.

Baseline comparison: I will report all of the above for both the fine-tuned model and the Groq zero-shot baseline. The delta in per-class F1 tells me whether fine-tuning actually helped on the hard labels, not just the easy ones.


6. Definition of Success

Minimum threshold for "good enough":


Fine-tuned model overall accuracy ≥ 0.70
Per-class F1 ≥ 0.60 for all three labels (no label left behind)
Fine-tuned model outperforms Groq baseline by at least 0.05 overall accuracy


Threshold for "genuinely useful" in a real community tool:


Overall accuracy ≥ 0.78
Per-class F1 ≥ 0.70 for all three labels
analysis precision ≥ 0.72 (a tool that surfaces analysis posts should rarely surface hot takes mislabeled as analysis)


What I would not deploy: A model where any single label has F1 below 0.50, or where the fine-tuned model performs equal to or worse than the zero-shot baseline. That result would indicate either label inconsistency in annotation or a data distribution problem that should be fixed before deployment.

Why these numbers are specific enough to evaluate objectively: Each threshold is a number I can compute directly from classification_report output. There is no ambiguity about whether I hit them.


7. AI Tool Plan

Label stress-testing

Before annotating 200 examples, I will give Claude my label definitions and edge case description and ask it to generate 8–10 posts that sit at the boundary between analysis and hot_take specifically (the hardest pair). If Claude produces posts I cannot cleanly classify using my decision rules, that is a signal my definitions need tightening — I will revise before collecting data. I will document which generated posts were genuinely ambiguous and what rule I applied.

Annotation assistance

I will use Groq (llama-3.3-70b-versatile) to pre-label batches of ~30 unlabeled posts using the same system prompt I plan to use as the baseline classifier. This speeds up annotation but does not replace it — I will review and correct every pre-assigned label individually before including it in the dataset. I will add a pre_labeled column (boolean) to my CSV to track which examples were AI-pre-labeled vs. human-first-labeled, and I will disclose this in the AI usage section of my README.

Failure analysis

After running the fine-tuned model on the test set and collecting the wrong predictions, I will paste the full list of misclassified examples (text + true label + predicted label) into Claude and ask it to identify common themes: are the errors concentrated on a specific label pair? Are they systematically short posts? Posts with sarcasm? Posts where a stat appears? I will then verify any pattern Claude identifies by re-reading the examples myself before writing it up. I will note in my README which patterns were AI-surfaced and which I confirmed independently.


Annotation Log (to be updated during Milestone 3)

This section will be filled in during data collection. I will document at least 3 specific examples that were genuinely difficult to label, which labels they could belong to, and what I decided.

Here's the annotation log table ready to paste into your planning.md:

| # | Post (abbreviated) | Candidate labels | Decision | Reasoning |
|---|---|---|---|---|
| 1 | "The Spurs were something like 28 and 0 when they shot 40% from distance. When they're hitting their shots they're literally unbeatable." | analysis / hot_take | analysis | The stat is used to build a conditional argument about the Spurs' shooting dependency and team identity, not just to punch up an opinion. The logical chain (shooting % → team style → defensive prescription) earns the analysis label despite the hyperbolic conclusion. |
| 2 | "Top 10 player ever but has been out of the second round twice in his 12 year career. It's crazy how much leeway this guy gets." | analysis / hot_take | hot_take | Contains a verifiable stat (second round exits) but no logical chain — the stat is used rhetorically to support a dismissive claim rather than as part of a structured argument. One stat with no reasoning = hot_take per decision rule. |
| 3 | "Incredible comebacks by the Knicks all series. Youth and inexperience of Spurs showed time and time again." | reaction / hot_take | reaction | Post starts as an emotional response to the series result and adds a tacked-on opinion with no supporting evidence. Primary intent is expressing a feeling, not making a point. Reaction + unsupported opinion = reaction per decision rule. |
1 "The Spurs were       analysis / hot_take      analysis     Stat used to  build a conditional argument about team identity, not just to punch up an opinion
something like 28 
and 0 when they 
shot 40% from 
distance..."

  Stat used to build a conditional argument about team identity, not just to punch up an opinion2"Top 10 player ever but has been out of the second round twice in his 12 year career..."analysis / hot_takehot_takeOne stat with no logical chain — used rhetorically to support a dismissive claim3"Incredible comebacks by the Knicks all series. Youth and inexperience of Spurs showed time and time again."reaction / hot_takereactionReaction + tacked-on unsupported opinion = still reaction per decision rule
