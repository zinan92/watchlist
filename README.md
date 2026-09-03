# watchlist

**Park Exposure Registry** — 唯一权威名单。行情/K 线侧与新闻 exposure 侧都从这里取，不各自维护副本。

- 数据：[`watchlist.yaml`](watchlist.yaml)
- 可视化：[`index.html`](index.html) · 三层归属树
- 当前版本：v6（2026-09-03）· 16 个资产 · 6 条宏观主线 · 26 个中观赛道 · 122 个 target

---

## 为什么存在

两件事长期被混为一谈，代价是信息丢失：

| | 回答的问题 | 内容 |
|---|---|---|
| `assets` | 我看什么价格 | 16 个有 K 线/行情的对象 |
| `targets` | 这条新闻在讲什么 | 公司、主题、宏观事件 |

早期的 exposure 过滤器要求每条新闻必须映射到 16 个资产之一才算相关，结果是英伟达财报和美联储决议都被压成 `QQQ`，入库后无法区分，也无法回答「给我看所有跟英伟达有关的新闻」。更糟的是不点名资产的新闻被整条丢弃——实测某个 24 小时窗口里 73.8% 的实时新闻被判 `unmatched`，其中包括「沙特船只在霍尔木兹海峡事故」（持有 WTI）、「OpenAI 将开发人形机器人」（AI 是四条主线之首）。

这个 registry 把两者分开：资产是资产，target 是 target，一条新闻可以同时命中多个 target 而不丢失任何一个的身份。

---

## 结构

```
root
└── macro        6 条宏观主线
    └── sector   24 个中观赛道
        └── target  109 个微观标的
```

### target 类型

| type | 例子 | 是否有市场代码 |
|---|---|---|
| `asset` | BTC、黄金、WTI | 有 |
| `company` | NVDA、长鑫科技(688825)、OpenAI | 上市有，未上市无 |
| `theme` | 光通信、具身智能、创新药 | 无 |
| `macro` | 美联储、霍尔木兹海峡、俄乌战争 | 无 |

### 字段

```yaml
- type: company           # asset | company | theme | macro
  id: "688825"            # 上市公司用市场代码，其余用 slug
  name: 长鑫科技
  market: CN              # US | CN | KR | JP | FX | CRYPTO | COMDTY
  listed: true            # 未上市为 false，且不得有 id 形式的市场代码
  reason: "..."           # 必填：它为什么在表里
  links_assets: [STAR50]  # 可选：与哪些 assets 有直接传导关系
  listed_at: "2026-07-27" # 可选
```

新闻匹配所需的代理词也在 `watchlist.yaml` 的 `matching.aliases` 中，直接引用 registry target；它只决定新闻是否值得送进 AI triage，不决定 High / Watch / Noise / Unknown。

---

## 四条硬规则

1. **每个 target 必须有 `reason`。** 写明它为什么在表里（持仓 / thesis / 研究兴趣）。无 reason 不入表——这是防止列表随时间腐烂的唯一机制。
2. **未上市公司 `listed: false`，绝不编造市场代码。** 当前未上市：OpenAI、Anthropic、GitHub、SpaceX、Figure、Hyperliquid。
3. **允许多归属。** 一个 target 可出现在多个 sector 下（如 TSM 同属晶圆制造与其他链条）。强行单亲会丢信息。
4. **命中 target ≠ 重要。** 这一层只回答「这条新闻值不值得让 AI 看」；High / Watch / Noise 的判断是下游 triage 的事。

---

## 怎么引用

```bash
# raw 直取（永久可用，随 commit 固定）
curl -s https://raw.githubusercontent.com/zinan92/watchlist/main/watchlist.yaml

# 锁定某个版本（推荐给生产系统，避免上游改动导致行为漂移）
curl -s https://raw.githubusercontent.com/zinan92/watchlist/<commit-sha>/watchlist.yaml
```

```python
import yaml, urllib.request

URL = "https://raw.githubusercontent.com/zinan92/watchlist/main/watchlist.yaml"
reg = yaml.safe_load(urllib.request.urlopen(URL))

# 展平成 target -> (sector, macro) 查找表
lookup = {}
for sector in reg["sectors"]:
    for t in sector["targets"]:
        lookup[t["id"]] = {
            "name":   t.get("name", t["id"]),
            "type":   t["type"],
            "listed": t.get("listed"),
            "sector": sector["name"],
            "macro":  sector["macro"],
        }

# 行情侧只要有 K 线的对象
tradable = [a["id"] for a in reg["assets"]]
```

生产系统建议 pin 到具体 commit，而不是跟随 `main`——名单变更应当是一次显式的升级动作，不是某天早上突然生效的意外。

---

## 变更流程

1. 改 `watchlist.yaml`，新增条目必须带 `reason`
2. 校验：`python3 -c "import yaml; yaml.safe_load(open('watchlist.yaml'))"`
3. `index.html` 同步更新（它是这份数据的人类可读视图）
4. 提交时说明改了什么、为什么

---

## 版本历史

| 版本 | 日期 | 变更 |
|---|---|---|
| v4 | 2026-09-03 | 删除「出口管制与台海」赛道（INTC/华虹并入晶圆制造）；AI 电力拆为「AI 发电」与「电力设备与线缆」 |
| v5 | 2026-09-03 | 新增 MiniMax、智谱、GitHub、腾讯、阿里巴巴、拼多多；新增「大消费与平台」赛道；确认 Robinhood 已在册 |
| v6 | 2026-09-03 | 新增 AI发展、AI算力、电子元件、AI发电、电力设备与线缆、地缘冲突、大消费与平台主题节点及新闻 aliases |
| v3 | 2026-09-03 | 长鑫科技 688825 单独成节点；宇树科技 688836 归位；核实两家上市状态 |
| v2 | 2026-09-03 | 每赛道补齐美股 + A 股标杆；新增俄乌战争、创新药、商业航天、机器人、AI 电力 |
| v1 | 2026-09-03 | 三层结构首版：5 宏观 / 18 赛道 |
