# Sau khi hoàn thành — hướng đi tiếp theo

> **Mục tiêu:** Biết rõ việc gì làm **sau** khi capstone đã pass đủ 8 tiêu chí (auth, LoRA, GPU, benchmark), và thuộc lòng bảng "chốt" hướng đi cuối cùng — thứ quyết định bạn không lan man suốt 18 tuần và sau này.
> Nguồn mở rộng từ: [08-FINAL-PROJECT.md](../../../08-FINAL-PROJECT.md)

---

## Mục 8 — "Sau khi hoàn thành" (không thuộc lộ trình)

Bốn hướng dưới đây **cố tình nằm ngoài 18 tuần**. Chúng hấp dẫn, có giá trị thật, và sẽ phá capstone nếu đưa vào sớm — mỗi hướng là nhiều tuần làm việc. Quy tắc: chỉ bắt đầu khi 8 tiêu chí nghiệm thu ở [06-acceptance-and-risks.md](06-acceptance-and-risks.md) đã pass hết.

### Thêm auth + multi-user (Supabase Auth free)
Hiện tại capstone chỉ có "1 tài khoản user đơn giản (hoặc không auth, private dùng)" — đúng scope. Khi muốn mở cho người khác dùng, Supabase Auth free tier xử lý đăng nhập (email/OAuth), phân quyền, session — không phải tự viết auth (tự viết auth là nguồn bug bảo mật kinh điển). Việc cần làm thêm: ràng buộc dữ liệu theo user (mỗi user chỉ thấy collection của mình), chính là bước đầu của multi-tenant đã bị loại khỏi scope.

### Fine-tune LoRA trên feedback user đã ghi nhận (US4)
Đây là **lý do thật sự** của US4: mỗi lần user bấm "không đúng" là 1 mẫu `(câu hỏi, câu trả lời sai, câu trả lời đúng)` — tập dữ liệu fine-tune nhỏ mà **đúng domain của bạn**, không đâu có. LoRA chỉ huấn luyện vài triệu tham số thích ứng (adapter) trên base model, chi phí tính toán thấp hơn nhiều so với fine-tune toàn bộ. Kết quả kỳ vọng: model trả lời đúng style và đúng "gu" dữ liệu của bạn mà không cần đổi model. Lưu ý: chất lượng LoRA phụ thuộc hoàn toàn vào chất lượng feedback — snapshot đầy đủ ở US4 là điều kiện tiên quyết.

### Chuyển LLM lên GPU khi có ngân sách (RunPod serverless)
RunPod serverless **chỉ tốn khi có request** — không có request thì không tính tiền, nên hợp với sản phẩm có traffic không đều. Lợi ích: token đầu giảm từ giây xuống mili giây, dùng được model lớn hơn (7B–14B) mà vẫn trả lời nhanh. Đây là "nâng cấp hạ tầng", không phải "sửa code" — nếu kiến trúc capstone tách backend/model sạch (Ollama qua API), việc thay CPU bằng GPU chỉ là đổi endpoint.

### Benchmark so sánh với RAG-as-a-service
Chạy cùng bộ 30 câu test với các dịch vụ RAG managed (không tự build) rồi so **chất lượng + chi phí + thời gian phát triển**. Mục tiêu không phải "thắng", mà là **biết giá trị tự build**: bạn sẽ thấy phần nào dịch vụ làm tốt hơn (thường là orchestration), phần nào tự build thắng (kiểm soát retrieval, chi phí 0đ, hiểu sâu). Bài benchmark này cũng chính là nội dung hay cho bài viết portfolio.

## Bảng "chốt" hướng đi (giữ nguyên từ spec)

| | Quyết định |
|---|---|
| **Học** | 5 giai đoạn, 18 tuần, mỗi giai đoạn 1 project (xem [README.md](../../../README.md)) |
| **Cốt lõi phải giỏi** | Transformer (tự code), RAG + **Eval**, deploy production |
| **Công cụ mặc định** | Ollama · pgvector · Qwen3-Embed · Ragas · FastAPI · Docker |
| **Sản phẩm cuối** | GenAI Knowledge Assistant (chat + RAG + VLM), 4 milestone |
| **Hạ tầng** | Oracle Always Free (Singapore/Osaka) + Cloudflare — **0đ** |
| **Không làm** | Framework-first, học mọi thứ, mở rộng scope, chờ "học đủ rồi mới build" |

## Giải thích bảng — ý nghĩa từng quyết định

- **Học — 5 giai đoạn, 18 tuần, mỗi giai đoạn 1 project:** tiến độ đo bằng **project chạy được**, không phải số video đã xem. Không hoàn thành project = không qua giai đoạn (quy tắc 1 trong [README.md](../../../README.md)).
- **Cốt lõi phải giỏi:** ba thứ phân biệt "kỹ sư AI" với "người dùng API": hiểu Transformer từ bên trong (tự code được attention), xây RAG **có đo lường** (Eval là kỹ năng đắt giá nhất), và ship lên production thật (deploy, monitor, chịu lỗi). Ngoài ba thứ này, mọi thứ khác là công cụ.
- **Công cụ mặc định:** một bộ tool duy nhất lặp lại qua mọi giai đoạn — thành thạo 1 stack sâu tốt hơn biết 10 stack hời hợt. Đây cũng chính là [bảng tech stack](04-tech-stack.md) của capstone.
- **Sản phẩm cuối & Hạ tầng:** đích đến cụ thể (GenAI Knowledge Assistant, 4 milestone) và cam kết chi phí 0đ — ràng buộc ngân sách chính là kỷ luật buộc bạn tối ưu thật sự.
- **Không làm — 4 mục:** (1) *Framework-first* — học LangChain trước khi hiểu RAG là tự bịt mắt; framework chỉ là công cụ rút ngắn **sau khi** hiểu bản chất. (2) *Học mọi thứ* — 18 tuần không đủ cho mọi nhánh AI; học theo đường tới sản phẩm. (3) *Mở rộng scope* — nguyên nhân số 1 khiến capstone chết. (4) *Chờ "học đủ rồi mới build"* — học xong GĐ3 là build được khối RAG ngay; kiến thức chỉ thành kỹ năng khi đem ra dùng, và phần thiếu sẽ tự hiện ra để học đúng lúc cần.

## Thứ tự ưu tiên gợi ý sau capstone

Bốn hướng không bình đẳng — nên xét theo "cái nào nuôi cái nào":

1. **Benchmark vs RAG-as-a-service** làm trước, vì rẻ nhất (chỉ chạy eval) và cho bức tranh rõ nhất về giá trị tự build — kết quả benchmark quyết định có nên đầu tư tiếp hay không.
2. **Auth + multi-user** nếu có người dùng thật muốn dùng sản phẩm — có user thật thì mới có feedback thật.
3. **LoRA trên feedback** khi đã gom đủ feedback chất lượng (vài trăm mẫu tốt hơn hàng nghìn mẫu ồn) — làm sớm trên dữ liệu ít thì tốn công mà model không khá hơn.
4. **GPU RunPod** cuối cùng, khi model đã tốt và chỉ còn nút thắt là tốc độ — nâng cấp hạ tầng khi chất lượng đã ổn thì tiền GPU mới không bị phung phí.

## Liên kết

- [01 — Tầm nhìn & Scope](01-goals-and-scope.md) — sản phẩm "chốt" là gì, tại sao scope giữ vậy
- [05 — Milestones](05-milestones.md) — 4 milestone hoàn thành trước khi tới phần này
- [06 — Acceptance & rủi ro](06-acceptance-and-risks.md) — 8 tiêu chí phải pass trước khi mở rộng
