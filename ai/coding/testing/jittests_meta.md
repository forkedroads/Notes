# JiTTests: Catching Bugs Before They Land — Summary

## 1. The Idea

**JiTTests (Just-in-Time Tests)** is a fundamentally different approach to catching
bugs in code changes, compared to today's dominant paradigm of LLM/agent-based
code *review*.

- **Current prevalent approach**: An LLM reads a diff and reasons about it like a
  human reviewer — "this looks risky," "you might have forgotten X" — without
  executing anything. It's fast, but it's an *opinion*, not *proof*.
- **JiTTests approach**: Instead of reasoning about the diff, the system
  **generates an executable test** that proves a *concrete behavioral difference*
  between the code before and after the change. A failing test isn't a guess —
  it's hard evidence that something changed.

> **Core principle**: Treat the diff itself as a *mutant* (in the mutation-testing
> sense) and try to generate a test that "kills" it — i.e., finds an input where
> the old code and new code disagree.

## 2. Why Is This Needed?

- Meta's codebase has **hundreds of millions of lines of code**, changing
  continuously. Traditional, hand-written unit tests can't keep pace — humans
  can't anticipate every edge case for every diff.
- LLM-based review catches *style smells* and *known bug patterns*, but it never
  **executes** code, so it can miss subtle behavioral regressions that only
  surface when you actually run the new logic against specific inputs.
- The goal: **catch risk before code lands**, not after it's in production —
  cheaply enough, and with few enough false alarms, that it doesn't slow down
  engineering velocity.

## 3. The Two Workflows — and How They Work Together

| Workflow | Strategy | Role |
|---|---|---|
| **Dodgy Diff** | Treats *every* diff as inherently buggy. Uses mutation-guided generation purely from the syntactic delta (no understanding of intent). | Casts a **wide net** — maximizes the pool of candidate "weak catches" to search through. High recall, lower precision. |
| **Intent-Aware** | Uses an LLM to infer *why* the change was made (from diff title, summary, code), then hypothesizes specific plausible mistakes an engineer could make while implementing that intent, and generates targeted tests for those risks. | **Precision-guided** — focuses effort on the specific ways *this* change could plausibly go wrong, catching risks a purely syntactic scan might miss. |

**They operate as a tandem pipeline**, not a choice between one or the other:
- Dodgy diff finds divergences the moment code structurally changes — cheap, broad coverage.
- Intent-aware finds divergences that require *semantic* understanding of what the engineer was trying to achieve — deeper, narrower coverage.
- Together, they maximize the chance that **if a real bug exists, some generated test will "kill" the diff and surface it as a weak catch.**

Both workflows produce failing tests ("weak catches") — proof of behavioral divergence, but *not yet proof of a bug*, since the divergence could be intentional.

## 4. Improving Signal-to-Noise Ratio (SNR)

A raw failing test only proves "behavior changed," not "this is wrong." Meta
layers two filtering mechanisms on top of every weak catch to separate real bugs
(**strong catches**) from false alarms:

- **LLM-as-Judge (ensemble)**: Multiple state-of-the-art LLMs independently
  evaluate the failing test code, its execution trace, the diff itself, and the
  inferred intent — then vote on whether the divergence **violates** the intent
  (→ real bug) or is **consistent with** it (→ intended change, suppress).
- **RubFake (rule-based assessor)**: A cheaper, deterministic layer that catches
  known false-positive patterns (e.g., flakiness signatures) before expensive LLM
  reasoning is even needed.
- **Human review is preserved**, but reserved for cases that survive both filters —
  dramatically reducing how often engineers are pulled in, while keeping labeling
  consistency with how a human would judge the same catch.

**Net effect**: cheap, mechanical "did behavior change?" checks do the bulk
filtering; expensive semantic "was this change intended?" judgment only touches
a small, pre-filtered set of survivors — preserving engineer trust without
drowning them in noise.

## 5. Corollary: Property-Based & Invariant Testing

JiTTests isn't entirely novel in isolation — it's best understood as a
**diff-scoped, disposable, LLM-driven version** of two classic testing
philosophies:

- **Property-based testing (PBT)**:
  - Instead of hand-written example assertions,
  you state a general property and let a framework search for falsifying inputs.
  JiTTests does this implicitly — the "property" being checked is *"parent and
  child should behave identically except where intended,"* and the search for a
  falsifying input is focused directly on the diff's changed region (smarter than
  PBT's typically random search). Example : Commutativity, associativity etc
  - Property-based testing isn't about testing specific input/output pairs; it's about testing general rules that should hold across a whole class of inputs.
- **Invariant testing**: Classic invariants are explicit, hand-coded conditions
  that must always hold (e.g., *"account balance never goes negative"*).
  JiTTests derives invariants **on the fly**, per diff, from inferred intent —
  used once to validate this specific change, then discarded (rather than living
  permanently in the codebase).

## Reference:
https://engineering.fb.com/2026/10/29/security/catching-bugs-before-they-land-jittestings-human-on-the-loop-approach-at-full-scale/
