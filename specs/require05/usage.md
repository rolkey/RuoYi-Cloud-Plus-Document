# 新增子模块实时消息接入指南

> 面向**新增子模块**的接入说明：新增一个后端业务模块（微服务）或一个前端微应用（子应用）时，如何收发实时消息。
> 核心原则：**接入方只使用高层接口，无需关心 WebSocket 连接、RabbitMQ、会话、心跳、重连等通讯细节**——这些由「主应用 + `ruoyi-common-websocket` 模块」统一封装。

---

## 一、为什么不用关心通讯细节

实时信息通道做了三层封装，接入方只接触最上层：

```
┌─────────────────────────────────────────────┐
│  接入方（新增子模块）                          │
│  · 后端业务模块：发布消息到消息服务(RabbitMQ)      │
│  · 前端微应用：    调用 msgBus.on / emit       │
├─────────────────────────────────────────────┤
│  通讯层（对接入方透明，无需了解）                │
│  · 主应用：WebSocket 建连 / 心跳 / 重连 / 鉴权  │
│  · 后端：RabbitMQ 分发 / 会话管理 / 订阅路由    │
└─────────────────────────────────────────────┘
```

接入方需要知道的只有三件事：**发给谁（主题）、发什么（消息体）、怎么收（事件）**。

---

## 二、后端新增业务模块

### 2.1 一次性接入（依赖）

业务模块**只连接消息服务（RabbitMQ）**，不引用 websocket 模块：

```xml
<!-- 消息服务：RabbitMQ -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
<!-- 共享消息 DTO + 常量（轻量，只含数据结构与主题工具，无连接/订阅逻辑） -->
<dependency>
    <groupId>org.dromara</groupId>
    <artifactId>ruoyi-api-websocket</artifactId>
</dependency>
```

> `ruoyi-api-websocket` 只提供消息 DTO、交换机/主题常量与主题工具；WebSocket 连接、订阅、推送逻辑在 `ruoyi-common-websocket` 模块内，业务侧不引用。

### 2.2 发送实时消息（发布到消息服务即可）

```java
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.dromara.common.json.utils.JsonUtils;
import org.dromara.websocket.api.constant.WebSocketConstants;
import org.dromara.websocket.api.dto.WebSocketMessageDto;
import org.dromara.websocket.api.utils.WebSocketTopicUtils;

@Autowired
private RabbitTemplate rabbitTemplate;

// 构建主题：his/order/10086（三级：主类/子类/业务ID）
String topic = WebSocketTopicUtils.build("his", "order", String.valueOf(orderId));

// 构建消息体（JSON 字符串）
String body = JSONUtil.toJsonStr(Map.of("event", "order.paid", "orderId", orderId));

// 发布到消息服务：订阅了 his / his/order / his/order/10086 的客户端都会收到
WebSocketMessageDto dto = new WebSocketMessageDto();
dto.setType(topic);
dto.setMessage(body);
rabbitTemplate.convertAndSend(WebSocketConstants.WEBSOCKET_EXCHANGE, "", JsonUtils.toJsonString(dto));
```

按需两种变体：

```java
dto.setSessionKeys(List.of(1001L, 1002L)); // 定向：指定 userId（优先级最高）
dto.setType(null);                          // 不设主题：广播给全部在线客户端
rabbitTemplate.convertAndSend(WebSocketConstants.WEBSOCKET_EXCHANGE, "", JsonUtils.toJsonString(dto));
```

### 2.3 你不需要关心的

- 不用引用 `ruoyi-common-websocket` 模块，更不用写任何 WebSocket 代码
- WebSocket 连接如何建立、谁在线、连在哪个实例
- RabbitMQ 消息如何被消费、如何路由到目标实例
- 订阅关系存储在哪、如何前缀匹配
- 会话的增删、心跳、断线清理

---

## 三、前端新增微应用（子应用）

### 3.1 前提（已有，无需重复做）

- 子应用已按现有 qiankun 流程注册（`microApps.json` / `appList.ts`）
- 主应用已把消息总线 `msgBus` 注入子应用（props 或 `window.__QIANKUN_MSG_BUS__`）
- 主应用已完成 WebSocket 建连与订阅（由主应用统一负责）

### 3.2 接收实时消息（就这一句）

```ts
// 获取主应用注入的消息总线
const msgBus = window.__QIANKUN_MSG_BUS__; // 或从 props 取

// 订阅业务主题，收到推送即触发
msgBus.on('ws:his/order', (data) => {
  console.log('订单推送', data);
});
```

### 3.3 申请订阅 / 发送消息

```ts
// 申请订阅某主题（由主应用转交后端，子应用不直接连后端）
msgBus.emit('m_wsSubscribe', ['his/order']);

// 发送业务消息（由主应用统一经 WebSocket 上送）
msgBus.emit('m_wsSend', { type: 'his/order/submit', data: { orderId: 10086 } });
```

### 3.4 你不需要关心的

- WebSocket 地址、鉴权 token、握手流程
- 心跳保活、断线重连、重连后重新订阅
- 消息总线底层实现、跨应用事件如何分发
- 后端如何把消息路由到你的子应用

---

## 四、接入清单

**后端新增业务模块**
1. `pom.xml` 引入 `spring-boot-starter-amqp` + 共享 `ruoyi-api-websocket`
2. 需要推送处：`rabbitTemplate.convertAndSend(交换机, "", dto)` 发布消息

**前端新增微应用**
1. 按 qiankun 流程注册子应用（沿用现有步骤）
2. 接收消息：`msgBus.on('ws:{主题}', handler)`
3. 需要订阅/上送：`msgBus.emit('m_wsSubscribe' / 'm_wsSend', ...)`

> 消息发布走 RabbitMQ；WebSocket 连接、心跳、重连、订阅路由、鉴权等通讯细节一律由主应用与 `ruoyi-common-websocket` 模块处理，接入方无需关心。
