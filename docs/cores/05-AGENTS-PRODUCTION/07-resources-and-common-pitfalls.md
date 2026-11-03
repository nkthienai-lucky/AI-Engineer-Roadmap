# 07 — Tài nguyên & Lỗi thường gặp (Agents & Production)

> **Mục tiêu:** Có danh sách tài nguyên học tập cho giai đoạn Agents & Production, và hiểu tường tận 4 lỗi thường gặp khiến dự án agent chết trước khi ra production — cùng cách phòng tránh cụ thể cho từng lỗi.
>
> Nguồn mở rộng từ: [05-AGENTS-PRODUCTION.md](../../../05-AGENTS-PRODUCTION.md)

---

## Tài nguyên

| Tài nguyên | Nội dung | Link |
|---|---|---|
| **DeepLearning.AI — Context Engineering** | Cho agent đúng thông tin, đúng lúc | [deeplearning.ai/courses/context-engineering](https://www.deeplearning.ai/courses/context-engineering) |
| **LangGraph docs** | Xây agent có trạng thái | [langchain-ai.github.io/langgraph](https://langchain-ai.github.io/langgraph/) |
| **Full Stack Deep Learning** | Deploy LLM production | [fullstackdeeplearning.com](https://fullstackdeeplearning.com/) |
| **AI Engineer Roadmap 2026** | Toàn cảnh kỹ năng + ý tưởng project | [dataskew.io/roadmaps/ai-engineering](https://dataskew.io/roadmaps/ai-engineering/) |
| **Langfuse** | Observability (self-host) | [langfuse.com](https://langfuse.com/) |

Cách dùng gợi ý: **Context Engineering** học trước khi thiết kế memory/prompt cho agent (file 02–03); **LangGraph docs** đọc khi flow agent bắt đầu có nhánh (file 02); **Full Stack Deep Learning** là tài liệu chính cho production hóa (file 05–06); **AI Engineer Roadmap 2026** dùng để đối chiếu lộ trình tổng thể và tìm ý tưởng mở rộng capstone; **Langfuse** đọc kèm file 05 khi cài observability.

## 4 lỗi thường gặp — mở rộng

Bốn lỗi dưới đây là những cái bẫy **tốn kém nhất** ở mảng agent & production — điểm chung của chúng là không nổ ngay lúc code mà "giết" dự án dần dần: chậm, tốn tiền, cải tiến ảo, hoặc chết khi mở rộng. Bảng tóm tắt trước, phân tích chi tiết sau:

| Lỗi | Dấu hiệu | Hậu quả | Liều chính |
|---|---|---|---|
| Ảo tưởng quyền lực | Agent có 15–20 tool "phòng khi cần" | Chậm, tốn token, gọi nhầm tool | Least privilege, gom tool |
| Đổi prompt không eval | Sửa theo cảm giác, "có vẻ hay hơn" | Cải tiến ảo, không biết đường nào tốt | Golden set + metric + CI gate |
| Không tính chi phí token | Demo ngon, mở rộng thì hóa đơn sốc | Sản phẩm chết vì chi phí | Tính $/user/ngày từ design |
| "Chạy máy mình" là xong | README ghi `python app.py` | Không ai dùng được, không biết khi nào hỏng | Deploy thật + domain + monitoring |

### 1. Agent "ảo tưởng quyền lực": cho quá nhiều tool không cần thiết

**Triệu chứng:** agent có 15–20 tool "phòng khi cần" → mỗi lượt model phải chọn trong một rừng lựa chọn → **chậm** (prompt dài hơn mỗi lượt gọi), **tốn token** (mỗi schema tool là token), và **hay sai** (chọn nhầm tool gần giống, hoặc gọi tool vô nghĩa cho câu hỏi đơn giản). Đây là lỗi kinh điển: mọi tool thừa đều là một cơ hội để model sai.

**Phòng tránh:** nguyên tắc **least privilege** cho tool — mỗi tool phải trả lời được câu hỏi "có kịch bản thật nào cần nó không?". Không có → cắt. Gom tool trùng chức năng thành một tool có tham số (ví dụ 3 tool `search_docs`/`search_web`/`search_db` thành 1 `search(source=...)`). Và mô tả tool phải phân biệt rạch ròi ranh giới giữa các tool (file 01). Ít tool sắc bén luôn thắng nhiều tool mập mờ.

**Tự kiểm:** đọc danh sách tool hiện tại — với mỗi tool, nếu không kể ngay được một tình huống user thật cần nó, hãy cắt hoặc gom nó vào tool khác.

### 2. Không có eval khi đổi prompt → cải tiến ảo

**Triệu chứng:** sửa prompt, thử 2–3 câu thấy "có vẻ hay hơn" → commit. Tuần sau đổi tiếp, "có vẻ" tiếp. Đến lúc nhìn lại, không ai biết hệ thống đang tốt hơn hay tệ hơn lúc đầu — mỗi lần sửa là một canh bạc mù.

**Phòng tránh:** **trước khi** sửa prompt, dựng bộ câu hỏi test + metric (Ragas offline — GĐ3; LLM-as-judge online — file 05). Mọi thay đổi prompt/model chạy lại bộ test đó và **so số**: điểm tăng mới tính là cải tiến. Nâng cao: chặn deploy khi eval regression fail (CI — file 06). Không có số liệu, "cải tiến" chỉ là cảm giác.

**Tự kiểm:** nếu bạn vừa đổi prompt mà không có con số nào trước/sau để so — hãy coi thay đổi đó là *chưa được chứng minh*, chưa phải cải tiến.

### 3. Bỏ qua chi phí token khi design → sản phẩm chết vì hóa đơn

**Triệu chứng:** demo chạy ngon với 10 user test, mở rộng thật thì hóa đơn API vượt dự toán — hoặc với model tự host, VPS sập vì không chịu nổi tải. Nguyên nhân: thiết kế không tính token/request (context phình, agent loop dài, retry không giới hạn) và không quy đổi ra tiền/người/ngày.

**Phòng tránh:** **tính chi phí/người dùng/ngày từ đầu** (công thức mẫu ở [file 04](04-guardrails-and-safety.md)): token mỗi request × giá model × số request/ngày/user. Sau đó thiết kế quanh con số đó: cache semantic (file 06), cắt context (file 03), giới hạn số bước agent, rate limit. Theo dõi chi phí thật bằng trace (file 05) ngay từ tuần đầu có user.

**Tự kiểm:** viết ra giấy con số "1 user dùng bình thường tốn bao nhiêu tiền/ngày" — viết không được nghĩa là bạn đang thiết kế mù.

### 4. Deploy "chạy trên máy mình" và gọi là xong

**Triệu chứng:** demo trên laptop dev, README ghi "chạy `python app.py`", thế là xong. Hậu quả: máy dev tắt thì sản phẩm chết, không ai ngoài bạn cài được, không domain, không HTTPS, không biết khi nào hệ thống hỏng giữa đêm.

**Phòng tránh:** deploy **lên server thật** đúng nghĩa — với lộ trình này là Oracle Always Free + Cloudflare ([kiến trúc 0đ](../07-DEPLOYMENT/01-overall-architecture.md)): có **domain** (người dùng gõ URL vào là dùng được), có **monitoring** (UptimeRobot cảnh báo sập — file 05/07-DEPLOYMENT), có CI/CD để code trên `main` = code đang chạy (file 06). Tiêu chí "xong" của production là: **người lạ mở link, dùng được, và bạn biết ngay khi nó hỏng**.

**Tự kiểm:** nhờ một người khác thử dùng sản phẩm từ máy của họ, qua URL public, không cần bạn ngồi cạnh hướng dẫn — không làm được thì chưa xong.

## Điểm chung & câu hỏi ôn tập

Cả bốn lỗi đều có chung một mẫu: **thành công lúc demo, thất bại lúc mở rộng** — và đều phát hiện được sớm nếu có số liệu (số tool, điểm eval, chi phí/ngày, monitoring). Tinh thần chung của GĐ5 là: agent production không chỉ là "model thông minh" mà là **kỹ thuật quanh model** — tool thiết kế chặt, eval đo được, chi phí tính trước, deploy thật.

Câu hỏi tự soát trước khi chuyển sang checklist hoàn thành ở [README.md](../../../README.md) của lộ trình:

1. Agent của bạn có tool nào mà bạn không kể được kịch bản thật cần nó không?
2. Lần sửa prompt gần nhất, bạn có con số trước/sau để so không?
3. 1 user dùng bình thường tốn bao nhiêu token (và tiền) mỗi ngày?
4. Nếu laptop của bạn tắt ngay bây giờ, sản phẩm có còn chạy và bạn có biết nó hỏng không?

## Liên kết

- [04 — Guardrails & An toàn](04-guardrails-and-safety.md) — chi phí per-request & rate limit (liên quan lỗi 1, 3)
- [05 — Observability & Eval](05-observability-eval-production.md) — eval online + trace (liên quan lỗi 2, 4)
- [06 — Production hóa & Capstone](06-productionization-and-capstone.md) — deploy thật + CI/CD (liên quan lỗi 4)
