# Project GĐ2 — "Mini-GPT" (Spec) — Tuần 7

> **Mục tiêu:** Spec project giai đoạn 2 — tự viết transformer bằng PyTorch, train trên dữ liệu nhỏ, sinh text, benchmark KV cache, bonus LoRA Qwen2.5-0.5B. Đây là "tạo khác biệt" giữa kỹ sư AI và người chỉ biết gọi API.
> Nguồn mở rộng từ: [02-LLM-INTERNALS.md](../../../02-LLM-INTERNALS.md)

---

## 1. Bối cảnh

Đầu ra của GĐ2: **Mini-GPT train trên tập dữ liệu riêng + bài viết giải thích kiến trúc**. Tuần 7 = ôn tập + hoàn thành project + viết blog/notebook "Attention explained by code" (tài liệu phỏng vấn giá trị).

## 2. Yêu cầu (5 mục)

1. **Tự viết kiến trúc transformer (attention + MLP block) bằng PyTorch — KHÔNG dùng `nn.Transformer`.**
   → Phải code tay: multi-head causal self-attention, residual, layer norm, MLP (feed-forward), positional embedding/RoPE, head sinh token. Dùng `nn.Linear`, `nn.LayerNorm`, `nn.Embedding` được — cấm là `nn.Transformer*` và `nn.MultiheadAttention`. Mục đích: hiểu từng dòng (file 03–04).
2. **Train trên tập dữ liệu nhỏ** (thơ/truyện tiếng Việt, hoặc Shakespeare theo Karpathy).
   → Gợi ý bắt đầu với Shakespeare (`input.txt` trong nanoGPT) để so được với Karpathy, sau đó thay bằng thơ tiếng Việt (lưu ý: tokenizer byte-level vẫn chạy được tiếng Việt, chỉ tốn token hơn).
3. **Sinh văn bản được sau khi train (dù chất lượng thấp cũng được).**
   → Autoregressive sampling: sinh token → ghép vào chuỗi → sinh tiếp; dùng temperature + top-k để điều khiển độ ngẫu nhiên.
4. **Benchmark: so sánh tốc độ sinh text với/without KV cache.**
   → Đo latency/token cho 2 phiên bản (có cache K,V / tính lại từ đầu), ghi kết quả vào README/notebook — xem memory KV cache ở file 04.
5. *(Bonus)* **Fine-tune `Qwen2.5-0.5B` bằng LoRA trên 1 tập dữ liệu nhỏ (dùng Colab free GPU).**
   → Dùng `peft` + `transformers` như demo ở file 06; mục tiêu chỉ là chạy được pipeline end-to-end.

## 3. Tiêu chí hoàn thành (DoD)

- [ ] Code attention ~50 dòng, giải thích được từng biến
- [ ] Loss giảm dần khi train, sample text "có vẻ giống" dữ liệu huấn luyện
- [ ] Notebook/blog giải thích kiến trúc (đây là tài liệu phỏng vấn giá trị)
- [ ] Trả lời được: "Tại sao attention là O(n²)? KV cache tiết kiệm gì?"

Gợi ý mapping DoD → tài liệu: câu hỏi O(n²) trả lời bằng file [03-self-attention.md](03-self-attention.md); KV cache bằng file [04-transformer-architecture.md](04-transformer-architecture.md).

## 4. Gợi ý cấu trúc code

```
mini-gpt/
├── model.py      # kiến trúc: Attention, MLP, Block, MiniGPT (KHÔNG nn.Transformer)
├── train.py      # load data → tokenize → training loop → loss giảm dần → lưu checkpoint
├── sample.py     # load checkpoint → sinh text (temperature/top-k), benchmark KV cache
├── data/         # input.txt (Shakespeare hoặc thơ tiếng Việt)
└── README.md     # log loss, kết quả benchmark, link blog/notebook
```

Mô tả từng file:

- **`model.py`** (~150 dòng):
  - `CausalSelfAttention`: `W_q, W_k, W_v, W_o`; `scores = QKᵀ/√d_k`; mặt nạ causal (torch.tril); optional `kv_cache` (dict layer → K,V).
  - `MLP`: `Linear(d, 4d) → GELU → Linear(4d, d)`.
  - `Block`: Pre-Norm — `x + attn(ln1(x))`, `x + mlp(ln2(x))`.
  - `MiniGPT`: token embedding + positional embedding + `n_layer` block + final LayerNorm + head sinh token; `forward(idx, targets=None)` trả `(logits, cross_entropy_loss)`.
- **`train.py`**: chia train/val theo tỷ lệ 90/10; optimizer AdamW; lr warmup + cosine (xem file 05); in loss theo mốc; lưu `checkpoint.pt`.
- **`sample.py`**: `model.generate(idx, max_new_tokens, temperature, top_k)` — với KV cache benchmark: đo thời gian sinh N token 2 chế độ, in ra bảng so sánh.

Cấu trúc này khớp với nanoGPT (`model.py`, `train.py`, `sample.py`) để dễ đối chiếu khi đọc codebase gốc.

## 5. Thứ tự làm gợi ý (tuần 7, hoặc tràn từ tuần 6)

1. **Ngày 1–2:** viết `model.py` — bắt đầu từ `CausalSelfAttention` thuần (xem snippet ở [03-self-attention.md](03-self-attention.md)), test forward trên batch ngẫu nhiên; thêm `Block` + `MiniGPT`.
2. **Ngày 3:** viết `train.py` trên Shakespeare nhỏ (hoặc thơ tiếng Việt) — xác nhận loss giảm dần (mục tiêu tham chiếu: mini-GPT Karpathy đạt val loss ~1.7–1.8 trên Shakespeare char-level; con số cụ thể phụ thuộc setup, quan trọng là **xu hướng giảm**).
3. **Ngày 4:** viết `sample.py` — sinh text; thêm nhánh `--no-kv-cache` vs `--kv-cache`, đo latency/token cho 2 chế độ trên cùng 1 prompt.
4. **Ngày 5:** viết blog/notebook "Attention explained by code" — trích code từ `model.py` + công thức + kết quả train/benchmark.
5. **Ngày 6–7 (bonus):** LoRA `Qwen2.5-0.5B` trên Colab free GPU; ôn tập trả lời 2 câu hỏi DoD.

Mẹo debug: train trên 1 batch nhỏ trước — nếu model không overfit được 1 batch (loss → gần 0), kiến trúc có bug; khi overfit 1 batch được rồi mới mở rộng dữ liệu.

## 6. Checklist nghiệm thu (tuần 7)

- [ ] `model.py` chạy được forward + backward trên batch ngẫu nhiên
- [ ] Loss train giảm dần trên dữ liệu thật (log loss minh họa trong README)
- [ ] `sample.py` sinh text "có vẻ giống" dữ liệu huấn luyện (thơ/Shakespeare)
- [ ] Bảng benchmark KV cache: with/without → số liệu + kết luận
- [ ] Notebook/blog "Attention explained by code" hoàn chỉnh (công thức + code + trực quan)
- [ ] Trả lời được 2 câu hỏi DoD (O(n²)? KV cache tiết kiệm gì?)
- [ ] (Bonus) LoRA Qwen2.5-0.5B train được trên Colab free GPU
- [ ] Ôn tập tuần 4–6: tokenization, RoPE, attention, training pipeline (đọc lại các file trong folder này)

## 7. Chuyển tiếp

Xong GĐ2 → bạn hiểu model "nghĩ" thế nào. GĐ3 ([03-RAG.md](../../../03-RAG.md)) giải quyết vấn đề thực tế số 1 của doanh nghiệp: *làm sao để model trả lời đúng dữ liệu riêng của mình*.

## Liên kết

- [03-self-attention.md](03-self-attention.md) — phần attention phải code tay trong `model.py`.
- [04-transformer-architecture.md](04-transformer-architecture.md) — kiến trúc block + KV cache cho benchmark.
- [05-training-finetuning.md](05-training-finetuning.md) — loss, learning rate, batch size cho `train.py`.
- [06-lora-qlora-quantization.md](06-lora-qlora-quantization.md) — bonus LoRA Qwen2.5-0.5B.
