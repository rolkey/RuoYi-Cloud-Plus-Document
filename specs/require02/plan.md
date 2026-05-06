# 跨租户权限规划

**关联需求**: RQ-001（扩展），与 RQ-001-001（跨科室）类似，增加跨租户能力
**状态**: 规划中
**版本**: 1.0.0
**最后更新**: 2026-05-04

## 概述

在完成了 RQ-001-001（人员多科室权限）之后，需要进一步支持人员多租户权限。与科室权限不同，租户权限涉及架构级别的数据隔离，更复杂。

## 背景：现有租户体系

### 当前架构

- 每个用户属于**唯一**租户，通过 `TenantEntity.tenantId` 继承字段
- MyBatis-Plus `TenantLineInnerInterceptor` 在**所有 SQL 查询**自动拼接 `AND tenant_id = 'xxx'`
- 4 层隔离：SQL（拦截器）、Redis（key 前缀）、Cache（缓存管理器）、SaToken（DAO）
- 动态租户切换（`TenantHelper.dynamic()`）目前仅给超级管理员临时查看用
- **无**用户-多租户关联表

### 与科室（dept）的关键区别

| 维度 | 科室（已实现） | 租户（规划中） |
|------|-------------|---------------|
| 隔离机制 | `@DataPermission` AOP → `dept_id IN (...)` | `TenantLineInnerInterceptor` → `tenant_id = ?` |
| 隔离层级 | 查询级别（SQL 注解） | 架构级别（MP 拦截器，全局生效） |
| 涉及表范围 | 仅标注了 `@DataPermission` 的 Mapper | 所有继承 `TenantEntity` 的表（20+ 张） |
| 改造难度 | 低（修改 PlusDataPermissionHandler） | 高（需修改或绕过 TenantLineInnerInterceptor） |

## 设计方案

### 方案对比

#### 方案 A：动态 IN 列表（推荐）

修改 `PlusTenantLineHandler`，当用户有多个可访问租户时，返回 `InExpression` 而非单值 `StringValue`。

```java
// 修改 PlusTenantLineHandler.getTenantId()
public Expression getTenantId() {
    LoginUser loginUser = LoginHelper.getLoginUser();
    if (loginUser != null && loginUser.getAllTenantIds().size() > 1) {
        List<String> allIds = loginUser.getAllTenantIds();
        // 生成: tenant_id IN ('000000', '974105', '318705')
        return buildInExpression(allIds);
    }
    // 单租户走原有逻辑
    return new StringValue(TenantHelper.getTenantId());
}
```

**优点**：用户体验好，自动可见所有授权租户的数据
**缺点**：MyBatis-Plus 的 `TenantLineHandler` 接口设计为返回单值，可能需要自定义拦截器
**风险**：中

#### 方案 B：自定义租户拦截器（更可控）

不用 `TenantLineInnerInterceptor`，在 `PlusDataPermissionInterceptor` 中增加租户过滤逻辑。

**优点**：完全可控，与数据权限体系一致
**缺点**：需要确保覆盖所有查询，改动量大
**风险**：高

#### 方案 C：动态切换租户（快速实现）

前端增加租户切换 UI，用户手动切换当前活跃租户，调用 `TenantHelper.setDynamic()`。

**优点**：改动极小，后端几乎不需要改动
**缺点**：用户一次只能在一个租户下操作
**风险**：低

#### 方案 D：数据库层按租户分库（适合 SaaS 场景）

每个租户独立数据库，通过动态数据源切换。当前项目不适用（已设计为单库+tenant_id 列）。

### 推荐方案：A（自动合并）+ C（手动切换）组合

- **默认行为**：使用方案 A，用户自动看到所有授权租户的数据（过滤合并）
- **备选行为**：用户可手动切换到特定租户（方案 C），此时仅看该租户数据
- **实现路径**：先实现方案 C 快速可用，再演进到方案 A

## 影响范围

### 第一阶段：基础关联（参考 RQ-001-001 实现）

| 序号 | 层 | 文件 | 操作 | 说明 |
|------|----|------|------|------|
| 1 | SQL | `sql/ry-cloud.sql` | 新增表 | `sys_user_tenant(user_id, tenant_id)` |
| 2 | Entity | `SysUserTenant.java` | **新建** | 同 SysUserDept 模式 |
| 3 | Mapper | `SysUserTenantMapper.java` | **新建** | BaseMapperPlus |
| 4 | BO | `SysUserBo.java` | 修改 | 增加 `tenantIds: String[]` |
| 5 | VO | `SysUserInfoVo.java` | 修改 | 增加 `tenantIds: List<String>` |
| 6 | Service | `ISysUserService.java` | 修改 | 增加 `selectUserTenantIds()` |
| 7 | Service | `SysUserServiceImpl.java` | 修改 | insert/update/delete 增加关联租户 CRUD |
| 8 | Controller | `SysUserController.java` | 修改 | 新增 `GET/PUT /{userId}/tenants` |
| 9 | DTO | `LoginUser.java` | 修改 | 增加 `tenantIds` + `getAllTenantIds()` |
| 10 | Dubbo | `RemoteUserServiceImpl.java` | 修改 | buildLoginUser 填充 tenantIds |
| 11 | DeptService | `SysTenantServiceImpl.java` | 修改 | 删除租户时检查 sys_user_tenant |

### 第二阶段：数据权限改造

| 序号 | 层 | 文件 | 操作 | 说明 |
|------|----|------|------|------|
| 12 | 拦截器 | `PlusTenantLineHandler.java` | 修改 | 支持多租户 InExpression |
| 13 | 拦截器 | `TenantConfiguration.java` | 可能修改 | 注册自定义拦截器（如需） |
| 14 | 上下文 | `TenantHelper.java` | 修改 | 增加多租户列表支持 |

### 第三阶段：前端

| 序号 | 文件 | 操作 | 说明 |
|------|------|------|------|
| 15 | `user/types.ts` | 修改 | UserForm/UserInfoVO 增加 tenantIds |
| 16 | `user/index.ts` | 修改 | 新增 getUserTenants/updateUserTenants API |
| 17 | `user/index.vue` | 修改 | 表单增加"关联租户"多选组件 |
| 18 | Navbar/下拉菜单 | 修改 | 增加"切换租户"功能 |

## 与已实现的跨科室功能的差异点

1. **字段类型不同**：`deptId` 是 `Long`，`tenantId` 是 `String`（6 位编号）。BO 中的 `tenantIds` 应为 `String[]`，VO 中的 `tenantIds` 应为 `List<String>`
2. **多租户列表的数据源**：关联科室的数据来自 `/system/user/deptTree`（部门树）；关联租户的数据来自 `/system/tenant/list`（租户列表）
3. **前端组件**：关联科室用 `el-tree-select`（树形选择）；关联租户用 `el-select` 多选（扁平列表）
4. **租户隔离更强**：租户不能简单地"展开子节点"，需要独立的数据权限改造

## 待讨论的问题

1. **跨租户数据权限的策略**：用户关联了租户 A 和 B，在租户 A 下的数据权限（如仅本人/本部门/自定义）是否应该独立配置？还是使用统一的角色数据权限？
2. **默认活跃租户**：用户在关联多个租户后，新建数据应该属于哪个租户？默认是主属租户？
3. **租户套餐限制**：用户关联的租户是否受套餐的 `account_count` 限制？
4. **与 HIS 模块的关系**：HIS 业务数据是否需要跨租户？还是仅系统管理功能需要？
