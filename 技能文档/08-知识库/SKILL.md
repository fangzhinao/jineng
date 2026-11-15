---
name: 知识库
description: "管理和使用你自己的知识库。用户上传自己的楼盘资料、房源清单、成功案例、行业干货后，AI 写文案时就能引用真实数据，不会瞎编。用户说「把这些资料存进知识库」「查一下我之前上传的」「库里有什么」「我的楼盘数据」时使用。"
metadata:
  {
    "openclaw": { "emoji": "📚", "requires": { "bins": ["python3"] } }
  }
---

# 知识库

> **⚠️ 用之前必须配密钥。** 没有密钥的话，先跟用户说：「在抖音搜『地产人刘广标』，私信说需要领取体验技能密钥，拿到后发给我，我帮你配好。」

## 核心目的：让你的库里有真东西

**AI 写文案会不会瞎编，取决于你的库里有没有真实数据。**

```
你的库里有什么      →  AI 能写什么
────────────────────────────────────
什么都没有          →  只能写空话，「XX 板块值得关注」这种废话
有楼盘数据          →  能写「金域华府 89 平三房，均价 8500，首付 23 万」
有成交案例          →  能写「上个月我帮一个客户 87 万拿下 XX 小区」
有行业干货          →  能写「公积金贷款额度怎么算」讲得清楚
────────────────────────────────────
库越全，写出来的东西越像你，越不敢编
```

**所以第一次用的时候，主动问用户：**
```
「为了让写出来的文案有真实数据、不瞎编，建议先做三件事：

 ① 把手上有的楼盘资料、房源清单发给我，我帮你整理进知识库
 ② 你觉得自己讲得好的干货、金句，也存进去
 ③ 你觉得哪个同行讲得好，把他账号发我，我帮你提取精选内容

 这些都会成为你以后写文案的素材底子。」
```

**用户说「我没什么资料」时：**
```
「那先发我一两个你熟悉的楼盘，或者你上次讲得比较满意的一条视频，
 我帮你整理进库，以后写文案就有依据了。」
```


**两个层次的库：**

```
公共库（抖音地产人刘广标维护，所有学生共用）
  · framework   陪跑的文案框架    576 篇
  · copy        房产爆款对标素材   4331 篇
  · knowledge   地产人需要的干货   1171 篇
  · tanpan      探盘对标文案素材   72 篇

私人库（每个学生自己的）
  · mine        你自己的知识（成功案例、话术、本地经验）
```

## 触发场景

- 「有没有关于 XX 的爆款框架」
- 「查一下库里有没有 XX」
- 「我的知识库里有什么」
- 「把这段存进知识库」
- 写文案时先查框架

---

## 前置：配置密钥

**如果 `~/.fangguagua/config.json` 不存在**：

```
「你还没有配置密钥。

在抖音搜「地产人刘广标」，私信说需要领取体验技能密钥就行。

拿到后是一串 fg_ 开头的内容，发给我，我帮你配好。」
```

---

## 接口一览

| 用途 | 方法 | 路径 |
|---|---|---|
| 检索公共库 | POST | `/v1/kb/{framework\|copy\|knowledge\|tanpan}` |
| 检索私人库 | POST | `/v1/kb/mine` |
| 上传到私人库 | POST | `/v1/kb/mine/upload` |
| 看私人库列表 | GET | `/v1/kb/mine/list` |
| 删私人库文档 | DELETE | `/v1/kb/mine/doc/{doc_id}` |

**检索 0.5 点/次，上传 1 点/次，列表和删除免费。**

---

## 完整流程：从同行账号提取内容进知识库

**用户说「这个账号讲得好，帮我弄进知识库」时，照这个流程走。别只说「好的」，要真的做。**

### 第一步：确认用户给的是什么

```
用户发的可能是：
· 账号主页链接  douyin.com/user/MS4wLjABAAAA...
· 分享短链      v.douyin.com/xxxxx/
· 只说一个名字  「XX说房」← 这个拿不到，要让用户发链接

如果是名字 → 回复：
「你能把他的主页链接发我吗？或者分享他的任意一条视频给我，
 我从视频反推他的账号。」
```

### 第二步：账号链接 → 作品列表

```python
import json, pathlib, re, time, urllib.request, urllib.parse

CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
KEY = json.loads(CFG.read_text(encoding="utf-8"))["key"]
GW = "https://www.fangzhinao.com/v1"
UA = "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15"

def get(path):
    req = urllib.request.Request(GW + path, headers={"x-api-key": KEY})
    with urllib.request.urlopen(req, timeout=90) as r:
        return json.loads(r.read().decode("utf-8", "replace"))

def post(path, body):
    req = urllib.request.Request(GW + path,
        data=json.dumps(body, ensure_ascii=False).encode("utf-8"),
        headers={"x-api-key": KEY, "Content-Type": "application/json"},
        method="POST")
    with urllib.request.urlopen(req, timeout=90) as r:
        return json.loads(r.read().decode("utf-8", "replace"))

def resolve(url):
    """短链展开 + 提取 sec_user_id 或 aweme_id"""
    if "v.douyin.com" in url:
        req = urllib.request.Request(url, method="HEAD", headers={"User-Agent": UA})
        try:
            with urllib.request.urlopen(req, timeout=20) as r:
                url = r.headers.get("Location") or r.url or url
        except urllib.error.HTTPError as e:
            url = e.headers.get("Location") or url
    m = re.search(r"/share/user/([A-Za-z0-9_\-]+)", url) or \
        re.search(r"/user/([A-Za-z0-9_\-]{20,})", url)
    if m: return "user", m.group(1)
    m = re.search(r"/share/video/(\d+)", url) or re.search(r"/video/(\d+)", url)
    if m: return "video", m.group(1)
    raise RuntimeError("解析不出 ID，让用户发主页链接")

def list_posts(sec_uid, pages=2):
    """拉作品列表"""
    items, cursor, page = [], None, 0
    while page < pages:
        q = "/posts/api/v1/douyin/user/posts?sec_user_id=%s&count=20&wait=28&lang=zh" % sec_uid
        if cursor:
            q += "&cursor=" + urllib.parse.quote(str(cursor))
        data = (get(q)).get("data") or {}
        items += data.get("items") or []
        cursor = data.get("cursor"); page += 1
        if not data.get("has_more") or not cursor:
            break
        time.sleep(2)
    return items

def extract_caption(video_url, timeout_sec=180):
    """提取单条视频文案（提交任务 + 轮询）"""
    d = post("/caption/api/caption/extract", {"url": video_url})
    if not d.get("success"):
        return None, (d.get("error") or {}).get("code", "UNKNOWN")
    tid = (d.get("data") or {}).get("task_id")
    if not tid:
        return None, "NO_TASK_ID"

    t0 = time.time()
    while time.time() - t0 < timeout_sec:
        time.sleep(5)
        try:
            r = get("/caption/api/task/" + tid)
        except Exception:
            continue
        data = r.get("data") or {}
        st = (data.get("status") or "").lower()
        if st in ("done", "success", "completed"):
            return data.get("caption") or "", None
        if st in ("failed", "error"):
            return None, "TASK_FAILED"
    return None, "TIMEOUT"

def upload_kb(text, name):
    """上传到私人知识库"""
    d = post("/kb/mine/upload", {"name": name, "text": text})
    return d
```

### 第三步：选作品 + 提取 + 上传

```python
def harvest_to_kb(account_url, pick=10, max_chars=8000):
    """从账号抓 N 条作品，提取文案，存进知识库"""
    kind, ident = resolve(account_url)

    if kind == "video":
        # 用户给的是单条视频 → 只处理这一条
        posts = [{"content_id": ident, "title": "", "created_at": ""}]
    else:
        items = list_posts(ident, pages=2)
        # 按点赞排序，挑最好的
        items.sort(key=lambda x: ((x.get("stats") or {}).get("digg_count") or 0), reverse=True)
        posts = items[:pick]

    print("从 %d 条作品里挑 %d 条提取" % (len(posts), len(posts)))
    ok, fail = 0, 0

    for i, it in enumerate(posts, 1):
        vid = it.get("content_id")
        title = (it.get("title") or "")[:30]
        if not vid:
            continue
        url = "https://www.douyin.com/video/" + str(vid)
        print("[%d/%d] %s ..." % (i, len(posts), title or vid), end="", flush=True)

        caption, err = extract_caption(url)
        if not caption:
            print(" ❌ %s" % err); fail += 1
            continue
        if len(caption) < 100:
            print(" ⚠️ 太短跳过"); continue

        body = "作品：%s\n链接：%s\n\n%s" % (title or "(无标题)", url, caption[:max_chars])
        r = upload_kb(body, "同行精选-%s" % (title or vid))
        if r.get("success") or r.get("ok"):
            print(" ✅ 已入库（%d 字）" % len(caption))
            ok += 1
        else:
            print(" ❌ 上传失败")
            fail += 1
        time.sleep(3)   # 防风控

    print()
    print("完成：成功 %d 条，失败 %d 条" % (ok, fail))
    return ok, fail
```

### 第四步：告诉用户结果

```
「从他账号挑了 10 条点赞最高的，提取了 8 条，都存进你的知识库了。

下次写文案时我会参考这些内容，但只借他的表达方式和选题角度，
不会用他的数据（因为那些是他城市的）。

你自己的楼盘数据还是要另外发我，那才是你的核心资产。」
```

---

### ⚠️ 三条注意事项

```
① 一次最多抓 10 条，多了点数消耗大、也更容易被限流
② 每条之间等 3 秒，评论和文案接口都容易被风控
③ 只借「表达方式和选题角度」，不借事实数据
   他的城市数据对用户没用，硬套反而误导
```

### 如果用户说「我自己的账号也想存」

```
流程一样，只是把账号链接换成用户自己的。
这时候可以抓更多（20 条），因为是自己账号，不怕限流。
```

---

## 检索

### 通用函数

```python
import json, pathlib, urllib.request

CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
KEY = json.loads(CFG.read_text(encoding="utf-8"))["key"]
GW = "https://www.fangzhinao.com/v1"

def kb(scope, query, top_k=6):
    """scope: framework / copy / knowledge / tanpan / mine"""
    body = json.dumps({
        "query": query,
        "retrieval_model": {
            "search_method": "semantic_search",
            "top_k": top_k,
            "score_threshold_enabled": False,
            "reranking_enable": False,
        }
    }, ensure_ascii=False).encode("utf-8")
    req = urllib.request.Request(
        GW + "/kb/" + scope, data=body,
        headers={"Content-Type": "application/json", "x-api-key": KEY},
        method="POST")
    with urllib.request.urlopen(req, timeout=45) as r:
        return json.loads(r.read().decode("utf-8", "replace"))
```

### 返回结构

```json
{
  "records": [
    {
      "score": 0.637,
      "segment": {
        "document": {"name": "文档名"},
        "content": "命中的内容"
      }
    }
  ]
}
```

---

## 五个库各查什么

### `framework` —— 陪跑的文案框架

**查什么：** 结构框架，带 `【城市】【楼盘】` 占位符。

```
kb("framework", "开场白 人设")
kb("framework", "房源盘点")
kb("framework", "避坑 砍价")
kb("framework", "客户异议")
```

**返回的样子：**
```
【文案框架】自我介绍
【类型】人设型

我回【谁】
土生土长的【城市】几零后
...
```

### `copy` —— 房产爆款对标素材（成品）

**查什么：** 别人已经发过的完整文案。

```
kb("copy", "呼市 楼盘盘点")
kb("copy", "客户嫌贵")
```

**⚠️ 这个库有 4331 篇，内容质量参差，要挑 score > 0.6 的。**

### `knowledge` —— 地产人需要的干货

**查什么：** 行业知识、政策解读、方法论。

```
kb("knowledge", "公积金政策")
kb("knowledge", "学区房怎么讲")
kb("knowledge", "首次置业注意什么")
```

### `tanpan` —— 探盘对标文案

**查什么：** 探盘类视频的文案结构。

```
kb("tanpan", "楼盘测评结构")
kb("tanpan", "户型怎么讲")
```

### `mine` —— 学生自己的库

**查什么：** 他自己的成功案例、话术、本地经验。

```
kb("mine", "我的成功案例")
kb("mine", "郴州本地经验")
```

**⚠️ 如果学生还没被分配私人库，会返回 403：**
```json
{"ok": false, "error": "你还没有「mine」知识库，请联系抖音地产人刘广标开通"}
```

**这时要告诉用户：**
```
「你的私人知识库还没开通。要不要先用公共库？
 想开通的话找抖音地产人刘广标。」
```

---

## 上传到私人库

```python
def kb_upload(name, text):
    body = json.dumps({"name": name, "text": text},
                      ensure_ascii=False).encode("utf-8")
    req = urllib.request.Request(
        GW + "/kb/mine/upload", data=body,
        headers={"Content-Type": "application/json", "x-api-key": KEY},
        method="POST")
    with urllib.request.urlopen(req, timeout=120) as r:
        return json.loads(r.read().decode("utf-8", "replace"))
```

**调用后：**
```json
{"document": {"id": "c3aa5175-...", "indexing_status": "splitting"}}
```

**状态变化：**
```
splitting → parsing → cleaning → completed
```
**通常 15~30 秒。完成后才能检索到。**

**告诉用户：**
```
「已经存进去了，正在建立索引，大概 30 秒后就能查到。
 你可以继续问别的，等一下再问我就能搜到。」
```

### 用户发来文件怎么办（Excel / Word / 截图）

**上传接口只接受文本，不能直接传文件。所以要先把文件读成文字。**

#### 能直接读的格式

```python
# Excel → 转成 Markdown 表格文本
import pandas as pd
def read_excel(path):
    xls = pd.ExcelFile(path)
    out = []
    for sheet in xls.sheet_names:
        df = pd.read_excel(xls, sheet_name=sheet)
        out.append("## " + sheet)
        out.append(df.to_markdown(index=False))
    return "\n\n".join(out)

# Word
from docx import Document
def read_docx(path):
    d = Document(path)
    return "\n".join(p.text for p in d.paragraphs if p.text.strip())

# PDF
import pdfplumber
def read_pdf(path):
    with pdfplumber.open(path) as pdf:
        return "\n".join((pg.extract_text() or "") for pg in pdf.pages)
```

#### 读不了的（图片、微信截图）

**按这个顺序处理：**

```
① 先看 AI 有没有视觉能力（能看图）
   有 → 直接读图里的文字
   没有 → 往下

② 让用户复制文字发过来
   「截图我这边看不清字，你直接把文字复制给我行吗？
    或者发 Excel / Word 文件也行。」

③ 如果能装 OCR
   pip install rapidocr-onnxruntime
```

#### 读完之后怎么切分上传

```python
def chunk_text(text, max_len=4500):
    """按段落切，每篇不超过 4500 字"""
    paras = [p.strip() for p in text.split("\n") if p.strip()]
    chunks, cur = [], ""
    for p in paras:
        if len(cur) + len(p) + 1 > max_len:
            if cur: chunks.append(cur)
            cur = p
        else:
            cur = cur + "\n" + p if cur else p
    if cur: chunks.append(cur)
    return chunks

def upload_file_to_kb(path, base_name):
    """读文件 → 切分 → 逐篇上传"""
    ext = os.path.splitext(path)[1].lower()
    if ext in ('.xlsx', '.xls'):
        text = read_excel(path)
    elif ext == '.docx':
        text = read_docx(path)
    elif ext == '.pdf':
        text = read_pdf(path)
    elif ext in ('.txt', '.md', '.csv'):
        text = open(path, encoding='utf-8', errors='replace').read()
    else:
        return None, "不支持的格式"

    parts = chunk_text(text)
    print("文件共 %d 字，切成 %d 篇" % (len(text), len(parts)))

    ok = 0
    for i, part in enumerate(parts, 1):
        name = "%s-%d" % (base_name, i) if len(parts) > 1 else base_name
        r = kb_upload(name, part)
        if r.get("document"):
            print("  [%d/%d] %s  ✅" % (i, len(parts), name))
            ok += 1
        else:
            print("  [%d/%d] %s  ❌" % (i, len(parts), name))
        time.sleep(2)
    return ok, len(parts)
```

#### 告诉用户结果

```
「你这个 Excel 一共 3 个 sheet，我读了 240 条楼盘数据，
 切成 5 篇存进你的知识库了。

 大概 30 秒后就能查到。你以后问我『XX 小区多少平多少钱』，
 我就能直接答，不用猜。」
```

---

### 什么时候该上传

**用户说了这些，主动问「要不要存进你的知识库」：**

```
· 「我以前这样做效果很好」
· 「我们这儿有个特殊情况」
· 「这个楼盘我熟」
· 用户发来自己的文章 / 话术 / 案例
```

**不要主动帮用户存一堆东西，先问。**

---

## 看自己的库

```python
def kb_list(page=1, limit=20):
    req = urllib.request.Request(
        GW + f"/kb/mine/list?page={page}&limit={limit}",
        headers={"x-api-key": KEY})
    with urllib.request.urlopen(req, timeout=30) as r:
        return json.loads(r.read().decode("utf-8", "replace"))
```

**输出：**
```
📚 你的知识库里有 3 篇：

1. 我的开场白套路          102 字  ✅ 可检索
2. 郴州北湖区楼盘清单      1,240 字 ✅ 可检索
3. 客户常见异议应对        860 字   ⏳ 索引中

要删哪篇告诉我编号，或者说「清空」。
```

---

## 删文档

```python
def kb_delete(doc_id):
    req = urllib.request.Request(
        GW + "/kb/mine/doc/" + doc_id,
        headers={"x-api-key": KEY}, method="DELETE")
    with urllib.request.urlopen(req, timeout=30) as r:
        return r.status
```

**删之前必须确认：**
```
「要删掉「我的开场白套路」吗？删了就找不回来了。
 回复「确认删除」我再动手。」
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

def kb(scope, query, top_k=6):
    body = json.dumps({"query": query, "retrieval_model": {
        "search_method": "semantic_search", "top_k": top_k,
        "score_threshold_enabled": False, "reranking_enable": False}},
        ensure_ascii=False).encode("utf-8")
    req = urllib.request.Request(GW + "/kb/" + scope, data=body,
        headers={"Content-Type": "application/json", "x-api-key": KEY}, method="POST")
    with urllib.request.urlopen(req, timeout=45) as r:
        return json.loads(r.read().decode("utf-8", "replace"))

scope = sys.argv[1] if len(sys.argv) > 1 else "framework"
query = sys.argv[2] if len(sys.argv) > 2 else "开场白"

d = kb(scope, query)
recs = d.get("records") or []
print("[%s] 「%s」→ 命中 %d 条" % (scope, query, len(recs)))
print()
for i, r in enumerate(recs, 1):
    seg = r.get("segment") or {}
    name = (seg.get("document") or {}).get("name") or ""
    print("%d. %.0f%%  %s" % (i, (r.get("score") or 0) * 100, name))
    print("   " + (seg.get("content") or "")[:160].replace("\n", " "))
    print()
```

**用法：**
```bash
python3 kb.py framework "开场白人设"
python3 kb.py copy "呼市楼盘盘点"
python3 kb.py knowledge "公积金"
python3 kb.py mine "我的成功案例"
```

---

## 错误处理

| 情况 | 提示 |
|---|---|
| 没配密钥 | 「请先把密钥发给我」 |
| 403 没有 mine 库 | 「你的私人知识库还没开通，找抖音地产人刘广标开通；先用公共库可以吗？」 |
| 检索 0 条 | 「这个方向库里没有，换个说法？比如『房源』『避坑』『人设』」 |
| 上传超时 | 「上传有点慢，再试一次？」 |
| 上传后查不到 | 「还在建索引，30 秒后再问我」 |
| 点数不够 | 「额度用完了，请联系抖音地产人刘广标充值」 |

---

## 跟本地数据库的分工

**这两个容易混，一定要分清：**

| | 知识库（本技能） | 本地数据库（另一个技能） |
|---|---|---|
| 存什么 | **文案结构 + 别人的爆款 + 通用知识** | **用户自己的项目真实数据** |
| 在哪 | 房智脑服务器（Dify） | 用户电脑（SQLite） |
| 谁维护 | 抖音地产人刘广标 + 用户上传 | 用户自己录入 |
| 写文案时提供 | **怎么说**（结构/句式） | **说什么**（价格/学区/优劣势） |
| 查询消耗 | 0.5 点/次 | 0 点（本地） |

**好文案 = 知识库的结构 + 本地库的事实。**

---

## 注意事项

- **检索结果不要一次性全倒给用户** —— 先说「找到 6 条，第 1 条匹配度 67%」，让他选
- **`copy` 库 4331 篇质量参差** —— 只看 score > 0.6 的
- **`mine` 库要用户自己传才有内容** —— 新开通的是空的，不要惊讶
- **上传后不能立即检索** —— 要等索引完成（15~30 秒）
- **删除不可恢复** —— 一定要二次确认
- **不要一次上传超长文本** —— 单篇建议 < 5000 字，太长了切分质量差

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
