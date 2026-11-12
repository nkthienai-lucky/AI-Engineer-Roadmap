# Stage 2 Mini-GPT Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an independent PyTorch Mini-GPT project that trains on Shakespeare char-level text, generates text, and measures manual attention generation with and without KV cache.

**Architecture:** Create a standalone Git repository at `D:\Thien\Code\mini-gpt`, run primarily from WSL2 with CUDA when available and CPU fallback always supported. Keep data/tokenization, model, training/checkpointing, generation/benchmarking, and documentation as separate boundaries. The model is a configurable decoder-only pre-norm Transformer whose attention core explicitly computes Q/K/V, scaled scores, causal masking, softmax, and weighted values.

**Tech Stack:** Python 3.12+; PyTorch 2.11.0+cu128 in the roadmap WSL2 environment; pytest; standard library only beyond PyTorch for the baseline.

**Spec:** `AI-Engineer-Roadmap/docs/superpowers/specs/2026-11-09-mini-gpt-stage-2-design.md`

## Global Constraints

- The project is an independent Git repository at `D:\Thien\Code\mini-gpt`; Stage 1 `chatbot-cli` remains unchanged.
- The supported workflow is WSL2 Ubuntu with the existing CUDA environment and RTX 5060 when available; CPU fallback is required for tests and smoke runs.
- The baseline dataset is Shakespeare char-level with a contiguous 90/10 train/validation split.
- The educational attention core must explicitly compute Q/K/V projections, scaled dot products, causal masking, softmax, and weighted values.
- `torch.nn.functional.scaled_dot_product_attention`, `nn.Transformer`, `nn.Transformer*`, and `nn.MultiheadAttention` are forbidden in the core.
- The baseline uses learned positional embeddings; RoPE, BPE, Vietnamese data, LoRA/QLoRA, and chatbot integration are follow-up work.
- The model must use configurable compact defaults suitable for an 8 GB GPU and CPU smoke tests.
- Training uses next-token cross-entropy, AdamW, deterministic seeds where practical, periodic evaluation, and a checkpoint containing model config, vocabulary, and training metadata.
- Generation supports temperature and optional top-k; fixed settings and seed must be reproducible.
- CPU KV-cache benchmarking is sufficient for acceptance when CUDA is unavailable; CUDA results are preferred when available and must use synchronization for timing.
- The README must include setup, commands, architecture/shape explanation, loss evidence, sample output, benchmark conditions/results, limitations, and answers to the O(n²) and KV-cache questions.

## Review Focus

- **Short or malformed corpus:** an empty/one-character input must fail with a clear validation error instead of producing invalid vocab or batches; covered in Task 1 data tests.
- **Context boundary:** a sequence at `block_size` is accepted, while an overlong prompt is rejected or explicitly truncated by the generation contract; covered in Task 4 generation tests.
- **Causal leakage:** changing a future token must not change logits for earlier positions; covered in Task 2 model tests.
- **Cache correctness:** cached and uncached greedy generation must produce identical token IDs for the same prompt/checkpoint; covered in Task 4 tests.
- **Device timing and checkpoint portability:** CUDA timing synchronizes correctly, and a CUDA-created checkpoint can load on CPU; covered in Tasks 3–4 tests/verification.

---

### Task 1: Bootstrap Project and Dataset Utilities

**Files:**
- Create: `mini-gpt/.gitignore`
- Create: `mini-gpt/requirements.txt`
- Create: `mini-gpt/config.py`
- Create: `mini-gpt/data.py`
- Create: `mini-gpt/tests/test_data.py`
- Create: `mini-gpt/data/input.txt` or a documented data-fetch command, depending on corpus size/licensing

**Interfaces:**
- Produces `CharTokenizer`, `TextDataset`, and `get_batch` used by Tasks 3–4.
- `CharTokenizer.from_text(text: str) -> CharTokenizer`
- `CharTokenizer.encode(text: str) -> torch.Tensor` with dtype `torch.long`
- `CharTokenizer.decode(ids: torch.Tensor | list[int]) -> str`
- `CharTokenizer.vocab_size: int`, `stoi: dict[str, int]`, `itos: dict[int, str]`
- `load_text_dataset(path: Path, train_fraction: float = 0.9) -> TextDataset`
- `TextDataset.train` and `.val` are one-dimensional `torch.long` tensors; the split is contiguous and the first 90% belongs to train.
- `get_batch(data: torch.Tensor, block_size: int, batch_size: int, device: torch.device, generator: torch.Generator | None = None) -> tuple[torch.Tensor, torch.Tensor]`

- [ ] **Step 1: Write failing data tests**

  Add tests for:
  - round-trip encode/decode preserving every character;
  - deterministic vocabulary ordering;
  - contiguous 90/10 split on a known 10-character string;
  - batch shapes `(batch_size, block_size)` and shifted targets;
  - empty/one-character input raising `ValueError` with a useful message.

- [ ] **Step 2: Run the focused tests and verify they fail**

  Run from the future repo root:

  ```bash
  pytest tests/test_data.py -q
  ```

  Expected: collection or import failures because `data.py` and its interfaces do not exist yet.

- [ ] **Step 3: Create the standalone repo and implement the data boundary**

  Initialize `D:\Thien\Code\mini-gpt` as its own Git repository from WSL2. Implement deterministic character vocabulary construction (sorted characters), contiguous split, and random contiguous training windows. Reject a corpus too short for `block_size + 1` with `ValueError`; never silently leak samples across the train/validation boundary.

  Put model and training defaults in a `Config` dataclass in `config.py`, including `seed`, `block_size`, `n_layer`, `n_head`, `n_embd`, `dropout`, `batch_size`, `learning_rate`, `max_steps`, `eval_interval`, `eval_iters`, `warmup_steps`, `device`, `temperature`, and `top_k`. Keep exact compact defaults in one place; later tasks may override them in tests/CLI.

- [ ] **Step 4: Run the focused tests and verify they pass**

  ```bash
  pytest tests/test_data.py -q
  ```

  Expected: all data tests pass on CPU.

- [ ] **Step 5: Commit the bootstrap**

  ```bash
  git add .gitignore requirements.txt config.py data.py tests/test_data.py data/input.txt
  git commit -m "feat: bootstrap mini-gpt dataset pipeline"
  ```

---

### Task 2: Implement Manual Transformer Model

**Files:**
- Create: `mini-gpt/model.py`
- Create: `mini-gpt/tests/test_model.py`

**Interfaces:**
- Consumes `Config` from Task 1.
- `KVCache = list[tuple[torch.Tensor, torch.Tensor]]`
- `CausalSelfAttention.forward(x: torch.Tensor, kv_cache: tuple[torch.Tensor, torch.Tensor] | None = None, use_cache: bool = False) -> tuple[torch.Tensor, tuple[torch.Tensor, torch.Tensor] | None]`
- `MLP.forward(x: torch.Tensor) -> torch.Tensor`
- `Block.forward(x: torch.Tensor, kv_cache: tuple[torch.Tensor, torch.Tensor] | None = None, use_cache: bool = False) -> tuple[torch.Tensor, tuple[torch.Tensor, torch.Tensor] | None]`
- `MiniGPT.forward(idx: torch.Tensor, targets: torch.Tensor | None = None, kv_cache: KVCache | None = None, use_cache: bool = False) -> tuple[torch.Tensor, torch.Tensor | None, KVCache | None]`
- `MiniGPT.generate(idx: torch.Tensor, max_new_tokens: int, temperature: float = 1.0, top_k: int | None = None, use_kv_cache: bool = True, generator: torch.Generator | None = None) -> torch.Tensor`

- [ ] **Step 1: Write failing model tests**

  Add tests for:
  - forward logits shape `(B, T, vocab_size)` and scalar cross-entropy loss;
  - backward pass producing finite gradients;
  - causal behavior: after changing token `j`, logits at positions `< j` remain equal within tolerance;
  - invalid `n_embd % n_head != 0` raises a clear `ValueError`;
  - cached and uncached one-step forward logits agree within tolerance;
  - source inspection rejects forbidden strings/usages: `nn.Transformer`, `nn.TransformerEncoder`, `nn.TransformerDecoder`, `nn.MultiheadAttention`, and `scaled_dot_product_attention` in `model.py`.

- [ ] **Step 2: Run the focused tests and verify they fail**

  ```bash
  pytest tests/test_model.py -q
  ```

  Expected: import/collection failure because `model.py` does not exist yet.

- [ ] **Step 3: Implement `model.py` with manual attention**

  Implement:
  - token embedding plus learned positional embedding;
  - `CausalSelfAttention` with four linear projections, reshape to `(B, n_head, T, head_dim)`, explicit scaled `q @ k.transpose(-2, -1)`, causal mask, `softmax`, `weights @ v`, merge heads, and output projection;
  - cache concatenation of past K/V for decode steps, with a mask that permits all cached positions plus the current token;
  - pre-norm `Block`: `x + attention(LayerNorm(x))`, then `x + mlp(LayerNorm(x))`;
  - `MLP`: `Linear(n_embd, 4*n_embd) -> GELU -> Linear(4*n_embd, n_embd)`;
  - `MiniGPT`: embeddings, blocks, final LayerNorm, vocabulary head, shifted cross-entropy, and per-layer KV cache;
  - greedy sampling when `temperature <= 0`, temperature sampling otherwise, top-k filtering before sampling;
  - explicit validation for rank, sequence length, and cache batch/device compatibility.

  Do not use a fused attention helper. Keep attention tensors readable enough for the README explanation. Apply `torch.no_grad()` inside generation, not inside `forward`.

- [ ] **Step 4: Run the focused tests and verify they pass**

  ```bash
  pytest tests/test_model.py -q
  ```

  Expected: all model tests pass on CPU.

- [ ] **Step 5: Commit the model**

  ```bash
  git add model.py tests/test_model.py
  git commit -m "feat: implement manual decoder-only transformer"
  ```

---

### Task 3: Add Training Loop and Portable Checkpoints

**Files:**
- Create: `mini-gpt/train.py`
- Create: `mini-gpt/tests/test_train.py`
- Create: `mini-gpt/checkpoint.py`

**Interfaces:**
- Consumes `Config`, `TextDataset`, `get_batch`, and `MiniGPT` from Tasks 1–2.
- `estimate_loss(model: MiniGPT, dataset: TextDataset, config: Config) -> dict[str, float]`
- `train_model(config: Config, data_path: Path, checkpoint_path: Path) -> dict[str, list[float]]`
- `save_checkpoint(path: Path, model: MiniGPT, optimizer: torch.optim.Optimizer, config: Config, tokenizer: CharTokenizer, step: int, metrics: dict) -> None`
- `load_checkpoint(path: Path, device: torch.device) -> tuple[MiniGPT, CharTokenizer, dict]`

- [ ] **Step 1: Write failing training/checkpoint tests**

  Add tests for:
  - a tiny deterministic model overfitting a repeated toy corpus for a bounded number of steps, with final loss lower than initial loss;
  - `estimate_loss` returning finite `train` and `val` values;
  - checkpoint round-trip restoring model config, tokenizer, step, and identical evaluation logits on CPU;
  - `map_location` loading a checkpoint created on another device without requiring CUDA.

- [ ] **Step 2: Run the focused tests and verify they fail**

  ```bash
  pytest tests/test_train.py -q
  ```

  Expected: import/collection failure because training and checkpoint interfaces do not exist yet.

- [ ] **Step 3: Implement checkpoint serialization**

  Store a plain dictionary with `format_version`, model config values, `model_state_dict`, `optimizer_state_dict`, tokenizer `stoi`/`itos`, current step, and metric history. Use `torch.save`; load with `map_location=device`. Reject unsupported/missing format versions with a clear error.

- [ ] **Step 4: Implement the training loop and CLI**

  Implement AdamW, deterministic seed initialization, contiguous dataset use, periodic evaluation under `torch.no_grad()`, optional linear warmup followed by cosine decay, finite-loss checks, and best/latest checkpoint saving. Expose CLI flags for data path, checkpoint path, device, steps, batch size, and seed. Print device and compact model configuration at startup, then print train/validation loss at evaluation intervals.

  Keep the loop usable for both a short CPU smoke run and a longer CUDA run; do not require W&B or external services.

- [ ] **Step 5: Run the focused tests and verify they pass**

  ```bash
  pytest tests/test_train.py -q
  ```

  Expected: all training and checkpoint tests pass on CPU.

- [ ] **Step 6: Run a CPU smoke training command**

  ```bash
  python train.py --device cpu --max-steps 20 --eval-interval 10 --checkpoint checkpoints/smoke.pt
  ```

  Expected: the command exits successfully, prints finite train/validation losses, and creates the checkpoint.

- [ ] **Step 7: Commit training and checkpoints**

  ```bash
  git add train.py checkpoint.py tests/test_train.py
  git commit -m "feat: add training loop and portable checkpoints"
  ```

---

### Task 4: Add Sampling and KV-Cache Benchmark

**Files:**
- Create: `mini-gpt/sample.py`
- Create: `mini-gpt/benchmark.py`
- Create: `mini-gpt/tests/test_sample.py`

**Interfaces:**
- Consumes `load_checkpoint` and `MiniGPT.generate` from Tasks 2–3.
- `generate_text(model: MiniGPT, tokenizer: CharTokenizer, prompt: str, max_new_tokens: int, temperature: float, top_k: int | None, use_kv_cache: bool, device: torch.device, seed: int) -> str`
- `benchmark_generation(model: MiniGPT, prompt_ids: torch.Tensor, new_tokens: int, repeats: int = 3, warmup: int = 1, device: torch.device | None = None) -> dict[str, dict[str, float | str]]`
- Benchmark result keys: `device`, `total_seconds`, `tokens_per_second`, `milliseconds_per_token`, `repeats`, `warmup`, `prompt_tokens`, `generated_tokens`, and `cuda_synchronized`.

- [ ] **Step 1: Write failing sampling/benchmark tests**

  Add tests for:
  - `generate_text` returning the prompt plus non-empty decoded continuation;
  - fixed seed plus fixed settings producing identical output;
  - cached and uncached greedy generation producing identical token IDs;
  - prompt length exactly `block_size` raising a clear error when requesting new tokens that exceed the model context, and a shorter valid prompt succeeding;
  - benchmark returning both `uncached` and `cached` records with finite non-negative timing values and correct token counts.

- [ ] **Step 2: Run the focused tests and verify they fail**

  ```bash
  pytest tests/test_sample.py -q
  ```

  Expected: import/collection failure because sampling and benchmark interfaces do not exist yet.

- [ ] **Step 3: Implement generation CLI**

  Load the checkpoint on the selected device, encode the prompt with the saved tokenizer, call `MiniGPT.generate`, decode only the full result for display, and support `--prompt`, `--max-new-tokens`, `--temperature`, `--top-k`, `--no-kv-cache`, `--device`, and `--seed`. Reject context overflow before generation rather than silently changing the prompt.

- [ ] **Step 4: Implement reproducible benchmark timing**

  Benchmark the same prompt/checkpoint/settings in both modes. Run warmups, repeat timing, use `time.perf_counter()`, and call `torch.cuda.synchronize()` immediately before and after timed CUDA sections. Report median or mean consistently (choose median for robustness and document it). Do not include checkpoint loading or tokenization in generation timing.

- [ ] **Step 5: Run the focused tests and verify they pass**

  ```bash
  pytest tests/test_sample.py -q
  ```

  Expected: all sampling and benchmark tests pass on CPU.

- [ ] **Step 6: Run the benchmark smoke command**

  ```bash
  python sample.py --checkpoint checkpoints/smoke.pt --prompt "ROMEO:" --max-new-tokens 32 --temperature 0 --benchmark --device cpu
  ```

  Expected: generated text is printed, followed by uncached/cached timing records and device metadata.

- [ ] **Step 7: Commit sampling and benchmark**

  ```bash
  git add sample.py benchmark.py tests/test_sample.py
  git commit -m "feat: add generation and kv cache benchmark"
  ```

---

### Task 5: Document Results and Verify the Complete DoD

**Files:**
- Create: `mini-gpt/README.md`
- Create: `mini-gpt/Makefile` or `mini-gpt/scripts/smoke.sh` (choose one WSL-friendly command surface)
- Modify: `mini-gpt/.gitignore`

**Interfaces:**
- Consumes commands and outputs from Tasks 1–4.
- Documents the exact baseline commands, model config, data provenance/checksum, loss log, sample output, and benchmark conditions/results.

- [ ] **Step 1: Add the full documentation checklist**

  Write README sections for:
  - purpose and explicit non-goals;
  - WSL2 setup, CPU fallback, dependency installation, and data acquisition;
  - project tree and tensor-shape table;
  - manual attention formula and why causal masking is required;
  - training and checkpoint commands;
  - generated sample with checkpoint/config details;
  - benchmark table for uncached vs cached generation, including device, prompt length, generated length, warmup/repeats, synchronization, and timing statistic;
  - concise answers: attention is O(n²) because `QKᵀ` considers every token pair; KV cache saves recomputation of historical K/V during autoregressive decoding at the cost of memory;
  - known limitations and follow-up backlog (BPE, Vietnamese, RoPE, LoRA).

- [ ] **Step 2: Add a single smoke command and ignore generated artifacts**

  Ensure the smoke command runs tests plus a short CPU training/generation/benchmark path. Ignore `.venv/`, `__pycache__/`, `checkpoints/`, logs, and generated benchmark outputs while keeping source data instructions/checksum reproducible.

- [ ] **Step 3: Run the complete CPU verification**

  ```bash
  pytest -q
  bash scripts/smoke.sh
  ```

  Expected: all tests pass; smoke training shows a decreasing loss trend; checkpoint loads; sample text is non-empty; benchmark reports both modes.

- [ ] **Step 4: Run the preferred CUDA verification when WSL2 CUDA is available**

  ```bash
  python -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'CPU')"
  python train.py --device cuda --max-steps <short-run> --checkpoint checkpoints/cuda-smoke.pt
  python sample.py --checkpoint checkpoints/cuda-smoke.pt --prompt "ROMEO:" --max-new-tokens 64 --temperature 0 --benchmark --device cuda
  ```

  Expected: CUDA run completes, benchmark metadata says `cuda_synchronized=true`, and the README records the actual measured result. If CUDA is unavailable, retain the CPU result and state that CUDA verification was skipped.

- [ ] **Step 5: Perform a final source/DoD audit**

  Verify no forbidden attention APIs or Stage 1 imports are present, the contiguous 90/10 split is documented, cached and uncached greedy generation agree, and all four required project outcomes are evidenced in README.

- [ ] **Step 6: Commit the complete project**

  ```bash
  git add README.md .gitignore scripts/ tests/ data.py config.py model.py train.py checkpoint.py sample.py benchmark.py requirements.txt
  git commit -m "docs: complete stage 2 mini-gpt project"
  ```

## Execution Order

Implement Tasks 1 through 5 in order. Each task ends with focused tests and a commit; do not begin a later task while the prior task's tests are failing. The LoRA/QLoRA, BPE, Vietnamese, RoPE, and chatbot integration backlog is intentionally outside this plan.
