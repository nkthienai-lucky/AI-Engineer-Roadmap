# CLIP & Visual Tokens — Cầu nối Vision ↔ Language

> **Mục tiêu:** Hiểu CLIP huấn luyện contrastive learning ảnh-text thế nào (đối sánh ảnh–mô tả trong không gian chung), dùng CLIP cho zero-shot classification, và khái niệm "visual tokens" — cách biến ảnh thành chuỗi token để ghép thẳng vào LLM.
> Nguồn mở rộng từ: [04-VLM.md](../../../04-VLM.md)

---

## CLIP là gì

CLIP (Contrastive Language–Image Pre-training, OpenAI 2021) gồm **hai encoder**:
- **Image encoder** (thường là ViT hoặc ResNet) → nhúng ảnh thành một vector.
- **Text encoder** (Transformer kiểu BERT) → nhúng mô tả (caption) thành một vector.

Cả hai được huấn luyện **cùng lúc** sao cho vector của ảnh và vector của mô tả đúng về nó nằm **gần nhau trong cùng một không gian embedding chung**, còn ảnh và mô tả sai về nhau thì nằm xa.

## Contrastive learning ảnh-text hoạt động thế nào

Ý tưởng: lấy một batch gồm N cặp (ảnh, caption) đúng. Với N ảnh và N caption trong batch, ta có N cặp đúng (đường chéo) và N²−N cặp sai. Loss (InfoNCE) kéo cặp đúng lại gần, đẩy các cặp sai ra xa:

```python
import torch, torch.nn.functional as F

# Giả sử đã có embedding chuẩn hóa (L2-normalized) từ 2 encoder
img_emb  = F.normalize(image_features, dim=-1)   # (N, D)
text_emb = F.normalize(text_features, dim=-1)    # (N, D)

logits = img_emb @ text_emb.T / 0.07             # ma trận độ tương đồng N×N, temp=0.07
labels = torch.arange(N)                          # cặp đúng nằm trên đường chéo
loss = (F.cross_entropy(logits, labels) + F.cross_entropy(logits.T, labels)) / 2
```

Điểm then chốt: **caption đóng vai trò "nhãn tự nhiên"**. Thay vì huấn luyện trên 1000 lớp ImageNet cố định ("cat", "dog"…), CLIP học trên hàng trăm triệu cặp ảnh–mô tả web — nên nó hiểu cả khái niệm mở ("a photo of a dog wearing sunglasses"). Sau huấn luyện, hai encoder "nói chung một ngôn ngữ": so sánh ảnh với bất kỳ câu nào bằng cosine similarity.

## Zero-shot classification với CLIP

Vì không gian chung, ta phân loại ảnh **không cần huấn luyện lại**: tạo các prompt mô tả từng lớp ("a photo of a {class}"), so sánh embedding ảnh với từng prompt, chọn lớp có similarity cao nhất.

```python
from transformers import CLIPProcessor, CLIPModel
from PIL import Image

model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32")
proc = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")

image = Image.open("dog.jpg")
candidates = ["a photo of a dog", "a photo of a cat", "a photo of a car"]
inputs = proc(text=candidates, images=image, return_tensors="pt", padding=True)

outputs = model(**inputs)
probs = outputs.logits_per_image.softmax(dim=1)   # xác suất từng lớp
print(dict(zip(candidates, probs[0].tolist())))
# {'a photo of a dog': 0.98, 'a photo of a cat': 0.015, 'a photo of a car': 0.005}
```

"Zero-shot" nghĩa là model chưa từng thấy nhãn cụ thể của bộ dữ liệu của bạn — chỉ cần viết prompt mô tả lớp là phân loại được ngay. Đổi lại, độ chính xác thường **thấp hơn fine-tune supervised** trên cùng dữ liệu, và chất lượng phụ thuộc cách viết prompt ("a photo of a {class}" thường tốt hơn chỉ "{class}").

## Visual tokens — cầu nối giữa vision và language

Để ghép ảnh vào LLM (một mô hình chỉ quen **chuỗi token văn bản**), ta cần biến ảnh thành token. Cách trực quan nhất: dùng vision encoder (ViT) chia ảnh thành patch, mỗi patch → một embedding vector — mỗi vector đó là một **visual token**. Chuỗi visual token này được đưa vào LLM **xen kẽ/đè lên vị trí token text** trong prompt:

```
[text: "Mô tả bức ảnh:"] [vis₁] [vis₂] … [vis₁₉₆] [text: "→"]
                                    ↓
                            LLM sinh câu trả lời bằng text
```

```python
# Minh họa (pseudo-code với HuggingFace):
# image_features: (1, 196, 4096) — 196 visual token cùng chiều với embedding của LLM
inputs_embeds = torch.cat([text_prefix_embeds, projected_image_features, text_suffix_embeds], dim=1)
output = llm.generate(inputs_embeds=inputs_embeds, max_new_tokens=100)
```

Hai điểm quan trọng:

1. **Chiều phải khớp:** vector visual token phải được chiếu (project) về đúng chiều hidden của LLM — đây chính là công việc của "projector" trong kiến trúc VLM (file 03).
2. **Số visual token quyết định chi phí:** 1 ảnh = hàng trăm token ngữ cảnh → tốn RAM/latency. Các model nhẹ (SmolVLM2) dùng kỹ thuật **pixel shuffle / pooling** để giảm số visual token, đổi chi tiết lấy tốc độ.

CLIP cho ta bước đầu (ảnh và text cùng không gian), còn visual tokens cho ta bước cuối (ảnh thành token để LLM đọc). Ghép hai ý tưởng này lại là toàn bộ kiến trúc VLM hiện đại.

## Liên kết

- [01 — CNN → ViT](01-vision-foundation-cnn-vit.md) — kiến trúc vision encoder phía sau CLIP
- [03 — VLM Architecture](03-vlm-architecture.md) — cách visual token đi vào LLM qua projector
- [05 — Multimodal RAG & OCR](05-multimodal-rag-ocr.md) — dùng embedding chung không gian để index ảnh + text
