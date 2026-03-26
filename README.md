# Agent OS: OpenClaw Two-Agent Flow with Locally Hosted Models

Two-agent compliance pipeline benchmarked at 1, 5, and 10 concurrent sessions
on a single NVIDIA T4 GPU (16GB). One agent generates lending responses, one
validates against hardcoded compliance rules. Both backed by Qwen2.5-7B
running locally via Ollama and vLLM — no external API calls.

## Quick Results

| Backend | c=1 (p50) | c=5 (p50) | c=10 (p50) | c=10 Slowdown |
|---------|-----------|-----------|------------|---------------|
| Ollama (GGUF Q4_K_M) | **13.7s** | 50.6s | 92.3s | 6.7× |
| vLLM (AWQ 4-bit) | 15.3s | **17.1s** | **22.1s** | **1.4×** |

**Crossover point:** Between c=1 and c=2. Ollama wins single-request by 1.6s.
vLLM wins by 33.5s at c=5 and 70.2s at c=10.

96 total data points · 48 per backend · 3 measured runs + 1 warmup per configuration

---

## The Two-Agent Flow

```
Borrower Query
      │
      ▼
┌─────────────────┐
│  Agent 1        │  Borrower Response Agent
│  (LLM call)     │  Generates lending advice
└────────┬────────┘
         │ proposed_response
         ▼
┌─────────────────┐
│  Agent 2        │  Compliance Validator Agent
│  (LLM call)     │  Checks 3 hardcoded rules
└────────┬────────┘
         │ {rule_1_pass, rule_2_pass, rule_3_pass, all_pass, reasoning}
         ▼
    Return to caller
```

### The Three Compliance Rules

1. **No specific rate promises** — Must not guarantee a specific interest rate
2. **No approval guarantees** — Must not promise loan approval
3. **Advisor referral required** — Must recommend consulting a licensed financial advisor

### Test Queries

10 borrower scenarios covering first-time homebuyers, refinancing, self-employed
borrowers, low credit scores, HELOCs, student loan consolidation, investment
properties, and auto loans. Each designed to tempt the model into violating
at least one compliance rule.

---

## Environment

|                  | Ollama               | vLLM                           |
|------------------|----------------------|--------------------------------|
| **Version**      | 0.18.3               | 0.6.6.post1                    |
| **Backend**      | llama.cpp (CUDA)     | PyTorch + PagedAttention       |
| **Quantization** | GGUF Q4_K_M          | AWQ 4-bit                      |
| **Model**        | qwen2.5:7b           | Qwen/Qwen2.5-7B-Instruct-AWQ  |
| **GPU**          | NVIDIA T4 16GB (Colab) | NVIDIA T4 16GB (Colab)       |
| **Concurrency**  | Sequential queue     | Continuous batching             |

OpenClaw v2026.3.24 configured with two agents pointing at local Ollama
provider. Gateway verified running on ws://127.0.0.1:18789.

---

## Detailed Results

### Full Latency Table (p50 / p95)

| Backend | Concurrency | Agent 1 (p50) | Agent 2 (p50) | Total (p50) | Total (p95) |
|---------|-------------|---------------|---------------|-------------|-------------|
| Ollama  | 1           | 10.3s         | 3.4s          | 13.7s       | 13.9s       |
| Ollama  | 5           | 25.7s         | 24.9s         | 50.6s       | 56.9s       |
| Ollama  | 10          | 42.9s         | 50.7s         | 92.3s       | 105.2s      |
| vLLM    | 1           | 12.0s         | 3.3s          | 15.3s       | 18.6s       |
| vLLM    | 5           | 12.2s         | 4.8s          | 17.1s       | 21.0s       |
| vLLM    | 10          | 15.6s         | 6.5s          | 22.1s       | 25.7s       |

### Concurrency Scaling

| Concurrency | Ollama Slowdown | vLLM Slowdown | Winner          |
|-------------|-----------------|---------------|-----------------|
| 1           | 1.0× (baseline) | 1.0× (baseline) | Ollama by 1.6s |
| 5           | 3.7×            | 1.1×          | **vLLM by 33.5s** |
| 10          | 6.7×            | 1.4×          | **vLLM by 70.2s** |

### Validator Agent Reliability

| Backend | Valid JSON | All Rules Pass | Rule 1 Fails | Rule 2 Fails | Rule 3 Fails |
|---------|-----------|----------------|-------------|-------------|-------------|
| Ollama  | 48/48 (100%) | 25/48 (52%)  | 23          | 0           | 0           |
| vLLM    | 48/48 (100%) | 33/48 (69%)  | 11          | 2           | 4           |

**Finding:** Qwen2.5-7B produces valid JSON 100% of the time — strong structured
output compliance. However, the borrower agent violates Rule 1 (specific rate
promises) in nearly half of responses. The model has a tendency to generate
specific numbers when asked about rates, even when instructed not to.

This is a fundamental limitation of 7B models for compliance-critical agent
pipelines: instruction following degrades when the task conflicts with the
model's learned patterns of being "helpful" with specific information.

---

## Analysis

### 1. The Concurrency Collapse Mechanism

Each two-agent turn requires **two serial LLM calls**. At concurrency N:

**Ollama (sequential queue):**
- 10 concurrent sessions = 20 LLM calls
- Each call waits behind all preceding calls
- Session 10's validator (call #20) waits for ~19 prior calls to complete
- Result: near-linear latency scaling with concurrency

```
c=1:   [A1][A2]                                      → 13.7s
c=5:   [A1][A2][A1][A2][A1][A2][A1][A2][A1][A2]     → 50.6s
c=10:  [A1][A2]×10 in sequence                        → 92.3s
```

**vLLM (continuous batching):**
- All Agent 1 calls batch together on the GPU
- All Agent 2 calls batch together after their respective Agent 1 completes
- PagedAttention shares GPU memory across concurrent KV caches
- Result: sublinear scaling — 10× concurrency costs only 1.4× latency

```
c=1:   [A1][A2]                                      → 15.3s
c=5:   [A1,A1,A1,A1,A1][A2,A2,A2,A2,A2]            → 17.1s
c=10:  [A1×10 batched][A2×10 batched]                → 22.1s
```

### 2. Why Ollama Wins at c=1

At single-request, Ollama's llama.cpp backend has two advantages on T4:

1. **Kernel optimization**: Hand-tuned CUDA kernels for Turing (SM 7.5) architecture.
   vLLM runs with `--enforce-eager` on T4 because CUDA graph capture isn't supported,
   forgoing kernel fusion and launch overhead optimization.

2. **Quantization efficiency**: GGUF Q4_K_M uses per-block mixed precision — keeping
   sensitive layers at higher precision while aggressively quantizing FFN layers.
   This produces faster decode on memory-bandwidth-bound T4.

### 3. Why This Matters for Multi-Agent Production Systems

The KrimOS voice pipeline runs multiple Karta agents coordinating concurrently
on the same borrower interaction. This is exactly the c=5 to c=10 scenario:

- A borrower calls in → routing agent, response agent, compliance agent,
  escalation agent may all need inference within the same turn
- Sequential queuing means each additional agent **multiplies** the wait time
- At c=5, a borrower waits **51 seconds** on Ollama vs **17 seconds** on vLLM
- At c=10, it's **92 seconds** — outside any acceptable SLA

Continuous batching isn't a nice-to-have. It's a **hard requirement** for
multi-agent serving.

### 4. Local Model Limitations for Production Agent Reliability

The validator analysis reveals a critical limitation:

- **JSON compliance is excellent** (100%) — Qwen2.5-7B reliably produces structured output
- **Rule compliance is unreliable** (52-69%) — The borrower agent frequently violates
  Rule 1 by promising specific rates, despite explicit instructions not to

This means a 7B validation agent **cannot be trusted as a hard gate** in production.
Options for production:
- Use the validator as a soft signal (flag for human review, not block)
- Use constrained decoding / JSON schema enforcement for the validator
- Use a larger model (70B+) for the validation layer specifically
- Implement rule-based regex checks as a fallback alongside LLM validation

---

## OpenClaw Integration

### Configuration

Two agents registered in `~/.openclaw/openclaw.json`:

```json
{
  "gateway": { "mode": "local", "port": 18789 },
  "models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://127.0.0.1:11434",
        "api": "ollama",
        "models": [{ "id": "qwen2.5:7b" }]
      }
    }
  },
  "agents": {
    "defaults": { "model": { "primary": "ollama/qwen2.5:7b" } },
    "list": [
      { "id": "borrower", "model": "ollama/qwen2.5:7b" },
      { "id": "validator", "model": "ollama/qwen2.5:7b" }
    ]
  }
}
```

### Agent Behavioral Definitions

Each agent defined via Markdown workspace files:
- `AGENTS.md` — Operating instructions and compliance rules
- `SOUL.md` — Persona and boundaries
- `IDENTITY.md` — Name and role
- `TOOLS.md` — Tool usage restrictions

### Verified Working

```
$ openclaw agents list
- borrower (default) — Model: ollama/qwen2.5:7b
- validator          — Model: ollama/qwen2.5:7b

$ openclaw agent --agent borrower --message "What mortgage rates for 720 credit?"
→ Response generated via local Ollama (46.0s including OpenClaw tool overhead)

$ openclaw agent --agent validator --message "[validation input]"
→ Compliance validation returned (37.7s)

Full two-agent turn via OpenClaw CLI: 83.7s
```

### Benchmark Approach

The concurrency benchmark uses the same system prompts and two-agent sequential
logic but hits the model endpoints directly. This isolates the model backend
concurrency behavior from OpenClaw's Gateway and tool orchestration overhead,
producing cleaner measurements for the scaling analysis.

See [WHAT_BROKE.md](WHAT_BROKE.md) for details on OpenClaw setup challenges
and the tool-disabling issue.

---

## Hardware Note

Both **vLLM ≥0.18** and **SGLang ≥0.5** have dropped NVIDIA T4 (SM 7.5)
kernel support. This benchmark uses vLLM 0.6.6.post1 (last T4-compatible
version) with transformers==4.45.2. On modern GPUs (A100/L4/H100), expect:

- 2-3× faster per-call latency via FlashAttention-2 + CUDA graphs
- 50-70 tok/s decode throughput
- Even wider gap between sequential and batched backends at concurrency

---

## Repository Contents

```
├── .openclaw/
│   ├── openclaw.json              # Two-agent config
│   └── agents/
│       ├── borrower/agent/        # AGENTS.md, SOUL.md, IDENTITY.md, TOOLS.md
│       └── validator/agent/       # AGENTS.md, SOUL.md, IDENTITY.md, TOOLS.md
├── notebook.ipynb                 # Complete runnable Colab notebook
├── benchmark_results.png          # 6-panel visualization
├── raw_results.json               # All 96 data points
├── README.md
├── WHAT_BROKE.md
└── WHAT_ID_TRY_NEXT.md
```




