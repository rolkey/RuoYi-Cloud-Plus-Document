# 实时信息通道开发计划

> 配套文档：`requirement.md`（需求）、`plan.md`（架构与实施方案）、`test-plan.md`（测试计划）
> 本计划基于对代码基的实际盘点修订（`AGENTS.md` + 程序目录），取代"从零新建服务"的假设，改为**演进现有 `ruoyi-common-websocket` 公共模块**。

---

## 一、目标定位

| 项 | 结论 |
|----|------|
| 目标后端 | `RuoYi-Cloud-Plus/`（活跃参照版，完整微服务） |
| 目标前端 | `RuoYi-Cloud-Plus-UI/`（微前端主应用 shell） + `RuoYi-Cloud-Plus-UI-HIS/`（子应用） |
| 落地方式 | **演进现有公共模块** `ruoyi-common/ruoyi-common-websocket`，**非新建 `ruoyi-visual/ruoyi-websocket` 服务** |

> 关键结论：代码基**已有** WebSocket 公共模块（Spring WebSocket + Sa-Token 鉴权 + Redis pub/sub 分发），也有 RabbitMQ 基建（`ruoyi-common-bus` + `application-common.yml` 的 `spring.rabbitmq` + `ruoyi-test-mq` 示例）和前端 WebSocket 客户端 + mitt 消息总线。本需求的核心工作是**增量改造**：Redis pub/sub → RabbitMQ、新增三级订阅与前缀匹配路由、前端按 `type` 派发；**业务侧只连接消息服务（RabbitMQ）发布消息，不引用 websocket 模块**。

---

## 二、现状盘点（代码基已有资产）

### 2.1 后端 `ruoyi-common-websocket`（`RuoYi-Cloud-Plus/ruoyi-common/ruoyi-common-websocket/src/main/java/org/dromara/common/websocket/`）

| 类 | 现状 | 本需求改动 |
|----|------|-----------|
| `config/WebSocketConfig.java` | Spring WebSocket 自动装配，`@ConditionalOnProperty(websocket.enabled=true)` | 增补 RabbitMQ 交换机/队列 Bean |
| `config/properties/WebSocketProperties.java` | path / allowedOrigins / enabled | 基本不变 |
| `handler/PlusWebSocketHandler.java` | 连接建立/关闭、文本/心跳处理 | 基本不变（连接/心跳逻辑复用） |
| `interceptor/PlusWebSocketInterceptor.java` | Sa-Token `LoginHelper.getLoginUser()` 鉴权 | 基本不变（订阅接口鉴权复用其思路） |
| `holder/WebSocketSessionHolder.java` | 内存 Map，key=`userId`(Long) | 保持（每实例只推本地会话） |
| `dto/WebSocketMessageDto.java` | `{sessionKeys, message}` | **迁至共享 api 模块**，新增 `type`/`source` 字段 |
| `constant/WebSocketConstants.java` | LOGIN_USER_KEY / WEB_SOCKET_TOPIC / PING / PONG | **迁至共享 api 模块**，增补交换机常量 |
| `utils/WebSocketUtils.java` | Redis pub/sub 的 publish/subscribe | **移除 publish/subscribe，仅保留 sendMessage（内部推送）** |
| `listener/WebSocketTopicListener.java` | Redis pub/sub 订阅者（ApplicationRunner） | **改为 @RabbitListener 消费 + 前缀匹配路由** |

### 2.2 RabbitMQ 基建（可直接复用）

| 资产 | 位置 | 用途 |
|------|------|------|
| AMQP 依赖参考 | `ruoyi-common/ruoyi-common-bus/pom.xml`（`spring-cloud-starter-bus-amqp`） | 参考引入 `spring-boot-starter-amqp` |
| RabbitMQ 连接配置 | `script/config/nacos/application-common.yml`（`spring.rabbitmq` localhost:5672） | 直接复用 |
| 生产者/消费者示例 | `ruoyi-example/ruoyi-test-mq/src/main/java/org/dromara/stream/` | 参考 `RabbitConfig`、`RabbitConsumer` 写法 |

### 2.3 前端（`RuoYi-Cloud-Plus-UI/`）

| 文件 | 现状 | 本需求改动 |
|------|------|-----------|
| `src/utils/websocket.ts` | `initWebSocket()`：`useWebSocket`(vueuse) 建连 + 心跳 ping + 通知弹窗 | **改为按 `type` 派发到消息总线** |
| `src/micro/messageBus.ts` | mitt 总线，`window.__QIANKUN_MSG_BUS__`，已注入子应用 | 复用（新增 `ws:{type}` 事件约定） |
| `src/micro/microApp.ts` | `createMicroApp` 将 `msgBus` 作为 props 传给子应用 | 复用 |
| `src/api/websocket/` | 无 | **新增订阅接口封装** |

---

## 三、需求 vs 现状差距分析

| 需求点 | 现状 | 差距 | 应对 |
|--------|------|------|------|
| 各服务通过 RabbitMQ 实时交互 | Redis pub/sub（`global:websocket`） | 分发机制不同 | 用 RabbitMQ 替换 Redis pub/sub |
| 消息格式 JSON | 已是 JSON | 无 | 无（保持） |
| 三级订阅（主类/子类/业务ID） | 无订阅，仅 `sessionKey=userId` 定向 + 广播 | 缺订阅模型 | 新增订阅存储 + `type` 字段 + 前缀匹配 |
| 客户端经业务接口订阅 | 无订阅接口 | 缺接口 | 新增订阅/退订/查询 REST 接口 |
| 服务器按订阅情况推送 | 仅定向/广播 | 缺订阅路由 | 消费端按前缀匹配路由 |
| 服务器与客户端只创建一个连接 | 前端已单连接（`initWebSocket` 单例） | 无 | 保持，确认子应用不直连 |
| 微前端消息总线交互 | `messageBus` 已存在并注入子应用 | 缺按 `type` 派发 | 主应用 onmessage 后按 `type` emit |

---

## 四、改造方案要点

### 4.1 会话与订阅键统一为 userId（不引入 clientKey）

- 现状 `PlusWebSocketInterceptor` 已用 Sa-Token 提取 `LoginUser`，会话 key 为 `userId`。
- 订阅也以 `userId` 为键（订阅接口用 `LoginHelper.getUserId()` 识别调用者），**不新增 clientKey**，避免额外的身份映射。

### 4.2 RabbitMQ 分发（fanout + 每实例独立队列）

- 交换机：`websocket.exchange`（fanout，广播到所有实例）。
- 队列：每实例启动时声明自己的队列（如 `websocket.queue.{spring.application.name}-{随机}`，autoDelete），绑定到 fanout。
- 生产端：业务服务通过 `RabbitTemplate` 直接 `convertAndSend(exchange, "", dto)` 发布（业务侧只依赖 `spring-boot-starter-amqp` + 共享 api 模块，**不引用 websocket 模块**）。
- 消费端：每实例 `@RabbitListener` 消费全量消息 → 本地会话 + 订阅过滤 → 推送到**本实例**持有的连接。
- 好处：消息必达持有会话的实例，无需跨实例寻址，与现有"内存会话 Map"模型完全兼容。

### 4.3 三级订阅与前缀匹配

- 主题格式：`{主类}/{子类}/{具体业务ID}`（1~3 段，`/` 分隔）。
- 存储（Redis，正向+反向索引）：
  - `global:ws:sub:{topic}` → Set<userId>（精确主题 → 订阅用户）
  - `global:ws:userSub:{userId}` → Set<topic>（用户 → 已订阅主题，断开清理用）
- 路由：消费端对 `type` 生成全部前缀（如 `his/order/10086` → `his`、`his/order`、`his/order/10086`），并集各 `global:ws:sub:{前缀}` 得目标 userId，推送本地会话。
- 优先级：`sessionKeys` 非空（定向）> `type` 非空（订阅前缀匹配）> 均空（广播）。

---

## 五、任务分解（WBS）

### Phase 1：后端（共享 api 模块 + 演进 websocket 模块）（预估 6 人日）

| 任务 | 内容 | 交付物 | 依赖 | 预估 |
|------|------|--------|------|------|
| T1.1 | 共享 api 模块 | 新建 `ruoyi-api-websocket`：`WebSocketMessageDto`（type/sessionKeys/message/source）、`WebSocketConstants`（交换机名）、`WebSocketTopic`、`WebSocketTopicUtils` | 共享模块 | 无 | 1 人日 |
| T1.2 | MQ 拓扑 | websocket 模块 `pom.xml` 加 `spring-boot-starter-amqp`；新增 `WebSocketRabbitConfig`（fanout 交换机 + 每实例队列） | 配置类 | 无 | 0.5 人日 |
| T1.3 | 订阅存储 | 新增 `WebSocketSubscribeService`（Redis 双向索引 subscribe/unsubscribe/getSubscribedUsers/cleanup） | 服务类 | T1.1 | 1 人日 |
| T1.4 | 订阅接口 | 新增 `WebSocketSubscribeController`（订阅/退订/查询，Sa-Token 鉴权） | 接口 | T1.3 | 1 人日 |
| T1.5 | MQ 消费替换 | `WebSocketUtils` 移除 publish/subscribe（保留 sendMessage）；`WebSocketTopicListener` 改为 `@RabbitListener` 消费 + 前缀匹配路由 | 核心链路 | T1.1~T1.3 | 1.5 人日 |
| T1.6 | 配置与联调 | 复用 `application-common.yml` 的 rabbitmq/redis 配置；本地启动自测 | 配置 | T1.5 | 1 人日 |

**Phase 1 验收（M1）：** 服务可启动；客户端可鉴权连接；可订阅/退订；RabbitMQ 消息按三级前缀匹配推送到目标客户端。

### Phase 2：业务侧接入（预估 1 人日）

| 任务 | 内容 | 交付物 | 依赖 | 预估 |
|------|------|--------|------|------|
| T2.1 | 打通示例链路 | `ruoyi-system` 通知业务触发点通过 `RabbitTemplate` 发布消息到交换机（依赖 amqp + 共享 api，**不引用 websocket 模块**） | 调用点 | Phase 1 | 1 人日 |

**Phase 2 验收（M2）：** 业务事件 → RabbitMQ → 消费 → 订阅前缀匹配 → 客户端推送，全链路日志可追踪。

### Phase 3：前端接入（预估 3.5 人日）

| 任务 | 内容 | 交付物 | 依赖 | 预估 |
|------|------|--------|------|------|
| T3.1 | 演进 `websocket.ts` | onmessage 解析 JSON，按 `type` 通过 `msgBus.emit('ws:'+type, data)` 派发 | 客户端 | Phase 1 | 1 人日 |
| T3.2 | 订阅接口封装 | 新增 `src/api/websocket/index.ts`（subscribe/unsubscribe/查询） | API 模块 | 无 | 0.5 人日 |
| T3.3 | 登录后建连与初始订阅 | 登录成功后调用 `initWebSocket` + 初始订阅 | 集成点 | T3.1、T3.2 | 0.5 人日 |
| T3.4 | HIS 子应用接入 | 子应用通过 `msgBus.on('ws:his/*')` 接收推送示例 | 示例 | T3.1 | 1.5 人日 |

**Phase 3 验收（M3）：** 主应用单连接复用；服务端推送经总线到达子应用；断线可重连并重新订阅；HIS 子应用收到 `his/*` 推送。

---

## 六、里程碑与排期

| 里程碑 | 完成标志 | 预估累计 | 验收口径 |
|--------|----------|----------|----------|
| M1 后端可连接 | Phase 1 完成 | 6 人日 | 连接/订阅/MQ 推送闭环 |
| M2 端到端打通 | Phase 2 完成 | 7 人日 | 业务事件 → 客户端推送全链路可追踪 |
| M3 前端接入完成 | Phase 3 完成 | 10.5 人日 | 主应用单连接 + 总线派发 + 子应用可接收 |

> 总工作量预估 **10.5 人日**。Phase 2 与 Phase 3 可并行（后端打通示例链路的同时，前端搭连接与订阅）。

---

## 七、角色与分工

| 角色 | 职责 | 对应任务 |
|------|------|----------|
| 后端开发 | 共享 api 模块、演进 `ruoyi-common-websocket`、业务接入 | T1.1~T1.6、T2.1 |
| 前端开发 | 演进 `websocket.ts`、订阅封装、总线派发、子应用 | T3.1~T3.4 |
| 测试 | 按 `test-plan.md` 执行验证 | 全阶段 |
| 架构评审 | 主题命名、Redis 结构、RabbitMQ 拓扑、路由优先级评审 | T1.1、T1.3、T1.5 完成后 |

---

## 八、交付物清单

| # | 交付物 | 类型 | 归属任务 |
|---|--------|------|----------|
| 1 | 共享 api 模块 `ruoyi-api-websocket` 源码 | 代码 | T1.1 |
| 2 | 演进后的 `ruoyi-common-websocket` 模块源码 | 代码 | T1.2~T1.6 |
| 3 | RabbitMQ 交换机/队列配置类 | 代码 | T1.2 |
| 4 | 订阅/退订/查询接口 | 接口 | T1.4 |
| 5 | 业务服务发布调用点（RabbitTemplate） | 代码 | T2.1 |
| 6 | 演进后的 `websocket.ts` + 订阅封装 | 代码 | T3.1~T3.2 |
| 7 | 消息总线派发逻辑 | 代码 | T3.1 |
| 8 | HIS 子应用接入示例 | 代码 | T3.4 |
| 9 | `test-plan.md` 用例执行记录 | 文档 | 全阶段 |

---

## 九、风险与应对

| 风险 | 影响 | 应对 |
|------|------|------|
| RabbitMQ fanout + 每实例队列拓扑配置错误 | 消息不达/重复 | 参考 `ruoyi-test-mq` 的 `RabbitConfig` 写法；M1 双实例联调 |
| 前缀匹配边界误判（`his/order` 匹配 `his/orderX`） | 错发消息 | `WebSocketTopicUtils.prefixes` 按段拆分，单测覆盖边界 |
| 会话内存化 + 订阅 Redis 化不一致 | 推送丢失 | 订阅 key=userId 与会话 key 严格一致；断开清理走反向索引 |
| 断线重连后订阅丢失 | 收不到推送 | 前端重连后强制重新订阅，纳入 M3 验收 |
| 与既有 `ruoyi-common-bus`（Spring Cloud Bus）事件串扰 | 消息混淆 | websocket 走独立 `websocket.exchange`，不占用 bus 的事件总线 |
| 与既有消息总线事件命名冲突 | 前端事件串扰 | 统一 `ws:{type}` 命名，接入前盘点现有 mitt 事件 |

---

## 十、开发规范约束

- 复用现有 Spring WebSocket 骨架（`WebSocketConfig`/`PlusWebSocketHandler`/`PlusWebSocketInterceptor`），只做增量改造，不重写。
- 主题统一由 `WebSocketTopicUtils` / `WebSocketTopic` 常量生成，禁止业务侧手写字符串拼接。
- 会话 key 沿用 `userId`，订阅 key 沿用 `userId`，禁止引入 clientKey 双重身份体系。
- 业务模块只连接消息服务（RabbitMQ），**禁止引用 `ruoyi-common-websocket` 模块**；共享契约走 `ruoyi-api-websocket`。
- 禁止类型逃逸（`as any` / `@ts-ignore` / `@SuppressWarnings` 掩盖类型问题）。
- 复用 `application-common.yml` 现有 `spring.rabbitmq` 与 `spring.data.redis` 配置，不在代码内硬编码连接信息。
- 接口变更同步更新 `plan.md` 与本计划；提交粒度与代码库既有规范一致。

---

## 附：架构分层说明

本计划遵循「业务侧与通讯侧解耦」原则：

- **业务模块**：只连接消息服务（RabbitMQ），通过 `RabbitTemplate` 发布消息，依赖 `ruoyi-api-websocket`（DTO/常量/主题工具），**不引用 `ruoyi-common-websocket`**。
- **websocket 模块**：负责 WebSocket 连接、订阅路由、RabbitMQ 消费与推送，对业务侧透明。
