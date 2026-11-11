# Qwen3.5-0.8B WCAG Gauntlet Tool-Use LoRA (+ Q4_K_M GGUF)

LoRA adapter that teaches **Qwen3.5-0.8B** the agentic tool-routing skill from [wcag-skill](https://github.com/turbolego/wcag-skill):
while running the [AI-WCAG-Gauntlet](https://github.com/turbolego/AI-WCAG-Gauntlet) benchmark
loop, pick the right validator (`w3c` / `axe` / `pa11y` / `qualweb` / `fix` / `check`) and run
a self-check on your own code **before** the benchmark's official iteration check.

Published model (LoRA adapter + merged **Q4_K_M GGUF**): https://huggingface.co/Turbolego/qwen3.5-0.8b-wcag-gauntlet-lora

---

## Download the GGUF (Ollama / llama.cpp ready)

| Source | File | Size |
|---|---|---|
| **Hugging Face** (direct, LFS) | [`qwen3.5-0.8b-wcag-gauntlet-lora-q4_k_m.gguf`](https://huggingface.co/Turbolego/qwen3.5-0.8b-wcag-gauntlet-lora/resolve/main/qwen3.5-0.8b-wcag-gauntlet-lora-q4_k_m.gguf) | 505 MB |
| **GitHub release asset** | [v1.0-q4k-m-gguf](https://github.com/turbolego/qwen3.5-0.8b-wcag-gauntlet-lora/releases/download/v1.0-q4k-m-gguf/qwen3.5-0.8b-wcag-gauntlet-lora-q4_k_m.gguf) | 505 MB |

```bash
# Hugging Face
hf download Turbolego/qwen3.5-0.8b-wcag-gauntlet-lora \
  qwen3.5-0.8b-wcag-gauntlet-lora-q4_k_m.gguf --local-dir ./

# or GitHub release
curl -L -o qwen3.5-0.8b-wcag-gauntlet-lora-q4_k_m.gguf \
  https://github.com/turbolego/qwen3.5-0.8b-wcag-gauntlet-lora/releases/download/v1.0-q4k-m-gguf/qwen3.5-0.8b-wcag-gauntlet-lora-q4_k_m.gguf

# Ollama
cat > Modelfile <<'EOF'
FROM ./qwen3.5-0.8b-wcag-gauntlet-lora-q4_k_m.gguf
TEMPLATE """{{- if .System }}{{ .System }}{{ end }}
{{- range .Messages }}{{ .Role }}: {{ .Content }}
{{- end }}
assistant: """
PARAMETER stop "user:"
PARAMETER stop "assistant:"
EOF
ollama create qwen-gauntlet -f Modelfile
ollama run qwen-gauntlet "State: w3c_errors=2, axe_violations=5. Which validator next? Format: <word>\\n<yes|no>"
# expected: w3c \n yes
```

---

## Quick facts

| | |
|---|---|
| **Base model** | [Qwen/Qwen3.5-0.8B](https://huggingface.co/Qwen/Qwen3.5-0.8B) (bf16) |
| **Method** | LoRA, rank 32, α=64, dropout 0.05 |
| **Target layers** | `q_proj`, `k_proj`, `v_proj`, `o_proj`, `gate_proj`, `up_proj`, `down_proj` |
| **Trainable params** | 12.8M (1.67% of 765M) |
| **Optimizer** | AdamW, lr 2e-4, cosine schedule, 5% warmup |
| **Hardware** | Single NVIDIA T4 (Kaggle kernel `hummern/lora-tooluse-skill-v7`) |
| **Training data** | 300 fit / 40 val / 155 test rows (495 total) |
| **License** | Apache 2.0 |
| **Collaborators** | Turbolego (owner), hummern (author), kveita |

---

## Performance

### Test set — 155 held-out rows

| Metric | Base Qwen3.5-0.8B | After LoRA |
|---|---|---|
| **`next_tool`** (6-way validator routing) | 0% (emits prose, no tool-use) | **100%** |
| **`run_precheck`** (self-check yes/no) | 6.5% | **100%** |

### Validation set — 40 rows

100% / 100% after 2 epochs.

### Base model context

On plain WCAG question-answering (not this task), base Qwen3.5-0.8B is already at 95% test accuracy — this embedding task is about *agentic tool use* (running axe-core / pa11y / QualWeb / W3C validation on your own code during the gauntlet loop), not about knowing WCAG facts.

---

## Files in this repo

| File | Purpose |
|---|---|
| `lora_tooluse_skill.py` | Training script (LoRA fine-tune on Qwen3.5-0.8B, T4-ready) |
| `tooluse_fit.jsonl` | Training data — 300 rows |
| `tooluse_val.jsonl` | Validation data — 40 rows |
| `tooluse_test.jsonl` | Held-out test data — 155 rows |
| `tooluse_questions.json` | Vegaml question definitions used in training |
| `LICENSE` | Apache 2.0 |

> The merged **Q4_K_M GGUF** (505 MB) is too large for git push — download it from
> [Hugging Face](https://huggingface.co/Turbolego/qwen3.5-0.8b-wcag-gauntlet-lora/resolve/main/qwen3.5-0.8b-wcag-gauntlet-lora-q4_k_m.gguf)
> or the [GitHub release `v1.0-q4k-m-gguf`](https://github.com/turbolego/qwen3.5-0.8b-wcag-gauntlet-lora/releases/tag/v1.0-q4k-m-gguf).

---

## How it was trained

### Dataset construction

1. 15 real gauntlet page states were captured, each run through the actual
   `test-suite.sh` check suite (W3C validator + axe-core + pa11y + QualWeb + HTML5 tag
   coverage, Chrome for Testing on a local host).
2. Each state was expanded into multiple variants: different tag-coverage levels
   (early/mid/late/final iteration), post-fix variants (one flag cleared at a time),
   and edge-case combinations.
3. Gold labels were derived from the validator flags per the wcag-skill triage rules:
   **w3c first** (structural errors cascade), then landmarks/structure, then perceivable,
   then operable, then robust (axe ACT rules), then code/tag work, then `check` when
   everything passes.

### Training

```
pip install torch transformers peft trl datasets
python lora_tooluse_skill.py   # trains ~2 epochs on Kaggle T4
```

The training script loads Qwen3.5-0.8B in bf16, wraps it with a rank-32 PEFT LoRA,
trains on the 300 fit rows with per-epoch validation on the 40 val rows, and saves
the adapter to `./out_tooluse_lora/`.

---

## Closed-form embedding (MEMIT) — did NOT transfer

A rank-1 MEMIT column edit on layer-23 `down_proj` (the same method that works for
factual WCAG Q&A skill embedding) scored **6.5%** on this task — below the 16.7%
random floor; it actively hurt.

Tool routing is procedural behavior, not a factual logit gap. A rank-1 broadcast into a
single layer's `down_proj` cannot move the next-token distribution the way tool routing
requires. Only gradient-based LoRA transferred the skill.

---

## Usage

### Python (transformers + peft)

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import PeftModel

tok = AutoTokenizer.from_pretrained("Qwen/Qwen3.5-0.8B")
base = AutoModelForCausalLM.from_pretrained("Qwen/Qwen3.5-0.8B", device_map="auto")
model = PeftModel.from_pretrained(base, "Turbolego/qwen3.5-0.8b-wcag-gauntlet-lora")
```

### Ollama + VS Code

This repo ships a PEFT LoRA adapter, not a GGUF — Ollama needs a merged model:

1. Download adapter + base:
   ```bash
   hf download Turbolego/qwen3.5-0.8b-wcag-gauntlet-lora --local-dir ./gauntlet-lora
   hf download Qwen/Qwen3.5-0.8B --local-dir ./qwen-base
   ```
2. Merge:
   ```bash
   pip install torch transformers peft llama-cpp-python
   python - <<'PY'
   from transformers import AutoModelForCausalLM, AutoTokenizer
   from peft import PeftModel
   tok = AutoTokenizer.from_pretrained("./qwen-base")
   base = AutoModelForCausalLM.from_pretrained("./qwen-base", torch_dtype="auto")
   model = PeftModel.from_pretrained(base, "./gauntlet-lora")
   merged = model.merge_and_unload()
   merged.save_pretrained("./qwen-gauntlet-merged")
   tok.save_pretrained("./qwen-gauntlet-merged")
   PY
   git clone https://github.com/ggerganov/llama.cpp && cd llama.cpp && make -j
   python convert_hf_to_gguf.py ../qwen-gauntlet-merged --outfile ../qwen-gauntlet-q4.gguf --outtype q4_k_m
   ```
3. Create an Ollama model (`Modelfile`):
   ```
   FROM ./qwen-gauntlet-q4.gguf
   TEMPLATE """{{- if .System }}{{ .System }}{{ end }}
   {{- range .Messages }}{{ .Role }}: {{ .Content }}
   {{- end }}
   assistant: """
   PARAMETER stop "user:"
   PARAMETER stop "assistant:"
   ```
   ```bash
   ollama create qwen-gauntlet -f Modelfile
   ollama run qwen-gauntlet "Which validator comes next if only axe violations remain? Answer:"
   ```
4. In VS Code, install **Continue** (or **Cline**) and add Ollama as a provider:
   ```json
   {
     "models": [{
       "title": "qwen-gauntlet (Ollama)",
       "provider": "ollama",
       "model": "qwen-gauntlet"
     }]
   }
   ```

### Quick sanity check

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "qwen-gauntlet",
  "messages": [{"role": "user", "content": "State: w3c_errors=2, axe_violations=5, pa11y_issues=0, qualweb_issues=1, missing_tags=[nav]. Question 1: which validator or action comes next (w3c/axe/pa11y/qualweb/fix/check)? Question 2: run a pre-check on your own code? Format: <word>\\n<yes|no>"}],
  "stream": false
}'
# expected answer: w3c \n yes
```

---

## Test on the AI-WCAG-Gauntlet

1. Clone the benchmark harness: https://github.com/turbolego/AI-WCAG-Gauntlet
2. Create a fresh submission directory and write original HTML/CSS/JS in it.
3. Point the gauntlet's agent at your VS Code + Ollama `qwen-gauntlet` model
   (or run the harness against the Ollama API: `curl localhost:11434/api/chat`).
4. Let the loop run; the model should pick `w3c → axe → pa11y → qualweb → fix` in
   triage order and answer `yes` to running a pre-check on its own edits.
5. Run the official test suite and keep its output separate from production
   accessibility reports (see the benchmark README's integrity rule).

---

## Sources

- Benchmark harness: https://github.com/turbolego/AI-WCAG-Gauntlet
- Skill definition: https://github.com/turbolego/wcag-skill
- Skill card: https://clawhub.ai/turbolego/skills/wcag-skill
- Training repo + data + results: https://github.com/turbolego/vegaml-wcag
  (`notebooks/lora_tooluse_skill.py`, `notebooks/tooluse_*.jsonl`,
  `LORA_TOOLUSE_RESULTS.md`)
- Base model: https://huggingface.co/Qwen/Qwen3.5-0.8B
- Published model: https://huggingface.co/Turbolego/qwen3.5-0.8b-wcag-gauntlet-lora
