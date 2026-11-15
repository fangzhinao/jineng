---
name: 爆款文案框架
description: "从陪跑文案框架库里检索可套用的爆款结构，填入本地城市/楼盘/人名后输出成品文案。用户要写抖音口播、小红书、朋友圈文案，或说「给我一条文案」「写个开场白」「找个人设文案」「房源怎么介绍」时使用。输出的是结构框架，不是现成成品。"
metadata:
  {
    "openclaw": { "emoji": "📋", "requires": { "bins": ["python3"] } }
  }
---

# 爆款文案框架

从**陪跑文案框架库**（576 个已验证的爆款结构）里，挑最匹配的一个，把 `【城市】【楼盘】【名字】` 这些占位符换成真实的，输出可以直接念的成品。

## 触发场景

- 「给我写条文案」「写个开场白」
- 「房源怎么介绍」「这个人设怎么写」
- 「帮我找个框架」「有没有现成的模板」
- 「我要拍关于 XX 的视频，文案帮我写一下」

---

## 前置一：配置密钥

**如果 `~/.fangguagua/config.json` 不存在**：

```
「你还没有配置密钥。

在抖音搜「地产人刘广标」，私信说需要领取体验技能密钥就行。

拿到后是一串 fg_ 开头的内容，发给我，我帮你配好。」
```

用户发来后：

```python
import json, pathlib
p = pathlib.Path.home() / ".fangguagua"
p.mkdir(parents=True, exist_ok=True)
(p / "config.json").write_text(json.dumps({"key": "用户给的密钥"}), encoding="utf-8")
print("✅ 密钥已配置")
```

## 前置二：城市 / 名字（一次性记住）

**如果配置文件里没有 `city` 或 `name`，第一次写文案时问一次，然后存进去：**

```python
cfg = json.loads(CFG.read_text(encoding="utf-8"))
if not cfg.get("city"):
    # 问用户：你在哪个城市？怎么称呼你？
    cfg["city"] = "郴州"        # ← 用户回答
    cfg["name"] = "刘广标"      # ← 用户回答
    CFG.write_text(json.dumps(cfg, ensure_ascii=False), encoding="utf-8")
```

**以后写文案直接用 `cfg["city"]` / `cfg["name"]`，不再问。**

---

## 执行流程

### 第一步：判断要什么类型

| 用户说法 | 检索词 | 类型 |
|---|---|---|
| 「写个人设」「自我介绍」「讲讲我自己」 | `人设 自我介绍 坚持 良心` | 人设型 |
| 「房源怎么介绍」「盘点楼盘」「推荐房子」 | `房源 盘点 楼盘 性价比` | 获客型 |
| 「客户嫌贵怎么办」「怎么避坑」 | `避坑 套路 砍价 买房注意` | 获客型 |
| 「讲政策」「学区」「首付」 | `政策 学区 首付 公积金` | 获客型 |
| 「客户不回复」「怎么跟进」 | `沟通 跟进 信任 成交` | 获客型 |
| 用户没说清 | 直接检索用户原话 | — |

### 第二步：检索框架（1 次调用，0.5 点）

```python
DS = "42a4bcea-9074-4012-b661-c6a6f152a049"   # 陪跑的文案框架

def retrieve(query, top_k=6):
    body = json.dumps({
        "query": query,
        "retrieval_model": {
            "search_method": "semantic_search",
            "top_k": top_k,
            "score_threshold_enabled": False,
            "reranking_enable": False,
        }
    }).encode("utf-8")
    req = urllib.request.Request(
        GW + "/kb/datasets/" + DS + "/retrieve",
        data=body,
        headers={"x-api-key": KEY, "Content-Type": "application/json"},
        method="POST",
    )
    with urllib.request.urlopen(req, timeout=45) as r:
        return json.loads(r.read().decode("utf-8", "replace"))
```

**返回结构：**
```json
{
  "records": [
    {
      "score": 0.45,
      "segment": {
        "document": {"name": "自我介绍"},
        "content": "【文案框架】自我介绍\n【类型】人设型\n\n我回【谁】\n土生土长的【城市】几零后\n…\n背景音乐：瞬间的永恒"
      }
    }
  ]
}
```

### 第三步：给用户看框架列表，让他选

**不要直接把一堆框架全倒出来。先列表格让他挑：**

```
📋 找到 6 个匹配的框架：

1. 【自我介绍】· 人设型 · 匹配度 45%
   适合：讲自己是谁、为什么做这行

2. 【你们知道XX月【城市】的新房有多便宜吗】· 获客型 · 42%
   适合：播报行情、吸引想买房的

3. 【我们【城市】中介自己都不买的5类房子】· 获客型 · 41%
   适合：讲避坑、建立专业感

…

告诉我编号，或者直接说「第 2 个」，我帮你填好。
```

### 第四步：填充占位符

**拿到用户选的框架 `content` 后，替换这些占位符：**

| 占位符 | 填什么 | 来源 |
|---|---|---|
| `【城市】` / `城市` | 城市名 | `cfg["city"]` |
| `【名字】` / `XXX`（结尾处） | 你的名字 | `cfg["name"]` |
| `【楼盘名字】` / `【XXX楼盘】` | 具体楼盘 | **问用户** |
| `【数字】` / `XX` / `X` | 具体数字 | **问用户**（或者用真实数据） |
| `【首付XX万】` | 首付金额 | **问用户** |
| `【月薪XXXX】` | 月薪档位 | **问用户** |
| `【谁】` | 打招呼对象 | 默认「大家好」或省略 |
| `【多少天】` / `【X】年` | 从业时间 | **问用户一次，存进 cfg** |

**规则：**
```
✅ 城市、名字 → 从配置里直接填，不问
✅ 从业年限 → 第一次问，存进 cfg["years"]，以后不问
⚠️ 楼盘、数字、首付 → 每个框架不同，必须问
❌ 不要留着占位符输出，全部要替换掉
❌ 不要自己编数字（比如随便写「卖了200套」）
```

**如果用户没给楼盘/数字，框架里又必须要：**

```
这个框架需要填「楼盘名字」和「10 月成交套数」。
你告诉我具体是哪个盘、多少套，我就帮你填。
或者我换一个不需要具体数字的框架？
```

### 第五步：输出成品

```
📋 框架：自我介绍（人设型）

---
我回【替换后的正文，完整输出】

---
背景音乐建议：瞬间的永恒

💡 提示：
· 这条 1 分 40 秒左右，适合放视频开头做自我介绍
· 【城市】已经全部换成「郴州」
· 结尾的「我是郴州地产刘广标，买房卖房就找刘广标」是你的钩子，念的时候慢一点
```

---

## 完整可跑脚本

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""检索爆款文案框架"""
import json, pathlib, sys, urllib.request

CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
if not CFG.exists():
    sys.exit("❌ 未配置密钥")
cfg = json.loads(CFG.read_text(encoding="utf-8"))
KEY = cfg["key"]
GW = "https://www.fangzhinao.com/v1"
DS = "42a4bcea-9074-4012-b661-c6a6f152a049"

def retrieve(query, top_k=6):
    body = json.dumps({
        "query": query,
        "retrieval_model": {"search_method": "semantic_search", "top_k": top_k,
                            "score_threshold_enabled": False, "reranking_enable": False}
    }).encode("utf-8")
    req = urllib.request.Request(GW + "/kb/datasets/" + DS + "/retrieve",
        data=body, headers={"x-api-key": KEY, "Content-Type": "application/json"}, method="POST")
    with urllib.request.urlopen(req, timeout=45) as r:
        return json.loads(r.read().decode("utf-8", "replace"))

query = sys.argv[1] if len(sys.argv) > 1 else "开场白"
d = retrieve(query, 6)
recs = d.get("records") or []
print("检索:", query, "→ 命中", len(recs))
print()

for i, r in enumerate(recs, 1):
    seg = r.get("segment") or {}
    name = (seg.get("document") or {}).get("name") or "(无标题)"
    content = seg.get("content") or ""
    typ = ""
    for line in content.split("\n")[:4]:
        if "【类型】" in line:
            typ = line.replace("【类型】", "").strip()
    print("%d. 【%s】· %s · %.0f%%" % (i, name[:36], typ, (r.get("score") or 0) * 100))
    print("   全文 %d 字" % len(content))
print()
print("想看哪一条的全文？告诉我编号。")
```

**用法：**
```bash
python3 framework.py "人设 自我介绍"
python3 framework.py "房源盘点"
python3 framework.py "避坑 砍价"
```

---

## 错误处理

| 情况 | 提示 |
|---|---|
| 没配密钥 | 「请先把密钥发给我」 |
| 检索 0 条 | 「这个方向库里没有框架，换个说法试试？比如『房源』『避坑』『人设』」 |
| 点数不够 | 「额度用完了，请联系抖音地产人刘广标充值」 |
| 超时 | 「知识库有点慢，再试一次」 |
| 用户没给楼盘 | 「这个框架要具体楼盘名，你告诉我是哪个盘？」 |

---

## 注意事项

- **这是「结构框架」，不是「现成成品」** —— 必须填占位符才能用
- 库里有 **576 个框架**，别一次全倒出来，先给 6 个让他选
- **不同类型不能混用**：`获客型` 讲房源/避坑，`人设型` 讲故事/立人设
- 框架名本身通常就是**标题**，可以直接用
- 有的框架带 `背景音乐：XXX`，输出时一起带上
- **数字绝不能编** —— 必须问用户真实数据，编了会被平台判定虚假宣传

---

## 跟「爆款成品改写」的区别

| | 爆款文案框架（本技能） | 爆款成品改写 |
|---|---|---|
| 检索的数据集 | `陪跑的文案框架`（576 个结构） | `房产爆款对标素材文案`（4331 条成品） |
| 拿到什么 | 带 `【城市】` 占位符的**结构** | 别人已经发过的**完整文案** |
| 怎么用 | 填占位符 | AI 换城市/楼盘改写 |
| 优点 | 结构是验证过的，稳 | 语言更自然，快 |
| 缺点 | 要填的东西多 | 容易撞车 |

**用户要「结构稳」选本技能，要「快」选成品改写。**

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
