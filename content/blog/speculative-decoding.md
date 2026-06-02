---
title: "Speculative Decoding"
date: 2026-06-03T00:12:34+02:00
draft: false
toc: true
---

> This is part 2 of the LLM inference series. Part 1: [I Kinda Understand LLM Serving](/blog/i-kinda-understand-llm-deployment/).

# Introduction

Greetings! In the [last post](/blog/i-kinda-understand-llm-deployment/), I went through continuous batching and PagedAttention. This time we are looking at **Speculative Decoding**, a technique that speeds up LLM generation without any loss in output quality.

The idea: use a small, fast model to guess what the big model would say, then verify all those guesses in one shot. If the guesses are correct, you got free tokens. If some are wrong, you fix them and move on. The output is mathematically identical to just running the big model directly. No quality loss. Just speed.

Two papers independently arrived at essentially the same idea around the same time:

1. Google's *"Fast Inference from Transformers via Speculative Decoding"* [1]
2. DeepMind's *"Accelerating Large Language Model Decoding with Speculative Sampling"* [2]

And at the end I will cover *Speculative Speculative Decoding* [3], a recent paper that takes the same idea one level deeper.

---

# Autoregressive Inference

Quick refresher on how decoding normally works.

When you send a prompt to an LLM, there are two phases:

**Prefill**: The model processes your entire prompt in one go. All input tokens are fed through the transformer in parallel. This is fast and GPU-friendly.

**Decode**: The model generates tokens one at a time. It samples token 1, feeds it back in, samples token 2, feeds that back in, and so on until done.

This second part is the bottleneck. Every single token generation requires a full forward pass through the entire model. And this is not slow because the GPU is not doing enough math. It is slow because of **memory bandwidth**.

A 70B parameter model has ~140GB of weights (in fp16). Every time you do a forward pass, those weights need to be read from HBM (the GPU's high bandwidth memory) into the compute units. That data movement is the bottleneck. When generating tokens one by one, you are doing this giant weight-loading exercise for every single token, even though you are only getting one token out of it each time.

As put in [2]:

> *"A memory bound model call only generates a single token for every sequence in the batch, hence generating multiple tokens introduces a large amount of latency in any system which makes use of it."*

So the real question becomes: can we load those weights once and squeeze out multiple tokens?

In code, normal autoregressive sampling looks like this:

```python
import torch

def autoregressive_sample(model, prompt_ids, max_new_tokens):
    input_ids = prompt_ids.clone()

    for _ in range(max_new_tokens):
        with torch.no_grad():
            logits = model(input_ids).logits  # full forward pass
        
        # sample next token from distribution
        next_token = torch.multinomial(
            torch.softmax(logits[:, -1, :], dim=-1), num_samples=1
        )
        input_ids = torch.cat([input_ids, next_token], dim=-1)

    return input_ids
```

One forward pass. One token. Repeat `max_new_tokens` times. For a 70B model generating 200 tokens, that is 200 full round-trips through 140GB of weights.

---

# Speculative Decoding

The key observation is that not every token is equally hard to predict. If I am generating `The Eiffel Tower is located in ___`, the word "Paris" is dead obvious, a tiny model would get it right. But for complex multi-step reasoning or something creative, the big model's judgment matters a lot more.

Speculative decoding delegates the "easy" tokens to a small, fast model. Instead of trying to figure out which tokens are "easy" upfront (which is hard), it lets the small model guess a batch of tokens, and then uses the big model to verify all of them at once. Correct guesses are free tokens. Wrong guesses get fixed.

The two models are:
- **Draft model** (`M_q`): small and fast, used for speculation
- **Target model** (`M_p`): big and accurate, used for verification

![Autoregressive vs Speculative Decoding](/images/speculative-decoding/autoregressive-vs-speculative.svg)

The win: if the small model gets 3 out of 4 tokens right, you got 3 tokens for the cost of roughly 1 big model call (plus some cheap small model calls).

## How Parallel Verification Works

When the draft model gives you `[guess₁, guess₂, guess₃, guess₄]`, you need to know:
- What would the target model predict at position 1 (given just the prefix)?
- What would it predict at position 2 (given prefix + guess₁)?
- What would it predict at position 3 (given prefix + guess₁ + guess₂)?
- ...and so on.

These look like 4 separate questions with 4 separate contexts, requiring 4 separate forward passes. But they do not, because of **causal masking**.

During training, transformers process the entire sequence at once and use a causal mask to make sure each position can only attend to positions *before* it. When the model processes token at position `i`, it only "sees" tokens at positions `0` through `i-1`. This is what makes teacher-forcing possible during training.

That exact same property works for verification. A single forward pass with the input:

```
[prefix_token_1, prefix_token_2, ..., prefix_token_t, guess₁, guess₂, guess₃, guess₄]
```

produces the output at position `t` giving us `p(· | prefix)`, position `t+1` giving us `p(· | prefix, guess₁)`, position `t+2` giving us `p(· | prefix, guess₁, guess₂)`, and so on. **All in one forward pass.**

![Parallel Verification via Causal Masking](/images/speculative-decoding/causal-mask-verification.svg)

A transformer's forward pass over a sequence of length `n` already computes next-token distributions at all `n` positions simultaneously. We are just using that existing property for verification instead of training.

In PyTorch, this forward pass looks like:

```python
import torch
import torch.nn.functional as F

def get_target_distributions(target_model, prefix_ids, draft_ids):
    """
    Run target model once on prefix + draft tokens.
    Returns the probability distributions at each draft position.
    """
    # Concatenate prefix + all draft tokens into one sequence
    full_sequence = torch.cat([prefix_ids, draft_ids], dim=-1)  # (1, prefix_len + gamma)

    with torch.no_grad():
        logits = target_model(full_sequence).logits  # (1, prefix_len + gamma, vocab_size)

    # We want distributions starting from the last prefix token position
    # Output at position (prefix_len - 1) predicts the first draft token
    # Output at position (prefix_len) predicts what comes after draft token 1
    # ...and so on
    prefix_len = prefix_ids.shape[-1]
    gamma = draft_ids.shape[-1]

    # Extract distributions at draft positions (and one extra for the bonus token)
    relevant_logits = logits[:, prefix_len - 1: prefix_len + gamma, :]  # (1, gamma+1, vocab_size)
    distributions = F.softmax(relevant_logits, dim=-1)  # (1, gamma+1, vocab_size)

    return distributions  # gamma+1 distributions: for each draft token + bonus
```

So you get `γ + 1` distributions out of a single forward pass. That is what makes the whole thing work.

## The Rejection Sampling Algorithm

Here is the algorithm from [2]:

```
Algorithm: SpeculativeDecodingStep
────────────────────────────────────────────────────────────────
Inputs: Mp (target model), Mq (draft model), prefix

1. Sample γ draft tokens from Mq autoregressively:
   for i = 1 to γ:
       qᵢ(x) ← Mq(prefix + [x₁, ..., xᵢ₋₁])
       xᵢ ~ qᵢ(x)

2. Run Mp once on the full sequence to get target distributions:
   p₁(x), ..., pᵧ₊₁(x) ← Mp(prefix), ..., Mp(prefix + [x₁, ..., xᵧ])
   (one parallel forward pass)

3. Find the first rejected draft token:
   For each i = 1 to γ:
       draw rᵢ ~ Uniform(0, 1)
       if rᵢ > pᵢ(xᵢ) / qᵢ(xᵢ):
           reject xᵢ, set n = i - 1, break
   if all accepted, n = γ

4. Sample a correction token at the rejection point:
   if n < γ:
       sample from norm(max(0, pₙ₊₁(x) - qₙ₊₁(x)))
   else:
       sample from pᵧ₊₁(x)   ← bonus token!

Return: prefix + [x₁, ..., xₙ, correction_token]
────────────────────────────────────────────────────────────────
```

The rejection sampling step (step 3) is the clever part. For each draft token, we accept it with probability `min(1, p(token) / q(token))`:
- If the big model agrees with the small model (or likes the token even more), we just take it.
- If the big model thinks this token is less likely than the draft model did, we accept it only sometimes, proportional to how much less likely it is.

Step 4 keeps the output distribution exactly correct. When we reject a token, instead of just sampling from `p` directly, we sample from `p - q` (normalized, with negative values clamped to 0). This correction accounts for the fact that we already rejected some probability mass. The result: the whole process produces tokens with the same distribution as running the big model alone.

![Rejection Sampling Flow](/images/speculative-decoding/rejection-sampling.svg)

Here is the full speculative decoding step in code:

```python
import torch
import torch.nn.functional as F

def speculative_decode_step(target_model, draft_model, prefix_ids, gamma=4):
    """
    One step of speculative decoding.
    Returns an extended prefix with 1 to gamma+1 new tokens.
    """
    vocab_size = target_model.config.vocab_size
    draft_ids = []
    draft_probs = []

    # Step 1: Draft gamma tokens using the small model
    current_ids = prefix_ids.clone()
    for _ in range(gamma):
        with torch.no_grad():
            draft_logits = draft_model(current_ids).logits[:, -1, :]
        draft_dist = F.softmax(draft_logits, dim=-1)  # (1, vocab_size)
        next_token = torch.multinomial(draft_dist, num_samples=1)  # (1, 1)
        
        draft_ids.append(next_token)
        draft_probs.append(draft_dist[0, next_token[0, 0]])  # scalar probability
        current_ids = torch.cat([current_ids, next_token], dim=-1)

    draft_ids_tensor = torch.cat(draft_ids, dim=-1)  # (1, gamma)

    # Step 2: Score all draft tokens with the target model in ONE forward pass
    full_seq = torch.cat([prefix_ids, draft_ids_tensor], dim=-1)
    with torch.no_grad():
        target_logits = target_model(full_seq).logits  # (1, prefix_len+gamma, vocab_size)

    prefix_len = prefix_ids.shape[-1]
    target_dists = F.softmax(
        target_logits[:, prefix_len - 1: prefix_len + gamma, :], dim=-1
    )  # (1, gamma+1, vocab_size) -- the +1 is the bonus token distribution

    # Step 3: Accept or reject each draft token
    n_accepted = 0
    for i in range(gamma):
        token_id = draft_ids[i][0, 0]
        p_i = target_dists[0, i, token_id]   # target model's prob for this token
        q_i = draft_probs[i]                  # draft model's prob for this token

        r = torch.rand(1).item()
        acceptance_prob = min(1.0, (p_i / q_i).item())

        if r < acceptance_prob:
            n_accepted += 1
        else:
            break  # reject this token and everything after it

    # Step 4: Sample the correction/bonus token
    if n_accepted < gamma:
        # Adjust distribution at the rejection point to correct for bias
        reject_dist = target_dists[0, n_accepted, :]   # p_{n+1}
        draft_dist_at_reject = F.softmax(
            draft_model(torch.cat([prefix_ids, draft_ids_tensor[:, :n_accepted]], dim=-1)).logits[:, -1, :],
            dim=-1
        )[0]   # q_{n+1}
        corrected = F.relu(reject_dist - draft_dist_at_reject)
        corrected = corrected / corrected.sum()
        bonus_token = torch.multinomial(corrected, num_samples=1)
    else:
        # All accepted! Sample a bonus token from the target model
        bonus_token = torch.multinomial(target_dists[0, gamma, :], num_samples=1).unsqueeze(0)

    # Return prefix + accepted draft tokens + correction/bonus token
    accepted_draft = draft_ids_tensor[:, :n_accepted]
    new_ids = torch.cat([prefix_ids, accepted_draft, bonus_token], dim=-1)
    return new_ids
```

In practice you would want to be smarter about KV cache management (the draft model builds up a cache too, which needs to be rewound on rejection) and batch multiple sequences, but this captures the core logic.

---

# Results

Both papers show solid speedups.

**DeepMind** [2] tested on Chinchilla 70B and got **2-2.5x speedup** in a distributed setup. The output distribution is provably unchanged. Not "approximately the same", literally identical.

**Google** [1] tested on T5-XXL (11B) and got **2-3x speedup** over their standard T5X implementation, also with identical outputs.

Two groups, same idea, same time period, similar results. That usually means the idea is right.

The speedup you get in practice depends on the **acceptance rate**, how often the draft model's guesses pass the rejection test. If the draft model is bad, most tokens get rejected and you are just doing extra work for nothing. If it is good, you can get 4-5 accepted tokens per target model call and the speedup is real. The acceptance rate also depends on the task: repetitive or formulaic text (code boilerplate, translations of common phrases) tends to have high acceptance rates. Creative or highly constrained generation tends to be lower.

---

# Further Improvements: Speculative Speculative Decoding

Standard speculative decoding still has one sequential dependency: you draft a batch, wait for verification, then draft the next batch. Even though verification is fast (one forward pass), you cannot start drafting the next batch until you know what got accepted.

*Speculative Speculative Decoding* by Kumar, Dao, and May [3] (ICLR 2026) attacks this. The name is recursive on purpose.

The idea (called SSD) is to overlap drafting and verification: while the target model is busy verifying batch N, the draft model is *already* working on batch N+1. It speculatively predicts what the verification outcome will be and generates tokens based on that prediction. If the prediction is right, batch N+1 is ready to go the moment verification finishes, zero wait. If it is wrong, you fall back.

![Speculative Speculative Decoding Pipeline](/images/speculative-decoding/ssd-pipeline.svg)

The hard part is that verification does not always accept all `γ` tokens, it might accept 2, or 4, or 0. So the draft model has to prepare for multiple possible outcomes simultaneously. The paper's Saguaro algorithm handles this and gets **~30% speedup on top of already-optimized speculative decoding**, and up to **5x over standard autoregressive decoding** on open-source engines.

I find this recursive application of the same trick to be a neat idea. Using speculation to hide the latency of the verification step itself.

---

# Wrapping Up

The main takeaway: autoregressive decoding is bottlenecked by memory bandwidth, not compute. You are paying the cost of loading model weights for every single token. Speculative decoding amortizes that cost by verifying multiple tokens per target model call, using causal masking to make the verification parallel for free. And because of the rejection sampling math, the output is identical to what you would get from just running the big model.

Speculative decoding has been picked up widely in production systems and is now supported in most major inference frameworks (vLLM, TensorRT-LLM, etc.). If you are doing anything with LLM inference at scale, it is worth understanding properly.

Next in the series, I will probably look at either continuous batching or disaggregated prefill-decode. Till then. Peace!

---

**References:**
1. Yaniv Leviathan, Matan Kalman, Yossi Matias. *Fast Inference from Transformers via Speculative Decoding*. ICML 2023. [arXiv:2211.17192](https://arxiv.org/abs/2211.17192)
2. Charlie Chen, Sebastian Borgeaud, Geoffrey Irving, Jean-Baptiste Lespiau, Laurent Sifre, John Jumper. *Accelerating Large Language Model Decoding with Speculative Sampling*. 2023. [arXiv:2302.01318](https://arxiv.org/abs/2302.01318)
3. Tanishq Kumar, Tri Dao, Avner May. *Speculative Speculative Decoding*. ICLR 2026. [arXiv:2603.03251](https://arxiv.org/abs/2603.03251)