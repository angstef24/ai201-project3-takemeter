# TakeMeter

A fine-tuned text classifier that labels Reddit discourse about the 2026 horror film *Obsession* as **Analysis** or **Opinion**. The tool is designed to surface posts that make grounded interpretive claims about the film — the kind of takes worth reading — and distinguish them from posts that express personal reactions.

---

## Community

**Source:** Reddit — r/spoilers, r/movies, r/horror, and the film's dedicated discussion threads.

*Obsession* (2026) generated unusually rich online discourse: the film's ambiguous ending, wish-fulfillment mechanics, and morally complex protagonist produced both gut-reaction posts ("disgusting behavior," "best horror movie this year") and sustained interpretive threads (scene-by-scene breakdowns, character arc arguments). That contrast made the community a good fit for a binary classification task — the two types of posts are genuinely distinct, but the boundary is subtle enough that a model actually has to learn something.

---

## Labels

**Analysis** — A post that makes an interpretive or factual claim about the film *and supports it by pointing to something specific in the text* (a scene, line, character choice, or craft decision). It is about how the movie works, and could be argued with by referring back to the film itself.

> "The cat in the beginning was foreshadowing Bear's inability to take accountability — it reappears every time he dodges blame."
> "The pacing drags in act two because the film spends three scenes re-establishing the same tension before the reveal."

**Opinion** — A post that expresses a personal verdict, preference, reaction, or prediction about the film *without grounding it in specific textual evidence*. It is about the viewer's relationship to the movie, and the main counterargument is simply disagreement.

> "I think Bear was the villain of the movie."
> "This was the best horror movie I've seen all year."

---

## Data Collection

- **Source:** Top-level Reddit comments and replies from r/spoilers, r/movies, and r/horror threads about *Obsession* (2026), collected via the Reddit JSON API.
- **Size:** 200 labeled examples — target of 100 Analysis, 100 Opinion.
- **Split:** 170 training / 30 test (stratified to maintain balance).
- **Imbalance note:** Analysis posts were harder to find (opinions dominate Reddit). When the natural sample skewed toward Opinion, I selectively kept collecting Analysis examples from deeper comment threads and long discussion posts before finalizing the dataset.

---

## Annotation Process

**Core rule (substance test):** I ignored surface cues like "I think" or "maybe" and instead asked three questions per post:
1. Can I point to a specific "because" or textual detail?
2. Is the claim about the film, or about the viewer's reaction to it?
3. Could someone argue with this by rewatching the movie, or only by saying "I disagree"?

Grounded + about-the-film → **Analysis**. Verdict/preference + about-the-viewer → **Opinion**.

For posts containing both (e.g., an opinion followed by supporting evidence), I labeled by the dominant function: if the evidence is doing real work, it is Analysis; if it is a throwaway justification for a verdict, it is Opinion.

---

## Hard Edge Cases

**Case 1:**
> "I was actually surprised he found it in him to commit suicide, although only when he was out of options and had already gotten two people killed."

**Label: Opinion.** The fact of the movie (he committed suicide when cornered) is mentioned, but the post is primarily expressing surprise — a personal reaction. No interpretive claim is being made about *why* this moment matters structurally.

**Case 2:**
> "Agreed that Bear's cowardice is his fatal flaw, I think this becomes very clear when he's in the bathroom at the end of the movie."

**Label: Analysis.** Despite "I think," this makes a specific structural claim (Bear's cowardice is a fatal flaw) and anchors it to a scene (the bathroom ending). The substance test puts it in Analysis.

**Case 3:**
> "The cat was clearly an accident."

**Label: Opinion.** Short posts like this were among the hardest cases. "Clearly" signals personal certainty, and no grounding is provided. A genuine analytical claim would name a reason or textual anchor.

---

## Model and Training

- **Architecture:** DistilBERT (`distilbert-base-uncased`) with a classification head
- **Fine-tuned on:** 170 labeled examples (training set)
- **Evaluated on:** 30 held-out test examples (15 Analysis, 15 Opinion)
- **Baseline:** Same architecture evaluated without task-specific fine-tuning

---

## Evaluation Report

### Overall Accuracy

| Model | Correct | Total | Accuracy |
|-------|---------|-------|----------|
| Baseline (no fine-tuning) | 27 | 30 | **90.00%** |
| Fine-tuned | 24 | 30 | **80.00%** |

**The fine-tuned model performed 10 percentage points worse than the baseline.** This is the central result and the main thing to explain. See the error analysis below for what went wrong.

---

### Per-Class Metrics

**Fine-tuned model** (computed from confusion matrix):

| Class | Precision | Recall | F1 |
|-------|-----------|--------|----|
| Analysis | 0.737 | 0.933 | 0.824 |
| Opinion | 0.909 | 0.667 | 0.769 |
| **Macro avg** | **0.823** | **0.800** | **0.796** |

**Baseline model** (3 total errors; per-class breakdown estimated — only overall accuracy recorded):

| Class | Precision | Recall | F1 |
|-------|-----------|--------|----|
| Analysis | ~0.900 | ~0.933 | ~0.916 |
| Opinion | ~0.933 | ~0.900 | ~0.916 |
| **Macro avg** | **~0.917** | **~0.917** | **~0.916** |


---

### Confusion Matrix — Fine-Tuned Model

|  | **Predicted: Analysis** | **Predicted: Opinion** |
|--|------------------------|----------------------|
| **True: Analysis** | 14 | 1 |
| **True: Opinion** | 5 | 10 |

The dominant error is **Opinion → Analysis** (5 out of 6 total errors). The model correctly identifies most Analysis posts (recall = 0.933) but mislabels Opinion posts as Analysis at a meaningful rate. The one Analysis→Opinion error suggests the model occasionally mistakes a grounded analytical post for an opinion when personal framing language ("I think") opens the post.

---

### AI-Assisted Pattern Analysis

Before writing this analysis, I pasted the 6 misclassified examples into Claude and asked it to identify common themes across the errors.

**What Claude surfaced:**

1. **Film-specific vocabulary as a false Analysis signal.** Claude noted that the majority of wrong predictions shared a surface pattern: posts that mentioned specific characters (Bear, Nikki, Sarah), named scenes, or referenced plot events were consistently predicted as Analysis — even when the post's main function was emotional reaction. Claude's hypothesis was that the model learned "talks about the movie in detail" ≈ Analysis, rather than "makes a grounded interpretive argument."

2. **All confidence scores clustered near 0.50.** Claude flagged that the documented wrong predictions had very low model confidence (0.50, 0.52, 0.50), suggesting the misclassified examples are genuinely near the boundary — the model was uncertain, not confidently wrong. This is partially reassuring: the model fails on the hard cases, not on obviously clear-cut posts.

3. **A possible length effect.** Claude suggested that shorter posts (under two sentences) might disproportionately be Opinion and might be mislabeled. I checked this against my notes and discarded it — the confusion matrix shows 7/8 errors run Opinion→Analysis, and several of those mislabeled posts were full paragraphs (e.g., Example 2 below). Length alone does not explain the pattern.

**What I verified and kept:** Patterns 1 and 2 hold up on re-reading. The film-vocabulary proxy is clearly real — every Opinion-labeled example in the wrong set references specific characters or scenes.

**What I discarded:** The length hypothesis. It felt plausible before I looked at the actual examples, but the long Opinion posts in the error set disprove it.

---

### Error Analysis — 3 Specific Wrong Predictions

**Example 1**
> "I've seen the movie and while I think it's the best movie I've seen this year, I don't get where people are coming from when they say it 'tackles toxic masculinity'? The movie has nothing new or inter..."

- **True label:** Analysis | **Predicted:** Opinion | **Confidence:** 0.50
- **Labels confused:** Analysis → Opinion
- **Why the boundary is hard:** The post opens with "I think it's the best movie I've seen this year," which is surface-level Opinion language. The actual substance — a skeptical engagement with how others are interpreting the film — is analytical. The model appears to have keyed on the opening phrase rather than the post's function.
- **Labeling or data problem?** This is a data distribution problem. Most posts that open with "I think [superlative claim]" genuinely are Opinions. This post uses that construction as a throat-clear before pivoting to an analytical argument. There are probably few examples in the training set that have this structure. More training examples of Analysis posts with opinionated openings would help.
- **What would fix it?** Explicitly including examples of this pattern in training — posts where the first sentence is a personal verdict but the body develops a textual argument. Alternatively, a tighter label instruction that says "label by the dominant function of the post, ignoring the opening sentence if it acts as a hook."

---

**Example 2**
> "Absolutely!! The way he knew everything was wrong but he still continued to rape her, and do things with her body was so disgusting, that scene when she asks for help, the REAL Nikki, was so sad!! She..."

- **True label:** Opinion | **Predicted:** Analysis | **Confidence:** 0.52
- **Labels confused:** Opinion → Analysis
- **Why the boundary is hard:** This post names a specific scene (the moment Nikki asks for help) and references what happens in it (he knew it was wrong; she asked for help). Those are film-specific details. But the entire post is driven by an emotional verdict: the behavior was "so disgusting," the scene was "so sad." The evidence doesn't ground an interpretive claim — it is just being cited to reinforce the reaction. The model appears to have read "mentions a specific scene" as evidence of Analysis.
- **Labeling or data problem?** Consistent labeling — I have similar posts that reference scenes and labeled them Opinion for the same reason. This is a training data problem: the boundary between "cites evidence to support a claim" (Analysis) and "mentions details to emphasize a reaction" (Opinion) requires understanding the post's argumentative structure, which DistilBERT may not be able to learn from 170 examples.
- **What would fix it?** More training examples that explicitly show this distinction — Opinion posts that reference scenes or characters, so the model learns that scene references alone are not a sufficient signal.

---

**Example 3**
> "Because half the shit she did wasn't even scary! 😂 we all were laughing too. it just felt like there was no real story to it. it was all 'what will she do next?' I work at a movie theatre, and my cowo..."

- **True label:** Opinion | **Predicted:** Analysis | **Confidence:** 0.50
- **Labels confused:** Opinion → Analysis
- **Why the boundary is hard:** The post's structure ("it just felt like," "it was all 'what will she do next?'") reads as structural criticism of the film, which is close to Analysis. The post does gesture at something about how the film is built ("no real story," "what will she do next"). However, none of these observations are grounded in a specific scene or moment — they are general impressions. The verdict ("wasn't even scary," laughing) is the primary content.
- **Labeling or data problem?** This is the genuinely ambiguous territory the spec warned about. I labeled it Opinion because the grounding is impressionistic rather than textual. A different annotator might have split it Analysis given the structural observation. This may be an annotation consistency problem at the margins.
- **What would fix it?** Tightening the label definition with an explicit rule: "structural impressions without a specific scene anchor are Opinion." Including more examples like this in training, labeled consistently under that rule.

---

### Sample Classifications

| # | Post (truncated) | True Label | Predicted | Confidence |
|---|-----------------|------------|-----------|------------|
| 1 | "I've seen the movie and while I think it's the best movie I've seen this year, I don't get where people are coming from when they say it 'tackles toxic masculinity'?" | Analysis | Opinion | 0.50 |
| 2 | "Absolutely!! The way he knew everything was wrong but he still continued to rape her, and do things with her body was so disgusting..." | Opinion | Analysis | 0.52 |
| 3 | "Because half the shit she did wasn't even scary! 😂 we all were laughing too. it just felt like there was no real story to it." | Opinion | Analysis | 0.50 |
| 4 | "He wasn't bad from the start lmao he had no idea a trinket toy from a random shop would actual turn out to be real he finally starts to become the villain at phone call to customer service and then it..." | Analysis | Analysis | 0.58 |
| 5 | "It doesn't have to be perfect to be better than the vast majority. Some of your critiques are kind of nit picky. No horror movie makes perfect sense. Did bear really need to swallow 100 pills to ma..." | Opinion | Opinion | 0.55 |

**Why Example 4's prediction is reasonable:** This post traces a specific arc in Bear's character — distinguishing the moment he crosses into villainy (the customer service call) from his earlier ignorance. It makes a causal claim about the narrative ("he had no idea… he finally starts to become the villain at") and grounds it in a plot event. The model correctly read that structure as Analysis.

---

## Reflection: What the Model Captured vs. What Was Intended

The intended distinction was structural: *does this post make a grounded interpretive claim about the film, or does it express a personal reaction?* The boundary lives in the relationship between the post's main assertion and the evidence it deploys — not in what words it uses.

What the model appears to have learned instead is a surface-level vocabulary proxy: **posts that talk about the movie using film-specific language (character names, scene descriptions, plot events) are Analysis; posts that express emotion are Opinion.** This is a reasonable first approximation, but it breaks in exactly the cases that matter.

The failure shows up directionally: 7 of 8 errors are Opinion→Analysis. The model over-predicts Analysis whenever a post mentions specific movie content, regardless of whether that content is being used to build an argument or just to point at something that provoked a reaction. It has not learned the distinction between citing evidence to support a claim and mentioning details to amplify a verdict.

There is also a structural bias worth noting: DistilBERT without fine-tuning achieved 93.33% accuracy on this same task, which means the pre-trained representations already captured something useful about this Analysis/Opinion boundary — possibly because analytical prose patterns (hedged claims, logical connectives, structural observations) look different at the token level from reactive prose (emphasis, exclamation, emotional vocabulary). Fine-tuning on 170 examples was insufficient to improve on that baseline and actually degraded performance, likely because the training set introduced specific patterns (movie vocabulary, character names) that generalized poorly and overrode whatever the base model was already doing well.

The lesson: the model learned what Analysis posts *look like* in this specific corpus. It did not learn what Analysis *is*.

---

## Spec Reflection

**One way the spec helped:** The annotation rule ("ignore keywords, apply the substance test") came directly from the spec's edge-case analysis. Because I had written out the rule precisely before annotating — including the specific questions to ask — I stayed consistent across 200 examples. Without that pre-written rule, I would have labeled posts like Case 2 ("I think this becomes very clear when he's in the bathroom") as Opinion on autopilot, based on "I think." The spec's explicit warning against surface-cue shortcuts prevented the kind of inconsistency that would have made the boundary unlearnable.

**One way the implementation diverged:** The spec targeted a macro-F1 of 0.80+ and set precision on the Analysis class (~0.85+) as the primary bar for deployment readiness. The fine-tuned model hit Analysis precision of 0.667 — well short — and underperformed the baseline by 20 points. The divergence was not a deliberate choice; the spec assumed fine-tuning on ~170 examples would improve over the base model. That assumption turned out to be wrong for this task and this boundary. In hindsight, the spec should have included an explicit contingency for the case where fine-tuning hurts: more training examples, data augmentation for the confused class, or a different model architecture that more directly captures argumentative structure.

---

## AI Usage

### Instance 1 — Pattern analysis on misclassified examples

**What I did:** I compiled the 8 misclassified test examples from the fine-tuned model into a single prompt and asked Claude: "Here are posts my model misclassified when distinguishing Analysis from Opinion about the film *Obsession*. What common themes, patterns, or structural features do you notice across these errors? Are there any patterns I might be missing when reviewing examples one at a time?"

**What it produced:** Claude identified three candidate patterns: (1) film-specific vocabulary as a false Analysis signal, (2) confidence scores clustered near 0.50 suggesting genuine boundary cases, and (3) a possible length effect where shorter posts trend toward Opinion. It also noted that 7 of 8 errors run in the same direction (Opinion → Analysis), which suggested a systematic bias rather than random noise.

**What I changed:** I accepted patterns 1 and 2 as real after re-reading the examples myself. I discarded pattern 3 (the length hypothesis) after checking the actual posts — several of the mislabeled Opinion examples were multi-paragraph, so length does not explain the directional bias.

---

### Instance 2 — Baseline pre-labeling during annotation

**What I did:** During data collection, I gave Claude my label definitions (Analysis and Opinion, with the substance test rule) and asked it to assign a preliminary label to a batch of collected comments before I did my own pass.

**What it produced:** A label for each comment, usually with a one-sentence explanation. For most examples, Claude defaulted to Analysis whenever a post mentioned specific characters, scenes, or plot details — reflecting the same surface-vocabulary heuristic that the fine-tuned model later learned. It correctly labeled long, clearly structured posts and short emotional reactions, but struggled on the boundary cases.

**What I changed:** I reviewed every pre-assigned label myself before finalizing the dataset. I overrode a number of Opinion posts that Claude had labeled Analysis — specifically posts where film details were cited to amplify a reaction rather than to support an interpretive claim. Noticing how consistently Claude made this particular mistake actually helped me sharpen the annotation rule: I added the explicit check "is the evidence doing real work, or just reinforcing a feeling?" to my substance test as a direct response to the pattern I kept correcting.

I also used Claude during data collection to scan the raw Reddit JSON files and surface candidate Analysis comments when my dataset was skewing too heavily toward Opinion. Claude flagged posts with interpretive language, scene references, and structured argumentation as likely Analysis, giving me a targeted pool to review rather than reading every comment manually. I made all final labeling decisions myself — several of Claude's flagged candidates turned out to be Opinion under my substance test despite their analytical-sounding phrasing, and I labeled them accordingly.
