# 07 — Thứ tự setup & Pre-flight Checklist

> **Mục tiêu**: lắp toàn bộ kiến trúc + bẫy đã học vào **1 buổi setup** với 8 bước đúng thứ tự (điều gì trước, điều gì sau, tại sao), và có bộ PRE-FLIGHT CHECKLIST 8 mục với cách kiểm tra cụ thể cho từng mục — xong hết mới được tuyên bố "live".
>
> Nguồn mở rộng từ: [07-DEPLOYMENT](README.md)

---

## Thứ tự setup đề xuất (1 buổi)

Thứ tự này không phải ngẫu nhiên: mỗi bước đặt nền cho bước sau, và bước nguy hiểm nhất (billing) được xử lý **trước mọi thứ**.

```
0. Đăng ký Oracle → TẠO BUDGET ALERT $1 NGAY (mục A) — trước mọi thứ khác
1. Tạo VPS A1 (Ubuntu 24.04 aarch64, 2 OCPU/12GB, region Singapore/Osaka)
2. SSH vào → hardening (mục E) + thêm swap 4GB (mục D)
3. Cài Docker + docker compose (image phải arm64 — mục G)
4. Mở firewall ĐẦY ĐỦ 3 lớp (mục B) → cài nginx + certbot
5. Cài PostgreSQL + pgvector + Ollama (bind 127.0.0.1)
6. Deploy app: docker compose up (log rotate đã cấu hình — mục E)
7. Cài backup cron → Object Storage (mục F)
8. Chạy PRE-FLIGHT CHECKLIST bên dưới → khi nào xong hết mới "live"
```

### Giải thích từng bước

- **Bước 0 — Budget alert $1 NGAY:** mọi resource phía sau đều có thể tạo lại được, **tiền charge oan thì không đòi lại được**. Governance & Administration → Budgets → tạo `always-free-fail-safe` ở $1. Chi tiết: [02-oracle-always-free.md](02-oracle-always-free.md) + [04-pitfalls-billing-and-firewall.md](04-pitfalls-billing-and-firewall.md).
- **Bước 1 — Tạo VPS A1:** Ubuntu 24.04 **aarch64**, shape Ampere A1 **2 OCPU/12GB**, region **Singapore/Osaka** (latency VN 30–80ms; tránh Tokyo/Chuncheon). "Out of capacity" thì retry/đổi Availability Domain — xem capacity tại [cheaploy.com/tools/oracle-capacity-checker](https://cheaploy.com/tools/oracle-capacity-checker).
- **Bước 2 — Hardening + swap trước khi cài gì:** SSH key-only + 2FA + fail2ban + unattended-upgrades (AUDIT E) và **swap 4GB** (`fallocate -l 4G /swapfile`) — làm sớm để mọi service cài sau đều được bảo vệ và không OOM (AUDIT D).
- **Bước 3 — Docker + compose:** kiểm tra image **arm64** (`docker manifest inspect`) trước khi kéo — VPS ARM không chạy image amd64 (AUDIT G). Cấu hình luôn log rotate trong daemon.json (`max-size=10m, max-file=3`) trước khi container đầu tiên chạy.
- **Bước 4 — Firewall 3 lớp → nginx + certbot:** mở OCI Security List/NSG **và** iptables trong VM (`/etc/iptables/rules.v4`) **và** để service nghe đúng cổng — thiếu 1 lớp là "No route to host" (AUDIT B). Xong hạ tầng web mới cài nginx + certbot (SSL tự động renew).
- **Bước 5 — PostgreSQL + pgvector + Ollama:** cả hai đều bind `127.0.0.1` — Postgres không expose public, Ollama nhất định `127.0.0.1:11434` (AUDIT B/E). Đặt env Ollama: `OLLAMA_MAX_LOADED_MODELS=1`, `OLLAMA_KEEP_ALIVE` theo traffic, `num_ctx` 4096 (AUDIT D).
- **Bước 6 — Deploy app:** `docker compose up` với **log rotate đã cấu hình sẵn từ bước 3** — nếu để sau mới cấu hình thì log cũ đã phình. App chỉ bind nội bộ, chỉ nginx public.
- **Bước 7 — Backup cron → Object Storage:** cron pg_dump hằng ngày + rclone/aws cli đẩy lên Object Storage (Standard tier, **không** Archive — AUDIT A/F). Backup phải có **trước khi** có user thật.
- **Bước 8 — PRE-FLIGHT CHECKLIST:** chạy hết 8 mục bên dưới — khi nào **đủ 8/8** mới tuyên bố "live".

### Ước tính thời gian 1 buổi (tham khảo)

| Khối | Bước | Thời gian |
|---|---|---|
| Chuẩn bị | 0 (budget alert) + 1 (tạo VPS, chờ capacity) | 30–60 phút |
| Nền tảng | 2 (hardening + swap) + 3 (Docker) | 45–60 phút |
| Hạ tầng web | 4 (firewall 3 lớp + nginx + certbot) | 45–60 phút |
| Services | 5 (Postgres + Ollama) + 6 (deploy app) | 60–90 phút |
| An toàn | 7 (backup cron) + 8 (pre-flight) | 45–60 phút |

Lưu ý: bước 1 có thể kéo dài nếu region "out of capacity" (retry/đổi Availability Domain) — đừng hoảng, và đừng vì vội mà bỏ qua bước 0 hay bước 2. Hai bước đó là rẻ nhất và cứu bạn khỏi hai hậu quả đắt nhất: charge oan và mất server.

## ✅ PRE-FLIGHT CHECKLIST — trước khi tuyên bố "live"

1. ✅ **Budget alert $1 đã tạo và hoạt động.** *Cách kiểm tra:* vào OCI Console → Governance & Administration → Budgets thấy `always-free-fail-safe` với ngưỡng $1; (tuỳ chọn) test thật bằng cách tạo rồi xoá resource tính tiền nhỏ để xác nhận email/alert đến.
2. ✅ **`curl` từ máy ngoài vào domain → 200 qua HTTPS.** *Cách kiểm tra:* trên máy khác (không phải VPS, không cùng mạng): `curl -I https://your-domain` → `HTTP/2 200` và **không** báo certificate error. Chạy từ máy ngoài là để xác nhận đường đi User → Cloudflare → nginx → app thật sự thông ở cả 3 lớp firewall.
3. ✅ **SSL test (ssllabs.com) đạt A.** *Cách kiểm tra:* chạy SSL test trên ssllabs.com với domain của bạn → grade **A** (xác nhận cert hợp lệ, cipher mạnh, cấu hình Full (strict) đúng — AUDIT G).
4. ✅ **Port scan từ máy khác: chỉ 80/443/22 mở.** *Cách kiểm tra:* từ máy ngoài: `nmap -Pn your-domain` → chỉ thấy 80, 443, 22 (hoặc ít hơn nếu đã giới hạn SSH theo IP). Cổng lạ (5432 Postgres, 11434 Ollama, 8080 app…) hé mở là **fail** — nghĩa là bind nội bộ/firewall có lỗ (AUDIT B).
5. ✅ **Kill thử Ollama → service tự restart (`restart: always`).** *Cách kiểm tra:* `docker kill <ollama-container>` rồi đợi vài chục giây → `docker ps` thấy container chạy lại. Xác nhận compose có `restart: always` để server tự phục hồi sau crash/OOM/reboot (AUDIT D).
6. ✅ **Upload PDF 20MB → hỏi → streaming mượt, KHÔNG bị 524.** *Cách kiểm tra:* upload file PDF ~20MB (vượt `client_max_body_size` cũ 1MB → xác nhận config 50m), đặt câu hỏi về nội dung file, quan sát token **stream liên tục** về UI — không đứng hình >100s, không ra lỗi **524** (AUDIT C).
7. ✅ **Backup Postgres đã restore thử thành công ở máy local (backup không restore = không có backup).** *Cách kiểm tra:* tải dump mới nhất từ Object Storage về máy cá nhân, `psql`/`pg_restore` vào DB local trống → bảng + dữ liệu đầy đủ. Một backup chưa từng restore chỉ là một file hy vọng.
8. ✅ **UptimeRobot + Sentry/Langfuse đã nhận dữ liệu thật.** *Cách kiểm tra:* UptimeRobot hiện monitor **UP** với response time thật; Sentry (hoặc Langfuse nếu bật) đã ghi event/trace từ traffic thật của bạn — xác nhận hệ thống giám sát hoạt động **trước khi** có sự cố thật (mục 4 tại [03-domain-ssl-email-monitoring-devops.md](03-domain-ssl-email-monitoring-devops.md)).

## Ba nguyên tắc xuyên suốt (nhớ 3 điều này là đủ an toàn 80%)

1. **Budget alert $1 làm ĐẦU TIÊN, trước mọi resource** — tiền charge oan không đòi lại được, resource tạo lại được.
2. **Backup không restore = không có backup** — luôn restore thử ở máy local trước khi tin vào file dump.
3. **Mọi route chat phải stream SSE và firewall phải đủ 3 lớp** — hai lỗi tốn nhiều giờ debug nhất (524 và "No route to host") đều bắt nguồn từ việc bỏ qua hai quy tắc này.

## Chuyển tiếp

Sản phẩm cuối cần deploy theo đúng checklist này → xem [08-FINAL-PROJECT](../08-FINAL-PROJECT/README.md).

## Liên kết

- [01-overall-architecture.md](01-overall-architecture.md) — kiến trúc tổng thể mà các bước setup đang lắp ráp.
- [02-oracle-always-free.md](02-oracle-always-free.md) — bước 0–1: budget alert và tạo VPS Always Free.
- [04-pitfalls-billing-and-firewall.md](04-pitfalls-billing-and-firewall.md) — bước 4: firewall 3 lớp (mục B) và bẫy billing (mục A).
- [05-streaming-timeout-and-ram.md](05-streaming-timeout-and-ram.md) — bước 2/5: swap + env Ollama; checklist mục 6 (streaming, 524).
- [06-security-backup-and-infra-quirks.md](06-security-backup-and-infra-quirks.md) — bước 2/3/7: hardening, log rotate, backup cron.
