# 微信聊天每日 AI 总结 + 自动同步到滴答清单（手机提醒）

> 用 Claude Code 自动读取你昨日微信聊天 → 生成结构化日报 → 写入 Obsidian → 把待办和会议推送到滴答清单 → iPhone 自动提醒。
> 全程本地运行，零 API 费用，每天 23:00 自动跑。

---

## 一、能实现什么

**早上打开手机 → 滴答清单已经有了一组**「今日待办 + 今日会议」**，全部来自昨晚 Claude 对你微信的智能提炼**：

- **📅 有明确时间的事件**（会议、约访、电话会、面见、活动）→ 自动加日历事件，**提前 1 小时 + 10 分钟双提醒**
- **✅ 没明确时间的待办**（"给某人发资料"、"研究某公司"等）→ 加待办，**次日 09:00 提醒**
- **📝 Obsidian 每日笔记** → 结构化的当日聊天摘要 + 待办 + 日程 + 需跟进事项

闲聊、表情包、新闻链接、行情评论 **不会被做成任务**。

---

## 二、整体架构

```
微信数据库 (SQLCipher 加密)
    ↓ wechat-decrypt 从内存提取密钥后解密
解密后的 SQLite 数据库
    ↓ export_all_chats.py (按日期范围)
当日 JSON 文件 (每个会话一份)
    ↓ Python 脚本合并 + 过滤排除聊天
聚合 Markdown 消息文件
    ↓ Claude Code CLI (headless 模式)
    ↓ 输出两份结构化产出
总结 .md  +  tasks.json
    ↓                ↓
Obsidian Vault    滴答清单 API
    每日笔记          ↓
                  iPhone 推送提醒
```

---

## 三、技术栈

| 组件 | 作用 | 链接 |
|------|------|------|
| **wechat-decrypt** | 解密 Windows 微信 4.x 数据库 | [GitHub](https://github.com/ylytdeng/wechat-decrypt) |
| **Claude Code CLI** | 本地大模型总结（无 API 费用） | [官网](https://claude.com/claude-code) |
| **滴答清单 OpenAPI** | 创建带提醒的任务 | [开发者中心](https://developer.dida365.com/manage) |
| **Obsidian** | 笔记存储（任意 markdown 编辑器都可）| [obsidian.md](https://obsidian.md) |
| **Windows 任务计划程序** | 每日 23:00 自动触发 | 系统内置 |

需要的环境：
- Windows 10/11 + Python 3.10+
- 微信 4.x（电脑版，保持登录状态）
- 任意 Claude Code 订阅（Pro / Max / Team 都可）
- 滴答清单账号（国际版叫 TickTick）

---

## 四、详细搭建步骤

### Step 1：安装 wechat-decrypt

```powershell
# 下载源码
Invoke-WebRequest "https://github.com/ylytdeng/wechat-decrypt/archive/refs/heads/main.zip" -OutFile "$env:TEMP\wd.zip"
Expand-Archive "$env:TEMP\wd.zip" "$env:TEMP\wd_extract" -Force
Copy-Item "$env:TEMP\wd_extract\wechat-decrypt-main\*" "C:\wechat-decrypt\" -Recurse -Force

# 安装依赖
pip install pycryptodome zstandard mcp pyyaml
```

> ⚠️ 如果你的 git config 里有 URL 重写规则，需要直接下 zip，不要 `git clone`

**测试解密**（确保微信在运行 + 已登录）：

```powershell
cd C:\wechat-decrypt
$env:PYTHONIOENCODING="utf-8"; $env:PYTHONUTF8="1"
python main.py decrypt
```

成功后会在 `C:\wechat-decrypt\decrypted\` 看到 ~20 个解密后的 SQLite 数据库。

**测试导出某天的消息**：

```powershell
python export_all_chats.py --start 2026-05-29 --end 2026-05-30 "C:\wechat-decrypt\daily_export"
```

---

### Step 2：找到 Claude Code 可执行文件路径

```powershell
Get-ChildItem "$env:APPDATA\Claude\claude-code" -Recurse -Filter "claude.exe"
```

例如：`C:\Users\<你的用户名>\AppData\Roaming\Claude\claude-code\2.1.149\claude.exe`

确认它能调用：
```powershell
& "<上面那个路径>" --version
```

---

### Step 3：注册滴答清单开发者应用

1. 打开 https://developer.dida365.com/manage （国际版用 ticktick.com）
2. 用滴答清单账号登录 → **New App**
3. **OAuth redirect URL 必须填**：`http://localhost:8080/callback`（少一个字符都不行）
4. 创建成功后，去应用详情页**编辑**，再次确认 redirect URL 已保存（**这是最容易踩的坑**）
5. 记下 **Client ID** 和 **Client Secret**

---

### Step 4：项目目录结构

在任意位置建一个文件夹，例如 `C:\wechat_daily_summary\`：

```
wechat_daily_summary/
├── main.py                 主程序
├── config.yaml             配置（含凭据，注意保密）
├── ticktick_oauth.py       OAuth 一次性授权脚本
├── ticktick_client.py      滴答清单 API 客户端
├── ticktick_token.json     OAuth 后自动生成
├── run_daily.bat           计划任务入口
├── setup_daily_task.bat    创建 Windows 计划任务
└── requirements.txt
```

---

### Step 5：配置文件 `config.yaml`

```yaml
# Claude Code CLI 路径
claude_cli_path: "C:\\Users\\<你的用户名>\\AppData\\Roaming\\Claude\\claude-code\\<版本号>\\claude.exe"

# wechat-decrypt 项目路径
wechat_decrypt_path: "C:\\wechat-decrypt"
decrypted_db_dir: "C:\\wechat-decrypt\\decrypted"

# Obsidian vault 路径（任意 markdown 文件夹都行）
obsidian_vault_path: "<你的 Obsidian vault 绝对路径>"
daily_notes_folder: "微信每日总结"

# 不想总结的聊天（部分匹配）
excluded_chats:
  - "文件传输助手"
  - "微信团队"
  - "微信支付"

# 滴答清单同步配置
ticktick:
  enabled: true
  region: "dida"            # 国内版 dida，国际版 ticktick
  client_id: "<填你的 Client ID>"
  client_secret: "<填你的 Client Secret>"
  project_name: "微信每日跟进"
  default_tags:
    - "微信跟进"
  scheduled_reminders:
    - "TRIGGER:-PT1H"        # 提前 1 小时
    - "TRIGGER:-PT10M"       # 提前 10 分钟
  todo_default_time: "09:00"
  todo_reminders:
    - "TRIGGER:PT0S"          # 准时

# Claude 总结提示词
summary_prompt: |
  你是一个高效的个人助理。需要读取微信聊天记录文件，生成两个产出：
  1. 中文每日总结的 Markdown 文件
  2. 结构化的 tasks.json 任务清单（供滴答清单同步）

  ## 总结 Markdown 要求
  - 按聊天对象/群聊分组总结
  - 提取关键信息、待办事项、重要决定
  - 忽略无实质内容的闲聊
  - 输出简洁

  Markdown 格式（直接写入文件，不要代码块包裹）：
  ## 📋 今日概览
  ## 💬 聊天摘要
  ### [联系人/群名]
  ## ✅ 待办事项
  ## 📅 日程相关
  ## 🔍 需要跟进

  ## tasks.json 要求
  从聊天中提取明确的"事件"和"待办"，输出 JSON。
  - scheduled：有具体日期和时间的（会议、约访、电话会）
  - todo：需要主动行动但没绑定具体时间的
  - 闲聊、新闻、行情评论不要做成任务
  - priority: 5=紧急, 3=普通, 1=低
  - 不确定时间按 todo 处理

  JSON 示例：
  {
    "tasks": [
      {
        "title": "XX 公司专家电话会",
        "content": "背景/出处/详情",
        "type": "scheduled",
        "date": "2026-05-30",
        "time": "15:00",
        "duration_minutes": 60,
        "priority": 3,
        "tags": ["电话会议"]
      },
      {
        "title": "联系某某发资料",
        "content": "来源：xxx",
        "type": "todo",
        "date": "2026-05-30",
        "priority": 1,
        "tags": ["跟进"]
      }
    ]
  }
```

---

### Step 6：核心代码

#### `ticktick_oauth.py` — 一次性 OAuth 授权

启动一个本地 HTTP 服务监听 `:8080`，浏览器打开授权页，用户点同意后回调拿到 code，再换 access_token。Token 有效期约半年。

```python
"""
滴答清单 OAuth 一次性授权脚本
完成后生成 ticktick_token.json
"""
import sys, json, yaml, time, socket, urllib.parse, urllib.request, webbrowser
from pathlib import Path
from http.server import HTTPServer, BaseHTTPRequestHandler
from threading import Thread

if sys.platform == "win32":
    sys.stdout.reconfigure(encoding="utf-8")

REDIRECT_URI = "http://localhost:8080/callback"
SCOPE = "tasks:write tasks:read"
DIDA_AUTH = "https://dida365.com/oauth/authorize"
DIDA_TOKEN = "https://dida365.com/oauth/token"
TICKTICK_AUTH = "https://ticktick.com/oauth/authorize"
TICKTICK_TOKEN = "https://ticktick.com/oauth/token"

SCRIPT_DIR = Path(__file__).parent
TOKEN_FILE = SCRIPT_DIR / "ticktick_token.json"
CONFIG_FILE = SCRIPT_DIR / "config.yaml"

_received = {"code": None, "error": None}


class CallbackHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        params = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
        if "code" in params:
            _received["code"] = params["code"][0]
            self.send_response(200)
            self.send_header("Content-Type", "text/html; charset=utf-8")
            self.end_headers()
            self.wfile.write("<h1>授权成功！可以关闭网页</h1>".encode("utf-8"))
        elif "error" in params:
            _received["error"] = params["error"][0]
            self.send_response(400); self.end_headers()

    def log_message(self, *args): pass


def http_post(url, data, auth):
    import base64
    cred = base64.b64encode(f"{auth[0]}:{auth[1]}".encode()).decode()
    req = urllib.request.Request(
        url,
        data=urllib.parse.urlencode(data).encode(),
        headers={
            "Content-Type": "application/x-www-form-urlencoded",
            "Authorization": f"Basic {cred}",
        },
        method="POST",
    )
    with urllib.request.urlopen(req, timeout=30) as resp:
        return json.loads(resp.read().decode())


def main():
    cfg = yaml.safe_load(CONFIG_FILE.read_text(encoding="utf-8"))
    tt = cfg["ticktick"]
    client_id, client_secret = tt["client_id"], tt["client_secret"]
    region = tt.get("region", "dida")
    auth_url = DIDA_AUTH if region == "dida" else TICKTICK_AUTH
    token_url = DIDA_TOKEN if region == "dida" else TICKTICK_TOKEN

    authorize_url = f"{auth_url}?" + urllib.parse.urlencode({
        "client_id": client_id,
        "scope": SCOPE,
        "state": "wx-daily",
        "redirect_uri": REDIRECT_URI,
        "response_type": "code",
    })

    print(f"打开浏览器授权...\n{authorize_url}\n")

    server = HTTPServer(("localhost", 8080), CallbackHandler)
    Thread(target=server.serve_forever, daemon=True).start()
    webbrowser.open(authorize_url)

    deadline = time.time() + 300
    while time.time() < deadline:
        if _received["code"] or _received["error"]:
            break
        time.sleep(0.5)
    server.shutdown()

    if _received["error"]:
        print(f"❌ {_received['error']}"); sys.exit(1)

    resp = http_post(token_url, {
        "code": _received["code"],
        "grant_type": "authorization_code",
        "scope": SCOPE,
        "redirect_uri": REDIRECT_URI,
    }, (client_id, client_secret))

    TOKEN_FILE.write_text(json.dumps({
        "access_token": resp["access_token"],
        "expires_in": resp.get("expires_in"),
        "obtained_at": int(time.time()),
        "region": region,
    }, indent=2), encoding="utf-8")
    print(f"✓ Token 已保存到 {TOKEN_FILE}")


if __name__ == "__main__":
    main()
```

#### `ticktick_client.py` — 滴答清单 API 封装

```python
"""滴答清单 API 客户端"""
import json, urllib.request
from datetime import datetime, timedelta
from pathlib import Path


class TickTickError(Exception): pass


class TickTickClient:
    def __init__(self, access_token, region="dida"):
        self.access_token = access_token
        self.api_base = (
            "https://api.dida365.com/open/v1" if region == "dida"
            else "https://api.ticktick.com/open/v1"
        )

    def _request(self, method, path, body=None):
        url = f"{self.api_base}{path}"
        headers = {
            "Authorization": f"Bearer {self.access_token}",
            "Content-Type": "application/json",
        }
        data = json.dumps(body, ensure_ascii=False).encode() if body else None
        req = urllib.request.Request(url, data=data, headers=headers, method=method)
        try:
            with urllib.request.urlopen(req, timeout=30) as r:
                raw = r.read().decode()
                return json.loads(raw) if raw else {}
        except urllib.error.HTTPError as e:
            raise TickTickError(f"HTTP {e.code}: {e.read().decode()[:300]}")

    def list_projects(self):
        return self._request("GET", "/project")

    def ensure_project(self, name):
        for p in self.list_projects():
            if p.get("name") == name:
                return p["id"]
        return self._request("POST", "/project", {"name": name, "color": "#FF6161"})["id"]

    def create_task(self, task):
        return self._request("POST", "/task", task)


def normalize_task(t, default_todo_time, scheduled_reminders, todo_reminders, default_tags):
    """Claude 简化格式 → 滴答清单 API 格式"""
    title = (t.get("title") or "").strip()
    if not title:
        raise ValueError("title 不能为空")

    tags = list(t.get("tags") or [])
    for dt in default_tags:
        if dt not in tags:
            tags.append(dt)

    api = {
        "title": title,
        "content": (t.get("content") or "").strip(),
        "priority": int(t.get("priority", 1)),
        "tags": tags,
        "timeZone": "Asia/Shanghai",
    }

    if t.get("type") == "scheduled" and t.get("time"):
        h, m = t["time"].split(":")
        start = datetime.fromisoformat(t["date"]).replace(hour=int(h), minute=int(m))
        end = start + timedelta(minutes=int(t.get("duration_minutes", 60)))
        api.update({
            "startDate": start.strftime("%Y-%m-%dT%H:%M:%S+0800"),
            "dueDate": end.strftime("%Y-%m-%dT%H:%M:%S+0800"),
            "isAllDay": False,
            "reminders": scheduled_reminders,
        })
    else:
        h, m = default_todo_time.split(":")
        due = datetime.fromisoformat(t["date"]).replace(hour=int(h), minute=int(m))
        api.update({
            "startDate": due.strftime("%Y-%m-%dT%H:%M:%S+0800"),
            "dueDate": (due + timedelta(minutes=30)).strftime("%Y-%m-%dT%H:%M:%S+0800"),
            "isAllDay": False,
            "reminders": todo_reminders,
        })
    return api


def load_token(token_file):
    p = Path(token_file)
    if not p.exists():
        raise TickTickError(f"未找到 token 文件: {token_file}")
    return json.loads(p.read_text(encoding="utf-8"))
```

#### `main.py` — 主程序

完整流程：解密 → 导出 → 合并 → Claude → Obsidian + 滴答清单。
核心逻辑：

```python
"""微信每日总结主程序"""
import os, sys, json, yaml, shutil, subprocess, argparse, tempfile
from datetime import datetime, date, timedelta
from pathlib import Path

if sys.platform == "win32":
    sys.stdout.reconfigure(encoding="utf-8")
    sys.stderr.reconfigure(encoding="utf-8")


def load_config(p=None):
    p = p or Path(__file__).parent / "config.yaml"
    return yaml.safe_load(Path(p).read_text(encoding="utf-8"))


def run_subprocess(cmd, cwd=None, timeout=600):
    env = os.environ.copy()
    env["PYTHONIOENCODING"] = "utf-8"; env["PYTHONUTF8"] = "1"
    return subprocess.run(cmd, cwd=cwd, capture_output=True, text=True,
                          encoding="utf-8", errors="replace", timeout=timeout, env=env)


def decrypt_databases(wd_path):
    print("[1/5] 解密微信数据库...")
    r = run_subprocess([sys.executable, str(Path(wd_path) / "main.py"), "decrypt"],
                       cwd=wd_path, timeout=600)
    if r.returncode != 0:
        print(f"  ✗ 解密失败: {r.stderr[:300]}"); return False
    print("  ✓ 完成"); return True


def export_messages(wd_path, target_date, export_dir):
    if export_dir.exists(): shutil.rmtree(export_dir)
    export_dir.mkdir(parents=True)
    start = target_date.strftime("%Y-%m-%d")
    end = (target_date + timedelta(days=1)).strftime("%Y-%m-%d")
    print(f"[2/5] 导出 {start} 的聊天...")
    r = run_subprocess([sys.executable, str(Path(wd_path) / "export_all_chats.py"),
                        "--start", start, "--end", end, str(export_dir)],
                       cwd=wd_path, timeout=900)
    return r.returncode == 0


def load_exported_chats(export_dir, excluded):
    chats, excluded_lower = [], [e.lower() for e in (excluded or [])]
    for f in sorted(Path(export_dir).glob("*.json")):
        try:
            data = json.loads(f.read_text(encoding="utf-8"))
        except: continue
        name = data.get("chat", "")
        if any(e in name.lower() or e in data.get("username", "").lower() for e in excluded_lower):
            continue
        msgs = [m for m in data.get("messages", [])
                if m.get("type") != "system" and m.get("content", "").strip()]
        if not msgs: continue
        chats.append({**data, "messages": msgs})
    return chats


def build_markdown(chats, target_date):
    lines = [f"# 微信聊天记录 - {target_date}\n",
             f"共 {len(chats)} 个会话，{sum(len(c['messages']) for c in chats)} 条消息\n"]
    for c in sorted(chats, key=lambda c: -len(c["messages"])):
        marker = "👥" if c.get("is_group") else "👤"
        lines.append(f"\n## {marker} {c['chat']}\n_{len(c['messages'])} 条_\n")
        for m in c["messages"]:
            ts = m.get("timestamp", 0)
            tstr = datetime.fromtimestamp(ts).strftime("%H:%M") if ts else "??:??"
            sender = m.get("sender", "").strip() or "我"
            content = m.get("content", "").strip().replace("\n", " ⏎ ")
            mtype = m.get("type", "text")
            if mtype in ("image", "video", "voice", "emoji", "location", "file"):
                content = f"[{mtype}] {content}".strip()
            lines.append(f"- `{tstr}` **{sender}**: {content}")
    return "\n".join(lines)


def call_claude(claude_exe, prompt, msg_file, summary_file, tasks_file, target_date, model=None):
    full_prompt = (
        f"{prompt}\n\n"
        f"## 上下文\n今天是 {target_date}\n明天是 {target_date + timedelta(days=1)}\n\n"
        f"## 任务\n"
        f"1. 读 `{msg_file.as_posix()}`\n"
        f"2. 总结写入 `{summary_file.as_posix()}`\n"
        f"3. tasks.json 写入 `{tasks_file.as_posix()}`\n"
        f"4. 完成后直接结束\n"
    )
    cmd = [claude_exe, "-p", full_prompt,
           "--permission-mode", "acceptEdits",
           "--allowed-tools", "Read,Write",
           "--add-dir", str(msg_file.parent),
           "--add-dir", str(summary_file.parent),
           "--add-dir", str(tasks_file.parent),
           "--output-format", "text"]
    if model: cmd += ["--model", model]
    print("[4/5] 调用 Claude Code 总结...")
    r = run_subprocess(cmd, timeout=600)
    if r.returncode != 0:
        print(f"  ✗ Claude 失败: {r.stderr[:500]}"); return False
    if not tasks_file.exists():
        tasks_file.write_text('{"tasks": []}', encoding="utf-8")
    return summary_file.exists()


def sync_ticktick(tasks_file, cfg, target_date):
    tt = cfg.get("ticktick", {})
    if not tt.get("enabled"):
        print("[5/5] 滴答清单未启用"); return True
    from ticktick_client import TickTickClient, normalize_task, load_token
    if not tasks_file.exists(): return True
    tasks = json.loads(tasks_file.read_text(encoding="utf-8")).get("tasks", [])
    if not tasks: print("[5/5] 无任务"); return True
    print(f"[5/5] 同步 {len(tasks)} 个任务...")
    token = load_token(Path(__file__).parent / "ticktick_token.json")
    client = TickTickClient(token["access_token"], region=tt.get("region", "dida"))
    pid = client.ensure_project(tt.get("project_name"))
    s = f = 0
    for raw in tasks:
        try:
            api = normalize_task(raw,
                tt.get("todo_default_time", "09:00"),
                tt.get("scheduled_reminders"),
                tt.get("todo_reminders"),
                tt.get("default_tags", []))
            api["projectId"] = pid
            client.create_task(api)
            s += 1; print(f"  ✓ {api['title']}")
        except Exception as e:
            f += 1; print(f"  ✗ {raw.get('title')}: {e}")
    print(f"  {s} 成功 / {f} 失败")
    return f == 0


def run(config_path=None, target_date=None):
    cfg = load_config(config_path)
    target_date = target_date or date.today()
    print(f"\n=== 微信每日总结 {target_date} ===\n")

    decrypt_databases(cfg["wechat_decrypt_path"])
    export_dir = Path(tempfile.gettempdir()) / "wechat_daily_export"
    if not export_messages(cfg["wechat_decrypt_path"], target_date, export_dir):
        sys.exit(1)

    print("[3/5] 合并消息...")
    chats = load_exported_chats(export_dir, cfg.get("excluded_chats", []))
    if not chats: print("  无消息"); return
    print(f"  {len(chats)} 会话 / {sum(len(c['messages']) for c in chats)} 条")

    work = Path(tempfile.gettempdir()) / "wechat_summary"; work.mkdir(exist_ok=True)
    msg_file = work / f"messages_{target_date}.md"
    msg_file.write_text(build_markdown(chats, target_date), encoding="utf-8")

    vault = Path(cfg["obsidian_vault_path"]) / cfg.get("daily_notes_folder", "微信总结")
    vault.mkdir(parents=True, exist_ok=True)
    summary_file = vault / f"{target_date} 微信总结.md"
    tasks_file = work / f"tasks_{target_date}.json"
    for f in [summary_file, tasks_file]:
        if f.exists(): f.unlink()

    if not call_claude(cfg["claude_cli_path"], cfg["summary_prompt"],
                       msg_file, summary_file, tasks_file, target_date, cfg.get("model")):
        sys.exit(1)

    sync_ticktick(tasks_file, cfg, target_date)
    print(f"\n✓ 完成: {summary_file}\n")


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("--config", "-c")
    ap.add_argument("--date", "-d")
    args = ap.parse_args()
    td = datetime.strptime(args.date, "%Y-%m-%d").date() if args.date else None
    run(args.config, td)
```

#### `run_daily.bat` — 计划任务调用

```bat
@echo off
chcp 65001 >nul
set PYTHONIOENCODING=utf-8
set PYTHONUTF8=1
set DIR=%~dp0
if not exist "%DIR%logs" mkdir "%DIR%logs"
set LOG=%DIR%logs\run_%date:~0,4%-%date:~5,2%-%date:~8,2%.log
python "%DIR%main.py" >> "%LOG%" 2>&1
exit /b %errorlevel%
```

---

### Step 7：跑一次 OAuth 授权（一次性）

```powershell
cd C:\wechat_daily_summary
$env:PYTHONIOENCODING="utf-8"; $env:PYTHONUTF8="1"
python ticktick_oauth.py
```

浏览器自动打开 → 点同意 → 跳转 `localhost:8080/callback` 显示「授权成功」→ 生成 `ticktick_token.json`。

> **常见错误**：`At least one redirect_uri must be registered with the client`
> 解决：去开发者后台**编辑应用**，再次确认 redirect URL 已保存。

---

### Step 8：手动测试整个流程

```powershell
$env:PYTHONIOENCODING="utf-8"; $env:PYTHONUTF8="1"
python main.py
```

成功标志：
1. Obsidian vault 里出现 `微信每日总结/YYYY-MM-DD 微信总结.md`
2. iPhone 滴答清单出现「微信每日跟进」清单 + 一堆任务
3. 有时间的事件有双提醒

---

### Step 9：设置每日自动运行

以**管理员身份**打开 PowerShell：

```powershell
$bat = "C:\wechat_daily_summary\run_daily.bat"
schtasks /create /tn "WeChatDailySummary" `
    /tr "`"$bat`"" /sc daily /st 23:00 /rl HIGHEST /f
```

> ⚠️ 路径若在 OneDrive 同步目录可能会触发权限问题；推荐放在 `C:\` 根下的本地目录。

常用命令：

```powershell
schtasks /run /tn WeChatDailySummary       # 立即跑一次
schtasks /query /tn WeChatDailySummary     # 看状态
schtasks /change /tn WeChatDailySummary /st 22:00   # 改时间
schtasks /delete /tn WeChatDailySummary /f # 删除
```

---

## 五、踩坑总结

| 问题 | 原因 / 解决 |
|------|------------|
| `UnicodeEncodeError: 'gbk' codec can't encode character` | Windows 默认 GBK 编码，碰到 emoji 群名就崩。必须在脚本开头 `sys.stdout.reconfigure(encoding="utf-8")` + 环境变量 `PYTHONIOENCODING=utf-8` + `PYTHONUTF8=1` |
| `git clone` 拉 github.com 跳转到内网镜像 | 全局 git config 里有 URL 重写规则。临时用 `Invoke-WebRequest` 下 zip 包 |
| `FileNotFoundError: claude.exe` | 检查路径里的版本号；Claude Code 升级后路径会变 |
| `OAuth Error: invalid_request, At least one redirect_uri must be registered` | 滴答清单后台没保存 redirect URL，回去编辑应用再次保存 |
| 计划任务跑了但失败（错误码 0x80070005）| Access Denied，多数是 OneDrive 路径权限。把项目移到 `C:\` 本地目录 |
| Claude Code 找不到登录态 | 先在终端 `claude` 命令登录一次，session 长期有效 |
| 微信解密失败 | 微信必须**已登录**（不能只启动了进程没登录），脚本必须**管理员权限** |
| 滴答清单任务没推送提醒 | iPhone 系统设置 → 通知 → 滴答清单 → 允许通知 |

---

## 六、安全说明

- `ticktick_token.json` 含 access_token，**不要提交到 Git**
- `config.yaml` 的 `client_secret` 同样敏感
- 解密后的微信数据库 `C:\wechat-decrypt\decrypted` 包含全部历史聊天，注意磁盘加密
- 整条 pipeline **本地运行**，聊天内容不会发到任何外部服务（除了滴答清单 API 拿到的是任务标题+背景说明，不是原始聊天）

---

## 七、自定义建议

**改总结风格**：改 `config.yaml` 里的 `summary_prompt`，告诉 Claude 你想要的格式。

**改提醒时间**：
```yaml
ticktick:
  scheduled_reminders:
    - "TRIGGER:-PT30M"      # 提前 30 分钟
  todo_default_time: "08:00"
  todo_reminders:
    - "TRIGGER:PT0S"
```

**分清单**：目前所有任务进同一清单，想分类可让 Claude 在 tasks.json 里加 `tags`，靠滴答清单的智能清单过滤。

**接其他工具**：把 `ticktick_client.py` 换成 Todoist / Google Tasks / Notion API，结构完全一样。

---

## 八、致谢

- [@ylytdeng/wechat-decrypt](https://github.com/ylytdeng/wechat-decrypt) — 优秀的微信 4.x 解密工具
- [Anthropic Claude Code](https://claude.com/claude-code) — 本地无费用的 LLM CLI
- [滴答清单](https://dida365.com) — 开放 API 友好

---

整套搭建大概 1-2 小时，难点主要在调试编码问题。如果朋友照着搭遇到问题，欢迎来问。
