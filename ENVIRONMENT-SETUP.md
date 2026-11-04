# Môi trường AI Engineer Roadmap (WSL2) — Hướng dẫn cài đặt & sử dụng

> Môi trường học **trọn bộ 01 → 08** đã được cài đặt sẵn trên **WSL2 Ubuntu 26.04**,
> tận dụng **GPU RTX 5060 8GB** (CUDA). Một venv Python chung + các model local qua Ollama.
>
> Tất cả cài đặt theo hướng **không cần sudo** (sudo trên máy yêu cầu mật khẩu):
> Postgres chạy bằng Docker, Node qua nvm, Ollama ở user-mode, Python qua uv.

---

## 1. Tổng quan tài nguyên

| Thành phần | Giá trị |
|---|---|
| OS | Ubuntu 26.04.1 LTS (WSL2) |
| GPU | NVIDIA GeForce RTX 5060 — 8151 MiB (Blackwell, compute capability 12.0) |
| RAM | ~23 GB |
| Ổ cứng trống | ~930 GB |
| Python venv | `$HOME/ai-roadmap/.venv` (Python 3.12.16) |
| Docker | 2.40.3 (Compose plugin) |
| Node | v20.20.2 / npm 10.8.2 (nvm) |
| Ollama | v0.40.14 (user-mode, `~/.local/bin/ollama`) |
| Postgres | pgvector/pgvector:pg16 (container `roadmap_pg`, port 5432) |

---

## 2. Cách kích hoạt môi trường

```bash
# Vào WSL
wsl -d Ubuntu

# Python venv chung (mọi module 01→08 dùng chung)
source ~/ai-roadmap/.venv/bin/activate
# hoặc không activate, gọi trực tiếp:
~/ai-roadmap/.venv/bin/python

# Node (nvm)
export NVM_DIR="$HOME/.nvm"; . "$NVM_DIR/nvm.sh"; nvm use 20

# Ollama (user-mode)
export PATH="$HOME/.local/bin:$PATH"
ollama serve &        # nếu chưa chạy
ollama list           # xem models đã kéo
```

---

## 3. Dịch vụ & cách khởi động lại

| Dịch vụ | Khởi động | Kiểm tra |
|---|---|---|
| **Ollama** | `setsid nohup ~/.local/bin/ollama serve > ~/.local/ollama/ollama.log 2>&1 &` | `curl localhost:11434/api/version` |
| **Postgres + pgvector** | `docker start roadmap_pg` (đã có `--restart unless-stopped`) | `docker exec roadmap_pg psql -U roadmap -d roadmap -c "SELECT extversion FROM pg_extension WHERE extname='vector';"` |
| **Node** | nvm tự động khi `. nvm.sh` | `node --version` |

Kết nối Postgres: `host=localhost port=5432 dbname=roadmap user=roadmap password=roadmap`.

---

## 4. Models đã cài (Ollama) — lineup 11/2026 (sau benchmark trên RTX 5060 8GB)

| Model | Kích thước | Dùng cho | Benchmark đo tại chỗ |
|---|---|---|---|
| `qwen3.5:9b` | 6.6 GB | **LLM chatbot chính** (01, 05, 08) | 42 tok/s · TTFT 0.17s · prefill 2140 tok/s · quiz 3/4 |
| `gemma4:e4b` | 6.6 GB | **VLM tổng quát** + audio (04, 08) | OCR tiếng Việt chính xác nhất trong các model đo · 170 tok/s |
| `glm-ocr` | 2.7 GB | **Chuyên OCR/tài liệu** phức tạp (04, 08) | 221 tok/s · lỗi ~5% dấu tiếng Việt |
| `qwen3-embedding:0.6b` | 639 MB | Embedding cho RAG (03, 08) | — |

> Tất cả offload GPU (VRAM inference 5.5–6.6GB). Benchmark đầy đủ: `wsl_models_bench.sh` + `wsl_llm_rigorous.sh` (tại gốc `D:/Thien/Code/`).
>
> **Đã đo và loại (11/2026):** `qwen2.5:1.5b/3b`, `qwen3:8b`, `Sailor2-8B`, `qwen2.5vl:3b`, `SmolVLM2`, `minicpm-v4.6`, `lfm3-vl`, `qwen3-vl:4b/8b` (bug template thinking của Ollama — [issue #14798](https://github.com/ollama/ollama/issues/14798)), `qwen3.6:35b-a3b` (thông minh nhất — quiz 4/4 — nhưng prefill chỉ 203 tok/s; giữ làm model "escalation" khi cần chất lượng cao, kéo lại khi cần: `ollama pull qwen3.6:35b-a3b` ~22GB).

---

## 5. Python stack theo module

| Module | Packages chính (đã cài) |
|---|---|
| **02 — LLM Internals** | `torch 2.11.0+cu128`, `torchvision`, `torchaudio`, `tiktoken`, `einops`, `accelerate`, `transformers`, `wandb`; **minbpe** (clone tại `~/ai-roadmap/minbpe`, import qua `sys.path`) |
| **03 — RAG** | `langchain`, `llama-index`, `docling`, `unstructured`, `chromadb`, `pgvector`, `psycopg`, `sqlalchemy`, `faiss-cpu`, `sentence-transformers`, `FlagEmbedding` (bge-reranker), `ragas`, `deepeval` |
| **04 — VLM** | `transformers`, `peft` (LoRA/QLoRA), `bitsandbytes`, `qwen-vl-utils`, `pillow`, `opencv-python-headless` |
| **05 — Agents & Prod** | `langgraph`, `mcp`, `openai`, `anthropic`, `guardrails-ai`, `langfuse` |
| **07/08 — Deploy/Capstone** | `fastapi`, `uvicorn[standard]`, `sse-starlette`, `gradio`, `python-multipart` |

Tất cả nằm trong **một venv chung** `~/ai-roadmap/.venv` — import xuyên module, hợp cho capstone.

---

## 6. Smoke-test đã chạy (kết quả)

| Module | Kiểm tra | Kết quả |
|---|---|---|
| 01 | Ollama OpenAI-compatible `/v1/chat/completions` + gradio/httpx | ✅ |
| 02 | torch CUDA forward+backward trên GPU · tiktoken BPE · minbpe | ✅ |
| 03 | chromadb NN retrieval · ragas faithfulness · docling/llama-index | ✅ |
| 03b | pgvector insert + ANN query (`<->`) trên Docker Postgres | ✅ |
| 04 | transformers · qwen-vl-utils · peft LoRA config · bitsandbytes | ✅ |
| 05 | langgraph graph compile+invoke · mcp/langfuse/openai/anthropic | ✅ |
| 06–08 | Postgres+pgvector Up · FastAPI+SSE · SQLAlchemy+psycopg | ✅ |

**15/15 PASS.**

---

## 7. Ghi chú dependency (đã xử lý, cần nhớ khi nâng cấp)

Một venv chung chứa cả stack RAG + VLM + Agents nên có xung đột version — đã cân bằng như sau:

- **`langchain-community` = 0.3.29** (không nâng lên 0.4.x): ragas 0.4.3 import `langchain_community.chat_models.vertexai` — module này bị loại ở 0.4.x.
- **`huggingface-hub` = 1.33.0** (>=1.5,<2): transformers/sentence-transformers/peft yêu cầu >=1.5.
- **`transformers` = 5.17.0** (không phải bản mới nhất 5.22): tương thích sentence-transformers/ragas.

⚠️ Nâng cấp 1 trong các gói trên có thể phá vỡ cân bằng — kiểm tra lại `import ragas, transformers, peft, sentence_transformers` sau mọi thay đổi.

---

## 8. Cài đặt lại từ đầu (các script)

Các script cài đặt nằm tại `doc-ocr/workspace/wsl_*.sh` (chạy trong WSL):

| Script | Việc |
|---|---|
| `wsl_install_node_pg.sh` | Node (nvm) + Postgres/pgvector (Docker) |
| `wsl_install_ollama.sh` | Ollama user-mode (`.tar.zst`) |
| `wsl_start_ollama.sh` | Khởi động `ollama serve` |
| `wsl_pull_models.sh` | Kéo models roadmap (lineup cũ) |
| `wsl_models_pull.sh` / `wsl_models_bench.sh` / `wsl_llm_rigorous.sh` / `wsl_models_cleanup.sh` | Kéo / benchmark / đo kỹ chatbot / dọn models — lineup 11/2026 (tại gốc `D:/Thien/Code/`) |
| `wsl_venv_core.sh` | Venv + core (FastAPI/Gradio/jupyter) |
| `wsl_venv_torch.sh` | PyTorch cu128 + training stack |
| `wsl_fix_trainstack.sh` / `wsl_setup_minbpe.sh` | Sửa minbpe + phần training |
| `wsl_venv_rag.sh` | RAG stack |
| `wsl_venv_vlm_agents.sh` | VLM + Agents stack |
| `wsl_fix_ragas.sh` / `wsl_fix_hub.sh` | Sửa xung đột version |
| `wsl_verify_all_imports.sh` | Verify import toàn bộ |
| `wsl_smoke_all.sh` | Smoke test 01→08 |
