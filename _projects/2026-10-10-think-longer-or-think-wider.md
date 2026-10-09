---
layout: project
title: "Think Longer or Think Wider? Spending 8,000 Tokens in a Small Language Model"
description: Give a small reasoning model 8,000 tokens per math problem. Should it think once, for a long time, or think sixteen times, briefly, and vote? We bet on the vote, ran 13,760 reasoning traces on free Kaggle GPUs with Qwen3-1.7B and 4B, and lost the bet.
date: 2026-10-10
image: "/images/projects/think_longer_wider_thumbnail.jpg"
tags: [python, NLP, LLM, reasoning, test-time-compute, qwen3, vllm, kaggle, evaluation]
github: https://github.com/bhuwanadhikari/sequence-vs-parallel-cot-in-slm
favorite: true
published: true
---

# The Story of Think Longer or Think Wider

> <span style="font-weight: normal; font-size: 18px"> A small model on a phone gets a fixed amount of thinking. We were sure it should spend it on many quick guesses and a vote. The data said otherwise. </span>

---

<img src="/images/projects/think_longer_wider_poster.webp" alt="Research poster titled Think Longer or Think Wider? Allocating a Fixed Test-Time Compute Budget in Small Language Models" width="100%" style="border-radius: 8px;">

<center>
<i>The poster we presented at Universität Trier (MSc NLP), with Vishesh Chandrakar.</i>
</center>

---

## A Budget, Not a Model

Small language models are getting good enough to live on phones and laptops. A 1.7-billion-parameter model fits in a few gigabytes, runs without a server, and never sends your data anywhere. That's the dream of private, on-device AI.

The newest of these models can also *think*. Before answering, they write out a long chain of reasoning, like a student's scratch paper, and that reasoning makes them noticeably more accurate on math and logic.

But thinking costs tokens, and on a phone, tokens are time and battery. So in practice you don't ask *"how smart is this model?"* You ask *"I can afford about 8,000 tokens for this question. What's the best way to spend them?"*

There are two obvious answers:

- **Think longer (depth):** one long chain of thought, up to 8,000 tokens, then answer.
- **Think wider (width):** several short, independent attempts that split the same 8,000 tokens, then take a majority vote on the answers.

```
                1 × 8K    ████████████████████████████████  → answer
                2 × 4K    ███████████████ ███████████████   → vote
  Question  →   4 × 2K    ███████ ███████ ███████ ███████   → vote
                8 × 1K    ███ ███ ███ ███ ███ ███ ███ ███   → vote
              16 × 500    █ █ █ █ █ █ █ █ █ █ █ █ █ █ █ █   → vote
```

Same total length on every row, same cost. Which row gets the most problems right?

---

## Our Bet

My project partner Vishesh Chandrakar and I made a bet before running anything: **width wins**.

We had good reasons. A long chain of thought that takes a wrong turn early tends to keep building on that mistake; the model rarely walks back far enough to recover. Several short, independent attempts don't share mistakes. If most of them land on the same answer, that answer is probably right. Voting (often called *self-consistency*) has a strong track record, and it has been reported to help small models in particular.

Most earlier work on this trade-off used large models or models that don't think. We wanted the on-device size class, so we wrote down two research questions:

- **RQ1:** At the same token budget, does depth (one long chain) or width (many short chains + a vote) solve more problems?
- **RQ2:** Does the answer change with problem difficulty, model size, and thinking mode?

And two hypotheses: **H1**, several short attempts with a vote beat one long chain; **H2**, on hard problems with thinking on, the long chain does better, because hard problems need more tokens and short attempts run out.

---

## The Setup

- **Models:** Qwen3-1.7B and Qwen3-4B. Same family, same tokenizer, same training recipe, and both can switch thinking **on** or **off**. That let us vary model size and thinking mode without changing anything else.
- **Problems:** 215 problems from MATH-500, 43 from each difficulty level (1 easiest, 5 hardest), the same subset for every run.
- **Budget splits:** `1×8000`, `2×4000`, `4×2000`, `8×1000`, `16×500` tokens.
- **Scoring:** majority-vote accuracy (`maj@N`), with answers compared by mathematical equivalence, not by string.

On paper that's 2 models × 2 modes × 5 splits. Making it fair, and making it fit on free GPUs, took a few more ideas.

---

## Running Out Is Not the Same as Being Wrong

The first problem showed up immediately. A thinking model given 500 tokens almost never finishes. It's still in the middle of its scratch work when the cap hits, so it never writes `\boxed{...}`, and we would score it as wrong.

That would make the experiment unfair. We would be measuring *"can it finish in 500 tokens?"*, not *"does its reasoning lead to the right answer?"*

So we borrowed a trick called **budget forcing**. When a chain hits its cap, we close the thinking block ourselves and start the answer for it:

```python
def force_suffix(prefix_ids):
    need_close = THINK and (THINK_END_ID not in prefix_ids)
    return ("\n</think>\n\n" if need_close else "\n\n") + "The final answer is \\boxed{"
```

Then we let the model write 24 more tokens, greedily. Whatever it puts in the box is its answer. Every split now gets scored on what it figured out, not on whether it had time to say it.

---

## One Generation, Five Budgets

The second problem was compute. Each problem needs 16 samples (enough to build every vote size up to 16), at 5 budgets, for 2 models and 2 modes, across 215 problems. Generated separately, that would have taken far more GPU time than we had on Kaggle's free T4s.

Then we noticed something: **a sample capped at 500 tokens is exactly the first 500 tokens of the same sample capped at 8,000.** Sampling goes left to right; the cap only decides where it stops.

So each problem is generated **once**, 16 samples at 8,000 tokens, and every shorter budget is cut from it:

```python
for L in LENGTHS:                      # 8000, 4000, 2000, 1000, 500
    natural = len(full_ids) <= L and o.finish_reason == "stop"
    if natural:                        # finished within L on its own: keep its answer
        rec["by_L"][str(L)] = {"tokens": len(full_ids), "truncated": False, ...}
    else:                              # cut at L, then budget-force an answer
        pre = full_ids[:L]
        force_reqs.append((j, L, pid + pre + tok.encode(force_suffix(pre), add_special_tokens=False)))
```

It's statistically exact, not an approximation: one generation run instead of five, and every budget is cut from the very same samples, so the comparison between budgets is paired. vLLM's prefix caching kept the forcing pass cheap, since all the cut-off versions share the same beginning.

Free GPUs have one more catch: sessions die. The notebook commits and pushes its results to GitHub after **every finished problem**, and a new session clones the repo and continues from the next unfinished one. Twenty Kaggle sessions later (2 models × 2 modes × 5 levels), we had **13,760 reasoning traces**.

---

## Counting Votes With Math

Voting sounds simple until two samples answer `1/2` and `0.5`. Or `\frac{2}{3}` and `\dfrac{2}{3}`. As strings they're different answers that split the vote; as math they're the same answer.

So before voting, every problem's answers are grouped into equivalence classes using `math_verify`, and votes are counted per class. Instead of picking random subsets of samples, we computed `maj@N` exactly over all C(16, N) subsets, and confidence intervals come from a 2,000× problem-level bootstrap.

Then we looked at the results.

---

## We Lost the Bet

<img src="/images/projects/think_longer_wider_all_levels.webp" alt="Majority-vote accuracy as the 8K-token budget is split from 1 by 8K to 16 by 500, for Qwen3-1.7B and 4B with thinking on and off. Thinking-on lines peak at 1 by 8K; thinking-off lines peak at 4 by 2K." width="70%" style="display: block; margin: 0 auto;">

<center>
<i>Accuracy as 8K tokens are split from depth (left) to width (right). Solid: thinking on. Dashed: thinking off. Circles mark the best split.</i>
</center>

With thinking on, the line just goes down. Every time we split the budget, accuracy drops.

| Split | 1.7B · think | 1.7B · no-think | 4B · think | 4B · no-think |
|---|---|---|---|---|
| **1 × 8000** | **90.3** | 77.5 | **94.7** | 87.5 |
| 2 × 4000 | 85.6 | 77.6 | 90.4 | 87.4 |
| 4 × 2000 | 80.1 | **79.9** | 84.5 | **88.7** |
| 8 × 1000 | 72.5 | 77.8 | 72.3 | 85.9 |
| 16 × 500 | 51.2 | 64.0 | 53.3 | 70.0 |
| *One chain − best vote* | *+4.7\** | *−2.4\** | *+4.3\** | *−1.3* |

<center>
<i>% of problems solved. * = statistically significant difference between the single chain and the best voting split.</i>
</center>

**With thinking on, one long chain wins.** It solves 90.3% (1.7B) and 94.7% (4B), beating the *best* voting split by about 4.5 points, and beating all 10 same-budget comparisons we tested. H1 was rejected.

**With thinking off, a moderate vote wins.** `4 × 2K` is best for both models: +2.4 points for 1.7B (significant) and +1.3 for 4B (not significant). So our intuition wasn't wrong; it was pointed at the wrong mode.

**Splitting too thin always hurts.** At `16 × 500`, accuracy collapses to 51–53% with thinking on, and 64–70% without.

---

## What 500 Tokens Looks Like

A single problem shows why. This one is level 4:

> Let f(n) = ⌊n⌋ if n ≥ 4, and ⌈n⌉ if n < 4. Find f(π/3) + f(√45) + f(8^(2/3)).

The answer is 2 + 6 + 4 = **12**. With 8,000 tokens, Qwen3-1.7B gets it right in **all 16 samples**, and it doesn't even need the room: a typical trace finishes in about 2,000 tokens, working out each term, then going back to double-check.

Cut the same 16 traces at 500 tokens and force an answer, and this is what lands in the boxes:

```
2 + 7 + 8      2               2 + 5 + 6 = 13    2 + 2 + 4 = 8
2              2 + 4 + 8       2 + 6 + 6         2 + 5 + 8 = 15
6              2 + 4 + 8 = 14  6                 2
2 + 2 + 6 = 10 6               2 + 2 + 6 = 10    2 + 7 + 8
```

Zero of 16 are correct. The models were mid-thought: most got the first term right and then guessed the rest. Sixteen half-finished thoughts don't vote their way to a right answer; they just disagree in sixteen ways.

The pattern holds across the whole dataset:

<img src="/images/projects/think_longer_wider_outcomes.webp" alt="Stacked bars showing, for each token budget, the share of answers that are right, wrong but finished, or wrong because they ran out of tokens, for thinking on and off, for Qwen3-1.7B and 4B" width="100%">

<center>
<i>How each answer ends. Hatched = wrong because it ran out of tokens. With thinking on, nearly all short-budget errors are truncation, not bad reasoning.</i>
</center>

With thinking on, the blue hatched part grows as the budget shrinks; at 500 tokens about half of all answers are wrong *because they ran out of tokens*. The light blue "finished but wrong" sliver stays tiny. When a thinking model gets to finish, it's almost always right.

With thinking off, the story flips. Answers are short (around 760 tokens), so they mostly finish even at 2K, and the errors are the gray "finished but wrong" kind. That's exactly what voting is good at fixing, which is why `4 × 2K` wins there.

**Thinking mode decides the mechanism.** With thinking on, short splits fail by running out of time. With thinking off, they fail by being wrong, and votes can repair that.

---

## Harder Problems, Bigger Cost

RQ2 asked whether the answer changes with difficulty, size, and mode. Splitting the results by difficulty makes the difference obvious:

<img src="/images/projects/think_longer_wider_by_difficulty.webp" alt="Six panels of majority-vote accuracy by difficulty. With thinking on, the hard panel falls from about 83 to 91 percent at 1 by 8K to about 25 percent at 16 by 500. With thinking off, the curves are flatter and peak at 4 by 2K or 8 by 1K." width="100%">

<center>
<i>Left three: thinking on. Right three: thinking off. Easy (L1–2), medium (L3), hard (L4–5).</i>
</center>

On easy problems with thinking on, splitting barely matters until `16 × 500`; easy problems don't need many tokens. On **hard** problems, it's a cliff: from 83% (1.7B) and 91% (4B) with one chain, down to about **25%** at `16 × 500`. Hard problems need long reasoning, and short attempts simply run out before getting there. H2 was supported.

Model size mattered less than we expected. 4B is more accurate almost everywhere, but it doesn't move the best split: both models prefer one chain with thinking on and `4 × 2K` with thinking off. The clearest size difference: with thinking off, voting helps the 1.7B model significantly and the 4B model not significantly.

---

## Two Things That Surprised Us

**The long chain didn't even spend its budget.** A thinking `1 × 8K` chain used on average just **3,949 tokens** (1.7B) and **3,830** (4B), less than half the cap. The splits, on the other hand, use nearly every token they're given. So one long chain wins while *spending less*. The budget is a cap, not a cost, and that's why every result in the repository is plotted against measured tokens, not the cap.

**The right answer is often there, and the vote misses it.** At `16 × 500` with 1.7B and thinking on, the correct answer appears in at least one of the 16 samples for 65.1% of problems, but the majority vote picks it for only 51.2%. Width creates a selection problem: you need a way to recognize the right answer in the pile, and a plain vote isn't good enough when most of the pile is half-finished.

---

## What We Couldn't Settle

- **215 problems from one dataset.** The confidence intervals are wide, and math may not represent other kinds of reasoning.
- **Short budgets are cut from long ones.** That's what made the experiment affordable, but a model that *knows* it only has 500 tokens might reason differently from one that's simply cut off. Budget-aware generation is the obvious next experiment.
- **One result we can't explain:** with thinking on, the 4B model drops to the 1.7B model's level at `8 × 1K` (72.3% vs 72.5%), even though it's ahead at every other split.
- **Next:** more budgets, more datasets, more model families, and better selection than a plain vote.

---

## Back to the Phone

We started with a small model on a phone and 8,000 tokens to spend. Here's the answer we'd give now:

- **If the model can think, let it think once, for as long as it needs.** Don't chop its reasoning into pieces; it will mostly just run out of room.
- **If thinking is off, take a few medium-length answers and vote.** `4 × 2K` was the sweet spot for both models.
- **Never split too thin.** At 500 tokens per attempt, you're voting on unfinished sentences.

We lost our bet, but the result is more useful than the one we expected. On-device, the cheapest strategy turned out to also be the best one: give the small model one quiet, uninterrupted chance to think.

---

*Code, data, all 13,760 traces, and every table and figure: [sequence-vs-parallel-cot-in-slm](https://github.com/bhuwanadhikari/sequence-vs-parallel-cot-in-slm) · Notebook: [depth_vs_width_kaggle.ipynb](https://github.com/bhuwanadhikari/sequence-vs-parallel-cot-in-slm/blob/main/depth_vs_width_kaggle.ipynb) · Full findings: [FINDINGS.md](https://github.com/bhuwanadhikari/sequence-vs-parallel-cot-in-slm/blob/main/analysis/FINDINGS.md)*
