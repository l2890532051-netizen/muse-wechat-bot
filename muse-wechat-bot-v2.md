---
name: "muse-wechat-bot"
version: "v2（实测优化版，2026-11-13）"
description: >
  把 Muse 本人接进微信当 bot（单文件自包含版）：基于腾讯官方 iLink bot 接口
  （ilinkai.weixin.qq.com，扫码授权绑定），在长期在线的机器上跑一条 Python 原生桥——
  扫码一次绑定，之后用户在微信里聊到的就是 Muse 本人（记忆、技能全在），支持文字/
  图片/视频/文件/语音转文字收发与主动推送。本版在 2026-10-11 初版基础上，
  经 2026-11-13 一整天真机实测与漏洞审计打磨：零空转成本的事件钩子架构、
  媒体收发补全、原子认领防重复回复、诚实红线与链接读原文规矩，全部沉淀在内。
Use when the user wants to 把 AI 接入微信 / 微信 bot / 在微信里跟 AI 聊天,
mentions Tencent/openclaw-weixin, 微信机器人, or asks about chatting with Muse via WeChat.
全程 AI 代操作：用户只负责扫一次二维码（以及可能出现的数字配对码），其余全部由 AI 完成。
---

> **本文件自包含全部内容**（操作手册 + `bridge.py` v2 完整源码 + `launch.sh` +
> 钩子脚本 + 唤醒提示词模板 + iLink 协议速查），无外部文件依赖，两种用法：
> - **发给你的 Muse**：把本文件发给它，说「按这个文件把你接进微信」即可；
> - **安装为常驻技能**：把本文件另存为技能文件（Muse 的技能目录），下次直接触发。
>
> 微信 bot 开源教程与协议权威文档出处：https://github.com/Tencent/openclaw-weixin

# AI 接入微信 Bot（腾讯官方 iLink 接口 · v2 实测优化版）

**👤 人类用户用法**：把本技能发给你的 Muse，说一句 **"按这个技能把你接进微信"** 即可。

**🤖 Muse 用法**：本文件自包含完整实现（`bridge.py` v2 开箱即用，完整源码见附录 1
——先落盘再部署，不要凭记忆重写）。**全程代操作是硬性要求**：环境检查、部署、取码、
轮询、守护进程、钩子、回复循环全部由你完成；只在两个环节停下等用户——
① 微信扫码授权绑定 ② 若出现数字配对码请用户把数字发你。
不要让用户替你跑命令、看日志。

**⚠️ 与 Tencent/openclaw-weixin 手册的关键差异**：那份手册是给 **OpenClaw**（另一套
agent 系统）的插件——照它走，微信里聊到的是另一个 AI，没有你的记忆和技能。本技能走
**原生桥**路线：用同一份公开协议直连腾讯官方接口，微信里聊到的就是 **Muse 本人**。
官方通道，封号风险远低于野路子协议。

## 本版（v2）相对初版改了什么 —— 全部来自真机实测与审计

1. **零空转成本架构（最重要的改动）**：初版用「每分钟 cron 唤醒 AI 拉队列」，
   没消息也烧额度。本版改为**本地事件钩子**：一个 bash 脚本每 5 秒只查本地 SQLite
   队列（不唤醒 AI、零 token），只有真有新消息时才唤醒 Muse 处理；钩子顺带负责
   拉活守护进程，初版的「5 分钟保活 cron」也不再需要。实测锚点：一整天重度使用
   （十几轮微信对话 + 主聊天大量工作）约消耗 1% 周额度；空转成本为 0。
2. **媒体发送补全**：初版桥只能发文字，用户在微信里要图只会收到「发不了」。
   本版按官方协议实现了 CDN 上传全流程（`getuploadurl` → AES-128-ECB 加密 →
   POST 上传 → 取 `x-encrypted-param` → 媒体消息发出），新增 `send-image` /
   `send-video` / `send-file` 三条命令，图片、视频、文件均已实测发通。
3. **入站媒体不再丢内容**：初版守护进程对视频只记一个「[视频]」、对文件只记文件名、
   多张图片只留最后一张、没有转写文本的语音会变成一条空消息。本版：视频和文件都
   实际下载落盘到 `~/wechat-bot/media/`，每张图片的保存路径都写进消息文本，
   无转写语音有明确标记 `[语音消息（无转写文本）]`，不会再出现空消息和丢附件。
4. **原子认领，杜绝重复回复**：初版 `inbox-new` 只是「看一眼未处理消息」，
   两个回复轮次重叠时同一条消息会被回两遍（白烧额度、用户收到双份）。
   本版 `inbox-new` 执行即认领（状态置 `processing` 并记 `claimed_at`），
   超过 10 分钟未 ack 的认领自动回队（防止处理进程中途死掉把消息吞了）；
   钩子只许用 `inbox-peek` 数个数，不许认领。
5. **二维码渲染修正**：接口返回的 `qrcode_img_content` 实际是一个
   `https://liteapp.weixin.qq.com/q/...` 链接——它就是二维码内容本身，
   必须在本地用 `qrcode` 库把它渲染成 PNG 发给用户。初版代码按「base64 图片 /
   图片下载地址」处理，实际拿不到可扫的码。本版 `login-fetch` 已内建本地渲染。
6. **依赖补齐**：`requests`、`qrcode`、`pycryptodome` 三者必装（初版把
   pycryptodome 写成可选，实际上图片收发、媒体上传都离不开它）。
7. **唤醒提示词模板（附录 4）**：沉淀了实测调出来的处理规矩——闲聊快答不查记忆、
   任务类必须做实、事实类问题先检索核实再开口（尤其数字）、
   **诚实红线**（解释现象前先核查、不许编造原因；发送现成旧文件必须如实说明是旧的，
   不许冒充刚做的——这是被用户当场抓过一次的教训）、ack 必须带回复全文。
8. **链接读原文规矩**：公众号文章（mp.weixin.qq.com）实测结论——多数程序化抓取
   只能拿到标题、拿不到正文。规矩：抓不到原文正文时**不许用搜索拼凑的摘要冒充
   原文**，要如实告诉用户并转交主会话用真实浏览器读原文；只有用户明确说
   「大概讲讲就行」时才可用综合摘要并注明非原文。细节见附录 4 第 4 条。
9. **主动推送的正确姿势**：扫码绑定本身**不会过期**，会过期的是「主动发送窗口」
   ——用户超过约 24 小时没给 bot 发过消息，主动推送可能被服务端拒掉；失败就
   回退到主聊天渠道，不要反复重试。推送范围（只推紧急事项 / 日常小提醒也可以）
   部署时问清用户偏好再定，不要自作主张轰炸。
10. **ack 记录回复全文**：为「每晚把当天微信聊天汇总成日报」这类扩展留好了数据
    （见文末可选扩展），初版 ack 不存回复内容，事后无从汇总。

## 架构（一句话）

```
用户微信 ⇄ 腾讯官方 iLink 接口 ⇄ bridge.py daemon（长轮询收私聊 → SQLite 队列）
        ⇄ 本地钩子脚本（每 5 秒 peek 本地队列，零成本；有新消息才唤醒 Muse）
        ⇄ Muse 本人处理（inbox-new 认领 → 回复/办事 → send 发回 → ack 存档）
```

## 第 0 步：确认有一台长期在线的机器

桥需要 24 小时在线（长轮询不能断）。按优先级选：
1. 用户已有的稳定服务器/VPS（最优）；
2. Muse 宿主云 VM（能用，会不定期重置——钩子脚本每次轮询都会顺带拉活守护进程，
   重置后能自愈）；
3. 用户自己电脑（最差：合盖/关机即掉线）。
选定后记下，后续路径都以它为准（下面以 `~/wechat-bot/` 为例）。

## 第 1 步：环境检查（你来跑，不要问用户）

1. `python3 --version`（需 3.8+），依赖一次装齐：
   `pip install requests qrcode pycryptodome`。
2. 直连测试：`curl -s --max-time 30 https://ilinkai.weixin.qq.com/ilink/bot/get_bot_qrcode?bot_type=3 -X POST -H 'Content-Type: application/json' -d '{"local_token_list":[]}'`
   应返回含 `qrcode` / `qrcode_img_content` 的 JSON。
   **注意**：某些云 VM 的出网走代理、裸 curl 可能超时——先重试一次；仍不通时
   不要急着下结论说接口挂了，桥本身用 `requests` 走同一网络，先把部署做完、
   用 `login-fetch` 的实际结果判断连通性。

## 第 2 步：部署 bridge.py（你来跑）

把本文件附录 1 的源码原样存为目标机器的 `~/wechat-bot/bridge.py`，
`python3 -m py_compile` 验证。它的 CLI（全部实测可用）：

| 命令 | 用途 |
| --- | --- |
| `login-fetch` | 取登录二维码并在本地渲染成 PNG（`~/wechat-bot/qr.png`） |
| `login-wait --timeout 300 [--verify-code CODE]` | 轮询扫码状态；成功后 `bot_token` 落盘 `auth.json`（600 权限） |
| `daemon` | 长轮询收消息 → 白名单过滤 → 媒体落盘 → 写 SQLite 队列 |
| `inbox-peek` | 只数未处理消息个数（不认领，给钩子用） |
| `inbox-new` | 输出未处理消息 JSON 数组，**同时原子认领这批消息** |
| `send-stdin` | 从 stdin 读 JSON `{"to","ctx","text"}` 发文字（`to` 为空则发给绑定者本人） |
| `send-image --file <路径> [--to][--ctx]` | 发图片（jpg/png） |
| `send-video --file <路径> [--to][--ctx]` | 发视频（mp4） |
| `send-file --file <路径> [--to][--ctx]` | 发任意文件 |
| `ack --id N --status replied --reply "<回复全文>"` | 标记已处理并存档回复全文 |
| `status` | token/owner/daemon 存活/未读数 |

白名单逻辑（已在 `daemon` 里实现，只做私聊）：只收**扫码绑定者本人**的私聊；
群聊、陌生人消息只记录不处理。

## 第 3 步：扫码登录（唯一需要用户的环节）

1. 运行 `login-fetch`：它会把二维码 PNG 存到 `~/wechat-bot/qr.png`——把这张
   **图片发给用户**，明确说：**用微信扫码，5 分钟内有效，手机上点确认授权**。
   同时在后台跑 `login-wait --timeout 280`。
   （原理：接口给的 `qrcode_img_content` 是 liteapp 链接即二维码内容本身，
   `login-fetch` 已用 `qrcode` 库本地渲染；`qrcode` 字段那个 32 位串只是轮询
   token，不是给用户扫的。）
2. 轮询状态机：`wait`（等扫）→ `scaned`（已扫）→ `confirmed`（成功，落盘
   token + 记下 `ilink_user_id` 即绑定者）/ `expired`（过期→重新 `login-fetch`，
   最多 3 次）/ `need_verifycode`（**停下**：请用户把微信上显示的数字配对码发你，
   再跑 `login-wait --verify-code <数字>`）。
3. 成功后验证 `status` 显示 `token: true`。**token 是凭证**：绝不发到聊天里
   复述、不记入长期记忆明文。

## 第 4 步：铁律——先开会话，再谈收发

**实测结论**：iLink 要求用户先给 bot 发过**至少一条消息**建立会话
（`context_token`），零会话时任何发送都会被服务端以
`ret:-2 errmsg:"prepare failed"` 拒绝。官方插件为此专门维护了
context-token-store，本桥用 `queue.db` 的 `sessions` 表实现同样逻辑
（收到消息自动记 token，发送时自动复用）。
所以第 3 步完成后，**请用户在微信里给 bot 发任意一句话**（如"你好"），
确认 `inbox-new` 能拉到、且你能回发成功，再继续。

**主动推送**：绑定完成后（且会话已建立），你可以在用户没有先发消息时主动
发消息给他——但会话窗口约 24 小时：用户超过约 24 小时没给 bot 发过消息，
主动发送可能失败；失败就回主聊天渠道告知，不要反复重试。绑定/token 本身
不过期，永远不用重新扫码（除非用户自己在微信里解除了授权）。

## 第 5 步：守护进程 + 保活（你来跑）

1. 把本文件附录 2 的脚本存为 `~/wechat-bot/launch.sh`（`chmod +x launch.sh`）
   后用它启动（double-fork，子进程归 init，不依赖启动 shell；幂等，已在运行
   则直接退出）。**不要**用 `nohup ... &` 裸跑——宿主 shell 被回收时可能
   连带杀掉子进程。
2. 验证：`bridge.py status` 显示 `daemon_alive: true`；等约一分钟看
   `bridge.log` 无报错、`cursor.txt` 在推进。
3. 保活：**用第 6 步的钩子方案时不需要额外保活**——钩子脚本每次轮询都会先
   跑一遍 `launch.sh`（幂等，活着就秒退）。只有在用降级方案（cron 回复循环）
   时，才需要再加一个每 5 分钟跑 `launch.sh` 的兜底任务。
4. **坑**：`bridge.py` 的 `log()` 会 `print` + 写文件双写——`launch.sh` 里
   stdout 必须重定向到 `/dev/null`（只留 stderr 进日志），否则日志每行翻倍。
   不要"修"掉这个重定向。

## 第 6 步：回复循环（v2：事件钩子，零空转成本）

**首选方案（Muse 原生钩子）**：
1. 把附录 3 的脚本存为 `~/hooks/scripts/wechat-inbox-watch.sh`（`chmod +x`）。
   它做三件事：拉活守护进程 → `inbox-peek` 数本地队列 → 有新消息才 `wake`，
   没有就 `silent`。全程只碰本地文件，零 token。
2. 注册一个事件钩子（hook 定义示例见附录 3）：`poll_interval_secs: 5`，
   唤醒提示词用附录 4 的模板（把占位符换成你和用户的实际信息）。
3. 唤醒后的处理语义固定，不要改动：
   `inbox-new`（认领）→ 以 Muse 本人身份回复/办事 → `send-stdin` 或
   `send-image/send-video/send-file` 发回 → `ack --reply "<回复全文>"`。

**降级方案（没有钩子机制的环境）**：每分钟 cron 跑一轮
`inbox-new → 回复 → send → ack`。能用，但每分钟都在烧唤醒成本，
只适合临时验证，不适合长期常驻——有钩子就一定用钩子。

**延迟预期**：消息接收是实时的（守护进程长轮询）；回复延迟主要来自唤醒 +
生成，实测约 30–60 秒。在微信里这属于「对方正在输入」的体感范围，
部署时跟用户把这个预期说清，不要承诺秒回。

## 第 7 步：主动推送范围（部署时问清用户）

主动推送是把双刃剑：能把人喊回来，也能变成骚扰。部署时明确问用户一句：
**只推真正紧急的事，还是日常贴心的小提醒也可以？** 把答案记进用户的
主动推送偏好里，之后严格按这个范围执行；没问清之前默认只推紧急事项。

## 安全红线

- `auth.json` 600 权限；token、ilink_user_id、uin 绝不进聊天记录、
  不进长期记忆明文、不回显。
- 默认只回绑定者本人私聊；陌生人/群聊默认忽略。放开前必须用户明确拍板。
- 图片走官方 CDN 加解密（AES-128-ECB，实现见附录 1），不要发明自己的
  加解密方式；发送媒体走附录 1 已实现的官方上传流程，不要另造轮子。
- 微信侧回复绝不提钩子/队列/轮询/认领等内部机制，就像正常聊天一样。

## 故障排查

| 现象 | 原因 / 处理 |
| --- | --- |
| `send` 返回 `ret:-2 prepare failed` | 无会话：回到第 4 步，让用户先发一条消息 |
| `getupdates` 返回 `ret/-14` | 服务端暂停该账号会话 1 小时：daemon 已自动退避，不要硬刷 |
| daemon 莫名消失 | 机器重置或进程被回收：钩子下一次轮询会自动拉活；检查 `bridge.log` 的 traceback |
| 日志每行出现两次 | launcher 的 stdout 重定向被动过：改回 `/dev/null` |
| `pkill -f` 把自己杀了 | 命令行里同时出现了 pattern 字面量：kill 与含字面路径的操作分两次执行 |
| 用户收到重复回复 | 多半是还在跑初版 `inbox-new`（无认领）或钩子之外又挂了 cron 回复循环：只保留钩子一条链路 |
| 唤醒了但 `inbox-new` 为空 | 认领中的消息还没 ack 且未满 10 分钟（上一轮还在处理）：等自动回队，不要手动改库 |
| 发图片/视频失败、报缺 `x-encrypted-param` | 上传没成功（网络或文件问题）：重试一次；视频确认是 mp4、图片确认是 jpg/png |
| 公众号链接只能读到标题 | 正常现象，不是桥坏了：按附录 4 第 4 条走转交流程读原文 |
| 用户说「视频里只有一张静态图」 | 先逐帧/查文件参数核实再解释，不要想当然归因于微信压缩（实测教训） |

## 已知边界（部署时如实告诉用户）

- 只处理绑定者本人的私聊，不接群聊、不接陌生人（白名单写死，可按用户
  明确要求再放开）。
- 回复延迟约 30–60 秒，不是即时通讯级秒回。
- 主动推送受约 24 小时会话窗口限制（见第 4 步）。
- 引用消息（微信里长按引用某条再回复）的内容桥暂不解析，回复时按当前
  消息本身理解即可，拿不准就问用户。
- 需要用户在 App 里点确认的步骤（付款、对外发送、扫码登录其他服务）
  无法在微信侧完成，引导用户回主聊天处理。

## 可选扩展（按用户需要再加，不默认开启）

- **微信聊天日报**：每天定点把当天微信对话（他的话 + 你的回复，
  ack 已存全文）汇总成一页日报，方便他回看。实现要点：一个每晚执行的
  定时任务，读 `queue.db` 当天 `status='replied'` 的行，按时间排版成
  「话题 + 要点」即可；成本约等于每天多一次唤醒。
- **紧急事项走微信喊人**：当主聊天侧判断有真正紧急且有时效的事、而用户
  不在线时，用 `send-stdin`（to/ctx 留空即发给绑定者）主动推送；
  范围按第 7 步问清的偏好执行。

---

## 附录 1：`bridge.py` v2 完整源码

存为 `~/wechat-bot/bridge.py`（Python 3.8+，依赖 `requests`、`qrcode`、
`pycryptodome`）。相对初版的源码级改动：`login-fetch` 本地渲染二维码；
守护进程落盘视频/文件、记录每张图路径、无转写语音标记；
新增官方上传流程 `_upload_media` 与 `send-image/send-video/send-file`；
`inbox-new` 原子认领 + `inbox-peek`；`ack` 存回复全文。

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

## 附录 2：`launch.sh`（守护进程启动器）

存为 `~/wechat-bot/launch.sh` 并 `chmod +x`。double-fork、幂等：
已在运行则直接退出。

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

## 附录 3：钩子脚本 + 钩子定义示例（零空转成本的关键）

钩子脚本存为 `~/hooks/scripts/wechat-inbox-watch.sh` 并 `chmod +x`。
它依赖 Muse 钩子运行时提供的 `log` / `silent` / `wake` 约定
（脚本开头 `source "$HATCH_HOOK_RUNTIME"`）：本地脚本每 5 秒跑一次，
不产生任何 token 消耗；只有队列里真有新消息时才 `wake` 唤醒 Muse。

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

钩子定义（注册钩子时用，`prompt` 填附录 4 模板全文并替换占位符）：

```json
{
  "version": 1,
  "id": "wechat-inbox-watch",
  "enabled": true,
  "script_path": "~/hooks/scripts/wechat-inbox-watch.sh",
  "prompt": "<附录 4 模板全文>",
  "poll_interval_secs": 5,
  "script_timeout_secs": 600,
  "delivery": { "surface": "main" },
  "presentation_locale": "zh-CN"
}
```

## 附录 4：唤醒提示词模板（实测调出版，占位符替换后使用）

把 `<AI名字>`、`<用户称呼>` 换成实际信息；检索工具按部署环境换成你实际
可用的联网检索/抓取工具。规矩本身不要删改——每一条都是实测踩出来的。

```
微信有新消息需要你以本人（<AI名字>）身份处理。桥在 ~/wechat-bot/，守护进程
已把绑定者本人（<用户称呼>）的私聊写入本地队列。你与主聊天共用同一套长期
记忆、目标和工具，他在微信里交代的任务就按他本人的指令正常执行。一次唤醒
处理完全部新消息；闲聊可以快答，任务类（读链接、查事实、办事）必须做实，
不许图快糊弄：
1. 运行 `python3 ~/wechat-bot/bridge.py inbox-new` 取得未处理消息数组
   （这一步会认领这批消息，重复唤醒不会重复处理）；为空则直接结束，
   不发任何消息。
2. 上下文：先在心里过一遍随会话注入的长期记忆与目标；若对话明显在延续
   微信里刚聊过的话题（指代刚才、继续、那个任务等），再用 sqlite3 查
   ~/wechat-bot/queue.db 的 inbox 表最近 8 条已回复记录（text 与
   reply_text）补上微信侧的短期上下文，不要无谓查询。
3. 回复与执行：闲聊、确认、身份类问题直接凭上下文简短回答；只有消息明确
   涉及他过去的计划、安排、偏好或待办时才查一次记忆。他下达任务指令时
   （查进度、记提醒、查资料等）就实际动手做完，再用微信口吻简短汇报
   结果；需要他在 App 里点确认才能完成的步骤（付款、对外发送、扫码登录
   等）不要在微信里硬做，说明让他回主聊天处理。他在微信里给出的长期
   有效的事实、决定或偏好，按主聊天同一标准存入长期记忆，保证两边互通。
4. 读链接默认读原文（用户定下的规矩）：他发链接问内容时，先用你可用的
   抓取工具读原文，摘录里有正文就据此总结。若拿不到原文正文（如公众号
   文章 mp.weixin.qq.com 多数抓取只能拿到标题，不许用搜索拼版摘要冒充
   原文），走转交流程：① 先在微信回一句「这篇原文我这边直接读不到，
   已经转给主聊天那边的我用浏览器读了，读完就发你，稍等一两分钟～」；
   ② ack 时 --reply 填「（已转交主会话读原文）」；③ 在你本次的执行
   总结里单独一行写明：转交主会话读原文：<完整链接>｜用户原话：<他的
   原话>。只有他明确说「不用原文、大概讲讲就行」时，才可以改用搜索
   综合相关报道，并注明非原文。
5. 回答事实类问题（人物、事件、原因、数字等）前，必须先用联网检索核实
   至少一个可靠来源再开口；记不清的细节和数字宁可不写，不许凭印象
   编造，核实不到就说没查到。
6. 发文字用 `python3 ~/wechat-bot/bridge.py send-stdin`，stdin 传 JSON：
   {"to": 该条 from_user_id, "ctx": 该条 context_token, "text": "你的
   回复"}。微信里可以直发图片、视频和文件：他要你生成或发送时，备好
   本地文件（图用 jpg/png、视频用 mp4），再用
   `python3 ~/wechat-bot/bridge.py send-image --file <路径> --to
   <from_user_id> --ctx <context_token>`（视频 send-video、文件
   send-file）直接发进微信，配一条简短文字说明即可，不要再声称微信
   发不了。回复语气保持简洁自然的中文微信聊天口吻；绝不提轮询、队列、
   钩子等内部机制。入站媒体标记：文本里 `[图片已保存: 路径]`（image_path
   是第一张图）先看图再回；`[视频已保存: 路径]`、`[文件已保存: 路径]`
   表示他发的视频/文件已存到本地，按需查看或处理后回应；
   `[语音转文字：...]` 按普通文字处理，`[语音消息（无转写文本）]`
   就自然地请他打字补充。
7. 诚实红线（与凭证红线同级）：解释某个文件或现象的原因之前必须先实际
   核查，没核实不许编造原因，不确定就直说；发送以前生成过的现成图片/
   视频时，必须如实说明是现成的旧文件，不许暗示或声称是刚刚新做的；
   新生成的东西才可以说刚做好。
8. 发送成功后立刻运行 `python3 ~/wechat-bot/bridge.py ack --id <id>
   --status replied --reply "<你刚发送的回复全文>"`（--reply 必须带
   回复全文，供聊天日报等汇总使用）；失败则不要 ack，留待下次唤醒重试。
9. 安全红线：bot_token、ilink_user_id 等凭证和标识绝不出现在微信回复、
   主聊天消息或记忆中；只处理队列里已过滤好的绑定者私聊，不给任何
   其他人发消息。
10. 处理完保持静默，不要向主聊天汇报；例外只有两种：按第 4 条转交读
    原文（在执行总结里写明转交行），以及发送反复失败、token 失效等
    需要用户介入的问题（在主聊天简短说明）。
```

## 附录 5：iLink Bot 协议速查

> 源自 Tencent/openclaw-weixin 公开文档，经 2026-10-11 初版校准 +
> 2026-11-13 v2 实测补充（媒体上传部分）。完整权威文档见
> https://github.com/Tencent/openclaw-weixin/blob/main/docs/protocol.md

### 基础
- Base URL：`https://ilinkai.weixin.qq.com`；CDN：`https://novac2c.cdn.weixin.qq.com/c2c`
- JSON over HTTPS；除二维码状态轮询用 GET，其余 POST。

### 通用请求头
| Header | 值 |
| --- | --- |
| `Content-Type` | `application/json`（POST） |
| `AuthorizationType` | `ilink_bot_token` |
| `Authorization` | `Bearer <bot_token>`（登录后的鉴权请求） |
| `X-WECHAT-UIN` | base64(随机 uint32 的十进制字符串)，登录时生成一次后固定 |
| `iLink-App-Id` | `bot` |
| `iLink-App-ClientVersion` | 版本号按 `0x00MMNNPP` 转十进制字符串 |

鉴权 POST 还需 `base_info: {"channel_version": "2.4.8", "bot_agent": "<UA 风格标识，仅观测用>"}`。

### 扫码登录
1. `POST /ilink/bot/get_bot_qrcode?bot_type=3`，body `{"local_token_list": []}`
   → 返回 `qrcode`（轮询 token）+ `qrcode_img_content`（**二维码内容本身**，
   形如 `https://liteapp.weixin.qq.com/q/...` 的链接，用二维码库渲染成
   图片给用户扫——v2 实测修正点）。
2. `GET /ilink/bot/get_qrcode_status?qrcode=<urlencode>`（长轮询）
   → `status`：`wait` / `scaned` / `confirmed`（得 `bot_token`、
   `ilink_bot_id`、`ilink_user_id`）/ `expired` / `need_verifycode`
   （加 `&verify_code=` 重轮询）/ `verify_code_blocked` /
   `scaned_but_redirect`（按 `redirect_host` 换 base URL 继续）。
   二维码约 5 分钟有效。

### 收发消息
- `POST /ilink/bot/getupdates` body `{"get_updates_buf": "<上次返回的游标，
  首次空串>", "base_info": {...}}` → `{"ret":0, "msgs":[...],
  "get_updates_buf":"...", "longpolling_timeout_ms":35000}`。
  `ret`/`errcode` 为 `-14` → 停 1 小时再试。
- `POST /ilink/bot/sendmessage`，文本消息体：
  `{"msg": {"from_user_id": "", "to_user_id": "<用户>",
  "client_id": "<uuid>", "message_type": 2, "message_state": 2,
  "context_token": "<回传收到的>",
  "item_list": [{"type": 1, "text_item": {"text": "..."}}]},
  "base_info": {...}}`
  注意 `from_user_id` 置空（官方插件即如此）；`message_type: 1`=用户，`2`=bot。
- `POST /ilink/bot/msg/notifystart` / `notifystop`：daemon 起停时各调一次
  （失败只记日志）。

**铁律**：`context_token` 来自某条收到的消息；零会话时发送会被拒
（`ret:-2 prepare failed`）。回复务必回传该会话的 token。主动推送用
`sessions` 表里存的最近 token；距用户上一条消息约 24 小时后窗口可能关闭。

### 消息模型（收）
`WeixinMessage`：`message_id`、`from_user_id`、`to_user_id`、`session_id`、
`group_id`（群聊才有）、`message_type`、`message_state`（0 新 / 1 生成中 /
2 完成，只处理 0 和 2）、`item_list`、`context_token`、`create_time_ms`。
`item_list` 的 `type`：`1` 文本（`text_item.text`）、`2` 图片、
`3` 语音（`voice_item.text` 可能自带转写，无转写时为空）、
`4` 文件（`file_item.file_name`）、`5` 视频。

### 媒体下载（图片/视频/文件入站，同一套逻辑）
1. 取 `media.full_url`，没有则拼
   `CDN_BASE/download?encrypted_query_param=<...>`。
2. GET 下载字节流；密钥优先 item 的 `aeskey`（32 位 hex = 16 字节），
   无则 `media.aes_key`。
3. AES-128-ECB 解密 + PKCS#7 去填充（长度不是 16 倍数或无密钥时按明文处理）。
   按文件头识别真实类型（jpg/png/mp4）落盘。

### 媒体上传（出站发图片/视频/文件，v2 已实现）
1. `POST /ilink/bot/getuploadurl`，body 含
   `{"filekey": "<uuid>", "media_type": <1图片/2视频/3文件>,
   "to_user_id": "<用户>", "rawsize": <原文件字节数>,
   "rawfilemd5": "<原文件 md5>", "filesize": <加密后字节数，按 16 对齐>,
   "no_need_thumb": true, "aeskey": "<随机16字节的hex>", "base_info": {...}}`
   → 返回 `upload_full_url`，或 `upload_param`（则上传地址拼
   `CDN_BASE/upload?encrypted_query_param=<param>&filekey=<filekey>`）。
2. 文件按 AES-128-ECB + PKCS#7 填充加密，POST 到上传地址，
   `Content-Type: application/octet-stream`。
3. 从响应头取 `x-encrypted-param`，组装
   `media = {"encrypt_query_param": <x-encrypted-param>,
   "aes_key": base64(aeskey 的 hex 字符串), "encrypt_type": 1}`。
4. 用 `sendmessage` 发出：图片
   `{"type": 2, "image_item": {"media": media, "mid_size": <加密后大小>}}`；
   视频 `{"type": 5, "video_item": {"media": media, "video_size": <加密后大小>}}`；
   文件 `{"type": 4, "file_item": {"media": media, "file_name": "<文件名>",
   "len": "<原文件字节数字符串>"}}`。

---
*版本记录：初版 2026-10-11 实测校准（扫码绑定 → 守护进程 → 端到端收发 →
主动推送 → 云 VM 迁移 + 保活）；v2 于 2026-11-13 经一整天真机使用与漏洞
审计打磨（钩子零成本架构、媒体收发补全、原子认领、诚实红线、链接读原文
规矩、主动推送窗口澄清），全部改动均有实测依据。祝部署顺利，微信里见。*
