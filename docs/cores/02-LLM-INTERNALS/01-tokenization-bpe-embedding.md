# Tokenization (BPE) & Embedding — Tuần 4

> **Mục tiêu:** Hiểu BPE tokenizer hoạt động ra sao, tại sao "token ≠ từ", tự viết được tokenizer đơn giản, và hiểu embedding + cosine similarity là gì.
> Nguồn mở rộng từ: [02-LLM-INTERNALS.md](../../../02-LLM-INTERNALS.md)

---

## 1. BPE tokenizer hoạt động ra sao

### Ý tưởng cốt lõi

BPE (Byte Pair Encoding) bắt đầu từ **byte** (hoặc ký tự) và lặp lại quy tắc: *đếm tất cả các cặp ký tự liền nhau, gộp cặp xuất hiện nhiều nhất thành một token mới*. Lặp lại cho đến khi đạt vocab size mong muốn (ví dụ GPT-2: 50.257 token, GPT-4/o200k: ~200k).

Lý do dùng BPE thay vì tách theo từ:
- Từ vựng mở (từ mới, tên riêng, typo) không làm model "vỡ" — từ lạ được ghép từ các mảnh con.
- Không cần ngôn ngữ-specific tokenizer rule ("huấn luyện viên" vs "huấn_luyện_viên").
- Cân bằng: chuỗi thường (the, và) = 1 token; chuỗi hiếm = nhiều token.

### Ví dụ minh họa cách gộp cặp byte

Giảm vocab ban đầu là các ký tự riêng lẻ, chuỗi huấn luyện:

```
"low low low lower lower newest newest widest widest"
```

Các cặp liền nhau và số lần xuất hiện (đếm tay):

| cặp   | số lần |
|-------|--------|
| `l o` | 3      |
| `o w` | 3      |
| `e s` | 4      |
| `e r` | 2      |
| `w e` | 3      |
| ...   | ...    |

Bước 1: cặp `e s` (4 lần) thắng → gộp thành token `es`. Chuỗi thành `... newest → n e w es t ...`.
Bước 2: đếm lại, cặp `es t` xuất hiện 2 lần, `l o` 3 lần... → chọn `lo`, rồi `low`, rồi `new`, `est`, `lower`...

Cứ thế, token dài dần: `low` ban đầu là 3 token `l`,`o`,`w` → 1 token `low`. **Từ càng phổ biến trong dữ liệu huấn luyện → càng được gộp thành token càng dài (thường là 1 token duy nhất).**

Một demo tối giản bằng Python (chỉ 1 vòng lặp gộp):

```python
def get_stats(ids):
    counts = {}
    for a, b in zip(ids, ids[1:]):
        counts[(a, b)] = counts.get((a, b), 0) + 1
    return counts

def merge(ids, pair, idx):
    new_ids, i = [], 0
    while i < len(ids):
        if i < len(ids) - 1 and (ids[i], ids[i+1]) == pair:
            new_ids.append(idx); i += 2
        else:
            new_ids.append(ids[i]); i += 1
    return new_ids

ids = [ord(c) for c in "lowlowlower"]   # bắt đầu từ byte/char
pair = max(get_stats(ids), key=get_stats(ids).get)  # cặp hay nhất
ids = merge(ids, pair, 256)             # token mới = id 256
```

Đây chính là lõi của video **Karpathy `minbpe`** — phần "tự viết tokenizer" bên dưới.

### Tại sao "token ≠ từ"

- "Tokenization" (tiếng Anh) thường là 1 token; nhưng "Tokenization" viết hoa bất thường có thể là 2–3 token (`Token` + `ization`).
- Tiếng Việt có dấu + không dấu: `"huấn luyện"` có thể tách thành `hu`, `ấn`, ` luy`, `ện` — model **không thấy "từ"**, nó thấy chuỗi token.
- Số học: `123456789` có thể thành `123`, `456`, `789` — đó là tại sao LLM yếu với số lớn / đếm ký tự ("how many r in strawberry" từng làm hỏng nhiều model).

Hệ quả thực tế: **chi phí API tính theo token, context window tính theo token** — 1 từ tiếng Việt ≈ 1.5–3 token, đắt hơn tiếng Anh.

## 2. Tự viết tokenizer đơn giản (Karpathy `minbpe`)

Gợi ý "code theo Karpathy" (video `minbpe` trong playlist Zero to Hero — xem mục Liên kết ở file 08):

1. **Bước 1 — `Tokenizer` cơ bản:** encode văn bản → list id (ban đầu `ord(c)`), decode id → chuỗi. Có `train(text, vocab_size)`: lặp `get_stats` → `merge` như ví dụ trên, lưu lại danh sách `merges` và `vocab`.
2. **Bước 2 — regex split:** chia văn bản thành các mảnh nhỏ (words, số, khoảng trắng) rồi áp BPE **trong từng mảnh** — tránh gộp kiểu `e` + ` ` + `t` xuyên từ (GPT-2/GPT-4 đều làm vậy).
3. **Bước 3 — special tokens:** thêm `<|endoftext|>` với id riêng, không cho merge.
4. **Bước 4 — byte-level:** map 256 byte vào "vocab giả" để mọi byte đều encode được, kể cả emoji/UTF-8 tiếng Việt.

Mục tiêu: sau khi tự viết, bạn hiểu *từng dòng* trong `tiktoken`/`minbpe` thay vì coi tokenizer là hộp đen.

## 3. Embedding — vector biểu diễn ngữ nghĩa

### Embedding là gì

Token id (ví dụ `4213`) vô nghĩa về mặt toán học — nó là nhãn. **Embedding** là bảng tra (lookup table) `E[4213]` trả về một vector thực, ví dụ 768 chiều, được **học trong lúc train** sao cho token có ngữ nghĩa giống nhau thì vector gần nhau.

```python
import torch.nn as nn
emb = nn.Embedding(vocab_size=50257, embedding_dim=768)
token_ids = torch.tensor([4213, 998])
vec = emb(token_ids)   # shape: (2, 768)
```

Nghĩa là: **mỗi token = 1 điểm trong không gian 768 chiều**. "queen" ≈ "king" − "man" + "woman" là hiệu vector học được (Word2Vec-era; LLM hiện đại vẫn giữ ý tưởng này nhưng embedding sâu hơn nhiều lớp).

### Cosine similarity — đo "gần nhau" trong không gian vector

Hai vector cùng hướng thì tương tự, bất kể độ dài:

$$
\text{cosine\_sim}(\mathbf{a}, \mathbf{b}) = \frac{\mathbf{a} \cdot \mathbf{b}}{\|\mathbf{a}\| \, \|\mathbf{b}\|} = \frac{\sum_i a_i b_i}{\sqrt{\sum_i a_i^2}\,\sqrt{\sum_i b_i^2}}
$$

Giá trị từ −1 (đối hướng) đến 1 (cùng hướng), 0 = vuông góc (không liên quan). **Intuition:** cosine đo *góc* giữa hai vector, không đo độ dài — hai câu "Tôi đói" và "Tui cần ăn" có vector độ dài khác nhau nhưng cùng hướng.

```python
import torch.nn.functional as F

a = torch.tensor([1.0, 2.0, 3.0])
b = torch.tensor([2.0, 4.0, 6.0])   # cùng hướng, gấp đôi
c = torch.tensor([-1.0, -2.0, -3.0])

F.cosine_similarity(a, b, dim=0)   # tensor(1.0) — giống hệt về hướng
F.cosine_similarity(a, c, dim=0)   # tensor(-1.0) — đối hướng
```

Ứng dụng ngay: **vector search / RAG** (GĐ3) — so sánh cosine giữa câu hỏi và các chunk tài liệu để tìm đoạn liên quan nhất. Đây là lý do file này là nền cho GĐ3.

## 4. Ôn tập nhanh

- BPE: gộp cặp byte phổ biến nhất, lặp lại → token ≠ từ, số ký tự/token thay đổi tùy chuỗi.
- Tự viết tokenizer = hiểu `tiktoken` + biết tại sao context window "hụt" nhanh với tiếng Việt.
- Embedding: token id → vector học được; cosine similarity đo ngữ nghĩa bằng góc giữa vector.

## Liên kết

- [02-positional-encoding-rope.md](02-positional-encoding-rope.md) — sau khi có embedding, cần thêm "vị trí" cho token.
- [03-self-attention.md](03-self-attention.md) — embedding là đầu vào của attention.
- [08-resources-and-common-pitfalls.md](08-resources-and-common-pitfalls.md) — video `minbpe` trong Karpathy Zero to Hero.
