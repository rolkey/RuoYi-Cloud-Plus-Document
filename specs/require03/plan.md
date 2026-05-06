# 租户功能调整实施计划

> 基于 `document/specs/require03/requirement.md` 的代码分析

---

## 一、需求理解

1. **上下级租户**：`sys_tenant` 表新增 `parent_tenant_id` 字段，最多两级关系（不能多级）
2. **用户功能调整**：用户必须属于一级租户，可以给用户分配二级租户的功能

---

## 二、涉及文件清单

### 后端（RuoYi-Cloud-Plus）

| # | 文件 | 改动量 | 说明 |
|---|------|--------|------|
| 1 | `sql/ry-cloud.sql` — `sys_tenant` DDL | 1行 | 新增 `parent_tenant_id` 列 |
| 2 | `sql/ry-cloud.sql` — seed data INSERT | 少量 | 种子数据补充新字段 |
| 3 | `.../system/domain/SysTenant.java` | 1行 | 新增 `parentTenantId` 字段 |
| 4 | `.../system/domain/bo/SysTenantBo.java` | 2-3行 | 新增字段 + validation |
| 5 | `.../system/domain/vo/SysTenantVo.java` | 1行 | 新增字段 + Excel 导出注解 |
| 6 | `.../system/mapper/SysTenantMapper.java` | 0行 | MyBatis-Plus 自动映射，无需改动 |
| 7 | `.../system/service/ISysTenantService.java` | 2-3方法 | 新增 `queryTenantTree()`、`queryByParentId()` 等 |
| 8 | `.../system/service/impl/SysTenantServiceImpl.java` | ⭐核心改动 | `insertByBo` 支持 parent_tenant_id、查询树、删除校验子租户、限制两级深度 |
| 9 | `.../system/controller/system/SysTenantController.java` | 少量 | 可选新增获取租户树 API |
| 10 | `.../common/tenant/handle/PlusTenantLineHandler.java` | 0行 | 行级租户隔离已通过 `tenant_id` 列工作，无需改动 |
| 11 | `.../system/domain/SysUserTenant.java` | 0行 | 已有 `userId`+`tenantId` 映射，直接复用 |

### 前端（RuoYi-Cloud-Plus-UI）

| # | 文件 | 改动量 | 说明 |
|---|------|--------|------|
| 12 | `src/api/system/tenant/types.ts` | 2行 | `TenantVO`、`TenantForm`、`TenantQuery` 加 `parentTenantId` |
| 13 | `src/api/system/tenant/index.ts` | 少量 | 可选新增 `listTenantTree()` 方法 |
| 14 | `src/views/system/tenant/index.vue` | ⭐⭐核心改动 | 表单加父租户字段、列表改为树形表格 |
| 15 | `src/api/system/user/types.ts` | 少量 | 区分一级租户与二级租户字段 |
| 16 | `src/views/system/user/index.vue` | 中等 | 加一级租户必选、二级租户多选限制为二级租户 |
| 17 | `src/lang/zh_CN.ts` | 少量 | 新增 i18n 翻译 |
| 18 | `src/lang/en_US.ts` | 少量 | 新增 i18n 翻译 |

### 无需改动的模块

| 模块 | 理由 |
|------|------|
| `RuoYi-Cloud-Plus-UI-HIS/` | HIS 子应用没有租户页面，路由动态加载 |
| `RuoYi-Cloud-Plus-UI-Base/` | 仅有全局 i18n 字符串，无租户视图 |
| `RuoYi-Cloud-Plus-Base/` | 简化版 WIP，此计划针对上游参考版 `RuoYi-Cloud-Plus` 制定 |

---

## 三、详细实施步骤

### Phase 1：数据库层

**Step 1.1** — `sql/ry-cloud.sql` 的 `sys_tenant` CREATE TABLE 中添加列：

```sql
parent_tenant_id bigint DEFAULT NULL COMMENT '父租户ID'
```

放在 `account_count` 和 `status` 之间（或 `id` 后），参考列序。

**Step 1.2** — 种子数据 INSERT 补充对应占位列。

### Phase 2：后端实体/DTO 层

**Step 2.1** — `SysTenant.java` 加字段：

```java
/**
 * 父租户ID
 */
private Long parentTenantId;
```

**Step 2.2** — `SysTenantBo.java` 加字段 + 校验注解。可空——顶级租户无 parent。

**Step 2.3** — `SysTenantVo.java` 加字段 + `@Excel` 导出注解。

### Phase 3：后端服务层（核心业务逻辑）

**Step 3.1** — `ISysTenantService` 新增方法：

```java
List<SysTenantVo> queryTenantTree();                    // 租户树
List<SysTenantVo> queryByParentId(Long parentTenantId); // 查子租户
```

**Step 3.2** — `SysTenantServiceImpl` 改动：

- `insertByBo()` — 接收并持久化 `parentTenantId`，校验 parent 存在
- `updateByBo()` — 不允许修改 `parentTenantId`（或限制场景）
- `deleteWithValidByIds()` — 如果某租户有子租户，禁止删除
- 新增私方法 `validateTenantHierarchy()` — 验证两级深度：
  - 如果 `parentTenantId` 不为空，检查 parent 本身没有 parent（防止 3 级）
  - 不允许循环引用（parent == self）
- 新增 `queryTenantTree()` — 构建树形结构
- `buildQueryWrapper()` — 若传入 `parentTenantId` 参数，按父 ID 过滤

**Step 3.3** — `SysTenantController` 改动：

- 可选：新增 `GET /tenant/tree` 端点返回租户树
- `POST /tenant` 和 `PUT /tenant` 传递 `parentTenantId`（BO 已支持则自动映射）

### Phase 4：前端租户管理页

**Step 4.1** — `types.ts` 类型补充 `parentTenantId?: number`

**Step 4.2** — `index.vue` 改动：

- 表单 dialog 新增"上级租户"字段（树形选择器 `el-tree-select` 或普通 `el-select`）
  - 新建租户时可选
  - 顶级租户（无 parent）该项为空 / 隐藏
  - 编辑时不可修改（或有限制）
- 租户列表可新增"父租户"列
- 考虑改为树形表格展示租户层级

### Phase 5：前端用户管理页

需求：用户必须属于一级租户 + 可分配二级租户功能

**Step 5.1** — `types.ts` 改动：

- `UserForm` 区分 `level1TenantId: string`（一级租户，必选）与 `tenantIds: string[]`（二级租户，可选）
- 实际可复用一个字段加过滤逻辑，取决于后端设计

**Step 5.2** — `index.vue` 改动：

- 用户表单增加独立"所属一级租户"字段（单选框，必填）
- 现有"关联租户"多选改为只列二级租户
- 创建用户时校验必选一级租户

### Phase 6：后端用户-租户关联逻辑

**Step 6.1** — 确认 `SysUserServiceImpl` 中的用户创建 / 更新逻辑如何处理 `tenantId` 和 `tenantIds`：

- `insertUser()` 应该确保用户有所属一级租户（`tenantId` 不为空）
- `updateUser()` 同理
- `sys_user_tenant` 表关联的应该是二级租户

### Phase 7：验证

- 后端编译：`cd RuoYi-Cloud-Plus && mvn compile -pl ruoyi-modules/ruoyi-system -am`
- LSP diagnostics clean on changed files
- 若有测试，运行相关测试

---

## 四、潜在风险与建议

| 风险 | 建议 |
|------|------|
| `PlusTenantLineHandler` 自动追加 `tenant_id` 过滤——查询子租户时可能被当前租户上下文干扰 | 在 `queryByParentId()` 中使用 `TenantHelper.ignore()` 或 `TenantHelper.dynamic()` 绕过行级过滤 |
| 前端 `listTenant()` 返回所有租户，但用户应只看到可见范围的租户 | 后端需根据登录用户角色决定返回范围（superadmin 全量，普通租户看自己的子租户） |
| 两级限制：禁止三级嵌套 | 验证逻辑需完整覆盖：创建、更新、批量导入场景 |
| 删除租户时级联处理（子租户 → 用户 → 用户关联） | 删除前检查所有后代租户及其用户，提示清理顺序 |
| `SysTenant` 的 `tenant_id` 是 `varchar(20)`，而 `parent_tenant_id` 建议用 `bigint`（引用 `id`） | 保持一致：用 `bigint` 引用 `id`（主键），而非 `tenant_id`，避免级联问题 |

---

## 五、实施顺序

```
Phase 1 (DB)
  → Phase 2 (Entity/BO/VO)
    → Phase 3 (Service/Controller)
      → Phase 4 (前端租户) ── 可并行 ──→ Phase 5 (前端用户)
        → Phase 6 (后端用户关联)
          → Phase 7 (验证)
```

前后端可部分并行：Phase 1-3 完成后端，Phase 4-5 并行做前端。
