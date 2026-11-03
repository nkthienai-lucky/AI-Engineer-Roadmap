# 01 — Kiến trúc tổng thể deploy 0đ

> **Mục tiêu**: hiểu toàn bộ kiến trúc deploy một sản phẩm GenAI hoàn chỉnh (domain + DB + model) với chi phí 0đ, biết vai trò từng thành phần trong stack và hiểu **tại sao** cả stack chạy vừa trong 12GB RAM với model ≤3B quantized — nền tảng để đọc các file "bẫy" phía sau.
>
> Nguồn mở rộng từ: [07-DEPLOYMENT.md](../../../07-DEPLOYMENT.md)

---

## Sơ đồ kiến trúc tổng thể

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

Toàn bộ stack chạy trong 12GB RAM với model ≤3B quantized. Mọi chi phí = 0đ nếu tuân thủ đúng hạn mức Always Free — các file sau giải thích từng ranh giới bị tính tiền.

## Vai trò từng thành phần

### Cloudflare (DNS + CDN + DDoS + SSL miễn phí)

Lớp ngoài cùng mà mọi request phải đi qua. Cloudflare lo **DNS** (trỏ domain về VPS), **CDN** (cache asset tĩnh, giảm tải cho VPS), **DDoS protection** (chặn tấn công trước khi chạm server) và **SSL miễn phí** (mã hoá user → Cloudflare). Không có Cloudflare thì vẫn chạy được, nhưng phải tự lo SSL + tự chịu tấn công. Bẫy lớn nhất của lớp này: **Cloudflare cắt request > 100–125s** (xem [05-streaming-timeout-and-ram.md](05-streaming-timeout-and-ram.md)).

### nginx (HTTPS, reverse proxy)

Cửa vào duy nhất của VPS: nhận traffic 80/443 từ Cloudflare, chấm SSL (Let's Encrypt/certbot) rồi **reverse proxy** vào app backend. Mọi thứ phía sau (FastAPI, Ollama, Postgres) **không expose ra internet** — chỉ nginx nói chuyện với bên ngoài. nginx cũng là nơi đặt rate limit cho `/api/chat`, giới hạn upload PDF và tắt buffering cho streaming.

### App backend (FastAPI)

Ứng dụng chính của bạn — API nghiệp vụ, RAG pipeline, luồng chat. FastAPI chạy nội bộ sau nginx, kết nối Ollama (LLM) và PostgreSQL+pgvector (dữ liệu + vector search). App là thành phần **duy nhất nên public qua nginx**; còn lại bind `127.0.0.1`.

### Ollama / llama.cpp (LLM ≤3B Q4, VLM 2B)

Lớp inference **chạy ngay trên VPS** — không tốn tiền API. Model ≤3B quantized (Q4) là giới hạn thực tế của 12GB RAM: ví dụ Qwen2.5-3B Q4 (~2GB khi load) và SmolVLM2 2.2B (VLM ~2.5GB khi load). Ollama cung cấp API `localhost:11434` cho app gọi. **Phải bind `127.0.0.1`** — để public là người lạ dùng model chùa.

### PostgreSQL + pgvector (RAG + dữ liệu)

Một database phục vụ hai việc: **dữ liệu ứng dụng** (user, lịch sử…) và **RAG** (lưu embedding qua extension `pgvector`, truy vấn top-k vector gần nhất). Gộp chung một Postgres thay vì thêm Vector DB riêng giúp tiết kiệm RAM đáng kể — thêm service là thêm 0.5–1GB.

### Langfuse (observability, optional)

Ghi log/trace các lượt gọi LLM (prompt, latency, token) để debug và đánh giá chất lượng. Thành phần **optional** — nếu RAM căng hoặc chưa cần, có thể bỏ qua hoặc dùng Langfuse Cloud free thay vì tự host.

### Các thành phần ngoài VPS

- **Oracle Object Storage 20GB** — lưu file tĩnh (PDF upload) và **backup** (Postgres dump). Backup ra ngoài VPS là bắt buộc: VPS chết thì dữ liệu user vẫn còn.
- **Brevo (email)** — gửi email transactional (verify, reset password…). Bắt buộc dùng provider ngoài vì **Oracle block outbound port 25**.
- **UptimeRobot (giám sát)** — ping 5 phút một lần: biết service chết sớm + chứng minh instance có traffic thật (chống idle reclamation).

## Tại sao cả stack chạy vừa trong 12GB RAM?

### Model ≤3B quantized là "van an toàn" của RAM

Mỗi tham số model ở FP16 tốn 2 byte → một model 7B "nguyên bản" đã ~14GB, vượt cả cây RAM. **Quantization** nén trọng số xuống 4-bit (Q4): model 3B chỉ còn ~2GB khi load, chất lượng chấp nhận được cho chat/RAG tóm tắt. Đó là lý do kiến trúc chốt ở **LLM ≤3B Q4, VLM 2B** — không phải vì CPU yếu mà vì RAM là tài nguyên khan hiếm duy nhất.

### Budget RAM tổng thể (xem chi tiết ở file 05)

Ước tính nhanh: OS+nginx ~0.7GB, Postgres+pgvector 1–1.5GB, FastAPI 0.3–0.5GB, LLM Q4 ~2GB, KV cache 0.3–1GB, VLM (khi load) ~2.5GB → **~8–9GB khi LLM + VLM cùng load**: vừa nhưng không dư dả. Ba nguyên tắc giữ stack sống:

1. **Chỉ 1 model trong RAM tại một thời điểm** (`OLLAMA_MAX_LOADED_MODELS=1`) — model thứ 2 vào là model thứ nhất bị unload.
2. **`num_ctx` là RAM ngầm** — context 8K–32K tốn thêm GB cho KV cache; đặt 4096 cho chat thường.
3. **Thêm swap 4GB** — phao cứu sinh khi cao điểm, tránh OOM kill.

Chi tiết từng mục + bảng RAM đầy đủ: [05-streaming-timeout-and-ram.md](05-streaming-timeout-and-ram.md).

## Dòng đời một request (ghép các lớp lại)

1. User gọi `https://your-domain` → **Cloudflare** (DNS resolve, CDN cache, DDoS filter, SSL terminate).
2. Cloudflare proxy về **nginx** trên VPS (cổng 443 đã mở ở cả 3 lớp firewall — xem [04-pitfalls-billing-and-firewall.md](04-pitfalls-billing-and-firewall.md)).
3. nginx reverse proxy vào **FastAPI** (bind nội bộ), áp rate limit + streaming config.
4. FastAPI gọi **Ollama** (`127.0.0.1:11434`) để sinh token **stream SSE** về user (bắt buộc để không bị timeout — file 05) và truy vấn **Postgres+pgvector** cho phần RAG.
5. Phụ trợ: **Brevo** gửi email, **UptimeRobot** ping giám sát, **Object Storage** nhận backup/file.

## Liên kết

- [02-oracle-always-free.md](02-oracle-always-free.md) — hạ tầng nền: VPS Ampere A1 2 OCPU/12GB và toàn bộ tài nguyên Always Free.
- [04-pitfalls-billing-and-firewall.md](04-pitfalls-billing-and-firewall.md) — 3 lớp firewall để request ở bước 2 của sơ đồ thật sự đi qua được.
- [05-streaming-timeout-and-ram.md](05-streaming-timeout-and-ram.md) — ngân sách RAM chi tiết và cơ chế streaming giữ stack sống.
- [07-setup-and-pre-flight.md](07-setup-and-pre-flight.md) — lắp toàn bộ sơ đồ trên vào 1 buổi với 8 bước setup.
