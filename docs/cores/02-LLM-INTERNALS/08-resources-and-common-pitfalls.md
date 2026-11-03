# Tài nguyên & Lỗi thường gặp — Tuần 4–7

> **Mục tiêu:** Danh sách tài nguyên học theo đúng thứ tự (Karpathy Zero to Hero BẮT BUỘC kèm lịch xem), và 3 lỗi thường gặp được mở rộng: tại sao xem thụ động không hiểu thật, tại sao bỏ qua toán sẽ không debug được, tại sao đốt giai đoạn bị giới hạn sau này.
> Nguồn mở rộng từ: [02-LLM-INTERNALS.md](../../../02-LLM-INTERNALS.md)

---

## 1. Tài nguyên (học theo thứ tự này)

| # | Tài nguyên | Nội dung | Link |
|---|---|---|---|
| 1 | **Andrej Karpathy — Zero to Hero** (bắt buộc) | Tự code GPT từ đầu: micrograd → makemore → GPT | [youtube playlist](https://www.youtube.com/playlist?list=PLAqhIrjkxbuWI23v9cThsA9GvCAUhRvKZ) |
| 2 | **3Blue1Brown — Deep Learning** (xem trước/song song) | Trực quan hóa attention | [youtube playlist](https://www.youtube.com/playlist?list=PLZHQObOWTQDNU6R1_67000Dx_ZCJB-3pi) |
| 3 | **nanoGPT** | Codebase GPT sạch để đọc | [github.com/karpathy/nanoGPT](https://github.com/karpathy/nanoGPT) |
| 4 | **Attention Is All You Need** (đọc lướt) | Paper gốc transformer | [arxiv.org/abs/1706.03762](https://arxiv.org/abs/1706.03762) |
| 5 | **Transformers in Practice** (Sharon Zhou) | Áp dụng thực tế | [deeplearning.ai/courses/transformers-in-practice](https://www.deeplearning.ai/courses/transformers-in-practice) |
| 6 | **nanoMFM** (đọc khi học VLM sau) | GPT + VLM từ scratch | [nano-mfm.epfl.ch](https://nano-mfm.epfl.ch/) |

### Cách học từng tài nguyên

1. **Karpathy Zero to Hero (BẮT BUỘC):** xem video **và gõ code theo từng dòng**, không copy-paste. Mỗi video là một project hoàn chỉnh: `micrograd` (autograd engine ~100 dòng), `makemore` (MLP → RNN → transformer cho tên), `GPT` (minGPT từ đầu). Sau mỗi video: tự giải thích lại cho "người không biết gì" — không giải thích được là chưa hiểu.
2. **3Blue1Brown:** xem để có **trực quan hình học** (attention là gì trong không gian vector, backprop "chảy" thế nào). Xem trước/song song với Karpathy — video này không thay thế việc code.
3. **nanoGPT:** đọc codebase sau khi tự viết GPT xong — so sánh code của mình với Karpathy, học trick (weight tying, GELU, warmup...). Đây cũng là cấu trúc tham chiếu cho [07-project-mini-gpt.md](07-project-mini-gpt.md).
4. **"Attention Is All You Need":** đọc lướt mục 3.2 (attention) + 3.5 (positional encoding) trước, phần còn lại đọc khi cần tra công thức gốc.
5. **Transformers in Practice (Sharon Zhou):** khóa áp dụng thực tế — học sau khi nắm bản chất để không thành "chỉ biết gọi API".
6. **nanoMFM:** dành cho GĐ4 (VLM) — chưa cần đọc trong GĐ2.

**Lịch xem Karpathy gợi ý:** tuần 4: `micrograd` + `makemore p1-2`; tuần 5: `makemore p3-5` + `GPT`; tuần 6: đọc `nanoGPT` + fine-tune thử.

### Checklist học tập 4 tuần (đối chiếu với lịch trên)

- **Tuần 4:** xem `micrograd` + `makemore p1-2`; đọc [01-tokenization-bpe-embedding.md](01-tokenization-bpe-embedding.md) + [02-positional-encoding-rope.md](02-positional-encoding-rope.md); tự viết được hàm `get_stats`/`merge` của BPE từ trí nhớ; tính cosine similarity bằng PyTorch.
- **Tuần 5:** xem `makemore p3-5` + video `GPT`; đọc [03-self-attention.md](03-self-attention.md) + [04-transformer-architecture.md](04-transformer-architecture.md); tự viết attention ~50 dòng không nhìn tài liệu; giải thích được KV cache cho người khác.
- **Tuần 6:** đọc `nanoGPT` + fine-tune thử; đọc [05-training-finetuning.md](05-training-finetuning.md) + [06-lora-qlora-quantization.md](06-lora-qlora-quantization.md); chạy loss giảm dần trên tập dữ liệu nhỏ; (bonus) LoRA Qwen2.5-0.5B.
- **Tuần 7:** hoàn thành Mini-GPT theo [07-project-mini-gpt.md](07-project-mini-gpt.md); viết blog/notebook "Attention explained by code"; ôn lại toàn bộ folder này.

## 2. 3 lỗi thường gặp (mở rộng)

### Lỗi 1: Xem video thụ động, không code theo → không hiểu thật sự

Xem Karpathy gõ code tạo cảm giác "hiểu" — đó là **illusion of competence**: não bạn nhận ra pattern khi xem, nhưng không tự tái tạo được. Bài kiểm tra đơn giản: đóng video, tự viết lại `get_stats`/`merge` của BPE hoặc ma trận attention — viết không ra nghĩa là chưa hiểu.

**Cách sửa:** *phải gõ code theo Karpathy.* Gõ tay từng dòng (không paste), sau đó **đổi tên biến, thêm print, phá code cố ý** (bỏ `√d_k` xem softmax nổ, bỏ causal mask xem model "nhìn thấy tương lai") — phá được và giải thích được hậu quả mới là hiểu. Mỗi video: code lại từ đầu trong file trống sau 1–2 ngày (spaced recall).

### Lỗi 2: Bỏ qua toán (đạo hàm, softmax) → sau này không debug được

Khi loss thành NaN, gradient bằng 0, hoặc model sinh text lặp vô hạn, "tune thử" mãi không ra — người hiểu toán biết kiểm tra ngay: softmax có input quá lớn không (→ bão hòa, gradient ≡ 0, xem [03-self-attention.md](03-self-attention.md))? Learning rate có quá lớn không (gradient explode)? Đạo hàm của cross-entropy qua softmax rất gọn (`p − onehot`) — hiểu nó thì đọc được mọi implementation.

**Cách sửa:** không cần chứng minh hàn lâm, nhưng **phải nắm**: đạo hàm (chain rule, gradient descent), softmax + cross-entropy (và vì sao 2 cái này ghép đẹp với nhau), kỳ luận/phương sai (để hiểu tại sao chia `√d_k`). Viết tay 1 lần các công thức này vào notebook — Karpathy `micrograd` chính là video dạy đạo hàm tự động bằng code.

### Lỗi 3: Đốt giai đoạn để "học RAG cho nhanh" → nền không vững, về sau bị giới hạn

Học RAG/agent khi chưa hiểu tokenization, attention, context window sẽ cho ra hệ thống "chạy được nhưng không hiểu tại sao hỏng": chunk vỡ token, context đầy mà không biết vì sao, hallucination không truy vết được, tối ưu latency không biết KV cache là gì. Người chỉ biết gọi API bị **giới hạn ở mức độ lỗi của framework** — framework không làm hộ việc chẩn đoán. Khi framework upgrade hoặc cần custom (prompt dài bất thường, model nội bộ, latency khắt khe), người thiếu nền không còn đường lui.

**Cách sửa:** đi đủ GĐ2 (tuần 4–7) trước GĐ3. Mỗi giai đoạn có đầu ra cụ thể (Mini-GPT, blog) làm "cửa ải" — qua cửa ải rồi mới tiến. Chậm mà chắc ở GĐ2 = nhanh hơn tổng thể, vì GĐ3–GĐ5 (RAG, VLM, agents, deployment) đều **dựng trên đúng các khái niệm của folder này**: embedding + cosine (RAG), attention + KV cache (deployment), training pipeline (fine-tune).

### Dấu hiệu nhận biết mình đang mắc lỗi (tự kiểm tra)

| Dấu hiệu | Lỗi tương ứng | Hành động |
|----------|---------------|-----------|
| Xem video xong không viết lại được code | Lỗi 1 (thụ động) | Đóng video, code lại từ đầu |
| Loss NaN / gradient = 0 mà không biết kiểm tra gì | Lỗi 2 (bỏ qua toán) | Viết tay softmax + cross-entropy, xem lại file 03 |
| Làm RAG "chạy được" nhưng không giải thích được vì sao hỏng | Lỗi 3 (đốt giai đoạn) | Quay lại hoàn thành Mini-GPT trước |

## 3. Cách dùng folder này

- File 01–06 là **study note mở rộng** từng gạch đầu dòng của file gốc — dùng như tài liệu đọc trước/sau video, không thay thế việc gõ code.
- Khi xem Karpathy gặp đoạn không hiểu (ví dụ: tại sao chia `√d_k`, RoPE xoay nghĩa là gì) → mở đúng file tương ứng trong folder để đọc giải thích + snippet.
- Khi trả lời phỏng vấn, trả lời bằng ngôn ngữ của mình rồi mới trích công thức/code — học thuộc y hệt là dấu hiệu của lỗi 1 bên trên.
- Giữ DoD ở [07-project-mini-gpt.md](07-project-mini-gpt.md) làm "cửa ải": chưa tick đủ thì chưa chuyển GĐ3 — đó là cơ chế chống lỗi 3 bằng hành động cụ thể, không chỉ bằng ý chí.
- Sau mỗi tuần, viết 3–5 dòng tóm tắt "tuần này hiểu được gì" vào notebook — nếu viết không ra, tuần đó học thụ động (lỗi 1) và nên code lại trước khi xem tiếp.

## Liên kết

Đọc theo đúng thứ tự 01→08:

- Đọc tuần 4–5 (bản chất): [01](01-tokenization-bpe-embedding.md) → [02](02-positional-encoding-rope.md) → [03](03-self-attention.md) → [04](04-transformer-architecture.md).
- Đọc tuần 6 (training): [05](05-training-finetuning.md) → [06](06-lora-qlora-quantization.md).
- Tuần 7 làm project theo [07](07-project-mini-gpt.md), trước khi phỏng vấn thì ôn lại checklist của file 07 + bảng dấu hiệu lỗi ở file này.
- File gốc rút gọn: [02-LLM-INTERNALS.md](../../../02-LLM-INTERNALS.md).
