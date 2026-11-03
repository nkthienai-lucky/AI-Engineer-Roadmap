# 02 — Hạ tầng Oracle Always Free

> **Mục tiêu**: nắm trọn bảng tài nguyên Always Free vĩnh viễn của Oracle Cloud (kèm link chính thức cho từng mục), hiểu hạn mức thực tế của VPS Ampere A1 tính đến 7/2026, và thuộc lòng các "lưu ý vận hành" quyết định instance của bạn sống hay bị thu hồi — trong đó quy tắc **budget alert $1 làm ĐẦU TIÊN** là quan trọng nhất.
>
> Nguồn mở rộng từ: [07-DEPLOYMENT.md](../../../07-DEPLOYMENT.md)

---

## Mục 1 — Bảng Always Free đầy đủ (vĩnh viễn)

Đây là các tài nguyên bạn được dùng **miễn phí vĩnh viễn**, không giới hạn thời gian — khác với gói $300 credit chỉ có hạn 30 ngày. Toàn bộ stack ở file [01-overall-architecture.md](01-overall-architecture.md) dựng trên các tài nguyên này.

| Thành phần | Free | Link |
|---|---|---|
| Đăng ký tài khoản | $300 credit 30 ngày + Always Free vĩnh viễn | [oracle.com/cloud/free](https://www.oracle.com/cloud/free/) |
| Danh sách Always Free chính thức | Docs | [Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm) |
| FAQ | Chính sách | [oracle.com/cloud/free/faq](https://www.oracle.com/cloud/free/faq/) |
| VPS ARM Ampere A1 | **2 OCPU / 12GB RAM**, chạy 24/7 | [Compute shapes](https://docs.oracle.com/en-us/iaas/Content/Compute/References/computeshapes.htm) |
| VPS AMD nhỏ ×2 | 1/8 OCPU + 1GB (cron/backup) | cùng link trên |
| Block storage | 200 GB tổng | [Block Volume](https://docs.oracle.com/en-us/iaas/Content/Block/Concepts/overview.htm) |
| Object Storage (S3) | 20 GB | [Object Storage](https://docs.oracle.com/en-us/iaas/Content/Object/Concepts/objectstorage.htm) |
| Autonomous DB ×2 | 20GB mỗi DB | [Autonomous Database](https://docs.oracle.com/en-us/iaas/Content/Database/Concepts/adboverview.htm) |
| HeatWave MySQL | 1 instance + 50GB | [HeatWave](https://docs.oracle.com/en-us/iaas/Content/Database/Concepts/mysqloverview.htm) |
| Load Balancer | 1 instance, 10 Mbps | [Load Balancing](https://docs.oracle.com/en-us/iaas/Content/Balance/Concepts/balanceoverview.htm) |
| Monitoring & Alerting | Metrics cơ bản | [OCI Monitoring](https://docs.oracle.com/en-us/iaas/Content/Monitoring/Concepts/monitoringoverview.htm) |
| Terraform stack mẫu | Tạo hạ tầng 1 click | [Get Started Free Tier](https://docs.oracle.com/en/learn/cloud_free_tier/) |
| **Budget alert (⚠️ làm ĐẦU TIÊN)** | Free — cảnh báo khi bị tính tiền | [OCI Budgets](https://docs.oracle.com/en-us/iaas/Content/Billing/Tasks/createbudget.htm) |

## Giải thích từng tài nguyên (dùng để làm gì trong stack này)

- **VPS ARM Ampere A1 — 2 OCPU / 12GB RAM, chạy 24/7**: "trái tim" của kiến trúc — nơi chạy nginx/FastAPI/Ollama/Postgres. Đây là tài nguyên Always Free giá trị nhất (CPU ARM mạnh, RAM đủ cho model ≤3B quantized) và cũng là tài nguyên **hay bị "out of capacity"** nhất khi tạo.
- **VPS AMD nhỏ ×2 (1/8 OCPU + 1GB)**: máy phụ cho cron/backup/monitoring — không đủ chạy LLM nhưng hợp tác vụ nhẹ tách khỏi VPS chính (ví dụ máy chạy rclone đẩy backup nếu VPS chính chết).
- **Block storage 200 GB tổng**: boot volume của các VPS dùng chung hạn mức này — đừng tạo block volume phụ tùy tiện (xem [06-security-backup-and-infra-quirks.md](06-security-backup-and-infra-quirks.md)).
- **Object Storage (S3) 20GB**: kho file + **backup Postgres dump** — backup ra khỏi VPS là điều bắt buộc.
- **Autonomous DB ×2 (20GB mỗi DB)** và **HeatWave MySQL (1 instance + 50GB)**: lựa chọn DB managed thay thế self-host Postgres nếu muốn. Stack mặc định vẫn dùng Postgres+pgvector trên VPS vì cần extension vector.
- **Load Balancer (1 instance, 10 Mbps)**: có sẵn nếu cần LB — nhưng với 1 VPS thì nginx reverse proxy là đủ, LB chỉ thêm khi có nhiều instance.
- **Monitoring & Alerting (OCI Monitoring)**: metrics cơ bản của hạ tầng — bổ sung cho UptimeRobot/Grafana ở tầng ứng dụng.
- **Terraform stack mẫu**: dựng hạ tầng 1 click, hữu ích khi làm lại từ đầu (VPS chết, đổi region phụ…).
- **$300 credit 30 ngày**: dùng để thử nghiệm tài nguyên trả phí (GPU, LB thêm…) — **không thuộc** Always Free, hết 30 ngày là hết. Mọi thứ cần chạy dài hạn phải nằm trong cột Always Free.

## Lưu ý vận hành Oracle (quan trọng)

### 1. Home region: Singapore hoặc Japan Central (Osaka)

Chọn **Singapore** hoặc **Japan Central (Osaka)** — latency từ VN ~30–80ms, đủ nhanh cho chat/realtime. Tránh:

- **Tokyo** — hay báo "out of capacity" cho Ampere A1 (nhiều người chọn, hết chỗ).
- **Chuncheon** — không hỗ trợ Always Free A1.

Kiểm tra capacity thực tế trước khi đăng ký: [cheaploy.com/tools/oracle-capacity-checker](https://cheaploy.com/tools/oracle-capacity-checker)

### 2. Hạn mức hiện tại (7/2026): 2 OCPU / 12GB

Không còn mức 4 OCPU / 24GB như trước — thiết kế budget RAM quanh con số 12GB (xem [05-streaming-timeout-and-ram.md](05-streaming-timeout-and-ram.md)). Chạy 24/7 vẫn nằm trong hạn mức **1.500 OCPU-hours/tháng** (2 OCPU × 24h × 30 ngày = 1.440 OCPU-hours < 1.500 → vừa khít, không dư nhiều).

### 3. Idle reclamation — instance bỏ xó bị thu hồi

Oracle có thể thu hồi instance "idle" (không có traffic/hoạt động) để nhường tài nguyên. Cách phòng: **luôn chạy service có traffic/monitoring đều** — UptimeRobot ping 5 phút + traffic thật là đủ bằng chứng hoạt động. Instance bỏ xó nhiều tuần là có nguy cơ bị thu hồi.

### 4. Không đổi được home region sau khi đăng ký

Home region **cố định trọn đời tài khoản** — chọn sai là phải đăng ký tài khoản mới (và mất mọi resource cũ). Vì vậy bước chọn region ở mục 1 phải làm cẩn thận ngay từ đầu, xem capacity trước.

### 5. Bảo mật & bẫy deploy

Firewall Oracle có 3 lớp (thiếu 1 lớp là "No route to host"), Docker bypass firewall, Cloudflare cắt stream > 100s… — đọc toàn bộ checklist A–G tại [04-pitfalls-billing-and-firewall.md](04-pitfalls-billing-and-firewall.md), [05-streaming-timeout-and-ram.md](05-streaming-timeout-and-ram.md) và [06-security-backup-and-infra-quirks.md](06-security-backup-and-infra-quirks.md) trước khi bật server.

## Quy tắc: budget alert $1 làm ĐẦU TIÊN

> **Việc ĐẦU TIÊN khi có tài khoản** (trước khi tạo bất cứ resource nào): **Governance & Administration → Budgets** → tạo budget `always-free-fail-safe`, cảnh báo ở **$1**. Link: [OCI Budgets](https://docs.oracle.com/en-us/iaas/Content/Billing/Tasks/createbudget.htm)

**Tại sao phải làm đầu tiên?** Trong Always Free, mọi thứ "đúng hạn mức" đều 0đ — nhưng một thao tác sai (tạo NAT Gateway, snapshot lớn, nâng cấp PAYG quên theo dõi…) sẽ **charge thẳng vào thẻ** và bạn chỉ phát hiện khi bị tính tiền. Budget alert $1 là **tấm lưới an toàn duy nhất** báo cho bạn ngay khi có đồng nào phát sinh — trong khi đó các resource tạo xong vẫn đợi được, còn tiền đã charge thì khó đòi. Chi tiết các bẫy billing cụ thể: [04-pitfalls-billing-and-firewall.md](04-pitfalls-billing-and-firewall.md).

## Liên kết

- [01-overall-architecture.md](01-overall-architecture.md) — kiến trúc tổng thể dùng các tài nguyên này như thế nào.
- [04-pitfalls-billing-and-firewall.md](04-pitfalls-billing-and-firewall.md) — AUDIT A (bẫy billing) & AUDIT B (firewall 3 lớp) của Oracle.
- [07-setup-and-pre-flight.md](07-setup-and-pre-flight.md) — thứ tự setup: budget alert → tạo VPS A1 → hardening.
