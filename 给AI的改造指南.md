# 做好人买好股 · 打分尺 —— 给 AI 的改造指南

> **怎么用这份文件**：把它整个丢给你的 AI（DeepSeek / 豆包 / ChatGPT / Claude / WorkBuddy 都行），
> 加一句「按这份指南把 `做好人买好股打分尺.html` 改成在线版」。
> 里面每一个接口、字段、公式都是实测过的，AI 不需要再去猜。

---

## 0. 你手里是什么

一个**离线快照**，不是实时工具：

- 单文件 HTML（约 120 KB），双击就开，不联网、不装环境
- 内置 9 只标的的完整数据，截止 **2026-10-30 收盘**
- 搜索框只能查这 9 只，查别的会提示收录范围

它是一张**样张**——让你看清"这把尺子怎么量"，不是"现在该买什么"。
数据一旧，分数就会骗人。这份指南就是教你的 AI 把它变成能自己取数的版本。

---

## 1. 任务：三选一

| 路线 | 做成什么 | 适合谁 | 难度 |
|---|---|---|---|
| **A 本地实时版** | 自己电脑跑个小服务，实时取数 | 会开命令行、只给自己用 | ★★☆ |
| **B 纯前端在线版** | 一个网页，打开就是实时数据，能发给别人 | 想分享出去 | ★★★ |
| **C 云端代理版** | 页面 + 一个转发服务，规避跨域 | 想上线给很多人用 | ★★★★ |

**推荐 B**。原因见第 3 节：腾讯行情和 K 线接口都开放了跨域，东财财务支持 JSONP，
**整套数据纯浏览器就能拿全，不需要服务器**。

---

## 2. 它怎么算的（改造前必须读懂）

### 2.1 三层结构

```
数据层  →  派生层  →  打分层  →  结论层
原始行情    技术指标    四维加权    五档 + 修正
财务分红    筹码分布
```

单文件里所有数据都塞在一个 `SNAP_DATA` 数组里，每只标的长这样（字段是完整的）：

```js
{
  code, name, sector, price, change,        // 基础
  pe, pb, divYield, payout, dps,            // 估值与分红
  roe, roeAnn, profitYoy, revYoy, debtRatio,// 盈利质量
  isFinancial, isFund, marketCap, bvps,
  quote: { open, high, low, prev, volume, amount, turnover,
           liangbi, weibi, bids[], asks[], time },
  hi52, lo52, pos52, dd52,                  // 52 周位置
  ma20, ma60, ma120, ma250,                 // 均线
  ret20, ret60, ret260, volAnn,             // 收益与波动
  rsi, dif, dea, hist, macdUp, kdjJ,        // 技术
  chip: { centre, profit, above, peak, dist[60], dist_prices[60] },
  indexVal: { ... },                        // 仅 ETF：跟踪指数的估值分位
  total,                                    // 预计算总分（改造后需重算）
  trigger                                   // 预生成解读文字（改造后需重生成）
}
```

> ⚠️ **`total` 和 `trigger` 是快照时算好的死数据。** 一旦你把价格换成实时的，
> 这两个字段必须**重新计算**，否则会出现"价格是今天的、结论是上周的"这种最坏情况。
> 重算入口：`score(s)` 出分数，`buildReasons(s, sc)` 出解读文字。

### 2.2 打分模型（四维 100 分）

| 维度 | 分值 | 细项 |
|---|---|---|
| **A 好公司** | 40 | 盈利质量 15（ROE/净利同比/营收同比）· 分红能力 15（股息率/支付率/分红历史厚度）· 财务稳健 10（负债率；银行保险给中性 6 分） |
| **B 好价格** | 35 | 绝对估值 15（PE<10 / PB<1 对齐）· 股息率门槛 12（≥4% 满分）· 价格位置 8（52 周分位，跌破年线 +1） |
| **C 逆向与筹码** | 15 | 获利盘 8（越低越符合"众人抛弃"）· 上方套牢盘 4 · 集中度 3 |
| **D 技术面** | 10 | 均线结构 4 · MACD 3 · 超买超卖 3（只微调，不主导） |

**两类特殊口径（这两条最容易写错，务必保留）**：

1. **周期股**（煤炭/有色/石油/钢铁，白名单在 `CYCLICAL_CODES`）：
   绝对估值里 **PE 只给中位 4.5 分、不判方向**，改看 PB（10.5 分权重）。
   原因：煤价/铜价下跌时利润先缩，PE 被动抬高——按 PE 扣分会把"赚得少了"读成"变贵了"，方向是反的。

2. **金融股**（银行/保险，`FIN_CODES`）：
   触发"股息率陷阱"时**只降档、不扣分**。因为保险利润受投资收益与准备金折现率假设影响，
   银行受拨备节奏影响，单期 −20% 多半是会计口径，不是经营恶化。

**否决与修正（硬规则）**：

- 净利同比 < −20% 且股息率 > 4% → 标记"股息率陷阱"，**降一档**
- 支付率 > 90% → **−5 分**（分红不可持续）
- PB > 3 且 PE > 30 → **×0.7**
- 52 周位置 > 85% 且 RSI > 65 → **追高预警，无论多少分都不得给"推荐建仓"**

### 2.3 五档结论

| 个股总分 | 结论 | | 基金/ETF 总分 | 结论 |
|---:|---|---|---:|---|
| ≥75 | 推荐建仓 / 加仓 | | ≥75 | 推荐加投 |
| 62–75 | 推荐分批买入 | | 55–75 | 推荐定投 |
| 48–62 | 推荐持有不动 | | 38–55 | 推荐持有不动 |
| 35–48 | 推荐减仓 | | 22–38 | 推荐减仓 |
| <35 | 推荐只留底仓 | | <22 | 推荐只留底仓 |

个股和 ETF 是**两套尺子，不能横向比**。ETF 走"四腿"：估值分位 40 + 股息 30 + 价格位置 20 + 偏离年线 10。

---

## 3. 数据源（2026-10-31 实测，可直接抄）

### 3.1 接口清单与跨域状态

| 用途 | 接口 | 跨域 | 备注 |
|---|---|---|---|
| **实时行情**（价格/盘口/PE/PB/量比） | `https://qt.gtimg.cn/q=sh601988` | ✅ `Access-Control-Allow-Origin: *` | 返回 **GBK**，`~` 分隔 |
| **前复权日线** | `https://web.ifzq.gtimg.cn/appstock/app/fqkline/get?param=sh601988,day,,,260,qfq` | ✅ `*` | JSON，最稳的 K 线源 |
| **财务**（ROE/净利/营收/负债率） | `https://datacenter-web.eastmoney.com/api/data/v1/get?reportName=RPT_F10_FINANCE_MAINFINADATA&...` | ❌ 无 CORS | **支持 JSONP**：加 `&callback=cb` |
| **分红明细** | 同上，`reportName=RPT_SHAREBONUS_DET` | ❌ 无 CORS | 同上，加 `&callback=` |
| **代码搜索** | `https://searchapi.eastmoney.com/api/suggest/get?input=xxx&type=14&count=12` | ❌ 无 CORS | 纯前端可跳过，用手输 6 位代码 |

> 结论：**行情和 K 线可以直接 `fetch`**；财务/分红走 **JSONP**（动态插 `<script>`）绕开跨域。
> 两者都是 https，不会触发混合内容拦截。

### 3.2 腾讯行情字段（`~` 分隔后的下标）

```js
// const p = txt.split('="')[1].rstrip('";').split('~')
p[1]  名称          p[3]  现价         p[4]  昨收
p[5]  今开          p[30] 时间         p[31] 涨跌额
p[32] 涨跌幅%       p[33] 最高         p[34] 最低
p[36] 成交量(手)    p[37] 成交额(万)    p[38] 换手率
p[39] 市盈率TTM     p[43] 振幅         p[44] 流通市值
p[45] 总市值        p[46] 市净率       p[49] 量比
// 五档买 p[9..18]，五档卖 p[19..28]，成对为 (价, 量)
```

**GBK 解码（关键）**：

```js
const res = await fetch(url);
const buf = await res.arrayBuffer();
const txt = new TextDecoder('gbk').decode(buf);   // 不能直接用 res.text()
```

### 3.3 代码 → 市场前缀

```js
function splitCode(code) {
  const c = String(code).replace(/\D/g, '');
  if (c.length !== 6) throw new Error('代码必须是 6 位数字');
  const h = c[0];
  if ('659'.includes(h)) return { secid: '1.' + c, pfx: 'sh' };  // 沪市/沪ETF/沪B
  if ('0312'.includes(h)) return { secid: '0.' + c, pfx: 'sz' }; // 深市/深ETF/深B
  if ('48'.includes(h))   return { secid: '0.' + c, pfx: 'bj' }; // 北交所
  return { secid: '1.' + c, pfx: 'sh' };
}
```

---

## 4. 三条路线怎么做

### 路线 B（推荐）：纯前端在线版

改造步骤，按顺序做：

**① 把 `SNAP_DATA` 从"数据源"降级为"兜底"**

保留快照数组，但加一个标记：取数成功就用实时数据覆盖 `price / pe / pb / divYield / pos52 / ma* / rsi`
这几个关键字段，取数失败就退回快照并在页面顶部显示"实时取数失败，当前为 2026-10-30 快照"。

**② 写三件事**

```
fetchQuote(code)   → 腾讯行情 → 覆盖 price/pe/pb/hi52/lo52/量比
fetchKline(code)   → 腾讯 K 线 → 重算 ma20/60/120/250、pos52、rsi、macd、kdj、ret*
fetchFinance(code) → 东财 JSONP → 覆盖 roe/profitYoy/revYoy/debtRatio/dps
```

**③ 重算派生指标**

- `pos52 = (price - lo52) / (hi52 - lo52) * 100`（`lo52/hi52` 取近 250 日最高最低）
- `divYield = dps / price * 100`（`dps` 来自分红接口，取近 12 个月每股累计）
- RSI(14)、MACD(12,26,9)、KDJ(9,3,3) 都是标准算法
- 筹码分布用三角分布衰减模型：
  `_chips(rows, decay=0.985, nb=60)` —— 对每根日线，以当日均价为中心向 60 个价位桶做三角衰减累加，
  越近的交易日权重越高（`decay^i`）。**这只是位置参考，不进分数核心。**

**④ 调 `score(s)` 和 `buildReasons(s, sc)` 重算**——这两个函数已经在页面里，不用重写。

**⑤ 加时效标记**

页面顶部必须实时显示：`本次取数 2026-10-31 14:32:05 · 行情延迟约 15 秒`。
取数失败要显式报错，不能默默显示旧数据。

**⑥ 并发与限频**

9 只标的一次刷新 = 9×(1 行情 + 1 K线 + 2 东财) ≈ 36 个请求。
建议：`Promise.allSettled` 并发但每只间隔 200ms，失败重试 2 次（退避 0.5s / 1s）。
腾讯行情**不要 1 秒内打超过 5 次**，会被限。

---

### 路线 A：本地实时版（最稳）

适合只给自己用。改造思路：把页面里"取数"抽成一个函数，用本地 Python 服务代理。

```python
# 极简版：一个本地服务，前端 fetch 它，它去抓数据
import json, urllib.parse, urllib.request
from http.server import BaseHTTPRequestHandler, HTTPServer

def get(url, ref='https://gu.qq.com/', enc='utf-8'):
    req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0', 'Referer': ref})
    return urllib.request.urlopen(req, timeout=15).read().decode(enc, 'ignore')

class H(BaseHTTPRequestHandler):
    def do_GET(self):
        q = urllib.parse.parse_qs(urllib.parse.urlparse(self.path).query)
        code = q.get('code', [''])[0]
        # ... 按第 3 节取数、算分 ...
        body = json.dumps(result, ensure_ascii=False).encode()
        self.send_response(200)
        self.send_header('Content-Type', 'application/json; charset=utf-8')
        self.send_header('Access-Control-Allow-Origin', '*')   # 本地够用
        self.end_headers()
        self.wfile.write(body)

HTTPServer(('127.0.0.1', 8899), H).serve_forever()
```

打开 `http://127.0.0.1:8899/?code=601988` 就能拿到一只标的的完整算分结果。

---

### 路线 C：云端代理版

只在"要给很多人用、且量很大"时才需要。把路线 A 的服务换成 Cloudflare Worker：

```js
export default {
  async fetch(req) {
    const u = new URL(req.url);
    const target = u.searchParams.get('u');
    if (!target || !/^https:\/\/(qt|web\.ifzq)\.gtimg\.cn\//.test(target))
      return new Response('bad target', { status: 400 });
    const r = await fetch(target, { headers: { Referer: 'https://gu.qq.com/' } });
    return new Response(r.body, { headers: { 'Access-Control-Allow-Origin': '*' } });
  }
}
```

> 白名单必须写死，否则你的 Worker 会变成任意 URL 的开放代理。

---

## 5. 自适应：把尺子调成你自己的

这是这套模型最该被改的地方——**它的权重是"红利/价值派"的口味，不一定是你的口味**。

如果用户说"我偏成长"或"我只要股息"，按下面的参数改，不用动结构：

| 参数 | 位置 | 默认 | 调高会怎样 |
|---|---|---|---|
| `A.盈利质量` 权重 | `score()` | 15 | 高 → 更看重 ROE/增速，偏向好生意 |
| `B.股息率门槛` 权重 | `score()` | 12 | 高 → 更看重现金回报，偏向吃息 |
| `B.价格位置` 权重 | `score()` | 8 | 高 → 更看重"跌得够不够深" |
| `C.筹码/逆向` 权重 | `score()` | 15 | 高 → 更看重"没人要的时候买" |
| `D.技术` 权重 | `score()` | 10 | 建议 ≤10，别让它主导 |
| 股息率满分线 | `score()` | 4% | 调到 5% → 门槛更严，符合红利赛道 |
| 追高线 | `score()` | 52周85%/RSI65 | 调低 → 更厌恶追高 |

**三套预设，直接抄**：

```js
// 红利收息型：只认现金，其他都是配角
{ quality: 10, valuation: 10, divGate: 20, pricePos: 10, reverse: 15, tech: 5, divFull: 5.0 }

// 均衡型（默认就是这套）
{ quality: 15, valuation: 15, divGate: 12, pricePos: 8, reverse: 15, tech: 10, divFull: 4.0 }

// 逆向抄底型：越没人要越关注（但必须保留追高否决）
{ quality: 12, valuation: 18, divGate: 8, pricePos: 15, reverse: 20, tech: 7, divFull: 4.0 }
```

**改权重后必须做的一致性检查**：拿 3 只已知答案的标的跑一遍，看结论有没有跳到离谱的档位。
如果一只明显高估的股票在新权重下变成"推荐建仓"，说明权重加错了。

---

## 6. 红线：这些别改

这几条是模型踩过坑之后定下来的，改了会让结论方向性错误：

1. **周期股不能按 PE 扣分** —— 见 2.2 第 1 条。这是本项目最贵的一个修正。
2. **金融股单期利润下滑不扣分，只降档** —— 会计口径波动，不是经营恶化。
3. **追高否决是硬规则** —— 52 周 > 85% 且 RSI > 65，多少分都不给"推荐建仓"。这条不做成"扣分"。
4. **个股和 ETF 两套尺子不能合并** —— 混在一起比，结论一定错。
5. **结论只回答方向，不给金额、不给档位表** —— 买多少是仓位问题，不是这只股票的问题。
6. **筹码分布只作位置参考，不进分数核心** —— 它是衰减模型估出来的，精度有限。

---

## 7. 验收清单

改完必须逐条过，任何一条不过就不算改好：

- [ ] 输入一只**快照里没有**的代码（如 `600519`），能正常出分（这是在线版和快照版的分水岭）
- [ ] 顶部显示的取数时间，和当前时间差 < 1 分钟
- [ ] 断网后刷新，页面**明确提示"取数失败"**，而不是默默显示旧分数
- [ ] 同一只标的，实时价改动 1%，股息率跟着反向变动（`dps/price`）
- [ ] 中国神华（601088）仍走周期股口径，PE 不参与方向判定
- [ ] 中国太保（601601）净利 −20% 时只降档、不扣分
- [ ] 一只 52 周新高 + RSI > 70 的票，结论**不是**"推荐建仓"
- [ ] ETF（如 510880）走四腿模型，不和个股同表比较
- [ ] 页面在手机竖屏下不横向溢出

---

## 8. 最后一句

**这把尺子的作用是让你少犯错，不是让你多赚钱。**

它能把"我觉得它便宜"变成"它在四个维度上分别得几分"，
但分数高只说明它现在符合这套标准，不说明它明天会涨。

改造它的时候，请保留页面底部那两行字：

> 方法演示，不构成投资建议
> 股市有风险，决策请自己负责

---

*本指南随工具一同分发。工具本身是把公开流传的"好公司、好价格、股息率为锚、逆向投资"等原则做的工程化近似，
不代表任何个人的观点或推荐，与相关人士无授权、无背书、无合作关系。*
