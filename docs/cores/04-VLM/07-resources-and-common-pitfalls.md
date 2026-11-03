# Tài nguyên & Lỗi thường gặp (GĐ4)

> **Mục tiêu:** Danh sách tài nguyên học VLM chính thức từ roadmap (giữ nguyên toàn bộ link) + 3 lỗi thường gặp được mở rộng chi tiết: nguyên nhân, dấu hiệu nhận biết và cách khắc phục cụ thể khi chạy VLM local.
> Nguồn mở rộng từ: [04-VLM](README.md)

---

## Tài nguyên (giữ nguyên từ nguồn)

| Tài nguyên | Nội dung | Link |
|---|---|---|
| **HF Community CV Course — chương VLM** | Giới thiệu VLM, learning strategies | [hf.co/learn/computer-vision-course/en/unit4/multimodal-models/vlm-intro](https://huggingface.co/learn/computer-vision-course/en/unit4/multimodal-models/vlm-intro) |
| **Intro to Visual-Language Modeling** (notebook hands-on) | ViT → CLIP → VLM, tự code | [jpoberhauser.github.io/vision_language_course](https://jpoberhauser.github.io/vision_language_course/) |
| **OpenCV VLM Bootcamp** | Thực hành Qwen2.5-VL, caption, detect | [opencv.org/university/vision-language-model-bootcamp](https://opencv.org/university/vision-language-model-bootcamp/) |
| **Hugging Face SmolVLM course** | VLM nhỏ, chạy local/CPU | [hf.co/learn/smol-course/en/unit3/1](https://huggingface.co/learn/smol-course/en/unit3/1) |
| **nanoMFM** (EPFL) | VLM từ scratch (nâng cao) | [nano-mfm.epfl.ch](https://nano-mfm.epfl.ch/) |

**Thứ tự học gợi ý:** HF Community CV Course (nắm khái niệm) → SmolVLM course (thực hành model nhẹ — hợp với project GĐ4) → Intro to Visual-Language Modeling notebook (tự code ViT→CLIP→VLM, demo Colab cho LoRA ở file 05) → OpenCV VLM Bootcamp (nâng cao với Qwen2.5-VL) → nanoMFM (chỉ khi muốn build VLM từ scratch — nâng cao).

**Cách dùng tài nguyên theo tuần:** Tuần 12 đọc HF Community CV Course + notebook Intro to Visual-Language Modeling (phần ViT/CLIP); Tuần 13 xong phần VLM của notebook + SmolVLM course unit 3 rồi OpenCV Bootcamp cho Qwen2.5-VL; Tuần 14 quay lại notebook cho phần fine-tune LoRA nếu có sức. nanoMFM là tài liệu tự chọn ngoài lộ trình — chỉ dành cho ai muốn đi sâu hướng nghiên cứu.

## 3 lỗi thường gặp (mở rộng chi tiết)

Ba lỗi dưới đây là thứ khiến người mới **mất hàng giờ debug** hoặc thất vọng với VLM ngay từ project đầu. Mỗi lỗi được trình bày theo cùng format: dấu hiệu nhận biết → nguyên nhân → cách khắc phục cụ thể, kèm code mẫu và ma trận quyết định nhanh để tra cứu khi làm project GĐ4.

### 1. Chạy VLM 7B+ trên CPU → quá chậm

**Dấu hiệu:** mỗi câu trả lời mất 2–5 phút, hoặc RAM bị đẩy lên 16–32GB rồi swap đĩa ầm ầm, máy đơ. Nguyên nhân: tham số tỷ lệ với tốc độ — 7B tham số ở fp16 cần ~14GB chỉ để chứa weight, và CPU phải nhân ma trận cho **hàng nghìn token** (visual token chiếm phần lớn) mỗi lượt sinh.

**Cách khắc phục:**

- Dùng model **2–3B quantized**: `SmolVLM2-2.2B` (CPU OK) hoặc `Qwen2.5-VL-3B`; quantize 4-bit/8-bit qua `bitsandbytes` (GPU) hoặc `llama.cpp`/GGUF (CPU) — giảm ~4× RAM và tăng tốc đáng kể.
- Giảm sinh output: `max_new_tokens=128`, và **resize ảnh** (lỗi 2) để giảm số visual token.
- Chấp nhận trade-off: model nhỏ kém hơn model lớn một chút — nhưng project GĐ4 chỉ cần **đúng trên ảnh test** và **latency < 15s/ảnh**, không cần model mạnh nhất.

```python
# Nạp quantized 4-bit (GPU) hoặc dùng torch_dtype=float32 + resize ảnh cho CPU
model = AutoModelForImageTextToText.from_pretrained(
    "HuggingFaceTB/SmolVLM2-2.2B-Instruct", torch_dtype="auto"
)
```

### 2. Quên resize ảnh → tốn RAM/VRAM vô ích

**Dấu hiệu:** ảnh chụp điện thoại 12MP (4000×3000) đưa thẳng vào model → RAM/VRAM tăng vùn vụt, inference chậm hơn nhiều dù model chỉ nhìn ở ~448px. Nguyên nhân: visual token **tỷ lệ với diện tích ảnh** — ảnh càng lớn, số patch/token càng nhiều, attention cost tăng **bình phương** số token; trong khi đó model resize về input size cố định anyway → phần lớn chi tiết bị bỏ phí.

**Cách khắc phục:**

- **Resize trước khi inference**, giữ tỷ lệ khung hình, dài cạnh ~768–1024px là sweet spot cho hầu hết task (đọc chữ nhỏ thì crop vùng cần đọc rồi phóng to riêng).
- Dùng `PIL` chuẩn hóa ngay khi load ảnh — thành thói quen như chuẩn hóa token khi làm text.

```python
from PIL import Image

img = Image.open("photo_4000x3000.jpg")
img.thumbnail((1024, 1024))   # giữ tỷ lệ, dài cạnh ≤ 1024 — giảm ~15× số pixel
```

Với model hỗ trợ **dynamic resolution** (Qwen2.5-VL), vẫn nên resize hợp lý — model tự xử lý tốt hơn với ảnh "vừa đủ" hơn là ảnh 12MP ném thẳng vào.

### 3. Kỳ vọng VLM đếm chính xác đối tượng / đọc chữ nhỏ

**Dấu hiệu:** "có bao nhiêu con chim?" trả lời sai dù ảnh rõ; đọc số hóa đơn bị lệch 1–2 chữ số nhưng model vẫn trả lời **rất tự tin** (hallucination) — nguy hiểm nhất vì sai mà trông như đúng. Nguyên nhân sâu: VLM sinh text theo xác suất dựa trên biểu diễn thống kê của patch (xem chi tiết ở [03 — Kiến trúc VLM](03-vlm-architecture.md)) — nó **kể lại những gì "nghe hợp lý"**, không phải đo lường hay nhận diện ký tự.

**Cách khắc phục — thiết kế sản phẩm phù hợp:**

- **Đếm:** dùng detector (YOLO, OWL-ViT) đếm, VLM chỉ mô tả/kết luận. Không đưa VLM vào luồng đếm quan trọng (kho, kiểm kê).
- **Đọc chữ nhỏ:** pipeline OCR (Tesseract/PaddleOCR) + VLM cho ngữ cảnh; luôn hiển thị **ảnh gốc cạnh kết quả** để người dùng soát; bắt buộc human-in-the-loop với dữ liệu tài chính.
- **Prompt phòng thủ:** thêm "nếu không chắc, trả lời 'không xác định được'" — giảm mạnh hallucination (xem [04 — Thực hành & Prompt](04-vlm-practice-and-prompting.md)).

**Ma trận nhanh:** đếm → detector; chữ nhỏ/số liệu → OCR + validate; câu hỏi mở về ảnh → VLM; nghiệp vụ bắt buộc đúng 100% → không dùng VLM một mình.

## Liên kết

- [03 — Kiến trúc VLM](03-vlm-architecture.md) — lý thuyết đằng sau lỗi số 3
- [04 — Thực hành & Prompt](04-vlm-practice-and-prompting.md) — prompt phòng thủ, kỳ vọng hợp lý
- [06 — Project "Trợ lý ảnh"](06-project-vision-assistant.md) — tránh 3 lỗi trên khi làm project GĐ4
