# What Broke and How I Fixed It

Chronological debugging journal from setting up OpenClaw + local model
serving on Google Colab (NVIDIA T4).

---

## 1. Ollama Install — Missing zstd

**Error:** `This version requires zstd for extraction`

Ollama's install script bundles compressed binaries requiring zstd, which
isn't pre-installed on Colab.

**Fix:** `apt-get install -y zstd` before running the install script.

**Time lost:** 5 minutes.

---

## 2. Ollama GPU Detection — 80-Second "Cold Start"

**Symptom:** First inference call took 80.9 seconds. Install warned
"Unable to detect NVIDIA/AMD GPU."

**What happened:** Ollama couldn't find `lspci`/`lshw` to auto-detect the GPU,
but CUDA drivers were present. The 80s was model weight loading from disk to
VRAM (first-time cold start), not CPU inference. Second call: 0.5s.

**Evidence:** `nvidia-smi` showed 4878MiB used, 88% utilization, and the
ollama process mapped to the GPU.

**Fix:** No fix needed — the warning is cosmetic. Added a warmup call
before benchmarking.

**Lesson:** Always verify GPU usage with `nvidia-smi`, don't trust install
script warnings. Always include warmup runs in benchmarks.

---

## 3. OpenClaw npm — Placeholder Package

**Error:** `npm install -g openclaw` installed a package with:
```
Version: 0.0.1
Description: "Empty placeholder package"
Bin field: NOT FOUND
```

**What happened:** The npm name "openclaw" was squatted by a placeholder.
The real OpenClaw requires `openclaw@latest` which resolves to the actual
v2026.3.24 package.

**Fix:** `npm install -g openclaw@latest`

**Time lost:** 15 minutes investigating.

---

## 4. Node.js 12 — Optional Chaining Syntax Error

**Error:**
```
SyntaxError: Unexpected token '.'
    return typeof parsed?.rootHelpText === "string"
```

**What happened:** Colab ships Node.js v12.22.9. OpenClaw uses optional
chaining (`?.`) which requires Node 14+. The actual requirement is Node 22+.

**Fix attempts:**
1. NodeSource `setup_18.x` — didn't update (apt pinning conflict)
2. `apt-get install nodejs` — still v12
3. **Working fix:** Direct binary download of Node.js v22.16.0:
   ```
   curl -fsSL -o /tmp/node22.tar.xz \
     https://nodejs.org/dist/v22.16.0/node-v22.16.0-linux-x64.tar.xz
   tar -xJf /tmp/node22.tar.xz -C /usr/local --strip-components=1
   ```

**Time lost:** 20 minutes across three attempts.

**Lesson:** Colab's apt repositories are locked. For specific Node versions,
always use binary downloads or nvm, not apt.

---

## 5. OpenClaw Config Schema — Iterative Validation

Three rounds of config fixes before `openclaw config validate` passed:

**Round 1:** `models.providers.ollama.models: expected array, received undefined`
→ Added `"models": ["qwen2.5:7b"]`

**Round 2:** `models.providers.ollama.models.0: expected object, received string`
→ Changed to `"models": [{"id": "qwen2.5:7b", "name": "...", "contextLength": 2048}]`

**Round 3:** `Unrecognized key: "contextLength"`
→ Removed contextLength, kept only `{"id": "qwen2.5:7b"}`

**Final working schema:**
```json
"models": [{ "id": "qwen2.5:7b" }]
```

**Also needed:** `"gateway": { "mode": "local", "port": 18789 }` — doctor
flagged that gateway.mode was required.

**Lesson:** OpenClaw has strict config validation. The docs showed a simpler
config format than what the actual validator accepts. Iterating against
`openclaw config validate` was faster than reading docs.

---

## 6. OpenClaw TOOLS.md — Web Search Not Disabled

**Symptom:** Borrower agent response included "Based on the information
gathered from web searches" and URLs, despite TOOLS.md containing
"Do not use any external tools."

**What happened:** `TOOLS.md` provides *tool usage notes* — guidance for
how to use tools, not permissions for which tools are enabled. Tool policy
is controlled at a different level (likely agent config or skill
registration).

**Impact:** OpenClaw agent turns took 46s + 38s = 84s due to web search
latency. Benchmark uses direct model endpoints to isolate inference
performance.

**What I'd do differently:** Use `openclaw skills` CLI to inspect and
disable specific skills, or set tool policy in the agent config.

---

## 7. vLLM + Transformers Version Conflict

**Error 1:** With default Colab transformers:
```
AttributeError: Qwen2Tokenizer has no attribute all_special_tokens_extended
```

**Error 2:** With transformers==4.44.2:
```
ModuleNotFoundError: No module named 'transformers.models.mllama'
```

**What happened:** vLLM 0.6.6.post1 requires transformers>=4.45.2 but
the newest versions changed the tokenizer API. A narrow compatibility
window exists.

**Fix:** `pip install transformers==4.45.2` — the exact minimum version
vLLM 0.6.6 specifies.

**Time lost:** 10 minutes.

**Lesson:** When using pinned framework versions (vLLM 0.6.6 for T4
compatibility), pin the entire dependency chain. Add
`transformers==4.45.2` to requirements.txt.

---

## Total Debugging Time

| Issue | Time |
|-------|------|
| zstd dependency | 5 min |
| GPU detection false alarm | 5 min |
| npm placeholder package | 15 min |
| Node.js version (3 attempts) | 20 min |
| Config schema (3 rounds) | 15 min |
| TOOLS.md investigation | 10 min |
| vLLM transformers pinning | 10 min |
| **Total** | **~80 minutes** |

Every issue above would hit any team deploying OpenClaw + local models
on cloud GPU instances. The Node.js version and config schema issues
are the kind of friction that makes production deployment harder than
the "just run it locally" pitch suggests.



