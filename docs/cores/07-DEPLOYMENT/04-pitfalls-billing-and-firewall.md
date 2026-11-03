# 04 — Bẫy billing & Firewall 3 lớp (AUDIT A + B)

> **Mục tiêu**: tránh bị charge oan khi dùng Oracle Always Free (AUDIT A — 6 bẫy billing kinh điển) và hiểu tại sao mở cổng trên Oracle luôn là "No route to host" cho đến khi mở ở **đủ 3 lớp firewall** (AUDIT B) — bao gồm đặc thù Oracle tắt UFW âm thầm và Docker bypass mọi rule iptables.
>
> Nguồn mở rộng từ: [07-DEPLOYMENT.md](../../../07-DEPLOYMENT.md)

> Tổng hợp các lỗi **thật sự xảy ra** với Oracle Always Free + Docker + LLM, đã xác minh từ tài liệu chính thức và trường hợp thực tế. Mỗi mục = một "bẫy". Đọc hết trước khi bật server.

---

## AUDIT A — Bẫy BILLING: tránh bị charge oan

| Bẫy | Hậu quả | Cách tránh |
|---|---|---|
| Không đặt budget alert | Phát hiện khi đã bị tính tiền | **Việc ĐẦU TIÊN khi có tài khoản:** Governance & Administration → Budgets → tạo `always-free-fail-safe`, cảnh báo ở **$1** |
| **NAT Gateway** | Có thể tính tiền theo giờ (tùy region) | Dùng **public subnet + Internet Gateway** (free) cho VPS có public IP — không tạo NAT Gateway |
| Tạo resource ngoài Always Free (LB thứ 2, snapshot lớn, GPU...) | Charge thẳng vào thẻ | Trước khi tạo bất cứ gì: nó có nhãn **"Always Free-eligible"** không? |
| Nâng cấp Pay As You Go rồi quên | Mọi thứ vượt hạn mức đều tính tiền | Nếu nâng cấp → budget alert là bắt buộc + xem Cost Analysis hàng tuần |
| Object Storage: lưu free nhưng **retrieve từ Archive** tính tiền | Phí bất ngờ khi restore | Backup để ở **Standard** tier, không dùng Archive cho backup cần restore nhanh |
| Outbound > 10 TB/tháng | Tính tiền thêm | App cá nhân hầu như không chạm; để ý nếu host download/seed lớn |

### Giải thích từng bẫy (tại sao trúng và cách phòng)

1. **Không đặt budget alert** — Always Free không có "hard limit": bạn vượt hạn mức là Oracle tính tiền bình thường, không chặn lại. Không alert thì chỉ phát hiện khi sao kê thẻ. → **Budget alert $1 (`always-free-fail-safe`) là việc đầu tiên**, chi tiết tại [02-oracle-always-free.md](02-oracle-always-free.md).
2. **NAT Gateway** — nhiều hướng dẫn VPC mặc định tạo NAT Gateway để private subnet ra internet; trên OCI NAT Gateway **có thể tính tiền theo giờ (tùy region)**. Stack 1 VPS không cần NAT: dùng **public subnet + Internet Gateway** (free) cho VPS có public IP — nginx là cửa vào duy nhất.
3. **Resource ngoài Always Free** — LB thứ 2, snapshot lớn, GPU, shape vượt hạn mức… đều charge thẳng vào thẻ mà không cảnh báo. Nguyên tắc: **trước khi tạo bất cứ gì, kiểm tra nhãn "Always Free-eligible"** trong OCI Console (hoặc đối chiếu bảng [02-oracle-always-free.md](02-oracle-always-free.md)).
4. **Nâng cấp Pay As You Go (PAYG)** — nâng cấp để dùng tài nguyên ngoài free tier thì mọi thứ vượt hạn mức **đều tính tiền**. Nếu đã nâng cấp: budget alert là **bắt buộc** + vào **Cost Analysis** xem hàng tuần để phát hiện charge bất thường sớm.
5. **Retrieve từ Archive tier** — lưu Object Storage ở Archive thì rẻ/free, nhưng **khi retrieve (restore) ra sẽ bị tính tiền**. Backup phục vụ khôi phục nhanh phải để ở **Standard** tier — đừng "tiết kiệm" Archive cho backup cần restore nhanh (mục đích của backup là restore!).
6. **Outbound > 10 TB/tháng** — vượt thì bị tính tiền thêm. App cá nhân/chatbot hầu như không chạm ngưỡng này; chỉ để ý nếu VPS host download/seed/transfer lớn.

## AUDIT B — Firewall 3 LỚP: "No route to host" kinh điển của Oracle

Cổng phải mở ở **cả 3 nơi**, thiếu 1 nơi là không truy cập được:

```
Internet → ① OCI Security List / NSG → ② iptables trong VM → ③ service đang nghe
```

Đây là lý do phổ biến nhất khiến "server mới tạo mà curl không vào được" — người mở Security List rồi tưởng xong, trong khi iptables trong VM vẫn chặn (hoặc service chưa nghe đúng cổng).

### ① OCI Security List / NSG (tầng cloud)

Ingress mở **80, 443** (+ **22 giới hạn IP nguồn** nếu được — chỉ cho IP nhà/văn phòng SSH, không mở 22 cho cả internet). Đây là firewall cấp tenancy, chặn trước khi packet tới VPS.

### ② iptables trong VM (đặc thù Oracle!)

Ảnh Ubuntu của Oracle **tắt UFW âm thầm** — bật `ufw` có vẻ OK nhưng **không thật sự hoạt động** (rule không được enforce như mong đợi). Phải sửa trực tiếp `/etc/iptables/rules.v4`:

```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
```

rồi áp dụng:

```bash
sudo iptables-restore < /etc/iptables/rules.v4
```

**Tại sao phải sửa file rules.v4 thay vì chỉ `iptables -A`?** Rule thêm bằng lệnh chỉ sống tới khi reboot; `/etc/iptables/rules.v4` là file được restore khi boot — sửa file thì rule sống sót qua reboot. Thiếu bước này: reboot máy là "No route to host" quay lại.

### ③ Service đang nghe (tầng application)

Ổn định nhất: port publish qua Docker (`-p 8080:80`) **vượt qua mọi rule iptables/ufw** → cổng hở toang internet mà `iptables -L` vẫn "có vẻ" chặn. Nguyên nhân: Docker tự thêm rule vào chuỗi `DOCKER`/`NAT` của iptables, ưu tiên hơn rule INPUT bạn viết. Giải pháp:

- **Bind nội bộ**: `-p 127.0.0.1:8080:8080` → chỉ nginx (cùng máy) truy cập được, internet không thấy.
- **Ollama nhất định phải** bind `127.0.0.1:11434` — để public là người lạ dùng model chùa / tấn công qua API Ollama không có auth.

```yaml
# docker-compose.yml — bind nội bộ cho app/LLM, chỉ nginx public
services:
  ollama:
    ports:
      - "127.0.0.1:11434:11434"
  app:
    ports:
      - "127.0.0.1:8080:8080"
  nginx:
    ports:
      - "80:80"
      - "443:443"
```

Kiểm tra từ máy ngoài sau khi mở đủ 3 lớp: chỉ **80/443/22** mở, `curl https://your-domain` trả 200 (mục PRE-FLIGHT tại [07-setup-and-pre-flight.md](07-setup-and-pre-flight.md)).

## Liên kết

- [02-oracle-always-free.md](02-oracle-always-free.md) — bảng Always Free đầy đủ và quy tắc budget alert $1 làm đầu tiên.
- [05-streaming-timeout-and-ram.md](05-streaming-timeout-and-ram.md) — AUDIT C & D: timeout streaming và budget RAM (hai "bẫy" tiếp theo sau khi server đã mở được cổng).
- [06-security-backup-and-infra-quirks.md](06-security-backup-and-infra-quirks.md) — AUDIT E/F/G: hardening, backup và đặc thù hạ tầng.
- [07-setup-and-pre-flight.md](07-setup-and-pre-flight.md) — bước 4 mở firewall 3 lớp trong thứ tự setup và checklist port scan.
