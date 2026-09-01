# 服务隔离测试方法

> 目的：在不中断原有服务的前提下，用一套独立的「测试服务」验证功能；前端只连接测试服务，原有服务保持运行。
> 适用：本项目（RuoYi-Cloud-Plus 微服务 + 微前端）的模块功能测试。

---

## 一、核心机制

整个方法依赖三处「服务名」的联动：

```
Nacos 服务注册名  ──►  网关路由前缀  ──►  前端 service.json 映射
(spring.application.name)   (Path=/xxx/**)      (servicePres.xxx)
```

| 环节 | 作用 | 示例 |
|------|------|------|
| 后端 Nacos | 每个服务以 `spring.application.name` 注册，同名可并存多个实例 | `ruoyi-system` / `ruoyi-system-rk` |
| 网关 | 按 URL 路径前缀路由到对应服务（Nacos 服务发现） | `/system/**` → system，`/system-rk/**` → system-rk |
| 前端 `service.json` | 把「逻辑服务名」映射到「实际路径前缀」 | `"system": "system-rk"` |
| 前端 API 层 | 用 `useServiceStore().servicePres.xxx` 动态拼接 URL | `/${system()}/notice/list` |

> 关键点：前端通过改 `service.json` 就能把某类接口全部切到测试服务，**不用改任何 API 代码**；原服务（`system`）继续对外服务，互不影响。

---

## 二、操作步骤（以 system 服务为例）

### 步骤 1：后端新建测试服务

在 Nacos 新建一份配置（复制原服务配置改名），如 `ruoyi-system-rk.yml`：

- `spring.application.name: ruoyi-system-rk`（关键，决定注册名/路径前缀 `system-rk`）
- 端口换一个，避免与原服务冲突（如原 9201 → 9301）
- 其余配置（数据库、Redis、RabbitMQ 等）按需指向测试库或共用

启动该实例后，Nacos 里会同时存在 `ruoyi-system` 和 `ruoyi-system-rk` 两个服务。

### 步骤 2：网关加路由

在 Nacos 的网关配置（`ruoyi-gateway.yml`）加一条路由，把 `/system-rk/**` 指向新服务：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: ruoyi-system-rk
          uri: lb://ruoyi-system-rk
          predicates:
            - Path=/system-rk/**
```

> 若网关已开启基于服务发现的通用路由（DiscoveryClient 路由），可跳过此步。

### 步骤 3：前端改 service.json

`RuoYi-Cloud-Plus-UI/public/service.json` 中把 system 指向测试服务：

```json
{
  "development": {
    "system": "system-rk"
  }
}
```

### 步骤 4：验证与恢复

- 刷新前端，system 相关接口全部走 `/system-rk/...`，连到测试服务。
- 原 `system` 服务不受影响，其它用户仍连原服务。
- **恢复**：把 `service.json` 改回 `"system": "system"` 即可，测试服务可继续跑或停掉。

---

## 三、前端已支持动态化的服务

当前 `public/service.json` 中的字段（逻辑服务名 → 实际路径前缀）：

| service.json 字段 | 后端服务 | 说明 |
|-------------------|----------|------|
| `auth` | ruoyi-auth | 认证 |
| `system` | ruoyi-system | 系统管理 |
| `workflow` | ruoyi-workflow | 工作流 |
| `tool` | （工具类服务） | 按需 |
| `resource` | ruoyi-resource | 资源服务（WebSocket/SSE/OSS 宿主） |

**注意 `resource` 的现状**（本次实时通讯需求相关）：

| 功能 | 是否走 service.json | 现状 |
|------|--------------------|------|
| WebSocket 连接 | ✅ 已动态化 | `layout/index.vue` 用 `servicePres.resource` |
| WebSocket 订阅接口 | ✅ 已动态化 | `api/websocket/index.ts` 用 `servicePres.resource` |
| SSE 推送 | ❌ 硬编码 `/resource/sse` | `layout/index.vue`、`api/login.ts` |
| OSS 上传/下载 | ❌ 硬编码 `/resource/oss/...` | 约 9 处（api + 组件） |

所以若要给 `resource` 做隔离测试：

- **WebSocket** 部分：直接改 `service.json` 的 `resource` 即可（如 `resource-rk`）。
- **SSE / OSS** 部分：仍需先把手动硬编码的 `/resource/` 改为动态（参照 `api/websocket/index.ts` 的 `resource()` 写法）。

---

## 四、注意事项

1. **命名约定**：测试服务建议统一后缀（如 `-rk`），与原服务、正式服务清晰区分。
2. **端口**：测试服务换端口，避免同机启动冲突。
3. **数据源**：测试服务可共用正式库（只读验证）或指向测试库（避免污染正式数据），按需选择。
4. **网关路由**：`Path` 前缀必须与 `service.json` 里的 value 对应；改完网关配置需刷新/重启网关生效。
5. **前端缓存**：改 `service.json` 后需刷新前端（dev 下可能需重启 dev server）。
6. **恢复**：测试完把 `service.json` 改回原名即可，无需回滚后端。
7. **未动态化的硬编码**：除 `resource` 的 SSE/OSS 外，前端还有个别直接写死 `/resource/...` 的地方，做隔离测试前先确认相关功能是否已动态化。

---

## 五、快速参考

```
① Nacos 新建 ruoyi-xxx-rk.yml（spring.application.name = ruoyi-xxx-rk，换端口）
② 网关加 /xxx-rk/** 路由 → lb://ruoyi-xxx-rk
③ 前端 service.json 里 "xxx": "xxx-rk"
④ 刷新前端 → 测试；改回 "xxx": "xxx" → 恢复
```
