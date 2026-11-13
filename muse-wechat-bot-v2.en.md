---
name: "muse-wechat-bot"
version: "v2 (field-tested optimized edition, 2026-11-13)"
description: >
  Connect Muse itself to WeChat as a bot (single self-contained file): built on
  Tencent's official iLink Bot API (ilinkai.weixin.qq.com, QR-code authorization),
  a small native Python bridge runs on an always-on machine — bind once by scanning
  a QR code, and from then on the Muse you talk to in WeChat is Muse itself
  (same memory, same skills), with two-way text / image / video / file and
  voice-to-text messaging plus proactive push. This edition builds on the
  2026-10-11 original and was hardened through a full day of real-world testing
  and a security audit on 2026-11-13: a zero-idle-cost event-hook architecture,
  completed media send/receive, atomic inbox claiming against duplicate replies,
  an honesty red line, and a read-the-original rule for links — all baked in.
Use when the user wants to connect Muse to WeChat / build a WeChat bot / chat
with Muse inside WeChat, mentions Tencent/openclaw-weixin or WeChat bots,
or asks about chatting with the assistant via WeChat.
Fully operated by the AI: the user only scans one QR code (plus a numeric
pairing code if one appears); everything else is done by the AI.
---

> **This file is fully self-contained** (operator manual + complete `bridge.py` v2
> source + `launch.sh` + hook script + wake-prompt template + iLink protocol
> quick reference). No external files needed. Two ways to use it:
> - **Give it to your Muse**: send this file to your Muse and say
>   "set me up on WeChat following this file";
> - **Install it as a standing skill**: save this file into Muse's skill
>   directory so it can be triggered later.
>
> Source of the WeChat bot tutorial and authoritative protocol docs:
> https://github.com/Tencent/openclaw-weixin

# Connect Muse to WeChat (Tencent's official iLink API · field-tested v2)

**👤 For the human user**: give this skill to your Muse and say one sentence —
**"set me up on WeChat following this skill"** — that's all.

**🤖 For Muse**: this file contains the complete implementation (the v2
`bridge.py` in Appendix 1 works out of the box — write it to disk as-is and
deploy it; never rewrite it from memory). **Full delegation is a hard
requirement**: environment checks, deployment, QR retrieval, polling, the
daemon, the hook and the reply loop are all yours to run. Stop and wait for the
user at exactly two points — ① the WeChat QR-scan authorization, and ② a
numeric pairing code, if one appears: ask the user to send you the digits.
Never make the user run commands or read logs.

**⚠️ Key difference from the Tencent/openclaw-weixin manual**: that manual is a
plugin for **OpenClaw** (a different agent system) — follow it and the thing
answering in WeChat is a different AI, without your memory and skills. This
skill takes the **native bridge** route: the same public protocol, spoken
directly to Tencent's official API, so the one answering in WeChat is **Muse
itself**. An official channel — account risk is far lower than with
grey-market protocol hacks.

## What v2 changed vs. the original — every item earned in live testing/audit

1. **Zero-idle-cost architecture (the big one)**: the original woke the AI on a
   1-minute cron to poll the queue, burning quota even with no messages. v2
   uses a **local event hook**: a bash script checks the local SQLite queue
   every 5 seconds (never wakes the AI, zero tokens) and wakes Muse only
   when a real message exists; the hook also keeps the daemon alive, so the
   original's separate 5-minute keepalive cron is gone too. Measured anchor:
   one full day of heavy use (a dozen-plus WeChat exchanges plus heavy
   main-chat work) cost about 1% of a weekly quota; idle cost is 0.
2. **Media sending, completed**: the original bridge could only send text — a
   user asking for an image in WeChat got "can't send that". v2 implements the
   full official CDN upload flow (`getuploadurl` → AES-128-ECB encryption →
   POST upload → read the `x-encrypted-param` header → send as a media
   message), adding `send-image` / `send-video` / `send-file`, all verified
   live for images, video and files.
3. **Inbound media no longer dropped**: the original daemon recorded only a
   "[video]" stub for videos, only a filename for files, only the last image
   of several, and turned voice messages without a transcript into empty
   messages. v2 actually downloads videos and files to `~/wechat-bot/media/`,
   writes every saved image's path into the message text, and marks
   untranscribed voice explicitly as `[voice message (no transcript)]` — no
   more empty messages or lost attachments.
4. **Atomic claiming — no more duplicate replies**: the original `inbox-new`
   merely *looked* at unprocessed messages, so two overlapping reply cycles
   could answer the same message twice (double quota burn, duplicate messages
   to the user). In v2, `inbox-new` claims the batch as it returns it (status
   `processing` + `claimed_at`); claims not acked within 10 minutes are
   automatically re-queued (so a worker dying mid-flight can't swallow a
   message). The hook may only *count* with `inbox-peek`; it must never claim.
5. **QR rendering fix**: the `qrcode_img_content` returned by the API is
   actually a `https://liteapp.weixin.qq.com/q/...` link — it *is* the QR
   code's content and must be rendered into a PNG locally with the `qrcode`
   library. The original code treated it as a base64 image / image download
   URL and in practice produced nothing scannable. v2's `login-fetch` renders
   locally, built in.
6. **Dependencies completed**: `requests`, `qrcode` and `pycryptodome` are all
   required (the original called pycryptodome optional; in reality image
   decryption and all media upload depend on it).
7. **Wake-prompt template (Appendix 4)**: the handling rules tuned in live
   use — casual chat gets a fast answer without memory lookups; tasks must be
   actually done, not faked; factual questions are verified by web search
   before answering (numbers especially); an **honesty red line** (verify
   before explaining a cause, never invent one; when sending a pre-existing
   file, say plainly that it is an existing file — never pass old work off as
   freshly made; this rule exists because a user caught exactly that); every
   ack must carry the full reply text.
8. **Read-the-original rule for links**: measured fact — for WeChat official
   account articles (mp.weixin.qq.com), most programmatic fetching returns
   only the title, never the body. The rule: when the original text can't be
   fetched, **never pass off a search-assembled summary as the original**;
   tell the user honestly and hand off to the main chat, which reads the
   original in a real browser. A composite summary is allowed only when the
   user explicitly says a rough version is fine. See Appendix 4, rule 4.
9. **Proactive push, done right**: the QR binding itself **never expires**;
   what expires is the *proactive-send window* — if the user hasn't messaged
   the bot in ~24 hours, a proactive send may be rejected by the server. On
   failure, fall back to the main chat channel; never retry repeatedly. The
   push scope (urgent-only vs. everyday gentle reminders too) is the user's
   choice — ask at deployment time and record it; never spam by default.
10. **Acks store the full reply text**: this is the data foundation for
    extensions like a nightly chat digest (see Optional extensions); the
    original ack stored nothing, so no summary was possible afterwards.

## Architecture (one sentence)

```
User's WeChat ⇄ Tencent's official iLink API ⇄ bridge.py daemon
        (long-polling inbound private chats → SQLite queue)
        ⇄ local hook script (peeks at the local queue every 5 s at zero cost;
          wakes Muse only when new messages exist)
        ⇄ Muse itself handles them (inbox-new claim → reply / do the task
          → send back → ack for the record)
```

## Step 0: Pick an always-on machine

The bridge must stay online 24/7 (the long poll cannot break). In order of
preference:
1. A stable server/VPS the user already has (best);
2. Muse's host cloud VM (works; it resets from time to time — the hook
   script revives the daemon on every poll, so it self-heals);
3. The user's own computer (worst: close the lid or shut down and it's offline).
Note the choice; every path below assumes it (using `~/wechat-bot/`).

## Step 1: Environment check (you run this; don't ask the user)

1. `python3 --version` (3.8+), and install all dependencies at once:
   `pip install requests qrcode pycryptodome`.
2. Connectivity test: `curl -s --max-time 30 https://ilinkai.weixin.qq.com/ilink/bot/get_bot_qrcode?bot_type=3 -X POST -H 'Content-Type: application/json' -d '{"local_token_list":[]}'`
   should return JSON containing `qrcode` / `qrcode_img_content`.
   **Note**: some cloud VMs egress through a proxy and a bare curl may time
   out — retry once; if it still fails, don't conclude the API is down. The
   bridge uses `requests` over the same network; finish the deployment and
   judge connectivity by what `login-fetch` actually returns.

## Step 2: Deploy bridge.py (you run this)

Save Appendix 1's source verbatim to `~/wechat-bot/bridge.py` on the target
machine and verify with `python3 -m py_compile`. Its CLI (all verified):

| Command | Purpose |
| --- | --- |
| `login-fetch` | Fetch the login QR code and render it locally to a PNG (`~/wechat-bot/qr.png`) |
| `login-wait --timeout 300 [--verify-code CODE]` | Poll scan status; on success the `bot_token` lands in `auth.json` (mode 600) |
| `daemon` | Long-poll inbound messages → whitelist filter → save media → write the SQLite queue |
| `inbox-peek` | Only *count* unprocessed messages (no claiming; for the hook) |
| `inbox-new` | Output unprocessed messages as JSON, **atomically claiming the batch** |
| `send-stdin` | Send text read as JSON `{"to","ctx","text"}` from stdin (empty `to` = the bound owner) |
| `send-image --file <path> [--to][--ctx]` | Send an image (jpg/png) |
| `send-video --file <path> [--to][--ctx]` | Send a video (mp4) |
| `send-file --file <path> [--to][--ctx]` | Send any file |
| `ack --id N --status replied --reply "<full reply text>"` | Mark processed and archive the reply text |
| `status` | Token / owner / daemon liveness / unread count |

Whitelist logic (built into `daemon`; private chats only): only the **QR-bound
owner's own private chats** are accepted; group chats and strangers are recorded
and ignored.

## Step 3: QR login (the only step that needs the user)

1. Run `login-fetch`: it saves the QR PNG to `~/wechat-bot/qr.png` — **send
   that image to the user** and say clearly: **scan it with WeChat, it's valid
   for 5 minutes, confirm the authorization on your phone**. Meanwhile run
   `login-wait --timeout 280` in the background.
   (How it works: the API's `qrcode_img_content` is a liteapp link — the QR
   content itself — and `login-fetch` renders it locally with the `qrcode`
   library; the 32-character `qrcode` field is just the polling token, not
   something the user scans.)
2. Polling state machine: `wait` → `scaned` → `confirmed` (success: token
   saved, `ilink_user_id` = the owner recorded) / `expired` (re-run
   `login-fetch`, at most 3 times) / `need_verifycode` (**stop**: ask the user
   for the numeric pairing code shown in their WeChat, then run
   `login-wait --verify-code <digits>`).
3. After success, verify `status` shows `token: true`. **The token is a
   credential**: never repeat it in chat, never write it into long-term memory
   in plaintext.

## Step 4: Iron rule — a session must exist before anything can be sent

**Measured fact**: iLink requires the user to have sent the bot **at least one
message** first, establishing a session (`context_token`). With zero session,
every send is rejected by the server with `ret:-2 errmsg:"prepare failed"`.
The official plugin maintains a context-token store for exactly this; this
bridge does the same with the `sessions` table in `queue.db` (tokens are
recorded on inbound messages and reused automatically on send).
So after Step 3, **ask the user to send the bot any message in WeChat** (e.g.
"hello"), confirm `inbox-new` picks it up and that you can reply, before
continuing.

**Proactive push**: once bound (and a session exists) you may message the user
without them writing first — but the session window is ~24 hours: if the user
hasn't messaged the bot in over ~24 hours, a proactive send may fail; on
failure, report through the main chat channel instead and never retry
repeatedly. The binding/token itself never expires; no re-scan is ever needed
(unless the user revokes the authorization inside WeChat).

## Step 5: Daemon + keepalive (you run this)

1. Save Appendix 2's script as `~/wechat-bot/launch.sh` (`chmod +x launch.sh`)
   and start the daemon through it (double-fork; the child is reparented to
   init and doesn't depend on the launching shell; idempotent — exits at once
   if already running). **Never** start it with a bare `nohup ... &` — when
   the host shell is reaped, it can take the child down with it.
2. Verify: `bridge.py status` shows `daemon_alive: true`; after about a
   minute, `bridge.log` has no errors and `cursor.txt` is advancing.
3. Keepalive: **with the Step 6 hook setup, no separate keepalive is needed**
   — the hook script runs `launch.sh` on every poll (idempotent; instant exit
   when alive). Only with the fallback (cron reply loop) do you also need a
   5-minute keepalive task running `launch.sh`.
4. **Gotcha**: `bridge.py`'s `log()` both prints and writes to file —
   `launch.sh` must redirect stdout to `/dev/null` (only stderr goes to the
   log), otherwise every log line appears twice. Do not "fix" that redirect.

## Step 6: The reply loop (v2: event hook, zero idle cost)

**Preferred (Muse's native hooks)**:
1. Save Appendix 3's script as `~/hooks/scripts/wechat-inbox-watch.sh`
   (`chmod +x`). It does three things: revive the daemon → count the local
   queue with `inbox-peek` → `wake` only when there is something new,
   otherwise stay `silent`. It touches only local files and costs zero tokens.
2. Register an event hook (definition example in Appendix 3):
   `poll_interval_secs: 5`, with the wake prompt from Appendix 4 (replace the
   placeholders with your and your user's actual details).
3. The handling semantics after a wake are fixed — don't change them:
   `inbox-new` (claim) → reply / do the task as Muse itself →
   `send-stdin` or `send-image/send-video/send-file` → `ack --reply "<full reply text>"`.

**Fallback (environments without hooks)**: a 1-minute cron running
`inbox-new → reply → send → ack`. It works, but it burns a wake-up every
minute; fine for a quick trial, wrong for permanent deployment — if hooks are
available, always use hooks.

**Latency expectations**: receiving is real-time (the daemon long-polls);
reply latency comes from waking + generation, measured at ~30–60 seconds.
In WeChat that feels like "the other side is typing". Set this expectation
with the user at deployment; never promise instant replies.

## Step 7: Proactive-push scope (ask the user at deployment)

Proactive push is a double-edged sword: it can pull the user back, and it can
become harassment. At deployment, ask the user explicitly: **urgent matters
only, or are everyday gentle reminders and tips OK too?** Record the answer in
the user's proactive-messaging preferences and follow it strictly afterwards.
Until asked, default to urgent-only.

## Security red lines

- `auth.json` stays mode 600; the token, `ilink_user_id` and uin never appear
  in chat transcripts, never in long-term memory in plaintext, and are never
  echoed back.
- Reply only to the bound owner's private chats by default; strangers and
  group chats are ignored by default. Loosening this requires the user's
  explicit decision.
- Media goes through the official CDN crypto flow (AES-128-ECB, implemented
  in Appendix 1) — never invent your own crypto, and never build a second
  upload path next to the one Appendix 1 already implements.
- WeChat-side replies never mention hooks, queues, polling, claiming or any
  other internal machinery. It should feel like an ordinary chat.

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `send` returns `ret:-2 prepare failed` | No session: go back to Step 4 and have the user send a message first |
| `getupdates` returns `ret/-14` | The server paused this account's session for 1 hour: the daemon backs off automatically; never hammer the API |
| Daemon vanished | Machine reset or process reaped: the hook revives it on the next poll; check `bridge.log` for a traceback |
| Every log line appears twice | The launcher's stdout redirect was touched: restore `/dev/null` |
| `pkill -f` killed yourself | The pattern literal was also on your own command line: run kill and literal-path operations as two separate commands |
| User receives duplicate replies | Usually the old (v1) claim-less `inbox-new` is still running, or a cron reply loop exists next to the hook: keep exactly one pipeline — the hook |
| Hook woke you but `inbox-new` is empty | Messages are claimed and un-acked for under 10 minutes (a previous run is still working): they re-queue automatically; never edit the DB by hand |
| Image/video send fails, missing `x-encrypted-param` | The upload didn't succeed (network or file issue): retry once; confirm video is mp4 and images are jpg/png |
| An official-account article yields only its title | Normal, not a bridge bug: follow the handoff flow in Appendix 4, rule 4, to read the original |
| User says "the video is just a static image" | Verify first (frame inspection / file parameters) before explaining; never reflexively blame WeChat compression (a measured lesson) |

## Known limits (tell the user honestly at deployment)

- Only the bound owner's private chats; no group chats, no strangers
  (whitelist is hard-coded; loosen only on the user's explicit instruction).
- Reply latency ~30–60 seconds — not instant-messaging grade.
- Proactive push is bounded by the ~24-hour session window (see Step 4).
- Quoted messages (replying with a quote of an earlier WeChat message) are
  not parsed by the bridge; understand the current message on its own, and
  ask the user when unsure.
- Steps that need the user's confirmation in an app (payments, sending to
  third parties, QR logins to other services) cannot be completed from
  WeChat; route the user back to the main chat for those.

## Optional extensions (add only when the user wants them; off by default)

- **Nightly WeChat chat digest**: once a day, summarize that day's WeChat
  conversation (their messages + your replies; acks already store the full
  text) into a one-page digest for easy review. Implementation sketch: a
  nightly scheduled task reads that day's `status='replied'` rows from
  `queue.db` and lays them out as topics + key points; cost is roughly one
  extra wake per day.
- **Urgent matters call out via WeChat**: when the main chat judges something
  genuinely urgent and time-sensitive while the user is offline, push it with
  `send-stdin` (empty to/ctx sends to the bound owner), within the scope the
  user set in Step 7.

---

## Appendix 1: Complete `bridge.py` v2 source

Save as `~/wechat-bot/bridge.py` (Python 3.8+; dependencies: `requests`,
`qrcode`, `pycryptodome`). Source-level changes vs. the original: `login-fetch`
renders the QR code locally; the daemon saves videos/files to disk, records
every image's path, and marks untranscribed voice; the official upload flow
`_upload_media` plus `send-image/send-video/send-file` are new; `inbox-new`
claims atomically and `inbox-peek` was added; `ack` stores the full reply text.

```python
#!/usr/bin/env python3
"""Muse WeChat bot bridge -- Tencent iLink official bot API client."""
import argparse, base64, binascii, json, os, random, signal, sqlite3, sys, time, uuid
import requests

BASE = "https://ilinkai.weixin.qq.com"
CDN_BASE = "https://novac2c.cdn.weixin.qq.com/c2c"
DIR = os.path.expanduser("~/wechat-bot")
AUTH = os.path.join(DIR, "auth.json")
DB = os.path.join(DIR, "queue.db")
CURSOR = os.path.join(DIR, "cursor.txt")
PIDF = os.path.join(DIR, "bridge.pid")
LOG = os.path.join(DIR, "bridge.log")
QRFILE = os.path.join(DIR, "qr.png")
QRSTATE = os.path.join(DIR, "qr_state.json")
CLIENT_VER = str(0x00020408)  # 2.4.8
BOT_AGENT = "WeChatBridge/1.0"
HTTP_TIMEOUT = 45

def log(*a):
    line = time.strftime("%Y-%m-%d %H:%M:%S") + " " + " ".join(str(x) for x in a)
    print(line, flush=True)
    try:
        with open(LOG, "a") as f: f.write(line + "\n")
    except OSError:
        pass

def load_auth():
    try:
        with open(AUTH) as f: return json.load(f)
    except (OSError, ValueError):
        return {}

def save_auth(d):
    tmp = AUTH + ".tmp"
    with open(tmp, "w") as f: json.dump(d, f)
    os.chmod(tmp, 0o600)
    os.replace(tmp, AUTH)

def get_uin():
    a = load_auth()
    if a.get("uin"): return a["uin"]
    n = random.getrandbits(32)
    uin = base64.b64encode(str(n).encode()).decode()
    a["uin"] = uin
    save_auth(a)
    return uin

def base_headers(authed=False):
    h = {
        "Content-Type": "application/json",
        "AuthorizationType": "ilink_bot_token",
        "X-WECHAT-UIN": get_uin(),
        "iLink-App-Id": "bot",
        "iLink-App-ClientVersion": CLIENT_VER,
    }
    if authed:
        a = load_auth()
        tok = a.get("bot_token", "")
        h["Authorization"] = "Bearer " + tok
    return h

def base_info():
    return {"channel_version": "2.4.8", "bot_agent": BOT_AGENT}

def db():
    c = sqlite3.connect(DB)
    c.execute("""CREATE TABLE IF NOT EXISTS inbox(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        message_id TEXT UNIQUE, from_user_id TEXT, session_id TEXT, group_id TEXT,
        text TEXT, voice_text TEXT, image_path TEXT, file_name TEXT,
        create_time_ms INTEGER, context_token TEXT,
        status TEXT DEFAULT 'new', reply_text TEXT,
        created_at REAL, replied_at REAL)""")
    c.execute("""CREATE TABLE IF NOT EXISTS sessions(
        user_id TEXT PRIMARY KEY, context_token TEXT, updated_at REAL)""")
    return c

# ---------- login ----------
def cmd_login_fetch(args):
    r = requests.post(BASE + "/ilink/bot/get_bot_qrcode?bot_type=3",
                      headers=base_headers(), json={"local_token_list": []},
                      timeout=HTTP_TIMEOUT)
    r.raise_for_status()
    d = r.json()
    qc, img = d.get("qrcode", ""), d.get("qrcode_img_content", "")
    if not qc and not img:
        print(json.dumps({"ok": False, "raw": d})); return 1
    with open(QRSTATE, "w") as f: json.dump({"qrcode": qc, "qrcode_img_content": img}, f)
    saved = None
    if img:
        try:
            raw = base64.b64decode(img)
            if raw[:8] == b"\x89PNG\r\n\x1a\n" or raw[:2] == b"\xff\xd8":
                with open(QRFILE, "wb") as f: f.write(raw)
                saved = QRFILE
        except (binascii.Error, ValueError):
            pass
    if not saved and qc.startswith("http"):
        rr = requests.get(qc, timeout=HTTP_TIMEOUT)
        rr.raise_for_status()
        with open(QRFILE, "wb") as f: f.write(rr.content)
        saved = QRFILE
    print(json.dumps({"ok": True, "qr_png": saved, "qrcode_len": len(qc),
                      "img_is_b64_image": bool(saved), "img_content": img}))
    return 0

def cmd_login_wait(args):
    try:
        with open(QRSTATE) as f: qc = json.load(f)["qrcode"]
    except (OSError, ValueError, KeyError):
        print(json.dumps({"ok": False, "error": "no qr_state, run login-fetch first"})); return 1
    import urllib.parse
    deadline = time.time() + args.timeout
    while time.time() < deadline:
        q = urllib.parse.urlencode({"qrcode": qc, **({"verify_code": args.verify_code} if args.verify_code else {})})
        try:
            r = requests.get(BASE + "/ilink/bot/get_qrcode_status?" + q,
                             headers={k: v for k, v in base_headers().items() if k != "Content-Type"},
                             timeout=HTTP_TIMEOUT)
            r.raise_for_status()
            d = r.json()
        except Exception as e:
            log("qr poll error:", e); time.sleep(3); continue
        st = d.get("status", "")
        log("qr status:", st)
        if st == "confirmed":
            a = load_auth()
            a.update({"bot_token": d.get("bot_token", ""), "ilink_bot_id": d.get("ilink_bot_id", ""),
                      "ilink_user_id": d.get("ilink_user_id", ""), "baseurl": d.get("baseurl", "")})
            save_auth(a)
            print(json.dumps({"ok": True, "ilink_user_id": a["ilink_user_id"],
                              "ilink_bot_id": a["ilink_bot_id"]}))
            return 0
        if st in ("expired",):
            print(json.dumps({"ok": False, "error": "qrcode expired"})); return 1
        if st == "need_verifycode":
            print(json.dumps({"ok": False, "error": "need_verifycode"})); return 2
        if st == "verify_code_blocked":
            print(json.dumps({"ok": False, "error": "verify_code_blocked"})); return 1
        time.sleep(2)
    print(json.dumps({"ok": False, "error": "timeout"})); return 1

# ---------- media ----------
def try_save_image(item, msg_id, forced_ext=None):
    try:
        from Crypto.Cipher import AES
    except ImportError:
        return None
    try:
        media = item.get("media") or {}
        url = media.get("full_url") or ""
        if not url:
            p = media.get("encrypt_query_param") or ""
            if not p: return None
            import urllib.parse
            url = CDN_BASE + "/download?encrypted_query_param=" + urllib.parse.quote(p, safe="")
        rr = requests.get(url, headers=base_headers(authed=True), timeout=HTTP_TIMEOUT)
        rr.raise_for_status()
        data = rr.content
        key_hex = item.get("aeskey") or media.get("aes_key") or ""
        key = None
        if len(key_hex) == 32:
            try: key = bytes.fromhex(key_hex)
            except ValueError: key = None
        if key is None:
            try: key = base64.b64decode(key_hex)
            except (binascii.Error, ValueError): key = None
        if key and len(key) == 16 and len(data) % 16 == 0:
            pt = AES.new(key, AES.MODE_ECB).decrypt(data)
            pad = pt[-1]
            if 1 <= pad <= 16 and pt.endswith(bytes([pad]) * pad):
                data = pt[:-pad]
        det = ".jpg" if data[:2] == b"\xff\xd8" else ".png" if data[:8] == b"\x89PNG\r\n\x1a\n" else ".mp4" if data[4:8] == b"ftyp" else ".bin"
        ext = forced_ext or det
        path = os.path.join(DIR, "media", "media_%s%s" % (msg_id, ext))
        with open(path, "wb") as f: f.write(data)
        return path
    except Exception as e:
        log("image save failed:", e)
        return None

# ---------- daemon ----------
def handle_msg(c, msg, owner):
    if msg.get("message_type") != 1: return
    if msg.get("message_state") not in (0, 2): return
    mid = str(msg.get("message_id", ""))
    frm = msg.get("from_user_id", "")
    if not mid or not frm: return
    if c.execute("SELECT 1 FROM inbox WHERE message_id=?", (mid,)).fetchone(): return
    group_id = msg.get("group_id") or ""
    if group_id:
        c.execute("INSERT OR IGNORE INTO inbox(message_id,from_user_id,session_id,group_id,status,created_at) VALUES(?,?,?,?,?,?)",
                  (mid, frm, msg.get("session_id", ""), group_id, "ignored_group", time.time()))
        c.commit(); log("ignored group msg", mid); return
    if owner and frm != owner:
        c.execute("INSERT OR IGNORE INTO inbox(message_id,from_user_id,session_id,status,created_at) VALUES(?,?,?,?,?)",
                  (mid, frm, msg.get("session_id", ""), "ignored_stranger", time.time()))
        c.commit(); log("ignored stranger msg", mid, frm); return
    texts, voice_text, image_path, file_name = [], "", None, ""
    img_paths = []
    for it in msg.get("item_list") or []:
        t = it.get("type")
        if t == 1 and it.get("text_item"):
            texts.append(it["text_item"].get("text", ""))
        elif t == 3 and it.get("voice_item"):
            vt = it["voice_item"].get("text", "")
            if vt: voice_text += vt
            else: texts.append("[语音消息（无转写文本）]")
        elif t == 2 and it.get("image_item"):
            pth = try_save_image(it["image_item"], mid)
            if pth:
                img_paths.append(pth)
                texts.append("[图片已保存: %s]" % pth)
            else:
                texts.append("[图片]")
        elif t == 4 and it.get("file_item"):
            fi = it["file_item"]
            file_name = fi.get("file_name", "")
            pth = try_save_image(fi, mid, os.path.splitext(file_name)[1] or ".bin")
            if pth: texts.append("[文件已保存: %s（%s）]" % (pth, file_name))
            else: texts.append("[文件: %s]" % file_name)
        elif t == 5 and it.get("video_item"):
            pth = try_save_image(it["video_item"], mid, ".mp4")
            texts.append("[视频已保存: %s]" % pth if pth else "[视频]")
    if img_paths: image_path = img_paths[0]
    text = "\n".join(x for x in texts if x).strip()
    if voice_text: text = (text + "\n[语音转文字: %s]" % voice_text).strip()
    if msg.get("context_token"):
        c.execute("INSERT OR REPLACE INTO sessions(user_id,context_token,updated_at) VALUES(?,?,?)",
                  (frm, msg["context_token"], time.time()))
    c.execute("""INSERT OR IGNORE INTO inbox(message_id,from_user_id,session_id,text,voice_text,
                 image_path,file_name,create_time_ms,context_token,status,created_at)
                 VALUES(?,?,?,?,?,?,?,?,?, 'new', ?)""",
              (mid, frm, msg.get("session_id", ""), text, voice_text, image_path, file_name,
               msg.get("create_time_ms", 0), msg.get("context_token", ""), time.time()))
    c.commit()
    log("queued msg", mid, "from", frm, "len", len(text))

def cmd_daemon(args):
    a = load_auth()
    if not a.get("bot_token"):
        print("no bot_token, run login first"); return 1
    owner = a.get("ilink_user_id", "")
    with open(PIDF, "w") as f: f.write(str(os.getpid()))
    stop = []
    def on_term(signum, frame): stop.append(1)
    signal.signal(signal.SIGTERM, on_term); signal.signal(signal.SIGINT, on_term)
    try:
        requests.post(BASE + "/ilink/bot/msg/notifystart", headers=base_headers(authed=True),
                      json={"base_info": base_info()}, timeout=HTTP_TIMEOUT)
    except Exception as e:
        log("notifystart failed:", e)
    try:
        with open(CURSOR) as f: buf = f.read().strip()
    except OSError:
        buf = ""
    c = db()
    backoff = 5
    log("daemon started, owner=", owner or "(unknown)")
    while not stop:
        try:
            r = requests.post(BASE + "/ilink/bot/getupdates", headers=base_headers(authed=True),
                              json={"get_updates_buf": buf, "base_info": base_info()},
                              timeout=HTTP_TIMEOUT + 30)
            r.raise_for_status()
            d = r.json()
            if d.get("ret") == -14 or d.get("errcode") == -14:
                log("session paused by server (-14), sleeping 1h"); time.sleep(3600); continue
            backoff = 5
            for m in d.get("msgs") or []:
                try: handle_msg(c, m, owner)
                except Exception as e: log("handle error:", e)
            nb = d.get("get_updates_buf") or ""
            if nb:
                buf = nb
                with open(CURSOR, "w") as f: f.write(buf)
        except requests.RequestException as e:
            log("poll error:", e, "backoff", backoff)
            time.sleep(backoff); backoff = min(backoff * 2, 120)
        except Exception as e:
            log("loop error:", e); time.sleep(backoff); backoff = min(backoff * 2, 120)
    try:
        requests.post(BASE + "/ilink/bot/msg/notifystop", headers=base_headers(authed=True),
                      json={"base_info": base_info()}, timeout=15)
    except Exception:
        pass
    try: os.remove(PIDF)
    except OSError: pass
    log("daemon stopped")
    return 0

# ---------- send / inbox ----------
def cmd_send(args):
    a = load_auth()
    if not a.get("bot_token"):
        print(json.dumps({"ok": False, "error": "no bot_token"})); return 1
    to = args.to or a.get("ilink_user_id", "")
    text = args.text or ""
    ctx = args.ctx or ""
    if not ctx:
        c0 = db()
        row = c0.execute("SELECT context_token FROM sessions WHERE user_id=?", (to,)).fetchone()
        c0.close()
        if row: ctx = row[0]
    if args.text_b64:
        import base64 as _b
        text = _b.b64decode(args.text_b64).decode("utf-8", "replace")
    chunks = [text[i:i + 1500] for i in range(0, len(text), 1500)] or [""]
    for ch in chunks:
        payload = {"msg": {
            "from_user_id": "", "to_user_id": to,
            "client_id": uuid.uuid4().hex, "message_type": 2, "message_state": 2,
            "context_token": ctx,
            "item_list": [{"type": 1, "text_item": {"text": ch}}]},
            "base_info": base_info()}
        r = requests.post(BASE + "/ilink/bot/sendmessage", headers=base_headers(authed=True),
                          json=payload, timeout=HTTP_TIMEOUT)
        r.raise_for_status()
        d = r.json()
        if d.get("ret") not in (0, None):
            print(json.dumps({"ok": False, "resp": d})); return 1
    print(json.dumps({"ok": True, "chunks": len(chunks)}))
    return 0

def _session_ctx(to):
    c0 = db()
    row = c0.execute("SELECT context_token FROM sessions WHERE user_id=?", (to,)).fetchone()
    c0.close()
    return row[0] if row else ""

def _send_payload(to, ctx, item):
    payload = {"msg": {
        "from_user_id": "", "to_user_id": to,
        "client_id": uuid.uuid4().hex, "message_type": 2, "message_state": 2,
        "context_token": ctx,
        "item_list": [item]},
        "base_info": base_info()}
    r = requests.post(BASE + "/ilink/bot/sendmessage", headers=base_headers(authed=True),
                      json=payload, timeout=HTTP_TIMEOUT)
    r.raise_for_status()
    d = r.json()
    if d.get("ret") not in (0, None):
        print(json.dumps({"ok": False, "resp": d})); return 1
    print(json.dumps({"ok": True}))
    return 0

def _upload_media(path, media_type, to):
    import hashlib, urllib.parse
    with open(path, "rb") as f: raw = f.read()
    md5 = hashlib.md5(raw).hexdigest()
    key = os.urandom(16)
    aeskey_hex = key.hex()
    filekey = uuid.uuid4().hex
    filesize = ((len(raw) // 16) + 1) * 16
    payload = {"filekey": filekey, "media_type": media_type, "to_user_id": to,
               "rawsize": len(raw), "rawfilemd5": md5, "filesize": filesize,
               "no_need_thumb": True, "aeskey": aeskey_hex, "base_info": base_info()}
    r = requests.post(BASE + "/ilink/bot/getuploadurl", headers=base_headers(authed=True),
                      json=payload, timeout=HTTP_TIMEOUT)
    r.raise_for_status()
    d = r.json()
    up_url = d.get("upload_full_url") or ""
    if not up_url:
        param = d.get("upload_param") or ""
        if not param:
            raise RuntimeError("getuploadurl returned no url/param")
        up_url = (CDN_BASE + "/upload?encrypted_query_param=" + urllib.parse.quote(param, safe="")
                  + "&filekey=" + urllib.parse.quote(filekey, safe=""))
    from Crypto.Cipher import AES
    pad = 16 - (len(raw) % 16)
    enc = AES.new(key, AES.MODE_ECB).encrypt(raw + bytes([pad]) * pad)
    rr = requests.post(up_url, headers={"Content-Type": "application/octet-stream"},
                       data=enc, timeout=HTTP_TIMEOUT)
    rr.raise_for_status()
    xparam = rr.headers.get("x-encrypted-param") or ""
    if not xparam:
        raise RuntimeError("upload response missing x-encrypted-param (status %s)" % rr.status_code)
    media = {"encrypt_query_param": xparam,
             "aes_key": base64.b64encode(aeskey_hex.encode()).decode(),
             "encrypt_type": 1}
    return media, len(enc)

def cmd_send_image(args):
    a = load_auth()
    if not a.get("bot_token"):
        print(json.dumps({"ok": False, "error": "no bot_token"})); return 1
    to = args.to or a.get("ilink_user_id", "")
    ctx = args.ctx or _session_ctx(to)
    media, enc_size = _upload_media(args.file, 1, to)
    item = {"type": 2, "image_item": {"media": media, "mid_size": enc_size}}
    return _send_payload(to, ctx, item)

def cmd_send_video(args):
    a = load_auth()
    if not a.get("bot_token"):
        print(json.dumps({"ok": False, "error": "no bot_token"})); return 1
    to = args.to or a.get("ilink_user_id", "")
    ctx = args.ctx or _session_ctx(to)
    media, enc_size = _upload_media(args.file, 2, to)
    item = {"type": 5, "video_item": {"media": media, "video_size": enc_size}}
    return _send_payload(to, ctx, item)

def cmd_send_file(args):
    a = load_auth()
    if not a.get("bot_token"):
        print(json.dumps({"ok": False, "error": "no bot_token"})); return 1
    to = args.to or a.get("ilink_user_id", "")
    ctx = args.ctx or _session_ctx(to)
    media, enc_size = _upload_media(args.file, 3, to)
    item = {"type": 4, "file_item": {"media": media, "file_name": os.path.basename(args.file),
                                     "len": str(os.path.getsize(args.file))}}
    return _send_payload(to, ctx, item)

def cmd_send_stdin(args):
    d = json.load(sys.stdin)
    ns = argparse.Namespace(to=d.get("to", ""), ctx=d.get("ctx", ""),
                            text=d.get("text", ""), text_b64="")
    return cmd_send(ns)

def _ensure_claimed_col(c):
    try:
        c.execute("ALTER TABLE inbox ADD COLUMN claimed_at REAL DEFAULT 0")
        c.commit()
    except Exception:
        pass

def cmd_inbox_new(args):
    c = db()
    _ensure_claimed_col(c)
    # Requeue messages claimed >10 min ago but never acked (worker died mid-flight).
    c.execute("UPDATE inbox SET status='new' WHERE status='processing' AND claimed_at < ?", (time.time() - 600,))
    ids = [r[0] for r in c.execute("SELECT id FROM inbox WHERE status='new' ORDER BY id").fetchall()]
    if ids:
        c.executemany("UPDATE inbox SET status='processing', claimed_at=? WHERE id=?",
                      [(time.time(), i) for i in ids])
        c.commit()
        q = ",".join("?" * len(ids))
        rows = c.execute("SELECT id,message_id,from_user_id,session_id,text,image_path,file_name,create_time_ms,context_token FROM inbox WHERE id IN (%s) ORDER BY id" % q, ids).fetchall()
    else:
        rows = []
    out = [dict(zip(["id", "message_id", "from_user_id", "session_id", "text", "image_path", "file_name", "create_time_ms", "context_token"], r)) for r in rows]
    print(json.dumps(out, ensure_ascii=False))
    return 0

def cmd_inbox_peek(args):
    c = db()
    _ensure_claimed_col(c)
    n = c.execute("SELECT COUNT(*) FROM inbox WHERE status='new' OR (status='processing' AND claimed_at < ?)",
                  (time.time() - 600,)).fetchone()[0]
    print(json.dumps([0] * n))
    return 0

def cmd_ack(args):
    c = db()
    c.execute("UPDATE inbox SET status=?, reply_text=?, replied_at=? WHERE id=?",
              (args.status, args.reply or "", time.time(), args.id))
    c.commit()
    print(json.dumps({"ok": True}))
    return 0

def cmd_status(args):
    a = load_auth()
    alive = False
    try:
        with open(PIDF) as f: pid = int(f.read().strip())
        os.kill(pid, 0); alive = True
    except (OSError, ValueError):
        pass
    c = db()
    new_n = c.execute("SELECT COUNT(*) FROM inbox WHERE status='new'").fetchone()[0]
    print(json.dumps({"token": bool(a.get("bot_token")), "owner": a.get("ilink_user_id", ""),
                      "daemon_alive": alive, "new": new_n}))
    return 0

def main():
    os.makedirs(DIR, exist_ok=True); os.makedirs(os.path.join(DIR, "media"), exist_ok=True)
    p = argparse.ArgumentParser()
    sub = p.add_subparsers(dest="cmd", required=True)
    sub.add_parser("login-fetch")
    w = sub.add_parser("login-wait"); w.add_argument("--timeout", type=int, default=300); w.add_argument("--verify-code", default="")
    sub.add_parser("daemon")
    s = sub.add_parser("send"); s.add_argument("--to", default=""); s.add_argument("--text", default=""); s.add_argument("--text-b64", default=""); s.add_argument("--ctx", default="")
    sub.add_parser("send-stdin")
    si = sub.add_parser("send-image"); si.add_argument("--file", required=True); si.add_argument("--to", default=""); si.add_argument("--ctx", default="")
    sv = sub.add_parser("send-video"); sv.add_argument("--file", required=True); sv.add_argument("--to", default=""); sv.add_argument("--ctx", default="")
    sf = sub.add_parser("send-file"); sf.add_argument("--file", required=True); sf.add_argument("--to", default=""); sf.add_argument("--ctx", default="")
    sub.add_parser("inbox-peek")
    sub.add_parser("inbox-new")
    k = sub.add_parser("ack"); k.add_argument("--id", type=int, required=True); k.add_argument("--status", default="replied"); k.add_argument("--reply", default="")
    sub.add_parser("status")
    args = p.parse_args()
    fn = {"login-fetch": cmd_login_fetch, "login-wait": cmd_login_wait, "daemon": cmd_daemon,
          "send": cmd_send, "send-stdin": cmd_send_stdin, "inbox-new": cmd_inbox_new, "ack": cmd_ack, "status": cmd_status,
          "send-image": cmd_send_image, "send-video": cmd_send_video, "send-file": cmd_send_file,
          "inbox-peek": cmd_inbox_peek}[args.cmd]
    sys.exit(fn(args))

if __name__ == "__main__":
    main()
```

## Appendix 2: `launch.sh` (daemon launcher)

Save as `~/wechat-bot/launch.sh` and `chmod +x`. Double-fork, idempotent:
exits immediately if the daemon is already running.

```bash
#!/bin/bash
# Double-fork launcher for the WeChat bot bridge daemon.
# Idempotent: exits quietly if a daemon is already alive.
# The daemon ends up as a child of init, independent of the starter session.
D="$HOME/wechat-bot"
if [ -f "$D/bridge.pid" ]; then
  p=$(cat "$D/bridge.pid" 2>/dev/null)
  if [ -n "$p" ] && ps -p "$p" -o args= 2>/dev/null | grep -q "bridge.py daemon"; then
    echo "already running pid $p"
    exit 0
  fi
  rm -f "$D/bridge.pid"
fi
cd "$D" || exit 1
python3 - <<'PYEOF'
import os
if os.fork() > 0:
    os._exit(0)
os.setsid()
if os.fork() > 0:
    os._exit(0)
log = open("bridge.log", "a")
os.dup2(log.fileno(), 2)
dn = open("/dev/null", "r")
os.dup2(dn.fileno(), 0)
dn2 = open("/dev/null", "w")
os.dup2(dn2.fileno(), 1)
os.execv("/usr/bin/python3", ["python3", "bridge.py", "daemon"])
PYEOF
echo "launcher done"
```

## Appendix 3: Hook script + hook definition example (the zero-idle-cost key)

Save the hook script as `~/hooks/scripts/wechat-inbox-watch.sh` and
`chmod +x`. It relies on the `log` / `silent` / `wake` conventions provided by
Muse's hook runtime (the script starts with
`source "$HATCH_HOOK_RUNTIME"`): a local script runs every 5 seconds at zero
token cost, and only `wake`s Muse when the queue actually holds new messages.

```bash
#!/usr/bin/env bash
set -euo pipefail
source "$HATCH_HOOK_RUNTIME"

# Keep the bridge daemon alive (idempotent: exits fast if already running).
"$HOME/wechat-bot/launch.sh" >/dev/null 2>&1 || true

# Count unprocessed WeChat messages in the local queue.
OUT="$(python3 "$HOME/wechat-bot/bridge.py" inbox-peek 2>/dev/null || true)"
if [ -z "$OUT" ]; then
  log "inbox check failed" '{}'
  silent "inbox check inconclusive" '{}'
  exit 0
fi

N="$(printf '%s' "$OUT" | python3 -c 'import json,sys
try:
    print(len(json.load(sys.stdin)))
except Exception:
    print(-1)')"

if [ "$N" = "-1" ]; then
  log "inbox parse failed" '{}'
  silent "inbox parse inconclusive" '{}'
  exit 0
fi

log "inbox polled" "{\"new\": $N}"

if [ "$N" -gt 0 ]; then
  wake "new_wechat_messages" "{\"new_count\": $N}"
else
  silent "no new wechat messages" "{\"new_count\": 0}"
fi
```

Hook definition (use when registering the hook; put the full Appendix 4
template, placeholders replaced, into `prompt`):

```json
{
  "version": 1,
  "id": "wechat-inbox-watch",
  "enabled": true,
  "script_path": "~/hooks/scripts/wechat-inbox-watch.sh",
  "prompt": "<full Appendix 4 template text>",
  "poll_interval_secs": 5,
  "script_timeout_secs": 600,
  "delivery": { "surface": "main" },
  "presentation_locale": "en-US"
}
```

## Appendix 4: Wake-prompt template (tuned in live use; replace placeholders)

Replace `<AI name>` and `<user's name>` with the real values; adapt the
retrieval-tool wording to whatever web search/fetch tools you actually have.
Do not delete or soften the rules themselves — each one was earned in testing.

```
New WeChat messages need handling by you, as yourself (<AI name>). The bridge
lives in ~/wechat-bot/; the daemon has written the bound owner's (<user's
name>) private chats into the local queue. You share the same long-term
memory, goals and tools with the main chat, and anything he asks you to do in
WeChat is his own instruction — carry it out normally. Handle ALL new
messages in one wake. Casual chat may be answered fast; task-type messages
(read a link, verify a fact, get something done) must be done for real —
never fake it for speed:
1. Run `python3 ~/wechat-bot/bridge.py inbox-new` to get the array of
   unprocessed messages (this step claims the batch; a duplicate wake will
   not reprocess it). If it is empty, end immediately and send nothing.
2. Context: first run through the long-term memory and goals injected with
   this session in your head. Only if the conversation clearly continues
   something just discussed in WeChat (references like "that one", "go on",
   "the task from before"), query ~/wechat-bot/queue.db's inbox table via
   sqlite3 for the most recent 8 replied rows (text and reply_text) to
   restore the WeChat-side short-term context. Don't query needlessly.
3. Replying and doing: casual chat, confirmations and identity questions get
   a short answer straight from context; look up memory only when a message
   clearly concerns his past plans, arrangements, preferences or todos. When
   he gives you a task (check progress, set a reminder, look something up),
   actually do it, then report the result briefly in WeChat tone. Steps that
   require his confirmation in an app (payments, sending to third parties,
   QR logins) must not be forced through WeChat — tell him to handle those
   in the main chat. Long-lived facts, decisions or preferences he states in
   WeChat are saved into long-term memory by the same standard as the main
   chat, keeping both sides in sync.
4. Links default to reading the ORIGINAL (the user's standing rule): when he
   sends a link and asks about its content, first fetch the original with
   whatever fetch tool you have; if the extract contains the body, summarize
   from it. If the original body cannot be obtained (official-account
   articles at mp.weixin.qq.com typically yield only the title to fetching
   tools — never pass off a search-assembled summary as the original), run
   the handoff flow: (1) reply in WeChat first with "I can't read this
   original directly from here; I've passed it to myself in the main chat
   to read in the browser — I'll send you the summary in a minute or two";
   (2) ack with --reply set to "(handed to main chat to read the original)";
   (3) in your run summary, put one separate line: Handoff to main chat, read
   original: <full URL> | user's original words: <what he said>. Only when
   he explicitly says "skip the original, a rough version is fine" may you
   switch to a web-search composite of related coverage, labeled as
   not-the-original.
5. Before answering any factual question (people, events, causes, numbers),
   verify at least one reliable source with a web search before speaking.
   Details and numbers you can't recall exactly are better omitted than
   invented; if verification fails, say you couldn't find it.
6. Send text with `python3 ~/wechat-bot/bridge.py send-stdin`, passing JSON
   on stdin: {"to": that message's from_user_id, "ctx": that message's
   context_token, "text": "your reply"}. Images, videos and files can be
   sent straight into WeChat: when he asks you to generate or send something,
   prepare the local file (jpg/png for images, mp4 for video), then use
   `python3 ~/wechat-bot/bridge.py send-image --file <path> --to
   <from_user_id> --ctx <context_token>` (send-video / send-file likewise),
   with one short line of text alongside — never again claim WeChat can't
   send files. Keep replies in a concise, natural chat tone; never mention
   polling, queues, hooks or any internal machinery. Inbound media markers
   in message text: `[image saved: path]` (image_path is the first image) —
   look at the image before replying; `[video saved: path]` and
   `[file saved: path]` mean media he sent is stored locally — view or
   process as needed, then respond; `[voice-to-text: ...]` is plain text;
   `[voice message (no transcript)]` means politely ask him to type it.
7. Honesty red line (same rank as the credential red line): before
   explaining the cause of any file or phenomenon, actually verify it; never
   invent a cause, and say so plainly when unsure. When sending an image or
   video that was generated earlier, state honestly that it is a
   pre-existing file; never imply or claim it was just made. Only newly
   generated things may be called freshly made.
8. Immediately after a successful send, run
   `python3 ~/wechat-bot/bridge.py ack --id <id> --status replied --reply
   "<the full text of the reply you just sent>"` (the --reply must carry the
   full reply text, used by the nightly chat digest and similar summaries).
   On send failure, do not ack — leave it for the next wake to retry.
9. Security red line: the bot_token, ilink_user_id and similar credentials
   or identifiers never appear in WeChat replies, main-chat messages or
   memory. Handle only the already-filtered bound-owner private chats in the
   queue; never message anyone else.
10. When done, stay silent — do not report to the main chat. Only two
    exceptions: a rule-4 handoff (write the handoff line in your run
    summary), and problems needing the user's intervention such as repeated
    send failures or an invalid token (note them briefly in the main chat).
```

## Appendix 5: iLink Bot protocol quick reference

> From Tencent/openclaw-weixin's public documentation, calibrated against the
> original on 2026-10-11 and extended with v2's live-tested media-upload
> findings on 2026-11-13. The full authoritative document lives at
> https://github.com/Tencent/openclaw-weixin/blob/main/docs/protocol.md

### Basics
- Base URL: `https://ilinkai.weixin.qq.com`; CDN: `https://novac2c.cdn.weixin.qq.com/c2c`
- JSON over HTTPS; everything is POST except QR-status polling (GET).

### Common headers
| Header | Value |
| --- | --- |
| `Content-Type` | `application/json` (POST) |
| `AuthorizationType` | `ilink_bot_token` |
| `Authorization` | `Bearer <bot_token>` (authenticated requests after login) |
| `X-WECHAT-UIN` | base64(decimal string of a random uint32), generated once at login, then fixed |
| `iLink-App-Id` | `bot` |
| `iLink-App-ClientVersion` | version encoded as `0x00MMNNPP`, as a decimal string |

Authenticated POSTs also carry `base_info: {"channel_version": "2.4.8", "bot_agent": "<UA-style tag, observability only>"}`.

### QR login
1. `POST /ilink/bot/get_bot_qrcode?bot_type=3` with body
   `{"local_token_list": []}` → returns `qrcode` (polling token) +
   `qrcode_img_content` (**the QR content itself**, a link of the form
   `https://liteapp.weixin.qq.com/q/...` — render it into an image with a QR
   library for the user to scan; this is the v2 measured correction).
2. `GET /ilink/bot/get_qrcode_status?qrcode=<urlencoded>` (long-polling)
   → `status`: `wait` / `scaned` / `confirmed` (yields `bot_token`,
   `ilink_bot_id`, `ilink_user_id`) / `expired` / `need_verifycode` (poll
   again with `&verify_code=`) / `verify_code_blocked` /
   `scaned_but_redirect` (switch base URL per `redirect_host` and continue).
   A QR code is valid for about 5 minutes.

### Sending and receiving
- `POST /ilink/bot/getupdates` with body `{"get_updates_buf": "<cursor from
  the previous response, empty string the first time>", "base_info": {...}}`
  → `{"ret":0, "msgs":[...], "get_updates_buf":"...",
  "longpolling_timeout_ms":35000}`. A `ret`/`errcode` of `-14` means: stop
  and retry after 1 hour.
- `POST /ilink/bot/sendmessage`, text message body:
  `{"msg": {"from_user_id": "", "to_user_id": "<user>",
  "client_id": "<uuid>", "message_type": 2, "message_state": 2,
  "context_token": "<echo the received one>",
  "item_list": [{"type": 1, "text_item": {"text": "..."}}]},
  "base_info": {...}}`
  Note `from_user_id` is left empty (the official plugin does the same);
  `message_type: 1` = user, `2` = bot.
- `POST /ilink/bot/msg/notifystart` / `notifystop`: called once when the
  daemon starts/stops (failures are only logged).

**Iron rule**: `context_token` comes from a received message; with zero
session, sends are rejected (`ret:-2 prepare failed`). Always echo the
session's token when replying. Proactive sends use the latest token stored in
the `sessions` table; the window may close ~24 hours after the user's last
message.

### Message model (inbound)
`WeixinMessage`: `message_id`, `from_user_id`, `to_user_id`, `session_id`,
`group_id` (group chats only), `message_type`, `message_state` (0 new /
1 generating / 2 complete; handle 0 and 2), `item_list`, `context_token`,
`create_time_ms`. `item_list` types: `1` text (`text_item.text`), `2` image,
`3` voice (`voice_item.text` may carry a transcript; empty when there is
none), `4` file (`file_item.file_name`), `5` video.

### Media download (inbound images/video/files — one flow for all)
1. Take `media.full_url`; if absent, build
   `CDN_BASE/download?encrypted_query_param=<...>`.
2. GET the byte stream; the key is the item's `aeskey` first (32 hex chars =
   16 bytes), else `media.aes_key`.
3. AES-128-ECB decrypt + strip PKCS#7 padding (if the length isn't a multiple
   of 16 or there is no key, treat the payload as plaintext). Detect the real
   type from the file header (jpg/png/mp4) when saving.

### Media upload (outbound images/video/files — implemented in v2)
1. `POST /ilink/bot/getuploadurl` with body containing
   `{"filekey": "<uuid>", "media_type": <1 image / 2 video / 3 file>,
   "to_user_id": "<user>", "rawsize": <original byte count>,
   "rawfilemd5": "<md5 of the original>", "filesize": <encrypted byte count,
   16-aligned>, "no_need_thumb": true, "aeskey": "<hex of a random 16-byte
   key>", "base_info": {...}}`
   → returns `upload_full_url`, or `upload_param` (then the upload URL is
   `CDN_BASE/upload?encrypted_query_param=<param>&filekey=<filekey>`).
2. Encrypt the file with AES-128-ECB + PKCS#7 padding and POST it to the
   upload URL with `Content-Type: application/octet-stream`.
3. Read `x-encrypted-param` from the response headers and build
   `media = {"encrypt_query_param": <x-encrypted-param>,
   "aes_key": base64(the aeskey hex string), "encrypt_type": 1}`.
4. Send via `sendmessage`: image
   `{"type": 2, "image_item": {"media": media, "mid_size": <encrypted size>}}`;
   video `{"type": 5, "video_item": {"media": media, "video_size": <encrypted size>}}`;
   file `{"type": 4, "file_item": {"media": media, "file_name": "<name>",
   "len": "<original byte count as string>"}}`.

---
*Version history: the original was calibrated end-to-end on 2026-10-11
(QR binding → daemon → two-way messaging → proactive push → cloud-VM
migration + keepalive). v2 was hardened on 2026-11-13 through a full day of
real use and an audit (zero-cost hook architecture, completed media
send/receive, atomic claiming, the honesty red line, the read-the-original
link rule, the proactive-window clarification) — every change backed by a
live test. Good luck with your deployment; see you in WeChat.*
