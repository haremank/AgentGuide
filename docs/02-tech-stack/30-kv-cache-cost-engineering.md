---
type: 教程
status: 已发布
level: 高阶
topic:
  - Agent
  - 上下文工程
  - 框架工具
---

# KV-Cache 成本工程：把上下文当作预算来管理

> 生产级 Agent 与 demo 的差距,往往不在模型能力,而在单位成本。Manus 团队披露:一次典型任务中,约 90% 的 LLM token 花在重复读取同一个 prompt 前缀上。把 KV-cache 命中率当作一等公民来设计,可以将 Agent 运行成本降低一个数量级。

## 为什么是 KV-Cache

Transformer 推理时,每个 token 的注意力都要用到之前所有 token 的 Key/Value 向量。如果每次请求的前缀 token 相同,这些 K/V 可以直接复用,不必重新计算——这就是 prompt cache / KV-cache。

对用户:费用按 input token 计,缓存命中的前缀通常按 1/10 左右计价。
对延迟:复用 K/V 省去整段前缀的预填充(prefill)计算,首 token 延迟显著下降。

关键事实:**Agent 是循环调用 LLM 的系统**。第 N 轮的输入 = 第 N-1 轮输入 + 新增内容。前缀天然高度重复,这是 Agent 与普通 chat 应用最大的成本结构差异。

## 三条铁律(源自 Manus 实践)

### 铁律一:Append-Only,不要回头修改

上下文应当是只追加的序列:

- 不改历史消息(包括措辞、格式、排序)
- 不在中间插入内容
- 修改 prompt 中任何一个字符,该位置之后所有 token 的缓存全部失效

反面例子:给 system prompt 加"当前时间"字段。时间每次都变,缓存每轮全失效——正确做法是把时间放进用户消息或工具返回里。

### 铁律二:保持 prompt 前缀稳定

KV-cache 是**精确前缀匹配**,哪怕差一个 token 也断。常见破坏点:

| 破坏点 | 表现 | 修法 |
|:---|:---|:---|
| 动态 system prompt | 时间/随机 id/用户名注入前缀 | 动态内容后移到消息体 |
| 工具清单变化 | 权限导致 tools 集合每轮不同 | 固定工具集,用状态控制可用性 |
| 多轮消息重排 | 历史被 summarize 后重写 | 摘要只追加,不替换原文 |
| JSON 序列化不稳定 | 字典序/空格/键序抖动 | 序列化器固定键序与缩进 |

### 铁律三:用 Mask,不要用 Remove

工具多了以后,模型容易选错。直觉做法是"把不可用工具从列表里删掉"——但这会破坏前缀稳定性(工具定义位置之后全部失效)。

正确做法:工具定义常驻,**动态屏蔽其调用权限**(如 logits mask / 工具状态标记)。当工具集在语义上可枚举且状态机合理时,Manus 实测迭代速度可以快一个数量级。

## 上下文结构设计:稳定前缀 → 可复用尾部

把上下文切成三段:

```text
[ 稳定前缀: system prompt + 工具定义 + few-shot 示例 ]  ← 缓存命中率最高
[ 半稳定段: 历史 trace(只追加) ]
[ 动态尾部: 本轮用户输入 / 工具返回 / 注入的实时信息 ]  ← 每轮变化,不指望命中
```

设计要点:

1. **few-shot 多样性 ≠ 每轮换例子**。理想情况 few-shot 用"上下文学习反转":把真实使用记录(用户反馈过的成功/失败 case)追加进上下文,而不是手工维护一个静态示例集
2. **todo/plan 复述目标**:把当前 todo 列表放在上下文尾部,既让模型对齐目标,又天然是"每轮重写的动态段",不会破坏前缀
3. **保留错误轨迹**:失败的工具调用要留在上下文里("keep the wrong turns"),模型从错误中隐式更新信念,避免重复犯错;只把不可恢复的垃圾(如过大的报错堆栈)移出

## 结构化重复 vs 自然语言变化

同样一份数据:

- 自然语言段落每轮措辞都会漂移 → 前缀失配
- 结构化(JSON/YAML 键序固定)重复出现 → token 级一致,可复用

所以 trace、状态、工具返回尽量结构化;给"人看"的叙述才用自然语言,并接受它不进缓存。

## 落地 checklist

- [ ] system prompt 与工具定义在会话生命周期内字节级不变
- [ ] 动态信息(时间/用户/环境)只出现在动态尾部
- [ ] 历史消息 append-only;压缩产物以追加块形式进入
- [ ] 工具不可用时用状态 mask,而不是从定义中删除
- [ ] 监控每一轮的 cache hit rate(不少 provider 在 usage 中返回 `cached_tokens`)
- [ ] 为不同会话类型维护独立前缀,避免共享前缀互相污染
- [ ] 成本护栏:单任务 token 上限 + 高价工具调用告警

## 与本仓其他文档的关系

- 循环与工具的组织方式:见 [Agent Harness Engineering](./27-agent-harness-engineering.md) 的七层模型
- 上下文内容取舍:见 [上下文工程:业界最佳实践精华](./11-context-engineering-practices.md)
- 系统级上下文设计:见 [上下文工程完全指南](./18-context-engineering-guide.md)

## 参考来源

- Manus 博客: Context Engineering for AI Agents (KV-cache hit rate、append-only、mask not remove)
- Anthropic Engineering: prompt caching 与 effective context engineering 系列文章
