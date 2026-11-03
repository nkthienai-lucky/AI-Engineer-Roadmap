# Self-Attention — Tuần 5

> **Mục tiêu:** Hiểu self-attention từng bước: Query, Key, Value, scaled dot-product attention (công thức + tại sao chia √d_k), tính toán bằng numpy/PyTorch, và tại sao attention là O(n²).
> Nguồn mở rộng từ: [02-LLM-INTERNALS.md](../../../02-LLM-INTERNALS.md)

---

## 1. Query, Key, Value — ví dụ "tra cứu thư viện"

Mỗi token (đã qua embedding + positional encoding) sinh ra **ba vector** bằng ba ma trận học được `W_q, W_k, W_v`:

$$
Q = X W_q,\quad K = X W_k,\quad V = X W_v
$$

Ví dụ trực quan "tra cứu thư viện" với câu *"con mèo đuổi con chuột"* — khi xử lý token **"chuột"**:

- **Query (câu hỏi):** *"xung quanh tôi có từ nào cho biết tôi bị ai đuổi?"* — vector đại diện cho **điều token đang tìm**.
- **Key (nhãn sách trên kệ):** mỗi token khác trưng ra một nhãn — "mèo" có nhãn *"động vật hay đuổi"*, "đuổi" có nhãn *"hành động"*. Vector đại diện cho **điều token tự quảng cáo**.
- **Value (nội dung sách):** khi bạn chọn một cuốn, bạn lấy **nội dung** của nó, không phải cái nhãn. Vector đại diện cho **thông tin thực sự được truyền đi**.

Attention = **tra cứu mềm**: thay vì chọn đúng 1 cuốn sách, token "chuột" đọc *trộn* mọi cuốn, mỗi cuốn tỷ lệ với độ khớp giữa Query của nó với Key của cuốn đó. "mèo" khớp nhiều → nhận nhiều nội dung (Value) của "mèo" → vector mới của "chuột" mang ngữ cảnh "bị mèo đuổi".

## 2. Scaled dot-product attention

### Công thức

$$
\text{Attention}(Q, K, V) = \text{softmax}\!\left(\frac{Q K^\top}{\sqrt{d_k}}\right) V
$$

Từng bước:
1. `QKᵀ`: ma trận điểm khớp — mỗi phần tử `q_i · k_j` = mức token `i` "quan tâm" token `j`.
2. Chia `√d_k` (scaled) — xem bên dưới.
3. `softmax` theo mỗi hàng → trọng số dương, tổng = 1 (phân phối "đọc bao nhiêu % mỗi cuốn").
4. Nhân với `V` → tổng có trọng số các value = vector đầu ra sau contextualize.

### Tại sao chia √d_k

`q·k` là tích vô hướng của 2 vector `d_k` chiều: nếu mỗi thành phần ~ N(0,1) độc lập, phương sai của tích = `d_k` → kỳ vọng `q·k` lớn khi `d_k` lớn (ví dụ 128 → giá trị hàng chục). Softmax với input lớn bão hòa: gradient ≈ 0 (đỉnh hàm softmax đã "phẳng"), attention thành one-hot cứng và **không học được**. Chia `√d_k` đưa phương sai về ~1, softmax ở vùng "mềm", gradient chảy tốt. (Xem thêm "bỏ qua toán → không debug được" ở file 08.)

```python
import torch, math

scores = q @ k.T / math.sqrt(d_k)   # chuẩn hóa phương sai
weights = torch.softmax(scores, dim=-1)
out = weights @ v
```

## 3. Ví dụ tính toán nhỏ bằng PyTorch

Attention thuần (không nn.MultiheadAttention), 4 token, `d_model = 8`, `d_k = 8`:

```python
import torch
import torch.nn.functional as F
import math

torch.manual_seed(0)
X = torch.randn(4, 8)                       # 4 token đã embed + positional

W_q = torch.randn(8, 8, requires_grad=True)
W_k = torch.randn(8, 8, requires_grad=True)
W_v = torch.randn(8, 8, requires_grad=True)

Q, K, V = X @ W_q, X @ W_k, X @ W_v         # mỗi cái shape (4, 8)
d_k = Q.shape[-1]                            # = 8

scores  = Q @ K.T / math.sqrt(d_k)           # (4, 4) — ma trận điểm khớp
weights = F.softmax(scores, dim=-1)          # mỗi hàng tổng = 1
output  = weights @ V                        # (4, 8) — mỗi token = trộn ngữ cảnh

print(weights[3])   # token thứ 4 "đọc" các token trước bao nhiêu %
```

Với numpy chỉ khác `np.softmax` (tự viết: `np.exp(x)/np.exp(x).sum()`). **Bài tập:** thêm mặt nạ causal trên `scores` (che phần tam giác trên bằng `-inf` trước softmax) — đó chính là self-attention của GPT.

## 4. Tại sao attention là O(n²)

Với `n` token, `QKᵀ` tính **tất cả cặp** `(i, j)` → có `n × n` phần tử:

- Chi phí tính toán: `O(n² · d)` (nhân ma trận n×d với d×n).
- Bộ nhớ ma trận attention: `O(n²)` — với `n = 8192`, ma trận 67 triệu phần tử/layer/head.

Hệ quả thực tế: nhân đôi độ dài prompt → **gấp 4 lần** chi phí attention. Đó là lý do context window dài (128k, 1M) cần FlashAttention (tính attention theo block, không materialize ma trận n²) và cần KV cache (file 04). Tuy nhiên khi **sinh** token, mỗi bước chỉ thêm 1 hàng/cột mới → chi phí mỗi token là O(n), cộng dồn thành O(n²) cho cả chuỗi — hiểu đúng hai phát biểu này là câu hỏi phỏng vấn kinh điển ("Tại sao attention là O(n²)?").

Giới hạn này cũng là động lực của các hướng "linear attention" / sparse attention — nhưng chuẩn công nghiệp 2024–2025 vẫn là full attention + flash kernels.

## 5. Tóm tắt

- Q = câu hỏi, K = nhãn, V = nội dung; attention = tra cứu mềm, trộn Value theo độ khớp Q–K.
- `softmax(QKᵀ/√d_k)V`: chia `√d_k` để softmax không bão hòa.
- `O(n²)` vì xét mọi cặp token — gốc rễ của mọi bài toán context dài.

## Liên kết

- [02-positional-encoding-rope.md](02-positional-encoding-rope.md) — RoPE xoay Q, K trước khi tính điểm khớp.
- [04-transformer-architecture.md](04-transformer-architecture.md) — multi-head attention, residual, layer norm, KV cache.
- [07-project-mini-gpt.md](07-project-mini-gpt.md) — tự code lại ví dụ trên trong project Mini-GPT.
