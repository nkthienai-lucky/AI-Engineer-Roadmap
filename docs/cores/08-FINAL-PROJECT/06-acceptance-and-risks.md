# Acceptance Criteria & Rủi ro — GenAI Knowledge Assistant

> **Mục tiêu:** Nắm 8 tiêu chí nghiệm thu cuối (đo được, không cảm tính) và 7 rủi ro thường gặp kèm dấu hiệu nhận biết sớm + cách xử lý — để biết khi nào dự án "xong thật" và khi nào đang trượt đường ray thì sửa sớm thay vì muộn.
> Nguồn mở rộng từ: [08-FINAL-PROJECT](README.md)

---

## 8 tiêu chí nghiệm thu cuối (acceptance criteria)

Đây là checklist "done" của cả capstone. Nguyên tắc: **mỗi tiêu chí đều đo được bằng một phép kiểm tra cụ thể** — không có "hoạt động tốt", "ổn định", "mượt".

- [ ] **Live:** truy cập bằng domain (is-a.dev hoặc domain riêng), HTTPS hoạt động. Kiểm tra: mở domain trên trình duyệt khác/mạng khác (không phải localhost), ổ khóa HTTPS hiện, Cloudflare SSL active.
- [ ] **RAG đúng:** 30 câu hỏi test → faithfulness > 0.8, context recall > 0.7 (Ragas). Kiểm tra: script eval chạy 1 lệnh, in ra 2 con số; bộ 30 câu cố định (có sẵn đáp án + tài liệu nguồn) để so sánh được giữa các lần sửa.
- [ ] **Citation:** mỗi câu trả lời RAG trỏ được về đoạn nguồn gốc. Kiểm tra: với mỗi citation, mở đúng trang/đoạn trong PDF gốc và xác nhận nội dung được trích thật sự nằm ở đó — citation sai trang cũng tính là fail.
- [ ] **VLM:** 10 ảnh hóa đơn/biểu mẫu → trích JSON đúng ≥ 8. "Đúng" ở mức field: số tiền, ngày, nhà cung cấp phải khớp với ảnh; mỗi ảnh chấm tay theo bảng điểm cố định.
- [ ] **Performance:** chat trả lời token đầu < 3s (CPU), streaming mượt. Kiểm tra: đo thời gian từ lúc gửi câu hỏi đến token đầu tiên (p50 qua ≥ 20 lần thử), xem stream không bị giật cục hay đứng hình.
- [ ] **Stability:** chạy 7 ngày không crash; có UptimeRobot + log lỗi (Sentry/Langfuse). Kiểm tra: UptimeRobot ping 5 phút/lần suốt 7 ngày, uptime > 99%; khi cố tình gây lỗi, log phải ghi lại được exception để chẩn đoán.
- [ ] **Chi phí = 0đ:** mọi thành phần nằm trong danh sách free đã chốt. Kiểm tra: rà lại từng dòng bảng tech stack — không phát sinh thêm service trả phí ngầm (domain free, Cloudflare free, Langfuse free tier, Oracle Always Free).
- [ ] **Portfolio:** README + demo video ≤ 5 phút + bài viết "học được gì". Demo video ≤ 5 phút ép bạn **chọn lọc** thứ đáng khoe (RAG + citation là chính); README phải để người khác chạy được locally.

## 7 rủi ro & cách xử lý — kèm dấu hiệu nhận biết sớm

Phần quan trọng nhất của bảng dưới là cột **dấu hiệu**: rủi ro không xuất hiện đột ngột — luôn có tín hiệu báo trước, và xử lý khi còn là tín hiệu thì rẻ hơn nhiều khi đã là sự cố.

| Rủi ro | Dấu hiệu nhận biết | Xử lý |
|---|---|---|
| Model 3B trả lời chậm trên 2 OCPU | Token đầu > 5s; CPU 100% kéo dài; user phải chờ mỗi lượt | Quantize Q4_K_M, giới hạn max_tokens, cache semantic cho câu lặp |
| 12GB RAM không đủ (LLM + VLM + DB) | `free -h` thấy RAM trống < 1GB; swap bị dùng nhiều; hệ thống "ì" bất thường | Tải VLM theo demand (unload khi không dùng); hoặc tách VLM sang 1 VPS AMD nhỏ |
| Oracle "out of capacity" khi tạo VPS | Tạo instance báo lỗi capacity ngay hoặc pending mãi | Thử region Singapore/Osaka, thử lại nhiều lần; xem [07-DEPLOYMENT](../07-DEPLOYMENT/README.md) |
| RAG trả lời sai/hallucination | Câu trả lời có chi tiết không nằm trong context; citation trỏ đoạn không liên quan; feedback xấu tăng | Bắt buộc citation + guardrail "chỉ trả lời dựa trên context"; theo dõi feedback user (US4) |
| Cloudflare cắt stream (524) khi model sinh lâu | Response dài bị đứt giữa chừng, log Cloudflare hiện HTTP 524 sau ~100s | SSE + keepalive ping 15–30s + giới hạn max_tokens — xem 07-DEPLOYMENT mục C |
| OOM kill khi load cả LLM + VLM | Process Ollama/VLM chết không rõ lý do, `dmesg` có "Out of memory killed" | OLLAMA_MAX_LOADED_MODELS=1, swap 4GB, unload VLM khi không dùng — xem 07-DEPLOYMENT mục D |
| Quá ham tính năng mới | Mỗi ngày nghĩ ra 1 ý tưởng hay; milestone trượt dần; code nhánh chưa merge chồng chất | Quay lại đọc "Phạm vi" — mọi request mới ghi vào backlog, không thêm vào capstone |

### Đọc bảng theo nhóm

- **Nhóm hiệu năng (2 dòng đầu):** hệ quả tất yếu của việc "chạy thật trên CPU free" — đây không phải bug mà là **bài học** về tối ưu inference. Mục tiêu không phải biến 3B thành nhanh như API, mà giữ latency ở mức dùng được (token đầu < 3s).
- **Nhóm hạ tầng (out of capacity, 524, OOM):** đều là "bẫy" của hạ tầng 0đ. Điểm chung của cách xử lý: **giới hạn và giữ chỗ dư** — giới hạn max_tokens, giới hạn số model nạp, giữ swap, giữ buffer RAM. Chi tiết tại [07-DEPLOYMENT](../07-DEPLOYMENT/README.md).
- **Nhóm chất lượng (hallucination):** rủi ro nguy hiểm nhất vì nó **im lặng** — câu trả lời sai nghe vẫn rất tự tin. Ba lớp phòng vệ: guardrail prompt, citation bắt buộc, feedback user (US4) + eval Ragas định kỳ (chính là lý do tiêu chí "RAG đúng" tồn tại).
- **Nhóm con người (ham tính năng mới):** rủi ro duy nhất không phải kỹ thuật — và là rủi ro khiến dự án chết nhiều nhất. Cách xử lý không phải "cố gắng tập trung" mà là **quy trình**: ý tưởng → backlog → không đụng capstone.

## Cách tổ chức kỳ nghiệm thu cuối

Thứ tự chạy nên đi từ **dễ đo → khó đo**, để phát hiện vấn đề ở tầng hạ tầng trước khi kết luận sai về chất lượng model:

1. **Hạ tầng trước:** Live HTTPS + Stability (UptimeRobot 7 ngày) + Chi phí 0đ — ba mục này chỉ cần chờ và rà soát, không phụ thuộc chất lượng AI.
2. **Số liệu RAG sau:** chạy bộ 30 câu qua Ragas (RAG đúng) → chấm citation tay (Citation) → chấm 10 ảnh (VLM). Ba mục này là lõi portfolio.
3. **Cuối cùng:** Performance (token đầu < 3s) và Portfolio (README + video ≤ 5 phút).

Toàn bộ kỳ nghiệm thu nên **timebox trong 1 tuần** sau M4 — nếu một tiêu chí fail, sửa đúng chỗ đó rồi chạy lại đúng phép kiểm tra đã định nghĩa, không đánh giá lại bằng cảm tính.

## Liên kết

- [05 — Milestones](05-milestones.md) — nghiệm thu từng milestone trước khi tới nghiệm thu tổng này
- [03 — Kiến trúc & luồng xử lý](03-architecture-and-flows.md) — nơi sinh ra các rủi ro kỹ thuật (RAG, VLM, streaming)
- [07 — Sau khi hoàn thành](07-after-completion.md) — việc làm tiếp sau khi 8 tiêu chí pass hết
