---
name: 抖音评论区扫描
description: "扫描抖音账号作品的评论区，输出可追踪的跟进清单（哪条作品、谁评论的、说了什么、什么时候）。用户想跟进评论、维护客户、找选题、了解客户需求时使用。默认扫最近 7 天的作品。"
metadata:
  {
    "openclaw": { "emoji": "💬", "requires": { "bins": ["python3"] } }
  }
---

# 抖音评论区扫描

**核心目的：帮你把评论变成「可跟进的客户名单」，不是只给你一份分析报告。**

扫完之后你要能回答：
- 哪条作品下有人问了？（点进去就能找到）
- 谁问的？（名字、时间）
- 问了什么？（原话）
- 回了没有？（还没回的排前面）
- 值不值得重点跟？（问的人多、点赞高）

## 触发场景

- 「看看评论区有没有客户」
- 「帮我跟进一下评论」
- 「哪些评论我还没回」
- 「客户最关心什么」
- 「分析一下评论区，找选题」
- 用户发来主页链接或视频链接 + 提到评论

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

## 默认规则

```
✅ 默认只扫最近 7 天发的作品
✅ 每个作品最多抓 50 条评论（接口上限，写 100 会报 400）
✅ 每个作品之间等 3 秒（防风控）
✅ 一次最多扫 10 个作品
✅ 如果用户给的是单条视频链接 → 只扫这一条
✅ 同时导出 CSV，方便用户在 Excel 里标「已回复」

❌ 默认不扫全部历史作品（点数消耗大 + 风控风险高）
```

**用户说「扫全部」时要二次确认：**
```
「扫全部作品大概要 X 个作品 × 2 点 = X 点，评论抓取频率高也更容易被限流。
 建议先扫最近 7 天，要吗？」
```

---

## 输出格式（重点改了这里）

### 主输出：跟进清单（按「待回复优先 + 时间倒序」排）

```
💬 评论区跟进清单

【扫描范围】
· 账号：{昵称}
· 作品：{N} 条（{最早日期} ~ {最新日期}）
· 评论：{M} 条
· 未回复：{K} 条

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
【🔴 优先跟进（问具体问题的，还没回的）】
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 王姐（3 小时前 · 5 赞 · 0 回复）
   评论：「郴州现在首付最低多少？我手上就 20 万」
   作品：《首付 20 万在郴州能买哪里》
   链接：douyin.com/video/7xxxxxxxxxxxxxxxx
   → 建议回复：先问清是首套还是二套，首套最低 20% 首付，20 万大概能看总价 100 万左右的

2. 小李（昨天 20:15 · 2 赞 · 0 回复）
   评论：「XX 楼盘什么时候交房」
   作品：《郴州在售楼盘盘点》
   链接：douyin.com/video/7xxxxxxxxxxxxxxxx
   → 建议回复：该楼盘预计 2027 年 6 月交付，目前还在售

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
【🟡 值得互动（夸你的、闲聊的，回一句能加深印象）】
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

3. 房产老兵（2 天前 · 18 赞 · 1 回复）
   评论：「这个分析很实在，比那些只吹的好多了」
   作品：《为什么我不推荐这几个盘》
   → 建议回复：谢谢认可，后面会继续讲实在的

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
【⚪ 低优先级（点赞、表情、无意义）】
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

4. 用户8xxx（3 天前 · 0 赞）：「👍」
5. ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
【📊 客户在问什么（统计）】
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

· 问价格/首付：8 条  ← 最高频
· 问交房时间：5 条
· 问某个具体楼盘：4 条
· 问学区：3 条
· 嫌贵/怕跌：3 条

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
【💡 可以做下期视频的选题】
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 「首付 20 万在郴州到底能买哪」—— 8 个人在问
2. 「郴州各楼盘交房时间汇总」—— 5 个人在问
3. ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📎 完整清单已导出：桌面/评论跟进_{账号}_{日期}.csv
   （含每条评论的作品、作者、内容、时间、赞数、评论ID，可在 Excel 里标状态）
```

---

### 副输出：CSV 文件（给用户自己在 Excel 里跟进）

**文件路径：**`~/Desktop/评论跟进_{昵称}_{YYYYMMDD}.csv`

**列结构（UTF-8 BOM，Excel 直接打开不乱码）：**

```csv
序号,作品标题,作品链接,评论人,评论内容,评论时间,点赞数,回复数,评论ID,类型,建议动作,我的状态
1,首付20万在郴州能买哪里,https://www.douyin.com/video/7xxx,王姐,"郴州现在首付最低多少？我手上就20万",2026-11-14 18:32,5,0,7xxxxxxxxxxxxxxx,咨询,问清首套二套,
2,郴州在售楼盘盘点,https://www.douyin.com/video/7xxx,小李,XX楼盘什么时候交房,2026-11-13 20:15,2,0,7xxxxxxxxxxxxxxx,咨询,回复交房时间,
3,为什么我不推荐这几个盘,https://www.douyin.com/video/7xxx,房产老兵,这个分析很实在,2026-11-12 09:10,18,1,7xxxxxxxxxxxxxxx,认可,回一句感谢,
```

**「我的状态」这一列留空，让用户自己填「已回」「已加微信」「已到店」。**

---

## 评论分类规则（用来分优先级）

**抓完之后按关键词自动分类：**

```python
CATEGORY_RULES = [
    # (类型, 关键词列表, 优先级)
    ("咨询", ["多少", "怎么", "哪", "什么时候", "能不能", "有没有", "多少钱",
              "首付", "总价", "面积", "学区", "交房", "月供", "贷款", "限购"], "🔴 高"),
    ("异议", ["贵", "跌", "不值", "骗", "假", "坑", "远", "配套差", "太偏"], "🔴 高"),
    ("认可", ["实在", "专业", "讲得好", "有用", "谢谢", "学到了", "干货"], "🟡 中"),
    ("同行", ["同行", "中介", "我也是做"], "🟡 中"),
    ("无意义", ["👍", "沙发", "第一", "哈哈", "😂", "。"], "⚪ 低"),
]
```

**优先级判断还要叠加：**
```
· reply_count == 0 且 是咨询类  → 提到最前面（没人回，最需要你回）
· digg_count >= 5                → 加分（很多人认同，值得回）
· 评论时间在 24 小时内            → 加分（趁热）
```

---

## 执行流程

### 第一步：确定目标

```python
kind, ident = resolve(user_input)

if kind == "video":
    targets = [{"aweme_id": ident, "title": "", "created_at": ""}]
    sec_uid = None
else:
    sec_uid = ident
    targets = None
```

### 第二步：拿作品列表 + 筛时间范围

```python
if targets is None:
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

    # ⚠️ 列表不是按时间排的（置顶在最前），必须自己排
    items.sort(key=ts, reverse=True)

    cutoff = time.time() - 7 * 86400
    recent = [x for x in items if ts(x) >= cutoff]

    if not recent:
        latest = items[0] if items else None
        days = int((time.time() - ts(latest)) / 86400) if latest else 999
        print("这个号近 7 天没更新，最近一条是 %d 天前。" % days)
        print("要不要改成扫最近 10 个作品？")
        raise SystemExit(0)

    targets = [{"aweme_id": x.get("content_id"),
                "title": (x.get("title") or "").strip()[:40],
                "created_at": (x.get("created_at") or "")[:19],
                "digg": (x.get("stats") or {}).get("digg_count") or 0}
               for x in recent][:10]
```

### 第三步：逐个抓评论（限速 + 保留全部字段）

```python
def parse_time(s):
    """把 2026-11-14T18:32:10 转成可读时间"""
    try:
        dt = datetime.datetime.strptime((s or "")[:19], "%Y-%m-%dT%H:%M:%S")
        return dt
    except Exception:
        return None

def human_time(dt):
    """转成「3 小时前」这种"""
    if not dt: return "未知"
    delta = datetime.datetime.now() - dt
    sec = delta.total_seconds()
    if sec < 3600:      return "%d 分钟前" % int(sec/60)
    if sec < 86400:     return "%d 小时前" % int(sec/3600)
    if sec < 86400*7:   return "%d 天前" % int(sec/86400)
    return dt.strftime("%Y-%m-%d")

all_c = []
for i, t in enumerate(targets, 1):
    if i > 1:
        time.sleep(3)
    q = ("/comments/api/v1/douyin/video/comments"
         "?aweme_id=%s&count=50&wait=28&lang=zh" % t["aweme_id"])
    try:
        d = get(q)
    except Exception as e:
        print("  第 %d 个作品抓取失败：%s" % (i, str(e)[:60]))
        continue

    if not d.get("success"):
        print("  第 %d 个作品跳过：%s" % (i, (d.get("error") or {}).get("code")))
        continue

    data = d.get("data") or {}
    cmts = data.get("items") or []

    for c in cmts:
        author = c.get("author") or {}
        cdt = parse_time(c.get("created_at"))
        all_c.append({
            # ── 追踪必需的五个字段 ──
            "video_title":  t["title"] or "(无标题)",
            "video_id":     t["aweme_id"],
            "video_link":   "https://www.douyin.com/video/" + str(t["aweme_id"]),
            "author":       (author.get("nickname") or "").strip() or "匿名",
            "text":         (c.get("text") or "").strip(),
            "created_at":   c.get("created_at") or "",
            "created_dt":   cdt,
            "time_human":   human_time(cdt),
            # ── 判断优先级用的 ──
            "digg":         (c.get("stats") or {}).get("digg_count") or 0,
            "reply_count":  c.get("reply_count") or 0,
            "comment_id":   c.get("comment_id") or "",
            "is_pinned":    c.get("is_pinned") or False,
            "is_author_liked": c.get("is_author_liked") or False,
        })

    print("  [%d/%d] %-20s → %d 条" % (i, len(targets), (t["title"] or "")[:20], len(cmts)))
```

### 第四步：分类 + 排序

```python
def classify(text):
    for name, kws, pri in CATEGORY_RULES:
        if any(k in text for k in kws):
            return name, pri
    return "其他", "⚪ 低"

for c in all_c:
    cat, pri = classify(c["text"])
    c["category"] = cat
    c["priority"] = pri

# 排序：优先级 > 是否已回复 > 点赞 > 时间
PRI_ORDER = {"🔴 高": 0, "🟡 中": 1, "⚪ 低": 2}
all_c.sort(key=lambda x: (
    PRI_ORDER.get(x["priority"], 3),
    x["reply_count"],                    # 没人回的排前面
    -x["digg"],
    -(x["created_dt"].timestamp() if x["created_dt"] else 0),
))
```

### 第五步：导出 CSV

```python
import csv, os

nick = "{账号昵称}"   # 从 user 接口拿
date_str = datetime.datetime.now().strftime("%Y%m%d")
desktop = os.path.join(os.path.expanduser("~"), "Desktop")
csv_path = os.path.join(desktop, "评论跟进_%s_%s.csv" % (nick, date_str))

with open(csv_path, "w", encoding="utf-8-sig", newline="") as f:
    w = csv.writer(f)
    w.writerow(["序号", "作品标题", "作品链接", "评论人", "评论内容",
                "评论时间", "点赞数", "回复数", "评论ID",
                "类型", "建议动作", "我的状态"])
    for i, c in enumerate(all_c, 1):
        w.writerow([
            i,
            c["video_title"],
            c["video_link"],
            c["author"],
            c["text"],
            c["created_at"].replace("T", " ")[:16],
            c["digg"],
            c["reply_count"],
            c["comment_id"],
            c["category"],
            "",          # 建议动作，AI 分析后填
            "",          # 我的状态，用户自己填
        ])

print("CSV 已导出：", csv_path)
```

**注意 `encoding="utf-8-sig"` —— 带 BOM，Excel 打开中文不乱码。**

---

## 完整可跑脚本

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json, pathlib, sys, time, re, csv, os, datetime
import urllib.request, urllib.error, urllib.parse

CFG = pathlib.Path.home() / ".fangguagua" / "config.json"
if not CFG.exists():
    sys.exit("❌ 未配置密钥")
KEY = json.loads(CFG.read_text(encoding="utf-8"))["key"]
GW = "https://www.fangzhinao.com/v1"
UA = ("Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) "
      "AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Mobile/15E148")

CATEGORY_RULES = [
    ("咨询", ["多少", "怎么", "哪", "什么时候", "能不能", "有没有",
              "首付", "总价", "面积", "学区", "交房", "月供", "贷款", "限购"], "🔴 高"),
    ("异议", ["贵", "跌", "不值", "骗", "假", "坑", "远", "配套差", "太偏"], "🔴 高"),
    ("认可", ["实在", "专业", "讲得好", "有用", "谢谢", "学到了", "干货"], "🟡 中"),
    ("同行", ["同行", "中介", "我也是做"], "🟡 中"),
    ("无意义", ["👍", "沙发", "第一", "哈哈", "😂"], "⚪ 低"),
]

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
    m = re.search(r"/share/user/([A-Za-z0-9_\-]+)", url) or \
        re.search(r"/user/([A-Za-z0-9_\-]{20,})", url)
    if m: return "user", m.group(1)
    m = re.search(r"/share/video/(\d+)", url) or re.search(r"/video/(\d+)", url)
    if m: return "video", m.group(1)
    raise RuntimeError("解析不出 ID")

def ts(it):
    try:
        return datetime.datetime.strptime(
            (it.get("created_at") or "")[:19], "%Y-%m-%dT%H:%M:%S").timestamp()
    except Exception:
        return 0

def parse_time(s):
    try:
        return datetime.datetime.strptime((s or "")[:19], "%Y-%m-%dT%H:%M:%S")
    except Exception:
        return None

def human_time(dt):
    if not dt: return "未知"
    sec = (datetime.datetime.now() - dt).total_seconds()
    if sec < 3600:   return "%d 分钟前" % int(sec/60)
    if sec < 86400:  return "%d 小时前" % int(sec/3600)
    if sec < 86400*7:return "%d 天前" % int(sec/86400)
    return dt.strftime("%Y-%m-%d")

def classify(text):
    for name, kws, pri in CATEGORY_RULES:
        if any(k in text for k in kws):
            return name, pri
    return "其他", "⚪ 低"

target = sys.argv[1] if len(sys.argv) > 1 else ""
days = int(sys.argv[2]) if len(sys.argv) > 2 else 7
kind, ident = resolve(target)

nick = ""
if kind == "video":
    targets = [{"aweme_id": ident, "title": "(单条视频)", "created_at": ""}]
else:
    # 先拿昵称
    try:
        u = get("/account/api/v1/douyin/user?sec_user_id=%s&wait=28&lang=zh" % ident)
        nick = ((u.get("data") or {}).get("user") or {}).get("nickname") or ""
    except Exception:
        pass
    items, cursor, pages = [], None, 0
    while pages < 3:
        q = "/posts/api/v1/douyin/user/posts?sec_user_id=%s&count=20&wait=28&lang=zh" % ident
        if cursor:
            q += "&cursor=" + urllib.parse.quote(str(cursor))
        data = (get(q)).get("data") or {}
        items += data.get("items") or []
        cursor = data.get("cursor"); pages += 1
        if not data.get("has_more") or not cursor:
            break
    items.sort(key=ts, reverse=True)
    cutoff = time.time() - days * 86400
    recent = [x for x in items if ts(x) >= cutoff]
    if not recent:
        print("近 %d 天没作品，最近一条 %s" % (days, (items[0].get("created_at") or "")[:10] if items else "无"))
        sys.exit(0)
    targets = [{"aweme_id": x.get("content_id"),
                "title": (x.get("title") or "").strip()[:40],
                "created_at": (x.get("created_at") or "")[:19]}
               for x in recent][:10]

print("扫描", len(targets), "个作品")
all_c = []
for i, t in enumerate(targets, 1):
    if i > 1:
        time.sleep(3)
    try:
        d = get("/comments/api/v1/douyin/video/comments?aweme_id=%s&count=50&wait=28&lang=zh"
                % t["aweme_id"])
    except Exception as e:
        print("  [%d] 失败 %s" % (i, str(e)[:50])); continue
    if not d.get("success"):
        print("  [%d] 跳过 %s" % (i, (d.get("error") or {}).get("code"))); continue

    cm = (d.get("data") or {}).get("items") or []
    for c in cm:
        a = c.get("author") or {}
        cdt = parse_time(c.get("created_at"))
        all_c.append({
            "video_title": t["title"] or "(无标题)",
            "video_id": t["aweme_id"],
            "video_link": "https://www.douyin.com/video/" + str(t["aweme_id"]),
            "author": (a.get("nickname") or "").strip() or "匿名",
            "text": (c.get("text") or "").strip(),
            "created_at": c.get("created_at") or "",
            "created_dt": cdt,
            "time_human": human_time(cdt),
            "digg": (c.get("stats") or {}).get("digg_count") or 0,
            "reply_count": c.get("reply_count") or 0,
            "comment_id": c.get("comment_id") or "",
        })
    print("  [%d/%d] %-20s → %d 条" % (i, len(targets), (t["title"] or "")[:20], len(cm)))

for c in all_c:
    c["category"], c["priority"] = classify(c["text"])

PRI = {"🔴 高": 0, "🟡 中": 1, "⚪ 低": 2}
all_c.sort(key=lambda x: (PRI.get(x["priority"], 3), x["reply_count"], -x["digg"],
                          -(x["created_dt"].timestamp() if x["created_dt"] else 0)))

# 导出 CSV
date_str = datetime.datetime.now().strftime("%Y%m%d")
desktop = os.path.join(os.path.expanduser("~"), "Desktop")
csv_path = os.path.join(desktop, "评论跟进_%s_%s.csv" % (nick or "抖音", date_str))
with open(csv_path, "w", encoding="utf-8-sig", newline="") as f:
    w = csv.writer(f)
    w.writerow(["序号","作品标题","作品链接","评论人","评论内容","评论时间",
                "点赞数","回复数","评论ID","类型","建议动作","我的状态"])
    for i, c in enumerate(all_c, 1):
        w.writerow([i, c["video_title"], c["video_link"], c["author"], c["text"],
                    c["created_at"].replace("T"," ")[:16], c["digg"],
                    c["reply_count"], c["comment_id"], c["category"], "", ""])

print()
print("共 %d 条评论，CSV 已导出：" % len(all_c))
print(csv_path)

# 终端预览：只打高优先级的
print()
print("🔴 优先跟进：")
for c in [x for x in all_c if x["priority"] == "🔴 高"][:10]:
    print("  %s（%s · %d赞 · %d回复）" % (c["author"], c["time_human"], c["digg"], c["reply_count"]))
    print("    「%s」" % c["text"][:50])
    print("    作品：%s" % c["video_title"])
    print("    %s" % c["video_link"])
    print()
```

**用法：**
```bash
python3 scan.py "https://v.douyin.com/xxx/"      # 近 7 天
python3 scan.py "https://v.douyin.com/xxx/" 30   # 近 30 天
python3 scan.py "视频链接"                        # 单条
```

---

## 错误处理

| 情况 | 提示 |
|---|---|
| 没配密钥 | 「请先把密钥发给我」 |
| 近 N 天无作品 | 「这个号近 N 天没更新，最近一条是 X 天前，要扫最近 10 个作品吗？」 |
| `UPSTREAM_CHANGED` | 「抖音接口结构变了，这个作品暂时扫不了」——换下一个继续 |
| `429` | 立即停止，提示「请求太频繁被限流了，过 10 分钟再试」 |
| 点数不够 | 「额度用完了，请联系抖音地产人刘广标充值」 |
| 全部作品都抓不到 | 「这个号的评论区暂时拿不到，可能是权限限制」 |

**关键：单个作品失败不要中断整个任务，记下来继续下一个。**

---

## 注意事项

- **评论接口最容易触发风控** —— 必须保持 3 秒间隔
- `digg_count` 经常是 `null`，不是错误
- 评论有**置顶**（`is_pinned`），排最前面的不一定是最热
- **`created_at` 是 ISO 格式 `2026-11-14T18:32:10`**，要转成「3 小时前」给用户看
- **`reply_count` 是判断「回没回」的关键** —— 0 表示还没人回（包括你自己）
- 只扫**一级评论**，不扫回复
- 评论内容可能有广告/垃圾，分析时过滤

---

## 输出给用户时的人话

**不要说「API 返回 200，items 数组长度 47」。**

**要说：**
```
扫了 3 条作品，47 条评论，其中 12 条是问具体问题的、还没人回。

最急的是这条（3 小时前）：
  王姐在《首付20万能买哪》下面问「首付最低多少，我手上就20万」

完整清单我导到桌面了：评论跟进_刘广标_20261115.csv
你打开可以标「已回」「已加微信」，跟着表格走就行。
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
