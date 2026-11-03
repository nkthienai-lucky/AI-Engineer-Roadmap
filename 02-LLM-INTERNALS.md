# Giai đoạn 2 — Bản chất LLM: Transformer & Training (Tuần 4–7)

> **Mục tiêu:** Hiểu sâu transformer — tự code được attention, tự train được GPT nhỏ. Đây là giai đoạn **tạo khác biệt** giữa kỹ sư AI và người chỉ biết gọi API.
> **Đầu ra:** Mini-GPT train trên tập dữ liệu riêng + bài viết giải thích kiến trúc.

---

## 🎓 Nội dung cần học

### Tuần 4 — Tokenization & Embedding
- BPE tokenizer: hoạt động ra sao, tại sao "token ≠ từ"
- Tự viết tokenizer đơn giản (Karpathy video `minbpe`)
- Embedding: vector biểu diễn ngữ nghĩa, cosine similarity
- Positional encoding: tại sao cần, RoPE là gì (đọc khái niệm)

### Tuần 5 — Attention & Transformer
- Self-attention từng bước: Q, K, V, scaled dot-product
- Multi-head attention, residual connection, layer norm
- Kiến trúc decoder-only (GPT) vs encoder-decoder (T5)
- KV cache: tại sao sinh token nhanh hơn khi cache

### Tuần 6 — Training & Fine-tuning
- Pretraining vs fine-tuning vs RLHF/DPO (khái niệm)
- Loss function (cross-entropy), learning rate, batch size
- **LoRA / QLoRA**: fine-tune model lớn với GPU nhỏ (khái niệm + demo)
- Quantization: INT8/INT4 (GGUF, GPTQ, AWQ) — tại sao model 3B chạy được trên CPU

### Tuần 7 — Ôn tập + Project
- Hoàn thành Mini-GPT
- Viết blog/notebook giải thích: "Attention explained by code"

---

## 📚 Tài nguyên (học theo thứ tự này)

| # | Tài nguyên | Nội dung | Link |
|---|---|---|---|
| 1 | **Andrej Karpathy — Zero to Hero** (bắt buộc) | Tự code GPT từ đầu: micrograd → makemore → GPT | [youtube playlist](https://www.youtube.com/playlist?list=PLAqhIrjkxbuWI23v9cThsA9GvCAUhRvKZ) |
| 2 | **3Blue1Brown — Deep Learning** (xem trước/song song) | Trực quan hóa attention | [youtube playlist](https://www.youtube.com/playlist?list=PLZHQObOWTQDNU6R1_67000Dx_ZCJB-3pi) |
| 3 | **nanoGPT** | Codebase GPT sạch để đọc | [github.com/karpathy/nanoGPT](https://github.com/karpathy/nanoGPT) |
| 4 | **Attention Is All You Need** (đọc lướt) | Paper gốc transformer | [arxiv.org/abs/1706.03762](https://arxiv.org/abs/1706.03762) |
| 5 | **Transformers in Practice** (Sharon Zhou) | Áp dụng thực tế | [deeplearning.ai/courses/transformers-in-practice](https://www.deeplearning.ai/courses/transformers-in-practice) |
| 6 | **nanoMFM** (đọc khi học VLM sau) | GPT + VLM từ scratch | [nano-mfm.epfl.ch](https://nano-mfm.epfl.ch/) |

**Lịch xem Karpathy gợi ý:** tuần 4: `micrograd` + `makemore p1-2`; tuần 5: `makemore p3-5` + `GPT`; tuần 6: đọc `nanoGPT` + fine-tune thử.

---

## 🛠️ Project GĐ2 — "Mini-GPT"

**Yêu cầu:**
1. Tự viết kiến trúc transformer (attention + MLP block) bằng PyTorch — **không dùng `nn.Transformer`**
2. Train trên tập dữ liệu nhỏ (thơ/truyện tiếng Việt, hoặc Shakespeare theo Karpathy)
3. Sinh văn bản được sau khi train (dù chất lượng thấp cũng được)
4. Benchmark: so sánh tốc độ sinh text với/without KV cache
5. *(Bonus)* Fine-tune `Qwen2.5-0.5B` bằng LoRA trên 1 tập dữ liệu nhỏ (dùng Colab free GPU)

**Tiêu chí hoàn thành (DoD):**
- [ ] Code attention ~50 dòng, giải thích được từng biến
- [ ] Loss giảm dần khi train, sample text "có vẻ giống" dữ liệu huấn luyện
- [ ] Notebook/blog giải thích kiến trúc (đây là tài liệu phỏng vấn giá trị)
- [ ] Trả lời được: "Tại sao attention là O(n²)? KV cache tiết kiệm gì?"

---

## ⚠️ Lỗi thường gặp
- Xem video thụ động, không code theo → không hiểu thật sự. **Phải gõ code theo Karpathy.**
- Bỏ qua toán (đạo hàm, softmax) → sau này không debug được
- Đốt giai đoạn để "học RAG cho nhanh" → nền không vững, về sau bị giới hạn

---

## 🔗 Chuyển tiếp
Xong GĐ2 → bạn hiểu model "nghĩ" thế nào. GĐ3 ([03-RAG.md](03-RAG.md)) giải quyết vấn đề thực tế số 1 của doanh nghiệp: *làm sao để model trả lời đúng dữ liệu riêng của mình*.
