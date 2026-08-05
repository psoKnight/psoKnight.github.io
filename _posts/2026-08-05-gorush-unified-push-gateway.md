---
layout: post
title: 'gorush:把三家推送厂商收敛成一个入口,但「已受理」绝不等于「已送达」'
subtitle: '从 APNs / FCM / HMS 的协议差异出发,拆解队列、Worker、反馈与可观测性,把推送网关当成一条需要闭环的异步链路'
date: 2026-08-05 08:00:00 +0800
author: Gonzo Sun
tags: [gorush, Go, Push, APNs, FCM, HMS, 架构]
---

> 本文基于 [appleboy/gorush](https://github.com/appleboy/gorush) 的 README 与 `v1.18.9` 发布版本资料整理。结论先说：**gorush 是一个很实用的统一推送网关，适合把 iOS APNs、Android FCM、华为 HMS 收敛为一个 HTTP/gRPC 入口；但它解决的是「接入与投递」，不是「用户一定看到了」。** 如果把 HTTP `200` 当作业务送达，推送系统迟早会在静默失败里失明。

## 技术背景：推送不是一个 API，是三套外部系统

产品侧看起来只有一句话：“给用户发一条通知。”落到服务端，却至少要面对三套彼此独立的推送基础设施：

- **APNs**：iOS 的 Apple Push Notification service，证书、环境（sandbox / production）、`topic` 和 payload 都有自己的规则；
- **FCM**：Android 常用的 Firebase Cloud Messaging，通常以服务账号 JSON 建立身份；
- **HMS**：华为设备上的 Push Kit，凭据、消息体字段又是一套。

如果业务服务直接分别调用三家 SDK，平台判断、凭据加载、重试、并发、指标和失败处理会扩散到每个服务里。今天加一个活动通知，明天做订单通知，最后每处都长出一套半成品推送客户端。

gorush 的工作很明确：把这些差异收在一个用 Go + Gin 写的微服务里，对上提供统一的 REST API（也可启用 gRPC），对下适配 APNs / FCM / HMS。业务系统只要知道**目标 token、平台和业务 payload**。

## 架构设计：统一入口，分平台投递

请求进入 `/api/push` 后，不会让每个调用方去理解下游协议；gorush 按 `platform` 路由，再由队列和 Worker 消化实际发送工作：

```mermaid
graph LR
    BIZ[业务服务] -->|POST /api/push 或 gRPC| API[gorush API]
    API --> V[参数与数量校验]
    V --> Q[消息队列]
    Q --> W[可配置 Worker]
    W --> A[APNs<br/>iOS]
    W --> F[FCM<br/>Android]
    W --> H[HMS<br/>Huawei]
    W --> S[统计存储]
    W --> FB[失败反馈 Webhook]
    S --> M[/metrics 与统计 API]
```

统一请求里的平台编号是 `1 = iOS`、`2 = Android`、`3 = Huawei`。最小的 Android 推送请求如下：

```bash
curl -X POST http://localhost:8088/api/push \
  -H 'Content-Type: application/json' \
  -d '{
    "notifications": [{
      "tokens": ["device_token_1", "device_token_2"],
      "platform": 2,
      "title": "订单状态更新",
      "message": "你的订单已发货",
      "data": {"order_id": "202608050001"}
    }]
  }'
```

这层抽象有两个恰到好处的边界：`title` / `message` / `data` 是跨平台的共同语义；真正平台独有的能力仍然通过 `apns`、`notification`、`huawei_notification` 等字段下沉。**统一入口不等于抹平能力差异**——iOS 的 Live Activity、Android 的通知图标、华为的特有字段，本来就不该被强行压成同一个 DTO。

## 核心实现：队列把「接请求」和「发消息」解耦

推送的下游延迟不可控：厂商网络抖动、单个 token 无效、连接重建、短时限流，都会拖慢发送。如果 HTTP handler 同步逐条发送，业务调用方的超时会和推送吞吐强行绑死。

gorush 的默认本地队列是 Go `channel`，并允许替换为 NSQ、NATS 或 Redis Streams；Worker 数和队列长度由 `core.worker_num` / `core.queue_num` 控制：

```yaml
core:
  port: "8088"
  worker_num: 0       # 0 表示按 CPU 核数
  queue_num: 8192
  max_notification: 100
  sync: false

queue:
  engine: "redis"    # 单实例可用 local；多实例时考虑外部队列
  redis:
    addr: "redis:6379"
    group: "gorush"
    consumer: "gorush-worker"
    stream_name: "gorush"
```

这是一个典型的生产取舍：

- `local` 很轻，适合单副本、开发或可容忍进程故障丢失在途任务的场景；
- Redis Streams / NATS / NSQ 把队列从进程内移出去，才有跨副本消费和更好的故障边界；
- Worker 不是越多越好。它受下游厂商限流、连接数和应用自身 CPU/内存共同约束，应从指标压测得到并发，而不是拍脑袋填 `1000`。

一个简单但有用的容量估算是：若平均每次下游投递耗时为 `L` 秒，每个 Worker 可处理约 `1 / L` 次/秒；`W` 个 Worker 的理论上限约为 `W / L`。真实可用吞吐还要乘上厂商限流与重试带来的折损。队列持续增长时，优先确认下游失败/限流与消费延迟，而不是先无限扩容队列。

## 最重要的边界：成功响应不等于用户收到通知

这是推送系统最容易犯、也最昂贵的认知错误。

一次消息的状态至少分四层：

```text
业务请求被 gorush 接收
        ↓
gorush 已提交给 APNs / FCM / HMS
        ↓
厂商接受并尝试投递到设备
        ↓
设备在线、系统展示，用户实际看见 / 点击
```

gorush 可以给你请求受理、发送成功/失败的统计，并能在异步失败时回调 `feedback_hook_url`；但末两层仍受设备离线、系统通知权限、厂商策略、用户关闭通知等因素影响。**任何「营销触达率」或「订单已提醒」的业务结论，都不能只基于 gorush 的 HTTP 成功响应。**

建议把推送结果拆成两条数据流：

1. **投递链路指标**：提交数、成功/失败数、队列深度、Worker busy 数、下游错误码；
2. **业务效果事件**：App 收到后的 ACK、通知点击、关键页面打开或订单确认。

前者回答“网关有没有把任务交给厂商”，后者才回答“用户有没有被触达”。二者的差值，才是你真正要排查的运营和客户端问题。

## 可观测性与失败闭环：别让无效 token 永久重试

gorush 提供 `/api/stat/app`、`/api/stat/go`、`/sys/stats`、`/metrics` 和 `/healthz`，并支持将统计保存在 memory、Redis、BoltDB、BuntDB、LevelDB 或 BadgerDB。生产环境至少应接入 Prometheus，盯住以下信号：

| 信号 | 说明 | 触发动作 |
|---|---|---|
| 队列积压持续上升 | 进入速率超过投递速率 | 排查厂商延迟、限流与 Worker 数 |
| `failure_tasks` 激增 | 凭据、网络或 payload 可能出问题 | 按平台和错误码告警 |
| 无效 token 错误 | 客户端卸载、换机或 token 轮换 | 回写并失效该 token |
| 成功率正常但点击率下降 | 网关正常，用户侧体验异常 | 排查通知权限、文案和客户端埋点 |

失败反馈可以用 `feedback_hook_url` 异步送回业务系统。这个 hook 最值得做的不是“记一条错误日志”，而是建立 token 生命周期：遇到明确的失效/未注册错误，标记 token 不再可用；客户端下次启动上报新 token 后再恢复。否则系统会持续把成本打在永远不可能成功的地址上。

## 部署起步：配置、凭据、网络分层

本地可以直接用 Docker 启动：

```bash
docker run --rm -p 8088:8088 appleboy/gorush
```

真正上线时，建议用挂载的配置和密钥文件，而不是把私钥写进镜像或 YAML：

```yaml
android:
  enabled: true
  key_path: "/run/secrets/fcm-service-account.json"

ios:
  enabled: true
  key_path: "/run/secrets/apns-key.p8"
  key_type: "p8"
  key_id: "YOUR_KEY_ID"
  team_id: "YOUR_TEAM_ID"
  production: true

log:
  hide_token: true
  hide_messages: false
```

这里有三条不能省的纪律：

- **凭据进 Secret，不进仓库**：FCM 服务账号、APNs `.p8` / `.p12` / `.pem` 都应由 Kubernetes Secret、云密钥服务或受控挂载提供，并建立轮换计划。
- **网关不裸露在公网**：`/api/push` 是能直接触达用户的高权限接口，应置于内网、API 网关或服务鉴权之后；同时评估是否要对 `/api/config` 这类运维接口做额外隔离。
- **日志默认脱敏**：设备 token 可被用于向特定设备推送，`hide_token: true` 应保持开启；消息正文若涉及订单、验证码或个人信息，也要审慎决定是否记录。

## 什么时候适合用 gorush，什么时候不该把它当答案

gorush 很适合这些场景：多个业务服务需要统一推送能力；同时面对 iOS、Android、华为；希望快速获得队列、重试、指标、Docker/Kubernetes 部署和 gRPC 入口，而不想在每个业务里重复造轮子。

但它不是完整的“消息运营平台”。用户分群、定时编排、模板审核、频控、A/B 实验、打开率归因，以及设备最终回执，都需要业务系统或上层平台来做。把 gorush 放在正确的位置——**可靠的投递网关，而非用户触达真相的唯一来源**——它的边界就很清晰，也很有价值。

## 小结

gorush 最值得借鉴的不是“把 curl 包一下”，而是把多厂商推送抽象成一条可排队、可观测、可反馈的投递管道。它替业务服务收掉协议复杂度，但不会替你消灭异步系统的基本事实：接收、厂商受理、设备送达、用户看见，是不同的状态。

把这些状态拆开建模，用外部队列跨过进程边界，用指标发现积压，用反馈清理无效 token，再用客户端事件衡量真正触达——这套闭环建立起来，推送才能从“偶尔能发出去的功能”变成可运营的基础设施。
