# Giai đoạn 4 — VLM & Multimodal (Tuần 12–14)

> **Mục tiêu:** Hiểu model "nhìn" được ảnh + text — kiến trúc, cách dùng, cách deploy model nhẹ.
> **Đầu ra:** App hỏi đáp ảnh (image captioning / VQA) chạy local + demo public.

---

## 🎓 Nội dung cần học

### Tuần 12 — Vision foundation
- CNN → ViT (Vision Transformer): patch embedding, [CLS] token
- **CLIP**: contrastive learning ảnh-text, zero-shot classification
- Biểu diễn ảnh thành "visual tokens" — cầu nối giữa vision và language

### Tuần 13 — VLM kiến trúc & thực hành
- Kiến trúc VLM: vision encoder + projector + LLM decoder
- Các model nhẹ đáng biết: **SmolVLM2** (2.2B), **Qwen2.5-VL** (3B/7B), LLaVA, MiniCPM-V
- Dùng VLM qua API/transformers: captioning, VQA, OCR, object grounding
- Prompt cho VLM: hỏi gì với ảnh, giới hạn (đếm, đọc chữ nhỏ, không gian)

### Tuần 14 — Multimodal RAG + Project
- Multimodal RAG: index cả ảnh + text (embedding chung không gian)
- Trích xuất dữ liệu từ ảnh tài liệu (OCR + VLM) — **rất thực tế cho doanh nghiệp**
- Fine-tune VLM nhẹ bằng LoRA (khái niệm + demo Colab, nếu có sức)

---

## 📚 Tài nguyên

| Tài nguyên | Nội dung | Link |
|---|---|---|
| **HF Community CV Course — chương VLM** | Giới thiệu VLM, learning strategies | [hf.co/learn/computer-vision-course/en/unit4/multimodal-models/vlm-intro](https://huggingface.co/learn/computer-vision-course/en/unit4/multimodal-models/vlm-intro) |
| **Intro to Visual-Language Modeling** (notebook hands-on) | ViT → CLIP → VLM, tự code | [jpoberhauser.github.io/vision_language_course](https://jpoberhauser.github.io/vision_language_course/) |
| **OpenCV VLM Bootcamp** | Thực hành Qwen2.5-VL, caption, detect | [opencv.org/university/vision-language-model-bootcamp](https://opencv.org/university/vision-language-model-bootcamp/) |
| **Hugging Face SmolVLM course** | VLM nhỏ, chạy local/CPU | [hf.co/learn/smol-course/en/unit3/1](https://huggingface.co/learn/smol-course/en/unit3/1) |
| **nanoMFM** (EPFL) | VLM từ scratch (nâng cao) | [nano-mfm.epfl.ch](https://nano-mfm.epfl.ch/) |

---

## 🛠️ Project GĐ4 — "Trợ lý ảnh" (2 lựa chọn, chọn 1)

**Lựa chọn A — Hỏi đáp ảnh (VQA):** Upload ảnh → đặt câu hỏi → VLM trả lời
**Lựa chọn B — Trích xuất tài liệu ảnh:** Upload ảnh hóa đơn/biểu mẫu → trích JSON có cấu trúc (thực tế hơn cho sản phẩm)

**Yêu cầu chung:**
1. Chạy local bằng `SmolVLM2-2.2B` (CPU OK) hoặc `Qwen2.5-VL-3B`
2. Có UI Gradio upload ảnh
3. So sánh chất lượng 2 model khác nhau (bảng đánh giá tay 20 ảnh)
4. Deploy lên HF Spaces (ZeroGPU hoặc CPU)

**Tiêu chí hoàn thành (DoD):**
- [ ] App chạy được với ảnh thật, latency chấp nhận được (< 15s/ảnh với CPU)
- [ ] Bảng so sánh 2 model có ví dụ đúng/sai cụ thể
- [ ] Trả lời được: "VLM hoạt động khác gì so với LLM + OCR?"
- [ ] Demo URL hoạt động

---

## ⚠️ Lỗi thường gặp
- Chạy model VLM 7B+ trên CPU → quá chậm. Dùng model 2–3B quantized
- Quên resize ảnh → tốn VRAM/RAM vô ích
- Kỳ vọng VLM đếm chính xác đối tượng / đọc chữ nhỏ → giới hạn thật sự, cần biết để thiết kế sản phẩm phù hợp

---

## 🔗 Chuyển tiếp
Xong GĐ4 → bạn có đủ "vũ khí" (LLM + RAG + VLM). GĐ5 ([05-AGENTS-PRODUCTION.md](05-AGENTS-PRODUCTION.md)) ghép tất cả thành **sản phẩm**.
