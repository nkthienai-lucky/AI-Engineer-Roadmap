# 05 — Model chọn nhanh (chạy CPU / server nhỏ)

> **Mục tiêu**: chọn model local chạy được trên CPU/server nhỏ cho từng việc (chat, VLM, OCR, embedding) — bảng tra + hiểu bản GGUF quantized là gì, tìm trên Hugging Face thế nào.
>
> Nguồn mở rộng từ: [06-ECOSYSTEM.md](../../../06-ECOSYSTEM.md)

---

## Bảng tra nhanh

| Việc | Model đề nghị | Ghi chú |
|---|---|---|
| Chat tiếng Việt | **Qwen2.5-3B-Instruct Q4** | Qwen hỗ trợ tiếng Việt tốt |
| Chat nhẹ nhất | Qwen2.5-0.5B/1.5B Q4 | Rất nhanh trên CPU |
| VLM (ảnh) | **SmolVLM2-2.2B** | Chạy CPU được |
| OCR/VLM tài liệu | MiniCPM-V, Qwen2.5-VL-3B | Xem thêm dự án `doc-ocr` trong workspace |
| Embedding | Qwen3-Embedding-0.6B | Xem bảng trên |

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

- **Qwen2.5-3B-Instruct Q4**: best pick chat tiếng Việt trong tầm CPU — Qwen hỗ trợ tiếng Việt tốt, bản Q4 chạy mượt.
- **Qwen2.5-0.5B/1.5B Q4**: nhẹ nhất, cực nhanh trên CPU — hợp demo/nhiệm vụ đơn giản, đánh đổi chất lượng.
- **SmolVLM2-2.2B**: VLM nhỏ xử lý ảnh, chạy CPU được — hợp mô tả ảnh, truy vấn ảnh đơn giản.
- **MiniCPM-V, Qwen2.5-VL-3B**: OCR/VLM tài liệu — đọc bảng/công thức/ảnh chụp tài liệu (xem `doc-ocr` ở trên).
- **Qwen3-Embedding-0.6B**: embedding cho RAG — chi tiết ở [02-embedding-models.md](02-embedding-models.md).

## Liên kết

- [02-embedding-models.md](02-embedding-models.md) — bảng embedding đầy đủ (gồm Qwen3-Embedding-0.6B).
- [04-rag-pipeline-tools.md](04-rag-pipeline-tools.md) — Serving (Ollama/llama.cpp/vLLM) là nơi nạp các model này.
- [06-paid-infrastructure.md](06-paid-infrastructure.md) — khi nào model nhỏ free không đủ thì nâng cấp GPU trả phí.
