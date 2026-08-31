# 实时信息通道实施计划

> 基于 `document/specs/require05/requirement.md` 的架构设计（不涉及具体代码，按 RuoYi-Cloud-Plus 微服务架构经验设计）

---

## 一、需求理解

本需求要建立一条**服务器与客户端之间的实时信息通道**，核心要点：

1. **协议**：采用 WebSocket 协议。
2. **连接模型**：服务器与每个微前端客户端**只建立一个连接**（单连接复用），由主服务统一负责与客户端交互。
3. **多微服务推送**：后台为若依微服务架构，多个微服务（system、job、workflow、HIS 等）都能向客户端推送信息，并支持**广播**。
4. **服务间通信**：各微服务之间通过 **RabbitMQ** 实时交互，消息格式为 **JSON**。
5. **前端交互**：微前端客户端通过**消息总线（mitt）**与主微前端主应用交互，由主应用统一负责与后台建立/维护 WebSocket 连接并分发消息。
6. **消息订阅**：客户端可通过**业务接口**向服务器订阅消息；服务器收到 RabbitMQ 消息时，**根据订阅情况**仅向订阅了对应主题的客户端发送。
7. **订阅粒度（三级）**：订阅主题按 **主类（模块）/ 子类（业务）/ 具体业务ID** 三级划分，客户端可按任意粒度订阅。

> 结论：这是一条 **WebSocket + RabbitMQ + Redis + 微前端消息总线** 的实时推送链路，与 RuoYi-Cloud-Plus 原生 `ruoyi-websocket` 模块的设计一致，需新建一个独立的 websocket 服务模块；在连接复用的基础上新增**「三级主题订阅 + 前缀匹配路由」**能力。

---

## 二、总体架构设计

### 2.1 后端链路（推送方向：微服务 → 客户端）

```
┌──────────────┐   publish(JSON)   ┌──────────────┐
│ ruoyi-system │ ────────────────▶ │              │
│ ruoyi-job    │ ────────────────▶ │   RabbitMQ   │
│ ruoyi-his    │ ────────────────▶ │  (topic 交换) │
└──────────────┘                   └──────┬───────┘
                                          │ consume(JSON)
                                          ▼
                                ┌──────────────────┐
                                │  ruoyi-websocket │  ← 主服务，唯一负责与客户端交互
                                │  (单/多实例)      │  ← 订阅表存 Redis，按三级前缀匹配路由
                                └────────┬─────────┘
                                         │ WebSocket 推送（会话+订阅查询走 Redis）
                                         ▼
                                ┌──────────────────┐
                                │  微前端主应用      │  ← 只建一个连接
                                └──────────────────┘
```

- **生产端**：任意业务微服务调用 `WebSocketUtils.publishMessage(...)`，把消息序列化成 JSON 发到 RabbitMQ topic 交换机。
- **消费端**：`ruoyi-websocket` 服务订阅 RabbitMQ 队列，消费消息后**先按 `sessionKeys`（定向）→ `type`（三级订阅前缀匹配）→ 广播**的优先级，从 Redis 查出目标客户端会话，推送 JSON 文本。
- **会话/订阅存储**：WebSocket 会话元数据与订阅关系均存 Redis（Redisson），解决多实例部署时"消息落到哪个实例 / 订阅谁维护"的问题。

### 2.2 前端链路（推送方向：客户端 → 微服务 / 微服务 → 客户端）

```
┌───────────────┐     消息总线 mitt       ┌──────────────┐
│ 微前端子应用    │ ◀──────────────────▶ │ 微前端主应用  │
│ (HIS / 子应用) │   subscribe/publish    │ (RuoYi-UI)   │
└───────────────┘                        └──────┬───────┘
                                                │ 唯一 WebSocket 连接
                                                │ 订阅/退订走 REST 业务接口
                                                ▼
                                         ┌────────────────┐
                                         │ ruoyi-websocket│
                                         └────────────────┘
```

- 主应用维护**唯一** WebSocket 连接，负责建连、心跳、重连、鉴权。
- 子应用通过主应用暴露的消息总线（mitt，已存在）订阅/发送消息，自身不直接连接后台。
- 主应用（或子应用经主应用转发）通过 **REST 业务接口**向服务器声明订阅哪些主题（三级粒度）。

### 2.3 订阅路由流程

```
① 客户端订阅             ┌──────────────────────────────────────────────┐
  POST /subscribe ────▶ │ 写入 Redis:  ws:sub:{topic} → Set<clientKey> │
  {clientKey,types}     │              ws:clientSub:{key} → Set<topic> │
                        └──────────────────────────────────────────────┘

② 服务推送                                                ③ 按前缀匹配路由
  业务服务 publishMessage ─▶ RabbitMQ ─▶ Consumer ─▶ 拆分 topic 生成前缀
                                                       ─▶ 并集各 ws:sub:{前缀} 的 clientKey
                                                       ─▶ 推送
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
  "sessionKeys": [],
  "message": "{\"orderId\":10086,\"status\":\"paid\"}",
  "source": "ruoyi-his"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `type` | `String` | 是 | 三级主题路径，用于订阅前缀匹配 |
| `sessionKeys` | `List<String>` | 否 | 目标 clientKey 列表（定向推送，优先级最高，见 4.4） |
| `message` | `String` | 是 | 业务消息体（JSON 字符串，由业务服务自定义，服务端原样透传） |
| `source` | `String` | 否 | 消息来源服务名（用于追踪/日志） |

### 4.2 订阅接口（REST，由 websocket 服务提供）

**订阅：**

```
POST /websocket/subscribe
请求体：
{
  "clientKey": "a1b2c3d4",
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
  "clientKey": "a1b2c3d4",
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
GET /websocket/subscription/{clientKey}
响应：
{
  "code": 200,
  "msg": "操作成功",
  "data": ["system", "his/order/10086"]
}
```

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
| 1 | `sessionKeys` 非空 | 定向推送给指定 clientKey 列表（跳过订阅匹配） |
| 2 | `sessionKeys` 为空 且 `type` 非空 | 按 `type` 做三级前缀匹配，推送给命中的订阅客户端 |
| 3 | `sessionKeys` 为空 且 `type` 为空 | 广播全部在线客户端 |

### 4.5 会话与订阅存储（Redis）

| Key | 类型 | 说明 |
|-----|------|------|
| `ws:session:{clientKey}` | String(JSON) | 会话元数据（userId、心跳时间、实例标识等） |
| `ws:sub:{topic}` | Set | 该**精确主题**已订阅的 clientKey 集合（正向索引，topic 为 1~3 段） |
| `ws:clientSub:{clientKey}` | Set | 该客户端订阅的全部主题（反向索引，断开清理用） |

> 说明：订阅按「精确主题」逐条存储，**不做通配符展开**。路由时由消费端将消息主题拆成全部前缀再逐条查询（见 8.2），保证主类/子类/业务ID 三种粒度都能命中。

---

## 五、涉及模块 / 文件清单

### 5.1 后端（新建 `ruoyi-websocket` 模块）

| # | 类 / 配置 | 职责 |
|---|-----------|------|
| 1 | `pom.xml`（父 pom 注册模块） | 注册 websocket 服务，引入 spring-boot-starter-websocket、amqp、redisson、sa-token 鉴权依赖 |
| 2 | `WebSocketApplication` | 服务启动入口 + `@EnableAsync` |
| 3 | `WebSocketConfigurator` / `WebSocketEndpoint` | `@ServerEndpoint("/websocket/{clientKey}")`，握手时鉴权、注册会话 |
| 4 | `WebSocketSession` | 会话包装类，持有原始 Session + 心跳时间 + clientKey |
| 5 | `WebSocketSessionService`（接口）+ `PlusWebSocketSessionService` | 会话的增删查，基于 Redis 存储，支持跨实例查询 |
| 6 | `WebSocketSubscribeService` | 三级订阅关系的增删查：维护 `ws:sub:{topic}` 与 `ws:clientSub:{key}`，断开时清理 |
| 7 | `WebSocketSubscribeController` | 订阅/退订/查询 REST 接口（对应 4.2） |
| 8 | `WebSocketTopicUtils` | 主题工具：拼接、拆分、校验、生成前缀列表（见 8.1/8.2） |
| 9 | `WebSocketTopic`（枚举/常量） | 定义主类（模块）常量 + 常见子类，统一主题命名，避免手写拼接错误 |
| 10 | `WebSocketUtils` | 静态工具：`publishMessage(topic, sessionKey, message)`、`publishAll(topic, message)`，内部走 RabbitTemplate |
| 11 | `WebSocketConsumer` | `@RabbitListener` 消费队列，按路由优先级（定向→订阅前缀匹配→广播）查会话并推送 |
| 12 | `WebSocketMessageDto` | 服务间消息 DTO（对应 4.1） |
| 13 | `application.yml` | RabbitMQ、Redis、WebSocket 端口、心跳超时等配置 |

### 5.2 后端（业务侧调用，仅新增少量代码）

| # | 位置 | 改动 |
|---|------|------|
| 14 | 各业务服务（system/his 等） | 需要推送处调用 `WebSocketUtils.publishMessage(...)`，主题用 `WebSocketTopicUtils` 构建，无需感知连接/订阅细节 |

### 5.3 前端（主应用 + 子应用）

| # | 文件 | 改动 |
|---|------|------|
| 15 | 主应用 `src/utils/websocket.ts` | WebSocket 客户端单例：建连、心跳、重连、收发 |
| 16 | 主应用 `src/micro/messageBus.ts`（已有） | 复用 mitt 总线，主应用收到服务端消息后按 `type` 派发；同时监听子应用上送 |
| 17 | 主应用订阅 API 封装（`src/api/websocket/index.ts`） | 封装 subscribe / unsubscribe / 查询接口，供主应用或子应用调用 |
| 18 | 子应用（如 HIS） | 订阅总线对应事件接收推送；通过总线 `emit` 上送消息、申请订阅 |
| 19 | 鉴权/登录处 | 登录成功后生成 `clientKey` 并发起连接、初始订阅 |

---

## 六、详细实施步骤

### Phase 1：后端基础设施（ruoyi-websocket 模块）

**Step 1.1 — 模块骨架**
- 在 `ruoyi-visual/ruoyi-websocket/`（或等价位置）新建模块，父 pom 注册。
- 引入依赖：`spring-boot-starter-websocket`、`spring-boot-starter-amqp`、`redisson-spring-boot-starter`、`sa-token`、Nacos 注册/配置中心。

**Step 1.2 — 主题工具**
- 实现 `WebSocketTopicUtils`：`build(module, business, id)` 拼接、`split(topic)` 拆分、`validate(topic)` 校验、`prefixes(topic)` 生成前缀列表。
- 定义 `WebSocketTopic` 常量（主类 + 常见子类）。

**Step 1.3 — 会话管理**
- 实现 `WebSocketSession`（持有 clientKey、Session、心跳时间戳）。
- 实现 `PlusWebSocketSessionService`，用 Redis 存 `ws:session:{clientKey}`，提供 `addSession / removeSession / getSessionsByKeys / getSessionKeyList`。

**Step 1.4 — 端点与鉴权**
- `@ServerEndpoint("/websocket/{clientKey}")`，`@OnOpen` 校验 token、注册会话；`@OnClose`/`@OnError` 移除会话；`@OnMessage` 处理心跳（ping/pong）与客户端上送消息。

**Step 1.5 — 订阅管理**
- 实现 `WebSocketSubscribeService`：`subscribe(clientKey, types)` / `unsubscribe(clientKey, types)` / `getSubscribedClientKeys(topic)` / `cleanup(clientKey)`。
- 维护双向 Redis 索引（4.5 节）；`@OnClose` 时调用 `cleanup` 清理订阅。
- 实现 `WebSocketSubscribeController` 暴露 REST 接口（4.2 节）。

**Step 1.6 — RabbitMQ 生产/消费**
- 定义 `WebSocketMessageDto`；`WebSocketUtils` 用 `RabbitTemplate.convertAndSend` 发送 JSON（Jackson 序列化）。
- `WebSocketConsumer` 用 `@RabbitListener` 监听队列，按路由优先级（4.4 节）匹配订阅并推送。

**Step 1.7 — 配置**
- `application.yml` 配置：MQ 交换机/队列/路由 key、Redis、心跳超时、WebSocket 端口。

### Phase 2：业务侧接入

**Step 2.1 — 打通一条示例链路**
- 选一个业务服务（如 system 的通知公告），在产生事件处调用 `WebSocketUtils.publishMessage(WebSocketTopicUtils.build("system","sysNotice","123"), null, json)`。
- 验证：服务端日志确认消息经 MQ 到 websocket 模块，再按订阅前缀匹配推到客户端。

### Phase 3：前端接入

**Step 3.1 — 主应用 WebSocket 客户端**
- 实现 `websocket.ts` 单例：登录后生成 `clientKey`、建连 `/websocket/{clientKey}`、心跳保活、断线重连（指数退避）、`onmessage` 统一解析 JSON。

**Step 3.2 — 订阅接口封装与初始订阅**
- 实现 `src/api/websocket/index.ts` 封装 subscribe/unsubscribe/查询。
- 登录/建连成功后，根据当前登录用户权限或子应用声明，调用 subscribe 完成初始订阅（可按模块级或业务级粒度）。

**Step 3.3 — 消息总线派发**
- 主应用 `onmessage` 后，按 `type` 通过 mitt `emit` 给子应用。
- 子应用通过 `on(type)` 订阅；需要上送消息或调整订阅时 `emit` 给主应用，主应用统一 `send` / 调订阅接口。

**Step 3.4 — 子应用接入示例**
- HIS 子应用订阅 `his/order`、`his/patient` 等事件，展示推送；必要时上送消息、动态调整订阅粒度（如进入订单详情后订阅 `his/order/10086`）。

---

## 七、实施顺序

```
Phase 1（后端模块+主题工具+会话+订阅+MQ） → Phase 2（业务打通示例链路） → Phase 3（前端连接+订阅+总线派发）
```

- Phase 1 内部：骨架 → 主题工具 → 会话 → 端点 → 订阅管理 → MQ → 配置，需顺序进行（依赖关系）。
- Phase 2 依赖 Phase 1 完成；Phase 3 依赖 Phase 1（后端可连接、可订阅）完成。
- Phase 2 与 Phase 3 可并行（一边打通后端推送，一边搭前端连接与订阅）。

---

## 八、程序处理要求

### 8.1 主题规范化与校验

- 主题由 **1~3 段**组成，用 `/` 分隔，如 `system/sysNotice/123`。
- 每段**非空**，字符限制 `[A-Za-z0-9_-]`（或按约定），**禁止**包含 `/` 以外的分隔符、空白、特殊字符。
- 发布方必须携带**完整三段**（主类/子类/业务ID）；订阅方允许 1~3 段。
- 非法主题：校验不通过则**拒绝并记录日志**（订阅接口返回错误码，发布方抛异常），不得静默丢弃。

### 8.2 订阅匹配与路由算法

- 消费端收到消息后，对 `type` 做**前缀匹配**：
  1. 拆分 `type` 为段数组，如 `["his","order","10086"]`。
  2. 生成全部前缀：`["his", "his/order", "his/order/10086"]`。
  3. 逐个查询 `ws:sub:{前缀}` 集合，**并集去重**得到目标 clientKey 集合。
  4. 对集合内每个 clientKey 查会话并推送；无命中且非定向/广播则丢弃。
- 定向推送（`sessionKeys` 非空）**跳过**订阅匹配，直接按 key 推送。
- 广播（`type` 与 `sessionKeys` 均为空）推送给**全部**在线客户端。

### 8.3 路由优先级

1. `sessionKeys` 非空 → 定向推送（优先级最高）
2. `type` 非空 → 三级前缀匹配订阅推送
3. 均空 → 广播

> 命中即停止，避免同一客户端因同时命中定向与订阅而重复收到消息（实现时对目标集合统一去重）。

### 8.4 订阅生命周期管理

- **subscribe**：校验 clientKey 有效性 → 对每个 topic 校验合法性 → 写入 `ws:sub:{topic}`（SADD，天然幂等去重）与 `ws:clientSub:{key}`。
- **unsubscribe**：从 `ws:sub:{topic}` 移除 clientKey；若 set 为空则删除该 key；同步更新反向索引。
- **disconnect**：读 `ws:clientSub:{key}` 得到其全部订阅主题，逐个从 `ws:sub:{topic}` 移除 clientKey，最后删除反向索引与会话，**不留脏数据**。
- **重复订阅幂等**：同一客户端重复订阅同一主题不产生重复推送（Set 去重）。

### 8.5 鉴权要求

- **握手阶段**：校验 token（Sa-Token），未认证连接直接拒绝。
- **订阅接口**：校验 `clientKey` 归属——`clientKey` 必须与当前已建立的连接（或登录会话）绑定，防止伪造 clientKey 订阅他人消息。
- **可选**：校验当前用户是否有权订阅某模块（如按租户/角色），越权拒绝。

### 8.6 推送异常处理

- 推送时发现会话已失效（连接断开但 Redis 未及时清理）→ **跳过并清理脏会话**，不抛异常。
- 单条推送失败 → 记录日志，**不阻断 MQ 消费**（正常 ack，避免消息堆积）。
- 消费端对消息体仅做外层 JSON 反序列化，`message` 字段**原样透传**，不做业务解析。

### 8.7 心跳与重连

- 服务端记录 `lastHeartbeat`，定时任务清理超时会话并同步清理其订阅。
- 前端断线指数退避重连；**重连成功后重新注册 clientKey 并重新订阅**（连接断开期间订阅可能已被清理）。

---

## 九、注意事项

| 注意点 | 说明 |
|--------|------|
| **单连接复用** | 主应用只建一个连接，子应用不得各自直连后台，避免连接风暴 |
| **多实例会话一致性** | 会话元数据与订阅关系必须放 Redis，否则 MQ 消息可能落到未持有该会话/订阅的实例而推送失败 |
| **前缀匹配正确性** | 必须按**分段边界**匹配，避免 `his/order` 误匹配 `his/orderX` 这类前缀误判 |
| **订阅不展开通配符** | 订阅按精确主题存储，路由时对消息主题生成前缀查询，而非对订阅主题展开，避免 Redis 冗余 |
| **订阅生命周期** | 客户端断开必须清理订阅（走反向索引），否则 Redis 残留脏数据导致推送空转 |
| **重连后重新订阅** | 连接重建后 clientKey 可能变化/订阅被清理，需前端重新订阅 |
| **广播语义** | `type` 与 `sessionKeys` 均为空 = 广播全部在线客户端，需在消费端显式区分 |
| **消息 JSON 规范** | MQ 消息与 WebSocket 推送均使用 JSON；`message` 字段为字符串，前端需二次 JSON.parse |
| **主题命名统一** | 主题统一由 `WebSocketTopicUtils` / `WebSocketTopic` 常量生成，禁止业务侧手写字符串拼接 |
| **幂等/顺序** | 若业务对顺序敏感，注意 MQ 队列多消费者竞争消费，必要时按主类（模块）分队列 |
| **微前端消息总线** | 复用已有 mitt 总线，事件命名统一（如 `ws:{type}`），避免与现有事件冲突 |
