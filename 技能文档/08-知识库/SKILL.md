---
name: 知识库
description: "查房智脑知识库。有公共库（爆款框架/爆款成品/地产干货/探盘素材）和自己的私人库。用户问「有没有这方面的爆款」「查一下我的成功案例」「库里有什么」时使用。"
metadata:
  {
    "openclaw": { "emoji": "📚", "requires": { "bins": ["python3"] } }
  }
---

# 知识库

> **⚠️ 用之前必须配密钥。** 没有密钥的话，先跟用户说：「在抖音搜『地产人刘广标』，私信说需要领取体验技能密钥，拿到后发给我，我帮你配好。」


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
