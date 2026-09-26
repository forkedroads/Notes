# AI Primer Notes: Reinforcement Learning Approaches

*Notes based on TypeSafe AI's "AI primer" — comparing RLHF, RLVR, and RLCD*

---

## 1. Reinforcement Learning — The Basics

**Q: Give me a very high level description of reinforcement learning.**

RL is a way of training a system through trial and feedback, rather than telling it the "right answer" directly.

**The basic loop:**
1. An **agent** takes an action (a model generates a response / makes a decision)
2. It receives a **reward or penalty** based on how good that action was
3. It **adjusts its behavior** to get more reward next time
4. Repeat, many times

> Analogy: training a dog with treats — you don't explain *why* sitting is good, you just reward it, and the behavior gets reinforced over time.

**Why it matters for AI models:**
Pretrained language models know a lot about language, but not what "good" output looks like for a given purpose. RL is applied *after* pretraining to steer the model — by rewarding good outputs and penalizing bad ones.

**Key insight:** The method (RL) stays the same across all three approaches below. What changes is **what counts as "good"** — i.e., the reward signal. That single design choice determines whether you get a chatbot, a reasoning model, or a decision model.

---

## 2. The Policy — Core Notation

The model itself is called the **policy**, written as:

    π_θ(y | x)

- `x` = the prompt/input
- `y` = a possible response
- `θ` (theta) = the model's parameters (weights)
- `π_θ(y|x)` = probability the model assigns to response y given prompt x

Training = adjusting θ so the policy assigns higher probability to "better" responses.

**Q: How are the parameters (θ) actually adjusted in RL?**

- Standard gradient-based optimization, like any neural net training — except the "loss" comes from *reward* instead of a labeled target.
- Problem: you can't directly differentiate through the sampling step (the model *samples* a response — that's random and non-differentiable).
- **Policy gradient trick**: increase the probability of actions that led to high reward; decrease probability of actions that led to low reward.
- In practice this is unstable, so algorithms like **PPO (Proximal Policy Optimization)** add stabilizers:
  - **Advantage**: was this response better/worse than *expected* for this prompt (not just raw reward)?
  - **Clipping**: limit how much the policy can change in a single update step.

**High-level loop:**
1. Sample responses from current policy
2. Score them with reward signal
3. Estimate advantage (better/worse than expected)
4. Nudge θ toward good responses, away from bad ones — in small, controlled steps
5. Repeat

---

## 3. RLHF — Reinforcement Learning from Human Feedback

**Pipeline (3 stages):**

1. **Supervised fine-tuning (SFT)** — fine-tune pretrained model on human-written examples → gives initial policy `π_SFT`

2. **Train a reward model** — humans compare pairs of responses (y1, y2); train `r_φ(x,y)` to predict preference, e.g. via Bradley-Terry model:

       P(y1 ≻ y2 | x) = σ( r_φ(x, y1) − r_φ(x, y2) )

   (σ = sigmoid function)

3. **RL optimization** — maximize expected reward with a KL penalty to avoid drifting too far from π_SFT:

       max_θ  E[ r_φ(x, y) − β · KL( π_θ(·|x) ‖ π_SFT(·|x) ) ]

   Typically optimized with PPO.

**Result:** Chatbots (ChatGPT, InstructGPT). Co-invented by Diogo Almeida, TypeSafe's cofounder.

**Problem — Mode Collapse & Mode Dropping** *(see section 6 below for full explanation)*

RLHF can reward sycophancy and confident-sounding hallucinations, because it optimizes for **what humans prefer**, not for correctness or honesty.

---

## 4. RLVR — Reinforcement Learning with Verifiable Rewards

**Key difference from RLHF:** reward comes from **automatic verification**, not human preference.

    r(x, y) = 1   if y is verified correct
    r(x, y) = 0   otherwise

- No reward model needed — a program/checker verifies correctness directly (code compiles + passes tests, math answer matches, proof checker accepts).
- Same optimization shape as RLHF (policy gradient / PPO), but reward is objective, not learned from preferences.
- Rewards long chain-of-thought reasoning, since more "thinking" tends to increase odds of a verified-correct answer.

**Result:** Strong reasoning models (math, code) — but slower and more expensive.

**Weakness:** Only works where correctness is mechanically checkable. Says nothing about *how confident the model should be* — a model can be right by luck while claiming near-certainty, or right while hedging.

---

## 5. RLCD — Reinforcement Learning for Calibrated Decisions

**Q: What does "calibrated decisions" actually mean? Can you make it simpler?**

**Analogy:** Two students take a quiz and state confidence (0–100%) with each answer.
- **Student A** says "95% sure" on everything, even guesses. Checked afterward: only right 60% of the time when saying 95%. → **Not calibrated.**
- **Student B** says "95% sure" only when they really know it, "55% sure" when guessing. Checked afterward: right 95% of the time at the 95% claims. → **Calibrated.**

Both students might get the *same number of questions right overall* — calibration isn't about accuracy, it's about whether **the confidence number itself is honest**.

**Formal definition:**

    P(correct | model states confidence = p) ≈ p

Across many predictions: outcomes assigned 0.8 should occur ~80% of the time. This is a property of *groups* of predictions, not a guarantee about any single answer.


    r = −(p_a − c)²

Where:
- `p_a` = the probability the model stated for its chosen answer
- `c` = 1 if correct, 0 if wrong

| Model said... | Correct? | p_a | c | Reward |
|---|---|---|---|---|
| 90% confident | Right | 0.9 | 1 | **−0.01** (best — confidently right) |
| 90% confident | Wrong | 0.9 | 0 | **−0.81** (worst — confidently wrong) |
| 30% confident | Right | 0.3 | 1 | **−0.49** |
| 50% confident | Wrong | 0.5 | 0 | **−0.25** |

This matches intuition: **confidently right = best outcome, confidently wrong = worst.**

**Why honesty wins long-run:** Because the penalty is squared, overconfidence gets punished hard whenever it's wrong. The only strategy that maximizes *average* reward across many predictions is to state confidence that matches your true hit rate — not to always claim high confidence. (This is called a *proper scoring rule* — the property that the best strategy is to report your true belief.)

**How this differs from RLHF/RLVR reward targets:**

| | Reward signal | Optimizes for |
|---|---|---|
| RLHF | Learned preference model | What humans like |
| RLVR | Automatic verifier | Whether the final answer is correct |
| RLCD | Calibration-sensitive reward (proper scoring rule) | Whether stated confidence matches actual correctness |

**Result:** TypeSafe's decision models — return typed decisions + calibrated probabilities instead of generated text. Designed for software to act on directly (thresholds, escalation, routing) rather than for a human to read and judge.

---

## 6. Mode Collapse vs. Mode Dropping

**Q: The page mentions mode collapse and mode dropping — what are these?**

**"Mode"** = a distinct, common valid pattern among possible outputs. E.g., for "write a birthday message," valid modes include: short & sweet, funny, poetic, formal, heartfelt. A healthy model should be able to produce any of these.

**Mode collapse (origin: GANs)**
- GANs pit a *generator* against a *discriminator*.
- Mode collapse = generator finds *one* output that reliably fools the discriminator and just repeats it, ignoring the true diversity of valid outputs.
- Analogy: a student discovers one vague, safe essay style always gets an A, and reuses it for every prompt regardless of what's actually being asked.

**Mode dropping (RLHF's milder version)**
- Instead of collapsing to one output, the model **narrows its distribution** — it still has variety, but systematically **loses probability on some valid modes** in favor of ones raters preferred.
- Per the page: "the model learns to favor a particular style, such as instruction following, while reducing the probability of other possible outputs."
- Example: if raters consistently prefer long, polite, hedge-y answers, RLHF training suppresses terser or blunter — but equally valid — response styles.

**Why this matters for the argument:**
Mode dropping means the model's output distribution reflects **what looked good to raters**, not the **true likelihood of correctness**. This directly undermines calibration: if probability mass has been reshaped by preference rather than by honesty, the model's stated probabilities can't be trusted as calibrated — even if the model still "sounds smart." This is the technical bridge the page uses to justify why RLCD (a separate training objective centered on calibration) is needed.

---

## Summary Table

| Method | Reward Signal | Optimizes For | Produces | Key Weakness |
|---|---|---|---|---|
| **RLHF** | Learned reward model from human preference comparisons | What humans prefer | Chatbots | Sycophancy, mode dropping, confident hallucination |
| **RLVR** | Automatic program/verifier | Verifiably correct answers | Reasoning models (math/code) | Slow, expensive, narrow domain; says nothing about confidence honesty |
| **RLCD** | Proper scoring rule (e.g. Brier score) comparing stated confidence to actual outcome | Calibrated, honest probabilities | Decision models for software automation | Requires verifiable outcomes; gives up free-text generation |
