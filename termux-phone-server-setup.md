**TL;DR:** คู่มือตั้งมือถือ Android ให้เป็น "เซิร์ฟเวอร์ในกระเป๋า" — Termux + sshd + OpenCode :4096 + pinggy tunnel + Cloudflare Worker + Termux:Boot ให้ขึ้นเองหลังรีบูต. จุดที่พลาดกันบ่อยที่สุดคือ **APK ของ Termux:Boot ต้องเซ็นด้วยกุญแจเดียวกับ Termux ที่ลงอยู่** (F-Droid คู่ F-Droid, GitHub คู่ GitHub) ไม่งั้นติดตั้งไม่ผ่าน และ **ต้องถือ wake-lock** ไม่งั้น Android ฆ่า background ทิ้ง.

**Key Points:**
- `~/server-up` = entry point เดียว (idempotent): wake-lock → sshd → OpenCode → tunnel; เรียกจาก boot hook **และ** wrapper ของ `opencode`
- Termux:Boot ต้องโหลดจาก F-Droid (`com.termux.boot_1000.apk`) เมื่อ Termux มาจาก F-Droid — GitHub build เป็น debug key ติดตั้งไม่ผ่าน (`INSTALL_FAILED_SHARED_USER_INCOMPATIBLE`)
- ห้ามตัดสินว่า "พัง" จาก `/proc/net/tcp`, `ss`, `netstat` (อ่านไม่ได้/ไม่มีบน Android 10) — ใช้ `curl` หรือ `/dev/tcp`
- หลังรีบูตต้องรอ **~90 วิ** ก่อนสรุปว่า public URL ล่ม (stack ~40 วิ + Cloudflare KV cache ~60 วิ)

Tags: {#termux} {#android-server} {#opencode} {#tunnel} | Updated: 2026-10-24

---

# ตั้งมือถือ Android เป็น Termux Server (OpenCode + SSH + public tunnel)

> เขียนจากของจริงที่ตั้งบน Black Shark DLT-H0 (Android 10, `192.168.1.114`) วันที่ 2026-10-24
> ทุกขั้นตอนผ่านการรีบูตทดสอบจริง 2 ครั้ง — ใช้เป็น runbook ตั้งเครื่องใหม่ได้ตรงๆ

## 0. ภาพรวมสถาปัตยกรรม

```
                    ┌──────────────────── มือถือ (Termux) ────────────────────┐
  LAN ─────────────►│ sshd :8022 ──► shell (u0_a188)                          │
  192.168.1.114     │ OpenCode :4096  (bound 0.0.0.0, password ~/.oc-pass)    │
                    │   └─ tmux session "oc"  ── ~/oc-run                     │
                    │ pinggy tunnel   (tmux "pg")   → https://xxx.free.pinggy.net
                    │ pgpush          (tmux "pgpush") → POST /__set to worker │
                    └────────────────────────────────────────────────────────┘
                                       │
                    Cloudflare Worker  ▼  https://oc-phone.anut.workers.dev  (URL คงที่)
```

หลักการ: **ทุกอย่างต้องถูกเรียกจาก `~/server-up` ตัวเดียว** และ `server-up` ต้องเรียกซ้ำได้ไม่พัง (idempotent) เพราะมันถูกเรียกจาก 3 ทาง: boot hook, wrapper ของ `opencode`, และมือของมนุษย์

---

## 1. เตรียม Termux (สำคัญ: เลือก "ร้าน" ให้ดี)

ติดตั้ง Termux จาก **F-Droid หรือ GitHub อย่างใดอย่างหนึ่ง แล้วจดไว้** เพราะส่วนเสริม (Termux:Boot, Termux:API) ต้องมาจาก**ร้านเดียวกันเท่านั้น** — เป็นข้อบังคับของ Android (`sharedUserId` + signature)

```bash
# เก็บบันทึกว่าเครื่องนี้มาจากไหน (ทำทันทีหลังติดตั้ง)
echo "version: $(getprop com.termux.version 2>/dev/null) / $(termux-info 2>/dev/null | head -3)"
pm list packages -i | grep com.termux        # ดู installer
pm path com.termux                            # ดู path ของ base.apk
```

จากตัวอย่างเครื่องนี้: `installer=com.google.android.packageinstaller`, APK ที่โหลดมาคือ `com.termux_1022.apk` (F-Droid build) → signer `CN=FDroid`

ตรวจ signer ของ Termux ที่ลงอยู่ (ต้องมี `grun` + openssl ของ glibc):

```bash
BASE=$(pm path com.termux | sed 's/package://')
unzip -Z1 "$BASE" | grep -E '^META-INF/.*\.(RSA|DSA|EC)$'    # ชื่อไฟล์ cert
unzip -p "$BASE" META-INF/84D3000E.RSA > /tmp/c.rsa                # ปรับชื่อตามผลข้างบน
grun $PREFIX/glibc/bin/openssl pkcs7 -inform DER -in /tmp/c.rsa -print_certs -out /tmp/c.pem
grun $PREFIX/glibc/bin/openssl x509 -in /tmp/c.pem -noout -subject -fingerprint -sha256
# → subject=C=UK, ..., O=fdroid.org, OU=FDroid, CN=FDroid
```

## 2. ติดตั้งแพ็กเกจที่ต้องใช้

```bash
pkg update -y && pkg upgrade -y
pkg install -y openssh tmux curl unzip python git nano
pkg install -y glibc-runner          # ให้คำสั่ง grun (รันไบนารี glibc) — OpenCode ต้องใช้
termux-wake-lock                     # เปิดค้างไว้ตลอด
```

> `termux-setup-storage` ไม่จำเป็นสำหรับเซิร์ฟเวอร์ และบนบางเครื่องรันแล้วไม่เกิดอะไร

## 3. ตั้ง IP ให้คงที่

ตั้ง static IP ที่ router (หรือ DHCP reservation) เช่น `192.168.1.114` — เพราะ URL/สคริปต์ทั้งหมดอ้าง IP นี้

```bash
ifconfig | grep 'inet '        # ยืนยัน IP (Android ไม่มีคำสั่ง ip)
ping -c1 192.168.1.1           # gateway ต้องตอบ
```

## 4. เปิด SSH (sshd :8022)

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
# วาง public key ของเครื่องที่จะต่อเข้ามา
cat >> ~/.ssh/authorized_keys <<'EOF'
ssh-ed25519 AAAA...your-laptop-key... laptop
EOF
chmod 600 ~/.ssh/authorized_keys

# เปิด password auth ด้วย (สำหรับเครื่องที่ยังไม่มีคีย์)
grep -q '^PasswordAuthentication yes' $PREFIX/etc/ssh/sshd_config || \
  echo 'PasswordAuthentication yes' >> $PREFIX/etc/ssh/sshd_config

sshd
timeout 4 bash -c 'cat < /dev/tcp/127.0.0.1/8022'    # ต้องเห็น SSH-2.0-OpenSSH_x.y
```

| เรื่อง | ความจริง |
|---|---|
| username | **อะไรก็ได้** (`anut` คือของที่ใช้บ่อย) — Termux ไม่มี system user จริง ทุกชื่อ resolve เป็น uid ของแอป |
| port | 8022 (hardcode ใน Termux) |
| Android 10 | ต้องถือ wake-lock ไม่งั้น sshd ตายเงียบ |

## 5. ติดตั้ง OpenCode (รันผ่าน grun)

```bash
# ติดตั้งตามวิธีปกติของ OpenCode 2.x (จะได้ ~/.opencode/bin/opencode-real)
# เก็บ password ของ server
openssl rand -hex 16 > ~/.oc-pass && chmod 600 ~/.oc-pass
```

**ไฟล์ `~/oc-run`** — ตัวที่รัน server จริง (จด `OC_BIND` ไว้สลับโหมด):

```bash
#!/data/data/com.termux/files/usr/bin/bash
export PATH="/data/data/com.termux/files/usr/bin:$PATH"
export OPENCODE_SERVER_PASSWORD="$(cat /data/data/com.termux/files/home/.oc-pass)"
# 0.0.0.0 = เปิดให้ทั้ง LAN เข้า http://<ip>:4096/ ; ตั้ง OC_BIND=127.0.0.1 เพื่อปิด LAN
BIND="${OC_BIND:-0.0.0.0}"
exec /data/data/com.termux/files/usr/bin/opencode serve --hostname "$BIND" --port 4096
```

**ไฟล์ `~/oc-server`** — ตัวห่อ (สร้าง tmux session "oc", เช็ก health):

```bash
#!/data/data/com.termux/files/usr/bin/bash
tmux kill-session -t oc 2>/dev/null
tmux new -d -s oc "$HOME/oc-run"
sleep 6
curl -s -m 3 -o /dev/null -w "HTTP:%{http_code}\n" http://127.0.0.1:4096/ || true
tmux capture-pane -pt oc | tail -4
```

**ไฟล์ `~/oc-stop`**:

```bash
#!/data/data/com.termux/files/usr/bin/bash
tmux kill-session -t oc 2>/dev/null && echo stopped || echo "not running"
```

**wrapper `$PREFIX/bin/opencode`** — ทำให้พิมพ์ `opencode` แล้วของขึ้นครบเอง:

```bash
#!/data/data/com.termux/files/usr/bin/bash
# Termux wrapper: ถ้า server ไม่มี ให้ดันทั้ง stack ขึ้นก่อนแล้วค่อย attach
U=http://127.0.0.1:4096
H=/data/data/com.termux/files/home
G=/data/data/com.termux/files/usr/bin/grun
R=$H/.opencode/bin/opencode-real
export OPENCODE_SERVER_PASSWORD="$(cat $H/.oc-pass 2>/dev/null)"

for a in "$@"; do case "$a" in --server|--standalone) exec "$G" "$R" "$@" ;; esac; done
case "${1:-}" in
  serve|upgrade|update|uninstall|service|debug|completions|--version|-v|--help|-h) exec "$G" "$R" "$@" ;;
esac

if curl -s -m 2 -o /dev/null "$U/" 2>/dev/null; then
  "$H/server-up" --quiet >/dev/null 2>&1 &
else
  "$H/server-up" >/dev/null 2>&1
fi
exec "$G" "$R" "$@" --server "$U"
```

chmod ให้ครบ: `chmod +x ~/oc-run ~/oc-server ~/oc-stop $PREFIX/bin/opencode`

> `--server` ต้องอยู่**หลัง** subcommand เสมอ

## 6. Tunnel สาธารณะ (pinggy) + Cloudflare Worker ให้ URL คงที่

pinggy ให้ HTTPS ที่ stream SSE ได้ (Cloudflare quick tunnel **ใช้ไม่ได้** — มัน buffer SSE ทำให้ UI ค้าง "Recent sessions: Loading")

**`~/.ocproxy`** (mode 600) — config ของ worker:

```bash
PROXY_URL=https://oc-phone.<subdomain>.workers.dev
PUSH_TOKEN=<random-secret>
```

**ไฟล์ `~/pg`** — start/stop tunnel + pusher:

```bash
#!/data/data/com.termux/files/usr/bin/bash
LOG="$HOME/pg.log"; SESS="pg"; PUSH_SESS="pgpush"
URLRE='https://[a-z0-9.-]+\.free\.pinggy\.(net|io|link)'
case "${1:-status}" in
  start)
    tmux kill-session -t "$SESS" 2>/dev/null; tmux kill-session -t "$PUSH_SESS" 2>/dev/null
    rm -f "$LOG"
    tmux new -d -s "$SESS" "while true; do ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ServerAliveInterval=30 -o ExitOnForwardFailure=yes -p 443 -R0:localhost:4096 free@a.pinggy.io >> $LOG 2>&1; sleep 5; done"
    tmux new -d -s "$PUSH_SESS" "$HOME/pgpush >> $HOME/pgpush.log 2>&1"
    for i in $(seq 1 20); do sleep 2; U=$(grep -aoE "$URLRE" "$LOG" | tail -1); [ -n "$U" ] && break; done
    [ -n "$U" ] && echo "url: $U" || echo "no url yet - check ~/pg.log" ;;
  url)   grep -aoE "$URLRE" "$LOG" 2>/dev/null | tail -1 ;;
  push)  . "$HOME/.ocproxy" 2>/dev/null; U=$(grep -aoE "$URLRE" "$LOG" 2>/dev/null | tail -1)
         [ -n "$U" ] && curl -s -m 10 -X POST -H "x-token: $PUSH_TOKEN" --data "$U" "$PROXY_URL/__set" ;;
  status)
    tmux has-session -t "$SESS" 2>/dev/null && echo "tunnel: up" || echo "tunnel: down"
    tmux has-session -t "$PUSH_SESS" 2>/dev/null && echo "pusher: up" || echo "pusher: down"
    echo "url: $(grep -aoE "$URLRE" "$LOG" 2>/dev/null | tail -1)"
    . "$HOME/.ocproxy" 2>/dev/null; [ -n "$PROXY_URL" ] && echo "stable: $PROXY_URL" ;;
  stop)  tmux kill-session -t "$SESS" 2>/dev/null; tmux kill-session -t "$PUSH_SESS" 2>/dev/null; echo stopped ;;
esac
```

**ไฟล์ `~/pgpush`** — เฝ้า `pg.log` แล้วดัน URL ใหม่เข้า worker ทุกครั้งที่เปลี่ยน:

```bash
#!/data/data/com.termux/files/usr/bin/bash
CFG="$HOME/.ocproxy"; LAST="$HOME/.ocproxy.last"
URLRE='https://[a-z0-9.-]+\.free\.pinggy\.(net|io|link)'
[ -f "$CFG" ] || { echo "no $CFG"; exit 1; }
. "$CFG"
while :; do
  U=$(grep -aoE "$URLRE" "$HOME/pg.log" 2>/dev/null | tail -1)
  if [ -n "$U" ] && [ "$U" != "$(cat "$LAST" 2>/dev/null)" ]; then
    R=$(curl -s -m 10 -X POST -H "x-token: $PUSH_TOKEN" --data "$U" "$PROXY_URL/__set")
    case "$R" in ok*) echo "$(date +%H:%M:%S) pushed $U"; echo "$U" > "$LAST" ;;
                 *)   echo "$(date +%H:%M:%S) push failed: $R" ;; esac
  fi
  sleep 15
done
```

> **Worker source ไม่ได้อยู่บนมือถือ** — deploy จากเครื่องที่มี wrangler (`wrangler deploy`, project `oc-phone`)
> ตัว worker รับ `POST /__set` (header `x-token`) แล้วเก็บ URL ใน KV; ถ้า KV ว่าง/ตายจะโชว์หน้า "phone offline" ที่ auto-refresh เอง

## 7. Termux:Boot — ตัวที่ทำให้ "ขึ้นเองหลังรีบูต"

**นี่คือขั้นที่คนพลาดมากที่สุด** ถ้าไม่มีตัวนี้ ทุกอย่างจะไม่ขึ้นจนกว่าจะมีคนเปิด Termux มือ

```bash
# ⚠️ ต้องมาจากร้านเดียวกับ Termux — F-Droid คู่ F-Droid
curl -L -o /sdcard/Download/tb-fdroid.apk https://f-droid.org/repo/com.termux.boot_1000.apk
```
แล้ว**คนต้องกดติดตั้งบนจอเอง** — จาก Termux ติดตั้งไม่ได้:

```
$ pm install /sdcard/Download/tb-fdroid.apk
Security exception: You either need MANAGE_USERS or CREATE_USERS permission
```
เปิด installer ช่วยได้: `termux-open --content-type application/vnd.android.package-archive /sdcard/Download/tb-fdroid.apk`
(หรือใช้ File Manager แตะไฟล์ — ทางที่ชัวร์สุด)

หลังติดตั้ง ต้องทำ 2 อย่าง:
1. **เปิดแอป Termux:Boot หนึ่งครั้ง** (ถ้าไม่เปิด receiver ไม่ลงทะเบียน)
2. **Settings → Apps → Manage apps → Termux:Boot → Autostart = เปิด** (MIUI/Xiaomi จำเป็นมาก ไม่งั้นมันฆ่า receiver ก่อน `BOOT_COMPLETED`)

ตรวจว่าเข้าแล้ว: `pm list packages | grep termux.boot`

## 8. `~/server-up` — entry point เดียว (ก๊อปทั้งไฟล์)

```bash
#!/data/data/com.termux/files/usr/bin/bash
# server-up — bring the phone server stack up. Idempotent: safe to call any number of times.
#   server-up            full start (waits for the OpenCode server)
#   server-up --quiet    no output; tunnel start detached
# Started by: ~/.termux/boot/start-server.sh, the opencode wrapper, and by hand.
[ -z "$HOME" ] && export HOME=/data/data/com.termux/files/home
export PATH="/data/data/com.termux/files/usr/bin:/system/bin:/system/xbin:$PATH"
H="$HOME"

quiet=0
[ "${1:-}" = "--quiet" ] && quiet=1
say() { [ "$quiet" -eq 1 ] || echo "$@"; }

# log 1 บรรทัดต่อครั้ง: หลักฐานว่าใครเรียกตอนไหน (boot hook vs มือ)
printf '%s  up=%s  caller=%s  pid=%s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  "$(cut -d. -f1 /proc/uptime 2>/dev/null || echo '?')" "${0##*/}$*" "$$" \
  >> "$H/server-up.log" 2>/dev/null || true

# 1. wake-lock — ไม่มีตัวนี้ Android ฆ่าทุกอย่างใน background
termux-wake-lock 2>/dev/null && say "wake-lock: held"

# 2. sshd
if pgrep -x sshd >/dev/null 2>&1; then say "sshd: already up"
else sshd 2>/dev/null && say "sshd: started" || say "sshd: FAILED to start"; fi

# 3. OpenCode :4096
if curl -s -m 2 -o /dev/null http://127.0.0.1:4096/ 2>/dev/null; then say "opencode: already up"
else
  say "opencode: starting..."
  "$H/oc-server" >/dev/null 2>&1
  curl -s -m 3 -o /dev/null -w "opencode: HTTP:%{http_code}\n" http://127.0.0.1:4096/ || say "opencode: FAILED"
fi

# 4. tunnel
if tmux has-session -t pg 2>/dev/null; then
  say "tunnel: already up — $(grep -aoE 'https://[a-z0-9.-]+\.free\.pinggy\.(net|io|link)' "$H/pg.log" 2>/dev/null | tail -1)"
else
  # ตอนบูต WiFi ยังไม่เสร็จ — รอ gateway ก่อน (สูงสุด 60 วิ) ไม่งั้น pinggy ได้แต่ retry เปล่า
  for i in $(seq 1 30); do
    ping -c1 -W1 192.168.1.1 >/dev/null 2>&1 && { say "network: gateway up (${i}x2s)"; break; }
    sleep 2
  done
  # snapshot URL เก่าไว้ก่อน! ~/pg start จะ rm pg.log — ถ้าไม่กัน ตัวรอจะอ่าน URL ของรอบก่อน
  # แล้วดัน URL ที่ตายแล้วขึ้น worker (เกิดจริง 2026-10-24)
  OLDU=$("$H/pg" url 2>/dev/null)
  say "tunnel: starting (bg)"
  nohup "$H/pg" start >/dev/null 2>&1 &
  ( U=""
    for i in $(seq 1 40); do
      C=$("$H/pg" url 2>/dev/null)
      if [ -n "$C" ] && [ "$C" != "$OLDU" ]; then U=$C; break; fi
      sleep 3
    done
    [ -z "$U" ] && U=$("$H/pg" url 2>/dev/null)
    if [ -n "$U" ]; then
      "$H/pg" push >/dev/null 2>&1
      printf '%s  stable-url pushed -> %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$U" >> "$H/server-up.log"
    fi
  ) >/dev/null 2>&1 &
fi

say "--- status ---"
pgrep -x sshd >/dev/null 2>&1 && say "sshd      : up" || say "sshd      : DOWN"
curl -s -m 2 -o /dev/null http://127.0.0.1:4096/ 2>/dev/null && say "opencode  : up" || say "opencode  : DOWN"
tmux has-session -t pg 2>/dev/null && say "tunnel    : up" || say "tunnel    : DOWN"
```

**boot hook `~/.termux/boot/start-server.sh`** (mode 700, ชื่อไฟล์อะไรก็ได้ในโฟลเดอร์นี้):

```bash
#!/data/data/com.termux/files/usr/bin/bash
exec "$HOME/server-up"
```

```bash
mkdir -p ~/.termux/boot
# เขียนไฟล์ข้างบนลงไป แล้ว:
chmod 700 ~/.termux/boot/start-server.sh
chmod +x ~/server-up
```

## 9. ตรวจรับงาน (ทำทุกครั้งหลังตั้งเสร็จ + หลังรีบูต)

```bash
# 1) boot hook ยิงจริงไหม — ต้องมีบรรทัดเวลาไล่เลี่ยกับเวลาบูต
tail -5 ~/server-up.log

# 2) stack
tmux ls                       # ต้องมี oc, pg, pgpush
~/pg status                   # tunnel: up / pusher: up
uptime                        # ใช้เทียบว่า boot เมื่อไหร่

# 3) ปลายทาง
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:4096/              # 200
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.1.114:4096/          # 200
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.1.114:4096/api/info  # 401 = auth ทำงาน (ปกติ!)
curl -s -o /dev/null -w "%{http_code}\n" "$(~/pg url)/"                      # 200
curl -s -o /dev/null -w "%{http_code}\n" https://oc-phone.anut.workers.dev/  # 200
timeout 4 bash -c 'cat < /dev/tcp/127.0.0.1/8022'                            # SSH-2.0-OpenSSH_10.6

# 4) wake-lock ต้องถืออยู่
pgrep -f termux-wake-lock >/dev/null && echo HELD
```

**เกณฑ์เวลา:** หลังรีบูต stack เสร็จ ~40 วิ, public URL พร้อม ~90 วิ (KV cache) — เช็กที่ T+10 วิไม่มีความหมาย

## 10. กับดักที่ต้องรู้ (เสียเวลามาแล้วทั้งนั้น)

| กับดัก | อาการ | ทางแก้ |
|---|---|---|
| Termux:Boot ผิดร้าน | "App not installed" ทันที, ไม่มี dialog | ติดตั้งจากร้านเดียวกับ Termux (F-Droid คู่ F-Droid) |
| ไม่ได้เปิดแอป Termux:Boot | ติดตั้งแล้วแต่ boot ไม่ทำงาน | เปิดแอปหนึ่งครั้งหลังติดตั้ง |
| MIUI Autostart ปิด | บูตแล้วบางทีขึ้นบางทีไม่ขึ้น | เปิด Autostart ให้ Termux:Boot |
| ไม่ถือ wake-lock | ทุกอย่างตายเงียบๆ หลังผ่านไปพัก | `termux-wake-lock` ใน `server-up` |
| `pkill -f sshd` | ฆ่า shell ตัวเองตาย (command line มีคำว่า sshd) | ใช้ `pkill -x sshd` |
| ตรวจ port ด้วย `/proc/net/tcp` | ไฟล์ว่าง/อ่านไม่ได้ → เข้าใจผิดว่าตาย | ใช้ `curl` หรือ `/dev/tcp` |
| `ss` / `netstat` | command not found | ใช้ `curl`/`/dev/tcp` หรือ `tmux ls` |
| `pm install` จาก Termux | `SecurityException: need MANAGE_USERS` | ให้มนุษย์กดติดตั้งบนจอ |
| `dumpsys` / `logcat` | permission denied | อย่าใช้ ให้ log ตัวเอง (`server-up.log`, `pg.log`) |
| `/tmp` | Read-only file system | ใช้ `$PREFIX/tmp/opencode/` |
| pinggy free | URL หมดอายุ 60 นาที | `~/pg` มี loop ต่อเอง + `pgpush` ดัน URL ใหม่ขึ้น worker |
| Cloudflare KV cache | URL ใหม่แล้วแต่ยังโชว์ "phone offline" ~60 วิ | รอ 60–90 วิ ก่อนสรุปว่าล่ม |
| push URL เก่าตอนบูต | worker ชี้ท่อที่ตายแล้ว | snapshot URL เก่าแล้วรับเฉพาะ URL ที่ต่าง (อยู่ใน `server-up` แล้ว) |
| Tailscale | `100.x.x.x` ไม่ตอบหลังรีบูต | **ไม่ได้อยู่ใน boot hook** โดยเจตนา — ต้อง `~/ts start` เอง |
| Cloudflare quick tunnel | UI ค้าง "Recent sessions: Loading" | ใช้ pinggy (SSE ผ่าน) |

## 11. Recovery: อาการ → ทำอะไร

| อาการ | สาเหตุที่น่าจะเป็น | คำสั่ง |
|---|---|---|
| SSH เข้าไม่ได้ | sshd ตาย | `sshd` หรือ `~/server-up` |
| LAN :4096 ไม่ตอบ | server ตาย | `~/oc-server`, ดู `tmux capture-pane -pt oc` |
| public URL ขึ้น "phone offline" | tunnel ตาย หรือ URL ใหม่ยังไม่ push | `~/pg status` → `~/pg start` → `~/pg push` |
| ทุกอย่างขึ้นแต่ไม่ขึ้นเองหลังรีบูต | boot hook ไม่ยิง | `tail ~/server-up.log` (ไม่มีบรรทัดใหม่ = hook ไม่ทำงาน) → เช็ก Termux:Boot + Autostart |
| public URL ชี้ไปท่ออื่น | push ผิด/ค้าง | `~/pg push` แล้วรอ 60 วิ |

---

## ภาคผนวก: หลักฐานจากเครื่องจริง (2026-10-24)

| เวลา (UTC) | เหตุการณ์ |
|---|---|
| 04:41:43 | boot (reboot ครั้งที่ 1) |
| 04:42:20–22 | opencode server ขึ้นเอง (39 วิหลังบูต) — ก่อนมี client ใดๆ 3 นาที 50 วิ |
| 04:46:12 | client ของผู้ใช้ attach (พิสูจน์ว่าไม่ใช่คนไปสั่งสตาร์ท) |
| 04:47 | เพิ่ม logging เข้า `server-up.log` |
| 04:57:06 | boot ครั้งที่ 2 |
| 04:57:31 | `server-up` ถูกเรียกจาก hook (25 วิหลังบูต) |
| 04:57:39 | push URL — **เจอบั๊กดัน URL เก่า** → แก้ด้วย snapshot (หัวข้อ 8) |
| 04:59 | stable URL 200 (TTFB 0.50 s), LAN 200, sshd banner ok |

บทเรียนหลัก: **"เซิร์ฟเวอร์ไม่ขึ้น" เกือบทุกครั้งไม่ใช่เซิร์ฟเวอร์พัง แต่คือ "ไม่มีใครสั่งให้มันขึ้น"** — ต้องมีหลักฐานว่า hook ยิงหรือไม่ (`server-up.log`) ก่อนไปแก้ที่อื่น
