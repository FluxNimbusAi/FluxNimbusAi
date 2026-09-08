# Qwen Model Capabilities — Research Breakdown

**Prepared by:** Flux (VALOR CORP SWARM COMMANDER)
**Date:** 2026-09-08
**Scope:** Qwen3.8-27B (local/Ollama) vs Qwen3.7-Flash (QwenCloud API)

---

## 1. Overview

| Property | Qwen3.8-27B | Qwen3.7-Flash |
|----------|-------------|---------------|
| **Architecture** | Dense transformer, 27B params | MoE (Mixture-of-Experts), 43B total / 3.6B active |
| **Deployment** | Local (Ollama/RTX 4090) | Cloud API (QwenCloud dashscope-intl) |
| **Cost per token** | $0 marginal (electricity only) | Pay-per-token |
| **Latency** | ~2-5s per response (local GPU) | ~1-3s (cloud, network RTT) |
| **Context window** | 131,072 tokens (128K) | 131,072 tokens (128K) |
| **Max output** | 16,384 tokens | ~8,192 tokens |
| **Modalities** | Text | Text + Vision (multimodal) |
| **Speed** | Slower per token, unlimited tokens | Faster per token, token-limited |
| **Offline** | ✅ Yes | ❌ No (API dependency) |

---

## 2. Qwen3.8-27B (Local)

### Strengths
- **Zero marginal cost** — heavy analysis is $0 per token. Run 100-token loops with no financial pressure.
- **Unlimited token generation** — can produce full audit reports, multi-file code, long reasoning chains without hitting a wall.
- **Privacy** — code and data never leave the machine. No API key exposure, no data retention concerns.
- **Deep reasoning** — 27B dense parameters give strong logical depth for cross-file analysis, finding validation, and architectural reasoning.
- **No rate limits** — can chain 50 tool calls in a turn without throttling.

### Limitations
- **No vision** — cannot process images, diagrams, or PDFs with visual content.
- **Single model, single lane** — no fallback if the local GPU is busy or the model OOMs.
- **Lower ceiling on creative/novel tasks** — 27B dense is strong but not frontier-class for the hardest reasoning.
- **Context management is manual** — must explicitly control what's in context; no server-side memory.

### Sweet spot config (locked)
```
num_ctx: 65536
maxTokens: 16384
reasoning: false
```

---

## 3. Qwen3.7-Flash (Cloud API)

### Strengths
- **Multimodal** — text + vision. Can process screenshots, UI mockups, architecture diagrams.
- **Fast per-token** — optimized for low-latency responses. Good for quick drafts and preprocessing.
- **Token budget flexibility** — can burst on a single large request (e.g., full contract analysis in one call).
- **Always-on** — no local hardware dependency. Works even if the RTX is down.
- **Lower latency for short tasks** — 2-3 second responses for simple queries.

### Limitations
- **Pay-per-token** — every response costs money. Heavy loops are expensive.
- **Token cap** — output is bounded; very long reports require multiple calls.
- **No offline** — API dependency is a single point of failure.
- **Data leaves the machine** — code sent to QwenCloud API.

### Best use cases
- Quick drafts, summaries, preprocessing
- Vision tasks (screenshot analysis, diagram interpretation)
- Parallel crawl/aggregation (Xeon's role)
- Backup when local GPU is unavailable

---

## 4. Routing Strategy (Locked)

| Task Type | Route to | Why |
|-----------|----------|-----|
| Deep architectural analysis | **Qwen3.8 (local)** | $0 cost, unlimited tokens, privacy |
| Cross-file audit validation | **Qwen3.8 (local)** | Needs full context, deep reasoning |
| Report writing (long) | **Qwen3.8 (local)** | 16K+ token output, no cost pressure |
| Quick drafts / summaries | **Qwen3.7-Flash (cloud)** | Fast, good enough |
| Vision / image analysis | **Qwen3.7-Flash (cloud)** | Only model with vision |
| Crawl / aggregation | **Qwen3.7-Flash (cloud)** | Xeon's lane, parallel |
| Code generation (short) | **Qwen3.7-Flash (cloud)** | Fast, cheap for small outputs |
| Code generation (long) | **Qwen3.8 (local)** | Token budget, depth |
| Bug finding / vulnerability | **Qwen3.8 (local)** | Deep reasoning, cross-reference |

### 2x→3x Rule (standing)
- Heavy mechanical task → **Xeon first** (Qwen3.7-Flash cloud)
- Two failed local attempts → **Xeon before attempt #3**
- Judgment, strategy, money, final decisions → **Flux (Qwen3.8 local) always**

---

## 5. Operational Notes

### Local (Qwen3.8)
- **Restart:** `ollama_restart.sh` only
- **Contention:** Single GPU — don't run multiple heavy tasks concurrently
- **OOM risk:** Context > 64K + long outputs can OOM. Keep `num_ctx` at 65536 max.
- **Monitoring:** `ollama ps` to check active loads

### Cloud (Qwen3.7-Flash)
- **Auth:** `QWEN_API_KEY` env var (managed via secrets)
- **Rate limits:** QwenCloud standard tier — burst OK, sustained heavy use may throttle
- **Cost control:** Set per-task token caps; avoid 100-token loops
- **Fallback:** If API down, fall back to local Qwen3.8 (slower but functional)

---

## 6. Decision Framework

**When to use Qwen3.8 (local):**
- The task requires deep reasoning or cross-reference
- The output will be long (report, full code, multi-file)
- Cost sensitivity matters (heavy iteration)
- Privacy matters (sensitive code, financial logic)

**When to use Qwen3.7-Flash (cloud):**
- The task is quick and bounded (draft, summary, classification)
- Vision is required
- Local GPU is unavailable
- Parallelism is needed (multiple simultaneous crawls)

**When to use both:**
- Cloud for the first pass (fast, broad) → local for deep dive (precise, deep)
- Cloud for preprocessing (extract, classify) → local for analysis (validate, synthesize)

---

## 7. Summary

**Qwen3.8-27B** is the deep-thinking workhorse — free, private, powerful for the hard stuff. It's the brain.

**Qwen3.7-Flash** is the fast, flexible sidekick — cheap, multimodal, good for the quick stuff. It's the hands.

**The routing is locked:** heavy = local, quick = cloud, judgment = always local.
