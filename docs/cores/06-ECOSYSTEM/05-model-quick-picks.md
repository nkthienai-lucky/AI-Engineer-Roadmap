# 05 — Model chọn nhanh (chạy CPU / server nhỏ)

> **Mục tiêu**: chọn model local chạy được trên CPU/server nhỏ cho từng việc (chat, VLM, OCR, embedding) — bảng tra + hiểu bản GGUF quantized là gì, tìm trên Hugging Face thế nào.
>
> Nguồn mở rộng từ: [06-ECOSYSTEM](README.md)

---

## Bảng tra nhanh

| Việc | Model đề nghị | Ghi chú |
|---|---|---|
| Chat tiếng Việt | **Qwen3.5-9B** (Q4) | Benchmark 11/2026 trên RTX 5060: 42 tok/s, TTFT 0.17s — điểm ngọt chatbot |
| Chat nhẹ nhất | Qwen2.5-0.5B/1.5B Q4 | Rất nhanh trên CPU |
| VLM (ảnh) | **Gemma 4 E4B** | Đa nhiệm (ảnh + audio); OCR tiếng Việt chính xác nhất trong các model đo (170 tok/s) |
| OCR/VLM tài liệu | **GLM-OCR** (2.7GB, 221 tok/s) | Chuyên tài liệu phức tạp; MiniCPM-V / Qwen-VL khi cần VLM kèm; xem dự án `doc-ocr` |
| Model "khỏe" cho máy 8GB VRAM | Qwen3.6-35B-A3B (MoE) | Chất lượng 35B nhưng chỉ active 3B/token → 15-27 tok/s; prefill chậm (203 tok/s) |

Nguồn model: [huggingface.co](https://huggingface.co/) (tìm bản GGUF quantized).

---

## Bản GGUF quantized là gì?

- **GGUF** là định dạng file model của llama.cpp (hậu duệ GGML) — đóng gói trọng số + metadata thành 1 file, load trực tiếp bằng llama.cpp/Ollama, tối ưu cho CPU.
- **Quantized (lượng tử hóa)**: nén trọng số từ FP16 (16-bit) xuống 4–8 bit. Ví dụ **Q4** = 4-bit: model 3B "học" hết cỡ vài chăng MB → còn ~2GB RAM, chạy mượt trên CPU/laptop. Trade-off: mất nhẹ chất lượng (thường khó nhận thấy với Q4_K_M), đổi lấy RAM/VRAM giảm 3–4× và tốc độ tăng.
- Các hậu tố hay gặp: `Q4_K_M` (4-bit, khuyến nghị chung — cân bằng tốt), `Q5_K_M`, `Q8_0` (gần nguyên gốc, nặng hơn), `F16` (nguyên gốc). Chữ `K_M`/`K_S` = biến thể "medium/small" của schema k-quant (K-quants).
- Nguyên tắc chọn quant: máy yếu → Q4_K_M; cần chất lượng tối đa, còn RAM → Q8_0; so sánh bằng eval thật (ví dụ CER/WER với OCR) trước khi chốt.

## Tìm trên Hugging Face thế nào?

1. Vào [huggingface.co](https://huggingface.co/), tìm theo tên model + "GGUF" — ví dụ: `Qwen2.5-3B-Instruct GGUF`, `SmolVLM2 GGUF`.
2. Chọn bản do cộng đồng đóng gói uy tín (nhiều repo cộng đồng re-quant rất tốt); kiểm tra tab Files xem có đúng kích thước quant bạn cần (`Q4_K_M`...).
3. Tải 1 file `.gguf` duy nhất → nạp vào **Ollama** (`ollama run`) hoặc **llama.cpp** (xem mắt xích Serving trong [04-rag-pipeline-tools.md](04-rag-pipeline-tools.md)).
4. Với model đa phương thức (VLM/OCR) cần thêm file projector thị giác (thường `mmproj-*.gguf`) đi kèm.

## Tham chiếu dự án `doc-ocr` trong workspace

- Dự án `doc-ocr` (thư mục `doc-ocr` ngoài cùng workspace) là ví dụ chạy thật pattern này: **MiniCPM-OCR 1.0 (VLM 1.3B)** chạy **llama.cpp** trên GPU cá nhân (RTX 5060 8GB, WSL), layout bằng PP-DocLayoutV3, số hóa sách giáo khoa tiếng Việt sang Markdown, chất lượng đo bằng CER/WER.
- Điểm đáng học: cùng một hướng "model nhỏ + llama.cpp + quantization" — 3 cấu hình server llama.cpp (F16 1 slot / F16 8 slot / Q8_0 8 slot) cho thấy trade-off quant ↔ throughput được đo đạc hẳn hoi trước khi chọn.

---

## Ghi chú từng dòng

- **Qwen3.5-9B Q4**: best pick chat tiếng Việt cho máy 8GB VRAM (đã đo: thông minh nhất trong nhóm dense <10B, trả lời tiếng Việt tốt).
- **Qwen2.5-0.5B/1.5B Q4**: nhẹ nhất, cực nhanh trên CPU — hợp demo/nhiệm vụ đơn giản, đánh đổi chất lượng.
- **Gemma 4 E4B**: VLM đa nhiệm (ảnh + audio); trong test OCR tiếng Việt 11/2026 cho kết quả khớp ground-truth chính xác nhất.
- **GLM-OCR**: chuyên OCR tài liệu phức tạp (layout rối, bảng, dấu tiếng Việt) — nhanh nhưng có thể sai ~5% dấu; dùng kèm VLM để kiểm chứng khi cần độ chính xác cao.
- **Qwen3.6-35B-A3B**: MoE 3B active — "chất lượng 35B, giá 3B"; hợp khi cần model khỏe trên máy nhỏ (xem mục tối ưu bên dưới).
- **Qwen3-Embedding-0.6B**: embedding cho RAG — chi tiết ở [02-embedding-models.md](02-embedding-models.md).

---

## Tối ưu deploy model mạnh hơn trên GPU nhỏ (11/2026)

Các kỹ thuật chạy model "lớn hơn sức chứa VRAM" không cần mua card mới:

- **MoE CPU offload** (`llama.cpp --n-cpu-moe N`): giữ expert của MoE trong RAM, trunk + KV cache vẫn ở GPU — mỗi token chỉ kích hoạt vài expert (VD Qwen3.6-35B-A3B active 3B/35B). Đây là "mánh" chính giúp 8GB VRAM chạy được model 35B.
- **KV cache quantization** (`-ctk q8_0 -ctv q4_0`): nén cache attention → giảm 50-66% VRAM mỗi token context → context dài hơn trong cùng VRAM.
- **Speculative decoding / MTP**: draft model nhỏ (hoặc đầu MTP có sẵn của model, VD Qwen3.8) đoán trước token — speedup decode tới ~2.8x.
- **Chọn quant đúng**: `Q4_K_M` là điểm ngọt; quant 1-bit (IQ1) nhét vừa VRAM nhưng **chất lượng sụp** (benchmark: 27B 1-bit thua 8B 4-bit); imatrix quant (`IQ4_XS`) tốt hơn ở bit thấp.
- **Chọn engine**: `llama.cpp` khi cần full control flags (GPU nhỏ) > `Ollama` (UX tốt, đủ dùng 1 người) > `vLLM`/`SGLang` khi cần **nhiều user đồng thời** (continuous batching).
- **Flag hay quên**: flash attention (`-fa on`), `--load-mode none` (nạp RAM thay vì mmap), tune batch (`-b 512 -ub 256`), `keep_alive` dài để giữ model nóng.
- **Hybrid local + cloud**: gateway (VD LiteLLM) route theo độ phức tạp/độ nhạy cảm dữ liệu — câu dễ xử lý bằng model local, câu khó đẩy lên cloud API; thực tế 70-80% traffic đi được đường rẻ.

## Liên kết

- [07-tech-stack-lifecycle.md](07-tech-stack-lifecycle.md) — bản đồ công nghệ toàn lifecycle 16 khâu (GPU pipeline, Rust, VectorDB, job queue, security, CI/CD).
- [02-embedding-models.md](02-embedding-models.md) — bảng embedding đầy đủ (gồm Qwen3-Embedding-0.6B).
- [04-rag-pipeline-tools.md](04-rag-pipeline-tools.md) — Serving (Ollama/llama.cpp/vLLM) là nơi nạp các model này.
- [06-paid-infrastructure.md](06-paid-infrastructure.md) — khi nào model nhỏ free không đủ thì nâng cấp GPU trả phí.
