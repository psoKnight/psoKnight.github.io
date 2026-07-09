---
layout: post
title: 'eino 逐能力核对(12)Agent Evaluation:v0.8.12 没有,但这恰好逼你把它做对'
subtitle: '第三阶段·企业级 —— 全仓 eval 零命中,离线评测 + 过程评估(RunPath)+ 在线可观测的自建闭环(系列完)'
date: 2026-07-09 08:11:00 +0800
author: Gonzo Sun
tags: [eino, Go, Agent, Evaluation, 可观测, 架构]
---

> eino「逐能力核对」系列第 12 篇,收官。第三阶段第四项 **Agent Evaluation**,结论:**❌ v0.8.12 没有评估模块**。这是全系列唯一一个「完全没有」的能力。但我想把这一篇写成一个正面的东西:**评估是区分「发布了一个 Agent」和「能持续改进一个 Agent」的那条线**,而框架的缺席恰好逼你把评估的所有权拿回自己手里——这未必是坏事。三层架构见 [第 1 篇]({% post_url 2026-07-09-eino-01-prompt %})。

## 结论:❌ v0.8.12 无评估模块

我把整包 v0.8.12 源码 grep 了 `eval` / `evaluat` / `benchmark` / `judge`,**零命中**。没有 evaluator 抽象、没有数据集运行器、没有 LLM-as-judge 内建、没有指标聚合。**评估在 v0.8.12 是彻底的应用层责任。**

这不丢人——评估本身高度业务相关,框架给不了通用答案。但资料里若写「eino 支持 Agent 评估」,默认它指的是「你可以自己接」,而不是开箱即用。

## 问题挑战:Agent 的失败是静默的、且发生在过程里

为什么 Agent 尤其需要评估?因为它的失败模式,比单次 LLM 调用恶劣得多。

单次调用还能靠肉眼看输出。但 Agent 是**多轮、带工具、带分支、带多智能体路由**的系统([第 6 篇]({% post_url 2026-07-09-eino-06-react-agent %})、[第 9 篇]({% post_url 2026-07-09-eino-09-multi-agent %})),它有两个致命特性:

- **失败静默**(呼应 [第 3 篇 RAG]({% post_url 2026-07-09-eino-03-rag %}) 那条教训):一个 prompt 改动可能在第 5 轮才翻车,而模型会语气自信地给出错误结果,不抛异常、不报错。你不主动测,就不知道它坏了。
- **错在过程,不只在结果**:最终答案对,不代表过程对——它可能绕了一大圈、调错了工具、路由到了错的子 Agent,只是碰巧蒙对。只看最终答案的评估,会漏掉这些「对的答案,错的过程」,而错的过程迟早在别的输入上暴露成错的答案。

所以 Agent 评估必须回答两个问题:**结果对不对**,以及**过程对不对**。后者是 Agent 相比单模型评估多出来的、也是最容易被忽略的维度。

## 架构设计:离线 + 在线的评估闭环

自建评估不是写几个 assert,是搭一个闭环:

```mermaid
graph LR
    DS[离线数据集] --> RUN[跑测 Agent]
    RUN --> JUDGE[打分: 结果 + 过程]
    JUDGE --> BASE[基线/回归看板]
    PROD[线上流量] -->|callbacks + RunPath| OBS[在线可观测]
    OBS -->|坏 case 回流| DS
    BASE --> SHIP[决定能否发布]
    style DS fill:#e1f5ff
    style OBS fill:#fff3e0
```

左边离线(上线前拦回归),右边在线(上线后抓真实坏 case),坏 case 回流成新的离线用例——闭环转起来,Agent 才是可持续改进的,而不是发布即冻结。

## 核心实现:离线评测三种打分

标准结构是数据集 + 跑测 + 打分:

```go
type Case struct {
	Input    string
	Expected string // 或一组断言/评分标准
}

func runEval(ctx context.Context, agent adk.Agent, cases []Case) {
	for _, c := range cases {
		iter := agent.Run(ctx, &adk.AgentInput{
			Messages: []*schema.Message{schema.UserMessage(c.Input)},
		})
		got, path := collectResult(iter) // 收集最终回复 + RunPath

		// 三种打分,按场景组合:
		score := judgeByLLM(ctx, c.Input, c.Expected, got) // ① LLM-as-judge:开放式回答
		_ = exactOrRuleAssert(c.Expected, got)             // ② 规则断言:确定性任务
		_ = judgeProcess(path, c)                          // ③ 过程评估:路径/工具序列对不对
		record(c, got, path, score)
	}
}
```

- **① LLM-as-judge**:用一个模型给「回复 vs 期望」打分,适合没有唯一正解的开放式回答。
- **② 规则断言**:精确匹配、关键词、工具调用序列比对,适合确定性任务。
- **③ 过程评估**:这是 Agent 评估的灵魂。借 [第 11 篇 Runtime]({% post_url 2026-07-09-eino-11-runtime %}) 的 `AgentEvent.RunPath`,检查**路径对不对**——多智能体是否走对了子 Agent、是否调了该调的工具、循环了几圈([第 6 篇]({% post_url 2026-07-09-eino-06-react-agent %}) 的圈数)。这一维,是纯结果评估永远看不到的。

## 生产实践:在线可观测兜底

线上没有「期望答案」,靠可观测兜底。两个抓手,都来自前面的篇章:

- **compose callbacks**:[第 5 篇]({% post_url 2026-07-09-eino-05-compose %}) 的编排层支持回调,在每个节点前后埋点——记录延迟、token、工具调用、错误率,打到 APM/日志。
- **AgentEvent 流 + RunPath**:[第 11 篇]({% post_url 2026-07-09-eino-11-runtime %}) 的事件流是天然的 trace 源。把每次运行的路径、每步耗时落库,做「哪条链路慢、哪个子 Agent 常出错」的分析。

线上采到的坏 case,回流成离线数据集的新用例——闭环合上。

其余几条硬经验:

- **上线前就要有基线数据集**:Agent 的回归极隐蔽,别等出事才做评估。哪怕只有 20 个用例,也远胜于零——它是你改 prompt、换模型时唯一的安全网。
- **LLM-as-judge 要锁死裁判**:换了裁判模型或裁判 prompt,分数就漂移,前后不可比。judge 的模型和 prompt 必须版本锁定,当代码管。
- **别全自己造轮子**:把 eino Agent 包成一个 `func(input) → (output, RunPath)` 的黑盒,对接成熟的外部评测框架/你司评测平台,评测逻辑交出去,eino 这边只提供稳定入口 + 结构化事件流。
- **过程比结果更能定位问题**:最终答案错了,`RunPath` 告诉你错在哪一步、哪个子 Agent。只看最终答案,等于放弃了 Agent 评估最大的优势。

## 系列收官:十二篇的成绩单

评估这一篇没有 API 可讲,但它是整个系列逻辑上的终点——**你把前十一项能力都用上、拼成一个 Agent 之后,靠评估来确认它到底行不行、以及怎么变得更行**。回看整张成绩单:

| 阶段 | 能力 | v0.8.12 结论 |
|---|---|---|
| 一·掌握 | [Prompt]({% post_url 2026-07-09-eino-01-prompt %}) / [Function Calling]({% post_url 2026-07-09-eino-02-function-calling %}) / [RAG]({% post_url 2026-07-09-eino-03-rag %}) / [Embedding]({% post_url 2026-07-09-eino-04-embedding %}) | ✅ 全一等 |
| 二·学习 | [compose]({% post_url 2026-07-09-eino-05-compose %}) ✅ / [ReAct]({% post_url 2026-07-09-eino-06-react-agent %}) ✅ / [MCP]({% post_url 2026-07-09-eino-07-mcp %}) ⚠️ / [Memory]({% post_url 2026-07-09-eino-08-memory %}) ⚠️ | 两一等两非一等 |
| 三·企业 | [多智能体]({% post_url 2026-07-09-eino-09-multi-agent %}) ✅ / [Skill]({% post_url 2026-07-09-eino-10-skill %}) ✅ / [Runtime]({% post_url 2026-07-09-eino-11-runtime %}) ✅ / **Evaluation** ❌ | 三一等一缺失 |

一句话总结 eino v0.8.12:**编排(compose)最硬核,Agent 三范式 + Skill 齐全且真对标 Anthropic,中断恢复让它能上生产;但 MCP 在 eino-ext、Memory 非一等、Evaluation 完全没有——这三处是选型时要自己补的坑。**

而贯穿全系列、比任何单个 API 都更值得记住的,是那几条反复出现的主线:**一切收敛到 `[]*schema.Message`;流与非流靠 concat/box/merge/copy 自动缝合;上下文是有限预算,记忆和 Skill 都在管理它;模型输出是不可信输入;Agent 是会烧钱的循环;凡是让模型「选一个」的地方,命运都押在 Description 上。** 把这些内化了,你用的就不只是 eino,而是任何一个严肃的 Agent 框架。

核对完毕。感谢一路读到这里。(完)

> **系列导航 · 逐能力核对**
> 第一阶段·掌握:[Prompt]({% post_url 2026-07-09-eino-01-prompt %}) · [Function Calling]({% post_url 2026-07-09-eino-02-function-calling %}) · [RAG]({% post_url 2026-07-09-eino-03-rag %}) · [Embedding]({% post_url 2026-07-09-eino-04-embedding %})
> 第二阶段·学习:[compose]({% post_url 2026-07-09-eino-05-compose %}) · [ReAct]({% post_url 2026-07-09-eino-06-react-agent %}) · [MCP]({% post_url 2026-07-09-eino-07-mcp %}) · [Memory]({% post_url 2026-07-09-eino-08-memory %})
> 第三阶段·企业级:[多智能体]({% post_url 2026-07-09-eino-09-multi-agent %}) · [Skill]({% post_url 2026-07-09-eino-10-skill %}) · [Runtime]({% post_url 2026-07-09-eino-11-runtime %}) · **Evaluation(本篇)**
