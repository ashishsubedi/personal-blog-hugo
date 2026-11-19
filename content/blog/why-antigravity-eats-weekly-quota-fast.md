---
title: "Why Antigravity Eats Weekly Quota Fast - and What I Measured"
date: 2026-11-19T01:55:00+01:00
draft: false
toc: true
---

> Gemini 4 Hydrogen, Gemini 4 Argon, and `hy4-flash` via OpenCode used for the binary reverse engineering, empirical cache probing, and keep-alive tooling.

# Introduction

Greetings! If you use Google Antigravity (`agy`) with the big frontier models, you have probably noticed how fast your weekly quota disappears during normal coding sessions [1, 4].

My suspicion was simple: **a short prompt-cache lifetime**. When you are coding with an agent, you don't send messages every 10 seconds. You wait for a response, read the diff, run a test, think for 5 minutes, and then send your next prompt. If the prompt cache expires during that 5-minute pause, your next message has to re-ingest the entire conversation history at full price.

Instead of just assuming that was true, I decided to actually test it and look inside the Antigravity binaries to see how caching works under the hood. What started as a quick TTL check turned into disassembling a 193 MB Go binary, discovering a dormant keep-alive engine that Google already wrote, binary-searching the exact eviction cliff down to a 15-second window, testing whether `/btw` side-questions can keep the cache warm, and finally unlocking Google's built-in zero-trajectory keep-alive engine on a live 6-minute run.

---

# 1. Background: Implicit Caching & The Idle Tax

To understand why pauses matter, let's look at how Gemini models handle context in Antigravity.

Every turn in an agentic coding session carries a massive prefix: the system prompt, tool schemas, connected MCP servers, skill definitions, and all previous turns. On Gemini 4 models, Antigravity relies exclusively on **implicit caching** [2, 3]. Unlike explicit caching (where you manually place cache breakpoints and pay for a guaranteed storage lease), implicit caching is an automatic, best-effort prefix match on Google's backend.

When your request hits the implicit cache, cached tokens (`cache_read_tokens`) get a **90% discount** compared to regular uncached `input_tokens` [2]. When the cache misses, you pay 100% on the entire prefix.

Google does not publish a fixed TTL for implicit caching. If that TTL is shorter than a typical code-review pause, interactive coding turns into a worst-case workload.

---

# 2. What the Binary Says

Before running live probes, I inspected the local Antigravity installation on macOS (`arm64`): the desktop app (`/Applications/Antigravity.app`, `v2.33.0`), its core Go agent binary (`language_server`), and the standalone CLI (`~/.gemini/bin/agy`, `v1.5.11` / `1.5.12`) across isolated CLI runs.

## 2.1 Telemetry Buckets Give Away the 5-Minute Cliff

Inspecting the Go symbol table in `language_server` immediately confirmed that Antigravity tracks cumulative context through `CumulativePromptHandler` and buckets turn gaps into two telemetry categories: **`lt_5m`** (less than 5 minutes) and **`ge_5m`** (5 minutes or more). That alone was a strong clue that 5 minutes is the magic number.

## 2.2 Google Already Built a Keep-Alive Engine

Digging deeper, I found a file path in the binary's symbol table: `modelapiccpa/keepalive.go`. Disassembling its functions (`keepAlivePings`, `beforeRequest`, `scheduleNextPingLocked`, `ping`, and `trimLastBlock`) revealed that **Google already built an automatic background prompt-cache keep-alive mechanism.**

Here is how their internal keep-alive works:
1. **Zero-Trajectory Pollution**: When a turn finishes (`onRequestDone`), the client schedules a background timer. When the timer fires, it clones the last request protobuf, calls `trimLastBlock` to replace the final prompt block with a single period (`"."`), sets `max_output_tokens = 1`, tags the request with `"prompt_cache_keep_alive": "true"`, and sends it in the background. When the 1-token response comes back, it throws it away so it **never pollutes your conversation history**.
2. **Exact Hardcoded Timers**:
   - **Ping interval**: Hardcoded in `scheduleNextPingLocked` to **4 min 30 s** (`270s`).
   - **Staleness cutoff**: Hardcoded in `ping` to **4 min 45 s** (`285s`). If the timer wakes up more than 4m 45s after the last turn (e.g., after laptop sleep), it aborts the ping because it knows the server cache is already dead.

Why do we still lose the cache every 5 minutes? Two gates in the binary stop it from running:
- **Gate A (`keepAlivePings`)**: Locked behind server-side Mendel experiment flags (`enable-prompt-cache-keep-alive` and `prompt_cache_keep_alive_max_pings`) that aren't exposed in `settings.json` or Antigravity hooks.
- **Gate B (`beforeRequest`)**: Checks the internal model provider enum (`0x5` or `0xa`) and currently restricts background pings to explicit-cache provider enums (like Claude), completely skipping Gemini models.

---

# 3. Experimental Setup: Measuring Cache Hits

Since account-level quota percentages are noisy (other background agents share the same quota pool), I used `cache_read_tokens` from `agy --input-format stream-json --output-format stream-json` as the primary ground-truth signal.

At the end of every turn, `agy` emits a `DONE` event for `agent_response` with three numbers:
- **`input_tokens`** (`T_uncached`): Tokens billed at full price.
- **`cache_read_tokens`** (`T_cached`): Prefix tokens served from the implicit cache.
- **`output_tokens`**: Generated response tokens.

| Observed Signal | What It Means |
| :--- | :--- |
| Large `cache_read_tokens` (~14,770), small `input_tokens` (~2,150) | **Cache Hit** - prefix served from cache, only tail billed at full price. |
| `cache_read_tokens = 0`, large `input_tokens` (~16,820+) | **Cache Miss** - full prefix re-ingested from scratch. |

For the baseline TTL tests, I used **`gemini-4-hydrogen-low`** with `--effort low` and the prompt `"Reply with only OK."` so output tokens stayed tiny (5 to 25 tokens). To run multi-turn conversations inside a single long-lived process (matching how the desktop app's `language_server` runs), I fed NDJSON user events into `agy` through a persistent FIFO pipe.

---

# 4. What I Measured

## 4.1 Separate CLI Processes Don't Share the Cache

First, I tried running two separate one-shot `agy -p --conversation <same-id>` commands back-to-back:

| Mode | Turn 1 (`input` / `cache_read`) | Turn 2 (`input` / `cache_read`) | Result |
| :--- | :---: | :---: | :---: |
| **Separate processes** (`agy -p`) | `16,818` / `0` | `16,818` / `0` | **Miss** |
| **Same process** (`stream-json`) | `16,816` / `0` | `2,152` / `14,762` | **Hit** |

Two things stood out immediately:
1. Even on a brand-new conversation with a 5-word prompt (`"Reply with only OK."`), Turn 1 sends **16,816 tokens** (system prompt, tool declarations, MCP servers, and skills).
2. In the same process, Turn 2 hits the cache for **14,762 tokens**, leaving only **2,152 uncached tokens**. Across separate one-shot `agy -p` processes, `cache_read` is always `0` because one-shot mode re-initializes the step timestamp at step 0, breaking prefix continuity.

## 4.2 The 8-Minute Idle Test (With vs. Without a Ping)

Next, I tested an **8-minute idle gap** with no activity, and compared it against an 8-minute window with a single keep-alive ping at **+4 minutes**:

| Scenario | Turn | Time | `input_tokens` | `cache_read_tokens` | What Happened |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **8-min idle (No Ping)** | 1 | `0:00` | 16,824 | 0 | Cold start |
| | 2 | `+8:00` | 16,921 | 0 | **Cache Miss (Full re-ingest)** |
| | 3 | `+8:02` | 2,146 | 14,872 | Cache Hit (Turn 2 rebuilt it) |
| **8-min idle (With 4-min Ping)** | 1 | `0:00` | 16,823 | 0 | Cold start |
| | 2 (Ping) | `+4:00` | 2,160 | 14,769 | **Cache Hit (Refreshes TTL)** |
| | 3 | `+8:00` | 2,144 | 14,880 | **Cache Hit** |

This proved two things:
- Without a ping, waiting 8 minutes forces Turn 2 to pay full price (`16,921` tokens), which rebuilds the cache for Turn 3.
- With a ping at `+4:00`, the cache stays warm right through `+8:00`. **Hitting the implicit cache resets its TTL timer.**

## 4.3 Binary Search for the Exact Expiry Boundary

To find the exact cliff where the cache dies, I ran a binary search across idle intervals:

| Idle Gap | `cache_read_tokens` on Next Turn | Result |
| :---: | :---: | :--- |
| `0 s` | 14,762 | **Hit** |
| `4 min 00 s` (`4.00 min`) | 14,769 | **Hit** |
| `4 min 30 s` (`4.50 min`) | 14,769 | **Hit** |
| `4 min 45 s` (`4.75 min`) | 14,764 | **Hit** |
| `5 min 00 s` (`5.00 min`) | 0 | **Miss** |
| `8 min 00 s` (`8.00 min`) | 0 | **Miss** |

The boundary is between **4 min 45 s and 5 min 00 s**. That matches both my initial 5-minute estimate and the **4 min 45 s (`285s`)** staleness cutoff hardcoded inside Google's `keepalive.go`.

---

# 5. Cost Analysis & Scaling Math

Since Gemini gives a 90% discount on cached tokens [2], I calculated the **effective token cost** (`C`) of any turn by weighting `cache_read_tokens` at `0.1x`:

$$C = T_{\text{uncached}} + 0.1 \times T_{\text{cached}}$$

## 5.1 Baseline Cost (~16.8k Prefix)

In my measured 8-minute test window:
- **Cold / expired turn**: ~16,900 effective tokens
- **Warm turn / keep-alive ping**: `2,150 + 0.1 * 14,770` = ~3,640 effective tokens

| Scenario (8-Minute Window) | Messages | Uncached Input | Cached Read | Effective Tokens |
| :--- | :---: | :---: | :---: | :---: |
| No keep-alive, 8-min gap | 2 | 33,745 | 0 | **33,745** |
| No keep-alive, 8-min gap + next turn | 3 | 35,891 | 14,872 | **37,378** |
| **With 4-min keep-alive ping** | 3 | 21,127 | 29,649 | **24,092** |

Even on an empty conversation with just the 14.8k base prefix, one well-timed ping saves **9,653 to 13,286 effective tokens** (**29-36% savings**), a **3.7x return** on the cost of the ping.

## 5.2 What Happens in a Real 150k-Token Coding Session

My test conversation had almost zero history (`~14.8k` cached prefix). In a real coding session where you have read several files and built up a **150,000-token context** (`~147,850` cached prefix + `~2,150` uncached tail), the numbers scale quickly:

| Turn Type (150k Context) | Uncached Input | Cached Read (`0.1x`) | Effective Tokens |
| :--- | :---: | :---: | :---: |
| **Cold turn (after 5-min pause)** | 150,000 | 0 | **150,000** |
| **Warm turn / keep-alive ping** | 2,150 | 147,850 (`14,785` eff.) | **16,935** |
| **Net savings from 1 ping** | - | - | **133,065 (~8.9x ROI)** |

In a 150k-token session, pausing for 5 minutes to read a diff, read a response, or grab a coffee costs you **133,065 extra effective tokens** on your very next message.

## 5.3 When to Stop Pinging: The Break-Even Cap (`N_max`)

Keep-alive is cheaper than a miss, not free. If you walk away from your desk for two hours and let a script ping every 4 minutes, you will burn far more tokens pinging than if you just let the cache expire once.

The maximum number of consecutive idle pings (`N_max`) before keep-alive costs more than a single cold miss (`C_cold - C_warm = 0.9 * T_cached`) is:

$$N_{\text{max}} = \left\lfloor \frac{0.9 \times T_{\text{cached}}}{T_{\text{uncached}} + 0.1 \times T_{\text{cached}}} \right\rfloor$$

- At **16.8k context**: `N_max = floor(13,320 / 3,630)` = **3 pings** (~13.5 minutes of pings, keeping the cache warm for ~18 minutes of idle time).
- At **150k context**: `N_max = floor(133,065 / 16,935)` = **7 pings** (~31.5 minutes of pings).

This is why Google's internal `keepalive.go` tracks a `prompt_cache_keep_alive_max_pings` budget per conversation.

---

# 6. The `/btw` Experiment: Can Side-Questions Keep the Cache Warm?

Sending a normal user message as a keep-alive in the CLI has one downside: each ping adds a user and assistant turn to your trajectory.

Antigravity has a `/btw` slash command that lets you ask a side question in a forked thread without adding anything to the main conversation history. I tested whether `/btw` could act as a clean keep-alive.

Because `/btw` is blocked in print and stream mode, I drove the interactive TUI through a pseudo-terminal (`pty`) and read the resulting token counts from `transcript.jsonl`:

| Thread | Event | Elapsed Time | `input_tokens` | `cache_read_tokens` | Result |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **Main Conversation** | Turn 1 | `0:00` | 16,824 | 0 | Cold start |
| **`/btw` Fork** | Side question | `+3:31` | 17,697 | 0 | **Cold Miss (Fork re-ingest)** |
| **Main Conversation** | Final Turn | `+6:41` | 2,151 | 14,770 | **Warm Hit!** |

Here is what happened in the trace:
1. **`/btw` did keep the main cache warm**: The final normal turn happened at `+6 min 41 s`, past the 5-minute TTL cliff. Without `/btw` at `+3:31`, it would have missed. Instead, it hit (`cache_read_tokens = 14,770`).
2. **But `/btw` itself was a cold call**: The `/btw` fork paid `17,697` uncached input tokens with `cache_read_tokens = 0`.

Disassembling `setSideQuestionPromptSection` and `newSideQuestionBoundaryStep` in `agy` explains why: `/btw` creates a brand-new child conversation ID (`cascade_id`) and injects side-question instructions (~873 extra tokens). Because the child fork has a different conversation ID, its own call misses the cache and pays full price, which simultaneously refreshes the shared 14,770-token base prefix in the backend cache for the parent conversation.

| Keep-Alive Method | Ping Cost (eff.) | Final Turn Cost (eff.) | Total for Cycle |
| :--- | :---: | :---: | :---: |
| **Normal warm ping** | ~3,640 | ~3,630 | **~7,270** |
| **No keep-alive (accept 1 miss)** | 0 | ~16,900 | **~16,900** |
| **`/btw` side question** | 17,697 | 3,628 | **21,325** |

So `/btw` works, but it is the most expensive option: it costs **~4.9x a normal warm ping** and **~26% more than just accepting a cache miss**.

---

# 7. Unlocking Google's Built-In Engine & Live Verification

To get both the low token cost of a normal warm ping and the zero trajectory pollution of `/btw`, I looked back at the dormant internal engine from Section 2.2.

Google's engineers already wrote that exact logic inside `modelapiccpa/keepalive.go` (`trimLastBlock` + `max_output_tokens = 1`, scheduled at `4m 30s` with a `4m 45s` staleness cutoff). It was just gated off by two checks in the `agy` binary:
1. **Gate A (`keepAlivePings` at `0x102bd3000`)**: Returned `0` unless the server-side Mendel flag `enable-prompt-cache-keep-alive` was enabled.
2. **Gate B (`beforeRequest` at `0x102bd32dc`)**: Checked the provider enum (`0x5` / `0xa`) and skipped Gemini models.

To test whether Google's internal engine works end-to-end on Gemini once unlocked, I built a verification and binary-patching script:
- At **Gate A**, I patched the prologue to `mov x0, #3; ret` (giving a default budget of **3 idle pings**, covering ~13.5 minutes of refreshes / ~18 minutes of total idle time before stopping).
- At **Gate B**, I patched the provider branch so Gemini models pass straight into the keep-alive state machine.

## 7.1 Live 6-Minute End-to-End Test (`gemini-3.8-flash-low`)

I ran a live 6-minute trial on the patched `agy` binary using `gemini-3.8-flash-low` (`--effort low`, prompt `"Reply with only OK."`), sending only **two** user messages **6 minutes 6 seconds apart**, past the 4m 45s-5m 00s eviction cliff.

First, I ran a fast 20-second smoke test on a temporary binary copy with the internal timer patched to `15s` (`--interval-sec 15 --staleness-sec 45`), which confirmed that the background `streamGenerateContent` ping fired automatically at `+17s` and Turn 2 hit the cache (`11,835` cached tokens). Then I ran the full **6-minute test** using Google's stock `270s` (`4m 30s`) timer:

| Event | Clock Time | Elapsed Gap | `input_tokens` | `cache_read_tokens` | `output_tokens` | Transcript Steps Added |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **User Turn 1** | `02:32:10` | `0:00` | `13,884` | `0` | `1` | `step 0` (`USER`), `step 1` (`AGENT`) |
| **Internal Keep-Alive** | `02:36:42` | **`+4 min 32 s`** (`272s`) | *(silent background ping)* | *(refreshes TTL)* | `1` | **`0 steps` (Discarded)** |
| **User Turn 2** | `02:38:16` | **`+6 min 06 s`** (`366s`) | **`2,121`** | **`11,835`** | `1` | `step 2` (`USER`), `step 3` (`AGENT`) |

Checking the CLI's HTTP trace log (`http_helpers.go`) and the conversation's `transcript.jsonl` (`b476755e-baa4-451f-913a-e70cfb0bc892`) confirmed all three signals simultaneously:
1. **Exact `270s` timer execution**: Even though only 2 user turns were sent, the HTTP log recorded **3** `streamGenerateContent` requests: Turn 1 at `02:32:10`, the automatic background keep-alive at `02:36:42` (`+4m 32s`), and Turn 2 at `02:38:16` (`+6m 06s`).
2. **Full cache hit past the 5-minute cliff**: At `+6 min 06 s`, Turn 2 read **11,835 tokens from the cache** and billed only **2,121 uncached tokens**.
3. **Zero trajectory pollution**: Inspecting `transcript.jsonl` showed **only 4 steps total** (`step_index 0..3`, two user inputs and two `"OK"` responses). The `+4m 32s` background ping refreshed the backend cache and vanished without adding a single character to the conversation history.

---

# 8. Wrapping Up & Practical Takeaways

Here is what I learned from putting Antigravity under the microscope:

1. **The 5-minute cache cliff is real**: Both empirical binary search (`4m 45s - 5m 00s`) and Google's own hardcoded `285s` (`4m 45s`) cutoff in `keepalive.go` confirm that a 5-minute pause wipes your prompt cache.
2. **A 4m 30s keep-alive ping works, if you cap it**: Refreshing the cache before expiry saves 30% to 88% of effective tokens across a pause, provided you stop after `N_max` idle pings (3 pings by default, ~13.5 minutes of pings / ~18 minutes of total idle coverage) so you don't burn quota when away from your desk.
3. **Google's built-in `keepalive.go` engine works on Gemini**: Once its Mendel flag and provider-enum gates are unlocked, Antigravity's native background ping (`trimLastBlock` + `max_output_tokens = 1` at `4m 30s`) keeps Gemini caches warm across 6+ minute pauses with **zero trajectory growth**. *(Note: Since this relies on an internal, flag-gated subsystem inside the current binaries, Google could modify or remove `keepalive.go` in future releases).*
4. **Trim your baseline prefix**: Even a 5-word prompt starts with **13,800 to 16,800+ tokens** of system instructions, MCP schemas, and skills. Disabling unused MCP servers and plugins shrinks every cold turn and cache miss across the board.

Caveats worth keeping in mind: my effective token math uses Gemini's published 90% implicit-cache discount [2] (internal subscription quota weights may vary slightly), and implicit caching is best-effort, so backend TTLs can shift under load.

Till next time. Peace!

---

**References:**
1. Google. *Antigravity Documentation - Plans & Quotas*. [antigravity.google/docs/plans](https://antigravity.google/docs/plans)
2. Google AI for Developers. *Context Caching in the Gemini API*. [aistudio.google.com/docs/caching](https://aistudio.google.com/docs/caching)
3. Google Cloud. *Context Cache Overview - Gemini Enterprise Agent Platform*. [docs.cloud.google.com/gemini-enterprise-agent-platform/models/context-cache/context-cache-overview](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/context-cache/context-cache-overview)
4. GitHub Issue Tracker. *Unexpected weekly quota depletion on large models in Antigravity CLI*. `google-antigravity/antigravity-cli#1072` & `#1291`.
5. Google AI Developers Forum. *Implicit caching TTL and quota consumption telemetry*. Threads `196351` & `198430`.
