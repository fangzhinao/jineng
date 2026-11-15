---
name: 深度解读视频
description: "把一段口播文案做成横屏深度解读视频。自动配真实素材、字幕、章节、封面。用户说「做成视频」「生成视频」「出片」时使用。传文案就出片，不用剪辑。"
metadata:
  {
    "openclaw": { "emoji": "🎬", "requires": { "bins": ["python3"] } }
  }
---

# 深度解读视频

**传一段文案，出一条横屏视频 + 一张封面。字幕、章节、素材、封面全自动。**

## 触发场景

- 「把这条做成视频」「生成视频」「出片」
- 写完文案后用户说「可以，做视频吧」

---

## 前置：配置密钥

**如果 `~/.fangguagua/config.json` 不存在**：

```
「你还没有配置密钥。

在抖音搜「地产人刘广标」，私信说需要领取体验技能密钥就行。

拿到后是一串 fg_ 开头的内容，发给我，我帮你配好。」
```

---

## ⚠️ 顺序不能反：先清洗文案，再调接口

**这个服务的文案是直接喂给 TTS 配音的。不清洗就直接传，会念出很怪的声音。**

```
❌ 直接传 Markdown 文案 → 配音念「星号」「井号」「方括号」
✅ 先清洗成纯中文口语 → 配音自然
```

**清洗这一步是硬前提，不能跳。**

---

## 第一步：文案清洗（11 条自检）

### 必须改掉的（不改会念出怪声）

**① Markdown 符号**

```
❌ **买房必看**  ## 第一点  `代码`  > 引用  | 表格 |
✅ 买房必看    第一点
```
**星号会被念成「星号」，井号念「井号」。全删。**

**② 表情符号**

```
❌ 😀 🔥 ✅ 💰
✅ （全删，一个不留）
```
**TTS 会念「笑脸」「火」「对勾」。**

**③ 舞台提示**

```
❌ （笑）（停顿）（此处省略）（画面：客厅）（配乐：轻快）
✅ （全删）
```
**这些是剧本给演员看的，配音会照念。**

**④ 结构性标签**

```
❌ 【标题】买房必看
   第一点：查产权
   开场：……
   观点1：……
✅ 买房之前，有五件事必须先查清楚。
   第一，查产权。
```
**只留要念的字，标签不念。**

**⑤ 方括号、书名号堆叠**

```
能用顿号或句号代替就用，非用不可时一段里最多一个
```

### 最好改掉的（改了更自然）

**⑥ 长句**

```
❌ 超过四十个字一口气念不完的句子要拆成两三个短句用句号断开
✅ 超过四十个字，一口气念不完。拆成两三个短句，用句号断开。
```

**⑦ 连续标点**

```
❌ 真的很好！！！太棒了。。。
✅ 真的很好。太棒了。
```

**⑧ 符号念法**

```
❌ 30%  ¥100万  5°C  1/3
✅ 百分之三十  一百万  五度  三分之一
```

**⑨ 英文单词**

```
❌ VIP  OK  live  case
✅ 尊享  行  现场  案例
```
**中文 TTS 念英文很生硬。**

**⑩ 引号太多**

```
一段里超过 3 个引号就去掉 —— 配音听不出引号
```

**⑪ 括号注释**

```
❌ 查产权（也就是去不动产登记中心拉一份信息）
✅ 查产权。具体就是跟房东一起去不动产登记中心，拉一份产权信息。
```

---

### 对照示例

**反例（这些都不该出现）**
```
【标题】买房必看
第一点：查产权（很重要）
第二点：查土地性质
```

**正例（直接能念）**
```
买房之前，有五件事必须先查清楚。
第一，查产权。跟房东一起去不动产登记中心拉一份产权信息。
第二，查土地性质。是出让地还是划拨地，划拨地要补出让金。
```

---

## 第二步：检查长度

```
服务能处理的范围：60 ~ 600 字
最佳：200 ~ 400 字（约 1~2 分钟视频）

不到 60 字 → 会失败，先写长一点
超过 600 字 → 章节会太多，建议拆成两条
```

**字数换算（真人语速）：**
```
60 秒  ≈ 260~280 字
90 秒  ≈ 390~420 字
120 秒 ≈ 520~560 字
```

---

## 第三步：调接口

```python
import json, pathlib, urllib.request

CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
KEY = json.loads(CFG.read_text(encoding="utf-8"))["key"]
GW = "https://www.fangzhinao.com/v1"

def make_video(text, title, max_chapters=6, voice_id="realblog_male01"):
    """提交深度解读视频任务，返回 task_id"""
    body = {
        "text": text,                    # 已清洗的纯文案
        "title": title,                  # 15 字以内，最好带城市名
        "max_chapters": max_chapters,    # 章节数，建议 3~6
        "voice_id": voice_id,            # 音色，默认 realblog_male01
        "tts_engine": "minimax",
        "cover_enabled": True,           # 自动生成封面
        "cover_embed_sec": 0.5,
        "upload_oss": True,              # 传到 OSS，返回公网 URL
    }
    req = urllib.request.Request(
        GW + "/landscape/generate_landscape",
        data=json.dumps(body, ensure_ascii=False).encode("utf-8"),
        headers={"Content-Type": "application/json", "x-api-key": KEY},
        method="POST")
    with urllib.request.urlopen(req, timeout=180) as r:
        return json.loads(r.read().decode("utf-8", "replace"))
```

**返回：**
```json
{"task_id": "6fcad0f3-6df8-4708-aa9c-0152f41df7ac",
 "status": "processing",
 "message": "任务已提交，请轮询 /task/{task_id}"}
```

---

## 第四步：轮询等出片

```python
import time

def wait_video(task_id, timeout=600):
    """轮询直到完成，返回完整结果"""
    url = GW + "/landscape_task/task/" + task_id
    t0 = time.time()
    while time.time() - t0 < timeout:
        req = urllib.request.Request(url, headers={"x-api-key": KEY})
        with urllib.request.urlopen(req, timeout=30) as r:
            d = json.loads(r.read().decode("utf-8", "replace"))
        st = d.get("status")
        if st == "success":
            return d
        if st in ("failed", "error"):
            raise RuntimeError(d.get("error") or d.get("message") or "生成失败")
        time.sleep(8)
    raise TimeoutError("超时了，" + task_id + " 还在跑")
```

**轮询免费，不扣点。**

### 完成后的返回结构

```json
{
  "status": "success",
  "progress": 100,
  "video_url": "https://bksjk.oss-cn-beijing.aliyuncs.com/videos/landscape/xxx.mp4",
  "cover_url": "https://bksjk.oss-cn-beijing.aliyuncs.com/videos/landscape/covers/xxx.png",
  "cover_texts": {
    "kicker": "2026买房前必看",
    "title": "买房|必查",
    "sub": "五件事漏一件踩坑"
  },
  "duration": 16.26,
  "chapters": [
    {"name": "开篇", "start": 1, "end": 2, "bucket": "空镜"},
    {"name": "查产权", "start": 3, "end": 5, "bucket": "白天"},
    {"name": "土地性质", "start": 6, "end": 11, "bucket": "特写"}
  ]
}
```

---

## 成片规格（实测）

```
视频   1920 × 1080（横屏 16:9）
时长   16~60 秒（按文案长度）
封面   1080 × 1440，自动生成三行字
```

**自动带这些东西（不用你管，但要知道有）：**

| 元素 | 样子 | 位置 |
|---|---|---|
| **章节标记** | 「开篇」「别买顶楼」「一楼复式」 | 左上角 |
| **素材类型标注** | 「航拍·空镜」「特写」 | 章节旁边的灰字 |
| **主标题** | 黄色描边大字 | 顶部中间 |
| **关键词高亮** | 红色字，卡在说到的位置 | 画面下方 |
| **免责声明** | 「本视频仅为个人观点」竖排 | 左侧 |
| **字幕** | 自动打，跟语音同步 | 底部 |

**封面三行字：**
```
kicker（小标）    2026郴州楼市
title（书法大字）  买房|大实话
sub（副标）       郴州买房三个真相
```

**实测耗时：32 秒**（80 字文案，3 章节）
**点数：20 点/条**（轮询免费）

---

## ⚠️ 已知问题：偶发卡在 72%

**实测遇到过一次：**

```
任务状态停在 processing 72% 「渲染视频…」，等了 8 分钟没动
    ↓
但 102 机器上视频文件已经生成了（14 MB）
    ↓
查 OSS：404，说明上传这一步失败了，状态没更新
```

**概率不高，但要知道怎么处理：**

```python
# 轮询超时后，不要直接说"失败了"
if 超时:
    告诉用户：
    「这条生成得比较慢，可能还需要一会儿。
     我过几分钟再帮你查一下任务。」
    # 记住 task_id，稍后重查
```

**如果重查还是 72%，就重新提交一条。**

**别让用户以为白花钱了 —— 实际上这一条服务端可能没扣成功。**

---

## 完整可跑脚本

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json, pathlib, sys, time, urllib.request

CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
if not CFG.exists():
    sys.exit("❌ 未配置密钥")
KEY = json.loads(CFG.read_text(encoding="utf-8"))["key"]
GW = "https://www.fangzhinao.com/v1"

def post(path, body, timeout=180):
    req = urllib.request.Request(GW + path,
        data=json.dumps(body, ensure_ascii=False).encode("utf-8"),
        headers={"Content-Type": "application/json", "x-api-key": KEY},
        method="POST")
    with urllib.request.urlopen(req, timeout=timeout) as r:
        return json.loads(r.read().decode("utf-8", "replace"))

def get(path, timeout=30):
    req = urllib.request.Request(GW + path, headers={"x-api-key": KEY})
    with urllib.request.urlopen(req, timeout=timeout) as r:
        return json.loads(r.read().decode("utf-8", "replace"))

text  = sys.argv[1] if len(sys.argv) > 1 else ""
title = sys.argv[2] if len(sys.argv) > 2 else "房产干货"
if not text:
    sys.exit("用法: python3 video.py '文案正文' '标题'")

print("提交中…")
d = post("/landscape/generate_landscape", {
    "text": text, "title": title, "max_chapters": 5,
    "cover_enabled": True, "upload_oss": True,
})
tid = d.get("task_id")
print("任务:", tid, "（约 1~2 分钟）")

t0 = time.time()
while time.time() - t0 < 600:
    r = get("/landscape_task/task/" + tid)
    st = r.get("status")
    if st == "success":
        print()
        print("✅ 视频:", r.get("video_url"))
        print("🖼 封面:", r.get("cover_url"))
        print("⏱ 时长: %.1f 秒" % (r.get("duration") or 0))
        ct = r.get("cover_texts") or {}
        print("📝 封面字:", ct.get("kicker"), "/", ct.get("title"), "/", ct.get("sub"))
        break
    if st in ("failed", "error"):
        print("❌ 失败:", r.get("error") or r.get("message"))
        break
    print("   %s %s%% %s" % (st, r.get("progress"), r.get("message") or ""), end="\r")
    time.sleep(8)
```

---

## 交稿格式

```
🎬 视频做好了

视频：https://bksjk.oss-cn-beijing.aliyuncs.com/videos/landscape/xxx.mp4
封面：https://bksjk.oss-cn-beijing.aliyuncs.com/videos/landscape/covers/xxx.png

时长 16.3 秒 | 1920×1080 横屏 | 3 个章节

封面文字：
· 2026买房前必看
· 买房|必查
· 五件事漏一件踩坑

---
要改哪句话告诉我，我重新生成一条。
```

---

## 音色选项

| voice_id | 适合 |
|---|---|
| `realblog_male01` | 男声，真实博主感（默认，最像口播） |
| 其他 | 传其他 ID 会失败，先用默认 |

**想换音色先问一句：「默认是男声博主音，要换吗？」**

---

## 错误处理

| 情况 | 提示 |
|---|---|
| 没配密钥 | 「请先把密钥发给我」 |
| 文案 < 60 字 | 「太短了，加到 200 字以上再做视频」 |
| 文案没清洗 | **先清洗再传**（11 条自检） |
| 提交失败 | 「提交失败，可能服务在忙，再试一次？」 |
| 轮询超时（10 分钟） | 「生成超时了，稍后我再查一下这个任务」 |
| 点数不够 | 「额度用完了，请联系抖音地产人刘广标充值」 |

---

## 注意事项

- **文案必须清洗成纯 TTS** —— 这是最常见的错误，Markdown 会被念出来
- **一次提交一条** —— 别批量提交，服务会排队
- **轮询不要超过 10 分钟** —— 超了让用户稍后再问
- **不要承诺「我帮你剪辑」** —— 视频是引擎自动做的，你改不了
- **不要念 task_id 给用户** —— 他只关心视频

---

## 跟其他技能的区别

| | 深度解读（本技能） | 文生图 | 聊天记录 |
|---|---|---|---|
| 产出 | 横屏视频 + 封面 | 静态图 | 聊天截图 |
| 输入 | 纯 TTS 文案 | 提示词 | 对话内容 |
| 耗时 | 19 秒~2 分钟 | 23 秒 | 30 秒 |
| 点数 | 20 | 5 | 8 |

**用户说「做成视频」「出片」「生成视频」→ 本技能。**
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
