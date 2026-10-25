**TL;DR:** OpenCode bridge ล่มเงียบได้ 2 ทางที่เจอจริง — HUD เขียน state ข้ามไฟล์ (sessions หายทุกนาที) และ SSE ที่ไม่มี watchdog (heartbeat ทำให้ timeout ไม่ยิง → runner ค้างตลอดกาล). แก้ด้วย `bind_state()` ต่อ process, guard ไม่เขียน state บางส่วน, watchdog ปิด stream เมื่อไม่มี delta, และปิด summarizer session ให้มีโมเดล (`model: None` = ตอบ prose ไม่ใช่ JSON → unparsed ทุกนาที).

**Key Points:**
- **อาการหลอก:** คำตอบถูกสร้างแล้ว (มีใน session) แต่ไม่โผล่ในแชท — อย่าดูแค่ "โมเดลไม่ตอบ" ให้ดู `run/<bot>.busy` อายุค้าง + bubble pulse ที่ยังเป็น ⚡
- **เขียน state ข้ามไฟล์:** `write_state(st)` ไม่ระบุ path → default `run/state.json`; caller ที่ยืม state ของอีกบอท (HUD brief อ่าน `state-discord.json`) จะทับ Telegram จน `sessions/offset/anchors` หาย → restart ถัดไปได้ session ใหม่ + offset=0 (เสี่ยง replay update เก่า)
  - แก้: `bind_state(path)` ประกาศ default ต่อ process + guard "ถ้า dict ไม่มี `sessions`/`offset` ห้ามเขียน"
- **SSE ค้าง:** `urlopen(timeout=30)` คุมแค่ read ครั้งเดียว; `: heartbeat` ทุก ~25 วิ ทำให้ socket ไม่เงียบ → timeout ไม่ยิง; stall check ที่อยู่หลังการอ่านบรรทัดจึงไม่มีวันทำงาน
  - แก้: watchdog thread ปิด `resp` เมื่อไม่มี delta ครบ N วิ (ทำให้ iterator ปลดล็อก) + เช็ค stall บน heartbeat line + `HARD_LIMIT` + **หลัง stall ต้อง `break` ไม่ใช่ reconnect** (ถ้าไม่ break จะวน log รัวไม่จบ)
  - ทดสอบได้: ทำ stall ให้ตั้งผ่าน env (`OC_STALL=3`) แล้วป้อน response ปลอม 2 แบบ (heartbeat-only / เงียบสนิท) → ต้องคืนภายใน ~4 วิ
- **Summarizer `model: None`:** session สรุปไม่ได้ตั้งโมเดล → ใช้ default ของ server → ตอบ prose แทน JSON array → `unparsed` ทุกนาที และ `/api/brief` ค้าง >90 วิ
  - แก้: ปักหมุด `summarizer_model` หรือ `default_model` + log `raw[:100]` ของคำตอบที่ parse ไม่ได้ (เดิม log บอกแค่ "unparsed")
  - ผล: summarize 2 items 7.8 วิ · `/api/brief` 13.1 วิ (จาก >90 วิ timeout)
- **Restart ให้ถูกตัว:** `reload.sh` รอ busy marker ที่อาจไม่มีวันเคลียร์ → ใช้สคริปต์ต่อบอท (`tg-restart.sh` / `dc-restart.sh`) และลบ marker ก่อน start
- **process ซ้อน:** `pgrep -f discord_bot.py` เจอ 2 ตัวได้ (ตัวหลุดไม่มีใน pid file) → เทียบกับ `run/discord.pid` ทุกครั้ง

Tags: {#opencode} {#bridge} {#telegram} {#discord} {#sse} {#termux} {#debugging} | Updated: 2026-10-26

# OpenCode Bridge — State Cross-Writes & Hung SSE Streams

## 1. Symptom that lies

The model *did* answer (message exists in the OpenCode session), yet the chat shows only the
loading pulse (`⚡`). Two independent failures look identical from the chat:

1. `run/<bot>.busy` never cleared → the runner thread is wedged.
2. The bubble was never finalized → nothing replaces the pulse.

Always check, in this order: busy-marker age → session messages via `/api/session/{id}/message`
→ `state.json` size → number of processes per bot.

## 2. State cross-write (the silent one)

`bridge_core.write_state(st)` with no `path` writes to the module-level `STATE`
(`run/state.json`, Telegram's). Any caller that borrowed another bot's dict — the HUD's brief
reads `state-discord.json` and then calls the summarizer — silently overwrites Telegram's state
with Discord's (or with `{}` when the read fails).

Consequences: `sessions`, `offset`, `anchors` gone. Next restart starts a fresh session and
`offset=0`, which can replay up to 24 h of Telegram updates.

Fix pattern:

```python
# per process, at startup
bridge_core.bind_state(RUN / "state.json")        # telegram
bridge_core.bind_state(STATE)                     # discord → its own file

# inside any helper that persists: refuse partial dicts
if "sessions" in st or "offset" in st:
    write_state(st)
else:
    cur = read_state(); cur["k"] = v; write_state(cur)
```

Splitting state per *session* does not help — the bug is writing the wrong dict to the wrong
file, not file size.

## 3. SSE watchdog (the one that hangs forever)

`urllib.request.urlopen(req, timeout=30)` bounds a single socket read. OpenCode keeps the
connection warm with `: heartbeat` comments, so the timeout never fires, and a stall check
placed *after* a line is parsed never runs at all when no events arrive.

Required shape:

```python
stop_watch = threading.Event()
def _watchdog(resp):
    while not stop_watch.wait(2.0):
        if done or paused_form:
            return
        if time.monotonic() - last_delta > STALL or time.monotonic() > hard_deadline:
            resp.close()          # unblocks the iterator
            return
```

Then, after `except`, **break out of the reconnect loop when stalled** — reconnecting a stalled
run loops the log forever (hit this while testing: 28 identical lines and no return).

Test without waiting 2 minutes: make the stall configurable (`OC_STALL=3`) and feed two fake
responses — heartbeats only, and completely silent. Both must return in a few seconds with
`completed=False`.

## 4. Summarizer without a model

A dedicated summarizer session created via the API has `model: None`, so it uses the server
default and answers prose instead of the requested JSON array → `unparsed` fallback every
minute, and `/api/brief` blocks past its timeout. Pin the model (`default_model`) before
prompting, and log `raw[:100]` of the unparsed reply — without it the failure is undiagnosable.

## 5. Telegram transport: never hand-shake per call

`urllib` opens a fresh TLS connection for every API call: measured **850 ms** per
`editMessageText`, once **16.7 s** when the handshake stalled — and the first call of a run
is the `⚡` pulse, so the chat looked frozen while the bridge was healthy.

- Keep one `http.client.HTTPSConnection` alive, guarded by a lock, with a short timeout and
  one retry on a fresh socket (bounds a stall instead of letting it hang).
- Keep it **warm**: ping `getMe` every ~20 s. A 45 s cadence was too slow — Telegram drops
  idle sockets sooner, so the pulse paid a handshake every time (819 ms vs ~300 ms warm).
- Leave `getUpdates` on the urllib path: long polling needs a 50–70 s timeout.
- Measure, don't guess: log `[tg->log Ns]` (Telegram→handler) and `pulse after Nms`
  (handler→pulse). `tg->log 0s` proved the poll loop was innocent.
- `sendMessage` + `deleteMessage` instead of editing is **worse**: 835 ms/cycle vs 479 ms,
  two API calls, plus notification churn and holes in chat history.

## 6. Telegram will delete the whole message over one bad tag

`editMessageText`/`sendMessage` with `parse_mode=HTML` reject the ENTIRE message when a tag is
unbalanced (`can't parse entities: Can't find end tag corresponding to start tag "b"`). A long
answer chunked at a fixed length splits `**bold**` across the boundary and produces exactly that.

- Validate tag balance per chunk before sending; if it fails, send plain text.
- Answers over ~1500 chars: skip HTML entirely — chunk boundaries are unpredictable.
- Read `retry_after` from a 429 and actually wait. Retrying immediately turned one 400 into
  18 s of 429s, and then nothing could be delivered at all.
- **Never delete the old bubble before the new content is delivered.** Send first, delete after;
  otherwise a parse error plus a 429 leaves the chat completely empty.

## 7. Two-bubble layout: content on top, animation below

Users want the animation where their eye is (the newest message) and the content in a stable
place. Messages only append downward, so send the ANSWER bubble first, then the STATUS bubble:
the status stays the newest message while the answer grows above it, and it is deleted when the
run ends. Keep the model's reasoning out of the status bubble — it was read as "the answer".

## 8. Restart discipline

`reload.sh` waits for `run/telegram.busy` to clear, which may be exactly the marker that is
wedged. Use one-bot restart scripts that delete the marker first, and never restart the bot
whose reply is currently in flight.
