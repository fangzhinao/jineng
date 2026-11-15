---
name: 今日爆款推荐
description: "推荐抖音房产爆款视频。用户说「今日爆款」「推荐爆款」「今天有什么火的」「找对标内容」时使用。每次发最新入库的爆款，含视频原链接 + 文案全文 + 互动数据。结尾自动问「要不要低粉爆款」。"
metadata:
  {
    "openclaw": { "emoji": "🔥", "requires": { "bins": ["python3"] } }
  }
---

# 今日爆款推荐

从**房瓜瓜爆款数据库**（1.3 万条持续更新的房产爆款）里拿最新的，发给用户做对标。

## 触发场景

- 「获取抖音今日爆款」「推荐今日爆款」
- 「今天有什么火的」
- 「有什么值得看的探盘」
- 「给我几条对标内容」
- 「找爆款视频」

---

## 前置：配置密钥

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

---

## 数据源说明（读给自己看）

```
表 bksjk  ← 爆款库（13254 条，持续更新）
表 tpsjk  ← 探盘库

字段：
  nickname        博主昵称
  follower_count  粉丝数
  digg_count      点赞
  comment_count   评论
  collect_count   收藏
  share_count     转发
  title           标题
  content         文案全文（列表只给 150 字预览，要全文查详情）
  video_type      类型（获客型 / 人设型 / 流量型）
  publish_time    发布时间
  aweme_id        视频 ID
  share_link      视频原链接   ← 直接给用户
  cover_url       封面
  city / tags
```

---

## 执行流程

### 第一步：拿今日爆款

```python
import json, pathlib, sys, urllib.request, urllib.parse

CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
KEY = json.loads(CFG.read_text(encoding="utf-8"))["key"]
GW = "https://www.fangzhinao.com/v1"

def list_hot(table, days=1, page_size=10, **extra):
    """table: 'bksjk'（口播）或 'tpsjk'（探盘）"""
    q = {"page_size": page_size, "sort": "time_desc",
         "quick_date": days, "is_enabled": 1}
    q.update(extra)
    url = GW + "/bksjk/api/sjk/" + table + "?" + urllib.parse.urlencode(q)
    req = urllib.request.Request(url, headers={"x-api-key": KEY})
    with urllib.request.urlopen(req, timeout=45) as r:
        d = json.loads(r.read().decode("utf-8", "replace"))
    d["_table"] = table
    return d

def fetch_both(days=1, n_each=6, **extra):
    """两个表都拉，合并后按发布时间倒序"""
    rows = []
    for tbl in ("bksjk", "tpsjk"):
        d = list_hot(tbl, days=days, page_size=n_each, **extra)
        for r in (d.get("rows") or []):
            r["_table"] = tbl
            rows.append(r)
    rows.sort(key=lambda x: str(x.get("publish_time") or ""), reverse=True)
    return rows

rows = fetch_both(days=1, n_each=6)

# ⚠️ 今日可能是 0 条（爬虫还没跑），自动退到近 7 天
if not rows:
    rows = fetch_both(days=7, n_each=6)
    fallback = True
else:
    fallback = False
```

### 第二步：拿每条文案全文

**列表只给 150 字预览，要给用户完整文案必须查详情：**

```python
def detail(item_id):
    url = GW + "/bksjk/api/sjk/bksjk/" + str(item_id)
    req = urllib.request.Request(url, headers={"x-api-key": KEY})
    with urllib.request.urlopen(req, timeout=30) as r:
        return (json.loads(r.read().decode("utf-8", "replace")) or {}).get("row") or {}
```

**默认只查前 3 条的全文**（每条 1 点，查 10 条就是 10 点，太贵）：
```
用户没说要几条 → 推荐 5 条，前 3 条给全文，后 2 条给标题+链接
用户说「要 10 条」→ 10 条都只给标题+链接，问他要看哪几条全文
```

### 第三步：输出

```
🔥 今日爆款推荐（{日期}）

⚠️ 今天还没有新数据入库，这是最近 7 天的。（有 fallback 时才写这句）

━━━━━━━━━━━━━━━━━━━━━━
1️⃣ 【口播】{昵称} · 粉丝 {follower_count}
   赞 {digg}｜评 {comment}｜藏 {collect}｜转 {share} · {video_type}
   📅 {publish_time 的日期}

   📹 视频：{share_link}

   📝 文案：
   {content 全文}

━━━━━━━━━━━━━━━━━━━━━━
2️⃣ 【探盘】{昵称} · 粉丝 {follower_count}
   赞 {digg}｜评 {comment}｜藏 {collect}｜转 {share}
   📅 {publish_time 的日期}

   📹 视频：{share_link}
   ...
```

**标签规则：**
```
_table == "bksjk" → 【口播】+ video_type（获客型/人设型/流量型）
_table == "tpsjk" → 【探盘】（探盘库没有 video_type）
```

**排序：两个表合并后按 `publish_time` 倒序，让两类自然穿插，不要一堆口播接一堆探盘。**

**如果是只给标题的（省点数）：**
```
4️⃣ {昵称} · 粉丝 {f} · 赞 {digg}
   《{title 前 40 字}》
   📹 {share_link}
```

### 第四步：结尾必须问低粉爆款

**每次输出完，固定加这一句：**

```
━━━━━━━━━━━━━━━━━━━━━━

要不要把**低粉爆款**也发给你？
（粉丝 5000 以下、点赞 1000 以上的，这种更容易复制）
```

### 第五步：低粉爆款（用户说「要」）

```python
rows = fetch_both(days=30, n_each=8,
                  follower_max=5000,   # 粉丝 ≤ 5000
                  digg_min=1000)       # 点赞 > 1000
# 同样两个表都拉，合并排序
```

**输出格式同样，但多一行「为什么值得看」：**

```
🔥 低粉爆款（粉丝 ≤5000、赞 >1000）

1️⃣ 粉丝 1995 · 赞 18528 · 涨粉比 9.3 倍
   {昵称} · 2026-11-05 · 流量型
   📹 {share_link}
   📝 {content 全文}

   💡 为什么值得看：粉丝不到 2000，这条赞是粉丝数的 9 倍，
      说明内容本身能破圈，不是靠粉丝基数
```

**「涨粉比」= 点赞 / 粉丝，这个数字最能说明「内容能不能破圈」。**

---

## 完整可跑脚本

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json, pathlib, sys, urllib.request, urllib.parse, datetime

CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
if not CFG.exists():
    sys.exit("❌ 未配置密钥")
KEY = json.loads(CFG.read_text(encoding="utf-8"))["key"]
GW = "https://www.fangzhinao.com/v1"

def api(path, params=None):
    url = GW + path + (("?" + urllib.parse.urlencode(params)) if params else "")
    req = urllib.request.Request(url, headers={"x-api-key": KEY})
    with urllib.request.urlopen(req, timeout=45) as r:
        return json.loads(r.read().decode("utf-8", "replace"))

def hot(days=1, n=10, **extra):
    p = {"page_size": n, "sort": "time_desc", "quick_date": days, "is_enabled": 1}
    p.update(extra)
    return (api("/bksjk/api/sjk/bksjk", p).get("rows") or [])

mode = sys.argv[1] if len(sys.argv) > 1 else "today"
if mode == "low":
    rows = hot(days=30, n=10, follower_max=5000, digg_min=1000)
    print("低粉爆款：", len(rows), "条")
else:
    rows = hot(days=1, n=10)
    if not rows:
        rows = hot(days=7, n=10)
        print("⚠️ 今日无新数据，退到近 7 天")
    print("今日爆款：", len(rows), "条")

for i, r in enumerate(rows, 1):
    f = r.get("follower_count") or 0
    dg = r.get("digg_count") or 0
    ratio = round(dg / f, 1) if f else "—"
    print()
    print("%d. %s | 粉丝%s 赞%s 评%s | %s" % (
        i, str(r.get("nickname"))[:18], f, dg,
        r.get("comment_count") or 0, r.get("video_type") or ""))
    print("   %s" % str(r.get("title") or "")[:50])
    print("   %s" % str(r.get("share_link") or ""))
    print("   涨粉比 %s | %s" % (ratio, str(r.get("publish_time") or "")[:10]))
```

**用法：**
```bash
python3 hot.py           # 今日爆款
python3 hot.py low       # 低粉爆款
```

---

## 错误处理

| 情况 | 提示 |
|---|---|
| 没配密钥 | 「请先把密钥发给我」 |
| 今天 0 条 | 自动退到近 7 天，并说明「今天还没新数据，这是最近 7 天的」 |
| 近 7 天也 0 条 | 「库里最近没有新内容，可能是爬虫停了，我通知一下抖音地产人刘广标」 |
| 点数不够 | 「额度用完了，请联系抖音地产人刘广标充值」 |
| 某条详情查不到 | 跳过这条，用列表里的 title 顶替，不要中断 |

---

## 注意事项

- **不要编造爆款** —— 全部来自数据库真实数据
- **不要改标题** —— 原样给用户，他要的是对标
- **`share_link` 是 `iesdouyin.com` 的分享链接**，能直接打开看
- 列表的 `content_preview` 只有 150 字，**要全文必须查 `/api/sjk/bksjk/{id}`**
- **「今日」可能真的没有** —— 你的爬虫不是每天都跑，别当成 bug
- 一次别发太多，5 条刚好；用户要更多再翻页
- **低粉爆款的「涨粉比」是这个技能最有价值的数字**，一定要算给用户看

---

## 两种爆款的区别（跟用户解释时用）

| | 今日爆款 | 低粉爆款 |
|---|---|---|
| 筛选 | 最新入库 | 粉丝 ≤5000 且 赞 >1000 |
| 适合谁 | 想看大盘在火什么 | 想找「我能复制」的 |
| 为什么 | 热度即时 | 粉丝少还能爆 = 内容本身强 |
| 点数 | 1 点/次 + 每条详情 1 点 | 同左 |

**跟用户说人话：**
```
今日爆款是「今天大家都在看什么」
低粉爆款是「跟我一样粉丝不多的人，怎么就火了」—— 后者更能学
```

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
