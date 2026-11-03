# Prompt Engineering — Kỹ thuật viết prompt từ cơ bản đến structured output

> **Mục tiêu:** Thành thục 4 kỹ thuật prompt cốt lõi — zero-shot, few-shot, chain-of-thought, structured output (JSON mode) — hiểu vai trò system prompt, và viết được prompt theo 3 nguyên tắc: ngắn gọn, có role, có ví dụ output mẫu.
> **Đầu ra:** Chọn được kỹ thuật phù hợp cho từng tác vụ và thấy rõ "trước/sau" khi áp dụng từng kỹ thuật.

> Nguồn mở rộng từ: [01-FOUNDATIONS.md](../../../01-FOUNDATIONS.md)

---

## Vai trò của System Prompt

System prompt là "lệnh nền" gửi trước mọi lượt hội thoại, đặt **tính cách, vai trò, luật chơi** cho model suốt phiên chat. User prompt là câu hỏi từng lượt; system prompt định hình *cách* model trả lời mọi câu hỏi đó.

```python
messages = [
    {"role": "system", "content": "Bạn là trợ lý y khoa, chỉ trả lời ngắn gọn, luôn kèm câu 'Hãy hỏi bác sĩ'."},
    {"role": "user", "content": "Tôi bị đau đầu 3 ngày thì sao?"},
]
```

Tại sao quan trọng: cùng một câu hỏi, đổi system prompt → đổi hoàn toàn phong cách/độ dài/ranh giới câu trả lời. Đây cũng là chỗ để "khóa" format output (ví dụ: luôn trả JSON) mà không phải lặp lại trong mỗi câu hỏi. Trong project chatbot CLI, yêu cầu 3 (chế độ "hệ thống" tùy chỉnh) chính là cho phép user sửa system prompt.

---

## Zero-shot — không ví dụ

Hỏi thẳng, không đưa ví dụ. Model dựa vào kiến thức đã học để làm ngay. **Dùng khi:** tác vụ đơn giản, phổ biến (dịch, tóm tắt, phân loại quen thuộc).

**Trước (prompt mơ hồ):**

> Tóm tắt cái này giúp tôi: [văn bản dài]

**Sau (zero-shot rõ ràng):**

> Tóm tắt đoạn văn sau thành 1 câu tiếng Việt, tối đa 25 từ, không thêm ý kiến cá nhân:
> [văn bản dài]

Nhìn hai ví dụ trên sẽ thấy: zero-shot "không ví dụ" **không có nghĩa là cẩu tả** — vẫn cần chỉ rõ đầu ra mong muốn (độ dài, ngôn ngữ, phạm vi).

---

## Few-shot — kèm vài ví dụ

Đưa 2–5 ví dụ mẫu "input → output" ngay trong prompt. Model bắt chước format và "gu" của các ví dụ đó. **Dùng khi:** cần output theo format khó diễn tả bằng lời, hoặc tác vụ riêng của bạn (phân loại phản hồi khách hàng theo nhãn của công ty...).

**Trước (zero-shot, output không ổn định):**

> Phân loại phản hồi sau thành TÍCH_CỰC / TIÊU_CỰC / TRUNG_LẬP: "Giao hàng hơi chậm nhưng đóng gói đẹp"

**Sau (few-shot, output ổn định):**

> Phân loại phản hồi thành TÍCH_CỰC / TIÊU_CỰC / TRUNG_LẬP. Ví dụ:
> Phản hồi: "Sản phẩm tuyệt vời, sẽ mua lại" → TÍCH_CỰC
> Phản hồi: "Hàng lỗi, không ai trả lời" → TIÊU_CỰC
> Phản hồi: "Giao đúng hẹn, sp bình thường" → TRUNG_LẬP
> Phản hồi: "Giao hàng hơi chậm nhưng đóng gói đẹp" →

Càng nhiều ví dụ "dở" bị phân loại đúng, model càng vững ở vùng ranh giới. Lưu ý: ví dụ chiếm token trong context window — 3–5 ví dụ tiêu biểu thường là đủ.

---

## Chain-of-thought (CoT) — ép model suy luận từng bước

Thêm cụm từ "hãy suy luận từng bước" (hoặc vài ví dụ có bước suy luận) khiến model **giải bài trung gian** trước khi trả lời kết quả — giảm lỗi với tác vụ cần tính toán/logic/nhiều điều kiện.

**Trước (sai số học):**

> Cửa hàng có 23 táo, bán 7, nhập thêm 15. Còn bao nhiêu táo?
> → 23 (sai!)

**Sau (CoT):**

> Cửa hàng có 23 táo, bán 7, nhập thêm 15. Hãy suy luận từng bước rồi trả lời.
> → Bước 1: sau khi bán còn 23 − 7 = 16. Bước 2: nhập thêm 15 → 16 + 15 = 31. Đáp án: 31. (đúng)

Hạn chế cần biết: CoT làm output **dài hơn và chậm hơn** (nhiều token hơn). Với câu hỏi đơn giản không cần CoT; với bài toán nhiều bước hoặc code lỗi khó, CoT gần như bắt buộc.

---

## Structured output (JSON mode) — ép output thành dữ liệu máy đọc được

Khi ứng dụng cần **parse** kết quả (lưu DB, chuyển tiếp cho code), text tự do là ác mộng. Giải pháp: yêu cầu output JSON trong prompt (và bật JSON mode của API nếu hỗ trợ — xem [04-local-models-ollama-api.md](04-local-models-ollama-api.md)).

**Trước (output text tự do, khó parse):**

> Trích xuất tên và tuổi trong câu: "Nam, 28 tuổi, là kỹ sư"
> → Câu này nói về Nam, một kỹ sư 28 tuổi.

**Sau (JSON mode + schema trong prompt):**

> Trích xuất thông tin dưới dạng JSON, đúng 2 khóa "name" (string) và "age" (integer). Chỉ trả về JSON, không giải thích.
> Input: "Nam, 28 tuổi, là kỹ sư"
> → `{"name": "Nam", "age": 28}`

```python
import json

text = call_llm(messages, json_mode=True)   # model cam kết output JSON hợp lệ
data = json.loads(text)                      # parse an toàn
print(data["name"], data["age"])
```

Mẹo: kèm **ví dụ JSON mẫu** trong prompt (kết hợp few-shot) sẽ giảm mạnh trường hợp model bịa khóa lạ hoặc trả kèm lời giải thích quanh JSON.

---

## Nguyên tắc viết prompt (áp dụng mọi kỹ thuật)

Ba nguyên tắc từ nguồn gốc, mở rộng thành checklist:

1. **Ngắn gọn** — mỗi từ trong prompt là chi phí token và "nhiễu". Cắt tính từ thừa, gộp yêu cầu trùng, đưa ví dụ thay vì giải thích dài. Prompt ngắn + đúng chỗ còn giúp model không "lạc" giữa đống chỉ dẫn.
2. **Có role** — đặt vai trò trong system prompt ("Bạn là chuyên gia pháp lý") để model chọn đúng phong cách, vốn từ và mức độ thận trọng. Role = ngữ cảnh mảnh nhưng hiệu quả.
3. **Có ví dụ output mẫu** — một ví dụ mẫu đáng giá một đoạn mô tả. Kèm output mẫu (few-shot) hoặc schema JSON (structured output) sẽ "khóa" format chặt hơn mọi lời giải thích.

Checklist trước khi gửi prompt: đã nói rõ **đầu ra mong muốn** (độ dài, format, ngôn ngữ) chưa? Đã gỡ bỏ yêu cầu thừa không? Nếu output cần máy đọc — đã ép JSON chưa?

---

## Liên kết

- [02-genai-concepts-token-context.md](02-genai-concepts-token-context.md) — temperature thấp hợp với JSON mode, token quyết định độ dài few-shot
- [04-local-models-ollama-api.md](04-local-models-ollama-api.md) — bật JSON mode / truyền system prompt khi gọi API
- [05-project-chatbot-cli.md](05-project-chatbot-cli.md) — áp dụng system prompt tùy chỉnh trong project chatbot
- [06-resources-and-common-pitfalls.md](06-resources-and-common-pitfalls.md) — lỗi "prompt dài dòng lan man" và cách tránh
