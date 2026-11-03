# Positional Encoding & RoPE — Tuần 4

> **Mục tiêu:** Hiểu tại sao transformer bắt buộc cần positional encoding (attention bất biến với thứ tự), phân biệt absolute / sinusoidal / learned, và hiểu intuition của RoPE (xoay vector theo vị trí, ý tưởng tương đối).
> Nguồn mở rộng từ: [02-LLM-INTERNALS](README.md)

---

## 1. Tại sao transformer cần positional encoding

### Attention bất biến với thứ tự (permutation invariant)

Self-attention (xem file 03) chỉ dùng tích vô hướng giữa các vector token — phép toán này **không biết token nào đứng trước, token nào đứng sau**. Nếu hoán đổi vị trí các token trong câu, output của attention (sau khi hoán đổi tương ứng) **giống hệt nhau**:

```python
import torch
import torch.nn.functional as F

X = torch.randn(4, 8)                       # 4 token, 8 chiều
Wq = torch.randn(8, 8)
Q = X @ Wq                                  # "query"

attn = F.softmax(Q @ Q.T / 8**0.5, dim=-1)  # ma trận attention
out = attn @ X

# Hoán đổi token 0 và 1 → out chỉ bị hoán đổi theo, không đổi giá trị
perm = [1, 0, 2, 3]
Xp = X[perm]
Qp = Xp @ Wq
outp = F.softmax(Qp @ Qp.T / 8**0.5, dim=-1) @ Xp
# outp ≈ out[perm] — không có thông tin "token nào đứng trước"
```

Đây là khác biệt lớn với CNN (vị trí = thứ tự trong cửa sổ trượt) và RNN (duyệt tuần tự). Transformer xử lý **cả câu cùng lúc** → nó mù vị trí trừ khi ta **tiêm** thông tin vị trí vào.

Hệ quả: không có positional encoding thì "mèo đuổi chó" và "chó đuổi mèo" **giống hệt nhau** về mặt toán học — model không thể phân biệt ai đuổi ai.

### Cách tiêm vị trí

Chuẩn: `input = token_embedding + positional_encoding` (cộng theo từng vị trí). Ngoài ra có cách tiêm qua attention score (RoPE, ALiBi) — mục 3.

## 2. Các kiểu positional encoding

### Absolute (tuyệt đối)

Mỗi vị trí `pos` (0, 1, 2, ...) được gắn **một vector cố định hoặc học được**. Token ở vị trí 5 luôn nhận đúng vector vị trí 5, bất kể câu dài ngắn.

**Hạn chế:** khái quát kém khi gặp chuỗi dài hơn lúc train (vị trí 1024 nếu chưa từng thấy), và không diễn đạt trực tiếp "khoảng cách giữa 2 token" — model phải tự học khoảng cách gián tiếp.

### Sinusoidal (Transformer gốc — "Attention Is All You Need")

Vị trí được mã hóa bằng hàm sin/cos ở nhiều tần số:

$$
PE_{(pos, 2i)} = \sin\!\left(\frac{pos}{10000^{2i/d}}\right), \qquad
PE_{(pos, 2i+1)} = \cos\!\left(\frac{pos}{10000^{2i/d}}\right)
$$

```python
import torch, math

def sinusoidal_encoding(max_len, d_model):
    pe = torch.zeros(max_len, d_model)
    pos = torch.arange(max_len).unsqueeze(1).float()
    div = torch.exp(torch.arange(0, d_model, 2).float() * (-math.log(10000.0) / d_model))
    pe[:, 0::2] = torch.sin(pos * div)   # chiều chẵn: sin
    pe[:, 1::2] = torch.cos(pos * div)   # chiều lẻ: cos
    return pe
```

**Intuition:** mỗi cặp (sin, cos) là một "bánh răng" quay với tốc độ khác nhau — tần số cao (i nhỏ) đổi nhanh giữa các vị trí liền kề, tần số thấp đổi chậm. Nhờ tính chất `PE(pos+k)` là hàm tuyến tính (xoay) của `PE(pos)`, khoảng cách tương đối giữa 2 vị trí **có thể biểu diễn được**. Ưu điểm: khái quát được (extrapolate) ngoài độ dài đã train tốt hơn learned.

### Learned (học được)

Dùng `nn.Embedding(max_len, d_model)` — vector vị trí là **tham số học** giống token embedding (BERT, GPT-2 dùng cách này). Đơn giản, mạnh trên độ dài train, nhưng yếu khi extrapolate và "tiêu" tham số cho mỗi vị trí.

| Kiểu | Ưu | Nhược | Dùng ở |
|------|----|-------|--------|
| Sinusoidal | không cần học, khái quát khá | cứng, không tối ưu end-to-end | Transformer gốc |
| Learned | linh hoạt, học được | kém với chuỗi dài hơn lúc train | BERT, GPT-2 |
| RoPE | mô tả khoảng cách **tương đối**, hiệu quả dài | cần cache vị trí khi suy luận | LLaMA, Qwen, Mistral, GPT-NeoX |

## 3. RoPE (Rotary Position Embedding)

### Intuition: xoay vector theo vị trí

RoPE (Rotary Position Embedding, Su et al. 2021) không *cộng* vector vị trí vào embedding. Thay vào đó, nó **xoay** vector query và key trong mỗi cặp chiều 2D, **góc xoay tỷ lệ với vị trí** `pos`:

$$
q_{(pos)} = R(pos \cdot \theta)\, q, \qquad R(\phi) = \begin{pmatrix} \cos\phi & -\sin\phi \\ \sin\phi & \cos\phi \end{pmatrix}
$$

Mỗi cặp chiều `(x0, x1)` được coi như một điểm trong mặt phẳng 2D và bị quay góc `pos · θ` (θ khác nhau cho từng cặp — cùng triết lý "bánh răng đa tần số" như sinusoidal).

```python
def rope(pos, x_pair, theta=10000.0, freq_idx=0):
    """Xoay 1 cặp chiều (x0, x1) theo vị trí pos."""
    angle = pos / (theta ** (freq_idx / 2))
    x0, x1 = x_pair
    return (x0 * math.cos(angle) - x1 * math.sin(angle),
            x0 * math.sin(angle) + x1 * math.cos(angle))
```

### Tại sao "xoay" lại cho ý tưởng **tương đối**

Khi attention tính `q·k`, hai vector đã bị xoay `pos_q·θ` và `pos_k·θ`. Tích vô hướng của hai phép xoay chỉ phụ thuộc vào **hiệu góc** `(pos_q − pos_k)·θ` — tức là phụ thuộc vào **khoảng cách giữa 2 token**, không phụ thuộc vị trí tuyệt đối của chúng!

Đây là điểm mạnh cốt lõi: model học được "token cách 3 vị trí" hay "cách 50 vị trí" một cách tự nhiên, và khái quát tốt sang câu dài hơn lúc train. Vị trí chỉ được đưa vào **bên trong attention score**, không ảnh hưởng tới residual stream — nên token embedding vẫn "sạch".

Mặt nạ causal (chỉ cho phép attend về quá khứ) kết hợp với RoPE tạo ra hành vi "hiểu thứ tự" của GPT-style model.

## 4. Tóm tắt

- Attention mù thứ tự → bắt buộc tiêm thông tin vị trí (cộng embedding hoặc xoay trong attention).
- Sinusoidal: sin/cos đa tần số, không học. Learned: embedding vị trí học được (BERT/GPT-2).
- RoPE: xoay q,k theo vị trí; `q·k` chỉ phụ thuộc khoảng cách tương đối → đó là lý do các model hiện đại (LLaMA, Qwen...) đều dùng RoPE.

## Liên kết

- [01-tokenization-bpe-embedding.md](01-tokenization-bpe-embedding.md) — token → embedding, đầu vào trước khi cộng vị trí.
- [03-self-attention.md](03-self-attention.md) — nơi RoPE tác động (lên Q, K trước khi tính attention).
