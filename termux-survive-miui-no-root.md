# Termux อยู่รอดบน MIUI โดยไม่ต้อง root — และสถานะการปลดล็อก bootloader ของ Black Shark 2 Pro

> บันทึกจาก session 2026-10-24 บน **Black Shark 2 Pro (DLT-H0 / `darklighter` / Android 10 / JOYUI `V11.0.4.0.JOYUI`)**
> เป้าหมายเดิม: ตั้งมือถือเป็น always-on server (sshd + OpenCode :4096 + pinggy + Telegram bot) แต่ **MIUI ฆ่า Termux ทิ้งกลางวัน**
> ยืนยันแล้วว่าตั้งค่า battery = No restrictions + Autostart + lock in recents ครบแล้วก็ยังโดนฆ่า

---

## 1) ทางออกที่ไม่ต้อง root (ทำก่อนเสมอ) — **ปิด `com.miui.powerkeeper` ด้วย adb**

ตัวฆ่าแอปเบื้องหลังของ MIUI คือ **PowerKeeper** ไม่ใช่ AOSP doze — หลักฐาน: `com.termux` ถูก whitelist ใน
AOSP อยู่แล้ว (`cmd deviceidle whitelist` มีทั้ง `user,com.termux` และ `user,com.termux.boot`) และ
`RUN_IN_BACKGROUND` / `RUN_ANY_IN_BACKGROUND` = `allow` ครบ ⇒ ระบบที่ฆ่ามันคือ MIUI เอง

ปิดได้ด้วย adb ธรรมดา **ไม่ต้อง root** และไม่ล้างเครื่อง:

```bash
ADB=~/mibox/sdk/platform-tools/adb

# 1. ดูสถานะก่อน (enabled=0 คือปกติ, enabled=3 คือ disabled-user)
$ADB shell 'dumpsys package com.miui.powerkeeper | grep "User 0:"'

# 2. ปิด (persist ข้ามรีบูต)
$ADB shell 'pm disable-user --user 0 com.miui.powerkeeper'
#    → Package com.miui.powerkeeper new state: disabled-user

# 3. ยืนยัน
$ADB shell 'dumpsys package com.miui.powerkeeper | grep "User 0:"'   # ต้องเห็น enabled=3

# 4. ถ้าอยากย้อนกลับ
$ADB shell 'pm enable com.miui.powerkeeper'
```

**ข้อควรรู้**
- โปรเซสที่รันอยู่ *ตอนนั้น* ยังไม่ตาย (`ps -A | grep powerkeeper` ยังเห็น) และ `am force-stop` ไม่สามารถฆ่าได้
  เพราะเป็น system uid 1000 → **ต้องรีบูตหนึ่งครั้ง** ให้มันไม่ถูกสตาร์ทใหม่ ผลจริงจะเห็นหลังรีบูต
- ผลข้างเคียงที่อาจเจอ: หน้า battery stats / กราฟการใช้ไฟใน Security app อาจเพี้ยน, animation ตอนชาร์จอาจหาย
  (ไม่กระทบการชาร์จจริง — `dumpsys battery` ยังรายงานปกติ) ย้อนกลับได้ทันทีด้วย `pm enable`
- **ยังไม่ควรปิด `com.miui.securitycenter`** — เป็นแกนของ Security/Autostart/Permission ทั้งระบบ ปิดแล้วพังเยอะ

### ทางเสริมหากยังโดน kill อยู่

| ทาง | วิธี | ข้อดี / ข้อเสีย |
|---|---|---|
| **Termux:API + job scheduler** | ติดตั้ง `com.termux.api` (APK ต้องเซ็นกุญแจเดียวกับ `com.termux` → ใช้ F-Droid: `https://f-droid.org/repo/com.termux.api_1002.apk`) แล้ว `termux-job-scheduler --period-ms 900000 --persisted true --script ~/server-up` | Job อยู่ใน system_server → ปลุกแอปคืนได้แม้โปรเซสตาย และ `--persisted` อยู่ข้ามรีบูต. ข้อเสีย: คาบสั้นสุด ~15 นาที, MIUI อาจเลื่อน job |
| **ปิด MIUI optimization** | Developer options → เลื่อนหา "MIUI optimization" แล้วปิด (เครื่องจะรีสตาร์ท Settings) | ลดการจัดการ memory แบบ MIUI ลงชัดเจน; แลกกับ UI บางอย่างเปลี่ยน |
| **Watchdog จากเครื่องอื่นใน LAN** | Mi Box / PC / แล็ปท็อป รัน loop: `adb connect 192.168.1.114:5555` → ถ้า `pidof com.termux` ไม่มี → `adb shell am start -n com.termux/.HomeActivity` | ฟรี แต่ต้องมีเครื่องที่เปิดตลอด — ถ้ามีเครื่องนั้นอยู่แล้ว ก็ย้าย server ไปไว้ที่นั้นเลยจบกว่า |
| **เสียบชาร์จ + เปิดหน้าจอค้าง** | Developer options → "Stay awake while charging" | ได้ 100% (แอป foreground ไม่โดน kill) แต่กินไฟ + จอ OLED เบิร์น ไม่แนะนำ |

### ถ้าจะย้าย server ไปที่อื่นแทน (ไม่ต้องพึ่งมือถือเลย)
- **Oracle Cloud Always Free (ARM Ampere)** — ฟรีจริง 4 OCPU / 24 GB RAM, public IP คงที่, ไม่มี MIUI
  ⇒ sshd + OpenCode + Telegram bot ทำงาน *ดีกว่า* บน Android และไม่ต้องใช้ pinggy
- **Mi Box S (Android TV)** — เปิดตลอดและไม่มี MIUI แต่ userspace เป็น 32-bit ⇒ **binary `opencode` (arm64) รันไม่ได้**
- **PC (192.168.1.111)** — แรงสุด แต่ปิดเครื่องบ่อย

---

## 2) สถานะการปลดล็อก bootloader — ทำไมทาง Windows/Mi Unlock Tool ใช้ไม่ได้กับเครื่องนี้

ถ้าจะ root จริง ๆ ต้องปลดล็อกก่อน และการปลดล็อกบนเครื่องนี้ **ตายทุกทางที่免費** หลักฐานจากการทดสอบจริง:

| ทาง | คำสั่ง / หลักฐาน | ผล |
|---|---|---|
| XDA backdoor (Android 9) | `fastboot oem bs_unlock` | `FAILED (remote: 'unknown command')` — ถูกถอดออกจาก JOYUI 11 |
| AOSP standard | `fastboot flashing unlock` | `FAILED (reap_urb(0) failed: urb returned error: Protocol error)` (bootloader ตัด USB) |
| OEM token flow | `fastboot oem unlock` | `FAILED (remote: 'Token Verify Failed, Reboot the device')` — **รับคำสั่ง แต่ต้องใช้ token ที่เซิร์ฟเวอร์เซ็น** |
| ดึง token | `fastboot oem get_token` / `fastboot getvar token` | `unknown command` / `GetVar Variable Not found` ⇒ **ขอ token ไม่ได้** |
| สลับ slot ไปหาชุด Android 9 | `fastboot set_active b` | `FAILED (remote: 'Slot Change is not allowed in Lock State')` |
| เมนูผูก Mi account | Developer options | **ไม่มี "Mi Unlock status" / "Bootloader status" เลย** (สแกนจนสุด, ค้นหาใน Settings = "No results") |
| Activity ที่ซ่อนอยู่ | `am start com.android.settings/.bootloader.BsBootloaderStatusActivity` | `SecurityException: not exported from uid 1000` |
| เซิร์ฟเวอร์ผูกเครื่องของ Black Shark | `unlock.blackshark.com`, `unlock-api.blackshark.com` | **ไม่มี DNS แล้ว** |
| ARB/anti-rollback | `getvar rollback-index`, `arb`, `anti` | bootloader ไม่มีตัวแปรนี้ให้อ่าน → **ประเมิน ARB ไม่ได้เลย** |

### ทำไม "ลง global ROM เองด้วย fastboot" ถึงทำได้ทั้งที่เครื่องล็อก
การล็อก bootloader **ห้ามเฉพาะของที่ไม่มีลายเซ็น Xiaomi** — แพ็กเกจ **signed OTA / signed fastboot package**
ผ่านการตรวจ AVB signature ได้แม้เครื่องล็อก (นี่คือวิธีแปลง CN → Global ที่คนทำเองที่บ้าน)
ถ้าตอนนั้นแฟลชด้วย `flash_all.bat` จริง เครื่องต้อง **unlocked อยู่ก่อน** และมีร่องรอยค้าง: `sys.unlock.finished=1`
และ `persist.sys.unlock.finished=1` (ค่าหลังยังอยู่ข้ามรีบูต) — น่าจะถูกล็อกกลับภายหลังด้วยสคริปต์ `flash_all_lock`

### ทางที่เหลือจริง ๆ มี 2 ทาง
1. **ถอย ROM เป็น Android 9 (Route B)** — เอา *signed full OTA* Android 9 **Global** (`darklighter_global`,
   variant `th`) วางที่ `/sdcard/ota/update.zip` → กด `*#*#1027#*#*` ในแอปโทรศัพท์ → Local Update
   → ได้ bootloader (`abl`) รุ่นเก่าที่มี `bs_unlock` → เอา root ต่อ
   **ความเสี่ยง:** ARB ไม่รู้ค่า ถ้าตันจะบูตไม่ขึ้น = ต้อง EDL authorized (เสียเงิน) และต้องล้าง data
2. **EDL authorized (Route D)** — ร้าน/ศูนย์/บริการออนไลน์ที่ถือ Mi account authorized แฟลชได้ทุก partition
   ล็อกไม่มีผล, ไม่ต้องลุ้น ARB, กู้เครื่องได้ถ้าพัง — จ่ายเงินแลกความชัวร์

### ขั้นตอนบน Windows (ถ้าจะลองอยู่ดี — เพื่อเป็น knowledge)
> ⚠️ **กับเครื่องนี้จะล้มเหลวที่ขั้น 1** เพราะไม่มีเมนูให้ผูก account ⇒ ใส่ไว้เพื่อเป็น runbook ของเครื่องรุ่นอื่น
1. มือถือ: Settings → About phone → แตะ "MIUI version" 7 ครั้ง → Developer options
2. Settings → Additional settings → Developer options → **Mi Unlock status** → "Add account and device"
   *(เครื่องนี้: เมนูนี้ไม่มี → หยุดตรงนี้)*
3. Windows: โหลด **Mi Unlock Tool** (`miflash_unlock-*.zip`) จากทางการ → แตกไฟล์ → รัน `miflash_unlock.exe`
4. ล็อกอิน Mi account **ตัวเดียวกับที่ผูกในข้อ 2** → ต่อสาย USB → ปิดเครื่อง → กด **Power + Volume Down**
   ค้างไว้จนเข้า fastboot (หูฟัง: จอขึ้นกระต่าย/hamburger)
5. กด Unlock ในโปรแกรม → ถ้าขึ้น "wait 168 hours" ให้ **เริ่มนับทันที** (ห้ามล้างเครื่อง/เปลี่ยน account
   ระหว่างรอ ไม่งั้นนับใหม่) → ครบแล้วรันซ้ำ
6. ปลดล็อกสำเร็จ = เครื่องล้าง data ทั้ง `/data` และ `/sdcard` → **แบกอัป `$HOME` ของ Termux ออกไปก่อนเสมอ**

### ทางเลือก Linux-native (ถ้าไม่อยากใช้ Windows)
`MiUnlockTool` — https://github.com/MiForge/MiUnlockTool (`pip install miunlock`) ทำงานเหมือน MiFlashUnlock
แต่รันบน Linux ได้ ใช้ `fastboot oem get_token` → `/api/v3/ahaUnlock` → `encryptData` → `fastboot stage` + `fastboot oem unlock`
**บนเครื่องนี้ใช้ไม่ได้** เพราะขั้นแรก (`get_token`) ถูกถอดออกไปแล้ว

### หลังปลดล็อกได้ (ลำดับงาน)
```bash
fastboot getvar current-slot                 # ปกติ a
fastboot boot twrp.img                       # TWRP ของ darklighter (3.3.2B wzsx150)
adb pull /dev/block/bootdevice/by-name/boot_a ./boot_a.img   # ดึง boot เดิมจากเครื่องเอง ไม่ต้องโหลด ROM
# → Magisk Manager → Install → Select and Patch a File → เลือก boot_a.img
adb pull /sdcard/Download/magisk_patched.img
fastboot flash boot_a magisk_patched.img
fastboot reboot
# bootloop? fastboot --disable-verity --disable-verification flash vbmeta_a vbmeta.img
```
หลัง root: ย้ายการบูตจาก Termux:Boot ไปเป็น Magisk `service.d` script ที่รัน `~/server-up`,
`cmd deviceidle whitelist +com.termux`, และคุม `com.miui.powerkeeper` ได้ตามใจ

---

## 3) ตัวเลข/ข้อมูลอ้างอิงของเครื่องนี้
- `ro.product.mod_device=darklighter_global`, `ro.miui.region=TH`, `ro.miui.cust_variant=th`
- build `DLTR2111050OS00MQ4` / `V11.0.4.0.JOYUI` / security patch `2021-09-01`
- `ro.aais.android.build.version=2021-11-05_sm8150_mp_os_ota1_20200611/darklighter_user_oversea.global.unsign_unroot`
- `fastboot getvar all`: `oem_product:Darklighter`, `oem_stage:PVT`, `cpuid:7e58642`, `kernel:uefi`,
  `max-download-size:805306368`, `slot-successful:a/b:yes`, `slot-unbootable:a/b:no`
- backup `$HOME` ทั้งก้อน (239 MB zstd, 16,063 entries, sha256 `0f50cacba8b616b6208c9401e9c7341f96416fe69cf9b321c0b2456d54eb7544`)
  + `restore.sh` อยู่ใน `~/project/phone-root/backup/`

## 4) บทเรียนที่ได้จาก session นี้
- **ล็อก bootloader ไม่ได้แปลว่าแฟลชอะไรไม่ได้** — signed OTA ทำได้เสมอ
- **`pm disable-user` ทำได้จาก adb โดยไม่ต้อง root** กับแอป system หลายตัว — รวมถึงตัวฆ่าแอปของ MIUI
- **"ตั้งค่า battery ครบแล้ว" ไม่พอ** ถ้า whitelist ของ AOSP มีอยู่แล้วแต่ MIUI ยังฆ่า ⇒ ต้นเหตุอยู่นอก AOSP
- **`getvar all` คือเพื่อน** — 208 บรรทัดบอก slot/oem/cpuid ที่ `getvar <ชื่อ>` เดี่ยว ๆ ตอบไม่ได้
- zsh ไม่ word-split ตัวแปรที่ไม่ได้ quote → `./fastboot $cmd` ที่มีช่องว่างจะกลายเป็น argument เดียว
  (เคยทำให้เข้าใจผิดว่า `oem bs_unlock` เป็น "unknown command" ทั้งที่จริงคือ usage error)
