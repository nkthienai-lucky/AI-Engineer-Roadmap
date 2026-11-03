# 05 — Streaming, timeout & budget RAM (AUDIT C + D)

> **Mục tiêu**: hiểu vì sao chat với LLM qua Cloudflare hay gặp lỗi **524** và cách cấu hình streaming/nginx để không bao giờ trúng (AUDIT C), cùng ngân sách RAM thực tế của cây 12GB và các quy tắc giữ Ollama không bị OOM kill — lỗi #1 với LLM trên VPS nhỏ (AUDIT D).
>
> Nguồn mở rộng từ: [07-DEPLOYMENT.md](../../../07-DEPLOYMENT.md)

---

## AUDIT C — Streaming & timeout: lỗi 524 khi chat với LLM

| Vấn đề | Chi tiết | Giải pháp |
|---|---|---|
| Cloudflare cắt request im lặng > 100–125s | Proxy Read Timeout mặc định → **HTTP 524**; không sửa được ở plan free | ① Bắt buộc **stream SSE** (token về liên tục = không timeout) ② Gửi **SSE keepalive ping** mỗi 15–30s cho đoạn model "nghĩ" lâu ③ Giới hạn `max_tokens` hợp lý |
| nginx buffering nuốt stream | `proxy_buffering on` mặc định | `proxy_buffering off;` + `proxy_read_timeout 300s;` cho route chat |
| Upload PDF lớn bị 413 | `client_max_body_size` mặc định 1MB | `client_max_body_size 50m;` |
| Cần request không-stream > 100s | Cloudflare free không config timeout được | Dùng subdomain **grey cloud** (DNS only, không proxy) cho endpoint đó |

### Giải thích từng bẫy

**1. Cloudflare cắt request > 100–125s → HTTP 524.** Cloudflare free plan có Proxy Read Timeout ~100–125s: nếu origin (VPS) chưa trả response trong ngưỡng này, Cloudflare **chủ động ngắt** và trả **524** cho client — request của bạn chết mà server vẫn đang sinh token. **Không sửa được timeout này ở plan free**, nên phải thiết kế request không bao giờ "im lặng" quá lâu:

- **Bắt buộc stream SSE** (Server-Sent Events): token về liên tục = mỗi chunk là một lần có data qua proxy → Cloudflare coi request vẫn hoạt động, không timeout. Đây là lý do mọi route chat LLM phải stream.
- **SSE keepalive ping** mỗi **15–30s**: trong các đoạn model "nghĩ" lâu (suy luận dài trước khi ra token đầu), vẫn gửi comment/ping SSE đều đặn để giữ kết nối sống.
- **Giới hạn `max_tokens` hợp lý**: output sinh vô hạn = thời gian vô hạn = vẫn có thể chạm timeout ở đoạn giữa; chặn sớm bằng trần token.

**2. nginx buffering nuốt stream.** Mặc định `proxy_buffering on` — nginx **gom response vào buffer** rồi mới trả client → token không đến realtime, và worse, stream bị "đứng" tới khi buffer đầy. Chống lại bằng config cho route chat:

```nginx
location /api/chat {
    proxy_pass http://127.0.0.1:8080;
    proxy_buffering off;
    proxy_read_timeout 300s;
    client_max_body_size 50m;
}
```

**3. Upload PDF lớn bị 413.** `client_max_body_size` mặc định chỉ **1MB** — upload file RAG trên 1MB là nginx trả **413 Request Entity Too Large**. Đặt `client_max_body_size 50m;` (đã có trong snippet trên) cho route upload.

**4. Request không-stream > 100s (batch, export…).** Cloudflare free không config được timeout → dùng subdomain **grey cloud** (DNS only, **không proxy** — tắt đám mây cam) trỏ thẳng về IP VPS cho endpoint đó: request đi thẳng, không qua proxy Cloudflare, không bị cắt. Đổi lại: mất DDoS/CDN của Cloudflare cho subdomain đó — chấp nhận được với endpoint nội bộ/cron.

## AUDIT D — RAM 12GB: budget thực tế (OOM là lỗi #1 với LLM)

| Thành phần | RAM ước tính |
|---|---|
| OS + nginx + systemd | ~0.7 GB |
| PostgreSQL + pgvector | 1–1.5 GB |
| FastAPI + app | 0.3–0.5 GB |
| Qwen2.5-3B Q4 (đã load) | ~2 GB |
| KV cache (tùy `num_ctx`) | 0.3–1 GB |
| SmolVLM2 2.2B (khi load) | ~2.5 GB |
| **Tổng khi LLM + VLM cùng load** | **~8–9 GB** → vừa nhưng không dư dả |

**Tại sao OOM là lỗi #1?** Với LLM, RAM là tài nguyên tranh chấp gay gắt nhất: model load vào RAM "giữ chỗ" hàng GB, trong khi Postgres/app/OS cũng cần phần của chúng. Khi tổng vượt 12GB, kernel **OOM kill** process nặng nhất (thường là Ollama hoặc Postgres) → service chết lặng lẽ. Budget ~8–9GB ở bảng trên **vừa nhưng không dư dả** — mọi thứ thêm vào (model to hơn, context dài hơn) đều phải trừ vào chỗ trống ít ỏi.

### Quy tắc bắt buộc

- **`OLLAMA_MAX_LOADED_MODELS=1`** (mặc định) — model thứ 2 vào RAM sẽ **unload model thứ nhất**; đừng kỳ vọng 2 model cùng ngự trong RAM. Thiết kế app quanh giả định này: hoặc LLM, hoặc VLM, luân phiên.
- **`OLLAMA_KEEP_ALIVE`** chỉnh theo traffic — mặc định model bị unload sau 5 phút không dùng (tiết kiệm RAM nhưng load lại chậm); `-1` giữ model mãi trong RAM (nhanh nhưng tốn ~2GB thường trực). Traffic đều → giữ lâu; traffic thưa → để mặc định.
- **`num_ctx` là RAM ngầm:** KV cache của context 8K–32K tốn thêm **GB** cho attention — đặt **4096 cho chat thường**, chỉ nâng khi use case thật sự cần context dài.
- **Thêm swap 2–4GB** (`fallocate -l 4G /swapfile`) — phao cứu sinh khi cao điểm, cứu khỏi OOM kill (chậm hơn RAM nhưng thà chậm còn hơn chết):

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
# thêm vào /etc/fstab để giữ sau reboot:
# /swapfile none swap sw 0 0
```

- Dùng bản **Q4_K_M** thay vì Q8 — giảm ~2x RAM, chất lượng chấp nhận được cho chat/RAG tóm tắt (lý do kiến trúc chốt model ≤3B quantized: [01-overall-architecture.md](01-overall-architecture.md)).

## Liên kết

- [01-overall-architecture.md](01-overall-architecture.md) — tại sao kiến trúc chốt model ≤3B Q4 trên cây 12GB.
- [04-pitfalls-billing-and-firewall.md](04-pitfalls-billing-and-firewall.md) — AUDIT A & B: bẫy billing và firewall 3 lớp (điều kiện tiên quyết trước khi lo timeout).
- [06-security-backup-and-infra-quirks.md](06-security-backup-and-infra-quirks.md) — AUDIT E/F/G: bảo mật, backup, đặc thù hạ tầng.
- [07-setup-and-pre-flight.md](07-setup-and-pre-flight.md) — checklist test streaming: upload PDF 20MB → hỏi → không bị 524.
