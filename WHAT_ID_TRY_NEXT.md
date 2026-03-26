# What I'd Try Next

## 1. Concurrency Benchmark Through OpenClaw Gateway

The current benchmark hits model endpoints directly to isolate backend
concurrency behavior. The next step is benchmarking through OpenClaw's
WebSocket Gateway (ws://127.0.0.1:18789) to measure the full-stack
overhead: Gateway routing + agent context injection + model inference.

This would answer: does OpenClaw's Gateway add fixed overhead per request,
or does it introduce its own concurrency bottleneck?

## 2. agentToAgent Tool for Native Handoff

OpenClaw has an `agentToAgent` tool (disabled by default, requires
explicit allowlist). Enabling this would let Agent 1 (borrower) directly
invoke Agent 2 (validator) within a single session, rather than
orchestrating the handoff externally.

This changes the concurrency profile: instead of 2 separate LLM calls
per session, the Gateway handles the chaining internally. Worth
measuring whether this reduces handoff overhead.

## 3. vLLM on Modern Hardware (A100/H100)

vLLM 0.6.6 on T4 runs with `--enforce-eager` (no CUDA graphs) and
without FlashAttention-2. On A100:
- FlashAttention-2 reduces prefill latency 2-3×
- CUDA graph capture eliminates kernel launch overhead
- Estimated: ~5-8s per two-agent turn vs current 15s
- Concurrency scaling would be even better with more VRAM for KV cache

## 4. Hybrid Backend Strategy

Based on the crossover finding:
- **c≤2:** Ollama is faster (lower single-request latency)
- **c≥3:** vLLM is faster (continuous batching absorbs concurrency)

A production system could route based on current load:
- Low traffic → Ollama backend for minimum latency
- High traffic → vLLM backend for throughput
- This requires a load-aware proxy (e.g., Envoy with custom metrics)

## 5. Validator Reliability Improvements

The 7B model passes compliance rules only 52-69% of the time. For
production:

- **Constrained decoding**: Force JSON schema compliance via
  outlines/guidance library with vLLM
- **Regex fallback**: Hard-code regex patterns for the three rules
  (e.g., detect specific rate patterns like "$X.XX%") as a
  deterministic fallback alongside LLM validation
- **Larger validator model**: Use a 70B model for validation only
  (accepts higher latency for compliance accuracy)
- **Fine-tuning**: Fine-tune the 7B model on compliance validation
  examples to improve rule-following

## 6. Speculative Decoding

vLLM supports speculative decoding with a smaller draft model. For the
validator agent (short, structured JSON output), a 0.5B draft model
could accelerate decoding 2-3× without quality loss, since the output
format is highly predictable.

## 7. Prefix Caching for Repeated System Prompts

Both agents use the same system prompt across all sessions. vLLM's
automatic prefix caching would share the KV cache for system prompt
tokens across concurrent requests, reducing per-request prefill cost.
This is free performance at c≥2 — just needs `--enable-prefix-caching`
flag (available on vLLM 0.6.6+ but not tested on T4).

