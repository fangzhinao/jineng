---
name: 本地楼盘数据库
description: "在用户自己电脑上建一个楼盘数据库，存城市/区域/楼盘/二手房/避坑信息/爆款观点。写文案前先查这里，把真实数据填进去，不再编造。触发条件：用户说「初始化数据库」「建楼盘库」「添加楼盘」「查询楼盘」「录入项目」时使用。"
metadata:
  {
    "openclaw": { "emoji": "🏘️", "requires": { "bins": ["python3"] } }
  }
---

# 本地楼盘数据库

> **⚠️ 用之前必须配密钥。** 没有密钥的话，先跟用户说：「在抖音搜『地产人刘广标』，私信说需要领取体验技能密钥，拿到后发给我，我帮你配好。」


在**用户自己的电脑**上建一个 SQLite 数据库，存在本地，不上传任何服务器。

**核心用途：写文案时先查这个库，用真实数据，不编造。**

## 触发场景

- 「初始化数据库」「建楼盘库」
- 「添加楼盘 XX」「录入这个项目」
- 「查一下 XX 楼盘」「XX 区域什么情况」
- 「帮我写条 XX 楼盘的探盘文案」（会先查库）
- 用户发来楼盘资料说「存起来」

---

## 前置：配置密钥

**如果 `~/.fangguagua/config.json` 不存在**：

```
「你还没有配置密钥。

在抖音搜「地产人刘广标」，私信说需要领取体验技能密钥就行。

拿到后是一串 fg_ 开头的内容，发给我，我帮你配好。」
```

---

## 数据库位置

```
~/.fangguagua/楼盘数据库.db
```

**为什么放本地：**
```
· 数据是用户自己的，不该传服务器
· 查询快，不消耗点数
· 断网也能用
```

---

## 第一步：初始化（只需一次）

```python
import sqlite3, pathlib

DB = pathlib.Path.home() / ".fangguagua" / "楼盘数据库.db"
DB.parent.mkdir(parents=True, exist_ok=True)
conn = sqlite3.connect(str(DB))
cur = conn.cursor()

cur.execute("""CREATE TABLE IF NOT EXISTS cities (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    城市名称 TEXT NOT NULL,
    省份 TEXT,
    城市级别 TEXT,
    总人口 TEXT,
    常住人口 TEXT,
    人均GDP TEXT,
    GDP总量 TEXT,
    平均月工资 TEXT,
    平均房价 TEXT,
    房价增长率 TEXT,
    房价区间_低 TEXT,
    房价区间_高 TEXT,
    平均租金 TEXT,
    限购政策 TEXT,
    贷款政策 TEXT,
    公积金政策 TEXT,
    支柱产业 TEXT,
    重点发展区域 TEXT,
    城市优势 TEXT,
    城市劣势 TEXT,
    气候类型 TEXT,
    知名大学 TEXT,
    知名医院 TEXT,
    备注 TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)""")

cur.execute("""CREATE TABLE IF NOT EXISTS areas (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    城市名称 TEXT NOT NULL,
    区域板块名称 TEXT NOT NULL,
    区域定位 TEXT,
    区域等级 TEXT,
    整体房价范围 TEXT,
    区域均价 TEXT,
    核心标签 TEXT,
    适合人群标签 TEXT,
    区域描述 TEXT,
    地铁线路信息 TEXT,
    主干道 TEXT,
    大型商场 TEXT,
    大型超市 TEXT,
    优质小学 TEXT,
    优质初中 TEXT,
    优质高中 TEXT,
    优质幼儿园 TEXT,
    三甲医院 TEXT,
    公园 TEXT,
    主要产业 TEXT,
    主要企业 TEXT,
    板块优势 TEXT,
    板块劣势 TEXT,
    避坑项目 TEXT,
    发展规划 TEXT,
    备注 TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)""")

cur.execute("""CREATE TABLE IF NOT EXISTS properties (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    城市名称 TEXT NOT NULL,
    区域名称 TEXT,
    楼盘名称 TEXT NOT NULL,
    项目地址 TEXT,
    开发商 TEXT,
    物业公司 TEXT,
    总户数 TEXT,
    容积率 TEXT,
    绿化率 TEXT,
    建筑风格 TEXT,
    装修标准 TEXT,
    产权年限 TEXT,
    交房时间 TEXT,
    销售均价范围 TEXT,
    最低单价 TEXT,
    最高单价 TEXT,
    物业费 TEXT,
    车位费 TEXT,
    车位配比 TEXT,
    划片小学 TEXT,
    划片初中 TEXT,
    外部配套 TEXT,
    内部配套描述 TEXT,
    核心优势 TEXT,
    潜在问题 TEXT,
    产品标签 TEXT,
    目标客群 TEXT,
    历史最高价 TEXT,
    历史最低价 TEXT,
    价格变化说明 TEXT,
    降价幅度 TEXT,
    备注 TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)""")

cur.execute("""CREATE TABLE IF NOT EXISTS second_hand_houses (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    城市 TEXT NOT NULL,
    区域板块 TEXT,
    楼盘名称 TEXT,
    户型 TEXT,
    建筑面积 TEXT,
    楼层 TEXT,
    朝向 TEXT,
    建筑年代 TEXT,
    装修情况 TEXT,
    挂牌总价 TEXT,
    挂牌单价 TEXT,
    参考成交价 TEXT,
    房源状态 TEXT,
    满二满五 TEXT,
    是否唯一住房 TEXT,
    核心卖点 TEXT,
    优势描述 TEXT,
    周边配套 TEXT,
    存在问题 TEXT,
    备注 TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)""")

cur.execute("""CREATE TABLE IF NOT EXISTS avoid_areas (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    城市 TEXT NOT NULL,
    区域名称 TEXT NOT NULL,
    避坑信息 TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)""")

cur.execute("""CREATE TABLE IF NOT EXISTS avoid_properties (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    城市 TEXT NOT NULL,
    区域 TEXT,
    楼盘名字 TEXT NOT NULL,
    避坑信息 TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)""")

cur.execute("""CREATE TABLE IF NOT EXISTS hot_views (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    城市 TEXT,
    爆款观点 TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)""")

conn.commit()
conn.close()
print("✅ 数据库已建好：", DB)
print("   7 张表：cities / areas / properties / second_hand_houses /")
print("           avoid_areas / avoid_properties / hot_views")
```

---

## 这个数据库是干什么的（智能体必读）

**一句话：存「写文案要用的真实素材」，避免编造。**

### 七张表各管什么

| 表 | 存什么 | 什么时候用 |
|---|---|---|
| `cities` | 城市大盘数据 | 写城市类文案（人口/房价/政策/产业） |
| `areas` | 区域板块 | 写「XX 区怎么样」，配套/优劣势 |
| `properties` | 新房楼盘 | 写探盘文案，最关键的一张表 |
| `second_hand_houses` | 二手房 | 写二手房、砍价、成交价类 |
| `avoid_areas` | 避坑区域 | 写「这些区域别碰」 |
| `avoid_properties` | 避坑楼盘 | 写「这几个盘要注意」 |
| `hot_views` | 爆款观点 | 存验证过能爆的论点句式 |

### 字段怎么填（最重要的几个）

**`properties` 表 —— 写探盘文案必须有这几项：**

| 字段 | 填什么 | 举例 |
|---|---|---|
| `城市名称` | 必填 | 郴州 |
| `区域名称` | 必填 | 北湖区 |
| `楼盘名称` | 必填 | 某某府 |
| `销售均价范围` | 真实价格区间 | 8000-9500 元/㎡ |
| `最低单价` / `最高单价` | 具体数字 | 7800 / 10200 |
| `核心优势` | **写文案最常用的字段** | 地铁口 300 米、学区双名校、赠送面积大 |
| `潜在问题` | **写「真话」文案的关键** | 楼间距 30 米偏窄、西晒、车位比 1:0.7 |
| `划片小学` / `划片初中` | 真实学区 | 北湖小学 / 北湖实验中学 |
| `交房时间` | 期房/现房 | 2026 年 6 月 |
| `目标客群` | 谁适合买 | 首次置业、改善三房 |

**`areas` 表的 `板块优势` / `板块劣势` 是写区域文案的核心。**
**`avoid_properties` 的 `避坑信息` 是写「敢说真话」人设的关键 —— 这是最能涨粉的素材。**

---

## 第二步：录入数据

### 智能体怎么录（用户口述 → 结构化）

**用户可能这样说：**
```
「郴州北湖区有个盘叫金域华府，均价 8500，最小 89 平，
 地铁 2 号线 500 米，划片北湖小学，
 优势是户型方正、赠送面积，问题是车位少、靠马路吵，
 目标客群是首次置业」
```

**智能体要做的：**
```python
import sqlite3, pathlib
DB = pathlib.Path.home() / ".fangguagua" / "楼盘数据库.db"
conn = sqlite3.connect(str(DB))
cur = conn.cursor()

cur.execute("""INSERT INTO properties
    (城市名称, 区域名称, 楼盘名称, 销售均价范围, 最低单价, 最高单价,
     外部配套, 划片小学, 核心优势, 潜在问题, 目标客群)
    VALUES (?,?,?,?,?,?,?,?,?,?,?)""",
    ("郴州", "北湖区", "金域华府", "8500 左右", "8200", "8800",
     "地铁 2 号线 500 米", "北湖小学",
     "户型方正；赠送面积大", "车位配比低；临街楼栋偏吵", "首次置业"))

conn.commit()
print("✅ 已录入，id =", cur.lastrowid)
conn.close()
```

### 缺字段时怎么办

**不要瞎编。问用户：**
```
「金域华府录好了。还缺这几项，你方便的话补一下，文案能更准：
 · 交房时间（期房还是现房？）
 · 容积率和绿化率
 · 物业费多少钱
 不方便说也行，这几项文案里我就不提。」
```

### 用户发来文件怎么办

**用户可能发来 Excel / Word / 截图。智能体要：**
```
① 读文件 → 提取楼盘信息
② 列出来给用户确认：「我理解是这样的，对吗？」
③ 用户确认后再入库
④ 缺的字段标出来，不要填「暂无」占位
```

---

## 第三步：查询（写文案前必做）

```python
import sqlite3, pathlib
DB = pathlib.Path.home() / ".fangguagua" / "楼盘数据库.db"
conn = sqlite3.connect(str(DB))
conn.row_factory = sqlite3.Row      # ⚠️ 必须在 cursor 之前，否则查出来是元组
cur = conn.cursor()

# 按楼盘名模糊查
cur.execute("SELECT * FROM properties WHERE 楼盘名称 LIKE ?", ("%金域%",))
p = cur.fetchone()

if p:
    print("楼盘：", p["楼盘名称"])
    print("均价：", p["销售均价范围"])
    print("优势：", p["核心优势"])
    print("问题：", p["潜在问题"])
    print("学区：", p["划片小学"], "/", p["划片初中"])
    
    # 顺带查区域
    cur.execute("SELECT * FROM areas WHERE 城市名称=? AND 区域板块名称=?",
                (p["城市名称"], p["区域名称"]))
    a = cur.fetchone()
    if a:
        print("区域优势：", a["板块优势"])
        print("区域劣势：", a["板块劣势"])
else:
    print("库里没有这个楼盘。要不要先告诉我它的信息，我帮你录进去？")

conn.close()
```

**查不到时的说法：**
```
「库里还没有金域华府。你有它的资料吗？发给我我录进去，
 这样文案里就能用真实数据。或者我先按你口述的写，你补充？」
```

**绝不能做的：查不到就自己编数据。**

---

## 第四步：装查看器（可选，给「人」看）

**智能体用 Python 就能读写，不需要装任何软件。**
**只有用户想「自己打开看看数据」时才需要装。**

### 检测系统并给下载链接

```python
import sys, platform

if sys.platform == "win32":
    print("""Windows 查看器下载（解压即用）：
https://bksjk.oss-cn-beijing.aliyuncs.com/db/DB.Browser.SQLite-win64-3.13.1.zip

下完解压，双击里面的 DB Browser for SQLite.exe 就行。
打开后点「打开数据库」，选 ~/.fangguagua/楼盘数据库.db""")

elif sys.platform == "darwin":
    print("""Mac 查看器下载：
https://bksjk.oss-cn-beijing.aliyuncs.com/db/DB.Browser.SQLite-3.13.1.dmg

打开 dmg，把图标拖进「应用程序」。
首次打开可能提示「无法验证开发者」，去
系统设置 → 隐私与安全性 → 点「仍要打开」""")

else:
    print("Linux 用户：sudo apt install sqlitebrowser")
```

### ⚠️ 如果不兼容 / 下载失败 → 引导去官网

```
OSS 链接下不下来，或者装完跑不起来，告诉用户：

「这个安装包可能跟你的系统不兼容。你可以去官网自己下一个：

  https://sqlitebrowser.org/dl/

  页面上按你的系统选：
  · Windows → 找 '.zip (no installer) for 64-bit Windows'
  · macOS   → 找 'DB Browser for SQLite (Universal)'

  下完不用管它，我们的数据库文件在
  ~/.fangguagua/楼盘数据库.db，用这个软件打开就行。」
```

**注意：查看器只是给人看的，装不上也不影响智能体读写数据。**

---

## 常见操作速查

| 用户说 | 怎么做 |
|---|---|
| 「初始化数据库」 | 跑建表代码 |
| 「添加楼盘 XX」 | INSERT 到 properties，缺字段要问 |
| 「添加区域 XX」 | INSERT 到 areas |
| 「这个盘要避坑」 | INSERT 到 avoid_properties |
| 「查一下 XX 楼盘」 | SELECT LIKE 模糊查 |
| 「我库里有什么」 | 每张表 COUNT，列个汇总 |
| 「导出给我」 | `sqlite3` 的 `.dump` 或转 CSV |
| 「删掉 XX」 | 先确认，再 DELETE |

**查库汇总的说法：**
```
你的库里现在有：
· 城市 1 个（郴州）
· 区域 8 个
· 新房楼盘 23 个
· 二手房 15 套
· 避坑楼盘 4 个

写文案时我会自动查这些数据。
```

---

## 错误处理

| 情况 | 提示 |
|---|---|
| 数据库不存在 | 自动跑建表，说「数据库建好了」 |
| 表结构旧了缺字段 | `ALTER TABLE ... ADD COLUMN`，不删表 |
| 查不到 | 「库里没有 XX，要不要先录进去？」 |
| INSERT 报唯一约束 | 「这个楼盘已经录过了，要更新吗？」 |
| 查看器装不上 | 引导去官网 https://sqlitebrowser.org/dl/ |

---

## 五条铁律

```
① 数据必须真实 —— 查不到就问，绝不编价格/学区/配套
② 不删表 —— 升级只加字段，永远不 DROP
③ 本地优先 —— 数据库不上传任何服务器
④ 录入前确认 —— 用户口述的信息先列出来让他点头
⑤ 缺字段就空着 —— 不要填「暂无」「未知」占位，写文案时跳过
```

**技术提醒：`conn.row_factory = sqlite3.Row` 必须写在 `conn.cursor()` 之前，
否则查出来是元组，用字段名取值会报 `TypeError: object is not iterable`。**

---

## 跟知识库的分工

| | 本地楼盘数据库 | 知识库（爆款框架） |
|---|---|---|
| 存什么 | **你的真实项目数据** | 别人的爆款结构和成品 |
| 在哪 | 用户电脑 | 房智脑服务器 |
| 谁维护 | 用户自己录 | 抖音地产人刘广标维护 |
| 写文案时 | 提供**事实**（价格/学区/优劣势） | 提供**结构**（怎么组织语言） |

**好文案 = 知识库的框架 + 本地库的真实数据。两个都要用。**

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
