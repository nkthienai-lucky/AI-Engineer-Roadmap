# Stage 2 Mini-GPT Design

**Date:** 2026-11-09  
**Status:** Draft — awaiting approval

## Intent

Build the Stage 2 learning project from the AI Engineer Roadmap: a small decoder-only GPT trained from scratch on a Shakespeare char-level dataset. The project is intended to make the Transformer internals observable and explainable, not to replace the production-quality chatbot from Stage 1.

The outcome is a runnable Mini-GPT that can:

1. execute a hand-written causal self-attention and Transformer stack;
2. train with next-token cross-entropy and show a decreasing train/validation loss;
3. generate text from a saved checkpoint; and
4. provide a reproducible comparison of generation with and without KV cache.

The project will live independently at `D:\Thien\Code\mini-gpt`, alongside `chatbot-cli`, as its own Git repository. The supported workflow is WSL2 Ubuntu, using the existing CUDA environment and RTX 5060 when available; CPU fallback remains supported for tests and smoke runs. Stage 1 remains unchanged.

## Goals

- Understand and implement token IDs, embeddings, positional information, Q/K/V attention, causal masking, multi-head attention, residual connections, LayerNorm, MLP, logits, and next-token loss.
- Use PyTorch modules where they express individual building blocks (`Linear`, `Embedding`, `LayerNorm`, `GELU`) while writing the Transformer composition by hand.
- Validate the model in small steps: shape/causal-mask tests, backward pass, one-batch overfit test, then a short real training run.
- Produce enough logs and measurements to explain what happened in a README or notebook.
- Keep the baseline runnable on CPU, while using the existing WSL2 PyTorch/CUDA environment and RTX 5060 when available.

## Non-goals

- Do not integrate Mini-GPT as a third backend in `chatbot-cli` during this stage.
- Do not use `nn.Transformer` or `nn.Transformer*`, `nn.MultiheadAttention`, Hugging Face Trainer, or a pre-trained model for the baseline.
- Do not attempt production chat quality, instruction tuning, RLHF/DPO, RAG, deployment, or a web UI.
- LoRA/QLoRA is explicitly postponed until the baseline Mini-GPT and its measurements are complete.
- BPE and Vietnamese data are follow-up experiments, not blockers for the baseline acceptance.

## Constraints and decisions

- **Project boundary:** new independent `mini-gpt/` project; no changes to `chatbot-cli` logic.
- **Baseline data:** Shakespeare char-level. The data loader should make replacing `data/input.txt` straightforward, but no tokenizer abstraction is required beyond a small character vocabulary for the first version.
- **Framework:** PyTorch, using the roadmap's existing WSL2 environment (`torch 2.11.0+cu128`). CPU fallback is required for tests and small smoke runs.
- **Model shape:** a compact configurable model suitable for an 8 GB GPU and CPU smoke tests. The builder may choose exact defaults, but they must be exposed in one config/dataclass rather than scattered constants.
- **Position encoding:** learned positional embeddings for the first working baseline because they are simpler to verify. RoPE will be a documented follow-up experiment unless the builder can add it without weakening the baseline.
- **Attention implementation:** the educational core must compute Q/K/V projections, scaled dot products, causal masking, softmax, and weighted values explicitly. `torch.nn.functional.scaled_dot_product_attention` and other fused attention helpers are not allowed in the core. A separate optimized path may be added later only as an explicitly labeled comparison.
- **Architecture:** decoder-only, pre-norm blocks:
  - token embedding + learned positional embedding;
  - hand-written multi-head causal self-attention;
  - residual connection;
  - LayerNorm;
  - MLP `d_model -> 4*d_model -> d_model` with GELU;
  - final LayerNorm and language-model head;
  - cross-entropy over shifted next-token targets.
- **Training:** deterministic seed where practical, a contiguous 90/10 train/validation split (first 90% of the character stream for train, final 10% for validation), AdamW, configurable learning rate, optional warmup/cosine schedule, periodic evaluation, and checkpoint containing model/config/vocabulary/training metadata.
- **Generation:** autoregressive sampling with temperature and optional top-k. The output must be reproducible with a seed when sampling settings and checkpoint are fixed.
- **KV cache:** implement an explicit cache path in the model, not a hidden framework optimization. The uncached path recomputes the full prefix at each generation step; the cached path processes the prompt once and then one new token per step. Both paths must use the same checkpoint, prompt, number of generated tokens, and sampling seed for a fair benchmark.
- **Benchmark:** report total generation time and latency/token for both modes, plus a short explanation of the trade-off. The benchmark must state device, model config, prompt length, generated length, warm-up/repetitions, and whether synchronization was used on CUDA. CPU benchmark is sufficient for acceptance when CUDA is unavailable; when CUDA is available, run and record the preferred GPU benchmark as an additional result.
- **Documentation:** README includes setup, commands, architecture diagram or shape table, loss evidence, sample output, benchmark table, known limitations, and answers to “why is attention O(n²)?” and “what does KV cache save?”. A notebook is optional; the README is required.

## Approaches considered

### A — Independent `mini-gpt/` project (chosen)

Create a clean project beside `chatbot-cli`, with `model.py`, `train.py`, `sample.py`, `data/`, tests, and README. This preserves the Stage 1 application boundary and matches the roadmap's project spec.

### B — Add `mini_gpt/` inside `chatbot-cli`

This would keep one repository but mix the API chatbot's runtime dependencies, entrypoints, and documentation with a training experiment. It makes later packaging and deployment less clear, so it is not chosen.

### C — Add Mini-GPT as a chatbot backend

This would make the result look like one product sooner, but a small char-level model is not an equivalent chat backend. It would prioritize adapter/UI work over the Stage 2 learning outcomes and make failures harder to diagnose. It is not chosen.

## Proposed structure

```text
D:\Thien\Code\mini-gpt\
├── model.py          # CausalSelfAttention, MLP, Block, MiniGPT
├── train.py          # data, batching, optimizer, evaluation, checkpoint
├── sample.py         # checkpoint loading, generation, KV benchmark
├── config.py         # one configurable model/training config
├── data/
│   └── input.txt     # Shakespeare baseline, downloaded or supplied separately
├── tests/
│   ├── test_model.py # shapes, causal behavior, backward, cache equivalence
│   └── test_train.py # tiny smoke/overfit test
├── notebooks/        # optional explanation notebook
├── requirements.txt
└── README.md
```

The builder may simplify filenames if the same boundaries remain clear. Generated checkpoints, logs, and downloaded data should be ignored by Git; a small sample or checksum may be documented instead.

## Acceptance criteria

- `model.py` imports and runs a forward/backward smoke test on CPU.
- No forbidden Transformer convenience modules are used.
- Causal masking test demonstrates that a token cannot read future positions.
- The model can overfit one tiny batch, with loss approaching zero or clearly falling by the documented threshold.
- A short real run produces a decreasing loss trend and a saved checkpoint.
- `sample.py` loads the checkpoint and generates non-empty text.
- Cached and uncached generation produce equivalent token sequences under deterministic settings, or document the exact source of any expected difference.
- A benchmark table compares both modes with reproducible conditions.
- README answers the two required interview questions and records limitations.
- All tests and a CPU smoke command pass without requiring Ollama, an API key, or `chatbot-cli`.

## What remains for the builder

- Choose exact compact default hyperparameters after checking available device memory.
- Choose a reliable Shakespeare data source or include clear instructions for obtaining `input.txt`; do not silently commit a large generated/downloaded file.
- Implement tests and commands so the baseline can be verified without a long training run.
- Decide whether RoPE is added after the learned-position baseline as a separate experimental flag.
- Keep the implementation educational and readable; optimize only where needed for the benchmark.

## Follow-up backlog

1. Replace char-level encoding with a small BPE tokenizer and compare sequence length.
2. Add Vietnamese poetry/prose as a second dataset.
3. Implement and compare RoPE with learned positional embeddings.
4. Add LoRA fine-tuning of `Qwen2.5-0.5B` on Colab free GPU.
5. Consider a later, explicit integration experiment rather than changing the Stage 1 chatbot contract.
