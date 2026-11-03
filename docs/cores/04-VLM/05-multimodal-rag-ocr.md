# Multimodal RAG & Trích xuất tài liệu ảnh (Tuần 14)

> **Mục tiêu:** Hiểu multimodal RAG — index cả ảnh + text bằng embedding chung không gian; pipeline trích xuất dữ liệu từ ảnh tài liệu (OCR + VLM) — cực kỳ thực tế cho doanh nghiệp (hóa đơn, biểu mẫu); và khái niệm fine-tune VLM nhẹ bằng LoRA (kèm demo Colab nếu có sức).
> Nguồn mở rộng từ: [04-VLM](README.md)

---

## Multimodal RAG: index cả ảnh + text

RAG thông thường chỉ index văn bản. **Multimodal RAG** mở rộng để index cả **ảnh** (biểu đồ, sơ đồ, ảnh sản phẩm, trang scan) vào chung một index — để câu hỏi "biểu đồ nào cho thấy doanh thu quý 3?" tìm ra được **ảnh biểu đồ**, không chỉ đoạn text nhắc tới nó.

Bí quyết: dùng **embedding chung không gian** — chính là mô hình kiểu CLIP (file 02). Ảnh và text được nhúng thành vector **cùng chiều, cùng không gian**, nên so sánh trực tiếp bằng cosine similarity:

```python
from transformers import CLIPModel, CLIPProcessor
import torch.nn.functional as F

model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32")
proc = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")

# Index: mỗi ảnh + mỗi đoạn text → 1 vector cùng chiều 512
img_vec  = F.normalize(model.get_image_features(**proc(images=[img], return_tensors="pt")), dim=-1)
txt_vec  = F.normalize(model.get_text_features(**proc(text=["doanh thu quý 3"], return_tensors="pt")), dim=-1)

# Truy vấn text → tìm cả text lẫn ảnh gần nhất trong cùng index
```

Thực hành pipeline:

1. **Ingestion:** tách tài liệu thành chunk text + ảnh (figure, bảng scan) — mỗi phần một record.
2. **Embedding:** text đi qua text encoder, ảnh đi qua image encoder của **cùng một model** (hoặc dùng model unified như ColPali/ColQwen nếu nâng cao).
3. **Lưu vào vector DB** (pgvector/Qdrant — xem GĐ3) với metadata: nguồn, trang, loại (text/ảnh).
4. **Retrieval:** truy vấn text → trả về cả chunk text lẫn ảnh liên quan; đưa cả hai vào LLM/VLM để sinh câu trả lời.

Lưu ý: ảnh thường **không tự giải thích được** — nên lưu kèm caption/mô tả (sinh bằng VLM) để tăng khả năng tìm thấy; hoặc index cả hai (vector ảnh + vector caption).

## Trích xuất dữ liệu từ ảnh tài liệu (OCR + VLM) — rất thực tế cho doanh nghiệp

Hóa đơn, phiếu thu, hợp đồng scan, biểu mẫu hải quan… đều là **ảnh** nhưng chứa dữ liệu **cấu trúc** mà doanh nghiệp cần đưa vào hệ thống (kế toán, ERP, CRM). Đây là use case "hái ra tiền" nhất của VLM vì:

- **Volume lớn, lặp lại:** hàng nghìn hóa đơn/tháng, nhập tay tốn chi phí và sai sót.
- **Ảnh chụp thực tế:** lệch, mờ, chụp bằng điện thoại — OCR truyền thống hay hỏng, VLM hiểu ngữ cảnh bù lại.
- **Đầu ra là JSON:** thẳng vào hệ thống, không cần người đọc lại.

Pipeline kết hợp OCR + VLM (kết hợp đúng điểm mạnh từng bên — xem giới hạn ở file 03):

```python
# 1. OCR lấy chữ chính xác (không bịa) — Tesseract / PaddleOCR
text_ocr = pytesseract.image_to_string(invoice_img)

# 2. VLM đọc ngữ cảnh + sinh JSON có cấu trúc
prompt = f"""Ảnh hóa đơn kèm OCR text (có thể sai vài ký tự):
{text_ocr}
Trích xuất JSON: {{"nha_cung_cap", "ma_so_thue", "ngay", "tong_tien", "don_vi"}}
Chỉ dùng thông tin thấy trong ảnh/OCR, thiếu thì null."""

# 3. Validate JSON bằng schema (pydantic) → thiếu/sai thì đưa người duyệt
```

Tại sao kết hợp cả hai: OCR cho **chữ chính xác từng ký tự** (số tài khoản, mã số thuế), VLM cho **hiểu bố cục và ngữ cảnh** ("ô Tổng cộng" là tổng tiền — không phải số điện thoại). Dùng riêng OCR thì mất ngữ cảnh; dùng riêng VLM thì hay bịa số liệu — ghép lại thì mạnh.

## Fine-tune VLM nhẹ bằng LoRA

**Khái niệm:** thay vì train lại cả VLM (tỉ tham số — không khả thi với máy cá nhân), **LoRA** (Low-Rank Adaptation) đóng băng toàn bộ model gốc, chỉ train hai ma trận nhỏ (rank r) cắm vào các lớp attention của LLM/projector. (Chi tiết LoRA/QLoRA xem GĐ2.) Với VLM:

- **Thường chỉ cần fine-tune projector + LoRA trên LLM** — vision encoder giữ nguyên.
- **Dữ liệu:** vài nghìn cặp (ảnh, câu hỏi, câu trả lời) theo đúng domain của bạn (hóa đơn Việt Nam, ảnh y tế…).
- **Kết quả:** model 2–3B + LoRA domain-specific có thể vượt model 7B dùng chung trên đúng loại tài liệu của bạn, vẫn chạy CPU/GPU nhỏ.

```python
from peft import LoraConfig, get_peft_model

lora_cfg = LoraConfig(
    r=16, lora_alpha=32, target_modules=["q_proj", "v_proj"],  # chèn vào attention của LLM
    lora_dropout=0.05, task_type="CAUSAL_LM",
)
model = get_peft_model(vlm_model, lora_cfg)
model.print_trainable_parameters()  # chỉ ~1–2% tham số được train
```

**Demo Colab (nếu có sức):** dùng notebook trong tài nguyên "Intro to Visual-Language Modeling" (file 07) để tự code ViT → CLIP → VLM, rồi thử fine-tune LoRA trên bộ dữ liệu ảnh–caption nhỏ. Nếu thời gian hạn chế, nắm **khái niệm + đoạn code peft ở trên** là đủ cho mục tiêu giai đoạn này — phần thực hành sâu dành cho người muốn đi theo hướng model.

## Liên kết

- [02 — CLIP & Visual Tokens](02-clip-and-visual-tokens.md) — nền tảng embedding chung không gian
- [04 — Thực hành & Prompt](04-vlm-practice-and-prompting.md) — prompt trích xuất JSON cho tài liệu
- [06 — Project "Trợ lý ảnh"](06-project-vision-assistant.md) — lựa chọn B: trích xuất tài liệu ảnh → JSON
