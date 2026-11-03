# Training & Fine-tuning LLM — Tuần 6

> **Mục tiêu:** Phân biệt pretraining / fine-tuning / RLHF-DPO, hiểu pipeline 3 bước huấn luyện LLM hiện đại, loss cross-entropy với token tiếp theo, và ảnh hưởng của learning rate, batch size.
> Nguồn mở rộng từ: [02-LLM-INTERNALS.md](../../../02-LLM-INTERNALS.md)

---

## 1. Pretraining vs Fine-tuning vs RLHF/DPO

### Pretraining (tiền huấn luyện)

- **Dữ liệu:** hàng nghìn tỉ token text thô (web, sách, code) — không nhãn.
- **Mục tiêu:** học *ngôn ngữ + tri thức thế giới* bằng nhiệm vụ tự giám sát: **đoán token tiếp theo**.
- **Kết quả:** base model — biết tiếp tục văn bản nhưng chưa biết "trả lời cho tử tế" (prompt "Bạn là ai?" có thể bị hoàn thành thành một đoạn blog).
- **Chi phí:** hàng nghìn GPU-week; chỉ các lab lớn làm. Kỹ sư AI thường **dùng base model có sẵn**.

### Fine-tuning (tinh chỉnh)

- **Dữ liệu:** hàng nghìn–vài trăm nghìn mẫu **có nhãn** (instruction → response).
- **Mục tiêu:** dạy *định dạng và hành vi* cho nhiệm vụ cụ thể: trả lời câu hỏi, chat, trích xuất thông tin, giọng văn riêng.
- **Kỹ thuật:** full fine-tune (cập nhật mọi tham số) hoặc parameter-efficient (LoRA/QLoRA — file 06). Vẫn dùng loss cross-entropy như pretraining, chỉ khác dữ liệu.

### RLHF / DPO (căn chỉnh theo phản hồi con người)

- **RLHF** (Reinforcement Learning from Human Feedback): con người **xếp hạng** nhiều câu trả lời → train reward model → tối ưu policy (model) bằng PPO để sinh câu trả lời được điểm cao. Mục tiêu: hữu ích, trung thực, vô hại (HHH).
- **DPO** (Direct Preference Optimization): bỏ qua reward model + PPO, tối ưu **trực tiếp** trên cặp (response được ưa thích, response bị chê) — đơn giản hơn, ổn định hơn, đang rất phổ biến cho model mở.

### Pipeline 3 bước huấn luyện LLM hiện đại

```
[1] Pretraining (self-supervised, nghìn tỉ token)
        ↓  base model
[2] Fine-tuning (SFT: instruction-response có nhãn)
        ↓  instruction model
[3] RLHF / DPO (preference: cặp tốt/xấu của con người)
        ↓
   Chat model hoàn chỉnh (ChatGPT, Qwen-Instruct...)
```

Mỗi bước sửa "tật" của bước trước: pretraining biết nói nhưng không biết trả lời; SFT biết trả lời nhưng không biết cách nào con người thích; RLHF/DPO chọn cách con người thích.

## 2. Loss function: cross-entropy với token tiếp theo

LLM học bằng cách: cho chuỗi `t1...tn`, dự đoán phân phối xác suất `p(t_{i+1} | t1...ti)` cho **mỗi vị trí**. Cross-entropy đo xác suất model gán cho token **đúng**:

$$
\mathcal{L} = -\frac{1}{N}\sum_{i=1}^{N} \log p_\theta(t_i \mid t_{<i})
$$

**Intuition:** loss = "mức độ ngạc nhiên" của model. Model chắc 90% token đúng → loss ≈ 0.1 (nhỏ); chắc 1% → loss ≈ 4.6 (lớn). Train = kéo xác suất của token đúng tiến dần về 1.

```python
import torch.nn.functional as F

# logits: (batch, seq_len, vocab) — output thô của model
# targets: (batch, seq_len) — token thật (dịch phải 1 vị trí so với input)
loss = F.cross_entropy(
    logits.view(-1, vocab_size),
    targets.view(-1),
)   # trung bình -log softmax tại token đúng
```

Ghi chú kỹ thuật quan trọng: target là **input dịch trái 1 đơn vị** — token tại vị trí `i` phải dự đoán token `i+1`. Loss khởi đầu ≈ `ln(vocab_size)` (~10.8 với vocab 50k) = "đoán ngẫu nhiên"; loss giảm dần là tín hiệu train đúng hướng (yêu cầu DoD: "Loss giảm dần khi train").

**Cross-entropy với classification:** cùng công thức, nhưng ở LLM lớp "class" = vocab, và áp ở *mỗi* vị trí token trong chuỗi — một câu 100 token = 100 bài toán classification nhỏ.

## 3. Learning rate & batch size

### Learning rate (tốc độ học)

- **Quá lớn:** loss nổ (NaN) hoặc dao động không hội tụ — bước nhảy vượt qua vùng tối ưu.
- **Quá nhỏ:** hội tụ cực chậm, dễ kẹt ở điểm yên ngựa.
- Thực hành chuẩn: **warmup + cosine decay** — tăng dần LR ở vài trăm step đầu (tránh sốc gradient sớm) rồi giảm dần; LR peak phổ biến: `3e-4` (pretrain GPT nhỏ), `1e-5`–`2e-5` (fine-tune full), `2e-4` (LoRA).

```python
# nanoGPT-style: warmup rồi cosine về 10% peak
lr = lr_peak * (step / warmup_steps) if step < warmup_steps \
     else lr_min + 0.5*(lr_peak - lr_min)*(1 + cos(pi * progress))
```

### Batch size (kích thước lô)

- **Quá nhỏ:** gradient ồn (ước lượng xấu của gradient toàn tập), loss rung, GPU kém hiệu quả.
- **Quá lớn:** gradient mượt nhưng mỗi bước chậm, tốn VRAM; trên lý thuyết có thể dùng LR lớn hơn (linear scaling rule).
- Thực hành: chọn batch lớn nhất GPU chứa được, dùng **gradient accumulation** (gom nhiều micro-batch rồi mới update) để mô phỏng batch lớn. Ví dụ: micro-batch 8 × gradient-acc 4 = batch 32.

### Hai siêu tham số này liên hệ thế nào

Batch size và LR cùng quyết định "kích thước bước" trong không gian tham số. Fine-tune: LR nhỏ + vài epoch (tránh "quên" tri thức pretraining — catastrophic forgetting); pretrain: LR lớn hơn + cosine decay theo hàng nghìn tỉ token. Với Mini-GPT (file 07), bắt đầu với `lr=3e-4`, batch nhỏ, quan sát loss curve rồi chỉnh — **tuning loss curve là kỹ năng thực hành chính của tuần 6**.

## 4. Tóm tắt

- Pretraining (tri thức, không nhãn) → SFT (hành vi, có nhãn) → RLHF/DPO (sở thích con người).
- Cross-entropy = "ngạc nhiên" của model với token thật; mục tiêu kéo xác suất token đúng về 1.
- LR & batch size: quá lớn/lớn đều hại; dùng warmup + cosine decay, gradient accumulation.

## Liên kết

- [04-transformer-architecture.md](04-transformer-architecture.md) — kiến trúc transformer được huấn luyện ở bước pretraining.
- [06-lora-qlora-quantization.md](06-lora-qlora-quantization.md) — fine-tune hiệu quả tham số (LoRA/QLoRA) cho bước SFT.
- [07-project-mini-gpt.md](07-project-mini-gpt.md) — áp dụng loss/LR/batch vào Mini-GPT.
