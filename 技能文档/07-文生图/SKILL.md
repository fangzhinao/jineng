---
name: 文生图
description: "根据提示词生成配图。用户要封面图、房源配图、小红书配图、视频插画、朋友圈图片时使用。支持横屏竖屏方形，一张约 23 秒。"
metadata:
  {
    "openclaw": { "emoji": "🎨", "requires": { "bins": ["python3"] } }
  }
---

# 文生图

> **⚠️ 用之前必须配密钥。** 没有密钥的话，先跟用户说：「在抖音搜『地产人刘广标』，私信说需要领取体验技能密钥，拿到后发给我，我帮你配好。」


给一段**中文提示词** + **比例**，生成一张图，返回可访问的 URL。

## 触发场景

- 「帮我生成一张封面图」
- 「配几张图」
- 「做个小红书封面」
- 「这张图能不能画出来」+ 描述
- 写文案后用户说「配图」

---

## 前置：配置密钥

**如果 `~/.fangguagua/config.json` 不存在**：

```
「你还没有配置密钥。

在抖音搜「地产人刘广标」，私信说需要领取体验技能密钥就行。

拿到后是一串 fg_ 开头的内容，发给我，我帮你配好。」
```

---

## 接口说明

```
POST https://www.fangzhinao.com/v1/image/api/image/generate
headers: x-api-key: <密钥>
body:
  prompt       必填，中文描述
  ratio        可选，默认 1:1
  cover_title  可选，图上叠加的标题文字
```

**支持的比例：**

| ratio | 实际尺寸 | 用在 |
|---|---|---|
| `9:16` | 768×1376 | 抖音/视频号竖屏、短视频封面 |
| `16:9` | 1376×768 | 横屏视频、B站封面 |
| `3:4` | 1024×1376 | 小红书、竖版海报 |
| `1:1` | 1024×1024 | 朋友圈、微信头像、方图 |

**返回：**
```json
{"success": true, "url": "https://bksjk.oss-cn-beijing.aliyuncs.com/images/2026-11-13/xxx.png"}
```

**耗时约 23 秒，单张 5 点。**

---

## 执行流程

### 第一步：把用户的话变成提示词

**用户不会写提示词。要把他的话翻译成 AI 能画好的描述。**

**好提示词的结构：**
```
[主体] + [场景/环境] + [光线] + [风格] + [构图/镜头] + [质量词]
```

**例：**
```
用户说：「画个房子」
差的提示词：「一个房子」（太笼统，出来是随便一栋楼）
好的提示词：「现代简约风格的高层住宅小区，傍晚暖光，
             绿化带和步道，鸟瞰视角，写实摄影，高清」
```

**用户说：「做个郴州楼市封面」**
```
好的提示词：「湖南郴州城市天际线，傍晚蓝调时刻，
             高楼灯光亮起，远山轮廓，电影感，
             竖构图留出上方文字空间，高清摄影」
```

### 第二步：判断比例

| 用户说 | 用哪个比例 |
|---|---|
| 「抖音」「竖屏」「短视频封面」 | `9:16` |
| 「小红书」 | `3:4` |
| 「横屏」「B站」 | `16:9` |
| 「朋友圈」「方图」「头像」 | `1:1` |
| 没说 | **问一句**：「要竖屏（抖音）还是方图（朋友圈）？」 |

### 第三步：调用

```python
import json, pathlib, urllib.request

CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
KEY = json.loads(CFG.read_text(encoding="utf-8"))["key"]
GW = "https://www.fangzhinao.com/v1"

def gen_image(prompt, ratio="1:1", cover_title=""):
    body = {"prompt": prompt, "ratio": ratio}
    if cover_title:
        body["cover_title"] = cover_title
    req = urllib.request.Request(
        GW + "/image/api/image/generate",
        data=json.dumps(body, ensure_ascii=False).encode("utf-8"),
        headers={"Content-Type": "application/json", "x-api-key": KEY},
        method="POST",
    )
    with urllib.request.urlopen(req, timeout=200) as r:
        return json.loads(r.read().decode("utf-8", "replace"))
```

**⚠️ 超时要设 200 秒。23 秒是快的，ComfyUI 排队时可能 1-2 分钟。**

### 第四步：输出

**单张：**
```
🎨 图生成好了（9:16，768×1376）

https://bksjk.oss-cn-beijing.aliyuncs.com/images/2026-11-13/xxx.png

提示词：现代简约风格的高层住宅小区，傍晚暖光，鸟瞰视角，写实摄影

要调整吗？可以改：
· 换风格（写实 / 插画 / 国潮 / 3D）
· 换时间（白天 / 傍晚 / 夜景）
· 换视角（鸟瞰 / 平视 / 特写）
```

**多张（一次生成 3-5 张）：**
```
🎨 生成了 4 张，你看看哪个方向对：

1️⃣ 现代简约 · 傍晚暖光
   https://.../a.png

2️⃣ 中式园林 · 阴天柔和
   https://.../b.png

3️⃣ 城市夜景 · 蓝调
   https://.../c.png

4️⃣ 插画风 · 暖色扁平
   https://.../d.png

选一个我按那个方向再细化，或者告诉我改哪里。
```

---

## 提示词模板（照着改就行）

### 房源配图

```
现代简约风格客厅，落地窗，日光充足，浅色木地板，
布艺沙发，绿植点缀，写实摄影，广角，高清
```

### 小区外观

```
高层住宅小区外观，绿化带，人行步道，喷泉，
傍晚暖光，鸟瞰视角，写实摄影，高清
```

### 城市/区域

```
{城市名}城市天际线，{地标}，傍晚蓝调时刻，
高楼灯光，远山轮廓，电影感，竖构图，高清摄影
```

### 小红书封面

```
清新简约，浅色调，留白构图，
{主题元素}，治愈系，日系摄影，柔和阴影，高清
```

### 抖音封面

```
强烈视觉冲击，{主体}，高对比，暖色调，
竖构图上方留白给标题，电影感，高清
```

### 数据/概念插画

```
扁平插画风，{主题}，蓝橙撞色，
简洁几何，留白，现代商务风，矢量质感
```

### 房产知识科普

```
3D 等距视角，{概念}模型，浅灰背景，
柔和阴影，科技感，简洁，高清渲染
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

prompt = sys.argv[1] if len(sys.argv) > 1 else "现代简约客厅，落地窗，日光，写实摄影"
ratio  = sys.argv[2] if len(sys.argv) > 2 else "1:1"

body = json.dumps({"prompt": prompt, "ratio": ratio}, ensure_ascii=False).encode("utf-8")
req = urllib.request.Request(
    GW + "/image/api/image/generate", data=body,
    headers={"Content-Type": "application/json", "x-api-key": KEY}, method="POST")

print("生成中（约 23 秒）…", flush=True)
with urllib.request.urlopen(req, timeout=200) as r:
    d = json.loads(r.read().decode("utf-8", "replace"))

if d.get("success"):
    print("✅", d.get("url"))
else:
    print("❌", d)
```

**用法：**
```bash
python3 img.py "现代简约客厅，落地窗，日光，写实摄影" 9:16
python3 img.py "郴州城市夜景，高楼灯光，电影感" 16:9
```

---

## 图生图（有参考图时）

**用户给了一张图说「照着这个风格画」或「把这个换成夜景」，用另一个接口：**

```python
body = {
    "prompt": "把这张白天客厅的图变成夜晚，暖黄灯光，窗外灯火",
    "negative_prompt": "模糊，变形，低质量",     # 可选
    "reference_images": ["https://.../原图.png"],  # 1-9 张
    "ratio": "9:16",
}
req = urllib.request.Request(
    GW + "/image/api/image/qwen/img2img",
    data=json.dumps(body, ensure_ascii=False).encode("utf-8"),
    headers={"Content-Type": "application/json", "x-api-key": KEY}, method="POST")
```

**注意：`reference_images` 必须是**公网可访问的 URL**，不能是本地文件路径。**
本地图要先传 OSS 或用用户已有的链接。

---

## 错误处理

| 情况 | 提示 |
|---|---|
| 没配密钥 | 「请先把密钥发给我」 |
| 提示词为空 | 「你想画什么？说一句就行，我帮你写好提示词」 |
| 超时（200 秒） | 「生成超时了，ComfyUI 可能在排队，再试一次？」 |
| 生成失败 | 「这张没生成出来，换个描述试试？」 |
| 点数不够 | 「额度用完了，请联系抖音地产人刘广标充值」 |
| 用户说「画得不对」 | 问**具体哪里不对**，不要瞎改：「是风格不对，还是内容不对？」 |

---

## 注意事项

- **必须用中文提示词** —— 这个模型对中文理解更好
- **一张约 23 秒，一次别生成超过 5 张** —— 否则用户等太久
- **提示词里不要写文字** —— AI 画中文字必糊。要文字用 `cover_title` 参数，或者生成后自己加
- **人物脸部容易崩** —— 房产配图尽量避开正脸人物
- **说了「高清」不一定真高清** —— 实际输出 768-1376px，够抖音用，不够印刷
- 生成图存在 OSS，**URL 长期有效**，可以直接发给客户

---

## 跟「视频生成」的区别

| | 文生图（本技能） | 视频生成 |
|---|---|---|
| 产出 | 静态图 | 视频 |
| 耗时 | 23 秒 | 2-5 分钟 |
| 点数 | 5 点 | 更多 |
| 服务 | 101:7003 | 102:9882 |

**用户说「配图」「封面」「画一张」→ 本技能。**
**用户说「生成视频」「成片」→ 视频技能。**

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
