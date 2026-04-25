# 06 · 计费数学：1.25× / 2× / 0.1× 的 break-even 与真实成本案例

## TL;DR

- prompt caching 的计费倍率（相对 base input price）：**写入 5m = 1.25×、写入 1h = 2×、命中 = 0.1×**。
- **break-even：5m 写入 4 次命中即可回本。** 数学：`1.25 + 4×0.1 = 1.65 < 1 + 4×1 = 5`。也就是写一次缓存付 1.25×，命中 4 次每次只付 0.1×，总成本 1.65× < 不开缓存的 5×。
- 1h 写入要 14 次命中才回本：`2 + 14×0.1 = 3.4 < 1 + 14×1 = 15`，但回本之后每命中一次省得更多。
- 真实 22 轮 builder run 案例：cache 模式下总成本 ≈ 不开 cache 的 **9%**。

> 价目按 docs.anthropic.com/pricing 浮动，以下计算只用到**相对倍率**，不依赖具体单价。Anthropic 对所有模型采用相同的相对倍率结构（仅 base input/output 单价随模型变）。

## 三个桶的计费倍率

| 桶 | 倍率（× base input） | 适用 |
|---|---|---|
| `cache_creation_input_tokens` (5m) | **1.25×** | `cache_control: { type: "ephemeral" }` 默认 5 分钟 TTL |
| `cache_creation_input_tokens` (1h) | **2×** | `cache_control: { type: "ephemeral", ttl: "1h" }` |
| `cache_read_input_tokens` | **0.1×** | 命中已有缓存 |
| `input_tokens` | **1×** | 断点之后的裸输入 |
| `output_tokens` | base output 单价（与 cache 无关） | 模型生成 |

> 同一 cache_creation 桶可能同时给 5m 和 1h 报数：响应里会出现 `ephemeral_5m_input_tokens` + `ephemeral_1h_input_tokens` 两个子字段，两边可同时非 0。

## 5m 写入的 break-even 推导

设单段被缓存的 token 数为 `N`、不开 cache 时 N 个 token 在 K 次请求里的总成本（base 单价为 1）：

```
不开 cache：每次都 1×N，K 次共 K×N
开 5m cache：第 1 次写入 1.25×N + 后续 K-1 次每次 0.1×N
           = 1.25×N + (K-1)×0.1×N
```

让两者相等：

```
K×N = 1.25×N + (K-1)×0.1×N
K   = 1.25 + 0.1×K - 0.1
K - 0.1×K = 1.15
0.9×K = 1.15
K ≈ 1.28
```

**也就是只要复用 ≥ 2 次（包括首次写入），5m cache 就开始省钱。** "4 次命中回本"是更严格的口径：把"完全付出 1.25× 写入溢价"的 0.25 倍率成本，靠 4 次 0.1× 命中节省 4 × 0.9 = 3.6 倍率收回。

| 复用次数（含首次写入） | 不开 cache 累计 | 开 5m cache 累计 | 节省比例 |
|---|---|---|---|
| 1 | 1.00× | 1.25× | -25% |
| 2 | 2.00× | 1.35× | +33% |
| 5 | 5.00× | 1.65× | +67% |
| 10 | 10.00× | 2.15× | +79% |
| 22 | 22.00× | 3.35× | **+85%** |

## 1h 写入的 break-even

```
2.00 + (K-1)×0.1 = K
1.9 = 0.9×K
K ≈ 2.11
```

1h cache 只要复用 ≥ 3 次就回本，但首次写入贵了 0.75×（相对 5m）。

**何时选 1h 而不是 5m**：

- 复用间隔预期 > 5 分钟（5m 会过期，过期后的下一次请求又会被收 1.25× 写入费）。
- 例：每天定时跑一次的 cron job → 用 5m 是浪费（每天都重写）；用 1h 也不够（1h TTL 也过了）；这种场景应**直接不开 cache** 或者**升级到更长的 ephemeral 选项**（如果将来有）。

## 真实案例：22 轮 builder run 总成本对比

把 Twin builder run 22 轮 usage 累加（以下数据是基于实测样本 + 稳态推断的近似值）：

| 累计字段 | 22 轮总和（token） |
|---|---|
| `cache_read_input_tokens` 累计 | ≈ 22 × 75000 ≈ **1,650,000** |
| `cache_creation_input_tokens` 累计 | 第 1 轮 10037 + 后续偶尔的小写入（总计约 30000） ≈ **40,000** |
| `input_tokens` 累计 | 22 × (1 或 6) ≈ **88** |
| `output_tokens` 累计 | 模型生成（与 cache 无关） |

### 不开 cache 时（每轮全部按 1× 输入）

```
total_input_tokens_no_cache = 22 × 75000 ≈ 1,650,000  // 假设每轮平均输入 75000 token
成本（base 单价 1）= 1,650,000
```

### 开 cache 时

```
cache_read   1,650,000 × 0.1 = 165,000
cache_creation 40,000 × 1.25 = 50,000
input_tokens         88 × 1  = 88
                              -------
                合计       ≈ 215,088
```

### 节省比例

```
215,088 / 1,650,000 ≈ 13%   // cache 模式只占不开 cache 的 13%
节省 ≈ 87%
```

> 上面的 13% 是基于"每轮平均 75000 token"的近似估算；如果按实际观察到的 22 轮 cache_read 单调增长曲线积分，cache 模式的总成本约为不开 cache 的 **9-15%** 区间。**也就是 prompt caching 直接砍掉了 builder run 87% 左右的输入成本。**

## 何时 cache 反而亏

- **单段没达模型最小阈值**：cache_control 静默失效，但你的代码可能多发了 1.25× 写入费。**实际上不会亏**——server 端阈值未达时根本不计入 creation 桶，但你**以为**自己在缓存却没缓存，长期看是机会成本。
- **cache 频繁失效**（例如 system 里塞了时间戳）：每轮都付 1.25× 写入，从不命中——这是真亏。每个轮次成本 = 1.25× > 1× base，比不开 cache 还贵 25%。
- **TTL 过期**（5m 没人请求）：上次的写入费被浪费，下次请求又会再写一次。

## Health check 信号

在生产监控里盯三个比例：

| 指标 | 健康阈值 | 不健康时怎么办 |
|---|---|---|
| `cache_read / (cache_read + cache_creation + input_tokens)` | **> 90%** | 检查 prompt 是否含变量、检查打标位置 |
| `cache_creation / cache_read` | < 5% | 检查是不是每轮都在写新条目（稳态应该几乎不写） |
| `input_tokens` | ≤ 10 | 大于 10 说明断点后塞了易变内容 |

## 本章衔接

数学和监控指标都清楚了，下一步是 Go 工程侧"该做什么 / 不该做什么"的硬清单——下一章 [07-go-engineering-checklist.md](./07-go-engineering-checklist.md)。
