# 07 — Tech stack toàn lifecycle: chọn công nghệ theo từng khâu

> **Mục tiêu**: bản đồ công nghệ end-to-end cho sản phẩm GenAI (chat + RAG + VLM + agent) — từ dữ liệu đầu vào đến deploy/quan sát — đã research 11/2026, tối ưu cho **máy 8GB VRAM + deploy chi phí 0đ**.
>
> Nguồn mở rộng từ: [06-ECOSYSTEM](README.md). Phần model (LLM/VLM/embedding) xem [05-model-quick-picks.md](05-model-quick-picks.md); phần tối ưu inference xem mục "Tối ưu deploy" trong cùng file đó.

---

## 3 nguyên tắc xuyên suốt

1. **GPU-first, zero-copy**: giữ data trên GPU càng lâu càng tốt (batch `[B,C,H,W]`), tránh ping-pong CPU↔GPU; preprocess và inference nên "dính" nhau qua DLPack/CUDA Array Interface.
2. **Ít service, nhiều vai trò**: máy 0đ có RAM giới hạn — mỗi service phải gánh được nhiều việc (Postgres 3 vai trò, Redis 3 vai trò — xem cuối bài).
3. **Python orchestration + Rust hot path**: không rewrite toàn bộ bằng Rust; Rust đúng chỗ (preprocess GPU, gateway, đoạn CPU-bound) qua PyO3 hoặc service riêng.

---

## 🗺️ Bảng 16 khâu lifecycle

| # | Khâu | Công nghệ chọn | Ghi chú |
|---|---|---|---|
| 1 | Thu thập & ingestion | docling, unstructured · **MinIO** (S3-compatible, self-host 0đ) | Kho PDF/ảnh gốc |
| 2 | Preprocess GPU (lõi phần cứng) | **Kornia** (batch, differentiable) · **kornia-rs** (Rust+CUDA) · OpenCV fallback | Xem chi tiết bên dưới |
| 3 | CV inference | **ONNX Runtime (CUDA EP)** / TensorRT | Export `PP-DocLayoutV3` sang ONNX |
| 4 | LLM / VLM / Embedding | `qwen3.5:9b` · `gemma4:e4b` · `glm-ocr` · `qwen3-embedding:0.6b` | Xem [05-model-quick-picks.md](05-model-quick-picks.md) |
| 5 | Tối ưu inference | llama.cpp flags · MoE CPU offload · KV cache quant · speculative/MTP | Xem mục "Tối ưu deploy" ở [05](05-model-quick-picks.md) |
| 6 | VectorDB | **pgvector** (chính) · Qdrant (filter nặng) · Chroma (prototype) | Benchmark bên dưới |
| 7 | RAG pipeline | chunking · hybrid search · **bge-reranker** · ragas eval | Xem [03-RAG](../03-RAG/) + [04-rag-pipeline-tools.md](04-rag-pipeline-tools.md) |
| 8 | Agent / orchestration | langgraph · MCP · tool-calling schema | Xem [05-AGENTS-PRODUCTION](../05-AGENTS-PRODUCTION/) |
| 9 | Job queue / batch | **arq** (async-native, Redis) · cache theo SHA-256 tài liệu | Chi tiết bên dưới |
| 10 | Serving / API | FastAPI → **axum (Rust)** khi cần throughput · **LiteLLM gateway** | Chi tiết bên dưới |
| 11 | Security / guardrails | guardrails-ai + check ở gateway (prompt injection, PII) | OWASP #1 — chi tiết bên dưới |
| 12 | Observability | **Langfuse** (MIT self-host) · chuẩn **OpenTelemetry** | Chi tiết bên dưới |
| 13 | Evaluation & CI/CD | golden dataset · eval gate trong GitHub Actions | Chi tiết bên dưới |
| 14 | Deploy / Infra | Oracle Always Free · Docker · Nginx/Caddy · SSL | Xem [07-DEPLOYMENT](../07-DEPLOYMENT/) |
| 15 | Frontend | Gradio (demo) → Next.js/React (product UI) | Tùy giai đoạn |
| 16 | Cache | **Redis** (kiêm job broker + rate-limit) | Xem mục 9 |

---

## 1. Lớp lõi phần cứng — GPU pipeline

| Công nghệ | Dùng cho | Số liệu / lưu ý |
|---|---|---|
| **Kornia** (PyTorch) | Preprocess ảnh **differentiable trên GPU**: deskew, denoise, normalize, geometry — batch `[B,C,H,W]` | Thắng ở regime **batched float trên GPU**; OpenCV/albumentations thắng ảnh lẻ uint8 trên CPU → Kornia cho batch, OpenCV cho ảnh đơn. Differentiable → train end-to-end được |
| **kornia-rs** (Rust) | CV kernels chạy **CPU SIMD hoặc CUDA**, cùng 1 API | Zero-copy sang PyTorch/TensorRT (DLPack, CUDA Array Interface); "fuse 1 frame camera → model input trong 1 CUDA kernel"; **không GIL** → preprocess đa luồng song song Python |
| **ONNX Runtime (CUDA EP)** | Inference model CV (layout detect, classifier) | Linh hoạt hơn TensorRT; TensorRT nhanh nhất khi pipeline cố định |
| **CV-CUDA** (NVIDIA) | Pre/post-process production-scale | Dùng cho Bing Visual Search; hơi "nặng" cho dự án cá nhân |
| **CuPy** | NumPy trên GPU | Xử lý mảng thuần khi cần |

Pipeline OCR đề xuất: `decode → Kornia GPU preprocess (batch) → ORT/TensorRT layout detect → VLM đọc text` — toàn bộ trên GPU.

---

## 2. Rust trong stack — nằm ở đâu?

**Không** thay Python cho ML. Rust là lớp hiệu năng + hạ tầng:

| Công nghệ | Vai trò | Số liệu |
|---|---|---|
| **axum + ort/candle** | Dịch vụ inference / LLM gateway | Benchmark MiniLM embedding: **1.5x throughput, 3x ít RAM, p99 tốt hơn 3x** so FastAPI+onnxruntime; gateway mẫu: 5000+ req/s, <3ms |
| **ort** (v2.0) | ONNX Runtime bindings — production-ready | Inference ONNX trong Rust |
| **candle** / **burn** | candle = inference (WASM được); burn = training multi-backend | Candle nếu chỉ inference |
| **tokenizers** (HF) | Đã là Rust sẵn — dùng qua Python bindings | Nhanh hơn tokenizer thuần Python nhiều lần |
| **PyO3** | Nhúng Rust vào Python cho **hot path** (parse hậu xử lý, tính box, trim markdown...) | Chiến lược: Python orchestration, Rust hot-path |

---

## 3. VectorDB — chọn theo benchmark

Throughput single-thread (MedRAG-50k): `Redis 764 > Qdrant 377 > Milvus 342 > Weaviate 341 > pgvector 257 > Chroma 197 > LanceDB 70 QPS`

| DB | Điểm mạnh | Khi nào dùng |
|---|---|---|
| **pgvector** ⭐ | 1 DB với app data: vector + metadata + FTS hybrid + ACID + backup + JOIN; HNSW/IVFFlat, 6 distance metrics, binary quantization | **Mặc định** — deploy 0đ không thêm service; hàng triệu vector thoải mái |
| **Qdrant** (Apache 2.0, Rust) | Payload filter mạnh (per-user memory agent), hybrid BM25 | Khi RAG nặng filter / nhiều user |
| **Chroma** | Embedded kiểu SQLite, `pip install` | Prototype / notebook / eval |
| **LanceDB** | On-disk, không server | Data lớn không vừa RAM (đánh đổi: p95 ~22ms) |

---

## 4. Job queue & cache — pipeline batch (OCR, ingestion)

So sánh: **arq** (async-native `asyncio`, Redis, ops thấp, built-in cron) vs Celery (ops cao: beat + workers + Flower) vs RQ (sync-only) vs Dramatiq.

→ **Chọn arq** cho stack FastAPI hiện tại (native async, 1 worker, Redis sẵn có).

Pipeline tài liệu 7 bước: `ingestion → validation → preprocessing → OCR → post-processing → storage → notification` — mỗi bước là 1 job trong queue.

**Cache**: Redis `SETEX` theo **SHA-256 của tài liệu** — upload trùng bỏ qua OCR hoàn toàn; thêm `INCR` rate-limit + health-check endpoint trước khi production-ready.

---

## 5. Serving & Security ở gateway

**Serving**: FastAPI cho app; **axum (Rust)** khi cần latency/throughput; **LiteLLM** làm gateway thống nhất (model alias, fallback chain, budget) cho hybrid local↔cloud.

**Security — OWASP #1: prompt injection.** Nguyên nhân cấu trúc: LLM trộn "instructions tin cậy" và "data không tin cậy" vào cùng 1 token stream. Các lớp tấn công: direct (input user), **indirect (nguy hiểm nhất — ẩn trong tài liệu RAG / tool result mà agent đọc)**, tool-mediated, jailbreak.

- Check ở **gateway** (mọi request đều qua): detect prompt injection/jailbreak (classifier + ngưỡng), **PII redaction** (regex deterministic cho secrets/PII), allow/block model, policy cho tool/MCP calls
- Quyền hạn (authorization) của model → giữ ở **application**, không ủy quyền cho gateway
- Chế độ: **fail-open + audit log** (ghi mọi quyết định guardrail để review); policy "1 violation = block" cho nội dung zero-tolerance
- Tool: **guardrails-ai** (đã có trong stack) + hook của LiteLLM / gateway tương đương

---

## 6. Observability

| Tool | Điểm mạnh | Ghi chú |
|---|---|---|
| **Langfuse** ⭐ | MIT self-host toàn bộ workflow (Postgres + ClickHouse): tracing, prompt management, dataset, experiment, LLM-as-judge, annotation queue, alert | **Đã có trong stack** — self-host không giới hạn usage |
| Arize Phoenix | Apache 2.0, local-first (SQLite/ClickHouse) | Phương án 2; có migration path sang Langfuse |
| LangSmith | Hợp team LangChain/LangGraph | Proprietary SaaS |

Chuẩn hóa instrumentation bằng **OpenTelemetry (OTLP)** — vendor-neutral, đổi tool không phải viết lại.

---

## 7. Evaluation & CI/CD — "eval gate"

LLM non-deterministic → test 1 lần không đủ. Ba lớp liên tục:

1. **Offline regression (CI/CD)**: golden dataset **50-200 case** gắn failure taxonomy (`retrieval failure` / `reasoning error` / `format violation` / `safety violation`); production failure mới → bổ sung liên tục
2. **Eval gate trong GitHub Actions**: PR (đổi prompt/model/retrieval) → chạy dataset → chấm (ragas/LLM-judge) → **pass/fail exit code** chặn merge; **chạy nhiều lần** để bảo đảm ổn định
3. **Online telemetry + feedback loop**: trace production (Langfuse) → case xấu (thumbs-down) → đưa về golden set

Ngưỡng chấm: mỗi metric có ngưỡng calibrated từ case human-labeled; regression = so candidate vs baseline trên **cùng dataset** (điểm trung bình tăng vẫn có thể che regression ở case critical → phải xem cả case must-pass).

---

## 8. Nguyên tắc tiết kiệm service (máy 0đ)

| Service | Gánh bao nhiêu vai trò |
|---|---|
| **Postgres** | App data + **pgvector** + storage cho Langfuse |
| **Redis** | Cache (SHA-256) + job queue (arq) + rate-limit |
| **Ollama/llama.cpp** | LLM + VLM + embedding (1 server, 3 model) |

---

## Liên kết

- [05-model-quick-picks.md](05-model-quick-picks.md) — chọn model + tối ưu inference (phần 4-5 của lifecycle).
- [04-rag-pipeline-tools.md](04-rag-pipeline-tools.md) — serving tools chi tiết.
- [02-embedding-models.md](02-embedding-models.md) — bảng embedding.
- [07-DEPLOYMENT](../07-DEPLOYMENT/) — hạ tầng deploy (khâu 14).
