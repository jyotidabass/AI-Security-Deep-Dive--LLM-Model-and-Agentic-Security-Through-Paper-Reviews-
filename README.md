# AI Security Deep Dive: LLM Security Through Paper Reviews

Two papers. One shows how to **break** LLM safety. One shows how to **fix** it.

---

## What's in This Repo

| File | What it is |
|------|------------|
| `Adversarial Reasoning at Jailbreaking Time.pdf` | How attackers use smarter reasoning to trick LLMs into saying harmful things |
| `SAFELLM-AGENT PAPER.pdf` | How to permanently delete harmful knowledge from inside an LLM |

---

## Background: What is Jailbreaking?

LLMs like ChatGPT are trained to refuse harmful requests. **Jailbreaking** means finding a prompt that tricks the model into ignoring that refusal and producing harmful output anyway.

Think of it like this: the model has a security guard at the door. Jailbreaking finds a disguise that fools the guard.

---

## Paper 1: Adversarial Reasoning at Jailbreaking Time

**What it's about:** A smarter, more systematic way to jailbreak LLMs — including ones specifically hardened against attacks.

### The Problem with Old Attack Methods

There were two kinds of jailbreak attacks before this paper:

**Type 1 — Token attacks**
- Scramble random characters onto a prompt until the model breaks.
- Problem: These prompts look like nonsense. A simple filter catches them.

**Type 2 — Prompt attacks**
- Use another LLM to write clever, human-readable jailbreak prompts.
- Problem: These only get a yes/no signal — "did it work or not?" That's not enough info to improve efficiently against hardened models.

### What This Paper Does Differently

Instead of yes/no feedback, they use a **continuous loss score** — a number that tells the attacker *how close* they are to succeeding, not just whether they did.

This is like the difference between:
- ❌ "You're wrong" (binary)
- ✅ "You're 73% of the way there" (continuous)

With this signal, they run a **tree search** — try many paths, score each one, keep the best, backtrack from dead ends. The same technique that makes LLMs good at math is now being used to attack them.

### How the Attack Works (3 Steps)

```
Step 1 — REASON
  An "Attacker LLM" writes a jailbreak prompt using step-by-step reasoning.

Step 2 — VERIFY
  A "Feedback LLM" checks the loss score from the target model's output.
  Low loss = getting closer to a harmful response.

Step 3 — SEARCH
  A tree search keeps the best reasoning paths, prunes bad ones,
  and backtracks when stuck — like a chess engine exploring moves.
```

These 3 steps repeat until the jailbreak succeeds or runs out of tries.

### Results

- ✅ Best-in-class success rates against safety-trained models
- ✅ **100% success against DeepSeek**
- ✅ **56% success against OpenAI o1-preview** (which is specifically hardened)
- ✅ More compute = more jailbreaks found. It scales.
- ✅ Even a weak attacker LLM works well when using this framework

### The Key Insight

The same techniques used to make LLMs *smarter* (reasoning, tree search, test-time compute) can be directly used to make attacks *more effective*. There's no clean separation between capability and attack power.

---

## Paper 2: SafeLLM — Deleting Harmful Knowledge from LLMs

**What it's about:** A way to remove harmful knowledge *inside* an LLM — so even if someone jailbreaks it, the harmful output is no longer there to give.

### Why Normal Defenses Aren't Enough

Most LLM safety comes from training techniques like RLHF or fine-tuning. These teach the model to *refuse* harmful requests. But:

- The harmful knowledge is still **in the model weights** — the model just learned not to say it.
- A good jailbreak bypasses the refusal and reaches that knowledge anyway.
- It's like teaching someone not to talk about a secret — they still know the secret.

SafeLLM goes deeper: it tries to **erase the secret itself**.

### How SafeLLM Works (3 Stages)

**Stage 1 — Detect the harmful output**

When a harmful response is generated, SafeLLM catches it using two things:
- An external toxicity classifier (a separate model that flags bad content)
- The LLM's own internal scoring (it asks itself: "was this harmful?")

Using both together is more reliable than either alone.

**Stage 2 — Find where inside the model the harm came from**

LLMs are made of layers. Inside each layer are **Feedforward Networks (FFNs)** — these store the model's knowledge and learned behaviors.

SafeLLM traces which specific FFN neurons fired when the harmful tokens were generated. This pinpoints *which part of the model* produced the bad output — like finding which wire in a circuit caused a short.

**Stage 3 — Surgically suppress those neurons**

SafeLLM runs **constrained optimization** to reduce those specific neurons' contribution — without touching the rest of the model. It also uses adversarial training during this step so the harmful behavior can't come back through a slightly different prompt.

```
Normal defense:  Teach model to say "I won't answer that"
                 → Harmful knowledge still exists inside the weights

SafeLLM:         Find and suppress the neurons that store harmful knowledge
                 → Harmful knowledge is actually gone
```

### Results

Tested on Vicuna, LLaMA, and GPT-J:
- ✅ Significantly lower Attack Success Rate (ASR) vs. all baselines
- ✅ Outperforms SFT, DPO, Eraser, and other unlearning methods
- ✅ General performance preserved on OpenBookQA and TruthfulQA benchmarks
- ✅ Works against unseen jailbreak prompts, not just known attack patterns

---

## How the Two Papers Connect

| | Paper 1 (Attack) | Paper 2 (Defense) |
|--|--|--|
| **Core idea** | Use reasoning + loss signal to find jailbreaks | Use FFN tracing to erase harmful knowledge |
| **Key insight** | Smarter search beats binary feedback | Erasing beats refusing |
| **What it bypasses/targets** | The refusal layer of aligned models | The knowledge stored in FFN weights |
| **Main result** | SOTA attack success rates | Lower ASR, preserved model utility |

**Together they say:** Alignment-based safety (RLHF, SFT) is not enough on its own. Attackers can reason around it. True safety needs structural changes inside the model — not just better training signals on top.

---

## Quick Glossary

| Term | Simple meaning |
|------|---------------|
| **Jailbreaking** | Tricking an LLM into ignoring its safety rules |
| **Loss score** | A number measuring how far the model's output is from a target — lower = closer to harmful |
| **Chain-of-thought (CoT)** | Making an LLM reason step by step before answering |
| **Tree search** | Explore many paths, score them, keep the best, backtrack from dead ends |
| **RLHF** | Training method where humans rate outputs to guide model behavior |
| **Fine-tuning / SFT** | Retraining a model on specific examples to adjust its behavior |
| **Machine unlearning** | Selectively removing specific knowledge from a trained model |
| **FFN (Feedforward Network)** | Layers inside a transformer that store factual and behavioral knowledge |
| **Attack Success Rate (ASR)** | % of jailbreak attempts that successfully extracted harmful output |
| **Constrained optimization** | Modify specific parts of a model while keeping everything else intact |

---

## 4 Key Takeaways

1. **Jailbreaking is now a reasoning problem.** Attackers don't brute-force prompts anymore — they run structured searches guided by model feedback. Much harder to defend against.

2. **Refusal ≠ safety.** A model that says "no" can still be jailbroken. A model that doesn't *know* the harmful thing is much safer.

3. **Compute scales attacks too.** More inference-time compute doesn't just make models smarter — it makes attacks more effective. This matters a lot for frontier models.

4. **Token-level unlearning is the right direction.** Surgical removal of harmful knowledge from FFN layers is more robust than alignment fine-tuning, and doesn't hurt general performance.

---

> ⚠️ **Disclaimer:** These papers contain examples of harmful prompts and outputs for research purposes only. All content is intended for defensive security research.
