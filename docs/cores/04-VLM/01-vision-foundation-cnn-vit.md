# Vision Foundation: CNN → ViT (Tuần 12)

> **Mục tiêu:** Hiểu cách CNN "nhìn" ảnh bằng convolution — tại sao nó từng thống trị thị giác máy tính suốt hơn một thập kỷ, và vì sao Vision Transformer (ViT) thay thế: patch embedding (chia ảnh thành patch → token), [CLS] token, cùng bảng so sánh CNN vs ViT để biết khi nào dùng cái nào.
> Nguồn mở rộng từ: [04-VLM.md](../../../04-VLM.md)

---

## CNN hoạt động ra sao

### Convolution — bộ lọc trượt trên ảnh

CNN (Convolutional Neural Network) nhìn ảnh không phải "nguyên cục" mà bằng cách trượt các **bộ lọc nhỏ** (kernel, ví dụ 3×3) trên toàn bộ ảnh. Ở mỗi vị trí, bộ lọc nhân từng pixel với trọng số của nó rồi cộng lại — ra một giá trị duy nhất cho ô đó. Lặp lại trên mọi vị trí ta được một **feature map** (bản đồ đặc trưng).

```python
import torch, torch.nn as nn

img = torch.randn(1, 3, 32, 32)          # 1 ảnh RGB 32×32
conv = nn.Conv2d(in_channels=3, out_channels=16, kernel_size=3, padding=1)
feat = conv(img)
print(feat.shape)  # torch.Size([1, 16, 32, 32]) — 16 feature map
```

Intuition: một bộ lọc có thể học để phản ứng mạnh với **đường ngang**, **mép dọc**, **vệt màu**… Ở tầng đầu CNN phát hiện cạnh/vết; tầng sau ghép cạnh thành mắt, bánh xe; tầng sâu hơn ghép thành "mặt người", "ô tô". Đây gọi là **hierarchical features** — đặc trưng cấp thấp → cấp cao, và chính tính chất này khiến CNN cực kỳ hợp với ảnh.

### Tại sao CNN từng thống trị vision

1. **Tính chia sẻ tham số (weight sharing):** một bộ lọc 3×3 chỉ có ~27 trọng số nhưng quét được cả ảnh — ít tham số hơn hẳn mạng fully-connected, ít overfit, train nhanh hơn.
2. **Tính bất biến vị trí (translation invariance):** con mèo ở góc trái hay góc phải ảnh vẫn kích hoạt cùng bộ lọc "tai mèo" — CNN nhận ra vật thể bất kể vị trí.
3. **Hiệu quả tính toán:** convolution là phép nhân ma trận, GPU xử lý cực nhanh.

Nhờ đó, từ AlexNet (2012) → VGG → ResNet, CNN thống trị mọi bài toán vision: phân loại, phát hiện vật thể, phân đoạn. Nhưng CNN có giới hạn: receptive field (vùng nhìn thấy) tăng dần từng tầng nên khó nắm **quan hệ dài** trong ảnh (ví dụ: đầu người bên trái liên quan chân người bên phải), và kiến trúc thường phải thiết kế riêng cho từng bài toán.

## Chuyển sang ViT (Vision Transformer)

ViT (Vision Transformer, Dosovitskiy et al., 2020) áp dụng đúng kiến trúc Transformer của NLP cho ảnh — không convolution nào cả. Ý tưởng cốt lõi: **biến ảnh thành chuỗi token**, rồi để self-attention tự học mọi quan hệ.

### Patch embedding — chia ảnh thành patch → token

Bước 1: chia ảnh (ví dụ 224×224) thành lưới các **patch** vuông (ví dụ 16×16 → 196 patch). Bước 2: mỗi patch được "duỗi phẳng" (flatten) thành vector rồi chiếu qua một ma trận tuyến tính (linear projection) thành một **token** embedding — giống hệt cách NLP nhúng một từ thành token.

```python
import torch, torch.nn as nn

img = torch.randn(1, 3, 224, 224)              # ảnh RGB 224×224
patch_size = 16
n_patches = (224 // patch_size) ** 2           # = 196 patch

# Chia ảnh thành patch và flatten: (1, 3, 224, 224) → (1, 196, 768)
patches = img.unfold(2, patch_size, patch_size).unfold(3, patch_size, patch_size)
patches = patches.contiguous().view(1, 3, n_patches, patch_size * patch_size)
patches = patches.permute(0, 2, 1, 3).reshape(1, n_patches, 3 * patch_size * patch_size)

proj = nn.Linear(3 * patch_size * patch_size, 768)   # patch → token 768 chiều
tokens = proj(patches)                                # (1, 196, 768) — 196 "token" ảnh
```

Từ đây, 196 token ảnh này được đưa vào **cùng một kiến trúc Transformer encoder** mà BERT/GPT dùng cho text: multi-head self-attention cho phép mỗi patch "nhìn thấy" mọi patch khác ngay từ lớp đầu tiên — giải quyết đúng nhược điểm receptive field của CNN.

### [CLS] token

Giống BERT, ViT thêm vào đầu chuỗi một token đặc biệt `[CLS]` (classification token) — một vector học được, không đại diện patch nào cả. Sau khi qua các lớp Transformer, đầu ra tại vị trí `[CLS]` **tổng hợp thông tin cả ảnh** và được dùng cho bài toán phân loại:

```python
cls_token = nn.Parameter(torch.randn(1, 1, 768))          # token học được
tokens = torch.cat([cls_token, tokens], dim=1)            # (1, 197, 768): [CLS] + 196 patch
# → đưa qua Transformer encoder → lấy output[:, 0] làm biểu diễn cả ảnh
```

Ngoài ra ViT còn cộng thêm **positional embedding** (vector học được cho từng vị trí patch) vì self-attention vốn không biết thứ tự — không có nó, tráo patch sẽ không đổi kết quả.

## So sánh CNN vs ViT

| | CNN | ViT |
|---|---|---|
| Cách "nhìn" | Bộ lọc trượt cục bộ, tích chập | Chia patch → token, self-attention toàn cục |
| Quan hệ dài trong ảnh | Khó (receptive field tăng dần) | Tốt ngay từ lớp đầu (mọi patch thấy nhau) |
| Tính bất biến vị trí | Có sẵn (weight sharing) | Phải học từ positional embedding |
| Dữ liệu nhỏ | Vẫn tốt (inductive bias mạnh) | Dễ overfit — cần pretrain trên dữ liệu lớn |
| Dữ liệu lớn (ImageNet-21k, JFT) | Bị ViT vượt | Càng nhiều dữ liệu càng mạnh |
| Giải thích / mở rộng | Phức tạp khi scale | Đồng nhất với NLP — dùng chung hạ tầng Transformer |

**Điểm mấu chốt:** CNN có "thiên kiến" sẵn cho ảnh (cục bộ, bất biến vị trí) nên học tốt với ít dữ liệu; ViT không có thiên kiến nào — nó học mọi thứ từ dữ liệu, nên khi được pretrain trên kho dữ liệu khổng lồ thì vượt CNN và đặc biệt thuận lợi để **ghép với ngôn ngữ** (mọi thứ đều là token) — đây chính là cầu nối dẫn tới CLIP và VLM ở các file sau.

## Liên kết

- [02 — CLIP & Visual Tokens](02-clip-and-visual-tokens.md) — dùng ViT làm vision encoder, ghép ảnh với text
- [03 — VLM Architecture](03-vlm-architecture.md) — vision encoder + projector + LLM decoder
- [07 — Tài nguyên & Lỗi thường gặp](07-resources-and-common-pitfalls.md) — tài liệu học thêm ViT/CLIP
