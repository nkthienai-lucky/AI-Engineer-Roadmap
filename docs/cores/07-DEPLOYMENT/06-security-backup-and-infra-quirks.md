# 06 — Bảo mật, bền vững & đặc thù hạ tầng (AUDIT E + F + G)

> **Mục tiêu**: hoàn tất checklist bảo mật tối thiểu trước khi public (AUDIT E), thiết kế backup & phục hồi để mất VPS không mất dữ liệu (AUDIT F), và nắm các đặc thù "không giống ai" của hạ tầng Oracle/ARM/Cloudflare khiến deploy thất bại nếu không biết trước (AUDIT G).
>
> Nguồn mở rộng từ: [07-DEPLOYMENT.md](../../../07-DEPLOYMENT.md)

---

## AUDIT E — Bảo mật: checklist tối thiểu trước khi public

- [ ] **SSH: tắt password auth** (`PasswordAuthentication no`), chỉ dùng key. *Tại sao:* bot internet dò password SSH 24/7; password yếu là mất server. Key-only = không còn bề mặt tấn công brute-force.
- [ ] **2FA cho Oracle Console.** *Tại sao:* **mất tài khoản cloud = mất VPS** — kẻ có tài khoản console có thể xóa/mở lại toàn bộ hạ tầng, đổi firewall, lấy object storage. 2FA là lớp giữ tài khoản khi mật khẩu bị lộ.
- [ ] **fail2ban + `unattended-upgrades`.** *Tại sao:* fail2ban chặn IP dò SSH sau vài lần sai; `unattended-upgrades` tự vá lỗ hổng bảo mật kernel/OS mà không cần bạn nhớ. Server internet phải tự phòng thủ, không chờ người vào cập nhật tay.
- [ ] **Ollama/Postgres bind `127.0.0.1`**, không expose public. *Tại sao:* Ollama không có auth — để public là người lạ dùng model chùa; Postgres public là mất dữ liệu user (xem chi tiết bind trong [04-pitfalls-billing-and-firewall.md](04-pitfalls-billing-and-firewall.md)).
- [ ] **Rate limit nginx (`limit_req_zone`) cho /api/chat** — LLM endpoint là mục tiêu abuse số 1. *Tại sao:* mỗi request chat tốn CPU/RAM inference; bot spam có thể biến VPS thành "mỏ đào" cho người lạ hoặc sập service bằng quá tải:

```nginx
http {
    limit_req_zone $binary_remote_addr zone=chat:10m rate=5r/m;
}
location /api/chat {
    limit_req zone=chat burst=10 nodelay;
}
```

- [ ] **Docker: không `--privileged`, không mount docker.sock vào app container.** *Tại sao:* `--privileged` và docker.sock cho phép container **thoát ly sandbox, kiểm soát cả host** — một lỗ hổng app là mất toàn bộ VPS.
- [ ] **Log rotate:** Docker log mặc định KHÔNG giới hạn → đặt `log-opts: max-size=10m, max-file=3` trong daemon.json, nếu không disk đầy dần mà chết lặng lẽ. *Tại sao:* log chat/LLM sinh rất nhanh; disk đầy 100% = Postgres ghi không được = app chết mà không báo lỗi rõ ràng.

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

## AUDIT F — Bền vững: backup & phục hồi

- **Postgres dump hàng ngày → Object Storage** (cron + rclone/aws cli). *Tại sao:* model mất tải lại được, **mất dữ liệu user thì không** — backup là thứ duy nhất phân biệt "sự cố nhỏ" với "thảm hoạ". Object Storage nằm ngoài VPS nên VPS chết/cháy thì dữ liệu vẫn còn (dùng 20GB free, xem [02-oracle-always-free.md](02-oracle-always-free.md)):

```bash
# cron hằng ngày — dump rồi đẩy lên Object Storage
pg_dump "$DATABASE_URL" | gzip > /backup/db_$(date +%F).sql.gz
rclone copy /backup/db_$(date +%F).sql.gz oracle-objstore:backups/
```

- **Boot volume dùng chung hạn mức 200GB** — đừng tạo block volume phụ tùy tiện. *Tại sao:* mọi block volume cộng dồn vào hạn mức 200GB Always Free; vượt là bị tính tiền (bẫy billing — [04-pitfalls-billing-and-firewall.md](04-pitfalls-billing-and-firewall.md)).
- **Disk full = app chết lặng lẽ:** giám sát dung lượng (`df` trong cron + UptimeRobot). *Tại sao:* disk đầy không có "lỗi" nào rõ ràng — Postgres/ Docker từ từ ngừng ghi, app treo. Biểu thức `df` trong cron gửi cảnh báo (hoặc UptimeRobot keyword monitor) là hệ thống báo sớm rẻ nhất.
- **Kernel update cần reboot:** `needrestart` + reboot trong khung giờ ít user. *Tại sao:* vá kernel chỉ hiệu lực sau reboot; để nguyên là server chạy kernel cũ có lỗ hổng — nhưng reboot ban ngày thì downtime. `needrestart` báo khi nào cần reboot; canh giờ vắng để reboot.

## AUDIT G — Đặc thù hạ tầng (không biết là deploy fail)

| Hạng mục | Lưu ý |
|---|---|
| **ARM64 (aarch64)** | Mọi Docker image phải có bản `arm64` — Ollama có sẵn, nhưng image lạ phải check: `docker manifest inspect` |
| **Email** | Oracle **block outbound port 25** (chống spam) → bắt buộc SMTP provider ngoài (Brevo/Resend) qua cổng 587/465 |
| **Cloudflare SSL** | Đặt **Full (strict)** + "Always Use HTTPS" — chế độ Flexible là "bảo mật giả" (CF→origin vẫn HTTP) |
| **Real IP trong log** | Sau Cloudflare, nginx chỉ thấy IP Cloudflare — thêm `real_ip_header CF-Connecting-IP;` |
| **Out of capacity** | Retry nhiều lần / thử Availability Domain khác; theo dõi capacity: [cheaploy.com/tools/oracle-capacity-checker](https://cheaploy.com/tools/oracle-capacity-checker) |
| **Idle reclamation** | UptimeRobot ping 5 phút + traffic thật = an toàn; instance bỏ xó nhiều tuần là có nguy cơ bị thu hồi |

### Giải thích từng đặc thù

- **ARM64 (aarch64):** VPS Ampere A1 là máy **ARM** — image Docker chỉ build cho `amd64` sẽ fail khi pull/run ("no matching manifest"). Ollama đã có bản arm64; image lạ phải kiểm tra trước: `docker manifest inspect <image>` (hoặc build multi-arch trong GitHub Actions — [03-domain-ssl-email-monitoring-devops.md](03-domain-ssl-email-monitoring-devops.md)).
- **Email port 25 bị block:** Oracle chặn outbound port 25 để chống spam → VPS **không gửi SMTP trực tiếp** được, bắt buộc provider ngoài qua cổng **587/465** (Brevo/Resend — [03-domain-ssl-email-monitoring-devops.md](03-domain-ssl-email-monitoring-devops.md)).
- **Cloudflare SSL Full (strict):** chế độ **Flexible** mã hoá user→Cloudflare nhưng **CF→origin vẫn HTTP** — "bảo mật giả", attacker chen giữa CF và VPS đọc được hết. Đặt **Full (strict)** (bắt buộc origin có cert hợp lệ — Let's Encrypt/certbot) + bật "Always Use HTTPS".
- **Real IP trong log:** sau Cloudflare, mọi kết nối tới nginx mang IP của Cloudflare → log/rate limit/fail2ban nhầm người. Thêm `real_ip_header CF-Connecting-IP;` (kèm dải IP Cloudflare vào `set_real_ip_from`) để nginx lấy lại IP thật của client.
- **Out of capacity:** tạo Ampere A1 ở region đông người (đặc biệt Tokyo) hay fail "Out of capacity" → **retry nhiều lần / thử Availability Domain khác**, và theo dõi capacity: [cheaploy.com/tools/oracle-capacity-checker](https://cheaploy.com/tools/oracle-capacity-checker).
- **Idle reclamation:** UptimeRobot ping 5 phút + traffic thật = bằng chứng instance hoạt động → an toàn; instance bỏ xó nhiều tuần là có nguy cơ bị thu hồi (xem [02-oracle-always-free.md](02-oracle-always-free.md)).

## Liên kết

- [04-pitfalls-billing-and-firewall.md](04-pitfalls-billing-and-firewall.md) — AUDIT A & B: bind 127.0.0.1 chi tiết, bẫy billing quanh block storage.
- [05-streaming-timeout-and-ram.md](05-streaming-timeout-and-ram.md) — AUDIT C & D: streaming và budget RAM (khi service đã an toàn thì lo chống sập).
- [07-setup-and-pre-flight.md](07-setup-and-pre-flight.md) — bước hardening + swap trong setup, và checklist restore thử backup trước khi live.
