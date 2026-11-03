# 06 — Hạ tầng tính tiền (khi nào free không đủ)

> **Mục tiêu**: biết giá tham khảo và đặc điểm từng lựa chọn GPU/inference trả phí (Vast.ai, RunPod, Modal, SaladCloud) và nhận diện thời điểm nên nâng cấp từ free — bảng tra, không học thuộc.
>
> Nguồn mở rộng từ: [06-ECOSYSTEM](README.md)

---

## Bảng tra nhanh

| Nhu cầu | Nơi rẻ | Giá tham khảo |
|---|---|---|
| GPU theo giờ (train/batch) | **Vast.ai** marketplace | RTX 4090 từ ~$0.13–0.51/h |
| GPU ổn định hơn | **RunPod Community** | RTX 4090 $0.34/h ≈ $250/tháng |
| Inference API theo request | **RunPod Serverless / Modal** | Pay-per-request, ~$5–20/tháng app nhỏ |
| Rẻ nhất, chấp nhận rủi ro | SaladCloud | Từ $0.02/h (hay disconnect) |

---

## Từng lựa chọn là gì, hợp với ai?

- **Vast.ai (marketplace)**: chợ GPU peer-to-peer — người dùng cho thuê GPU rảnh, bạn bid/thuê theo giờ. Giá dao động **~$0.13–0.51/h với RTX 4090** (tùy host, tùy thời điểm). Rẻ nhất cho nhu cầu **train/batch theo giờ** (chạy xong tắt). Trade-off: chất lượng host không đồng đều, kết nối/hạ tầng phụ thuộc từng máy — hợp việc có checkpoint được, không hợp service cần uptime.
- **RunPod Community**: cũng là marketplace nhưng **ổn định hơn** (tiêu chuẩn host chặt hơn, UX quản lý tốt hơn) → giá cao hơn mặt bằng: **RTX 4090 $0.34/h ≈ $250/tháng** nếu chạy liên tục 24/7 (con số ≈ $250/tháng giúp hình dung: thuê theo giờ chỉ đáng khi chạy có khoảng nghỉ; chạy full tháng thì so với dedicated server).
- **RunPod Serverless / Modal**: **pay-per-request** — không thuê GPU 24/7 mà trả theo từng lượt inference, scale về 0 khi không dùng. **~$5–20/tháng cho app nhỏ** (traffic thấp). Hợp inference API cho sản phẩm, không hợp train (không gian session dài).
- **SaladCloud**: rẻ nhất, **từ $0.02/h** — tận dụng GPU gaming của cộng đồng, nhưng **hay disconnect** (bị host tắt máy bất chợt). Chỉ hợp workload chịu được gián đoạn: batch processing có retry, render, crawl — tuyệt đối không chạy service cần uptime hay train dài không checkpoint.

## Khi nào nên nâng cấp từ free?

- **Free tier chạm giới hạn**: rate limit/timeout của Ollama free host, Hugging Face Spaces free (CPU yếu), Colab/Kaggle hết giờ — khi thao tác thật bị chặn, không chỉ "muốn nhanh hơn".
- **Cần uptime/SLA** cho user thật: free tier ngủ khi idle, hay bị kill → production cần compute trả phí (thường RunPod Serverless/Modal trước, vì chỉ ~$5–20/tháng app nhỏ).
- **Train/fine-tune hoặc batch lớn**: CPU/free không khả thi về thời gian → thuê GPU theo giờ (Vast.ai rẻ nhất cho việc có điểm dừng).
- **Nguyên tắc chi phí**: tính trước "số giờ × giá giờ" so với giá gói tháng (VD: $0.34/h × 720h ≈ $250/tháng — nếu nhu cầu gần full-time, cân nhắc máy riêng/dedicated thay vì thuê giờ).

## Chọn nhanh theo kịch bản

- **Fine-tune/ train vài giờ cuối tuần** → Vast.ai (giá ~$0.13–0.51/h, chợ giá rẻ, việc có checkpoint).
- **Demo cho khách/ nhà tuyển dụng, cần máy ổn định vài ngày** → RunPod Community (ổn định hơn marketplace rẻ tiền).
- **App có user thật nhưng traffic thấp** → RunPod Serverless / Modal (pay-per-request ~$5–20/tháng, scale về 0 khi vắng).
- **Batch xử lý hàng loạt chịu được gián đoạn** (render, crawl, số hóa tài liệu có retry) → SaladCloud (từ $0.02/h, chấp nhận hay disconnect).
- **Red flag**: đừng đưa workload cần uptime lên SaladCloud; đừng thuê GPU 24/7 cho workload chỉ chạy 2 giờ/ngày (dùng Serverless thay vì Community).

---

## Liên kết giá live (trong nguồn)

- [gpudiff.com](https://gpudiff.com/) · [gputable.dev](https://gputable.dev/) · [gpucloudprices.com](https://gpucloudprices.com/) — bảng giá GPU cập nhật, dùng để đối chiếu giá tham khảo ở trên lúc quyết định (giá đổi liên tục).

## Liên kết

- [05-model-quick-picks.md](05-model-quick-picks.md) — nếu model nhỏ chạy CPU là đủ thì chưa cần tới các lựa chọn ở đây.
- [04-rag-pipeline-tools.md](04-rag-pipeline-tools.md) — Serving (Ollama/llama.cpp/vLLM) trên hạ tầng bạn chọn; Langfuse Cloud free cũng là cách tránh tốn VPS cho observability.
