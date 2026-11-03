# 07 — Deploy 0đ lên Oracle Always Free

> Checklist đầy đủ để deploy 1 sản phẩm GenAI hoàn chỉnh (domain + DB + model) với chi phí 0đ.
> Áp dụng cho mọi project trong lộ trình, đặc biệt là sản phẩm cuối.

---

## 🏗️ Kiến trúc tổng thể

```
User → Cloudflare (DNS + CDN + DDoS + SSL miễn phí)
         ↓
   Oracle Always Free VPS (Ampere A1 — 2 OCPU / 12GB RAM)
   ├── nginx            (HTTPS, reverse proxy)
   ├── App backend      (FastAPI)
   ├── Ollama/llama.cpp (LLM ≤3B Q4, VLM 2B)
   ├── PostgreSQL + pgvector (RAG + dữ liệu)
   └── Langfuse         (observability, optional)
   + Oracle Object Storage 20GB (file, backup)
   + Brevo (email) + UptimeRobot (giám sát)
```

Toàn bộ stack chạy trong 12GB RAM với model ≤3B quantized.

---

## ✅ Checklist từng thành phần (kèm link)

### 1. Hạ tầng — Oracle Always Free (vĩnh viễn)

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

### 2. Domain + SSL + CDN

| Thành phần | Free | Link |
|---|---|---|
| Domain phụ free (`*.is-a.dev`) | Vĩnh viễn | [is-a.dev](https://is-a.dev/) |
| Domain phụ free (`*.eu.org`) | Vĩnh viễn | [eu.org](https://eu.org/) |
| DNS + CDN + DDoS + proxy | Cloudflare free plan | [cloudflare.com](https://www.cloudflare.com/) |
| SSL tự động renew | Let's Encrypt + certbot | [letsencrypt.org](https://letsencrypt.org/) · [certbot.eff.org](https://certbot.eff.org/) |
| Domain riêng (.com) | ❌ Phải mua ~$10/năm | Namecheap / Cloudflare Registrar |

### 3. Email (transactional: verify, reset password...)

| Free | Link |
|---|---|
| Brevo — 300 email/ngày | [brevo.com](https://www.brevo.com/) |
| Resend — 3.000 email/tháng | [resend.com](https://resend.com/) |
| Zoho Mail — email theo domain | [zoho.com/mail](https://www.zoho.com/mail/) |

### 4. Giám sát & vận hành

| Thành phần | Free | Link |
|---|---|---|
| Uptime monitoring | 50 monitors, check 5 phút | [uptimerobot.com](https://uptimerobot.com/) |
| Status page | Better Stack free | [betterstack.com](https://betterstack.com/) |
| Log / metrics | Grafana Cloud free | [grafana.com](https://grafana.com/) |
| Error tracking | Sentry — 5K errors/tháng | [sentry.io](https://sentry.io/) |
| Web analytics | Cloudflare Web Analytics | [cloudflare.com/web-analytics](https://www.cloudflare.com/web-analytics/) |

### 5. DevOps

| Thành phần | Free | Link |
|---|---|---|
| Git + container registry | GitHub + GHCR | [github.com](https://github.com/) |
| CI/CD | GitHub Actions 2.000 phút/tháng | [docs.github.com/en/actions](https://docs.github.com/en/actions) |
| Secrets | GitHub Secrets / Doppler | [doppler.com](https://www.doppler.com/) |

---

## ⚙️ Lưu ý vận hành Oracle (quan trọng)

1. **Home region:** chọn **Singapore** hoặc **Japan Central (Osaka)** — latency từ VN ~30–80ms. Tránh **Tokyo** (hay "out of capacity" cho A1) và **Chuncheon** (không hỗ trợ Always Free A1). Kiểm tra capacity thực tế: [cheaploy.com/tools/oracle-capacity-checker](https://cheaploy.com/tools/oracle-capacity-checker)
2. **Hạn mức hiện tại (7/2026):** 2 OCPU / 12GB (không còn 4 OCPU / 24GB). Chạy 24/7 vẫn nằm trong hạn mức 1.500 OCPU-hours/tháng.
3. **Idle reclamation:** Oracle có thể thu hồi instance bỏ xó → luôn chạy service có traffic/monitoring đều.
4. **Không đổi được home region** sau khi đăng ký → chọn cẩn thận ngay từ đầu.
5. **Bảo mật & bẫy deploy:** xem checklist đầy đủ trong mục A–G bên dưới — firewall Oracle có 3 lớp (thiếu 1 lớp là "No route to host"), Docker bypass firewall, Cloudflare cắt stream > 100s...

---

## 🔴 AUDIT — Cạm bẫy & lưu ý quan trọng khi deploy

> Tổng hợp các lỗi **thật sự xảy ra** với Oracle Always Free + Docker + LLM, đã xác minh từ
> tài liệu chính thức và trường hợp thực tế. Mỗi mục = một "bẫy". Đọc hết trước khi bật server.

### A. Bẫy BILLING — tránh bị charge oan

| Bẫy | Hậu quả | Cách tránh |
|---|---|---|
| Không đặt budget alert | Phát hiện khi đã bị tính tiền | **Việc ĐẦU TIÊN khi có tài khoản:** Governance & Administration → Budgets → tạo `always-free-fail-safe`, cảnh báo ở **$1** |
| **NAT Gateway** | Có thể tính tiền theo giờ (tùy region) | Dùng **public subnet + Internet Gateway** (free) cho VPS có public IP — không tạo NAT Gateway |
| Tạo resource ngoài Always Free (LB thứ 2, snapshot lớn, GPU...) | Charge thẳng vào thẻ | Trước khi tạo bất cứ gì: nó có nhãn **"Always Free-eligible"** không? |
| Nâng cấp Pay As You Go rồi quên | Mọi thứ vượt hạn mức đều tính tiền | Nếu nâng cấp → budget alert là bắt buộc + xem Cost Analysis hàng tuần |
| Object Storage: lưu free nhưng **retrieve từ Archive** tính tiền | Phí bất ngờ khi restore | Backup để ở **Standard** tier, không dùng Archive cho backup cần restore nhanh |
| Outbound > 10 TB/tháng | Tính tiền thêm | App cá nhân hầu như không chạm; để ý nếu host download/seed lớn |

### B. Firewall 3 LỚP — "No route to host" kinh điển của Oracle

Cổng phải mở ở **cả 3 nơi**, thiếu 1 nơi là không truy cập được:

```
Internet → ① OCI Security List / NSG → ② iptables trong VM → ③ service đang nghe
```

1. **OCI Security List/NSG:** ingress 80, 443 (+ 22 giới hạn IP nguồn nếu được).
2. **iptables trong VM (đặc thù Oracle!):** ảnh Ubuntu của Oracle **tắt UFW âm thầm** — bật ufw có vẻ OK nhưng không thật sự hoạt động. Phải sửa `/etc/iptables/rules.v4`:
   ```
   -A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
   -A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
   ```
   rồi `iptables-restore < /etc/iptables/rules.v4`.
3. **Docker bypass firewall:** port publish qua Docker (`-p 8080:80`) **vượt qua mọi rule iptables/ufw** → cổng hở toang internet. Giải pháp:
   - Bind nội bộ: `-p 127.0.0.1:8080:8080` → chỉ nginx truy cập được
   - **Ollama nhất định phải** bind `127.0.0.1:11434` — để public là người lạ dùng model chùa / tấn công

### C. Streaming & timeout — lỗi 524 khi chat với LLM

| Vấn đề | Chi tiết | Giải pháp |
|---|---|---|
| Cloudflare cắt request im lặng > 100–125s | Proxy Read Timeout mặc định → **HTTP 524**; không sửa được ở plan free | ① Bắt buộc **stream SSE** (token về liên tục = không timeout) ② Gửi **SSE keepalive ping** mỗi 15–30s cho đoạn model "nghĩ" lâu ③ Giới hạn `max_tokens` hợp lý |
| nginx buffering nuốt stream | `proxy_buffering on` mặc định | `proxy_buffering off;` + `proxy_read_timeout 300s;` cho route chat |
| Upload PDF lớn bị 413 | `client_max_body_size` mặc định 1MB | `client_max_body_size 50m;` |
| Cần request không-stream > 100s | Cloudflare free không config timeout được | Dùng subdomain **grey cloud** (DNS only, không proxy) cho endpoint đó |

### D. RAM 12GB — budget thực tế (OOM là lỗi #1 với LLM)

| Thành phần | RAM ước tính |
|---|---|
| OS + nginx + systemd | ~0.7 GB |
| PostgreSQL + pgvector | 1–1.5 GB |
| FastAPI + app | 0.3–0.5 GB |
| Qwen2.5-3B Q4 (đã load) | ~2 GB |
| KV cache (tùy `num_ctx`) | 0.3–1 GB |
| SmolVLM2 2.2B (khi load) | ~2.5 GB |
| **Tổng khi LLM + VLM cùng load** | **~8–9 GB** → vừa nhưng không dư dả |

Quy tắc bắt buộc:
- `OLLAMA_MAX_LOADED_MODELS=1` (mặc định) — model thứ 2 sẽ **unload model thứ nhất**; đừng kỳ vọng 2 model cùng ngự trong RAM
- `OLLAMA_KEEP_ALIVE` chỉnh theo traffic (mặc định unload sau 5 phút; `-1` giữ mãi nhưng tốn RAM)
- **`num_ctx` là RAM ngầm:** context 8K–32K tốn thêm GB cho KV cache — đặt 4096 cho chat thường
- **Thêm swap 2–4GB** (`fallocate -l 4G /swapfile`) — cứu khỏi OOM kill cao điểm
- Dùng bản **Q4_K_M** thay vì Q8 — giảm ~2x RAM, chất lượng chấp nhận được

### E. Bảo mật — checklist tối thiểu trước khi public

- [ ] SSH: tắt password auth (`PasswordAuthentication no`), chỉ dùng key
- [ ] **2FA cho Oracle Console** (mất tài khoản cloud = mất VPS)
- [ ] fail2ban + `unattended-upgrades`
- [ ] Ollama/Postgres bind `127.0.0.1`, không expose public
- [ ] **Rate limit** nginx (`limit_req_zone`) cho /api/chat — LLM endpoint là mục tiêu abuse số 1
- [ ] Docker: không `--privileged`, không mount docker.sock vào app container
- [ ] **Log rotate:** Docker log mặc định KHÔNG giới hạn → đặt `log-opts: max-size=10m, max-file=3` trong daemon.json, nếu không disk đầy dần mà chết lặng lẽ

### F. Bền vững — backup & phục hồi

- **Postgres dump hàng ngày → Object Storage** (cron + rclone/aws cli): model mất tải lại được, **mất dữ liệu user thì không**
- Boot volume dùng chung hạn mức 200GB — đừng tạo block volume phụ tùy tiện
- Disk full = app chết lặng lẽ: giám sát dung lượng (`df` trong cron + UptimeRobot)
- Kernel update cần reboot: `needrestart` + reboot trong khung giờ ít user

### G. Đặc thù hạ tầng

| Hạng mục | Lưu ý |
|---|---|
| **ARM64 (aarch64)** | Mọi Docker image phải có bản `arm64` — Ollama có sẵn, nhưng image lạ phải check: `docker manifest inspect` |
| **Email** | Oracle **block outbound port 25** (chống spam) → bắt buộc SMTP provider ngoài (Brevo/Resend) qua cổng 587/465 |
| **Cloudflare SSL** | Đặt **Full (strict)** + "Always Use HTTPS" — chế độ Flexible là "bảo mật giả" (CF→origin vẫn HTTP) |
| **Real IP trong log** | Sau Cloudflare, nginx chỉ thấy IP Cloudflare — thêm `real_ip_header CF-Connecting-IP;` |
| **Out of capacity** | Retry nhiều lần / thử Availability Domain khác; theo dõi capacity: [cheaploy.com/tools/oracle-capacity-checker](https://cheaploy.com/tools/oracle-capacity-checker) |
| **Idle reclamation** | UptimeRobot ping 5 phút + traffic thật = an toàn; instance bỏ xó nhiều tuần là có nguy cơ bị thu hồi |

---

## 📝 Thứ tự setup đề xuất (1 buổi)

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

### ✅ PRE-FLIGHT CHECKLIST — trước khi tuyên bố "live"

1. ✅ Budget alert $1 đã tạo và hoạt động
2. ✅ `curl` từ máy ngoài vào domain → 200 qua HTTPS
3. ✅ SSL test (ssllabs.com) đạt A
4. ✅ Port scan từ máy khác: chỉ 80/443/22 mở
5. ✅ Kill thử Ollama → service tự restart (`restart: always`)
6. ✅ Upload PDF 20MB → hỏi → streaming mượt, KHÔNG bị 524
7. ✅ Backup Postgres đã **restore thử** thành công ở máy local (backup không restore = không có backup)
8. ✅ UptimeRobot + Sentry/Langfuse đã nhận dữ liệu thật

---

## 🔗 Chuyển tiếp
Sản phẩm cuối cần deploy theo đúng checklist này → xem [08-FINAL-PROJECT](../08-FINAL-PROJECT/README.md).
