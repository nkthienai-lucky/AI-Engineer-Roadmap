# Kiến trúc Transformer: Multi-Head, Residual, LayerNorm, KV Cache — Tuần 5

> **Mục tiêu:** Hiểu vai trò của multi-head attention, residual connection, layer norm; phân biệt decoder-only (GPT) vs encoder-decoder (T5) và khi nào dùng cái nào; hiểu KV cache — cơ chế hoạt động, tại sao sinh token nhanh hơn, và memory của KV cache.
> Nguồn mở rộng từ: [02-LLM-INTERNALS](README.md)

---

## 1. Ba "phụ kiện" quan trọng quanh attention

### Multi-head attention

Thay vì 1 attention với `d_model` chiều, tách thành `h` head (ví dụ 12 head × 64 chiều = 768), mỗi head học **một kiểu quan hệ khác nhau** — head 1 có thể bắt quan hệ ngữ pháp (đồng ngữ), head 2 bắt đại từ chỉ định, head 3 bắt khoảng cách gần. Các head chạy song song, output được nối lại rồi chiếu qua `W_o`:

```python
# ý tưởng: h head song song
heads = [softmax((X @ W_q[i]) @ (X @ W_k[i]).T / 8) @ (X @ W_v[i]) for i in range(h)]
multihead = torch.cat(heads, dim=-1) @ W_o   # nối + chiếu tuyến tính
```

Nếu 1 head "chết" (attention all-zero), các head khác vẫn hoạt động — redundancy giúp train ổn định.

### Residual connection

Mỗi block: `x = x + Attention(x)` và `x = x + MLP(x)` — tầng mới chỉ học **sửa đổi** (delta) trên luồng dữ liệu gốc, không bắt buộc tái tạo lại mọi thứ. Lợi ích: (1) gradient có đường "cao tốc" ngược về tầng đầu (chống vanishing gradient khi stack vài chục lớp), (2) thông tin gốc không bị mất qua nhiều phép biến đổi phi tuyến.

### Layer Norm

Chuẩn hóa activation về trung bình 0, phương sai 1 **theo từng token** (khác batch norm — chuẩn theo batch), rồi scale + shift bằng 2 tham số học được:

$$
\text{LayerNorm}(x) = \gamma \cdot \frac{x - \mu}{\sqrt{\sigma^2 + \epsilon}} + \beta
$$

Vai trò: giữ các giá trị activation trong vùng "ổn định" để softmax không bão hòa và learning rate không phải chỉnh quá nhỏ. Kiến trúc phổ biến: **Pre-Norm** (LayerNorm *trước* attention/MLP — GPT-2 trở đi) train sâu ổn định hơn Post-Norm gốc.

Một block decoder đầy đủ (chuỗi 4 bước): `LayerNorm → Attention → + residual → LayerNorm → MLP → + residual`.

## 2. Decoder-only (GPT) vs Encoder-decoder (T5)

| | Decoder-only (GPT, LLaMA, Qwen) | Encoder-decoder (T5, BART, mT5) |
|---|---|---|
| Cấu trúc | 1 chuỗi khối, mask causal (chỉ thấy quá khứ) | Encoder đọc toàn bộ input; decoder sinh output |
| Nhiệm vụ | sinh text tiếp theo (next-token) | seq2seq: dịch, tóm tắt, trả lời câu hỏi |
| Prompt | input + output cùng 1 luồng | input qua encoder, output qua decoder |
| Xu thế | thống trị LLM chat hiện đại | mạnh cho nhiệm vụ dịch/tóm tắt chuyên biệt |

**Khi nào dùng cái nào:**
- Dùng **decoder-only** khi xây chatbot/sinh text/agent — đây là chuẩn của GPT-4, Claude, Qwen2.5...; một mô hình, mọi nhiệm vụ đều là "sinh text tiếp theo".
- Dùng **encoder-decoder** khi nhiệm vụ rõ ràng seq2seq (dịch máy, ASR→text, tóm tắt) và muốn encoder "hiểu trọn" input không mask — mT5/mBART vẫn tốt cho tiếng Việt low-resource.
- Về mặt toán, T5 = "đọc hết rồi mới viết"; GPT = "viết tuần tự, mỗi bước chỉ được nhìn ngược lại".

## 3. KV Cache

### Cơ chế hoạt động

Khi sinh token tự hồi quy (autoregressive), bước `t` cần attention với **tất cả** token trước. Không cache thì mỗi bước sinh phải tính lại `K, V` cho toàn bộ chuỗi → tính lại `O(n)` ma trận mỗi token, tổng `O(n²)` phép tính thừa.

KV cache: **lưu lại `K` và `V` của mọi token quá khứ** (theo layer, theo head). Bước sinh mới chỉ cần:
1. Tính `q, k, v` cho **token mới duy nhất**.
2. Nối `k, v` mới vào cache.
3. Attention: query 1 dòng × toàn bộ K trong cache.

```python
# sinh token thứ t (đơn giản hóa)
k_new, v_new = layer.compute_kv(x_new)         # chỉ cho 1 token
cache_k = torch.cat([cache_k, k_new], dim=0)   # nối vào cache
cache_v = torch.cat([cache_v, v_new], dim=0)
out = softmax(q_new @ cache_k.T / sqrt(d_k)) @ cache_v
```

### Tại sao nhanh hơn khi cache

- Không cache: mỗi token sinh phải tính lại K,V cho toàn bộ prompt + chuỗi đã sinh → chi phí mỗi token tăng dần theo độ dài chuỗi.
- Có cache: mỗi token mới chỉ tốn `O(n)` attention với cache (nhân 1 query với n key), **không** tính lại K,V cũ — giảm phép tính ~hệ số bằng độ dài prompt trong prefill, và latency/token ổn định hơn nhiều. Benchmark so sánh with/without KV cache là yêu cầu số 4 của project Mini-GPT (file 07).

### Memory của KV cache

Đánh đổi lấy tốc độ là **bộ nhớ**: mỗi token lưu `2 (K+V) × n_layers × n_heads × d_head` phần tử. Ví dụ model 7B (32 layer, 32 head, 128 dim/head), fp16 (2 byte):

```
2 × 32 × 32 × 128 × 2 byte = 512 KB / token
```

→ context 8k token ≈ **4 GB** chỉ cho KV cache. Đó là tại sao:
- kĩ thuật **GQA (grouped-query attention)** chia sẻ K,V giữa nhiều query head → cache nhỏ đi 4–8× (Qwen2.5, LLaMA-3 dùng GQA).
- quantization KV cache (INT8/FP8) — xem file 06.
- server suy luận (vLLM) cần scheduler quản lý cache theo "paged memory".

## 4. Tóm tắt

- Multi-head: nhiều "góc nhìn" attention song song; Residual: gradient + giữ thông tin gốc; LayerNorm: ổn định activation.
- Decoder-only (GPT) cho sinh text/chat; encoder-decoder (T5) cho seq2seq.
- KV cache = lưu K,V quá khứ → sinh token nhanh, đổi lấy memory tỷ lệ với `n_layers × n_tokens`.

## Liên kết

- [03-self-attention.md](03-self-attention.md) — nền tảng Q,K,V trước khi bàn cache.
- [06-lora-qlora-quantization.md](06-lora-qlora-quantization.md) — quantization KV cache và model.
- [07-project-mini-gpt.md](07-project-mini-gpt.md) — benchmark KV cache trong project.
