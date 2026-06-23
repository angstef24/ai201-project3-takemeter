# TakeMeter — Planning

## Community
What community did you choose and why? Why is this community a good fit for a classification task — what makes the discourse varied enough to be interesting?
- The community will be online discourse on the new movie 'Obsession'. The recent horror movie found wild success at the box office and had many people chatting about it.

## Labels
What are your 2–4 labels? Define each in a complete sentence. Include 2 example posts per label.
- Analysis — A post that makes an interpretive or factual claim about the film *and supports it by pointing to something specific in the text* (a scene, line, character choice, or craft decision). It's about how the movie works, and could be argued with by referring back to the movie itself.
    - The cat in the beginning was foreshadowing Bear's inability to take accountability — it reappears every time he dodges blame.
    - The pacing drags in act two because the film spends three scenes re-establishing the same tension before the reveal.
- Opinion — A post that expresses a personal verdict, preference, reaction, or prediction about the film *without grounding it in specific textual evidence*. It's about the viewer's relationship to the movie, and the main counterargument is simply disagreement.
    - I think Bear was the villain of the movie.
    - This was the best horror movie I've seen all year.

## Hard edge cases
What type of post will be genuinely ambiguous between two labels? How will you handle it when you encounter it during annotation?
- The genuinely ambiguous posts are ones where the *wording* suggests one label but the *substance* is the other. Surface cues like "I think" or "maybe" are unreliable on their own and misfire both ways:
    - "I think the cat foreshadowed his arc *because* it reappears every time he lies." → has "I think," but it's Analysis (it names a specific detail and proposes a mechanism).
    - "Bear was the worst character ever written." → no "I think," but it's pure Opinion (a bare verdict with no grounding).
- My annotation rule: ignore keywords and apply the substance test instead — (1) Can I point to the "because" / specific textual evidence? (2) Is the claim about the film or about the viewer's reaction? (3) Could someone argue with it by rewatching the movie, or only by saying "I disagree"? Grounded + about-the-film → Analysis; verdict/preference + about-the-viewer → Opinion.
- For posts that contain both (e.g. an opinion followed by supporting evidence), I'll label by the dominant function of the post — if the evidence is doing real work, it's Analysis; if it's a throwaway justification for a verdict, it's Opinion.

## Data collection plan
Where will you collect examples? How many per label? What will you do if a label is underrepresented after 200 examples?
- Source: Reddit threads about Obsession (r/movies, r/horror, and the film's own discussion threads), pulling top-level comments and replies.
- Target: 200 examples, an even 100 Analysis / 100 Opinion split, held out into train/test.
- If a label is underrepresented after 200 (Analysis is the likely shortfall, since opinions dominate Reddit): I'll keep collecting beyond 200 and sample selectively for the rare class rather than down-sampling the common one and losing data. I'll widen sources if needed (Letterboxd reviews, YouTube comments, other subreddits) to find more grounded Analysis posts. As a last resort I'll note the residual imbalance and rely on macro-F1 + per-class metrics so the minority class isn't hidden.

## Evaluation metrics
Which metrics will you use to evaluate your model and why are those the right ones for this specific task? (Accuracy alone is not enough — explain what else you need and why.)
- I'll report every metric for both models (the base model and the fine-tuned model) on the same held-out test set, so the central question — did fine-tuning actually improve discourse classification? — can be answered directly rather than assumed.
- Overall accuracy (both models). Reported, but not trusted on its own: although my training set is balanced 100/100, real Reddit discourse skews heavily toward Opinion, so a model that mostly guesses "Opinion" can post a high accuracy while being useless. Accuracy is the headline number the rubric requires, not the number I judge success by.
- Per-class precision, recall, and F1 — especially for the Analysis class. Analysis is the rarer, higher-value label (the whole point of the tool is surfacing genuine insight), so I care most about: of posts flagged "Analysis," how many really are (precision), and how many true analysis posts I miss (recall). F1 balances the two.
- Macro-F1 as the metric I actually optimize for, since it weights both classes equally and won't let the majority class hide weak minority-class performance.
- A confusion matrix for each model, to see which direction the errors run — mislabeling hype as Analysis is a worse failure for this tool than the reverse, and a confusion matrix makes that asymmetry visible.
- Qualitative error analysis: at least 3 specific misclassified posts per model, with my read on why the model failed (e.g. did it latch onto surface cues like "I think" instead of grounded reasoning — the exact failure mode my edge-case rule warns about?).
- A learned-vs-intended reflection: comparing what the model appears to key on against the substance test I defined for the labels, to surface the gap between what I intended ("is the claim grounded?") and what it actually learned (possibly just "does this look analytical?").

## Definition of success
What performance would make this classifier genuinely useful? What would you accept as "good enough" for deployment in a real community tool?
- It has to beat the right baselines, not 0%. Two baselines matter: (1) majority-class guessing, and (2) the simple "I think / maybe" keyword rule from my edge-case plan. If the fine-tuned model can't clearly beat both — and clearly beat the base model — then fine-tuning added nothing and the cheap heuristic is good enough. That comparison is the real bar.
- Judge against the human ceiling, not 100%. Because Analysis vs. Opinion is subjective, I'd sanity-check inter-annotator agreement on a sample (a second labeling pass). The model can't realistically beat the rate at which two humans even agree, so "good" means approaching that ceiling, not perfection.
- Concrete target: macro-F1 around 0.80+, with Analysis-class precision ~0.85+ even at some cost to recall. For a "highlight the good takes" tool, missing a few genuine analyses is tolerable, but flagging hype as insight erodes user trust fast — so I'd tune toward precision on the Analysis class.
- Good enough for deployment: the fine-tuned model meaningfully outperforms the base model and both baselines, hits the precision target on Analysis, and its error analysis shows failures that are defensible (genuinely ambiguous posts) rather than embarrassing (keyword-fooled). At that point it's useful as a surfacing aid — not an autonomous judge, but a filter a human can trust to float the better takes upward.

## AI usage
Disclose any use of AI tools in data collection, labeling, or analysis. (Required if you use LLM pre-labeling — every pre-assigned label must be reviewed and corrected by hand.)
- Tools used: Claude
- Where used (collection / pre-labeling / analysis / writing): collection (scanning Reddit JSON for candidate Analysis posts), pre-labeling (preliminary labels on a batch of collected comments before my own pass), and analysis (pattern analysis on the 8 misclassified test examples after evaluation).
- If used for pre-labeling: Claude; prompt included my full label definitions (Analysis, Opinion, and the substance test rule); Claude assigned a label for a batch before I reviewed; I reviewed and corrected every pre-assigned label myself before finalizing the dataset.
- What I changed: I overrode a number of Opinion posts that Claude had labeled Analysis — the consistent pattern in Claude's mistakes was defaulting to Analysis whenever a post mentioned specific characters, scenes, or plot details, regardless of whether that content was doing argumentative work.

## Hard Edge Case
Case 1:
I was actually surprised he found it in him to commit suicide, although only when he was out of options and had already gotten two people killed.
What I labeled: Opinion
Thought process: Ultimately, I bleieved it was an opinion that the fact of the movie was surprising.

Case 2:
Agreed that Bear’s cowardice is his fatal flaw, I think this becomes very clear when he’s in the bathroom at the end of the movie.
What I labeled: Analysis
Thought process: Even though the phrase "I think" was used, ultimately it reads more as an analysis of events in the movie.

Case 3:
The cat was clearly an accident.
What I labeled: Opinion
Thought process: The shorter comments ended up being among the most difficult since there isn't much context to work with. However, the word "clearly" conveys a sense of opnion to me since no movie analysis should really be the absolute truth unless coming from the director.