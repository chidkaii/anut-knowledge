**TL;DR:** อ่านแบตบน Android 10 ที่ไม่ root ต้องมีแอป **Termux:API** (`com.termux.api`) เท่านั้น — `/sys/class/power_supply` ถูกบล็อก แต่การติดตั้ง APK จะโดน **Play Protect** บล็อกถ้าโหลดเอง ทางที่ผ่านคือ `adb install -r` จากเครื่องอื่น. จุดหลอกที่สำคัญ: ถ้าแอปยังไม่ติดตั้ง `termux-battery-status` **ไม่ error — มันค้างจน timeout** จึงต้องเช็ค `pm list packages com.termux.api` (37 ms) ก่อนยิงคำสั่งเสมอ.

**Key Points:**
- APK ต้องเป็น**ลายเซ็นร้านเดียวกับ `com.termux`** (เครื่องที่ลงจาก F-Droid → ใช้ `com.termux.api_1002.apk` จาก F-Droid, signer `CN=FDroid`) ไม่งั้นได้ `INSTALL_FAILED_SHARED_USER_INCOMPATIBLE`
- Play Protect ขึ้น "unsafe app" = Google ไม่รู้จักแอป ไม่อยู่บน Play — **อย่าปิด Play Protect** ให้ติดตั้งผ่าน ADB แทน (`adb install -r` ไม่ผ่านตัวสแกน)
- คำสั่ง Termux:API **ทุกตัว** (battery/toast/vibrate/clipboard…) ค้างจนครบ timeout เมื่อแอปหาย → ตรวจด้วย `pm list packages com.termux.api` ไม่ใช่การลองยิงแล้วรอ
- อุณหภูมิ: `/sys/class/thermal/thermal_zone*/temp` อ่านได้ แต่ต้องกรอง `lmh-dcvs-*` (= trip limit `75000` ไม่ใช่อุณหภูมิ) และ `soc` (= `274000` หน่วยเพี้ยน) — เอาเฉพาะ `cpu|gpu|apc|skin` ในช่วง 15–95 °C
- cron ที่ใช้จริง: `heartbeat.sh` (*/5) เก็บ `up/mem/sshd/oc/temp/disk/batt` เป็น history, `battery-guard.sh` (*/10) เตือน Telegram เมื่อ `batt <= 20%` และไม่ได้ชาร์จ หรือ `health != GOOD` (แคชสถานะกันสแปม 1 ชม./24 ชม.)

Tags: {#termux} {#android-server} {#battery} {#adb} | Updated: 2026-10-24

---

# อ่านสถานะแบตบน Termux (Android 10, ไม่ root) + ติดตั้ง Termux:API ให้รอดจาก Play Protect

> บันทึกจาก session 2026-10-24 · Black Shark 2 Pro (DLT-H0 / `darklighter_global` / Android 10 / MIUI V11 / region TH)
> เป้าหมาย: ให้ HUD/บอทบนโฟนรู้ระดับแบต เพื่อเตือนก่อนที่ไฟจะหมด (โฟนทำหน้าที่เป็น always-on server)

## 1. ทำไม `battery_status()` ขึ้น "API ไม่พร้อม" ทั้งที่ติดตั้ง `termux-api` แล้ว

```bash
$ dpkg -l | grep termux-api
ii  termux-api  0.60.0        # CLI มี
$ pm list packages | grep termux
package:com.termux             # ← แอป com.termux.api ไม่มี!
package:com.termux.boot
$ termux-battery-status
                               # ค้าง ไม่ตอบ ไม่ error (exit 124 = timeout)
```

คำสั่งทุกตัวของ Termux:API ทำงานด้วยการ **broadcast ไปหาแอป `com.termux.api`** ถ้าไม่มีแอป ก็ไม่มีใครตอบ
⇒ process ค้างจนครบ timeout ⇒ โค้ดที่ใช้ `timeout=3` จะได้สตริงว่าง แล้วรายงานว่า "API ไม่พร้อม" ตลอดกาล
(และถ้าใครไปเรียกใน poll loop ทุก 3 วิ จะกินเวลาเปล่า 2–3 วิต่อครั้ง)

**วิธีตรวจที่ถูก:** ถาม package manager ตรง ๆ (37 ms)

```bash
pm list packages com.termux.api        # มี output = มีแอป, ว่าง = ไม่มี
```

## 2. ติดตั้ง APK: กับดัก 2 ชั้น

| ชั้น | อาการ | ทางออก |
|---|---|---|
| ลายเซ็นไม่ตรง | `INSTALL_FAILED_SHARED_USER_INCOMPATIBLE` | ใช้ APK จากร้านเดียวกับ Termux (F-Droid คู่ F-Droid) |
| Play Protect | "unsafe app blocked" (Google ไม่รู้จักแอปนอก Play) | **อย่าปิด Play Protect** — ติดตั้งผ่าน ADB |

```bash
# บนเครื่องคอม (ไม่ต้อง root/sudo ถ้า /dev/bus/usb มี ACL ของ seat user)
curl -LO https://f-droid.org/repo/com.termux.api_1002.apk
curl -L -o pt.zip https://dl.google.com/android/repository/platform-tools-latest-linux.zip
unzip -q pt.zip && ./platform-tools/adb devices     # ต้องขึ้น "device" (ไม่ใช่ "unauthorized")
./platform-tools/adb install -r com.termux.api_1002.apk
```

ยืนยันลายเซ็นก่อนติดตั้งได้:

```bash
unzip -o -q app.apk 'META-INF/*.RSA' && \
openssl pkcs7 -inform DER -in META-INF/*.RSA -print_certs -out c.pem && \
openssl x509 -in c.pem -noout -subject -fingerprint -sha256
# subject=C=UK, ..., O=fdroid.org, OU=FDroid, CN=FDroid
```

หลังติดตั้ง:

```bash
termux-battery-status
{"present":true,"technology":"Li-ion","health":"GOOD","plugged":"PLUGGED_USB",
 "status":"CHARGING","temperature":34.0,"voltage":4230,"current":-27343,
 "percentage":92,"level":92,"scale":100,"charge_counter":2686786}
```

## 3. อ่านอุณหภูมิโดยไม่โดนหลอก

`/sys/class/power_supply` = **Permission denied** (ไม่มี root) จึงต้องใช้แอป แต่ `/sys/class/thermal` อ่านได้:

```bash
for z in /sys/class/thermal/thermal_zone*; do
  n=$(cat "$z/type"); t=$(cat "$z/temp")
  case "$n" in *cpu*|*gpu*|*apc*|*skin*) ;; *) continue ;; esac   # กรองหลอก
  c=$((t/1000)); [ "$c" -ge 15 ] && [ "$c" -le 95 ] && echo "$c $n"
done | sort -rn | head -1
```

เซนเซอร์ที่ **ห้ามเอามาใช้ตรง ๆ**:

| zone | ค่า | ความหมายจริง |
|---|---|---|
| `lmh-dcvs-00/01` | `75000` (75 °C) | **trip limit** ของ hardware ไม่ใช่อุณหภูมิปัจจุบัน |
| `soc` | `274000` | หน่วยเพี้ยน (อ่านตรง ๆ ได้ 274 °C) |

ถ้าไม่กรอง HUD จะโชว์ 75 °C หรือ 274 °C ตลอดเวลา = เตือนผิดตลอด.

## 4. เอาไปใช้กับ cron (ของจริงที่รันอยู่)

`~/project/cron/heartbeat.sh` (*/5 นาที) เขียนหนึ่งบรรทัดต่อรอบ — ใช้เป็น history ราคาถูก ไม่ต้องมี DB:

```
2026-10-24T18:40:01Z  up= 6:53  mem=5997MB  sshd=up  oc=up  temp=48C  disk=210G  batt=88
```

`~/project/cron/battery-guard.sh` (*/10 นาที):

- `batt <= 20%` **และไม่ได้ชาร์จ** → Telegram (เว้น 1 ชม. ต่อครั้ง)
- `health != GOOD` → Telegram (เว้น 24 ชม.)
- อ่านค่าไม่ได้ (แอปถูก MIUI ฆ่า/ถอน) → เงียบ + log 1 บรรทัด/ชม.
- มี `--dry` และ `BATTERY_JSON='{...}'` ให้ทดสอบโดยไม่ต้องรอแบตหมดจริง

```bash
BATTERY_JSON='{"percentage":15,"status":"DISCHARGING","health":"GOOD","temperature":31,"plugged":"UNPLUGGED"}' \
  ./battery-guard.sh --dry
```

## 5. ข้อควรระวังระยะยาว

- **อย่าถอนแอป Termux:API** — ถอนแล้วทุกอย่างกลับไปค้างที่ timeout และค่าว่างทันที
- ตั้ง **MIUI Autostart ON** ให้ `com.termux.api` ด้วย ถ้า MIUI ฆ่ามัน ค่าแบตจะเป็น `?` (ระบบไม่พัง แต่เงียบ)
- อย่าใช้การ "ลองยิงคำสั่งแล้วดูว่าตอบไหม" เป็น health check — ใช้ `pm list packages` (ถูกกว่า 50 เท่า)
