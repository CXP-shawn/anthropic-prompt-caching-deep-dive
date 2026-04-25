# samples/cache-debug-checklist.md · cache 异常 debug 决策树

## 入口

每次请求收到响应后，从 `usage` 字段读出三个数：`input_tokens` / `cache_read_input_tokens` / `cache_creation_input_tokens`。把它们对照下面的决策树。

## 决策树

```mermaid
flowchart TD
  Start[新一轮 usage] --> Q1{cache_read > 0 ?}
  Q1 -->|是| Q2{cache_creation 与 cache_read 比例 < 5% ?}
  Q1 -->|否| F1[cache 没命中]
  Q2 -->|是| Q3{input_tokens 属于 \{1, 6\} ?}
  Q2 -->|否| W1[警告: 写入比例过高<br/>检查 messages 历史是否每轮都在变]
  Q3 -->|是| OK[健康稳态]
  Q3 -->|否| W2[警告: 断点之后塞了易变内容<br/>检查最末一条 message 是否漏打 cache_control]
  F1 --> F2{cache_creation > 0 ?}
  F2 -->|是| F3[首轮或前缀刚变更<br/>等下一轮观察]
  F2 -->|否| F4{input_tokens >= 模型阈值 ?}
  F4 -->|是| F5[早期内容含变量<br/>检查 tools/system 是否每轮都变]
  F4 -->|否| F6[阈值未达<br/>检查 prompt 总长是否 < 模型最小可缓存长度]
```

## 5 大典型故障 → 修复手册

### 故障 1: cache_read=0 且 cache_creation=0 且 input_tokens 等于 prompt 全长

**诊断**：模型最小阈值未达。

**修复**：

1. 查模型阈值（Opus 4.7=4096 / Sonnet 4.6=2048 / Sonnet 4.5=1024 / Haiku 4.5=4096）。
2. 把 tools 全部 schema 圈进 cache_control（一个完整工具定义往往 200-800 token）。
3. 把 system prompt 写完整：persona / constraints / examples / few-shot。
4. 仍不够：考虑换更小阈值的模型，或合并多个小请求。

### 故障 2: cache_read=0 但 cache_creation 持续非 0

**诊断**：cache 写入正常，但下一轮没命中——前缀每轮在变。

**修复**：

1. 打印 prompt 的 SHA-256，对比相邻两轮：第一个不同的字节在哪？
2. 常见污染源：`time.Now()` / `uuid.New()` / `os.Getpid()` / debug log / trace_id。
3. 把所有变量挪到**断点 4 之后**（让它们计入 input_tokens 桶，而不是污染 prefix）。

### 故障 3: input_tokens > 6

**诊断**：最后一个断点之后被塞了 content。

**修复**：

1. 检查最末一条 message：是否漏打 cache_control？
2. 检查 messages 数组顺序：是否在最末追加了 user 消息但没挂 cache_control？
3. 给当前轮 user/tool_result 的最后一个 content block 挂 `cache_control: { type: "ephemeral" }`。

### 故障 4: cache_read 突然清零（之前都正常）

**诊断**：5m TTL 过期，或上游对 prompt 做了改动。

**修复**：

1. 查上次请求时间：距离当前 > 5 分钟 → TTL 自然过期，下次请求会重新写入。
2. 如果业务节奏天然 > 5 分钟：把 cache_control 升级为 `{ type: "ephemeral", ttl: "1h" }`，写入溢价从 1.25× 涨到 2×、命中仍 0.1×。
3. 查 prompt 模板 git log：是否最近改过 system prompt？任何字符变动都会断链。

### 故障 5: cache_creation 在 messages 历史延长时反复出现少量数

**诊断**：正常稳态——每轮 assistant + user 两条新 message 被加入缓存链，下次复用时进入命中桶。

**修复**：不需要修复。但要确认 cache_creation 的数额与 messages 增量大致吻合（每轮新增 100-500 token 是合理的，5000+ 才异常）。

## 监控告警建议

```
metric: cache_read_ratio = cache_read / (cache_read + cache_creation + input_tokens)
alert:  当滑动 5 分钟均值 < 0.85 持续 3 分钟 → 工程介入

metric: input_tokens_value
alert:  当采样到 input_tokens 不属于 {1, 6} 集合 → 立即检查最末一条 message 的 cache_control

metric: cache_creation_ratio = cache_creation / (cache_read + cache_creation + input_tokens)
alert:  当稳态阶段（运行 > 5 分钟后）cache_creation_ratio > 0.10 → 检查 prefix 污染源
```

## 一行 curl 重现一次健康请求

```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: prompt-caching-2024-07-31" \
  -H "content-type: application/json" \
  -d @samples/request-template.json
```

返回 JSON 里的 `usage` 字段对照本目录下 `response-usage-examples.json` 检查形态。

## 与正文章节互链

- 概念问题 → [01-three-fields-explained.md](../01-three-fields-explained.md)
- input_tokens 异常 → [02-the-1-vs-6-mystery.md](../02-the-1-vs-6-mystery.md)
- cache_control 打哪里 → [03-cache-control-mechanics.md](../03-cache-control-mechanics.md)
- 阈值问题 → [04-min-cache-length-trap.md](../04-min-cache-length-trap.md)
- 真实请求长什么样 → [05-real-request-anatomy.md](../05-real-request-anatomy.md)
- 计费数学 → [06-billing-math.md](../06-billing-math.md)
- Go 实现侧 → [07-go-engineering-checklist.md](../07-go-engineering-checklist.md)
