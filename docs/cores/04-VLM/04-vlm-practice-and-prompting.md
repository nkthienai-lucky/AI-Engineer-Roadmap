# Thực hành VLM: Inference & Prompt (Tuần 13)

> **Mục tiêu:** Biết dùng VLM qua transformers/API cho 4 task phổ biến — captioning, VQA, OCR, object grounding — với prompt mẫu cho từng task; biết cách đặt câu hỏi hiệu quả với ảnh và đặt kỳ vọng hợp lý khi thiết kế sản phẩm.
> Nguồn mở rộng từ: [04-VLM](README.md)

---

## Dùng VLM qua transformers

Mọi VLM modern dùng chung interface `image-text-to-text` của HuggingFace Transformers — nạp model, ghép ảnh + prompt, sinh text. Ví dụ với **SmolVLM2-2.2B** (chạy được CPU):

```python
from transformers import AutoProcessor, AutoModelForImageTextToText
from PIL import Image

model_id = "HuggingFaceTB/SmolVLM2-2.2B-Instruct"
processor = AutoProcessor.from_pretrained(model_id)
model = AutoModelForImageTextToText.from_pretrained(model_id, torch_dtype="auto")

image = Image.open("photo.jpg")
messages = [
    {"role": "user", "content": [
        {"type": "image"}, {"type": "text", "text": "Mô tả bức ảnh này bằng 2 câu."},
    ]},
]
prompt = processor.apply_chat_template(messages, add_generation_prompt=True)
inputs = processor(text=prompt, images=[image], return_tensors="pt")

out = model.generate(**inputs, max_new_tokens=200)
print(processor.batch_decode(out, skip_special_tokens=True)[0])
```

Với API (ví dụ Qwen2.5-VL qua provider hỗ trợ OpenAI-compatible vision): gửi `messages` gồm `image_url` + `text` tương tự — cấu trúc chat template giống nhau nên đổi model chỉ cần đổi `model_id` và cách nạp ảnh.

## Prompt cho từng task

### 1. Captioning (mô tả ảnh)

```
Mô tả bức ảnh này bằng 2–3 câu, tập trung vào đối tượng chính và bối cảnh.
```

Kèm **độ dài** ("2–3 câu") và **trọng tâm** ("đối tượng chính") để tránh caption dài dòng lan man. Dùng cho: alt-text tự động, mô tả kho ảnh, metadata cho RAG ảnh.

### 2. VQA (hỏi đáp ảnh)

```
Dựa vào bức ảnh, trả lời câu hỏi sau ngắn gọn. Nếu không thấy rõ, hãy nói "không xác định được".
Câu hỏi: Trong ảnh có bao nhiêu người và họ đang làm gì?
```

Mẹo: thêm câu "nếu không thấy rõ → nói không xác định được" giúp **giảm hallucination** rõ rệt — model chấp nhận "không biết" thay vì bịa.

### 3. OCR (đọc chữ trong ảnh)

```
Trích xuất toàn bộ văn bản nhìn thấy trong ảnh, giữ nguyên xuống dòng và số liệu.
Chỉ ghi lại đúng những chữ thực sự đọc được, không suy đoán.
```

Hai câu lệnh then chốt: "giữ nguyên xuống dòng/số liệu" (hóa đơn cần số đúng) và "không suy đoán" (chống bịa). Với chữ nhỏ: crop vùng chứa chữ → phóng to → mới cho model đọc (xem file 03 — giới hạn đọc chữ nhỏ).

### 4. Object grounding (định vị vật thể)

```
Tìm chiếc cốc màu đỏ trong ảnh. Xuất tọa độ bounding box theo định dạng [x1, y1, x2, y2]
(tọa độ pixel tuyệt đối của ảnh).
```

Grounding yêu cầu model xuất **tọa độ** thay vì diễn đạt mồm — Qwen2.5-VL hỗ trợ tự nhiên (định dạng `<|box_start|>...`); model không có grounding gốc thì hay sai tọa độ, cần kiểm tra lại bằng detector. Dùng cho: khoanh vùng sản phẩm, đánh dấu lỗi trong ảnh công nghiệp.

## Prompt cho VLM: hỏi gì với ảnh, format nào hiệu quả

**Nguyên tắc format câu hỏi:**

1. **Một task mỗi lượt:** không hỏi "mô tả ảnh và cho biết có bao nhiêu người và đọc số điện thoại" trong một prompt — tách từng câu, model trả lời từng task tốt hơn hẳn.
2. **Cụ thể > mơ hồ:** "Có mấy chiếc xe buýt?" tốt hơn "Ảnh này có gì?"; "Đọc số tiền trên hóa đơn" tốt hơn "Đọc chữ trong ảnh".
3. **Cho phép từ chối:** luôn thêm biến thể "nếu không thấy/không chắc → nói không xác định được".
4. **Chỉ định format đầu ra:** "trả lời bằng JSON gồm các key…", "danh sách gạch đầu dòng", "1 câu duy nhất" — để output parse được bằng code.

```python
# Prompt yêu cầu JSON — parse được thẳng bằng code cho sản phẩm
prompt = """Trích xuất thông tin từ ảnh hóa đơn, trả về JSON đúng format:
{"ten_cua_hang": "...", "ngay": "YYYY-MM-DD", "tong_tien": number, "don_vi": "VND"}
Nếu thiếu thông tin, dùng giá trị null. Không thêm text ngoài JSON."""
```

## Kỳ vọng hợp lý khi thiết kế sản phẩm

- **VLM đúng ~80–95%** trên câu hỏi dễ (ảnh đẹp, đối tượng rõ, chữ to) — chấp nhận được cho demo/alt-text/gợi ý, **không đủ** cho luồng nghiệp vụ bắt buộc đúng 100% (hóa đơn kế toán → cần OCR chuyên dụng + kiểm tra rule).
- **Latency tỷ lệ thuận với số visual token:** ảnh lớn → resize hợp lý (dài cạnh ~768–1024px thường là sweet spot) để không tốn RAM/VRAM mà vẫn đủ chi tiết.
- **Luôn có đường lui:** cho phép người dùng sửa kết quả (human-in-the-loop), hiển thị ảnh gốc cạnh kết quả, đánh dấu ô "model không chắc".
- **Đo lường trước khi tin:** chuẩn bị 10–20 ảnh test thật (đúng/sai rõ ràng) như yêu cầu trong project GĐ4 — bảng so sánh đó là bằng chứng thay vì cảm nhận.

## Liên kết

- [03 — Kiến trúc VLM](03-vlm-architecture.md) — giải thích tại sao có các giới hạn khi prompt
- [05 — Multimodal RAG & OCR](05-multimodal-rag-ocr.md) — pipeline OCR + VLM cho tài liệu doanh nghiệp
- [06 — Project "Trợ lý ảnh"](06-project-vision-assistant.md) — áp dụng các prompt trên vào app thật
