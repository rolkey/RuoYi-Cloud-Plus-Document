# 实时信息通道实施计划

> 基于 `document/specs/require05/requirement.md` 与代码基盘点（`AGENTS.md` + 已读取 `ruoyi-common-websocket` 模块、前端 `micro/` 结构）修订。
> 核心修正：**不新建独立服务，而是演进现有 `ruoyi-common-websocket` 公共模块**；会话/订阅身份统一沿用 `userId`（Sa-Token 已识别），不引入 clientKey。

---

## 一、需求理解

本需求要建立一条**服务器与客户端之间的实时信息通道**，核心要点：

1. **协议**：采用 WebSocket 协议。
2. **连接模型**：服务器与每个微前端客户端**只建立一个连接**（单连接复用），由持有该连接的服务负责与客户端交互。
3. **多微服务推送**：后台为若依微服务架构，多个微服务（system、job、workflow、HIS 等）都能向客户端推送信息，并支持**广播**。
4. **服务间通信**：各微服务之间通过 **RabbitMQ** 实时交互，消息格式为 **JSON**。
5. **前端交互**：微前端客户端通过**消息总线（mitt）**与主微前端主应用交互，由主应用统一负责与后台建立/维护 WebSocket 连接并分发消息。
6. **消息订阅**：客户端可通过**业务接口**向服务器订阅消息；服务器收到 RabbitMQ 消息时，**根据订阅情况**仅向订阅了对应主题的客户端发送。
7. **订阅粒度（三级）**：订阅主题按 **主类（模块）/ 子类（业务）/ 具体业务ID** 三级划分，客户端可按任意粒度订阅。

> 结论：这是一条 **WebSocket + RabbitMQ + Redis + 微前端消息总线** 的实时推送链路。代码基已有 `ruoyi-common-websocket` 公共模块（Spring WebSocket + Sa-Token 鉴权 + Redis pub/sub 分发），本需求在其基础上做**增量改造**：Redis pub/sub → RabbitMQ、新增**「三级主题订阅 + 前缀匹配路由」**能力。

---

## 二、总体架构设计

### 2.1 后端链路（推送方向：微服务 → 客户端）

```
┌──────────────┐   publish(JSON)   ┌────────────────────────┐
│ ruoyi-system │ ────────────────▶ │                        │
│ ruoyi-job    │ ────────────────▶ │   RabbitMQ             │
│ ruoyi-his    │ ────────────────▶ │  (fanout 交换机)        │
└──────────────┘                   └─────────┬──────────────┘
                                             │ 每实例独立队列消费(JSON)
                                             ▼
                            ┌────────────────────────────────────┐
                            │ 各服务内嵌 ruoyi-common-websocket    │
                            │ (持有该客户端连接的实例 = 其"主服务")  │
                            │ 订阅表存 Redis，按三级前缀匹配路由      │
                            └───────────────┬────────────────────┘
                                            │ WebSocket 推送（本实例内存会话）
                                            ▼
                                   ┌──────────────────┐
                                   │  微前端主应用      │  ← 只建一个连接
                                   └──────────────────┘
```

- **生产端**：任意业务微服务通过 `RabbitTemplate` 把消息序列化成 JSON 直接发布到 RabbitMQ fanout 交换机（业务侧**只连接消息服务，不引用 websocket 模块**）。
- **消费端**：每个启用 websocket 的服务实例有自己的队列，消费全部消息后**先按 `sessionKeys`（定向）→ `type`（三级订阅前缀匹配）→ 广播**的优先级，从 Redis 查出目标 userId，推送**本实例内存**中持有的会话。
- **会话/订阅存储**：会话存**各实例内存**（`WebSocketSessionHolder`，key=userId）；订阅关系存 **Redis**（共享）。fanout 保证消息必达持有会话的实例，无需跨实例寻址。

### 2.2 前端链路（推送方向：客户端 → 微服务 / 微服务 → 客户端）

```
┌───────────────┐     消息总线 mitt       ┌──────────────┐
│ 微前端子应用    │ ◀──────────────────▶ │ 微前端主应用  │
│ (HIS / 子应用) │   subscribe/publish    │ (RuoYi-UI)   │
└───────────────┘                        └──────┬───────┘
                                                │ 唯一 WebSocket 连接（经网关）
                                                │ 订阅/退订走 REST 业务接口
                                                ▼
                                         ┌────────────────┐
                                         │ 后端服务(内嵌    │
                                         │ websocket 模块) │
                                         └────────────────┘
```

- 主应用维护**唯一** WebSocket 连接，负责建连、心跳、重连、鉴权。
- 子应用通过主应用暴露的消息总线（mitt，已存在）订阅/发送消息，自身不直接连接后台。
- 主应用（或子应用经主应用转发）通过 **REST 业务接口**向服务器声明订阅哪些主题（三级粒度）。

### 2.3 订阅路由流程

```
① 客户端订阅            ┌──────────────────────────────────────────┐
  POST /subscribe ────▶ │ 写入 Redis:  ws:sub:{topic} → Set<userId> │
  {types}（Sa-Token）    │              ws:userSub:{userId} → Set<topic>│
                        └──────────────────────────────────────────┘

② 服务推送                                    ③ 按前缀匹配路由
  业务服务 publishMessage ─▶ RabbitMQ(fanout) ─▶ 每实例 Consumer
                                                 ─▶ 拆分 topic 生成前缀
                                                 ─▶ 并集各 ws:sub:{前缀} 的 userId
                                                 ─▶ 推送给本实例内存中该 userId 的会话
```

---

## 三、订阅类型设计（主类 / 子类 / 具体业务ID）

### 3.1 三级主题模型

订阅主题是一个层级路径，最多三段，用 `/` 分隔：

```
{主类}/{子类}/{具体业务ID}
 模块    业务      记录ID（可省略）

主题分段：
  第 1 段 = 主类（模块）     —— 对应后端微服务模块，如 system / his / job / workflow / gen
  第 2 段 = 子类（业务）     —— 模块下的具体业务，如 sysNotice / sysMsg / order / patient
  第 3 段 = 具体业务ID       —— 具体记录主键，用于精确订阅某条记录
```

订阅粒度示例：

| 订阅主题 | 粒度 | 含义 |
|----------|------|------|
| `system` | 主类 | 订阅 system 模块的**全部**消息 |
| `system/sysNotice` | 子类 | 订阅 system 模块「通知」业务的全部消息 |
| `system/sysNotice/123` | 业务ID | 只订阅「通知」中 ID=123 的消息 |
| `his` | 主类 | 订阅 HIS 模块的全部消息 |
| `his/order` | 子类 | 订阅 HIS 模块「订单」业务全部消息 |
| `his/order/10086` | 业务ID | 只订阅订单 10086 的消息 |

### 3.2 匹配规则（前缀匹配）

- **发布方**始终携带**完整**主题（三段，如 `his/order/10086`）。
- **订阅方**可携带 1~3 段，表示订阅以该路径为前缀的所有消息。
- **匹配判定**：消息主题 `M` 命中订阅 `S`，当且仅当 `S` 是 `M` 的**分段前缀**（逐段相等）。

匹配示例（消息主题 = `his/order/10086`）：

| 订阅主题 | 是否命中 | 原因 |
|----------|----------|------|
| `his` | ✅ | 模块级前缀命中 |
| `his/order` | ✅ | 业务级前缀命中 |
| `his/order/10086` | ✅ | 精确命中 |
| `his/patient` | ❌ | 子类不同 |
| `system` | ❌ | 模块不同 |

---

## 四、服务端交互 JSON 详情

### 4.1 服务间消息 DTO（RabbitMQ 传递）

```json
{
  "type": "his/order/10086",
  "sessionKeys": [1001, 1002],
  "message": "{\"orderId\":10086,\"status\":\"paid\"}",
  "source": "ruoyi-his"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `type` | `String` | 是 | 三级主题路径，用于订阅前缀匹配 |
| `sessionKeys` | `List<Long>` | 否 | 目标 userId 列表（定向推送，优先级最高，见 4.4） |
| `message` | `String` | 是 | 业务消息体（JSON 字符串，由业务服务自定义，服务端原样透传） |
| `source` | `String` | 否 | 消息来源服务名（用于追踪/日志） |

> 说明：`sessionKeys` 沿用现有 `WebSocketMessageDto` 的 `List<Long>`（userId），新增 `type`、`source` 字段。

### 4.2 订阅接口（REST，由 websocket 公共模块提供，Sa-Token 鉴权）

**订阅：**

```
POST /websocket/subscribe
请求体：
{
  "types": ["system", "system/sysNotice", "his/order/10086"]
}
响应：
{
  "code": 200,
  "msg": "操作成功"
}
```

**退订：**

```
POST /websocket/unsubscribe
请求体：
{
  "types": ["system/sysNotice"]
}
响应：
{
  "code": 200,
  "msg": "操作成功"
}
```

**查询订阅列表：**

```
GET /websocket/subscription
响应：
{
  "code": 200,
  "msg": "操作成功",
  "data": ["system", "his/order/10086"]
}
```

> 说明：调用者身份由 Sa-Token 识别（`LoginHelper.getUserId()`），**请求体无需携带 clientKey/userId**，从源头上杜绝伪造他人身份订阅。

### 4.3 客户端消息（WebSocket 推送 / 上送）

**推送（服务端 → 客户端）：**

```json
{
  "type": "his/order/10086",
  "data": { "orderId": 10086, "status": "paid" },
  "timestamp": 1735689600000
}
```

| 字段 | 说明 |
|------|------|
| `type` | 三级主题路径，前端据此路由到对应子应用/处理逻辑 |
| `data` | 业务数据（各服务自定义） |
| `timestamp` | 推送时间戳（毫秒） |

**上送（客户端 → 服务端）：**

```json
{
  "type": "his/order/submit",
  "data": { "orderId": 10086, "payload": "..." },
  "timestamp": 1735689600000
}
```

**心跳（客户端 → 服务端 → 客户端）：**

```json
{ "type": "ping" }
{ "type": "pong" }
```

### 4.4 消费端路由优先级

| 优先级 | 条件 | 行为 |
|--------|------|------|
| 1 | `sessionKeys` 非空 | 定向推送给指定 userId 列表（跳过订阅匹配） |
| 2 | `sessionKeys` 为空 且 `type` 非空 | 按 `type` 做三级前缀匹配，推送给命中的订阅用户 |
| 3 | `sessionKeys` 为空 且 `type` 为空 | 广播全部在线客户端（本实例全部会话） |

### 4.5 会话与订阅存储

| 位置 | Key | 类型 | 说明 |
|------|-----|------|------|
| 各实例内存 | `WebSocketSessionHolder` | Map | `userId → WebSocketSession`（本实例持有） |
| Redis | `ws:sub:{topic}` | Set | 该**精确主题**已订阅的 userId 集合（正向索引，topic 为 1~3 段） |
| Redis | `ws:userSub:{userId}` | Set | 该用户订阅的全部主题（反向索引，断开清理用） |

> 说明：订阅按「精确主题」逐条存储，**不做通配符展开**。路由时由消费端将消息主题拆成全部前缀再逐条查询（见 8.2），保证主类/子类/业务ID 三种粒度都能命中。

---

## 五、涉及模块 / 文件清单

### 5.1 后端（共享 api 模块 + 演进 websocket 模块）

**共享 api 模块 `ruoyi-api-websocket`（轻量，业务侧与 websocket 侧共同引用）：**

| # | 类 | 职责 |
|---|----|------|
| 1 | `WebSocketMessageDto` | 消息 DTO（type / sessionKeys / message / source） |
| 2 | `WebSocketConstants` | 交换机名等常量（供业务侧发布使用） |
| 3 | `WebSocketTopic` | 主类（模块）+ 常见子类常量 |
| 4 | `WebSocketTopicUtils` | 主题工具：build/split/validate/prefixes |

**websocket 模块 `ruoyi-common-websocket`（连接/订阅/消费/推送，业务侧不引用）：**

| # | 类 | 改动 |
|---|----|------|
| 5 | `config/WebSocketConfig.java` | 增补 RabbitMQ 交换机/队列 Bean |
| 6 | `config/properties/WebSocketProperties.java` | 基本不变 |
| 7 | `handler/PlusWebSocketHandler.java` | 基本不变（连接/心跳/关闭逻辑复用） |
| 8 | `interceptor/PlusWebSocketInterceptor.java` | 基本不变（Sa-Token 鉴权） |
| 9 | `holder/WebSocketSessionHolder.java` | 保持（内存会话，key=userId） |
| 10 | `utils/WebSocketUtils.java` | **移除 publish/subscribe，仅保留 sendMessage（内部推送）** |
| 11 | `listener/WebSocketTopicListener.java` | **改为 @RabbitListener 消费 + 前缀匹配路由** |
| 12 | `config/WebSocketRabbitConfig.java`（新增） | fanout 交换机 + 每实例独立队列（autoDelete） |
| 13 | `service/WebSocketSubscribeService.java`（新增） | 订阅关系增删查：`ws:sub:{topic}` 与 `ws:userSub:{userId}` |
| 14 | `controller/WebSocketSubscribeController.java`（新增） | 订阅/退订/查询 REST 接口（对应 4.2） |

### 5.2 后端（业务侧发布，仅新增少量代码）

| # | 位置 | 改动 |
|---|------|------|
| 15 | 各业务服务（system/his 等） | 依赖 `spring-boot-starter-amqp` + `ruoyi-api-websocket`，通过 `RabbitTemplate` 发布消息到交换机（**不引用 websocket 模块**） |

### 5.3 前端（主应用 + 子应用）

| # | 文件 | 改动 |
|---|------|------|
| 16 | 主应用 `src/utils/websocket.ts`（已有） | **演进**：onmessage 解析 JSON，按 `type` 通过 `msgBus.emit('ws:'+type, data)` 派发 |
| 17 | 主应用 `src/micro/messageBus.ts`（已有） | 复用 mitt 总线（已注入子应用），约定 `ws:{type}` 事件命名 |
| 18 | 主应用订阅 API 封装 `src/api/websocket/index.ts`（新增） | 封装 subscribe / unsubscribe / 查询接口 |
| 19 | 子应用（如 HIS） | 通过 `msgBus.on('ws:his/*')` 接收推送；必要时上送消息、申请订阅 |
| 20 | 鉴权/登录处 | 登录成功后调用 `initWebSocket` 建连 + 初始订阅 |

---

## 六、详细实施步骤

### Phase 1：后端（共享 api 模块 + 演进 websocket 模块）

**Step 1.1 — 共享 api 模块**
- 新建 `ruoyi-api-websocket`：`WebSocketMessageDto`（type/sessionKeys/message/source）、`WebSocketConstants`（交换机名 `websocket.exchange`）、`WebSocketTopic`、`WebSocketTopicUtils`（build/split/validate/prefixes）。

**Step 1.2 — MQ 拓扑**
- websocket 模块 `pom.xml` 新增 `spring-boot-starter-amqp` 依赖（参考 `ruoyi-common-bus`）。
- 新增 `WebSocketRabbitConfig`：声明 fanout 交换机 `websocket.exchange` + 每实例 autoDelete 队列并绑定。
- 复用 `application-common.yml` 现有 `spring.rabbitmq` 连接配置。

**Step 1.3 — 订阅存储**
- 新增 `WebSocketSubscribeService`：`subscribe(userId, types)` / `unsubscribe(userId, types)` / `getSubscribedUsers(topic)` / `cleanup(userId)`。
- 维护 Redis 双向索引（4.5 节）；`PlusWebSocketHandler.afterConnectionClosed` 时调用 `cleanup`。

**Step 1.4 — 订阅接口**
- 新增 `WebSocketSubscribeController`：`POST /websocket/subscribe`、`/unsubscribe`、`GET /websocket/subscription`，用 `LoginHelper.getUserId()` 识别调用者。

**Step 1.5 — MQ 消费替换**
- `WebSocketUtils` 移除 publish/subscribe（仅保留 sendMessage 内部推送）。
- `WebSocketTopicListener`（Redis pub/sub）改为 `@RabbitListener` 消费，按路由优先级（4.4 节）做前缀匹配，推送本实例内存会话。

**Step 1.6 — 配置与联调**
- 复用 `application-common.yml` 的 rabbitmq/redis 配置；本地启动自测连接、订阅、推送闭环。

### Phase 2：业务侧接入

**Step 2.1 — 打通一条示例链路**
- 在 `ruoyi-system` 通知业务触发点通过 `RabbitTemplate` 发布消息到 `websocket.exchange`（依赖 amqp + `ruoyi-api-websocket`，**不引用 websocket 模块**）。
- 验证：服务端日志确认消息经 MQ → 消费 → 订阅前缀匹配 → 推送到客户端。

### Phase 3：前端接入

**Step 3.1 — 演进主应用 WebSocket 客户端**
- 演进 `utils/websocket.ts`：`onmessage` 解析 JSON，按 `type` 通过 `msgBus.emit('ws:'+type, data)` 派发；保留现有心跳、重连逻辑。

**Step 3.2 — 订阅接口封装与初始订阅**
- 新增 `src/api/websocket/index.ts` 封装 subscribe/unsubscribe/查询。
- 登录/建连成功后，根据当前用户权限或子应用声明完成初始订阅（模块级/业务级粒度）。

**Step 3.3 — 消息总线派发**
- 主应用 `onmessage` 后按 `type` 派发；子应用通过 `msgBus.on('ws:{type}')` 订阅。
- 子应用需要上送消息或调整订阅时 `emit` 给主应用，主应用统一 `send` / 调订阅接口。

**Step 3.4 — 子应用接入示例**
- HIS 子应用订阅 `his/order`、`his/patient` 等事件，展示推送；必要时动态调整订阅粒度（如进入订单详情后订阅 `his/order/10086`）。

---

## 七、实施顺序

```
Phase 1（后端：共享 api + MQ 拓扑 + 订阅 + MQ 消费替换 + 联调） → Phase 2（业务打通示例链路） → Phase 3（前端演进+订阅+总线派发）
```

- Phase 1 内部：共享 api 模块 → MQ 拓扑 → 订阅存储 → 订阅接口 → MQ 消费替换 → 联调，需顺序进行（依赖关系）。
- Phase 2 依赖 Phase 1 完成；Phase 3 依赖 Phase 1（后端可连接、可订阅）完成。
- Phase 2 与 Phase 3 可并行（一边打通后端推送，一边演进前端连接与订阅）。

---

## 八、程序处理要求

### 8.1 主题规范化与校验

- 主题由 **1~3 段**组成，用 `/` 分隔，如 `system/sysNotice/123`。
- 每段**非空**，字符限制 `[A-Za-z0-9_-]`，**禁止**包含 `/` 以外的分隔符、空白、特殊字符。
- 发布方必须携带**完整三段**（主类/子类/业务ID）；订阅方允许 1~3 段。
- 非法主题：校验不通过则**拒绝并记录日志**（订阅接口返回错误码，发布方抛异常），不得静默丢弃。

### 8.2 订阅匹配与路由算法

- 消费端收到消息后，对 `type` 做**前缀匹配**：
  1. 拆分 `type` 为段数组，如 `["his","order","10086"]`。
  2. 生成全部前缀：`["his", "his/order", "his/order/10086"]`。
  3. 逐个查询 `ws:sub:{前缀}` 集合，**并集去重**得到目标 userId 集合。
  4. 对集合内每个 userId，查**本实例内存会话**（`WebSocketSessionHolder`）并推送；本实例无该会话则跳过。
- 定向推送（`sessionKeys` 非空）**跳过**订阅匹配，直接按 userId 推送。
- 广播（`type` 与 `sessionKeys` 均为空）推送给**本实例全部**在线会话。

### 8.3 路由优先级

1. `sessionKeys` 非空 → 定向推送（优先级最高）
2. `type` 非空 → 三级前缀匹配订阅推送
3. 均空 → 广播

> 命中即停止，避免同一用户因同时命中定向与订阅而重复收到消息（实现时对目标集合统一去重）。

### 8.4 订阅生命周期管理

- **subscribe**：以 `LoginHelper.getUserId()` 识别用户 → 对每个 topic 校验合法性 → 写入 `ws:sub:{topic}`（SADD，幂等去重）与 `ws:userSub:{userId}`。
- **unsubscribe**：从 `ws:sub:{topic}` 移除 userId；若 set 为空则删除该 key；同步更新反向索引。
- **disconnect**：读 `ws:userSub:{userId}` 得到其全部订阅主题，逐个从 `ws:sub:{topic}` 移除 userId，最后删除反向索引，**不留脏数据**。
- **重复订阅幂等**：同一用户重复订阅同一主题不产生重复推送（Set 去重）。

### 8.5 鉴权要求

- **握手阶段**：`PlusWebSocketInterceptor` 用 Sa-Token 校验 token，未认证连接直接拒绝（复用现有实现）。
- **订阅接口**：用 `LoginHelper.getUserId()` 识别调用者，**无需客户端传身份标识**，从源头杜绝伪造。
- **可选**：校验当前用户是否有权订阅某模块（如按租户/角色），越权拒绝。

### 8.6 推送异常处理

- 推送时发现会话已失效（连接断开但尚未清理）→ **跳过并清理脏会话**，不抛异常。
- 单条推送失败 → 记录日志，**不阻断 MQ 消费**（正常 ack，避免消息堆积）。
- 消费端对消息体仅做外层 JSON 反序列化，`message` 字段**原样透传**，不做业务解析。

### 8.7 心跳与重连

- 心跳复用现有 ping/pong 逻辑（`PlusWebSocketHandler.handlePongMessage`）。
- 前端断线指数退避重连；**重连成功后重新订阅**（连接断开期间订阅可能已被清理）。

---

## 九、注意事项

| 注意点 | 说明 |
|--------|------|
| **单连接复用** | 主应用只建一个连接，子应用不得各自直连后台，避免连接风暴 |
| **会话内存化 + 订阅 Redis 化** | 会话存各实例内存，靠 RabbitMQ fanout 保证消息必达持有会话的实例；订阅存 Redis 共享，二者 key 统一为 userId |
| **fanout 拓扑正确性** | 每实例独立 autoDelete 队列绑定 fanout，避免用共享队列造成消息被轮询到单实例而漏推 |
| **前缀匹配正确性** | 必须按**分段边界**匹配，避免 `his/order` 误匹配 `his/orderX` 这类前缀误判 |
| **订阅不展开通配符** | 订阅按精确主题存储，路由时对消息主题生成前缀查询，而非对订阅主题展开，避免 Redis 冗余 |
| **订阅生命周期** | 断开必须清理订阅（走 `ws:userSub:{userId}` 反向索引），否则 Redis 残留脏数据导致推送空转 |
| **重连后重新订阅** | 连接重建后订阅可能已被清理，需前端重新订阅（userId 不变） |
| **广播语义** | `type` 与 `sessionKeys` 均为空 = 广播，需在消费端显式区分 |
| **消息 JSON 规范** | MQ 消息与 WebSocket 推送均使用 JSON；`message` 字段为字符串，前端需二次 JSON.parse |
| **主题命名统一** | 主题统一由 `WebSocketTopicUtils` / `WebSocketTopic` 常量生成，禁止业务侧手写字符串拼接 |
| **与 ruoyi-common-bus 隔离** | websocket 走独立 `websocket.exchange`，不占用 Spring Cloud Bus 事件总线，避免消息混淆 |
| **微前端消息总线** | 复用已有 mitt 总线，事件命名统一（如 `ws:{type}`），避免与现有事件冲突 |
