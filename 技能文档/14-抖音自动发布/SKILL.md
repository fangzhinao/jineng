---
name: 抖音自动发布
description: "帮用户把做好的视频自动发到抖音（支持立即发布和定时发布）。用户说「发抖音」「帮我发出去」「定时明天早上发」「批量发布」时使用。注意：这个技能运行在用户自己的 Windows 电脑上，不是云端接口。"
metadata:
  {
    "openclaw": { "emoji": "📤", "requires": { "bins": ["python3", "ffmpeg"] } }
  }
---

# 抖音自动发布

**把视频自动发到抖音创作者后台。支持立即发布、定时发布、自动填标题/描述/话题标签。**

## ⚠️ 这个技能和别的技能不一样

| | 技能 01-13 | 这个技能 |
|---|---|---|
| **运行在哪** | 云端接口（fangzhinao.com） | **用户自己的电脑** |
| **要什么** | 只需要 API 密钥 | 需要本地装环境 + 扫码登录 |
| **谁能用** | 任何电脑 | 只有装了环境的这台 |

**所以第一次用之前，必须先在这台电脑上把环境配好（第二步）。**

---

## 触发场景

- 「帮我把这个视频发抖音」
- 「定时明天早上 9 点发」
- 「批量发布这几条」
- 「发出去之后告诉我链接」

---

## 一、铁律：`headless` 必须 `False`

```
headless=True   → 抖音判定非真人 → 掉登录 / 要短信验证 → 发不出去
headless=False  → 抖音认为正常浏览器 → 直接发出去
```

**这是踩了一整夜的坑。以后写任何抖音脚本，`headless` 一律设 `False`。**

**推论：这台电脑必须有桌面会话（显示器连着 / 远程桌面登着）。锁屏没关系，但断开远程桌面会杀掉浏览器。**

---

## 二、第一次用：装环境（只做一次）

### 2.1 检查有没有现成环境

```bash
# Windows
if exist D:\social-auto-upload\venv\Scripts\python.exe echo 已装
```

**如果已装，跳到第三步。**

### 2.2 装 social-auto-upload + patchright

```powershell
# 1. 装 Python（3.10+）和 ffmpeg（如果没有）
winget install Python.Python.3.12
winget install Gyan.FFmpeg

# 2. 下载 social-auto-upload 到 D:\social-auto-upload
#    GitHub: https://github.com/dreammis/social-auto-upload
#    （或者用户手上已有的压缩包解压过去）

# 3. 建虚拟环境
cd /d D:\social-auto-upload
python -m venv .venv
.venv\Scripts\python.exe -m pip install -U pip

# 4. 装依赖（关键：patchright 不是 playwright）
.venv\Scripts\python.exe -m pip install patchright
.venv\Scripts\python.exe -m patchright install chromium
```

**注意：代码里用的是 `from patchright.async_api import async_playwright`，不是标准 playwright。两个不通用。**

### 2.3 扫码登录（只做一次，登录态能保持几周）

**跑登录脚本，会弹出浏览器停在抖音登录页，让用户扫码：**

```bash
cd /d D:\social-auto-upload
.venv\Scripts\python.exe _login.py
```

**登录成功后，登录态和「可信设备」都保存在 `_profile` 目录。**

---

## 三、`_profile` 是命根子

```
D:\social-auto-upload\_profile\
    ↑
抖音认的就是这个目录
    ↓
别删、别在多个进程同时用、别强杀浏览器
```

**如果登录态坏了（`LOGIN_REQUIRED`）：**
```powershell
# 1. 杀掉所有 chrome 进程
taskkill /F /IM chrome.exe
taskkill /F /IM msedge.exe

# 2. 清掉锁文件
del /q D:\social-auto-upload\_profile\SingletonLock
del /q D:\social-auto-upload\_profile\lockfile

# 3. 重新扫码登录
```

---

## 四、日常发布流程

### 步骤 1：压视频（必须，否则上传超时）

```bash
ffmpeg -y -i 原片.mp4 -c:v libx264 -preset fast -crf 27 -c:a aac -b:a 96k ^
  -movflags +faststart D:\social-auto-upload\videos\pub1.mp4
```

**原片 20 MB+ 压到 10 MB 左右最稳。5 分钟的视频上传要几分钟。**

### 步骤 2：写任务参数

**创建 `D:\social-auto-upload\publish_job.json`：**

```json
{
  "video": "pub1.mp4",
  "title": "标题（30 字内，抖音限制）",
  "desc": "描述正文",
  "tags": ["房产经纪人", "房产文案", "郴州房产"],
  "schedule": null
}
```

| 字段 | 说明 |
|---|---|
| `video` | `videos/` 目录下的文件名 |
| `title` | 作品标题，**30 字以内** |
| `desc` | 描述正文 |
| `tags` | 话题标签数组，**不要带 # 号**（脚本会自动加） |
| `schedule` | `null` = 立即发；`"2026-11-15 09:00"` = 定时发 |

### 步骤 3：执行发布

**必须用计划任务启动，不能直接 Python 跑。**

```powershell
# 一次性创建计划任务（只需一次）
schtasks /create /tn "DouyinPublish" /tr "D:\social-auto-upload\_run_pubgen.bat" /sc once /st 23:59 /ru Administrator /it /f

# 每次发布就跑它
schtasks /run /tn "DouyinPublish"
```

**为什么必须用 schtasks：**
```
SSH 或终端里直接跑 → 会话一断，子进程被杀 → 发布中断
schtasks 跑      → 独立会话，不受影响
```

### 步骤 4：看日志（30 秒后开始看）

```powershell
type D:\social-auto-upload\_pubgen.log
```

**成功的样子：**
```
>>> 已登录，上传 pub1.mp4
>>> 编辑页就绪（4s）
>>> 填标题: 10个房产号，9个在拍房子
>>> 填描述
>>> 点发布 [8]
>>> PUBLISHED ✅
>>> URL: https://creator.douyin.com/creator-micro/content/manage
```

**失败的样子：**

| 日志 | 含义 | 怎么办 |
|---|---|---|
| `>>> LOGIN_REQUIRED` | 登录态掉了 | 重新扫码（第三节） |
| `>>> NO_BUTTON` | 页面结构变了 | 检查是否还在上传页 |
| `>>> SMS_REQUIRED` | 要短信验证 | 见第五节 |

---

## 五、短信验证怎么处理

**正常不用管。但日志出现 `SMS_REQUIRED` 时：**

```
① 脚本会自动点「获取验证码」
② 用户手机收到 6 位数
③ 你把码写进文件：
   echo 816206 > D:\social-auto-upload\_code.txt
④ 脚本每 3 秒检查一次，读到就自动填入、提交
```

**所以遇到 SMS_REQUIRED：**
```
告诉用户：「抖音要短信验证，你手机上收到的 6 位数发我，我填进去。」
用户给码 → 写文件 → 等 30 秒 → 看日志变 PUBLISHED ✅
```

---

## 六、完整脚本（`_pubgen.py`，照抄就行）

**保存到 `D:\social-auto-upload\_pubgen.py`：**

```python
# -*- coding: utf-8 -*-
import asyncio, sys, os, io, json
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8', errors='replace', line_buffering=True)
R = r"D:\social-auto-upload"
PROFILE = os.path.join(R, "_profile")
OUT = os.path.join(R, "_steps")
CODE_FILE = os.path.join(R, "_code.txt")
UPLOAD_URL = "https://creator.douyin.com/creator-micro/content/upload"

def say(x): print(x, flush=True)

async def main():
    job = json.load(open(os.path.join(R, "publish_job.json"), encoding='utf-8'))
    VIDEO = os.path.join(R, "videos", job["video"])
    TITLE = job["title"]
    DESC = job.get("desc", "")
    TAGS = job.get("tags", [])
    SCHEDULE = job.get("schedule")

    from patchright.async_api import async_playwright
    os.makedirs(OUT, exist_ok=True)
    async with async_playwright() as p:
        ctx = await p.chromium.launch_persistent_context(
            PROFILE, headless=False, viewport=None,
            args=["--start-maximized","--no-sandbox","--no-first-run","--no-default-browser-check"])
        pg = ctx.pages[0] if ctx.pages else await ctx.new_page()
        await pg.goto(UPLOAD_URL, wait_until="domcontentloaded", timeout=60000)
        await pg.wait_for_timeout(4000)
        n = await pg.locator("input[type=file]").count()
        if n == 0:
            say(">>> LOGIN_REQUIRED")
            await ctx.close(); return
        say(">>> 已登录，上传 %s" % os.path.basename(VIDEO))
        inputs = await pg.query_selector_all("input[type=file]")
        await inputs[0].set_input_files(VIDEO)

        for i in range(60):
            await pg.wait_for_timeout(2000)
            if await pg.locator('input[placeholder*="作品标题"]').count() > 0:
                say(">>> 编辑页就绪（%ds）" % ((i+1)*2)); break
        await pg.wait_for_timeout(3000)

        say(">>> 填标题: %s" % TITLE)
        tp = pg.locator('input[placeholder*="作品标题"]').first
        await tp.click(); await tp.fill(TITLE)

        say(">>> 填描述")
        dp = pg.locator('div[contenteditable="true"]').first
        await dp.click(); await dp.type(DESC, delay=8)
        for t in TAGS:
            await pg.wait_for_timeout(400)
            await dp.type(" #" + t, delay=35)
            await pg.wait_for_timeout(1400); await pg.keyboard.press("Enter")
        await pg.wait_for_timeout(800)
        await pg.evaluate("window.scrollTo(0, document.body.scrollHeight)")
        await pg.wait_for_timeout(1000)

        if SCHEDULE:
            say(">>> 设定时发布: %s" % SCHEDULE)
            try:
                await pg.locator('text=定时发布').first.click(timeout=6000)
                await pg.wait_for_timeout(1500)
                ti = pg.locator('input[placeholder*="时间"], input[placeholder*="选择"]').first
                await ti.click(); await ti.fill(SCHEDULE)
                await pg.wait_for_timeout(1000)
                await pg.keyboard.press("Enter")
                await pg.wait_for_timeout(1500)
            except Exception as e:
                say("   定时设置失败（改立即发布）: %s" % str(e)[:100])

        btns = await pg.evaluate("""() => {const r=[];document.querySelectorAll('button').forEach((e,i)=>{const t=(e.innerText||'').trim();if(t)r.push({i,t,vis:e.offsetParent!==null});});return r;}""")
        idx = [x['i'] for x in btns if x['t']=='发布' and x['vis']]
        say(">>> 点发布 %s" % idx)
        if not idx:
            say(">>> NO_BUTTON"); await ctx.close(); return
        await pg.locator("button").nth(idx[0]).click()

        for i in range(60):
            await pg.wait_for_timeout(2000)
            try: t = await pg.evaluate("()=>document.body.innerText")
            except Exception: t = ""
            if "验证码" in t and ("短信" in t or "手机" in t):
                say(">>> SMS_REQUIRED")
                try:
                    await pg.locator('text=获取验证码').first.click(timeout=8000)
                    say(">>> 已点获取验证码")
                except Exception: pass
                code = None
                for k in range(120):
                    await pg.wait_for_timeout(3000)
                    if os.path.exists(CODE_FILE):
                        c = open(CODE_FILE, encoding='utf-8').read().strip()
                        if len(c) >= 4: code = c; break
                    if k % 10 == 0: say("   等验证码 %ds" % (k*3))
                if code:
                    say(">>> 填入 %s" % code)
                    try:
                        inp = pg.locator('input[placeholder*="验证码"]').first
                        await inp.click(); await inp.fill(code)
                        await pg.wait_for_timeout(700)
                        await pg.locator('text=验证').last.click(timeout=8000)
                        await pg.wait_for_timeout(12000)
                    except Exception as e:
                        say(">>> 填码失败: %s" % str(e)[:100])
                break
            if "发布成功" in t or "content/manage" in pg.url:
                say(">>> PUBLISHED ✅")
                break
            if i % 5 == 0: say("   [%ds] %s" % ((i+1)*2, pg.url[-45:]))
        await pg.screenshot(path=os.path.join(OUT,"gen_final.png"))
        say(">>> URL: %s" % pg.url)
        await ctx.close()

asyncio.run(main())
```

**配套的 `_run_pubgen.bat`（计划任务入口）：**

```bat
@echo off
chcp 65001 >nul
cd /d D:\social-auto-upload
.venv\Scripts\python.exe -u _pubgen.py > _pubgen.log 2>&1
```

---

## 七、你必须遵守的 5 条

```
1. headless 一律 False
2. _profile 不能删、不能被强杀
3. 电脑必须有桌面会话（锁屏可以，断远程桌面不行）
4. 只用 schtasks 启动，不要终端里直接跑 Python
5. 视频先压缩到 10 MB 以内
```

---

## 八、踩过的坑

| 现象 | 真实原因 | 解法 |
|---|---|---|
| 日志一直「正在上传视频」然后失败重试 | 检测逻辑误判，其实上传成功了 | 别用工具自带的 upload-video，用上面的 `_pubgen.py` |
| `LOGIN_REQUIRED` | 用了 headless | 改成 `headless=False` |
| 掉登录 / 要短信 | profile 被多进程占用 / 被强杀 | 杀 chrome、清 SingletonLock、重新扫码 |
| 点发布没反应 | 页面还在上传/生成封面 | 等「编辑页就绪」再操作 |
| `input[type=file]` 数量为 0 | 在登录页 或 不在上传页 | 重新 goto 上传页 |
| 进程一启动就退 | 终端断开导致子进程被杀 | 用 schtasks |

---

## 九、给用户的话术

**不要说「playwright 启动 persistent context，定位 input[type=file]」这种话。**

**要说：**
```
视频已经发出去了 ✅
标题：10个房产号，9个在拍房子
标签：房产经纪人、房产文案、房产运营

去抖音创作者中心能看到，大概 2-5 分钟后通过审核。
```

**要定时发：**
```
定好了，明天早上 9:00 自动发。
你电脑得开着（锁屏没关系），别关机就行。
```

**要短信验证：**
```
抖音要验证一下手机，你收到的 6 位数发我，我填进去。
```

---

## 十、批量发布

**一次发多条：**

```python
# 循环处理
for job in job_list:
    # 写 publish_job.json
    # schtasks /run
    # 等日志出现 PUBLISHED 或 失败
    # 间隔 3-5 分钟再发下一条（别太快，容易风控）
```

**建议：一天最多发 3 条，间隔 2 小时以上。发太快会触发风控。**
