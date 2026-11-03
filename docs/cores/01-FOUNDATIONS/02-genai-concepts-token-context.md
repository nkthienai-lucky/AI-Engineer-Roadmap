# Tuần 2 — Khái niệm GenAI: Token, Context Window, Sampling & Ranh giới

> **Mục tiêu:** Hiểu bản chất LLM là gì, token là gì (và tại sao token ≠ từ), context window, temperature/top-p ảnh hưởng ra sao đến output; phân biệt 4 loại model (LLM / chat model / embedding model / VLM); nhận diện và đối phó với 3 ranh giới lớn: hallucination, cutoff kiến thức, không có bộ nhớ dài hạn.
> **Đầu ra:** Giải thích được cho người khác "model sinh text như thế nào" và biết khi nào không nên tin model.

> Nguồn mở rộng từ: [01-FOUNDATIONS.md](../../../01-FOUNDATIONS.md)

---

## LLM là gì

LLM (Large Language Model) là mạng nơ-ron khổng lồ được huấn luyện trên lượng text mênh mông với một nhiệm vụ duy nhất: **đoán token tiếp theo**. Khi bạn gõ "Hôm nay trời", model tính xác suất cho mọi token có thể đến sau và chọn một token — rồi lặp lại cho đến khi xong câu.

Hệ quả quan trọng: LLM **không tra cứu dữ liệu, không suy luận theo kiểu lập trình** — nó sinh chuỗi token "có vẻ đúng" dựa trên mẫu đã học. Điều này giải thích cả sức mạnh (viết, tóm tắt, dịch, lập trình) lẫn ranh giới (hallucination — xem cuối file).

```python
# Gọi LLM về bản chất là: gửi danh sách message, nhận về token sinh ra
messages = [{"role": "user", "content": "Định nghĩa LLM trong 1 câu"}]
completion = call_llm(messages)   # model sinh lần lượt từng token
```

---

## Token là gì — token ≠ từ

**Token** là đơn vị nhỏ nhất model xử lý: có thể là một từ (`"hello"`), một phần từ (`"play" + "ing"`), dấu câu, hoặc ký tự. Văn bản được "cắt" thành token bởi bộ tách (tokenizer) trước khi vào model, và output cũng là chuỗi token được ghép lại thành text.

**Token ≠ từ** — ví dụ cụ thể:

| Văn bản | Số từ (ước lượng) | Số token (ước lượng) |
|---|---|---|
| `"Hello, how are you?"` | 4 | ~6 (`Hello`, `,`, ` how`, ` are`, ` you`, `?`) |
| `"Xin chào bạn khỏe không?"` | 5 | ~8–10 (tiếng Việt thường tách nhỏ hơn) |
| `"I love programming"` | 3 | ~4 (`I`, ` love`, ` program`, `ming`) |

Hệ quả thực tế:

- **Giới hạn context, giá API, tốc độ** đều tính bằng **token**, không phải từ. Rule of thumb: 1 token ≈ ¾ từ tiếng Anh; tiếng Việt thường tốn **nhiều token hơn** cho cùng lượng thông tin.
- Từ hiếm/tên riêng bị tách thành nhiều token (`"Kubernetes"` có thể thành 2–3 token), nên model học kém hơn với từ lạ.

```python
# Ước lượng nhanh (không chính xác tuyệt đối):
approx_tokens = len(text) / 4      # tiếng Anh
# Muốn chính xác: tokenizer của model (ví dụ tiktoken với OpenAI model)
```

---

## Context Window

Context window là **tổng số token** mà model có thể "nhìn thấy" trong một lần gọi: gồm cả prompt bạn gửi (system prompt + lịch sử hội thoại + câu hỏi) **và** completion model sinh ra. Ví dụ model "8K context" nghĩa là input + output cộng lại không quá ~8192 token.

Hệ quả:

- Lịch sử chat dài → prompt phình to → vượt context window sẽ bị báo lỗi hoặc bị cắt mất phần đầu.
- Prompt chiếm nhiều token → chỗ cho output bị thu hẹp lại.
- Quản lý context (tóm tắt lịch sử, cắt bớt phần cũ) là kỹ năng cần thiết ngay từ project chatbot CLI.

---

## Temperature / Top-p — hai "núm vặn" điều khiển độ ngẫu nhiên

Khi chọn token tiếp theo, model không chọn xác suất cao nhất một cách máy móc — mà **lấy mẫu** (sampling) từ bảng xác suất. Hai tham số chính kiểm soát việc này:

### Temperature

Điều chỉnh "độ bốc" của bảng xác suất:

| Temperature | Hành vi | Dùng cho |
|---|---|---|
| `0` (hoặc rất thấp) | Gần như luôn chọn token xác suất cao nhất → output **ổn định, lặp lại**, dễ tái lập | Trích xuất dữ liệu, JSON mode, trả lời câu hỏi chính xác |
| `0.7` (mặc định phổ biến) | Cân bằng giữa đúng và tự nhiên | Chatbot, viết văn |
| `1.0`+ | Ít thiên vị → output **đa dạng, sáng tạo**, nhưng dễ lan man/lạc đề | Sáng tác, brainstorm |

### Top-p (nucleus sampling)

Giới hạn "vùng chọn": chỉ lấy mẫu trong nhóm token nhỏ nhất mà tổng xác suất ≥ p. `top_p = 0.9` nghĩa là bỏ qua 10% token "đuôi" hiếm — cắt bớt phương án vô lý mà không làm output đơn điệu.

**Cách dùng thực tế:** chỉnh **một trong hai** là đủ (thường là temperature); để cả hai ở mức "mạnh tay" sẽ khó kiểm soát output.

```python
params = {"temperature": 0.2}          # cần chính xác, deterministic
params = {"temperature": 0.9, "top_p": 0.95}   # cần sáng tạo
```

---

## Phân biệt: LLM / chat model / embedding model / VLM

Bốn tên gọi này hay bị nhầm là một — thực chất chúng là **các kiểu model khác nhau về nhiệm vụ và output**:

| | LLM (text completion) | Chat model | Embedding model | VLM |
|---|---|---|---|---|
| **Nhiệm vụ** | Sinh text tiếp theo cho một chuỗi | Hội thoại qua danh sách message có `role` | Đưa text thành vector số (embedding) | Hiểu **ảnh + text**, trả lời bằng text |
| **Input** | Chuỗi text thô | `[{role, content}, ...]` | Text (hoặc mảng text) | Ảnh + câu hỏi dạng text |
| **Output** | Text (completion) | Text (trả lời assistant) | Vector nhiều chiều (ví dụ 768–3072 số) | Text mô tả/trả lời về ảnh |
| **Ví dụ dùng** | Hoàn thành câu, sinh văn bản | Chatbot, trợ lý ảo | Tìm kiếm ngữ nghĩa, RAG | "Ảnh này có gì?", OCR, mô tả ảnh |
| **Ghi chú** | Ít dùng trực tiếp hơn chat model | Dạng dùng phổ biến nhất hiện nay | **Không** sinh text — chỉ so sánh tương đồng | Thường có tiền tố model kiểu `...-vl` |

Cách nhận biết nhanh: embedding model trả về **danh sách số** thay vì câu chữ; chat model nhận **danh sách message có role** thay vì một chuỗi text trơn.

---

## Ranh giới của LLM: 3 điều model KHÔNG làm được

### 1. Hallucination — "nói bịa" mà cứ như thật

**Vì sao:** model sinh token theo xác suất để tạo văn bản "đáng tin về mặt ngôn ngữ", chứ không tra cứu sự thật. Khi thiếu dữ liệu, nó vẫn phải sinh token → bịa ra nội dung nghe hợp lý (số liệu, trích dẫn, tên người, link...).

**Cách đối phó:**

- Yêu cầu model **dẫn nguồn** hoặc nói rõ "không biết" khi không chắc.
- Với thông tin quan trọng: **kiểm chứng** bằng nguồn thật (RAG — xem GĐ3, hoặc tra cứu tay).
- Dùng temperature thấp cho tác vụ cần chính xác.

### 2. Cutoff kiến thức — không biết sự kiện sau ngày huấn luyện

**Vì sao:** model chỉ học dữ liệu đến một thời điểm (knowledge cutoff). Mọi thứ xảy ra sau đó (tin tức, phiên bản thư viện, giá cả...) model không hề biết — hoặc "bịa" để lấp khoảng trống.

**Cách đối phó:** cung cấp thông tin mới **ngay trong prompt** ("Dựa trên tài liệu sau: ..."), hoặc dùng công cụ tìm kiếm/RAG để nạp dữ liệu thời gian thực.

### 3. Không có bộ nhớ dài hạn

**Vì sao:** mỗi lời gọi API là **độc lập** — model không tự nhớ cuộc trò chuyện trước. "Lịch sử hội thoại" thực chất là do **ứng dụng của bạn** lưu lại và gửi kèm lại trong prompt mỗi lần gọi (đó là lý do context window nhanh đầy).

**Cách đối phó:** tự quản lý history trong code (xem [05-project-chatbot-cli.md](05-project-chatbot-cli.md)); tóm tắt lịch sử cũ khi dài; với trí nhớ dài hạn thật sự cần lưu ngoài (cơ sở dữ liệu, vector store — nội dung GĐ3).

---

## Liên kết

- [03-prompt-engineering.md](03-prompt-engineering.md) — viết prompt tốt để giảm hallucination, tận dụng temperature
- [04-local-models-ollama-api.md](04-local-models-ollama-api.md) — truyền các tham số temperature/top-p khi gọi API thật
- [05-project-chatbot-cli.md](05-project-chatbot-cli.md) — quản lý history (bộ nhớ ngắn hạn) trong code chatbot
