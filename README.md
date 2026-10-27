# MiniMax-M3 on Terminal-Bench 2.0

五种 harness 的逐题结果、压缩观测和模型 token 用量快照。
每组计划 89 题；压缩触发阈值 64000 tokens，agent 超时 1200 秒。
快照时间、各组运行标识和固定的 benchmark commit 见 `metadata.json`。
这是每题当前保留的一次结果，不是多随机种子平均成绩，也不是官方排行榜提交。

## 文件

| 文件 | 内容 |
|---|---|
| `task-results.csv` | 每行一个 harness × task，包含未计分和未开始的题目 |
| `compression-events.csv` | 每行一次实际观测的压缩事件，保留所属任务和 attempt |
| `harness-summary.csv` | 各组完成情况、正确率和 token 总量 |
| `metadata.json` | 快照时间、数据来源的相对路径与 SHA-256、版本信息 |
| `SHA256SUMS` | 发布文件校验和 |

CSV 使用 UTF-8。空单元格表示没有观测值；JSON 数组中的 `null` 表示该次事件缺少对应值。
没有发生压缩时，事件数是 `0`，前后 token 数组是 `[]`。

## 正确与错误

- `label=correct`、`success=1`：结果状态为 `completed` 且评分正确。
- `label=incorrect`、`success=0`：结果状态为 `completed` 且评分错误。
- `label=unscored`、`success` 为空：未完成、跳过、执行错误或未开始；不算答错。
- `status` 保留结果状态；缺少结果但已有请求记录时标记为 `in_progress`。
- `in_five_harness_scored_intersection=True`：五组均有有效评分的共同题目，适合比较正确率。

汇总正确率的分母是已计分题数，不是 89。各组完成进度可能不同。

## Token 用量

`task-results.csv` 的 token 来自本地代理记录的 provider usage，按请求 ID 去重，
包括该任务所有可见 attempt、执行调用、摘要调用及重试；不只统计最终成功的一次 attempt。
不包含代理未捕获的调用、独立 judge token、工具本身计算和 GPU 时间。
本组使用 Terminal 的离线 verifier，不将 verifier 的执行成本视为模型 token。

| 字段 | 定义 |
|---|---|
| `uncached_input_tokens` | 未命中缓存的输入 token |
| `cache_read_input_tokens` | 缓存读取 token |
| `cache_creation_input_tokens` | 缓存写入 token |
| `total_input_tokens` | 上述三种互斥输入字段之和，包含缓存命中 |
| `total_output_tokens` | provider 报告的输出 token；不额外重复加 reasoning token |
| `total_tokens` | `total_input_tokens + total_output_tokens` |
| `model_requests` | 去重后的生成请求数，不含 count_tokens |
| `accounting_complete` | 至少有一次请求，且每次都有完整结束记录及输入/输出 usage |
| `usage_missing_requests` | 缺少输入/输出 usage 的请求数 |
| `unfinished_requests` | 没有结束记录的请求数 |
| `incomplete_streams_with_usage` | 有 usage 但流未完整结束的请求数 |

代理已将 Responses 的输入和缓存字段转换为互斥计数，导出时不会再次扣除缓存。
`accounting_complete=False` 时，token 总量只是已观测用量，可能是下界，不代表完整消耗。
没有请求时 token 字段留空。汇总文件的 token 是全部题目已观测用量之和，
包含未计分任务；不要直接拿各组总量比较效率，应按共同题集筛选，并检查用量完整性。
Token 数不等于账单金额。Codex、DSH 的历史运行存在方舟到 MiniMax 官方 API 的切换，
用量跨接口累加；这些轨迹的差异不能完全归因于 harness。

## 压缩前后 token

`compression_before_tokens` 和 `compression_after_tokens` 是按事件顺序对齐的 JSON 数组，
完整事件详情在 `compression-events.csv`。`compaction_pairs_available` 是前后均有边界计数的事件数。
这些数表示上下文大小，与一次实验累计消耗的 token 不同；不要将多次压缩的减少量当成实际计费节省量。

| Harness | `before_tokens` | `after_tokens` |
|---|---|---|
| Claude Code | `compact_metadata.pre_tokens` | `compact_metadata.post_tokens` |
| ZCode | `payload.preCompactTokenCount` | `payload.truePostCompactTokenCount` |
| Pi | `tokensBefore` | 原生日志没有直接对应字段，留空 |
| Codex | 原生日志没有直接对应字段，留空 | 原生日志没有直接对应字段，留空 |
| DSH | 原生日志没有完整上下文边界字段，留空 | 原生日志没有完整上下文边界字段，留空 |

不同 harness 的上下文计数方式可能不同；这是各自记录的计数，不保证使用同一种 tokenizer。
ZCode 使用独立的 `truePostCompactTokenCount`，不使用可能仍带旧请求计数的 `postCompactTokenCount`。

对于 Pi、Codex、DSH，额外导出压缩后下一次生成调用的 `next_request_input_tokens` 及来源字段。
这包含系统提示词、工具定义及可能新增的消息，**不能直接替代压缩完成瞬间的上下文大小**。
若下一次压缩前未观测到生成调用，该值留空。
DSH 的 `shadowed_tokens` 只表示被覆盖的历史片段大小，不是压缩前完整上下文。
`summary_input_tokens`、`summary_output_tokens` 仅在原生事件明确记录摘要调用 usage 时填写，
不是压缩前后上下文大小；这些摘要用量已包含在请求账本的任务总量内，不应再加一次。

同一 attempt 内重复保存的同一边界去重；不同 attempt 的真实压缩事件均保留。
没有原生边界记录只能说明没有观测到事件，不能据此证明完全未发生压缩。

## 数据范围与复现

发布内容仅为上述指标、标识和字段说明，不包括密钥、提示词、模型原始回复、
工具输出、任务解答、会话数据库或完整实验配置。
`metadata.json` 中保留相对来源路径和文件哈希，供持有原始实验目录者复核。
各个源文件在读取时固定字节快照；运行仍在进行时，不保证所有源文件在同一瞬间更新。
`malformed_jsonl_lines` 记录无法解析的 JSONL 行数（例如运行中尚未完成的最后一行）。

在原始项目根目录重新导出：

```bash
python3 scripts/export_minimax_terminal_public.py
```

输出目录为 `exports/minimax-terminal-bench/`。公开推送时只提交该目录内列出的发布文件。
