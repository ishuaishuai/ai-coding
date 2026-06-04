# 从一行 SQL 到一张周报：单文件数据看板全链路实战

> 一套不到 800 行代码的小架构，每天 9 点到 23 点准时为团队产出一份能筛选、能下钻、能分享的数据看板。
> 没有 nginx，没有 docker，没有数据库，没有前端框架——但它跑得很稳。
>
> 本文把这套东西的**取数 → 缓存 → 渲染 → 筛选 → 会话展示 → 定时 → 部署 → 复盘**全链路梳理出来，并提炼出 6 条可复用的方法论。读完你应该能在 30 分钟内把它换皮到自己的数据集上。

---

## TL;DR

- **产物**：一个 50MB 的单文件 HTML，里面塞着 27k 行业务数据 + 全部 JS + 全部样式。
- **架构**：本机 mac 抓 Streamlit 后台 PV/UV → JSON → scp 到开发机；开发机查内网数据网关、组装报告、起 HTTP 服务。
- **关键代码量**:Python ~500 行、HTML 模板 ~190 行、Shell ~50 行。
- **日常耗时**:开发机渲染 24s,本机抓取 35s,完全能塞进 cron 的几分钟窗口。
- **复用成本**:换数据集大约 30 分钟,核心是改 SQL + 字段映射 + H1。

---

## 目录

1. 整体架构(本机 ↔ 开发机)
2. 6 条方法论提炼
3. 取数:MCP + 按天分批 + 长消息补全
4. **准确性:数据看板的命门(含"为何不让 LLM 进数据流")**
5. 缓存:可重入是定时任务的命门
6. 看板:单文件 HTML 的极简哲学
7. 筛选:所有过滤都跑在内存里
8. 会话展示:嵌套行 + 折叠
9. 定时:拆分本地与服务器,各干其能
10. 部署:tar over ssh + python http.server
11. 复盘:踩过的坑与权衡
12. 复用清单:换个数据集要改什么
13. 一句话总结

---

## 1. 整体架构

```
┌─────────────────────────────┐         ┌──────────────────────────────┐
│  本机 macOS(cron :10)       │         │  开发机 Linux(cron :12)      │
│                             │         │                              │
│  4 个 Streamlit Agent 后台  │         │  数据查询网关(SQL/内网)      │
│           │                 │         │           │                  │
│           ▼                 │         │           ▼                  │
│  scrape_agents.py           │         │  generate_report.py          │
│  (playwright + chromium)    │         │  (查询→缓存命中→拼装)         │
│           │                 │         │           │                  │
│           ▼                 │         │           ▼                  │
│  cache/agents.json          │  scp    │  cache/*.csv  agents.json    │
│  (原子写:.tmp → rename)     │ ──────► │  (一天一文件)                │
│           │                 │         │           │                  │
│           ▼                 │         │           ▼                  │
│  scp → 开发机               │         │  templates/report.html       │
│                             │         │           │                  │
│                             │         │           ▼                  │
│                             │         │  /srv/report/<project>/      │
│                             │         │  index.html(~50MB)           │
│                             │         │  http.server :8080 提供       │
└─────────────────────────────┘         └──────────────────────────────┘
                                                       │
                                                       ▼
                                  同事浏览器访问 http://<dev-server>:8080/<project>/
```

**核心思路**:取数与渲染落在开发机;本机只承担"开发机做不到"的事——抓需要浏览器的 Streamlit 后台。

---

## 2. 6 条方法论提炼

真正能复用到下一个项目的不是代码,是这几条思考方式。先放在前面,后面每节都会引用。

### ① 先把"昂贵"的隔离出来

把成本最高的环节(远程查询、外部 API、浏览器抓取)单独标记 → 加缓存 → 决定是否要"按天/按 ID"分片。一旦它能复用,整个流程的体感就从"分钟"变成"秒"。

### ② 幂等,让重跑无害

定时任务最怕"中间挂了重启就出问题"。每一步都设计成「再跑一次结果一样」:写文件用临时名 + rename、缓存按天 key、推送先校验目标。挂了?再跑一次。

### ③ 数据和视图分离

查询/解析输出纯 JSON,前端模板只负责渲染。这样:模板可以独立改样式、JSON 可以单独调试、最终塞进 `<!--DATA-->` 占位符。两边的人/AI 不会互相阻塞。

### ④ 能在客户端做的别在服务端做

27k 行数据在浏览器里跑筛选/排序/分页毫无压力。后端只负责"喂一次全量 JSON"。省掉了一整套 Web 服务、登录态、SQL 重查的复杂度。

### ⑤ 让两台机器各干其能

本机能装 chromium、能上外网;开发机能直连内网数据、能 7×24 跑 cron。不要让一台机器扛所有事——按"它擅长什么"切分职责,比"什么都能装"省心十倍。

### ⑥ 日志要能回答"为什么没跑"

每次 cron 运行先打时间戳、网络可达性、关键文件 mtime。出问题时翻日志一秒定位是**没触发**、**触发了但网络挂**、还是**跑完了但没生效**——这是定时任务运维的全部秘密。

---

## 3. 取数:MCP + 按天分批 + 长消息补全

数据集 14 天约 27k 行、含长文本会话内容。一次性 SELECT 会撞两个限制:网关大数据量风控、消息字段被截断到 1800 字符。

### 问题

- SQL 网关单次返回大于阈值会被风控拦截;
- 长消息字段(用户/AI 对话内容)被截断在 1800 字符,对"问题排查"页面是硬伤;
- 每次跑都查 14 天,绝大多数是浪费——历史天数据已经不变了。

### 方案

三步走,每步对应一个独立函数:

1. **初始化 MCP 会话。** 一次 `initialize` + `notifications/initialized` 拿到 `Mcp-Session-Id`,后续所有工具调用都带这个 header。
2. **按天循环 SELECT。** `WHERE date_dt = '2026-06-04'`,每天一个文件名 `cache/<dataset>_YYYY-MM-DD.csv`,避开大数据量限制。
3. **长消息分段补全。** 对 `CHAR_LENGTH(message_content) > 1800` 的行,用 `SUBSTRING(content, offset, 2000)` 分段查,一段段拼回去。

**关键代码片段(按天查 + 缓存命中跳过):**

```python
for d in dates:
    csv_file = _cache_path(cache_dir, d, "csv")
    is_today = (d == today)

    # 历史天命中缓存 → 直接读,不再查
    if not no_cache and not is_today and os.path.exists(csv_file):
        print(f"    {d} 缓存命中")
        with open(csv_file, "r", encoding="utf-8") as f:
            csv_text = f.read()
    else:
        # 当天数据每次都重查;缺失天补查
        if headers is None:
            headers = mcp_session()
        csv_text = _query_one_day(headers, d)
        with open(csv_file, "w", encoding="utf-8") as f:
            f.write(csv_text)
```

**长消息分段补全(offset 推进式查询):**

```python
offset = 1801
while offset < max_len:
    sql = (f"SELECT conversation_id, message_seq, "
           f"SUBSTRING(message_content, {offset}, 2000) as msg_ext "
           f"FROM datasource_table "
           f"WHERE date_dt = '{date_str}' "
           f"AND CHAR_LENGTH(message_content) > {offset - 1}")
    # ... 拼接到 extensions[(sid, seq)]
    if count == 0:
        break  # 这一天没有更长的了
    offset += 2000
```

### 方法论 takeaway

> 分片的 key 要选"自然边界"。日期是天然的、单调递增的、可哈希的——它同时满足"分片"**和**"缓存键"两个角色。如果你的 key 选的是"分页号"或"游标",缓存就会很难写。

---

## 4. 准确性:数据看板的命门

看板能跑起来不算赢;**跑出来的数对得上**才是。这套系统在 7 个失真点上各设了一道防线,外加每周一次自动对账——让"准确"成为可被验证、可被回溯、可被报警的状态,而不是一句"应该没问题"。

### 4.0 前提:数据流里没有大模型

这是这套方案数据可信的**第一前提**,而不是某条优化技巧——所有"取数 → 解析 → 聚合 → 渲染"全是确定性 Python 代码,**没有任何一步过 LLM**。LLM 只参与了写代码的过程("帮我写个按天查询"),不参与代码运行后处理任何一行真实数据。

| 容易出错的做法(LLM 进数据流) | 本方案的做法(确定性管道) |
|---|---|
| "让 GPT 帮我汇总一下这份 CSV" → 数字会被改写、四舍五入、幻觉补全 | `requests.post()` + 固定 SQL 查上游 → 字节级一致 |
| "AI 帮我从消息里抽取意图分类" → 可能 90% 准确,但**哪一行错了你不知道** | `csv.reader()` 解析 → 标准库实现,输入相同输出相同 |
| "用 LLM 判断这条会话有没有解决问题" → 同一条数据每次跑结果还不一样 | `collections.Counter` / `set` 聚合 → 完全确定 |
| "让 Agent 自己写 SQL 然后跑" → 字段名、表名、过滤条件都可能被胡编 | `json.dumps` 注入 HTML → 没有改写空间 |

**数据流逐点(每一步都能用单元测试钉死):**

```python
# 1. 查询:HTTP POST 走 dclaw-mcp,返回固定格式 CSV
csv_text = requests.post(url, headers=headers, json={...}).json()['data']

# 2. 解析:标准库 csv.reader,无任何模型参与
reader = csv.DictReader(io.StringIO(csv_text))
rows = list(reader)

# 3. 聚合:纯 Python 字典/集合,结果可被 assert 锁死
by_dept = collections.defaultdict(set)
for r in rows:
    by_dept[r['dept']].add(r['email'])

# 4. 渲染:模板字符串替换,没有"AI 重写"
html = template.replace("<!--DATA-->",
       f"<script>R={json.dumps(payload)}</script>")
```

每一步都满足「相同输入 → 相同输出」。这意味着:

- **可单元测试**:`assert sum(c.values()) == 27041` 直接钉数字;
- **可二分定位**:对账失败时,逐步打印中间值即可锁定哪一步偏差;
- **可重放**:给同一份 CSV 跑十次,HTML 字节级一致——这是 LLM 流水线给不了的特性;
- **可审计**:同事可以看代码确认"我看到的数字 = 数据库的数字 + 一段确定性变换",不必相信任何"模型也许是对的"。

> **分清 "AI 写代码" 和 "AI 处理数据" 的边界**:前者把人脑活外包出去(一次性成本,写完就固化),后者让 LLM 介入运行时(每次跑都引入不确定性)。**能用 Python 写死的,绝不让模型在运行时参与。** 看板这种"对得上的数字"场景,应该 100% 锁在前者。

> ⚠️ **什么时候才该让 LLM 进数据流?** 当任务本身就是"语义理解"——比如"对每条会话打情感标签"、"按主题聚类"——这时 LLM 是工具不是 bug。但即使在这种场景,也应该把 LLM 输出当成"一列原始字段"**持久化**下来,再让确定性管道汇总它,*不要*每次渲染都重跑模型。

### 4.1 可能让数据不准的 7 个口子

| # | 失真点 | 表现 | 所在环节 |
|---|---|---|---|
| 1 | SQL 截断 / 风控限流 | 返回行数比真实少;HTTP 502;HTTP 200 但只有部分行 | 查询网关 |
| 2 | 长字段截断 | 消息内容被砍到 1800 字符,"问题排查"页缺上下文 | 查询网关 |
| 3 | 跨日边界 | 时区错位,凌晨数据落到错误的 `date_dt` 分区 | 分片定义 |
| 4 | 历史天缓存被污染 | 上游回补了昨天的脏数据,但本地缓存还停留在旧版本 | 缓存层 |
| 5 | 当天命中缓存 | "今天"的数被锁死成首次跑那一刻的快照 | 缓存层 |
| 6 | CSV 解析错位 | 消息内容含逗号/换行,`split(',')` 把一行劈成多行 | 解析层 |
| 7 | 前端聚合 bug | Set 用错 key、JS Number 精度、隐式 sort 改原数组 | 看板渲染 |

### 4.2 7 条对策(每条对应一道防线)

1. **查询级 invariant:哨兵字段。** 每次 SELECT 同时取 `COUNT(*)` 与 `SUM(CHAR_LENGTH(message_content)>1800)` 两个数,写到日志。如果某天总行数突然砍半、或截断行数从 200 变成 0——不是数据变了,是表结构/列名变了,立即告警。
2. **长字段:分段补全 + 长度校验。** 对 `CHAR_LENGTH > 1800` 的行用 `SUBSTRING(content, offset, 2000)` 推进式拼回。拼完后断言 `len(restored) >= 1800`,否则该行打 `extension_failed=true` 标记并在看板上用红色边框显示——**宁可显眼地出错,不可悄悄丢失**。
3. **跨日边界:分片 key 唯一来自上游。** 不要在客户端做"按本地时间归类",直接用上游表的 `date_dt` 字段做 key。多机统一用 UTC 也行,但全链路只能有一种时区——别中途切换。
4. **历史天 invariant:checksum 漂移检测。** 每次跑完,把每个历史天 CSV 的 SHA256 写到 `cache/checksums.json`。下次启动先比对:上游补了昨天的数 → 哈希变了 → 自动失效那一天的缓存重查。
5. **当天永不命中缓存。** 这条规则的本质不是性能,是**"昨天写的数 ≠ 今天的真相"**。当天数据只要还有可能更新,缓存就是在制造谎言。
6. **解析:用 `csv.reader`,永远不要 `split(',')`。** CSV 里只要有一个用户在消息里输入了"你好,我想问下...",自己 split 就当场错位 5 个字段。Python 标准库的 `csv.reader` 正确处理引号转义;一行代码的成本,省一辈子的事故。
7. **跨源交叉验证:第二条独立链路当裁判。** 主链路从 SQL 网关算"今日活跃用户数";旁路 playwright 从 4 个 Streamlit Agent 后台抓 PV/UV。两个数本来就该相近——把它们并排画在看板顶部,差异 > 5% 自动飘红。**真正的准确性 = 两条不同源数据互相打脸的余量很小**。

### 4.3 嵌入"数据健康度"自检面板

看板顶部直接显示一组关键指标,**异常用户自己就能发现**,不必等 PM 来质问:

```python
def build_health_panel(p2_rows, today, yesterday):
    today_rows = [r for r in p2_rows if r['d'] == today]
    yest_rows  = [r for r in p2_rows if r['d'] == yesterday]
    truncated  = sum(1 for r in p2_rows if len(r.get('msg', '')) >= 1800)

    return {
        'total_rows':        len(p2_rows),
        'unique_sids':       len({r['sid']   for r in p2_rows}),
        'unique_users':      len({r['email'] for r in p2_rows}),
        'date_min':          min(r['d'] for r in p2_rows),
        'date_max':          max(r['d'] for r in p2_rows),
        'today_rows':        len(today_rows),
        'yest_rows':         len(yest_rows),
        # 同比 0.5–1.5 是正常区间;<0.3 或 >3 飘红
        'today_yest_ratio':  round(len(today_rows) / max(1, len(yest_rows)), 2),
        'truncated_rows':    truncated,    # 突变 0 或暴增都是警报
        'extension_failed':  sum(1 for r in p2_rows if r.get('ext_failed')),
        'generated_at':      datetime.now().isoformat(timespec='seconds'),
    }
```

把它渲染在 HTML 顶部:

```html
<div class="health">
  <span class="ok">✓ 共 27,041 行 / 1,827 会话 / 412 用户</span>
  <span class="ok">✓ 时间范围 2026-05-22 ～ 2026-06-04</span>
  <span class="warn">⚠ 今/昨日比 0.42(昨日数据可能尚未补全)</span>
  <span class="ok">✓ 截断行 213(与昨日 198 同量级)</span>
  <span class="muted">生成于 2026-06-05 09:12:34</span>
</div>
```

> 💡 **面板设计原则**:每个数都要能让人 5 秒内判断"对不对"。`27,041 行` 是有用的(可以和昨天比);`generated_at` 是有用的(可以判断是否过期);但 `SHA256` 之类的内部哈希*不要*放——人脑读不了。

### 4.4 主动对账:每周一次的"地真"校验

自检面板能发现"和昨天比异常",但发现不了"系统性偏差稳定 5 天了"。所以每周跑一次脚本,从历史数据里随机抽 5 个会话 ID,**用最原始的方式**查上游表,比对消息条数、首末时间、用户邮箱:

```python
# scripts/audit.py(cron 周一 9 点跑)
import random, csv, sys
from generate_report import mcp_session, _query_one_day

def audit(date_str, n=5):
    # 1) 从本地 CSV 读取声称的"事实"
    rows = list(csv.DictReader(open(f'cache/<dataset>_{date_str}.csv')))
    sample = random.sample(rows, min(n, len(rows)))

    # 2) 用一条独立的 SQL 重新查这些会话 ID
    sids = "', '".join({r['conversation_id'] for r in sample})
    sql = f"SELECT conversation_id, COUNT(*) c, MIN(ts) mn, MAX(ts) mx " \
          f"FROM datasource_table WHERE conversation_id IN ('{sids}') " \
          f"GROUP BY conversation_id"
    truth = run_sql(sql)  # 直查上游,绕过本地缓存

    # 3) 比对
    mismatch = []
    for sid, t in truth.items():
        local = [r for r in rows if r['conversation_id'] == sid]
        if len(local) != t['c']:
            mismatch.append((sid, len(local), t['c']))
    if mismatch:
        print('AUDIT FAILED', mismatch, file=sys.stderr); sys.exit(1)
    print(f'audit ok: {len(sample)} sessions matched')
```

跑失败就退非零、cron 邮件告警。这套抽样 + 直连上游对账,**等价于在数据流里随机插了一个不被你的代码影响的"探针"**——它能发现自检面板永远发现不了的稳定性偏差。

### 4.5 准确性的"3 道防线"心智模型

| 第 1 道:进数前 | 第 2 道:缓存层 | 第 3 道:呈现前 |
|---|---|---|
| 哨兵字段(COUNT、CHAR_LENGTH 截断数) | 历史天 SHA256 校验,漂移即失效 | 顶部健康度面板(用户自己看) |
| 分片 key 来自上游(不本地推算) | 当天永不缓存 | 跨源 PV/UV 交叉验证 |
| 长字段补全 + 长度断言 | 原子写(避免读到半截) | 每周抽样直查上游对账 |

> **真正的准确性是被持续验证的状态,不是一次性把代码写对。** 把每个环节的"我以为它对"换成"我有办法证明它对"——这就是从"能跑"到"能托付"的全部距离。

---

## 5. 缓存:可重入是定时任务的命门

缓存是性能优化,更是**可重入性**。把它做对,cron 任务才敢放心定时跑。

| 没缓存的样子 | 按天缓存后 |
|---|---|
| 每次跑全部 14 天 → 12 分钟 | 常态运行 24 秒(只查今天) |
| 查到一半挂了 → 一切重来 | 挂哪一天补哪一天 |
| 调试改一行代码 → 再等 12 分钟 | 调试瞬间出结果 |
| 风控随机拦截 → 全流程脆弱 | 风控偶发不影响整体 |

### 三条缓存设计准则

1. **"今天"永远不命中缓存。** 当天数据还在写,缓存了就锁死了。
2. **缓存文件名 = 缓存语义。** `<dataset>_2026-06-04.csv` 一眼看出来源、日期、格式,删起来也不慌。
3. **原子写入。** 先写 `.tmp` 再 `os.replace`。否则 cron 半路被杀,下次读到半截 JSON 就懵了。

**原子写入模板(任何"输出文件"都该这样写):**

```python
tmp = output_path + ".tmp"
with open(tmp, "w") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
os.replace(tmp, output_path)   # POSIX 原子操作,读端不会读到半截
```

> 💡 **小技巧**:给缓存目录加 `du -sh cache/` 监控。看到尺寸暴涨知道有数据膨胀;看到没增长知道 cron 没在跑——这是免费的健康检查。

---

## 6. 看板:单文件 HTML 的极简哲学

最终产物是一个**单文件 HTML**——50MB,包含全部数据 + 全部 JS + 全部样式。没有后端、没有数据库、没有登录态。

### 为什么不做成 Web 服务?

| 方案 | 开发成本 | 运维成本 | 分享成本 |
|---|---|---|---|
| Flask + 数据库 + 前端框架 | 高(鉴权/部署/接口) | 持续 | 需要发链接 + 解释怎么登录 |
| Streamlit / Dash | 中 | 持续 | 同上 |
| **单文件 HTML(本方案)** | 低(一个 Python 脚本拼出来) | ≈ 0 | 发 URL,别人点开就能看;甚至能下载到本地离线打开 |

### 实现方法

HTML 模板里留一个占位符 `<!--DATA-->`,Python 在最后一步把全量 JSON 注入进去:

**Python 端:把数据塞进模板**

```python
json_out = json.dumps(data, ensure_ascii=False, separators=(",", ":"))
json_out = json_out.replace("</", "<\\/")  # 防 XSS:避免 </script> 闭合

script_block = f"<script>\nR={json_out};\n{init_code}\n</script>"
html = template.replace("<!--DATA-->", script_block)
```

**HTML 端:所有变量从全局 R 拿**

```javascript
// R 由 Python 注入,里面有 p1/p2/allDepts/agentDash 等
function rdt(data){
    var m={};
    data.forEach(function(r){
        if(!m[r.dept]) m[r.dept]={u:new Set(), s:new Set(), g:0};
        m[r.dept].u.add(r.email);
        m[r.dept].s.add(r.sid);
        m[r.dept].g += r.q;
    });
    // ... 渲染表格
}
```

> ⚠️ **关键 trick**:`json.dumps(...).replace("</", "<\\/")` —— 否则消息里如果有 `</script>` 字符串就会提前关闭脚本块,前端直接崩。这是把数据塞进 inline script 的标准防护。

### 为什么 50MB 也能直接打开

- JSON 用 `separators=(",",":")` 紧凑序列化,省 30%+;
- 27k 行数据浏览器内存占用约 100MB,现代电脑无压力;
- HTTP 服务用 `python3 -m http.server 8080`,gzip 后传输只有 12MB;
- 所有筛选/排序/分页都在浏览器内做,零后端调用。

---

## 7. 筛选:所有过滤都跑在内存里

9 个筛选维度(日期、部门、Agent、来源、角色、用户名、邮箱、会话 ID、日期-小时)全部在前端跑。代码不到 30 行。

**详情页筛选函数(短到可以贴在 PPT 上):**

```javascript
function fp2(){
    var d1 = document.getElementById('p2ds').value,
        d2 = document.getElementById('p2de').value,
        rs = gc('p2rp'), dps = gc('p2dp2'),     // gc = getChecked,返回 Set 或 null
        as = gc('p2ap'), nv = id('p2n').toLowerCase();
    return R.p2.filter(function(r){
        if(d1 && r.d < d1) return false;
        if(d2 && r.d > d2) return false;
        if(rs && !rs.has(r.role)) return false;
        if(dps && !dps.has(r.dept)) return false;
        if(as && !as.has(r.agent)) return false;
        if(nv && r.name.toLowerCase().indexOf(nv) < 0) return false;
        // ... 其他条件
        return true;
    });
}
```

### 方法论 takeaway

- **`null` = 全选 = 不过滤**:让"未操作"和"全选"等价,用户体验上没歧义。
- **每个条件单独 `if`**:不要写一长串 `&&`。逐条 `return false` 短路,加新条件不会影响旧的。
- **Set 而不是 Array**:`rs.has(r.role)` 是 O(1),27k 行 × 9 维度 × 包含判断就是 240k 次操作,几十毫秒搞定。

> 所有筛选 UI 共用 4 个工具函数:`cdd` 创建下拉、`ub` 更新按钮文案、`fdd` 搜索过滤、`gc` 获取选中。每个筛选器声明 4 行就能用——这是组件化的最小代价。

---

## 8. 会话展示:嵌套行 + 折叠

"问题排查"页要看完整对话上下文。把消息按 `conversation_id` 聚合成会话行,点击展开看全部消息——纯 CSS 做折叠,零 JS 框架。

### 实现方式

1. 渲染时一次性输出"会话行 + 该会话的所有消息行",消息行带 `data-g="g0"` 标识;
2. 消息行默认 `display:none`,加了 `.open` 类才显示;
3. 点击会话行只是 toggle 一组 `.open` 类,没有任何重新渲染。

**CSS(核心就两行):**

```css
.mrow { display: none; background: #fafafa; }
.mrow.open { display: table-row; }
```

**JS toggle:**

```javascript
function tgs(id){
    var rows = document.querySelectorAll('tr[data-g="' + id + '"]'),
        tg   = document.getElementById('t' + id),
        open = tg.classList.contains('open');
    rows.forEach(function(r){ r.classList.toggle('open', !open); });
    tg.classList.toggle('open', !open);
}
```

> 💡 **小心翻页和折叠状态。** 翻页时所有展开状态会丢——这是预期行为。如果用户希望"翻页保留展开",就得改用 ID-based 的全局 state(成本骤增)。多数场景下"翻页即重置"是更可控的选择。

---

## 9. 定时:拆分本地与服务器,各干其能

最初版本:本机全包。结果就是要 7×24 开机连 VPN 才行。重构后:**本机只做开发机做不到的事**。

| 重构前(本机全包) | 重构后(拆分职责) |
|---|---|
| 本机 cron 调 `generate_report.py` | 本机 cron 只跑 `scrape_agents.py` → scp `agents.json` |
| 本机查数据网关,本机抓 4 个 Agent | 开发机 cron 跑主任务,读本机推来的 JSON |
| 本机生成 50MB HTML,scp 推送 | 开发机内网直连数据网关,更快 |
| 耗时 100s+,必须开机+连 VPN | 本机部分 35s,开发机部分 24s |
| 晚上电脑合盖就漏跑 | 开发机 7×24,本机偶尔漏跑也不影响主链路 |

### 关键决策

重构时差点踩了一个大坑:开发机访问不了 Playwright 的 CDN,`playwright install chromium` 卡死 24 分钟没动。这时不能"硬扛"——直接接受现实:

> ⚠️ **"开发机装不上 chromium → 让本机来抓"是一个被现实倒逼的设计。**
> 很多时候架构不是规划出来的,是被环境约束*挤*出来的——重要的是**识别约束并诚实接受**,而不是和环境硬刚。

### cron 错峰:避免抢占

本机推送在每小时 `:10`,开发机生成在 `:12`,留 2 分钟给 SSH 完成。如果同步 `:10` 撞分钟,scp 还在传开发机就开跑——读到的是上一个小时的旧数据。

| 机器 | Cron | 动作 | 耗时 |
|---|---|---|---|
| 本机 macOS | `10 9-23 * * *` | 抓 Agent → scp agents.json | ~35s |
| 开发机 Linux | `12 9-23 * * *` | 查 + 渲染 + 写到 HTTP 目录 | ~24s |

### cron 脚本三件套(直接抄就能用)

1. **网络可达性预检**:`curl --max-time 5` 探一下数据源,挂了直接跳过本次(不要试图重试,下次自然会再触发);
2. **新鲜度检查**:看上游文件的 mtime,太老就在日志里告警,但不阻断(数据老总比没数据强);
3. **顺手保活下游**:HTTP 服务挂了就自动 `nohup` 拉起来——把多个职责合并到一个 cron,少一个故障点。

```bash
# 网络可达性
code=$(curl -sS --max-time 5 -o /dev/null -w "%{http_code}" https://<data-gateway>/ || echo "000")
[ "$code" = "000" ] && { echo "[$(date)] 不可达,跳过"; exit 0; }

# 新鲜度
age_min=$(( ( $(date +%s) - $(stat -c %Y "$AGENTS_JSON") ) / 60 ))
[ "$age_min" -gt 120 ] && echo "警告:agents.json 已 ${age_min} 分钟未更新"

# 顺手保活 HTTP 服务
curl -sS --max-time 3 -o /dev/null http://127.0.0.1:8080/ \
  || nohup python3 -m http.server 8080 --bind 0.0.0.0 &
```

---

## 10. 部署:tar over ssh + python http.server

没有 nginx、没有 docker、没有 CI/CD。两条命令搞定文件同步,一行命令搞定 HTTP 服务。

### 同步代码:rsync 不在?用 tar over ssh

```bash
# 一行同步 scripts + templates + cache 三个目录到开发机
tar czf - scripts templates cache | ssh <user>@<dev-server> 'cd ~/data-report && tar xzf -'
```

这条命令的妙处:**压缩 + 网络传输 + 解压三步管道流式进行**,不落临时文件,比 rsync 还简单。在没装 rsync 的目标机上是最优解。

### HTTP 服务:一行起

```bash
# 在产物目录下起一个 8080 端口的静态服务
cd /srv/report && nohup python3 -m http.server 8080 --bind 0.0.0.0 &
```

对于"内网分享一个静态报告"这个场景,python 自带的 `http.server` 完全够用——支持 Range 请求、能处理几 MB 的并发、还自带访问日志。引入 nginx 是 over-engineering。

> 💡 **路径设计**:用 `/<project>/index.html` 而不是 `/report.html`。一来 `index.html` 让 URL 更短(直接访问目录),二来后续可以并存多个数据集(`/project-a/`、`/project-b/`...)共用同一个 HTTP 服务。

---

## 11. 复盘:踩过的坑与权衡

真正贵的经验不是"做对了什么",是"做错过什么"。

| 坑 | 表现 | 解法 / 取舍 | 状态 |
|---|---|---|---|
| SQL 一次取太多被风控 | HTTP 502 / 数据被截断 | 按天分批 + 缓存 | ✅ 已解 |
| 消息内容超 1800 字符被截断 | "问题排查"页看不到完整对话 | SUBSTRING 分段补全 + 按天 ext.json 缓存 | ✅ 已解 |
| 开发机装不上 chromium | playwright install 卡 20 分钟 | 放弃远程装,本机抓 → JSON → scp | ✅ 已解 |
| 本机合盖漏跑 cron | 报告几小时不更新 | 主任务迁开发机,本机只跑非关键的 Agent 抓取 | ✅ 已解 |
| scp 传到一半被读 | 开发机解析 agents.json 失败 | 原子写:本机先写 .tmp 再 rename,scp 一次性整文件传输 | ✅ 已解 |
| cron 同分钟撞车 | 读到旧数据 | 错峰 2 分钟(本机 :10,开发机 :12) | ✅ 已解 |
| JSON 嵌入 HTML 含 `</script>` | 页面解析提前结束,整个看板黑屏 | `.replace("</", "<\\/")` | ✅ 已解 |
| 50MB 单文件首次加载略慢 | 用户首次打开等 5 秒 | HTTP 自带 gzip 后 ≈12MB;可接受 | ⚪ 未优化 |
| 没有用户级别访问控制 | 内网任何人都能看到全部对话内容 | 可接受(团队工具 + 内网)。如需收紧:换 nginx + basic auth | ⚪ 按需 |

---

## 12. 复用清单:换个数据集要改什么

如果你想拿这套架构做自己团队的数据看板,按下面这张表逐项替换即可。

| 位置 | 改什么 | 难度 |
|---|---|---|
| `generate_report.py` 顶部 | 数据源 URL、Token、数据集表名(`datasource_table`) | ★ |
| `_query_one_day` 里的 SQL | 替换为你的字段列表与表达式 | ★ |
| `COL_*` 常量 | 映射成 CSV 列名(中文/英文都行) | ★ |
| `parse_csv` 里的 p1/p2 构造 | 决定哪些字段进汇总(p1)哪些进明细(p2) | ★★ |
| `templates/report.html` 顶部 H1 | 报告标题 + 默认日期范围 | ★ |
| `templates/report.html` 筛选区 | 按你的字段调整下拉选项数量 | ★★ |
| `rdt` / `rat` / `rut` 三个汇总函数 | 改成你想看的"分什么 × 算什么" | ★★ |
| `scrape_agents.py` | 如果不需要外部抓取,直接删;否则改 URL 列表 + 解析正则 | ★★ |
| cron 表达式 | 按你的更新频率改(建议偶发分钟数避开 :00) | ★ |
| 部署目标 | 开发机 IP + 路径 | ★ |

### 30 分钟启动模板

1. **第 1 步:跑通查询。** 把你的 SQL 跑一遍,输出一份 CSV,确认字段名。
2. **第 2 步:复用 generate_report.py。** 替换 SQL + COL_* 常量。本地跑一次,确认产物 HTML 能打开。
3. **第 3 步:调模板里的 H1 和默认筛选项。** 改 HTML 里的标题,运行确认日期范围合理。
4. **第 4 步:先手动 scp 一次到目标机。** 确认 HTTP 服务能服务这个文件(curl 看 HTTP 200)。
5. **第 5 步:装 cron。** 用上面给的脚本三件套,先跑一次验证。
6. **第 6 步:分享 URL。** 整个过程通常 30 分钟内完成。

---

## 13. 一句话总结

> 把昂贵的查询缓存住、把数据和视图分开、把客户端能干的不放服务端、让两台机器各干其能、让定时任务幂等到挂了无所谓——剩下的**不是工程问题,是审美问题**。

这套东西没有一个"现代框架"。它的全部分量:~500 行 Python、~190 行 HTML、~50 行 shell。但它每天 9 点到 23 点准时为团队提供一份能筛选、能下钻、能分享的数据看板——**这就是软件工程的一种克制美学**。

---

*本文为方法论提炼,可自由复用 / 改写。架构、代码片段、cron 模板均已脱敏,可直接套用到你自己的数据集。*
