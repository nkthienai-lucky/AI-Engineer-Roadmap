# LoRA, QLoRA & Quantization — Tuần 6

> **Mục tiêu:** Hiểu tại sao fine-tune model lớn với GPU nhỏ nhờ LoRA/QLoRA (ma trận hạng thấp, tham số train thêm), demo khái niệm; quantization INT8/INT4, khác nhau giữa GGUF/GPTQ/AWQ, tại sao model 3B chạy được trên CPU.
> Nguồn mở rộng từ: [02-LLM-INTERNALS.md](../../../02-LLM-INTERNALS.md)

---

## 1. LoRA — fine-tune model lớn với GPU nhỏ

### Bài toán

Full fine-tune Qwen2.5-7B (7 tỉ tham số, fp16) cần: tham số model 14 GB + gradient 14 GB + optimizer (Adam: gấp 2 lần) 28 GB → **> 56 GB VRAM** — bất khả thi với GPU cá nhân/Colab free.

### Ý tưởng: ma trận hạng thấp (low-rank)

Quan sát: khi fine-tune, **hiệu số** của ma trận trọng số `ΔW` có **hạng (rank) rất thấp** — phần lớn tri thức mới nằm trong một không gian con nhỏ. Thay vì cập nhật `W` (shape `d×d`, ví dụ 4096×4096 = 16M tham số), LoRA đông lạnh `W` và train thêm 2 ma trận nhỏ:

$$
W' = W + \Delta W = W + B A, \qquad A \in \mathbb{R}^{r \times d},\ B \in \mathbb{R}^{d \times r},\ r \ll d
$$

Với `r = 8`: `A` là 8×4096, `B` là 4096×8 → **65K tham số** thay vì 16M — nhỏ hơn ~250×.

```python
import torch.nn as nn

class LoRALinear(nn.Module):
    def __init__(self, d_in, d_out, r=8, alpha=16):
        super().__init__()
        self.W = nn.Linear(d_in, d_out, bias=False)   # ĐÔNG LẠNH, requires_grad=False
        self.W.weight.requires_grad = False
        self.A = nn.Linear(d_in, r, bias=False)       # train thêm
        self.B = nn.Linear(r, d_out, bias=False)      # train thêm
        nn.init.kaiming_uniform_(self.A.weight)       # A random
        nn.init.zeros_(self.B.weight)                 # B = 0 → ΔW = 0 lúc đầu

    def forward(self, x):
        return self.W(x) + self.B(self.A(x)) * (self.alpha / self.r)
```

Vì `B` khởi tạo bằng 0, lúc đầu model **giống hệt** model gốc — không phá tri thức pretraining. Train chỉ A, B → optimizer state chỉ cho ~0.1–1% tham số → **chạy được trên 1 GPU 8–16 GB** (yêu cầu bonus project: fine-tune `Qwen2.5-0.5B` bằng LoRA trên Colab free GPU).

Điểm cần nhớ: sau fine-tune, có thể **ghép** `W + BA` thành 1 ma trận (merge) để suy luận không tăng latency, hoặc lưu riêng adapter vài chục MB.

## 2. QLoRA

QLoRA = LoRA trên **model gốc đã quantize 4-bit (NF4)**: trọng số đông lạnh lưu 4-bit (7B chỉ còn ~4 GB), còn LoRA adapter train ở precision cao (bf16/fp32). Kết hợp thêm: double quantization (quantize cả hằng số lượng tử) + paged optimizer. → **fine-tune model 33B trên GPU 24 GB**, 65B trên 1 GPU 48 GB. Demo khái niệm (bằng `peft` + `transformers`):

```python
from transformers import AutoModelForCausalLM, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model

bnb = BitsAndBytesConfig(load_in_4bit=True, bnb_4bit_quant_type="nf4")
model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-0.5B", quantization_config=bnb)
lora = LoraConfig(r=8, lora_alpha=16, target_modules=["q_proj", "v_proj"])
model = get_peft_model(model, lora)   # chỉ train ~0.5% tham số
```

## 3. Quantization: INT8 / INT4

**Quantization** = biểu diễn trọng số bằng ít bit hơn (fp16 = 16 bit → INT8 = 8 bit, INT4 = 4 bit) bằng cách ánh xạ giá trị thực về lưới rời rạc (scale + zero-point). 7B fp16 = 14 GB → INT8 = 7 GB → INT4 = 3.5 GB. Mất một chút độ chính xác (đặc biệt với model nhỏ/nhiệm vụ suy luận), đổi lấy tốc độ + bộ nhớ.

Tại sao model 3B chạy được trên CPU: trọng số 3B INT4 ≈ **1.6 GB** — nằm gọn trong RAM/CPU cache, nhân ma trận số nguyên CPU (AVX2/ARM NEON) làm được; trong khi 3B fp16 = 6 GB + chậm gấp 2–4×. llama.cpp tối ưu kernel CPU → chatbot chạy trên laptop.

## 4. GGUF / GPTQ / AWQ — khác nhau ra sao

| Định dạng | Bit | Thiết kế cho | Đặc điểm |
|-----------|-----|--------------|-----------|
| **GGUF** (llama.cpp) | Q4_K_M, Q8_0... | **CPU + GPU nhẹ** | 1 file self-contained (metadata + trọng số), quantization "K-quant" hậu training (RTN có chọn lọc), chạy llama.cpp/Ollama — chuẩn cho máy cá nhân |
| **GPTQ** | 3/4/8-bit | **GPU** | Quantize từng lớp với calibration set (tối ưu sai số bù trừ giữa các weight), kernel GPU nhanh (ExLlamaV2), cần GPU để suy luận |
| **AWQ** | 4-bit | **GPU** (vLLM, TGI) | Bảo vệ 1% weight "quan trọng" (activation-aware) khỏi quantize → giữ chất lượng tốt hơn GPTQ ở cùng bit, batch throughput cao |

Khi nào dùng gì:
- Chạy trên laptop/CPU, Ollama, app desktop → **GGUF**.
- Phục vụ GPU, throughput cao, vLLM → **AWQ** (hoặc GPTQ nếu tool chỉ hỗ trợ).
- Fine-tune 4-bit (QLoRA) → dùng NF4 của bitsandbytes (khác cả 3 trên — nó dành cho training).

## 5. Tóm tắt

- LoRA: `W' = W + BA` với rank nhỏ — train ~1% tham số, GPU 8 GB fine-tune được model 7B.
- QLoRA: model gốc quantize 4-bit + LoRA — fine-tune 33B trên 24 GB.
- Quantization INT8/INT4 giảm 2–4× bộ nhớ; GGUF (CPU/llama.cpp), GPTQ (GPU), AWQ (GPU, chất lượng cao hơn ở 4-bit).
- Model 3B INT4 ≈ 1.6 GB → chạy CPU được — đó là nền tảng cho deployment (GĐ5).

## Liên kết

- [05-training-finetuning.md](05-training-finetuning.md) — LoRA/QLoRA là kỹ thuật cho bước fine-tuning/SFT.
- [04-transformer-architecture.md](04-transformer-architecture.md) — memory KV cache cũng có thể được quantize.
- [07-project-mini-gpt.md](07-project-mini-gpt.md) — bonus: LoRA Qwen2.5-0.5B trên Colab free GPU.
