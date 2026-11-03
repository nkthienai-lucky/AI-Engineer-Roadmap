# Generation, Citation & Chống Hallucination (Tuần 10)

> **Mục tiêu:** Biết sinh câu trả lời từ context kèm **trích dẫn nguồn** (citation format, vì sao bắt buộc), viết prompt chống hallucination ("chỉ trả lời dựa trên context") với mẫu cụ thể, và xử lý đúng khi context không đủ để trả lời.
> Nguồn mở rộng từ: [03-RAG.md](../../../03-RAG.md)

---

## Generation với context

Sau retrieval + rerank (file 04), ta có top-k chunk liên quan nhất. Bước generation ghép chúng vào prompt và yêu cầu LLM trả lời **dựa trên context đó**:

```
[System] Bạn là trợ lý hỏi đáp trên tài liệu. Chỉ trả lời dựa vào context được cung cấp.
[User] Context:
<chunk 1 [handbook.pdf, tr.12]> ... </chunk 1>
<chunk 2 [handbook.pdf, tr.15]> ... </chunk 2>

Câu hỏi: Sản phẩm được bảo hành bao lâu?
```

Nguyên tắc: **context là nguồn sự thật duy nhất**. LLM có thể "biết" câu trả lời từ kiến thức huấn luyện, nhưng trong RAG hệ thống phải trả lời theo tài liệu — vì tài liệu của người dùng có thể mới hơn, riêng tư hơn, hoặc mâu thuẫn với kiến thức cũ của model.

## Trích dẫn nguồn (citation)

### Citation format

Mỗi luận điểm trong câu trả lời phải gắn với nguồn cụ thể. Format nên dùng trong project (đáp ứng yêu cầu UI "trang X, đoạn Y"):

```
Sản phẩm được bảo hành 24 tháng kể từ ngày mua [1].
Riêng pin được bảo hành 6 tháng [2].

Nguồn:
[1] handbook.pdf — trang 12, mục "Chính sách bảo hành"
[2] handbook.pdf — trang 15, mục "Pin và sạc"
```

Hoặc inline ngắn gọn: `... bảo hành 24 tháng (handbook.pdf, tr.12)`. Điểm bắt buộc: mỗi citation phải trỏ về **metadata đã lưu lúc ingestion** (`doc_id + page` — xem [file 03](03-ingestion-parsing-chunking.md)), để UI click ra được **đoạn gốc**.

### Vì sao citation là bắt buộc

- **Kiểm chứng được:** người dùng không phải tin lời AI — họ click vào nguồn và tự đọc đoạn gốc. Đây là khác biệt giữa "chatbot nói bừa" và "hệ thống hỏi đáp tin cậy".
- **Bắt model "chỉ dùng context":** khi bị buộc đánh số nguồn, model không thể bịa — nếu không có nguồn để trỏ tới, nó phải nói "không tìm thấy trong tài liệu".
- **Debug retrieval:** citation sai/bịa chỉ vào chunk không liên quan → biết ngay retrieval đang kéo nhầm chunk nào.
- **DoD của project:** "Trả lời có trích dẫn nguồn, click ra được đoạn gốc" (file 07).

Có thể ép chặt hơn nữa: yêu cầu LLM trả JSON `{"answer": ..., "citations": [{"chunk_id": 3, "quote": "đoạn trích chính xác"}]}` — quote phải nằm nguyên văn trong chunk, nếu không thì loại.

## Prompt chống hallucination

Mẫu prompt cụ thể (dùng trực tiếp được):

```text
Bạn là trợ lý hỏi đáp trên tài liệu. Quy tắc BẮT BUỘC:
1. Chỉ trả lời dựa trên các đoạn context được cung cấp bên dưới.
2. KHÔNG sử dụng kiến thức bên ngoài, không suy đoán, không bịa.
3. Mọi luận điểm phải gắn trích dẫn dạng [số] trỏ về đoạn context tương ứng.
4. Nếu context không chứa đủ thông tin để trả lời, hãy nói rõ:
   "Tài liệu hiện tại không có thông tin này" — và gợi ý từ khóa tìm kiếm thêm.
5. Nếu context mâu thuẫn nhau, nêu rõ cả hai nguồn và đánh dấu mâu thuẫn.

Context:
[1] (handbook.pdf, tr.12): ...
[2] (handbook.pdf, tr.15): ...

Câu hỏi: {question}

Trả lời bằng tiếng Việt, ngắn gọn, kèm trích dẫn [số].
```

Các kỹ thuật bổ trợ:

- **Trích kèm số chunk id** trong context (`[1]`, `[2]`) để model trích dẫn dễ và chính xác.
- **Few-shot 1–2 ví dụ** về câu trả lời có citation chuẩn và một ví dụ "không đủ thông tin" — model bám format tốt hơn nhiều so với chỉ luật.
- **Temperature thấp** (0–0.2) cho tác vụ hỏi đáp trích dẫn: giảm sáng tạo → giảm bịa.

## Xử lý khi context không đủ để trả lời

Đây là tình huống **bắt buộc phải test** — RAG tệ nhất là khi nó tự tin trả lời sai. Kịch bản và cách xử lý:

1. **Context không liên quan gì câu hỏi** (retrieval fail hoàn toàn): model phải trả về câu "không tìm thấy thông tin" theo rule 4 ở trên. Có thể đặt ngưỡng cứng: nếu điểm rerank cao nhất < ngưỡng → không gọi LLM, trả thẳng "chưa có tài liệu phù hợp".
2. **Context liên quan nhưng thiếu chi tiết** (hỏi giá, nhưng tài liệu chỉ nói chính sách): trả lời phần có được, **ghi rõ phần thiếu**, gợi ý tài liệu/từ khóa cần bổ sung.
3. **Context mâu thuẫn** (hai tài liệu nói khác nhau): nêu cả hai kèm nguồn, không tự chọn — để người dùng quyết định nguồn nào đúng (rule 5).
4. **Câu hỏi vượt ngoài phạm vi tài liệu** ("thủ đô nước Pháp là gì?" trong kho handbook): từ chối trả lời theo kiến thức nền, giữ đúng cam kết "chỉ dựa trên context".

Kiểm chứng bằng eval: metric **faithfulness** (file 06) đo đúng điều này — câu trả lời có "bịa" ra ngoài context không. Nếu faithfulness thấp, sửa prompt + rule citation trước, sau đó mới xem lại retrieval.

## Liên kết

- [04 — Retrieval & Hybrid Search](04-retrieval-reranking-hybrid.md) — context chất lượng đến từ đâu
- [06 — Evaluation với Ragas](06-evaluation-ragas.md) — đo faithfulness/answer relevancy của generation
- [07 — Project Docs-QA](07-project-docs-qa.md) — UI hiển thị trích dẫn "trang X, đoạn Y"
