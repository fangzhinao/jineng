---
name: 抖音账号数据分析
description: "分析抖音账号数据。用户发来抖音主页链接、要看对标账号、分析粉丝/作品表现、找爆款规律、判断这个号值不值得学时使用。输出账号概况 + 作品表现表 + 数据分析。"
metadata:
  {
    "openclaw": { "emoji": "📊", "requires": { "bins": ["python3"] } }
  }
---

# 抖音账号数据分析

拿一个抖音账号的**主页信息 + 作品列表 + 每条作品的互动数据**，做出可读的分析报告。

## 触发场景

- 用户发来抖音主页链接
- 「分析一下这个号」
- 「这个号怎么样」
- 「看看对标的账号数据」
- 「他的爆款是哪几条」

---

## 前置：配置密钥

**如果 `~/.fangguagua/config.json` 不存在**，提示用户：

```
「你还没有配置密钥。

在抖音搜「地产人刘广标」，私信说需要领取体验技能密钥就行。

拿到后是一串 fg_ 开头的内容，发给我，我帮你配好。」
```

用户发来密钥后：

```python
import json, pathlib
p = pathlib.Path.home() / ".fangguagua"
p.mkdir(parents=True, exist_ok=True)
(p / "config.json").write_text(json.dumps({"key": "用户给的密钥"}), encoding="utf-8")
print("✅ 密钥已配置")
```

---

## 执行流程

### 第一步：短链解析（关键，不能省）

**dtk 只认 ID，不认 URL。所以必须先解析。**

```python
import urllib.request, re

def resolve(url):
    """短链 → (类型, ID)。返回 ("user", sec_user_id) 或 ("video", aweme_id)

    ⚠️ 关键：urlopen 默认会自动跟随 302，所以拿不到 Location 头。
       必须用 r.url（跟随后的最终地址），HTTPError 时再从 e.headers 取。
    """
    if "v.douyin.com" in url:
        req = urllib.request.Request(url, method="HEAD",
            headers={"User-Agent": UA})
        try:
            with urllib.request.urlopen(req, timeout=20) as r:
                url = r.headers.get("Location") or r.url or url
        except urllib.error.HTTPError as e:
            url = e.headers.get("Location") or url

    m = re.search(r"/share/user/([A-Za-z0-9_\-]+)", url) or \
        re.search(r"/user/([A-Za-z0-9_\-]{20,})", url)
    if m:
        return "user", m.group(1)
    m = re.search(r"/share/video/(\d+)", url) or re.search(r"/video/(\d+)", url)
    if m:
        return "video", m.group(1)
    raise RuntimeError("解析不出 ID，链接可能失效")
```

**如果用户给的是 `douyin.com/user/MS4wLjABAAAA...` 这种长链，直接从路径正则取，不用发请求。**

### 第二步：拿 sec_user_id

**两种情况：**

```python
kind, ident = resolve(user_input)

if kind == "video":
    # 给的是视频链接 → 先查视频拿到作者 ID
    d = get("/video/api/v1/douyin/video?aweme_id=%s&wait=28&lang=zh" % ident)
    sec_uid = (d["data"].get("author") or {}).get("sec_uid") or \
              (d["data"].get("author") or {}).get("sec_user_id")
else:
    sec_uid = ident
```

### 第三步：拉主页 + 作品列表

```python
# ① 主页
prof = get("/account/api/v1/douyin/user?sec_user_id=%s&wait=28&lang=zh" % sec_uid)
u = prof.get("data") or {}

# ② 作品列表（自动翻页，默认最多 3 页 = 约 40 条）
items, cursor, pages = [], None, 0
while pages < 3:
    q = "/posts/api/v1/douyin/user/posts?sec_user_id=%s&count=20&wait=28&lang=zh" % sec_uid
    if cursor:
        q += "&cursor=" + urllib.parse.quote(cursor)
    d = get(q)
    data = d.get("data") or {}
    items += data.get("items") or []
    cursor = data.get("cursor")
    pages += 1
    if not data.get("has_more") or not cursor:
        break

# ③ 按发布时间倒序（列表里置顶作品在最前，必须自己排）
def ts(it):
    s = it.get("created_at") or ""
    try:
        return datetime.datetime.strptime(s[:19], "%Y-%m-%dT%H:%M:%S").timestamp()
    except Exception:
        return 0

items.sort(key=ts, reverse=True)
```

### 第四步：输出报告

```
📊 账号分析报告 · {nickname}

【账号概况】
· 抖音号：{unique_id}
· 粉丝：{follower_count}
· 获赞：{total_digg}
· 作品数：{content_count}
· 关注：{following_count}
· 简介：{signature}

【作品表现】
共取 {N} 条作品，时间跨度 {最早日期} ~ {最新日期}

按点赞排序 Top 5：
1. {title}
   赞 {digg}｜评 {comment}｜藏 {collect}｜转 {share}｜{日期}
2. ...

【数据分析】
· 平均点赞：{avg_digg}
· 中位数点赞：{median_digg}
· 爆款（赞 > 1000）：{n} 条，占 {pct}%
· 更新频率：近 30 天发了 {m} 条
· 最近更新：{最新日期}（{x} 天前）
· 互动率：平均 (赞+评+藏+转) / 粉丝 = {rate}%

【内容方向】
（从标题里提取高频关键词，归纳 3-5 个内容方向）

---
需要我提取某条爆款的文案，或者扫一下评论区看客户在问什么吗？
```

---

## 完整可跑脚本

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json, pathlib, sys, time, re, urllib.request, urllib.parse, datetime

CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
if not CFG.exists():
    sys.exit("❌ 未配置密钥")
KEY = json.loads(CFG.read_text(encoding="utf-8"))["key"]
GW = "https://www.fangzhinao.com/v1"

UA = ("Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) "
      "AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Mobile/15E148")

def get(path):
    req = urllib.request.Request(GW + path, headers={"x-api-key": KEY})
    with urllib.request.urlopen(req, timeout=90) as r:
        return json.loads(r.read().decode("utf-8", "replace"))

def resolve(url):
    if "v.douyin.com" in url:
        req = urllib.request.Request(url, method="HEAD", headers={"User-Agent": UA})
        try:
            with urllib.request.urlopen(req, timeout=20) as r:
                url = r.headers.get("Location") or r.url or url
        except urllib.error.HTTPError as e:
            url = e.headers.get("Location") or url
    m = re.search(r"/share/user/([A-Za-z0-9_\-]+)", url) or re.search(r"/user/([A-Za-z0-9_\-]{20,})", url)
    if m:
        return "user", m.group(1)
    m = re.search(r"/share/video/(\d+)", url) or re.search(r"/video/(\d+)", url)
    if m:
        return "video", m.group(1)
    raise RuntimeError("解析不出 ID")

target = sys.argv[1] if len(sys.argv) > 1 else ""
kind, ident = resolve(target)

if kind == "video":
    d = get("/video/api/v1/douyin/video?aweme_id=%s&wait=28&lang=zh" % ident)
    a = (d.get("data") or {}).get("author") or {}
    sec_uid = a.get("sec_uid") or a.get("sec_user_id")
    if not sec_uid:
        sys.exit("❌ 拿不到作者 ID")
else:
    sec_uid = ident

prof = (get("/account/api/v1/douyin/user?sec_user_id=%s&wait=28&lang=zh" % sec_uid)).get("data") or {}
st = prof.get("stats") or {}
print("昵称:", prof.get("nickname"))
print("抖音号:", prof.get("unique_id"))
print("粉丝:", st.get("follower_count"), "| 获赞:", st.get("total_digg"),
      "| 作品:", st.get("content_count"), "| 关注:", st.get("following_count"))
print("简介:", (prof.get("signature") or "").replace("\n", " ")[:100])
print()

items, cursor, pages = [], None, 0
while pages < 3:
    q = "/posts/api/v1/douyin/user/posts?sec_user_id=%s&count=20&wait=28&lang=zh" % sec_uid
    if cursor:
        q += "&cursor=" + urllib.parse.quote(str(cursor))
    data = (get(q)).get("data") or {}
    items += data.get("items") or []
    cursor = data.get("cursor"); pages += 1
    if not data.get("has_more") or not cursor:
        break

def ts(it):
    try:
        return datetime.datetime.strptime((it.get("created_at") or "")[:19], "%Y-%m-%dT%H:%M:%S").timestamp()
    except Exception:
        return 0

items.sort(key=ts, reverse=True)
print("作品数:", len(items))
print()
print("按点赞 Top 5:")
for it in sorted(items, key=lambda x: ((x.get("stats") or {}).get("digg_count") or 0), reverse=True)[:5]:
    s = it.get("stats") or {}
    print(" ·", (it.get("title") or "")[:40])
    print("   赞%s 评%s 藏%s 转%s | %s" % (
        s.get("digg_count"), s.get("comment_count"),
        s.get("collect_count"), s.get("share_count"),
        (it.get("created_at") or "")[:10]))
```

---

## 错误处理

| 情况 | 提示 |
|---|---|
| 没配密钥 | 「请先把密钥发给我」 |
| 解析不出 ID | 「链接格式不对，请发主页链接或视频链接」 |
| `NOT_FOUND` | 「这个账号不存在，或者链接失效了」 |
| 账号私密 | 「这个号是私密账号，拿不到数据」 |
| 429 / 点数不够 | 「额度用完了，请联系抖音地产人刘广标充值」 |
| 作品为 0 | 「这个号还没有公开作品」 |

**失败直说，不要猜，不要换方案。**

---

## 注意事项

- **作品列表不是按时间排的**（置顶在最前），一定要按 `created_at` 自己排序
- 互动数据在列表里就有，**不用再逐个查 `/video`**，省点数
- `play_count`（播放量）通常是 `null`，抖音不给
- 默认最多翻 3 页（约 40 条），用户要更多再翻
- 粉丝数、获赞数是**实时快照**，不是历史趋势

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
