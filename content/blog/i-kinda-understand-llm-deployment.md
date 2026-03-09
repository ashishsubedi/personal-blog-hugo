---
title: "I Kinda Understand LLM Serving"
date: 2026-03-09T00:03:00+01:00
draft: false
toc: true
---

> LLM is used in every stage of this blog (learning, coding, and writing of this blog). 

# Introduction

Greetings! Welcome aboard to my humble blog. Recently, I have been trying to understand the internals of LLM serving. "Why?" you ask? Okay, let me rant. I use Ollama for local model deployment on my M4 MacBook. What I noticed is that it struggles to handle concurrent requests efficiently. I gave vLLM a try, and the difference in high-throughput handling was noticeable. Long story short, vLLM is awesome for high-throughput serving -> wanted to understand exactly what sets it apart from Ollama -> read the original paper published by the vLLM team -> learned about continuous batching and PagedAttention -> spent hours vibecoding and benchmarking my own implementation -> and now I am blogging about it!

[Source code](https://github.com/ashishsubedi/ingine)

# Chapter 1: Let's go basic

At its core, a LLM is just a massive collection of numbers organized into matrices. A model is just specific weights and biases saved to the disk. These days, everyone uses `.safetensors`. 

So, once we have got weights on disk, how do we actually run them? We can start with Hugging Face's `transformers` library. We load the model, call `model.generate()`, and it loops auto-regressively: predict next token, append it to the prompt, repeat. Simple, works great for testing. But here's the catch: it is sequential and inefficient.

Then, Georgi Gerganov came along and built GGML, a custom ML tensor library in pure C/C++. He quantized models down to formats like 4-bit integers, slashing memory usage, and optimized the math to run natively on CPUs and Apple Silicon. This evolved into the GGUF file format and the `llama.cpp` inference engine, which is what actually powers Ollama. Ollama basically wraps this hyper-optimized C++ backend (although they recently have their own implementation, we are going to forget about that for now 😉).

# Chapter 2: KV Cache & Batching Problem

Alright, so we have got models running, let's think about how to make it efficient. LLM inference is **memory-IO bound**, not compute bound. That means GPU spends more time loading weights from memory than actually doing math. To maximize throughput, we need batching. Processing multiple requests at once ensures that we are not reloading the massive weight matrices for every single token.

## KV Cache
When an LLM generates text, it does so auto-regressively. To predict the 100th token, the attention mechanism needs to look at the context of the previous 99 tokens. If we recompute the mathematical representations for those 99 tokens every single step, our compute costs scale quadratically.

The solution is the KV Cache. During the initial `prefill` phase (when the user submits the prompt), the model calculates the Key and Value matrices for the entire prompt and saves them to GPU memory. During the `decode` phase (generating new tokens one by one), the model only computes the Query for the single new token and compares it against the saved Key and Value history.

This optimization trades compute for memory footprint. We save massive amounts of math, but the VRAM usage explodes because it have to hold these massive tensor histories in memory for as long as the request is active.

## Static Batching: The Naive Approach

To maximize throughput and avoid reloading the model weights for every single token, we need batching. The simplest approach is static batching. You grab N requests, process them together, and wait until every single one finishes before starting the next batch. Sounds reasonable, right? But this is tricky as well. The main issue is that not all sequences finish at the same time. For example, one requests asks for a quick yes/no answer (2 tokens), another writes an essay (500 tokens). With static batching, the GPU sits there idle, waiting for the longest sequence to complete while the shorter ones have already finished. This is a massive waste of compute.

## Continuous Batching: Iteration-Level Scheduling

Enter continuous batching (also called dynamic batching or iteration-level scheduling). The idea behind this is that instead of waiting for the entire batch to finish, just swap in new requests as soon as any sequence completes. In this approach, every iteration (every single forward pass), the scheduler checks if anyone finished. If they did, it slots in a new request. This keeps the GPU completely saturated. 

## The Memory Fragmentation Problem
Even with continuous batching, we hit another wall: memory fragmentation. Every request has a different sequence length, which means their KV caches are all different sizes.

Traditional frameworks like PyTorch require tensors to be perfectly rectangular. To make the math work, we have to pad the smaller sequences with zeros to match the longest one in the batch. This wastes massive amounts of VRAM and forces constant reallocation as sequences grow. We end up running out of memory long before run out of compute power.

## PagedAttention: Virtual Memory for LLMs
This is where vLLM comes in with `PagedAttention`. The core insight is to treat GPU memory just like an OS manages RAM by using paging and virtual memory.

Instead of allocating one giant contiguous block per sequence, PagedAttention splits the KV cache into fixed-size blocks (16 tokens per block). Each request gets a block table, which is just a list of pointers to these physical blocks.

The blocks do not need to be contiguous in memory. When you need more space, you just grab another block from the pool. Done.

This practically eliminates fragmentation. According to vLLM's benchmarks, memory wastage drops to under 4%, occurring only in the last partially-filled block. The memory savings translate directly into higher batch sizes, which means much higher throughput.

# Chapter 3: Building It Myself

Okay, theory is cool, but I wanted to understand it practically. So, with the power bestowed upon me by LLM, I built my own continuous batching engine with PagedAttention in PyTorch. [Source code](https://github.com/ashishsubedi/ingine)

## The Architecture

Instead of letting PyTorch dynamically allocate and concatenate tensors for the KV cache, I bypassed the default caching entirely. Here is how it works:

1. **The Physical Hardware**: At startup, I pre-allocate a massive 5D tensor called the `KVCachePool`. I reserve all the memory exactly once.
2. **The Memory Manager**: Requests no longer hold contiguous tensors for their history. Instead, they hold a `block_table`. This is just a list of integers pointing to 16-token physical blocks scattered in the pool.
3. **Continuous Batching**: The scheduler acts as a state machine. It disaggregates the prefill and decode phases. It can pause decoding to slip a heavy prefill step into the batch, and then resume generating tokens for all active requests concurrently.

## Monkey-Patching Qwen2.5

I used `Qwen/Qwen2.5-1.5B-Instruct` for this project. To make the Hugging Face weights talk to my custom memory pool, I had to monkey-patch the model at runtime.

I replaced `Qwen2DecoderLayer.self_attn` with a custom `CustomQwenPagedAttention` class. By passing `use_cache=False` to the forward pass, I disabled Hugging Face's internal `DynamicCache`. Instead, my custom attention layer calculates the physical addresses from the request's `block_table` and writes the Keys and Values directly into the scattered physical memory pool.

One of the trickiest parts was handling Rotary Positional Embeddings (RoPE). Because the model no longer tracks its own cache history natively, it loses its sense of token order. I had to manually track the `logical_length` of every request in the scheduler and calculate the exact `position_ids` to pass down into the attention layer so the RoPE math remained accurate.

## The Apple Silicon (MPS) Float16 Bug

Since I do my local development on a Mac, I ran into a severe hardware-level bug. During testing, the model suddenly started outputting endless exclamation marks (!).

After some time of debugging, I found out the issue was rooted in Apple's MPS backend. During the prefill phase, I was applying a strict lower-triangular causal mask using `float("-inf")` to prevent tokens from attending to the future. It turns out that applying `-inf` inside a `float16` matrix multiplication on MPS causes numerical instability, generating `NaN` values. These `NaN`s poisoned the entire KV cache pool.

The fix was forcing the MPS compiler to give me more precision headroom. I had to upcast the attention scores to `float32` right before masking, apply a safe `-10000.0` instead of `-inf`, and then cast the tensor back down to `float16` before the softmax operation.

## Benchmarks and Trade-offs

To verify the system, I wrote a benchmark simulating 8 concurrent API requests with varying prompt and generation lengths. I compared my custom Paged Continuous Batching engine against the vanilla sequential Hugging Face model.

My custom engine actually had a slower total wall-clock time and lower overall throughput. This happens because my memory manager uses a `mock_gather` function to stitch the scattered physical blocks back together in a Python `for` loop. Hugging Face avoids this overhead by using highly fused, vectorized C++ SDPA kernels on contiguous memory.

However, the Time-To-First-Token (TTFT) and the end-to-end user latency improved drastically in my custom engine. Because the scheduler dynamically interleaves the generation steps, no request is stuck waiting idle in a queue.

In a real-world API, maximizing total throughput does not always equal a good user experience. Slower matrix math is often an acceptable trade-off if it means eliminating head-of-line blocking for the end user. This project was a great way to understand exactly why pure PyTorch is strictly a control-plane language, and why tensor manipulation needs to be pushed down to hardware-level kernels to achieve true speed.

# Wrapping Up

So, what did I learn? Continuous batching is a fundamental for how we think about LLM serving. The move from static to iteration-level scheduling unlocks massive efficiency gains, and PagedAttention takes it even further by treating GPU memory like a proper OS.

The Anyscale benchmarks showed 23x improvements with vLLM. My PyTorch implementation could not match that, but the latency improvements were very real. More importantly, I finally get why Ollama struggles with concurrency while vLLM crushes it.

Anyway, that is all for now. Peace!
