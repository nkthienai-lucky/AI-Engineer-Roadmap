# Project GĐ4 — "Trợ lý ảnh"

> **Mục tiêu:** Spec đầy đủ cho project giai đoạn 4 — app "Trợ lý ảnh" với 2 lựa chọn (A: VQA upload ảnh hỏi đáp; B: trích xuất tài liệu ảnh → JSON cấu trúc), yêu cầu chung, Definition of Done và checklist nghiệm thu.
> Nguồn mở rộng từ: [04-VLM.md](../../../04-VLM.md)

---

## Tổng quan

**Đầu ra:** app VLM chạy local + demo public (HF Spaces). Project này chứng minh bạn **dùng được VLM như một kỹ sư** — không chỉ gọi API mà còn so sánh model, biết giới hạn, deploy được. Đây là mảnh ghép cuối trước khi GĐ5 ([05-AGENTS-PRODUCTION.md](../../../05-AGENTS-PRODUCTION.md)) ghép tất cả thành sản phẩm.

## 2 lựa chọn (chọn 1)

### Lựa chọn A — Hỏi đáp ảnh (VQA)

**Upload ảnh → đặt câu hỏi → VLM trả lời.** Ví dụ: upload ảnh đường phố hỏi "có bao nhiêu xe máy?", upload ảnh tủ lạnh hỏi "có thể nấu món gì từ đồ trong ảnh?". Phù hợp nếu muốn demo tổng quát, dễ mở rộng thành chatbot đa lượt với ảnh.

### Lựa chọn B — Trích xuất tài liệu ảnh

**Upload ảnh hóa đơn/biểu mẫu → trích JSON có cấu trúc (thực tế hơn cho sản phẩm).** Ví dụ: ảnh hóa đơn điện → JSON `{nha_cung_cap, ngay, tong_tien, chi_tiet}`; ảnh CCCD → JSON `{ho_ten, ngay_sinh, so_cccd}`. Phù hợp nếu muốn portfolio nhắm doanh nghiệp — đây là use case nhà tuyển dụng hiểu ngay giá trị.

> Gợi ý kiến trúc cho B: ảnh → (OCR tùy chọn) → prompt trích JSON → validate bằng pydantic → hiển thị JSON + ảnh gốc cạnh nhau cho người kiểm tra (xem thêm [05 — Multimodal RAG & OCR](05-multimodal-rag-ocr.md)).

## Yêu cầu chung (giữ nguyên từ nguồn)

1. Chạy local bằng `SmolVLM2-2.2B` (CPU OK) hoặc `Qwen2.5-VL-3B`
2. Có UI Gradio upload ảnh
3. So sánh chất lượng 2 model khác nhau (bảng đánh giá tay 20 ảnh)
4. Deploy lên HF Spaces (ZeroGPU hoặc CPU)

### Gợi ý triển khai từng yêu cầu

1. **Model local:** dùng `transformers` như ví dụ ở [04 — Thực hành & Prompt](04-vlm-practice-and-prompting.md). Ưu tiên `SmolVLM2-2.2B-Instruct` nếu chạy CPU; `Qwen2.5-VL-3B` nếu có GPU nhỏ. Quantize 4-bit (bitsandbytes/llama.cpp) nếu RAM hạn chế.
2. **UI Gradio:** `gr.Image` + `gr.Textbox` (A) hoặc `gr.Image` + `gr.JSON`/`gr.Code` (B); thêm nút "Chạy", hiển thị thời gian xử lý (đo cho DoD latency).
3. **So sánh 2 model:** chọn cặp (SmolVLM2-2.2B + Qwen2.5-VL-3B), chạy cùng 20 ảnh test (5 ảnh dễ / 10 trung bình / 5 khó — cố tình có ảnh chữ nhỏ, nhiều đối tượng để lộ giới hạn), chấm tay đúng/sai/một phần, ghi **ví dụ đúng và sai cụ thể** vào bảng.
4. **Deploy HF Spaces:** repo Space chứa `app.py` + `requirements.txt`; CPU Space chạy được model 2–3B quantized (chậm nhưng OK — giữ latency expectation), ZeroGPU thì dùng `spaces.GPU` decorator.

## Tiêu chí hoàn thành (DoD — giữ nguyên từ nguồn)

- [ ] App chạy được với ảnh thật, latency chấp nhận được (**< 15s/ảnh với CPU**)
- [ ] Bảng so sánh 2 model có **ví dụ đúng/sai cụ thể**
- [ ] Trả lời được: **"VLM hoạt động khác gì so với LLM + OCR?"**
- [ ] **Demo URL hoạt động**

### Chuẩn bị sẵn câu trả lời cho DoD thứ 3

"LLM + OCR" là **hai bước rời**: OCR biến ảnh → text (mất bố cục, mất ngữ cảnh thị giác), rồi LLM đọc text đó. VLM là **một mô hình thống nhất**: visual token của ảnh đi thẳng vào LLM, nên nó giữ được ngữ cảnh thị liệu — biết chữ nằm ở ô 'Tổng cộng', biết người trong ảnh đang làm gì, trả lời câu hỏi mở về ảnh mà pipeline OCR+LLM không biểu diễn được. Đổi lại, LLM+OCR chính xác hơn trên chữ nhỏ cần đúng từng ký tự — nên thực tế hay **ghép cả hai** (xem file 05).

## 20 ảnh test — gợi ý bộ ảnh

Bảng so sánh chỉ có ý nghĩa khi bộ ảnh **phủ đủ loại khó**. Gợi ý phân bổ 20 ảnh:

| Nhóm | Số lượng | Ví dụ | Kiểm tra điều gì |
|---|---|---|---|
| Dễ | 5 | 1 vật thể rõ, phông nền sạch | Cả 2 model phải đúng — baseline |
| Trung bình | 10 | nhiều vật thể, cảnh đời thường, chữ vừa | Khác biệt chất lượng giữa 2 model |
| Khó | 5 | chữ nhỏ (hóa đơn), nhiều đối tượng giống nhau (đếm), ánh sáng kém | Lộ giới hạn — ghi vào bảng đúng/sai/một phần |

Mỗi ảnh ghi: câu hỏi/prompt dùng chung cho cả 2 model → output từng model → chấm tay (Đúng / Sai / Một phần) → ghi **lý do sai** (đếm sai, đọc nhầm chữ, hallucinate…). Bảng này là bằng chứng cho DoD "ví dụ đúng/sai cụ thể".

## Cấu trúc repo gợi ý

```
app.py              # UI Gradio + inference (model A và/hoặc B)
compare.py          # script chạy 2 model trên 20 ảnh → xuất bảng CSV/markdown
requirements.txt    # transformers, gradio, torch, pillow, (peft/bitsandbytes nếu cần)
README.md           # cách cài/chạy, bảng so sánh, latency thật, câu trả lời DoD số 3
images/             # 20 ảnh test (ảnh thật, không dùng ảnh stock đã crop đẹp)
```

## Checklist nghiệm thu

- [ ] `app.py` chạy local bằng ảnh thật, không lỗi (test 3 ảnh: dễ, chữ nhỏ, nhiều đối tượng)
- [ ] Latency đo được và **< 15s/ảnh trên CPU** (ghi số liệu thật vào README)
- [ ] UI Gradio có upload ảnh + hiển thị kết quả (kèm thời gian xử lý)
- [ ] Bảng so sánh SmolVLM2 vs Qwen2.5-VL trên **20 ảnh**, có cột đúng/sai/một phần + ví dụ cụ thể
- [ ] Ghi lại 1–2 case model sai (đếm, chữ nhỏ…) — chứng minh biết giới hạn (file 03)
- [ ] Viết được đoạn trả lời "VLM khác gì LLM + OCR?" (có thể để trong README)
- [ ] Deploy HF Spaces thành công, **demo URL hoạt động** khi mở ở trình duyệt lạ
- [ ] README mô tả: cách cài, cách chạy, model dùng, bảng so sánh, giới hạn đã biết

## Liên kết

- [03 — Kiến trúc VLM](03-vlm-architecture.md) — hiểu giới hạn để chấm bảng so sánh công tâm
- [04 — Thực hành & Prompt](04-vlm-practice-and-prompting.md) — prompt mẫu cho VQA / trích JSON
- [05 — Multimodal RAG & OCR](05-multimodal-rag-ocr.md) — pipeline OCR + VLM cho lựa chọn B
- [07 — Tài nguyên & Lỗi thường gặp](07-resources-and-common-pitfalls.md) — tránh lỗi RAM/latency khi làm project
