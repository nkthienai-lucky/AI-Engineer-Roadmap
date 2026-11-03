# Kiến trúc VLM & Các model nhẹ đáng biết (Tuần 13)

> **Mục tiêu:** Hiểu kiến trúc VLM hiện đại gồm ba khối — vision encoder + projector + LLM decoder — và luồng dữ liệu qua từng khối; nắm đặc điểm 4 model nhẹ đáng biết (**SmolVLM2** 2.2B, **Qwen2.5-VL** 3B/7B, LLaVA, MiniCPM-V); hiểu **giới hạn thật sự** của VLM (đếm đối tượng, đọc chữ nhỏ, hiểu không gian) và nguyên nhân đằng sau.
> Nguồn mở rộng từ: [04-VLM](README.md)

---

## Kiến trúc VLM: vision encoder + projector + LLM decoder

Mọi VLM hiện đại (LLaVA, Qwen2.5-VL, SmolVLM2…) đều là **ba khối ghép lại**:

```
Ảnh đầu vào
   │
   ▼
┌─────────────────┐   visual tokens    ┌────────────┐   embedding cùng chiều   ┌──────────────┐
│ Vision encoder  │ ─────────────────► │ Projector  │ ───────────────────────► │ LLM decoder  │
│ (ViT/CLIP/SigLIP)│  (196 patch, D_v) │ (MLP/linear│   (N tokens × D_llm)     │ (sinh text)  │
└─────────────────┘                    └────────────┘                          └──────────────┘
                                                                              ▲
Prompt text ("Mô tả ảnh này") ── token hóa ──────────────────────────────────┘
```

1. **Vision encoder** (thường là ViT từ CLIP/SigLIP đã pretrain): chia ảnh thành patch → chuỗi visual token (file 01–02). Encoder này **đông cứng hoặc chỉ fine-tune nhẹ** — phần lớn "kiến thức nhìn" đã có sẵn từ pretrain.
2. **Projector** (thường là 1–2 lớp MLP): chiếu mỗi visual token từ chiều của vision encoder sang **chiều hidden của LLM** — "dịch" ngôn ngữ ảnh sang ngôn ngữ token của LLM. Đơn giản nhưng quyết định chất lượng ghép nối.
3. **LLM decoder** (một mô hình ngôn ngữ tiêu chuẩn, ví dụ Qwen2.5, SmolLM, Llama): nhận chuỗi **[visual tokens] + [text tokens]** và sinh câu trả lời bằng text. Mọi "tư duy" về ngôn ngữ (đọc hiểu, suy luận, trả lời) diễn ra ở đây.

Quá trình huấn luyện điển hình (kiểu LLaVA) gồm 2 giai đoạn:

1. **Pretrain projector:** đóng băng vision encoder + LLM, chỉ train projector trên hàng triệu cặp (ảnh, caption) — dạy projector "dịch" visual token sang embedding mà LLM hiểu được.
2. **Fine-tune instruction:** train projector + LLM (có thể thêm LoRA) trên dữ liệu hội thoại ảnh–text chất lượng cao (VQA, mô tả, lý luận) — dạy model trả lời câu hỏi theo chỉ dẫn thay vì chỉ caption.

Vision encoder thường giữ nguyên (hoặc fine-tune rất nhẹ) — phần "nhìn" đã học xong từ pretrain CLIP/SigLIP.

Khi inference, ta thấy rõ 3 khối nối tiếp nhau:

```python
# Pseudo-code thể hiện luồng dữ liệu qua 3 khối
visual_tokens = vision_encoder(pixel_values)          # (1, 196, D_vision)
visual_embeds = projector(visual_tokens)              # (1, N_tokens, D_llm) — "dịch" sang ngữ cảnh LLM
inputs_embeds = concat(text_embeds_prefix, visual_embeds, text_embeds_suffix)
output_ids = llm.generate(inputs_embeds=inputs_embeds)  # sinh text như LLM thuần
```

## Các model nhẹ đáng biết

| Model | Kích thước | Đặc điểm nổi bật |
|---|---|---|
| **SmolVLM2** | 2.2B | Rất nhẹ, chạy được **CPU**, HuggingFace tự phát triển; dùng SigLIP + SmolLM2, giảm visual token bằng pixel shuffle; hợp demo local/ứng dụng di động |
| **Qwen2.5-VL** | 3B / 7B | Mạnh về **đọc chữ (OCR)**, grounding (trỏ tọa độ bbox), hiểu video; Qwen2.5-VL-3B dùng được trên máy cá nhân, 7B cần GPU |
| **LLaVA** | 7B+ | Kiến trúc "gốc" (CLIP vision encoder + MLP projector + Vicuna/Llama) — cực kỳ phổ biến để **học kiến trúc**, nhiều biến thể cộng đồng |
| **MiniCPM-V** | ~2–8B | Nhỏ nhưng chất lượng cao, nổi tiếng với bản chạy on-device (điện thoại), đọc chữ tốt trong nhóm model nhỏ |

Kinh nghiệm chọn: **demo trên CPU** → SmolVLM2-2.2B; **cần đọc chữ/grounding** → Qwen2.5-VL-3B; **học kiến trúc/đọc paper** → LLaVA; **on-device** → MiniCPM-V. Project GĐ4 (file 06) cho phép chọn SmolVLM2-2.2B (CPU OK) hoặc Qwen2.5-VL-3B.

## Giới hạn thật sự của VLM — và tại sao

### Đếm đối tượng (counting)

**Giới hạn:** hỏi "có bao nhiêu con ngựa trong ảnh?" với 7 con ngựa, VLM thường trả lời sai (5, 6, 8…).

**Tại sao:** LLM sinh text theo xác suất — nó "viết ra" một con số nghe hợp lý chứ không đếm thực sự. Bản thân visual token là biểu diễn thống kê của patch, không có biến đếm riêng. Đếm là bài toán **symbolic**, còn VLM học **pattern liên kết ảnh–text**.

**Thiết kế sản phẩm phù hợp:** dùng detector (YOLO, OWL-ViT) để đếm, VLM chỉ để mô tả/kết luận.

### Đọc chữ nhỏ (OCR)

**Giới hạn:** chữ nhỏ trong ảnh hóa đơn, watermark, chữ mờ → VLM đọc sai hoặc "bịa" nội dung nghe hợp lý (hallucination).

**Tại sao:** ảnh được resize về độ phân giải model (ví dụ 224–448px) — chữ nhỏ biến mất hoặc mờ sau patch embedding; model sinh text "giống chữ" hơn là "đúng chữ". Model như Qwen2.5-VL xử lý tốt hơn nhờ cơ chế dynamic resolution, nhưng vẫn không thay OCR chuyên dụng trên chữ nhỏ/nhiễu.

**Thiết kế sản phẩm phù hợp:** pipeline OCR (Tesseract, PaddleOCR) cho chữ chính xác + VLM cho ngữ cảnh ("hóa đơn này tổng bao nhiêu?"); hoặc crop vùng cần đọc rồi phóng to trước khi cho VLM.

### Hiểu không gian (spatial understanding)

**Giới hạn:** hỏi "vật bên trái bàn có màu gì?", "khoảng cách giữa hai vật" → hay sai quan hệ trái/phải, trên/dưới, kích thước tương đối.

**Tại sao:** patch embedding **mất tọa độ tuyệt đối** (positional embedding chỉ giữ thứ tự tương đối); biểu diễn "trái/phải" trong token là thống kê, không phải tọa độ hình học. Model hiện đại (Qwen2.5-VL có tọa độ bbox) khá hơn nhưng vẫn sai khi quan hệ phức tạp, nhiều vật chồng nhau.

**Thiết kế sản phẩm phù hợp:** yêu cầu model xuất **bbox tọa độ** thay vì diễn đạt không gian mơ hồ; kiểm chứng lại bằng detector; không dùng VLM cho bài toán đo đạc/khoảng cách.

**Tổng kết:** VLM là "người xem ảnh + kể lại" rất giỏi, **không phải** "công cụ đo lường". Hiểu giới hạn trên là chìa khóa để thiết kế sản phẩm không vỡ trận khi gặp case thật — cụ thể cách bù trừ (detector, OCR, prompt phòng thủ) xem [04 — Thực hành & Prompt](04-vlm-practice-and-prompting.md) và [07 — Tài nguyên & Lỗi thường gặp](07-resources-and-common-pitfalls.md).

## Liên kết

- [02 — CLIP & Visual Tokens](02-clip-and-visual-tokens.md) — nguồn gốc visual token đi vào projector
- [04 — Thực hành & Prompt cho VLM](04-vlm-practice-and-prompting.md) — dùng các model trên qua transformers/API
- [06 — Project "Trợ lý ảnh"](06-project-vision-assistant.md) — so sánh SmolVLM2 vs Qwen2.5-VL trên 20 ảnh
