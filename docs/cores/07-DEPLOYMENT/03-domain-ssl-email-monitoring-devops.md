# 03 — Domain, SSL, Email, Giám sát & DevOps

> **Mục tiêu**: biết chọn đúng công cụ free cho 4 tầng phụ trợ của stack — domain + SSL/CDN (Mục 2), email transactional (Mục 3), giám sát & vận hành (Mục 4), DevOps/CI-CD (Mục 5) — kèm lý do tại sao mỗi lựa chọn là cần thiết và các con số hạn mức free phải nhớ.
>
> Nguồn mở rộng từ: [07-DEPLOYMENT.md](../../../07-DEPLOYMENT.md)

---

## Mục 2 — Domain + SSL + CDN

| Thành phần | Free | Link |
|---|---|---|
| Domain phụ free (`*.is-a.dev`) | Vĩnh viễn | [is-a.dev](https://is-a.dev/) |
| Domain phụ free (`*.eu.org`) | Vĩnh viễn | [eu.org](https://eu.org/) |
| DNS + CDN + DDoS + proxy | Cloudflare free plan | [cloudflare.com](https://www.cloudflare.com/) |
| SSL tự động renew | Let's Encrypt + certbot | [letsencrypt.org](https://letsencrypt.org/) · [certbot.eff.org](https://certbot.eff.org/) |
| Domain riêng (.com) | ❌ Phải mua ~$10/năm | Namecheap / Cloudflare Registrar |

### Giải thích

- **Domain phụ free (`*.is-a.dev`, `*.eu.org`)**: thay vì mua `.com` (~$10/năm), đăng ký domain phụ miễn phí, **vĩnh viễn** — đủ cho demo/portfolio/sản phẩm cá nhân. Khi nào cần thương hiệu riêng mới trả tiền cho domain riêng (Namecheap / Cloudflare Registrar).
- **Cloudflare free plan**: một chỗ lo trọn **DNS** (trỏ domain về IP VPS), **CDN** (cache asset tĩnh, giảm tải VPS), **DDoS protection** và **proxy** (che IP thật của VPS). Đây là lớp ngoài cùng trong kiến trúc [01-overall-architecture.md](01-overall-architecture.md).
- **Let's Encrypt + certbot**: chứng chỉ SSL miễn phí, **tự động renew** (certbot đặt cron/systemd timer gia hạn trước khi hết hạn) — không phải lo gia hạn tay. Lưu ý bảo mật SSL: đặt chế độ **Full (strict)** bên Cloudflare, xem [06-security-backup-and-infra-quirks.md](06-security-backup-and-infra-quirks.md).
- **Bẫy timeout của Cloudflare**: proxy free plan cắt request > 100–125s → chat LLM phải stream SSE, xem chi tiết [05-streaming-timeout-and-ram.md](05-streaming-timeout-and-ram.md).

## Mục 3 — Email (transactional: verify, reset password...)

| Free | Link |
|---|---|
| Brevo — 300 email/ngày | [brevo.com](https://www.brevo.com/) |
| Resend — 3.000 email/tháng | [resend.com](https://resend.com/) |
| Zoho Mail — email theo domain | [zoho.com/mail](https://www.zoho.com/mail/) |

### Giải thích

- **Brevo — 300 email/ngày**: mặc định của stack (kiến trúc dùng Brevo) — 300/ngày là **dư cho app cá nhân** (verify, reset password…). Chia theo ngày phù hợp traffic đều.
- **Resend — 3.000 email/tháng**: hạn mức theo tháng thay vì theo ngày — hợp app có ngày cao điểm (3.000 dồn vào vài ngày vẫn OK), API dev-friendly.
- **Zoho Mail — email theo domain**: dùng khi cần **hộp mail theo domain** (info@your-domain) thay vì chỉ gửi transactional.
- **Tại sao bắt buộc provider ngoài?** Oracle **block outbound port 25** (chống spam) → VPS **không tự gửi email SMTP trực tiếp** được. Bắt buộc dùng SMTP provider ngoài (Brevo/Resend) qua cổng **587/465**. Đây là đặc thù hạ tầng được lặp lại trong [06-security-backup-and-infra-quirks.md](06-security-backup-and-infra-quirks.md).

## Mục 4 — Giám sát & vận hành

| Thành phần | Free | Link |
|---|---|---|
| Uptime monitoring | 50 monitors, check 5 phút | [uptimerobot.com](https://uptimerobot.com/) |
| Status page | Better Stack free | [betterstack.com](https://betterstack.com/) |
| Log / metrics | Grafana Cloud free | [grafana.com](https://grafana.com/) |
| Error tracking | Sentry — 5K errors/tháng | [sentry.io](https://sentry.io/) |
| Web analytics | Cloudflare Web Analytics | [cloudflare.com/web-analytics](https://www.cloudflare.com/web-analytics/) |

### Giải thích

- **UptimeRobot — 50 monitors, check 5 phút**: ping endpoint mỗi 5 phút, chết là biết ngay. Song song tác dụng phụ quan trọng trên Oracle: **traffic monitor đều = bằng chứng instance hoạt động**, chống idle reclamation (xem [02-oracle-always-free.md](02-oracle-always-free.md)). Tận dụng cả cho giám sát disk (`df` trong cron — [06-security-backup-and-infra-quirks.md](06-security-backup-and-infra-quirks.md)).
- **Better Stack free**: dựng **status page** công khai (user biết service đang chết hay sống) — chuyên nghiệp hơn mà vẫn 0đ.
- **Grafana Cloud free**: log/metrics tập trung — thay vì SSH vào VPS đọc log thủ công. (Tự host Grafana trên VPS cũng được nhưng tốn RAM — budget RAM rất căng, xem file 05.)
- **Sentry — 5K errors/tháng**: **error tracking** cho app — bắt exception phía backend + frontend, báo ngay khi có lỗi thay vì user phàn nàn mới biết. 5K errors/tháng đủ cho mọi app trong lộ trình.
- **Cloudflare Web Analytics**: thống kê truy cập web (pageview, nguồn…) **không cần thêm script bên thứ ba nặng** — đã có Cloudflare sẵn.

## Mục 5 — DevOps

| Thành phần | Free | Link |
|---|---|---|
| Git + container registry | GitHub + GHCR | [github.com](https://github.com/) |
| CI/CD | GitHub Actions 2.000 phút/tháng | [docs.github.com/en/actions](https://docs.github.com/en/actions) |
| Secrets | GitHub Secrets / Doppler | [doppler.com](https://www.doppler.com/) |

### Giải thích

- **GitHub + GHCR (GitHub Container Registry)**: code trên GitHub, **Docker image build xong đẩy lên GHCR** — VPS chỉ cần `docker pull` + `docker compose up`, không cần build trên máy yếu. Registry riêng của GitHub, free cho public repo.
- **GitHub Actions — 2.000 phút/tháng**: CI/CD free — push code là tự build image, push GHCR, thậm chí deploy qua SSH. 2.000 phút/tháng thoải mái cho project cá nhân (lưu ý: build image ARM64 cho VPS — xem [06-security-backup-and-infra-quirks.md](06-security-backup-and-infra-quirks.md)).
- **GitHub Secrets / Doppler**: chỗ chứa biến bí mật (API key, DB password, SMTP password) — **không commit secret vào repo**. GitHub Secrets đủ dùng cho Actions; [Doppler](https://www.doppler.com/) đồng bộ secret ra nhiều môi trường khi project lớn hơn.

## Liên kết

- [01-overall-architecture.md](01-overall-architecture.md) — vị trí của Cloudflare/email/giám sát trong kiến trúc tổng thể.
- [05-streaming-timeout-and-ram.md](05-streaming-timeout-and-ram.md) — bẫy Cloudflare timeout (524) và cách cấu hình streaming.
- [06-security-backup-and-infra-quirks.md](06-security-backup-and-infra-quirks.md) — SSL Full (strict), email port 25, Docker arm64, giám sát disk.
- [07-setup-and-pre-flight.md](07-setup-and-pre-flight.md) — bước cài nginx + certbot và checklist SSL test trước khi live.
