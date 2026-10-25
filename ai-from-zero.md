**TL;DR:** AI ที่ "เก่ง" ไม่ได้เริ่มจากอัลกอริทึมวิเศษ แต่เริ่มจาก weight สุ่มกองหนึ่ง แล้วถูกปรับซ้ำๆ ด้วยเกมเดียว — ทำนาย token ถัดไป (cross-entropy + backprop + gradient descent) ความต่างระหว่าง bigram กับ GPT-5 คือปริมาณ ไม่ใช่หลักการ. อยากเข้าใจจริงให้เขียน backprop ด้วยมือ 1 ครั้ง (โค้ด ~6,900 params รันจบใน 5 นาทีบนมือถือ) ถ้าอยากได้ของใช้ให้ fine-tune/RAG แทนการเทรนจากศูนย์ และห้าม `pip install torch` บน Termux.

**Key Points:**
- Loss ตอน step 0 ต้องเท่ากับ `ln(vocab_size)` — ไม่ตรงแปลว่า init หรือ loss ผิด
- Backprop ของ softmax-crossentropy สรุปได้ 3 บรรทัด: `dlogits = probs; dlogits[target] -= 1; dlogits /= N`
- ต้องผ่านการทดสอบ `overfit 1 batch` ก่อนไปต่อเสมอ (ไม่ผ่าน = gradient พัง)
- เทรน 10k params บนมือถือ = นาที; 1M+ params = ต้องย้ายไป laptop/PC ที่มี PyTorch

Tags: {#ai} {#deep-learning} {#training} {#from-scratch} | Updated: 2026-10-25

# AI จากศูนย์ — หลักการ การสอน และบันไดลงมือ

โน๊ตนี้ตอบสามคำถาม: โมเดลเริ่มจาก 0 ได้ยังไง, ทำไมมันเก่งขนาดนี้, และถ้าอยากสร้างเองเล็กๆ ต้องทำอะไรตามลำดับ

---

## 1. เริ่มจาก "0" จริงๆ — สมองที่ยังไม่เกิด

ตอน step 0 โมเดลไม่ได้ "รู้" อะไรเลย มันคือ **กองตัวเลขสุ่ม** หลายพันล้านตัว เรียงตามสูตรที่มนุษย์ออกแบบไว้ (Xavier/He init: สุ่มจากการแจกแจงปกติที่ scale ตามขนาด layer)

ถ้าเปิดดู output ตอน step 0 มันพ่นตัวอักษรเละเทะ เพราะ weight ทุกตัวยังเป็น noise → logits ออกมาเกือบ uniform → เดา token ถัดไปแบบสุ่มเท่ากันหมด

**ทุกความสามารถที่มันมีตอนนี้ ถูกเขียนลงบนกอง noise นั้นทีละนิด** ไม่มีเวทมนตร์

### ขั้น 1 · Tokenizer (ภาษา → integer)
BPE (Byte-Pair Encoding) — หาคู่ตัวอักษรที่เจอบ่อยสุดใน corpus รวมเป็นชิ้นเดียว ทำซ้ำจนได้ vocab ~50k–200k

```
"ก้าวหน้า" → [ก้][าว][หน้][า] → [12345, 8821, 44092, 71]
```

ภาษาไทยโดนหนักเพราะ BPE ถูก optimize บนอังกฤษก่อน → ใช้ token มากกว่า 2–4 เท่า (ช้ากว่า แพงกว่า)

แต่ละ token → vector ~4096 มิติ (embedding) ซึ่งตอนแรกก็สุ่มเหมือนกัน

### ขั้น 2 · Transformer (สถาปัตยกรรม)
หัวใจคือ **Self-Attention** — ทุก token มองทุก token อื่น แล้วถ่วงน้ำหนักว่าตัวไหนเกี่ยวข้อง

```
attention(Q,K,V) = softmax(QKᵀ/√d)·V
```

ตามด้วย MLP ที่ทำหน้าที่เหมือนหน่วยความจำเชิงข้อเท็จจริง (งานวิจัย ROME/MEMIT พบว่า factual knowledge ฝังใน weight ของ MLP กลางๆ) ซ้อน 30–100 ชั้น + residual + layernorm

attention ไม่ได้ "เข้าใจภาษา" — มันแค่ dot product แต่พอซ้อนหลายชั้นและเทรนบนข้อมูลมหาศาล มันเรียนรู้ที่จะคำนวณสิ่งที่หน้าตาเหมือนการเข้าใจขึ้นมาเอง (induction heads, in-context learning)

### ขั้น 3 · การสอน = Next Token Prediction (ทั้งหมดมีแค่นี้)
ไม่มีการติดป้ายเฉลย ไม่มีคนนั่งสอน ไม่มี reward ต่อคำตอบ มีเกมเดียวคือเดา token ถัดไป แล้ววัดว่าพลาดแค่ไหน

```
Input : "น้ำขึ้นให้รีบ"
Target: "ตัก"
Loss  = −log P("ตัก" | "น้ำขึ้นให้รีบ")     ← cross-entropy
```

แล้ว backprop ผ่านทุกชั้น ขยับ weight นิดเดียว (lr ~1e-4 ถึง 6e-4) ทำซ้ำหลายล้าน step

```
w ← w − η·∇L(w)
```

AdamW + warmup + cosine decay, batch 4–8 ล้าน token/step

**นี่คือการสอนทั้งหมด** — ทำนายเก่งขึ้นเรื่อยๆ ก็ต้องเรียนรู้ไวยากรณ์ ตรรกะ ข้อเท็จจริง โค้ด คณิตศาสตร์ไปด้วย เพราะทั้งหมดนั้นคือ "รูปแบบ" ที่ทำให้เดาตัวถัดไปแม่นขึ้น

> การทำนายที่ดี = การบีบอัดข้อมูลที่ดี = การเข้าใจโครงสร้างของโลก

### ขั้น 4 · Scaling Laws (ทำไมต้องใหญ่)
- **Kaplan 2020**: `Loss ∝ N^(-0.076)`, `Loss ∝ D^(-0.095)` — เพิ่มขนาดลด loss แบบคาดเดาได้ล่วงหน้า
- **Chinchilla 2022**: compute-optimal ≈ **20 token ต่อ 1 parameter** (70B ควรเทรน ~1.4T token)
- ข้อมูลดีๆ บน internet หมดแล้ว → ปัจจุบันเน้นคุณภาพ + synthetic + หลาย epoch

### ขั้น 5 · SFT → RLHF (สอนมารยาท)
- **SFT**: คู่ Q&A คุณภาพสูง ~หมื่น–แสนคู่ → สอน "รูปแบบผู้ช่วย"
- **RLHF**: ให้โมเดลตอบหลายคำตอบ → มนุษย์จัดอันดับ → เทรน reward model → PPO
- ผลข้างเคียง: เก่งขึ้นจริง แต่เรียนรู้ที่จะ **ประจบ** (sycophancy)

### ขั้น 6 · RLVR (สอน "คิด")
RLHF สอนรสนิยม แต่สอนความถูกต้องไม่ได้ **RLVR** ใช้ที่ตรวจได้ด้วยเครื่องเป็น reward — คณิตศาสตร์ตรวจคำตอบ, โค้ดรันเทสต์, ให้เขียน chain-of-thought ยาวๆ ก่อนตอบ, optimize ด้วย GRPO/PPO

o1 / R1 / โมเดล reasoning รุ่นใหม่ = ผลพลอยได้จากการ optimize ของที่ตรวจได้ ไม่ใช่จากการสอนให้คิด

---

## 2. ทำไมมันถึงเก่ง (3 คำตอบที่มีงานวิจัยรองรับ)

1. **Compression = Understanding** — ข้อมูลที่บีบอัดได้ดีสุด ต้องสร้าง world model ที่อธิบายข้อมูลนั้นได้
2. **Superposition** — neuron เดียวเก็บส่วนผสมหลายร้อย concept (transformer-circuits.pub) → representation ของ "ประเทศ/เมืองหลวง/เวลา" เป็น linear direction ที่ probing หาเจอ
3. **Emergence** — พอ scale ถึงจุดหนึ่ง ความสามารถใหม่โผล่มา (arithmetic, in-context learning, multi-step reasoning) เหมือนน้ำเป็นน้ำแข็งที่ 0°C

**สิ่งที่มันยังไม่ใช่**: ไม่มีเจตนา ไม่มีความรู้สึก ไม่มีโมเดลของตัวเองแยกจาก weight มันคือฟังก์ชันคณิตศาสตร์ 1e11 parameter ที่ fit กับข้อความ 1e13 token ได้ดีจนน่าตกใจ — ความน่าตกใจคือการทำนายให้แม่นนั้นต้องใช้แทบทุกอย่างที่เราเรียกว่าความฉลาดเป็นเงื่อนไข

---

## 3. บันได 4 ขั้น สำหรับสร้างเอง

| ขั้น | ทำอะไร | params | เครื่อง | เวลา |
|---|---|---|---|---|
| **0** | Bigram — นับคู่ตัวอักษรเป็น probability table (ไม่ใช่ neural net) | ~700 | มือถือ numpy | 5 นาที |
| **1** | **MLP makemore — เขียน backprop ด้วยมือ** ← ขั้นที่ต้องทำ | ~7k | มือถือ numpy | 30–60 นาที |
| **2** | Tiny char-GPT (ต้องมี autograd: tinygrad หรือ PyTorch) | 1–5M | laptop | 30–120 นาที |
| **3** | Fine-tune โมเดลจริงด้วย LoRA (ไม่ใช่ from scratch) | 1–3B | GPU | 1–3 ชม. |

**หลักการเดียวที่สำคัญที่สุด**: อย่าเริ่มจาก "สร้าง AI" ให้เริ่มจาก "สร้างตัวทำนายตัวอักษรถัดไป" เพราะขั้น 1 กับ GPT-5 ใช้สมการเดียวกัน

---

## 4. Loop 6 ท่อน — โครงที่ทุกโปรเจกต์ AI ใช้ร่วมกัน

```
[1] DATA      ข้อความดิบ → list ของ integer
[2] FORWARD   x → logits (ความน่าจะเป็นทุก token)
[3] LOSS      เทียบกับเฉลย → ตัวเลขเดียว (ยิ่งต่ำยิ่งดี)
[4] BACKWARD  ∇loss ทุก weight
[5] UPDATE    w ← w − lr·∇w
[6] EVAL      วัดบนข้อมูลที่ไม่เคยเห็น + sample ดูของจริง
                 ↑__________________________|  วนกลับ [2]
```

ทุกอย่างที่เหลือ — transformer, RLHF, reasoning — คือการตกแต่ง [2] กับ [3] เท่านั้น [1][4][5][6] แทบไม่เปลี่ยนตั้งแต่ยุค 1980

---

## 5. ห้าอย่างที่ต้องเข้าใจก่อน (ไม่เข้าใจ = ตันที่ step แรก)

1. **Loss = ความเซอร์ไพรส์** ไม่ใช่ความผิด — `−log P(ถูก)`; ค่าเริ่มต้นของโมเดลสุ่ม = `ln(vocab_size)` เสมอ (27 ตัวอักษร = 3.29, 50k token = 10.8) ← เลขสำหรับตรวจบั๊ก
2. **Gradient = ทิศที่ loss เพิ่ม** — เราเอาค่าลบของมัน
3. **Learning rate สำคัญกว่า architecture** — ใหญ่ไป NaN ใน 10 step, เล็กไปไม่ขยับ; เริ่ม `3e-4` (โมเดล <10M) หรือ `1e-4` (ใหญ่กว่า) + warmup 100 step + cosine decay
4. **Train vs Val คือทุกอย่าง** — train ลง val ขึ้น = ท่องจำ ไม่ใช่เรียนรู้
5. **Overfit ตัวเล็กคือเรื่องดี** — 10k params บนข้อมูล 1MB ควร overfit ได้ ถ้าไม่ได้เลย = gradient หาย/ผิด

---

## 6. สูตรขั้น 1 (ตัวเลขจริง)

```
DATA      corpus ~1MB → char-level vocab ~60
INPUT     ตัวอักษร 3 ตัวก่อนหน้า (block size 3)
EMBED     ตัวละ 2 มิติ → concat = 6 มิติ
LAYER 1   6 → 100 (tanh)
LAYER 2   100 → vocab
PARAMS    ~60·2 + 6·100 + 100 + 100·60 + 60  ≈ 6,900
OPT       AdamW, lr 1e-3, batch 64, 10,000 step
เป้าหมาย   loss 3.3 → ~2.1
```

**Backprop ด้วยมือ** — หัวใจอยู่ 3 บรรทัดนี้:

```
probs = softmax(logits)
dlogits = probs                   # ← จุดที่คนเข้าใจผิดบ่อยสุด
dlogits[i, target[i]] -= 1        # d(-log p_y)/dlogit = p - onehot
dlogits /= N
```

แล้วไล่ย้อน `dW2, db2, dhidden, dW1, db1` — เข้าใจ 3 บรรทัดนี้ = เข้าใจ deep learning 60%

---

## 7. ลำดับเรียน (อย่าข้าม)

| ลำดับ | เรื่อง | เวลา |
|---|---|---|
| 1 | Micrograd — สร้าง autograd engine 100 บรรทัด | 2 ชม. |
| 2 | Makemore part 1 — bigram | 1 ชม. |
| 3 | Makemore part 2 — MLP + manual backprop | 3 ชม. |
| 4 | Makemore part 3–5 — batchnorm, init, gradient flow | 4 ชม. |
| 5 | "Let's build GPT" — transformer จากศูนย์ | 5 ชม. |
| 6 | nanoGPT — เทรน shakespeare จริง | 2 ชม. |

ทั้งหมดคือ **Karpathy — Neural Networks: Zero to Hero** (YouTube, ฟรี) ไม่ต้องอ่านหนังสือก่อน ดูไปเขียนตามไป

---

## 8. Debug checklist (ของจริง — ไม่มีในบทความ)

1. **loss step 0 ตรงกับ `ln(vocab)` ไหม** → ไม่ตรง = init หรือ loss ผิด
2. **overfit 1 batch ได้ไหม** → ไม่ได้ = backprop พัง
3. **numerical gradient check** (ε=1e-4) เทียบ analytic
4. **NaN** → lr สูงไป / ไม่ได้ warmup / log(0)
5. **loss แบนค้าง** → lr ต่ำไป หรือ capacity ไม่พอ
6. **sample ซ้ำตัวเดิม** → temperature ต่ำเกิน หรือยังไม่เรียนอะไร
7. **val ต่ำกว่า train** → ปกติ ถ้าต่างมาก = regularize แรงเกิน

---

## 9. ทางลัด (ถ้าเป้าหมายคือได้ของใช้ ไม่ใช่เข้าใจ)

- **Fine-tune แทนเทรนจากศูนย์**: โมเดล 1–3B + LoRA (Unsloth / llama.cpp) + ข้อมูลตัวเอง 1,000–5,000 ตัวอย่าง → GPU 1 ตัว 1–3 ชม. LoRA แช่แข็ง weight เดิม เทรนแค่ adapter ~10M params
- **แต่**: fine-tune ไม่ได้ให้ความรู้ใหม่ แค่ปรับสไตล์/รูปแบบ ถ้าอยากให้รู้เรื่องที่มันไม่เคยรู้ → **RAG** (ยัดข้อความเข้า context) ถูกกว่าและได้ผลกว่าทุกกรณี

**ข้อห้าม**: (1) `pip install torch` บน Termux = dependency hell glibc (2) เริ่มจาก 1B params (3) ก๊อปโค้ดแล้วไม่รู้ว่าอะไรเกิดขึ้น (4) โหลด dataset 100GB ก่อนเคยเทรนจบสักตัว

---

## 10. ข้อจำกัดเครื่องนี้ (วัดจริง 2026-10-25 · Black Shark DLT-H0 / Termux)

| ของ | สถานะ |
|---|---|
| Python | 3.14.8 |
| numpy | ยังไม่ติดตั้ง — `pkg install python-numpy` = 2.5.4, 21.8 MB (ติดตั้งได้ ไม่เจ็บ) |
| torch / jax / tinygrad | ไม่มี (และไม่ควรพยายามลง torch ที่นี่) |
| CPU / RAM / disk | 6 cores · เหลือ ~6.4 GB · ว่าง 206 GB |
| Laptop `192.168.1.112` | ping ตอบ แต่ sshd ตาย (port 22 refused) → ต้องไปเปิด `sudo systemctl start ssh` ก่อน offload งาน |
| PC `192.168.1.111` | ดับ |

⇒ ขั้น 0–1 วิ่งบนมือถือได้จริง · ขั้น 2 ขึ้นไปต้องปลุก laptop ก่อน
