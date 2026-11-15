---
name: 微信聊天记录生成
description: "生成以假乱真的微信聊天记录截图，用来发朋友圈展示中介的靠谱、专业、真诚。用户说「做张聊天记录」「生成客户和我的对话」「朋友圈素材」时使用。支持一键生成（AI 写内容）和自定义渲染（自己给对话）。"
metadata:
  {
    "openclaw": { "emoji": "💬", "requires": { "bins": ["python3"] } }
  }
---

# 微信聊天记录生成

做一张看着完全真实的微信聊天截图，发朋友圈用。

**用途：** 让刷到朋友圈的人觉得这个中介**认真、负责、专业、真诚** —— 不是硬广，是「客户信任我」的侧面证明。

## 触发场景

- 「做张聊天记录」「生成一段客户对话」
- 「朋友圈需要素材」
- 「做个客户感谢我的截图」
- 「模拟一段我和客户的聊天」

---

## 前置：配置密钥

**如果 `~/.fangguagua/config.json` 不存在**：

```
「你还没有配置密钥。

在抖音搜「地产人刘广标」，私信说需要领取体验技能密钥就行。

拿到后是一串 fg_ 开头的内容，发给我，我帮你配好。」
```

---

## 三种模式

| 模式 | 接口 | 点数 | 什么时候用 |
|---|---|---|---|
| **一键生成** | `POST /v1/chat/api/one-click` | 8 点 | **最常用**，AI 自己写对话 + 出图 |
| 只生成内容 | `POST /v1/chatgen/api/generate-chat` | 3 点 | 想先看内容、自己改改再出图 |
| 直接渲染 | `POST /v1/chat/api/chat-record` | 8 点 | 对话已经定好了，只出图 |
| 上传头像 | `POST /v1/chatup/api/upload-image` | 1 点 | 用户发了图片文件时先传到这 |
| 生成头像 | `POST /v1/chatai/api/generate-avatar` | 5 点 | 用户只给描述时 |

---

## 一键生成（推荐）

### 请求

```python
import json, pathlib, urllib.request

CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
KEY = json.loads(CFG.read_text(encoding="utf-8"))["key"]
GW = "https://www.fangzhinao.com/v1"

def chat_one_click(city, user_name, properties, rounds=6, quality="", dark=False):
    body = {
        "city": city,                 # "郴州"
        "user_name": user_name,       # "刘广标"
        "properties": properties,     # ⚠️ 必须是对象数组，不能是字符串数组
        "rounds": rounds,             # 对话轮数，4~10 之间
        "quality": quality,           # 空=随机。负责/专业/真诚/踏实/细心
        "dark": dark,                 # 深色模式
    }
    req = urllib.request.Request(
        GW + "/chat/api/one-click",
        data=json.dumps(body, ensure_ascii=False).encode("utf-8"),
        headers={"Content-Type": "application/json", "x-api-key": KEY},
        method="POST")
    with urllib.request.urlopen(req, timeout=300) as r:
        return json.loads(r.read().decode("utf-8", "replace"))
```

### ⚠️ `properties` 的正确格式（最容易错的地方）

```python
# ❌ 错的：字符串数组 → 接口 500
properties = ["金域华府"]

# ✅ 对的：对象数组
properties = [
    {"name": "金域华府", "area": "北湖区", "price": "8500"},
]
```

**`name` 必填，`area` 和 `price` 可选但建议填（AI 会用上）。**

### 返回

```json
{
  "success": true,
  "url": "https://bksjk.oss-cn-beijing.aliyuncs.com/chatrec/output/679413d54a6648f6.png",
  "scene": {
    "quality": "负责",
    "client": {"name": "张哥", "age": 40, "gender": "male", "persona": "中年男性，孩子上初中，看重学位"},
    "avatar_hint": "风景照",
    "messages": [
      {"from": "them", "type": "text", "text": "小刘，金域华府那套8500的，学位到底稳不稳啊？"},
      {"from": "me", "type": "text", "text": "张哥，我昨天下午专门跑了一趟..."},
      {"from": "them", "type": "voice", "seconds": 12},
      ...
    ]
  }
}
```

**`url` 是长图，高度按对话条数自动撑开（可能 3000~6000 px）。**

---

## 只生成内容（先看再改）

```python
def chat_generate(city, user_name, properties, rounds=6,
                  scene_name="", custom_prompt="", cli_name=""):
    body = {
        "city": city,
        "user_name": user_name,
        "properties": properties,
        "rounds": rounds,
        "scene_name": scene_name,       # 场景名，如「学区验证」
        "custom_prompt": custom_prompt, # 自定义方向，如「突出我帮客户查了学位」
        "cli_name": cli_name,           # 指定客户名字，如「张哥」
    }
    req = urllib.request.Request(
        GW + "/chatgen/api/generate-chat",
        data=json.dumps(body, ensure_ascii=False).encode("utf-8"),
        headers={"Content-Type": "application/json", "x-api-key": KEY},
        method="POST")
    with urllib.request.urlopen(req, timeout=300) as r:
        return json.loads(r.read().decode("utf-8", "replace"))
```

**返回 JSON 字符串（不是写死的图）：**
```json
{"quality":"负责","client":{...},"messages":[...]}
```

**流程：**
```
① 生成内容 → 给用户看
② 用户说「第 3 条改成 XXX」
③ 改完 → 用「直接渲染」出图
```

---

## 直接渲染（对话已定）

```python
def chat_render(messages, me_name="我", them_name="客户",
                me_avatar="", them_avatar="",
                title="", bar_time="9:41", dark=False):
    body = {
        "title": title or them_name,     # 顶部标题
        "bar_time": bar_time,            # 状态栏时间
        "dark": dark,
        "me": {"name": me_name, "avatar": me_avatar},
        "them": {"name": them_name, "avatar": them_avatar},
        "messages": messages,
        "show_name": False,              # True 会显示昵称
        "show_bottom": True,             # 是否要底部输入框
        "width": 750,
    }
    req = urllib.request.Request(
        GW + "/chat/api/chat-record",
        data=json.dumps(body, ensure_ascii=False).encode("utf-8"),
        headers={"Content-Type": "application/json", "x-api-key": KEY},
        method="POST")
    with urllib.request.urlopen(req, timeout=300) as r:
        return json.loads(r.read().decode("utf-8", "replace"))
```

### 消息格式

```python
messages = [
    {"from": "them", "type": "text",  "text": "你好，那套房子还在吗"},
    {"from": "me",   "type": "text",  "text": "在的，您什么时候方便来看"},
    {"from": "them", "type": "voice", "seconds": 9},          # 语音，不写 text
    {"from": "me",   "type": "image", "url": "https://..."},  # 图片
]
```

| 字段 | 值 | 说明 |
|---|---|---|
| `from` | `me` / `them` | 谁说的 |
| `type` | `text` / `voice` / `image` | 消息类型 |
| `text` | 字符串 | `type=text` 时必填 |
| `seconds` | 整数 | `type=voice` 时用，2~20 |
| `url` | 图片 URL | `type=image` 时用 |

---

## 头像怎么给（重点）

**用户可能给三种东西，处理方式不同：**

| 用户给什么 | 怎么处理 | 消耗 |
|---|---|---|
| **一个公网图片链接** | 直接塞进 `avatar` 字段 | 0 点 |
| **一张图片文件**（微信发来的图） | 先上传，拿 URL，再塞进去 | 1 点 |
| **一句描述**（「中年男中介」） | 调生成头像接口 | 5 点 |

### 情况一：用户给链接

```python
# 直接当 avatar 用，不用上传
body = {
    "me":   {"name": "我",   "avatar": "https://任何公网链接.jpg"},
    "them": {"name": "李姐", "avatar": "https://任何公网链接.jpg"},
    ...
}
```

**实测过：GitHub 的图、任何公网 CDN 的图都能渲染，不要求是你 OSS 的。**

**⚠️ 但链接必须公网可直接访问：**
```
✅ https://xxx.com/a.jpg
❌ 需要登录才能看的链接
❌ file:///local/path.jpg
❌ 已过期的签名 URL
```

### 情况二：用户发来图片文件

**先上传，拿到 URL：**

```python
import uuid

def upload_avatar(file_path):
    """把本地图片传到房瓜瓜，返回 OSS 链接"""
    boundary = "----DshFormBoundary" + uuid.uuid4().hex
    with open(file_path, "rb") as f:
        raw = f.read()
    parts = [
        ("--" + boundary).encode(),
        b'Content-Disposition: form-data; name="file"; filename="avatar.png"',
        b"Content-Type: image/png",
        b"",
        raw,
        ("--" + boundary + "--").encode(),
        b"",
    ]
    body = b"\r\n".join(parts)
    req = urllib.request.Request(
        GW + "/chatup/api/upload-image", data=body,
        headers={"Content-Type": "multipart/form-data; boundary=" + boundary,
                 "x-api-key": KEY},
        method="POST")
    with urllib.request.urlopen(req, timeout=120) as r:
        return json.loads(r.read().decode("utf-8", "replace"))["url"]
```

**返回：**
```json
{"success": true, "url": "https://bksjk.oss-cn-beijing.aliyuncs.com/chatrec/upload/xxx.jpg", "size": 67473}
```

**然后把这个 URL 塞进 `avatar`。**

**⚠️ 上传的图会存在房瓜瓜 OSS 的 `chatrec/upload/` 目录下，占你 1 点空间但图很小。**

### 情况三：用户只给描述

**调生成头像接口（5 点）：**

```python
def gen_avatar(hint, gender="", age=0):
    body = {"hint": hint, "gender": gender, "age": age}
    req = urllib.request.Request(
        GW + "/chatai/api/generate-avatar",
        data=json.dumps(body, ensure_ascii=False).encode("utf-8"),
        headers={"Content-Type": "application/json", "x-api-key": KEY},
        method="POST")
    with urllib.request.urlopen(req, timeout=300) as r:
        return json.loads(r.read().decode("utf-8", "replace"))
```

**例：**
```python
gen_avatar("中年男中介，穿衬衫", "male", 38)
# → {"url": "https://.../xxx.png", "prompt": "中年男中介，微信头像风格..."}
```

---

## 生成头像（接口细节）

```python
def gen_avatar(hint, gender="", age=0):
    body = {"hint": hint, "gender": gender, "age": age}
    req = urllib.request.Request(
        GW + "/chatai/api/generate-avatar",
        data=json.dumps(body, ensure_ascii=False).encode("utf-8"),
        headers={"Content-Type": "application/json", "x-api-key": KEY},
        method="POST")
    with urllib.request.urlopen(req, timeout=300) as r:
        return json.loads(r.read().decode("utf-8", "replace"))
```

**例：**
```python
gen_avatar("中年男中介，穿衬衫", "male", 38)
gen_avatar("30岁女客户，短发", "female", 30)
```

**返回：**
```json
{"success": true, "url": "https://.../xxx.png", "prompt": "中年男中介，微信头像风格..."}
```

---

## 完整可跑脚本

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json, pathlib, sys, urllib.request

CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
if not CFG.exists():
    sys.exit("❌ 未配置密钥")
KEY = json.loads(CFG.read_text(encoding="utf-8"))["key"]
GW = "https://www.fangzhinao.com/v1"

def post(path, body, timeout=300):
    req = urllib.request.Request(
        GW + path, data=json.dumps(body, ensure_ascii=False).encode("utf-8"),
        headers={"Content-Type": "application/json", "x-api-key": KEY},
        method="POST")
    with urllib.request.urlopen(req, timeout=timeout) as r:
        return json.loads(r.read().decode("utf-8", "replace"))

city = sys.argv[1] if len(sys.argv) > 1 else "郴州"
name = sys.argv[2] if len(sys.argv) > 2 else "刘广标"
house = sys.argv[3] if len(sys.argv) > 3 else "金域华府"

d = post("/chat/api/one-click", {
    "city": city, "user_name": name,
    "properties": [{"name": house, "area": "", "price": ""}],
    "rounds": 6,
})
print("✅", d.get("url"))
sc = d.get("scene") or {}
print("质量:", sc.get("quality"), "｜客户:", (sc.get("client") or {}).get("name"))
for m in (sc.get("messages") or []):
    who = "客户" if m.get("from") == "them" else "我"
    if m.get("type") == "voice":
        print("  %s: [语音 %ss]" % (who, m.get("seconds")))
    else:
        print("  %s: %s" % (who, (m.get("text") or "")[:60]))
```

**用法：**
```bash
python3 chat.py 郴州 刘广标 金域华府
```

---

## 什么场景做成什么样的聊天

**用户说「做张聊天记录」，先问他要哪种：**

| 场景 | 聊什么 | 效果 |
|---|---|---|
| **学区验证** | 客户担心学位，中介跑去查了公示栏、查了前房东学位占用 | 专业 |
| **不硬推** | 客户看了很多套都不满意，中介劝他别急 | 靠谱 |
| **客户感谢** | 成交后客户发来感谢 | 口碑 |
| **功能展示** | 客户问公积金、问贷款，中介答得清楚 | 懂行 |
| **风险提示** | 中介主动告诉客户这套房的问题 | 真诚（最涨粉） |

**技能里可以主动建议：**
```
「做聊天记录，最抓人的三个方向：
 ① 你帮他查到了别人没查的东西（学位、产权、抵押）
 ② 你劝他别买（真诚）
 ③ 成交后他感谢你（口碑）

 你想做哪个方向？我按那个写。」
```

---

## 输出给用户

```
💬 聊天记录做好了

https://bksjk.oss-cn-beijing.aliyuncs.com/chatrec/output/xxx.png

对话内容：
客户：金域华府那套 8500 的，学位到底稳不稳啊？
我：张哥，我昨天下午专门跑了一趟，去看了区教育局门口的公示栏…
客户：[语音 12"]
我：另外这房子的学位有没有被前房东占用，我也去查了…

---
要改哪一句告诉我，或者换一个场景我再做一张。
```

---

## 错误处理

| 情况 | 提示 |
|---|---|
| 没配密钥 | 「请先把密钥发给我」 |
| `properties` 传了字符串 | 内部会 500，**一定要传对象数组** |
| 生成超时（300 秒） | 「生成有点慢，再试一次？」 |
| 内容不满意 | 问「哪里不对」，然后调 `custom_prompt` 重生成 |
| 点数不够 | 「额度用完了，请联系抖音地产人刘广标充值」 |

---

## 注意事项

- **`properties` 必须是 `[{"name": "...", ...}]`** —— 传 `["..."]` 会 500，这是最常见的错
- **一次生成一张** —— 别连着生成 10 张，AI + 渲染很慢
- **头像是随机的** —— 想固定就用 `me_avatar` / `them_avatar` 传 URL
- **长图很长** —— 6 轮对话大概 3000~4000 px 高，微信朋友圈能发
- **内容必须真实可信** —— 编造的对话被熟人看出来会翻车
- **不要用客户真实姓名** —— 用「张哥」「李姐」这种称呼

---

## 跟「文生图」的区别

| | 聊天记录（本技能） | 文生图 |
|---|---|---|
| 产出 | 仿微信界面长图 | 任意图片 |
| 内容 | AI 写的对话 | 提示词描述 |
| 用途 | 朋友圈信任背书 | 封面/配图 |
| 点数 | 8 点 | 5 点 |

**用户说「做张聊天记录」「模拟对话」→ 本技能。**
**用户说「画一张」「做封面」→ 文生图。**

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
