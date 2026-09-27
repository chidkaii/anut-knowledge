**TL;DR:** ComfyUI + Ollama 2-pass startup guide for Ubuntu — pre-flight checklist, service startup, node wiring for high-res upscaling, and troubleshooting.

**Key Points:**
- Ollama ตรวจสถานะที่ `http://127.0.0.1:11434/` และ restart ด้วย systemctl
- ComfyUI เริ่มที่ `python main.py` แล้วเปิด `http://127.0.0.1:8188`
- 2-pass: KSampler1 (denoise 1.00) → Upscale Latent → KSampler2 (denoise 0.35-0.40)
- คิวค้าง 0% → `curl -X POST http://127.0.0.1:8188/interrupt` แล้ว restart Ollama

Tags: {#comfyui} {#ollama} {#linux} | Updated: 2026-09-28
# ComfyUI & Ollama 2-Pass Workflow Startup Guide

คำแนะนำขั้นตอนการเปิดใช้งานระบบ **ComfyUI** ร่วมกับ **Ollama** สำหรับการเจนภาพ 2-Pass High-Resolution (1024x1024) บน Ubuntu

---

## 📋 Pre-flight Checklist (ข้อควรตรวจสอบก่อนเริ่ม)

1. **Model Checkpoint:** `realisticVisionV20_v20.safetensors` วางใน `ComfyUI/models/checkpoints/`
2. **Built-in VAE:** ไม่จำเป็นต้องต่อโหนด VAE แยกภายนอก ให้ต่อช่อง `VAE` จากโหนด `Load Checkpoint` โดยตรง
3. **Ollama Models:** ตรวจสอบโมเดลที่แนะนำ เช่น `phi4-mini:3.8b` หรือ `qwen2.5-coder:7b` เพื่อการตอบสนองที่รวดเร็ว

---

## 🚀 Step-by-Step Startup Guide

### 1. ตรวจสอบและเริ่มบริการ Ollama Service

เปิด Terminal และเช็คสถานะของ Ollama:

```bash
curl http://127.0.0.1:11434/
```

* **ถ้าขึ้น `Ollama is running`:** บริการพร้อมใช้งาน
* **ถ้าเชื่อมต่อไม่ได้ หรือบริการค้าง:** ให้ทำการ Restart บริการด้วยคำสั่ง:

```bash
sudo systemctl restart ollama
```

ตรวจสอบรายชื่อโมเดลที่มีในเครื่อง:

```bash
ollama list
```

---

### 2. เริ่มต้นใช้งาน ComfyUI

เปิด Terminal ใหม่ แล้วเข้าสู่โฟลเดอร์ ComfyUI และสั่งรัน:

```bash
cd ~/ComfyUI
python main.py
```

*(หรือถ้ารันผ่าน PM2/Systemd Service)*:

```bash
pm2 restart comfyui
# หรือ
sudo systemctl restart comfyui
```

เปิด Browser แล้วเข้าไปที่ `http://127.0.0.1:8188`

---

## 🛠️ Workflow Architecture Setup (2-Pass Upscale)

จัดวางลำดับโหนด (Node Wiring) ตามโครงสร้างดังนี้:

```
[Load Checkpoint] ──(MODEL)──> [KSampler 1] ───────────────(LATENT)──> [Upscale Latent] ──(LATENT)──> [KSampler 2] ──(LATENT)──> [VAE Decode] ──> [Save Image]
       │                          ▲                                           ▲                                    ▲                 ▲
       ├──(CLIP)──> [Ollama Gen]──┤ (Positive)                                 │                                    │                 │
       │                          │ (Negative)                                │                                    │                 │
       └──(VAE)───────────────────┴───────────────────────────────────────────┴────────────────────────────────────┴─────────────────┘
```

### Node Parameters Reference:

| Node | Parameter | Recommended Value | Note |
| :--- | :--- | :--- | :--- |
| **Empty Latent Image** | Width / Height | `760` x `512` | ใช้เฉพาะใน Pass 1 |
| **KSampler 1 (Pass 1)** | Steps / CFG / Denoise | `20` / `7.5` / `1.00` | สแกนสร้างโครงสร้างภาพหลัก |
| **Upscale Latent** | Target Resolution | `1140` x `768` (or Scale `1.5x`-`2.0x`) | รักษา Aspect Ratio |
| **KSampler 2 (Pass 2)** | Steps / CFG / Denoise | `20` / `8.0` / `0.35` - `0.40` | เติมรายละเอียดภาพขยาย |
| **Ollama Connectivity** | URL | `http://127.0.0.1:11434` | กด Reconnect ทุกครั้งหลังเปิด service |
| **Ollama Generate** | Model / Think | `phi4-mini:3.8b` / `OFF` | ปิด think เพื่อป้องกัน Timeout |

---

## 🔧 Troubleshooting & Quick Commands

### เมื่อคิวค้างที่ 0% (Ollama Timeout / Interrupted Task)

1. ยกเลิก Queue ผ่าน API:
   ```bash
   curl -X POST http://127.0.0.1:8188/interrupt
   ```
2. รีเซ็ต Process ของ ComfyUI หากปุ่ม Cancel บน UI ไม่ทำงาน:
   ```bash
   pkill -f "main.py"
   ```
3. รีสตาร์ต Ollama เพื่อเคลียร์ Prompt Cache & VRAM State:
   ```bash
   sudo systemctl restart ollama
   ```

### ย้ายสายให้เป็นระเบียบ (Noodle Cleanup)
* **Reroute Node:** กด `Alt + คลิกซ้าย` บนเส้นเชื่อมต่อเพื่อสร้างจุดดัดสาย
* **Straight Lines:** ไปที่ **Settings (⚙️)** ➔ **Link Render Mode** ➔ เปลี่ยนเป็น `Straight`
