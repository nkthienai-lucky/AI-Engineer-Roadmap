# Evaluation với Ragas (Tuần 11 — phần quan trọng nhất!)

> **Mục tiêu:** Nắm định nghĩa trực quan từng metric (faithfulness, answer relevancy, context precision/recall), thành thạo framework **Ragas** (bắt buộc), biết DeepEval/TruLens như lựa chọn thay thế, xây eval dataset chất lượng (câu hỏi + ground truth), chạy vòng lặp eval → sửa pipeline → đo lại, và đọc khái niệm Agentic RAG / GraphRAG.
> Nguồn mở rộng từ: [03-RAG](README.md)

---

> 💡 Vì sao đây là phần quan trọng nhất? Không có eval, mọi thay đổi (chunk size, reranker, embedding…) chỉ là **cảm tính** — bạn sửa pipeline trong mù và không biết phiên bản nào thật sự tốt hơn. Eval biến RAG từ "nghệ thuật" thành kỹ thuật có số liệu.

## Metrics: định nghĩa trực quan

### Faithfulness (độ trung thành)

**Câu trả lời có đúng dựa trên context đã truy vấn được không?** Ragas tính bằng cách tách câu trả lời thành từng luận điểm (claim), rồi kiểm tra từng luận điểm có được **hỗ trợ bởi context** không. Ít claim bịa → faithfulness cao.

*Trực quan:* "Mỗi câu LLM nói ra có trỏ ngược về được đoạn context không?" — metric này bắt đúng chứng bệnh hallucination (file 05).

### Answer relevancy (độ liên quan của câu trả lời)

**Câu trả lời có thật sự trả lời câu hỏi không?** Lưu ý tinh tế: answer relevancy **không** kiểm tra đúng/sai — chỉ kiểm tra có **liên quan đến câu hỏi** không. Một câu trả lời đúng context nhưng lạc đề (hỏi thời hạn bảo hành, trả lời chính sách đổi trả) sẽ có faithfulness cao nhưng answer relevancy thấp.

*Trực quan:* "Người hỏi có thấy mình được trả lời đúng câu hỏi không?"

### Context precision (độ chính xác của context)

**Trong các chunk được truy vấn, tỷ lệ chunk thực sự liên quan đến câu hỏi là bao nhiêu?** Context precision cao = top các chunk đều liên quan, ít "rác" lẫn vào.

*Trực quan:* "Đưa vào prompt có thừa chunk vô ích không?" — thấp → retrieval kéo nhầm nhiều → cần rerank/hybrid (file 04). Chunk thừa không chỉ vô hại: nó pha loãng attention của LLM và kéo faithfulness xuống theo.

### Context recall (độ bao phủ của context)

**Những thông tin cần để trả lời ground truth có nằm trong context không?** Context recall cao = không bỏ sót mảnh ghép nào.

*Trực quan:* "Tài liệu đúng có lọt vào top-k không?" — thấp → retrieval yếu (bỏ sót) → thử multi-query, hybrid search, tăng k, hoặc sửa chunking (file 03).

**Ma trận chẩn đoán nhanh:**

| Triệu chứng | Vấn đề | Sửa ở đâu |
|---|---|---|
| Context recall thấp | Retrieval bỏ sót | Multi-query, hybrid, chunk size |
| Context precision thấp | Retrieval kéo nhầm | Reranker, ngưỡng lọc |
| Faithfulness thấp | Model bịa | Prompt chống hallucination (file 05) |
| Answer relevancy thấp | Trả lời lạc đề | Query rewriting, prompt rõ ràng hơn |

## Framework: Ragas (bắt buộc), DeepEval, TruLens

- **Ragas** ([docs.ragas.io](https://docs.ragas.io/)) — **bắt buộc** theo roadmap. Chuyên đánh giá RAG, tính đủ 4 metric trên (một số metric dùng LLM-as-judge), xuất bảng số liệu chi tiết. Dùng trực tiếp:

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision, context_recall
from datasets import Dataset

data = Dataset.from_dict({
    "question": ["Sản phẩm bảo hành bao lâu?"],
    "answer": ["Bảo hành 24 tháng [1]."],
    "contexts": [["Sản phẩm được bảo hành 24 tháng..."]],
    "ground_truth": ["24 tháng"],
})
result = evaluate(data, metrics=[faithfulness, answer_relevancy,
                                 context_precision, context_recall])
print(result)   # bảng điểm từng metric
```

- **DeepEval** — framework test kiểu unit-test cho LLM (viết test case assert metric ≥ ngưỡng), hợp đưa eval vào CI/CD.
- **TruLens** — theo dõi/đánh giá ứng dụng LLM với bộ "triad" (context relevance, groundedness, answer relevance) tương đương 4 metric trên, mạnh về observability.

Quan trọng: **Ragas là bắt buộc** (đúng yêu cầu gốc); DeepEval/TruLens là mở rộng kiến thức — biết để chọn khi cần tích hợp CI hoặc monitoring.

## Xây eval dataset

Mỗi mẫu eval gồm: **câu hỏi + ground truth (câu trả lời chuẩn của người)** + context kỳ vọng (tùy metric). Chất lượng dataset quyết định ý nghĩa của con số eval.

- **Bao nhiêu câu?** Project GĐ3 yêu cầu **30+ câu hỏi** (file 07). Với dự án thật, 50–200 câu là phổ biến. 30 câu đủ thấy xu hướng, không đủ để so sánh sát sao — vì vậy luôn kèm bảng so sánh naive vs cải tiến trên **cùng một bộ câu hỏi**.
- **Chất lượng dataset:** câu hỏi phải **đa dạng kiểu**: fact đơn giản ("sản phẩm X giá bao nhiêu"), suy luận nhiều bước ("so sánh chính sách A và B"), câu hỏi cần ghép thông tin từ nhiều chunk, câu hỏi **ngoài tầm tài liệu** (đoạn trả lời đúng phải là "không có thông tin"), câu hỏi có từ khóa hiếm (đoạn hybrid search). Ground truth phải do người đọc tài liệu viết, không phải copy-paste nguyên văn một chunk (tránh eval thiên vị).
- **Train/test:** nếu tinh chỉnh pipeline trên eval set, hãy giữ một phần câu hỏi làm "test" chưa từng sửa để tránh overfit vào bộ câu hỏi.

## Vòng lặp eval → sửa pipeline → đo lại

```
Xây dataset (30+ câu) ──▶ Chạy pipeline ──▶ Đánh giá bằng Ragas
        ▲                                        │
        │                                        ▼
   mở rộng dataset ◀── đọc bảng điểm ──▶ sửa 1 biến: chunk size?
   (thêm case khó)                        rerank? embedding? prompt?
                                                 │
                                                 ▼
                                          đo lại, so số liệu cũ/mới
```

1. **Chạy baseline:** pipeline hiện tại + dataset → bảng điểm Ragas.
2. **Đọc kết quả:** dùng ma trận chẩn đoán ở trên — faithfulness thấp thì sửa prompt, context recall thấp thì sửa retrieval… **chỉ sửa một biến mỗi vòng** để biết thay đổi nào gây ra cải tiến.
3. **Đo lại:** cùng dataset, cùng metric — so số liệu trước/sau bằng bảng (yêu cầu DoD: bảng "naive vs +rerank vs +hybrid").
4. **Lặp** cho tới khi đạt DoD (faithfulness > 0.8, context recall > 0.7).

Đây chính là lý do "quên eval = sửa trong mù" là lỗi thường gặp (file 08).

## Advanced (đọc khái niệm)

- **Agentic RAG:** thay vì pipeline cố định retrieve-then-generate, một **agent quyết định** khi nào cần truy vấn, truy vấn gì, có cần truy vấn lại không, hay cần dùng công cụ khác (tính toán, web search). Linh hoạt hơn cho câu hỏi phức hợp; đổi lấy độ phức tạp và chi phí.
- **GraphRAG:** tổ chức tri thức thành **đồ thị thực thể/mối quan hệ** thay vì các chunk rời rạc, cho phép trả lời câu hỏi cần **suy luận liên kết** ("ai liên quan đến sự kiện X qua những khâu nào?") mà vector search thuần khó làm. Chi phí xây dựng tri thức cao hơn; đọc khái niệm ở mức biết khi nào nên cân nhắc.

## Liên kết

- [04 — Retrieval & Hybrid Search](04-retrieval-reranking-hybrid.md) — công cụ sửa retrieval khi context precision/recall thấp
- [05 — Generation & Chống hallucination](05-generation-citation-anti-hallucination.md) — sửa faithfulness/answer relevancy
- [07 — Project Docs-QA](07-project-docs-qa.md) — V3: eval Ragas 30+ câu hỏi, bảng so sánh, DoD số liệu
- [08 — Tài nguyên & Lỗi thường gặp](08-resources-and-common-pitfalls.md) — link Ragas docs
