# 06 — Production hóa + Capstone (Tuần 18)

> **Mục tiêu:** Đưa toàn bộ stack từ "chạy được trên máy dev" lên production thật: Docker hóa, deploy Oracle Always Free, CI/CD GitHub Actions, load test nhẹ, tối ưu hiệu năng (KV cache, batching, caching semantic), và hoàn thiện README + demo video cho portfolio. Project GĐ5 = **capstone** — sản phẩm cuối của cả lộ trình.
>
> Nguồn mở rộng từ: [05-AGENTS-PRODUCTION.md](../../../05-AGENTS-PRODUCTION.md)

---

## Docker hóa toàn bộ stack

Mục tiêu: **một lệnh là dựng lại toàn bộ hệ thống** trên máy bất kỳ — hết "chạy được trên máy mình". Stack cần đóng container: app backend (FastAPI), Ollama/llama.cpp (LLM ≤3B quantized), PostgreSQL + pgvector, nginx reverse proxy — đúng các thành phần trong [kiến trúc tổng thể deploy 0đ](../07-DEPLOYMENT/01-overall-architecture.md).

```yaml
# docker-compose.yml — khung sườn (rút gọn)
services:
  app:        { build: ./app, ports: ["8000:8000"], env_file: .env }
  db:         { image: pgvector/pgvector:pg16, volumes: ["pgdata:/var/lib/postgresql/data"] }
  ollama:     { image: ollama/ollama, volumes: ["ollama:/root/.ollama"] }
  nginx:      { image: nginx, ports: ["443:443"], depends_on: [app] }
volumes: { pgdata: {}, ollama: {} }
```

Nguyên tắc: **volume cho dữ liệu bền** (DB, model weights — mất là phải tải lại vài GB), `.env` cho bí mật (không commit), healthcheck cho từng service, và một `Makefile`/script `deploy.sh` để dựng stack bằng đúng một lệnh.

## Deploy lên Oracle Always Free

Triển khai đúng kiến trúc 0đ đã học ở [07-DEPLOYMENT/01-overall-architecture.md](../07-DEPLOYMENT/01-overall-architecture.md): VPS Ampere A1 (2 OCPU / 12GB RAM) chạy toàn bộ stack, Cloudflare lo DNS + CDN + DDoS + SSL free, Oracle Object Storage 20GB cho file/backup. Lưu ý đặc thù tuần này: **12GB RAM là ràng buộc cứng** — LLM ≤3B quantized + Postgres + app + monitoring phải chia nhau chỗ đứng; container nào không dùng thì tắt (ví dụ Langfuse Cloud thay vì self-host — file 05). Chi tiết các bẫy billing/timeout/RAM nằm ở những file "bẫy" trong folder 07-DEPLOYMENT.

## CI/CD cơ bản với GitHub Actions

CI/CD = mỗi lần `git push`, pipeline tự: **test → build image → deploy**. Tối thiểu nên có:

1. **Lint + test** — chạy pytest, ruff/flake8; fail thì dừng, không deploy code hỏng.
2. **Eval regression** (gắn từ file 05) — chạy golden set, so sánh ngưỡng chất lượng; prompt/model đổi mà chất lượng tụt thì **chặn deploy**.
3. **Build & push image** — build Docker image, tag theo commit.
4. **Deploy** — SSH vào VPS, kéo image mới, `docker compose up -d` (rolling: không tắt service cũ khi service mới chưa healthy).

```yaml
# .github/workflows/deploy.yml — khung sườn
jobs:
  test:    { steps: [checkout, setup-python, pip install -r requirements.txt, pytest] }
  deploy:  { needs: test, steps: [build-push-image, ssh: "cd /app && docker compose pull && docker compose up -d"] }
```

## Load test nhẹ & tối ưu hiệu năng

**Load test nhẹ** (Locust/k6) trước khi mở: mô phỏng 10–30 user đồng thời trong vài phút, đo latency P95, RAM, lỗi rate — mục tiêu không phải "chống DDOS" mà là **tìm điểm vỡ của 12GB RAM** trước khi user thật tìm thấy. Ba kỹ thuật tối ưu cần biết:

- **KV cache** — cache key/value của các token đã tính trong attention; model serving (llama.cpp/Ollama) bật sẵn — nếu tắt, mỗi token sinh ra phải tính lại toàn bộ prompt → chậm gấp nhiều lần. Lợi nhất khi prompt dài lặp lại (system prompt + RAG context).
- **Batching** — gom nhiều request xử lý cùng lượt qua GPU/CPU → tăng throughput, đánh đổi latency mỗi request tăng nhẹ. Với Ollama đã có batching cơ bản qua biến `OLLAMA_NUM_PARALLEL`.
- **Caching semantic** — cache theo **nghĩa** thay vì chuỗi trùng: embedding câu hỏi → nếu câu mới giống ≥0.95 một câu đã hỏi → trả lời cache. Rất hiệu quả với câu hỏi lặp (FAQ), cắt gần như toàn bộ chi phí cho câu cũ.

## README + demo video cho portfolio

Sản phẩm không ai dùng được nếu không ai hiểu: README phải có **tóm tắt 1 câu** sản phẩm làm gì, kiến trúc (sơ đồ), cách chạy local bằng một lệnh, và **link demo live**. Demo video 2–4 phút: bài toán → flow thật (upload → hỏi → trả lời có trích dẫn) → kiến trúc → cái đã học được. Đây là "bộ mặt" của dự án với nhà tuyển dụng — đừng để capstone hay mà không ai biết.

## Checklist "production-ready" trước khi khoe portfolio

Trước khi tuyên bố "xong", soát từng mục — thiếu mục nào thì sản phẩm vẫn còn ở mức demo:

| Hạng mục | Tiêu chí đạt |
|---|---|
| Reproducibility | `docker compose up` trên máy lạ là chạy được |
| Deploy | Chạy trên Oracle Always Free, có domain + HTTPS |
| CI/CD | Push lên `main` = code đang chạy; test + eval gate chặn deploy hỏng |
| Guardrails | Rate limit, auth, schema validation output đều hoạt động (file 04) |
| Observability | Trace từng request trên Langfuse, xem được chi phí + P95 (file 05) |
| Load test | Đã đo 10–30 user đồng thời, biết điểm vỡ RAM/latency |
| Tài liệu | README có hướng dẫn chạy + link demo; video demo 2–4 phút |

## Project GĐ5 = Capstone

Chính là **sản phẩm cuối** mô tả trong [08-FINAL-PROJECT/01-goals-and-scope.md](../08-FINAL-PROJECT/01-goals-and-scope.md) — "GenAI Knowledge Assistant": upload tài liệu → hỏi đáp có trích dẫn + hiểu ảnh/hóa đơn, tổng hợp LLM + RAG + VLM + agent, deploy hạ tầng free hoàn toàn. Mọi thứ học từ tuần 15–18 (tool calling, agent loop, memory, guardrails, observability) đều **hội tụ về đây**. Nguyên tắc giữ nhịp: bám scope chặt (scope phình = không hoàn thành — xem spec gốc), mỗi tuần hoàn thành một mảnh có kiểm chứng (eval + trace), tuần 18 chỉ production hóa — **không thêm tính năng mới trong tuần cuối**.

## Liên kết

- [05 — Observability & Eval](05-observability-eval-production.md) — CI/CD dùng eval làm cổng chặn deploy
- [07 — Tài nguyên & Lỗi thường gặp](07-resources-and-common-pitfalls.md) — lỗi "chạy máy mình là xong" và cách tránh
- [Kiến trúc deploy tổng thể](../07-DEPLOYMENT/01-overall-architecture.md) — bản đồ hạ tầng 0đ cho capstone
