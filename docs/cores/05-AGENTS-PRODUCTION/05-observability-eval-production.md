# 05 — Observability & Eval production: Langfuse, eval online, chi phí, A/B

> **Mục tiêu:** Thiết lập logging trace cho agent bằng **Langfuse**, chạy eval online theo dõi chất lượng theo thời gian thật, theo dõi chi phí token và latency P95, và biết A/B thử prompt/model khác nhau một cách có số liệu.
>
> Nguồn mở rộng từ: [05-AGENTS-PRODUCTION](README.md)

---

## Vì sao cần observability cho LLM?

Agent là hệ thống **không xác định**: cùng một input, output đổi theo prompt, model, retrieval. Khi chất lượng tụt, câu hỏi đầu tiên phải trả lời được là *"tụt ở khâu nào?"* — retrieval kém? prompt xấu? tool fail? model yếu? Không có trace thì mọi phán đoán là cảm tính. Observability = **nhìn thấy từng bước** của mỗi request.

## Logging trace với Langfuse

**Langfuse** là công cụ observability cho LLM: ghi lại từng **trace** — một request của người dùng — với cây phân cấp spans: retrieval → rerank → prompt → model call → tool calls → output, kèm token count, chi phí, latency từng bước. Khi một câu trả lời sai, bạn mở trace lên là thấy ngay "context retrieval đã sai từ đầu" hay "model bịa dù context đúng".

Tích hợp kiểu decorator — mỗi hàm trong pipeline là một span:

```python
from langfuse.decorators import observe, langfuse_context

@observe(name="retrieve")            # tự tạo span, ghi input/output + thời gian
def retrieve(query: str):
    chunks = vector_db.search(query, top_k=5)
    return chunks

@observe(name="generate")
def generate(query: str, chunks):
    answer = llm(build_prompt(query, chunks))
    langfuse_context.update_current_observation(
        usage_details={"input": count_tokens(query), "output": count_tokens(answer)})
    return answer
```

> **Ghi chú thực tế (đọc kỹ):** file gốc ghi Langfuse "self-host free trên Oracle VPS", nhưng với VPS **12GB RAM** chạy sẵn LLM + DB + app thì nên dùng **Langfuse Cloud free** — vì **self-host Langfuse v3 cần ClickHouse + Redis + Postgres**, quá nặng cho VPS 12GB (sẽ phải đánh đổi model/thiếu RAM cho LLM ≤3B quantized). Cloud free là lựa chọn thực dụng: vẫn có trace đầy đủ, không tốn RAM máy.

## Eval online — theo dõi chất lượng theo thời gian thật

Eval offline (bộ câu hỏi test chạy khi dev — GĐ3 với Ragas) cho biết *trước khi deploy*. **Eval online** cho biết *sau khi deploy*: chất lượng thực tế với traffic thật thay đổi thế nào theo thời gian.

Các cách làm, từ đơn giản đến nâng cao:

- **User feedback**: nút 👍/👎 trên mỗi câu trả lời — rẻ, trực tiếp, và là "ground truth từ người thật".
- **LLM-as-judge định kỳ**: mỗi đêm (hoặc sampling 5–10% request), một model chấm output thật theo tiêu chí (đúng trọng tâm? có trích dẫn? không bịa?) — ghi điểm vào Langfuse để vẽ xu hướng.
- **Regression test tự động**: bộ câu hỏi vàng (golden set) chạy lại mỗi khi đổi prompt/model — fail thì chặn deploy (xem CI ở [file 06](06-productionization-and-capstone.md)).

## Theo dõi chi phí token & latency P95

Hai metric vận hành phải thấy **hàng ngày**:

| Metric | Đo gì | Ngưỡng cảnh giác | Công cụ |
|---|---|---|---|
| Chi phí token | Token in/out theo ngày/user/feature | Vượt ngân sách/user/ngày đã tính | Langfuse usage + cost |
| Latency P95 | 95% request nhanh hơn mức này | Chat: vài giây; agent loop: vài chục giây | Langfuse trace spans |
| Error/tool-fail rate | Tỷ lệ tool call thất bại | Spike bất thường = có gì vỡ | Trace spans lỗi |
| Chất lượng (judge/feedback) | Điểm LLM-as-judge + 👍/👎 | Xu hướng giảm theo tuần | Eval online (mục dưới) |

- **Chi phí token**: tổng token in/out theo ngày, theo user, theo feature (trace Langfuse tính sẵn cost nếu khai unit price). Mục tiêu: chi phí/user/ngày nằm trong ngân sách tính từ lúc design (file 04). Spike bất thường = vòng lặp agent lỗi hoặc bị abuse.
- **Latency P95**: 95% request nhanh hơn ngưỡng này. Chọn P95 thay vì trung bình vì **trung bình nói dối** — vài request chậm 30s (agent loop dài, retrieval chậm) làm user bực mà trung bình vẫn "đẹp". Với chat, P95 mục tiêu vài giây; vượt thì phải tối ưu (streaming, cache — file 06).

## A/B thử prompt/model khác nhau

Đừng đổi prompt theo cảm tính ("có vẻ hay hơn"). A/B có số liệu:

1. **Chia traffic** — ví dụ 50/50 giữa prompt A (đang chạy) và prompt B (ứng viên), random theo user (một user luôn thấy cùng phiên bản để không lẫn trải nghiệm).
2. **Đo cùng bộ metric** — quality score (LLM-as-judge + 👍/👎), latency P95, chi phí/request.
3. **Chốt bằng số** — B chỉ thay A khi chất lượng tăng (hoặc không giảm) **và** chi phí/latency chấp nhận được. Langfuse cho phép gắn `metadata` (version prompt/model) vào trace để so sánh hai nhóm.

Nguyên tắc xuyên suốt: **mọi thay đổi prompt/model đều phải đi qua eval** — không eval thì "cải tiến" chỉ là ảo (xem [file 07](07-resources-and-common-pitfalls.md)).

Tóm lại, "eval production" khác "eval offline" ở đối tượng đo: offline đo **trước khi ship** trên bộ câu hỏi test, online đo **sau khi ship** trên traffic thật — và hai cái bổ trợ cho nhau: offline chặn regression lúc CI, online phát hiện vấn đề mà bộ test không lường trước (ngôn ngữ mới, kiểu câu hỏi mới, user mới).

## Liên kết

- [02 — Agent loop & LangGraph](02-agent-loop-react-planner.md) — agent nhiều bước là thứ cần trace nhất
- [04 — Guardrails & An toàn](04-guardrails-and-safety.md) — theo dõi chi phí là van an toàn chống abuse
- [06 — Production hóa & Capstone](06-productionization-and-capstone.md) — đưa observability vào stack deploy + CI
