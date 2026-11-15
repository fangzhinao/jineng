---
name: 抖音文案提取
description: "提取抖音视频的口播文案。用户发来抖音链接、要求提取文案、想扒同行的口播稿、分析某条视频说了什么时使用。输出完整口播稿 + 标题 + 作者 + 封面。"
metadata:
  {
    "openclaw": { "emoji": "🎬", "requires": { "bins": ["python3"] } }
  }
---

# 抖音文案提取

把一条抖音视频的**口播语音**转成文字，拿到完整文案。

## 触发场景

- 用户发来抖音链接（`v.douyin.com/xxx` 或 `douyin.com/video/xxx`）
- 「帮我提取这条视频的文案」
- 「这条视频讲了什么」
- 「把这条口播扒下来」

---

## 前置：配置密钥（只需一次）

**如果 `~/.fangguagua/config.json` 不存在**，说明用户还没配置密钥。执行：

```
告诉用户：
「你还没有配置密钥。

在抖音搜「地产人刘广标」，私信说需要领取体验技能密钥就行。

拿到后是一串 fg_ 开头的内容，发给我，我帮你配好。」
```

**用户发来密钥后，执行这段：**

```python
import json, pathlib
p = pathlib.Path.home() / ".fangguagua"
p.mkdir(parents=True, exist_ok=True)
(p / "config.json").write_text(json.dumps({"key": "把用户给的密钥填这里"}), encoding="utf-8")
print("✅ 密钥已配置")
```

**密钥 `fg_` 开头，40 位左右。不要在本文件里写任何真实密钥。**

---

## 执行流程

### 第一步：读密钥

```python
import json, pathlib, sys
CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
if not CFG.exists():
    print("❌ 未配置密钥，请先把密钥发给我")
    raise SystemExit
KEY = json.loads(CFG.read_text(encoding="utf-8"))["key"]
GW = "https://www.fangzhinao.com/v1"
```

### 第二步：提交提取任务

```python
import urllib.request, time

video_url = "https://v.douyin.com/UdnhMa7nsOw/"   # ← 用户给的链接

body = json.dumps({"url": video_url}).encode("utf-8")
req = urllib.request.Request(
    GW + "/caption/api/caption/extract",
    data=body,
    headers={"Content-Type": "application/json", "x-api-key": KEY},
    method="POST",
)
try:
    with urllib.request.urlopen(req, timeout=60) as r:
        d = json.loads(r.read().decode("utf-8", "replace"))
except Exception as e:
    print("❌ 提交失败：%s，请重试" % e)
    raise SystemExit(1)

if not d.get("success"):
    print("❌ 提交失败：", d.get("message"))
    raise SystemExit(1)

task_id = d["task_id"]
print("已提交，任务号", task_id)
```

### 第三步：轮询结果（最多 3 分钟）

```python
result = None
for i in range(36):            # 36 × 5 秒 = 180 秒
    time.sleep(5)
    q = urllib.request.Request(
        GW + "/caption/api/task/" + task_id,
        headers={"x-api-key": KEY},
    )
    try:
        with urllib.request.urlopen(q, timeout=30) as r:
            t = json.loads(r.read().decode("utf-8", "replace"))
    except Exception as e:
        print("查询失败：%s" % e)
        continue

    st = t.get("status")
    if st == "completed":
        result = t.get("result") or {}
        break
    if st == "failed":
        print("❌ 提取失败：", t.get("message"))
        raise SystemExit(1)
    print("  [%d%%] %s" % (t.get("progress") or 0, t.get("message") or "处理中"))

if not result:
    print("❌ 超时（3 分钟），请稍后重试")
    raise SystemExit(1)
```

### 第四步：输出

**拿到 `result` 后，按下面格式输出给用户：**

```
📝 文案提取成功

标题：{title}
作者：{author}
时长：{duration} 秒

---
{完整 caption}

---
需要我帮你拆解这条文案的结构，或者改写成你城市的版本吗？
```

**`result` 里可用的字段：**

| 字段 | 说明 |
|---|---|
| `title` | 视频标题 |
| `author` | 作者昵称 |
| `caption` | **完整口播稿**（核心） |
| `cover_url` | 封面图（是对象，取 `.url`） |
| `likes` / `comments` / `shares` / `collects` | 互动数据（**可能为 0，不代表视频没数据**） |
| `duration` | 时长（秒，**可能为 0**） |
| `create_time` | 发布时间（**可能为 0**） |

**⚠️ 重要：`likes/comments/duration/create_time` 经常返回 0，这是接口限制，不是出错。**
**如果需要准确的互动数据，用「账号数据分析」技能查作品列表，那里数据是全的。**

---

## 完整可跑脚本

**把下面整段存成 `extract.py`，改 `VIDEO_URL` 就能直接跑：**

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json, pathlib, sys, time, urllib.request

CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
if not CFG.exists():
    sys.exit("❌ 未配置密钥，请先把密钥发给我")
KEY = json.loads(CFG.read_text(encoding="utf-8"))["key"]
GW = "https://www.fangzhinao.com/v1"

VIDEO_URL = sys.argv[1] if len(sys.argv) > 1 else ""

def post(path, payload):
    req = urllib.request.Request(
        GW + path,
        data=json.dumps(payload).encode("utf-8"),
        headers={"Content-Type": "application/json", "x-api-key": KEY},
        method="POST")
    with urllib.request.urlopen(req, timeout=60) as r:
        return json.loads(r.read().decode("utf-8", "replace"))

def get(path):
    req = urllib.request.Request(GW + path, headers={"x-api-key": KEY})
    with urllib.request.urlopen(req, timeout=30) as r:
        return json.loads(r.read().decode("utf-8", "replace"))

d = post("/caption/api/caption/extract", {"url": VIDEO_URL})
if not d.get("success"):
    sys.exit("❌ " + str(d.get("message")))
tid = d["task_id"]

for _ in range(36):
    time.sleep(5)
    t = get("/caption/api/task/" + tid)
    if t.get("status") == "completed":
        r = t.get("result") or {}
        print("标题:", r.get("title"))
        print("作者:", r.get("author"))
        print()
        print(r.get("caption") or "")
        break
    if t.get("status") == "failed":
        sys.exit("❌ " + str(t.get("message")))
else:
    sys.exit("❌ 超时")
```

**用法：**
```bash
python3 extract.py "https://v.douyin.com/UdnhMa7nsOw/"
```

---

## 错误处理

| 情况 | 提示用户 |
|---|---|
| 没配密钥 | 「你还没有配置密钥，请把密钥发给我」 |
| 429 / 点数用完 | 「你的额度用完了，请联系抖音地产人刘广标充值」 |
| 403 密钥禁用 | 「你的密钥已被停用，请联系抖音地产人刘广标」 |
| 提交失败 | 「提取失败，请重试」——不要自己换方案 |
| 超时 | 「处理超时，请稍后重试」 |
| 视频私密/删除 | 「这条视频无法提取（可能是私密或已删除）」 |

**原则：失败就直说，不要猜原因，不要自己换别的办法。**

---

## 注意事项

- 只支持**公开可访问**的抖音视频
- 提取的是**语音转文字**，可能有少量错别字（同音字），属正常
- 短链接和长链接都支持，接口内部会自己解析
- 一次只提一条，多条请分开调用
- 提取成功后，可以配合「爆款框架」或「AI 原创文案」技能继续用

---

## 接口报错时的统一话术（很重要）

**每次调用接口出错，先看状态码，按这张表回用户。不要自己编理由，不要只说「失败了请重试」。**

| 状态码 / 内容 | 真实含义 | 对用户说的话 |
|---|---|---|
| **401** `密钥无效` 或 `缺少密钥` | 没配密钥，或密钥填错了 | 「你还没有配置密钥，或者密钥填错了。<br>在抖音搜「地产人刘广标」，私信说需要领取体验技能密钥。<br>拿到后是一串 `fg_` 开头的内容，发给我，我帮你配好。」 |
| **403** `密钥已被禁用` | 密钥被管理员停用了 | 「你的密钥已被停用。<br>找抖音「地产人刘广标」重新开通一下就行。」 |
| **403** `密钥已过期` | 密钥到期了 | 「你的密钥到期了。<br>找抖音「地产人刘广标」续期。」 |
| **403** `点数已用完` | 额度用光了 | 「你的额度用完了。<br>找抖音「地产人刘广标」充值，充完马上能用。」 |
| **403** 其他（没有知识库 / 没权限） | 套餐不含这个功能 | 「你的密钥还没开通这个功能。<br>找抖音「地产人刘广标」升级套餐。」 |
| **502** | 后端服务异常 | 「服务暂时不稳定，过 5 分钟再试一次。」 |
| **超时** | 任务太慢 | 「处理超时了，稍等一下再试。」 |

**⚠️ 铁律：凡是涉及「没密钥 / 密钥停用 / 额度用完」，必须说出「抖音地产人刘广标」，这是用户唯一的解决路径。**
