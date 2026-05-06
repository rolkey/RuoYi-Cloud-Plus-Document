# 下级租户功能完善实施计划

> 基于 `document/specs/require04/requirement.md` 的代码分析

---

## 一、需求理解

1. **租户标题全部换成医院** — 前端 UI 中所有 visible 的"租户"文字替换为"医院"，comments 中的也同步修改
2. **SysTenantController.tree 接口只查当前登录用户所在医院的所有下级医院** — 简化：不再构建树形结构，改为查列表（list 方式），只返回当前用户所属医院的下级医院
3. **表格中顶层医院可添加子医院** — 在操作栏中为没有 parent 的医院加一个"添加子医院"（加号）按钮

---

## 二、涉及文件清单

### 需求1：租户→医院 文字替换

| # | 文件 | 改动量 | 说明 |
|---|------|--------|------|
| 1 | `src/views/system/tenant/index.vue` | ~25处 | 主要租户管理页面中"租户"→"医院"、"分院"→"医院" |
| 2 | `src/views/system/user/index.vue` | ~8处 | 用户管理页面中"所属租户"→"所属医院"等 |
| 3 | `src/views/system/tenantPackage/index.vue` | ~8处 | 租户套餐页面中"租户套餐"→"医院套餐"、"分院权限"→"医院权限" |
| 4 | `src/api/system/tenant/index.ts` | ~12处 | API 注释中"租户"→"医院" |
| 5 | `src/api/system/tenantPackage/index.ts` | ~7处 | 同上 |
| 6 | `src/api/system/user/index.ts` | ~3处 | 同上 |
| 7 | `src/api/login.ts` | ~1处 | 注释 |
| 8 | `src/api/types.ts` | ~1处 | 注释 |
| 9 | `src/api/workflow/task/index.ts` | ~2处 | 注释 |
| 10 | `src/lang/zh_CN.ts` | ~1处 | i18n "请输入您的租户编号"→"请输入您的医院编号" |
| 11 | `src/layout/components/Navbar.vue` | ~4处 | 注释 |
| 12 | `src/views/register.vue` | ~2处 | 注释 |
| 13 | `RuoYi-Cloud-Plus-UI-Base/src/lang/zh_CN.ts` | ~1处 | 共享模块 i18n |

### 需求2：tree 接口只查下级医院

| # | 文件 | 改动量 | 说明 |
|---|------|--------|------|
| 14 | `SysTenantController.java` | ⭐核心 | tree 端点改为根据当前用户 tenant 查下级，移除 `@SaCheckRole(SUPER_ADMIN)` 限制 |
| 15 | `SysTenantServiceImpl.java` | ⭐核心 | `queryTenantTree()` 改为 `queryListByParent()` — 根据当前租户查下级返回 list |
| 16 | `LoginHelper.java` | 0行 | 已有 `getTenantId()` 可直接使用 |

### 需求3：表格加号按钮

| # | 文件 | 改动量 | 说明 |
|---|------|--------|------|
| 17 | `src/views/system/tenant/index.vue` | 中等 | 操作栏加"添加子医院"加号按钮，仅对顶级医院显示 |

---

## 三、详细实施步骤

### Phase 1：文字替换（租户→医院）

**Step 1.1** — `tenant/index.vue` 替换（用户可见的优先）：
- 所有 `label` / `placeholder` / `title` / `message` 中的"租户"→"医院"
- "分院"→"医院"
- 注释中的"租户"→"医院"
- 注意 `chooseTenant` / `tenant id` 等代码标识符不改

**Step 1.2** — `user/index.vue` 替换：
- "所属租户"→"所属医院"（label + placeholder）
- 注释中的"租户"→"医院"

**Step 1.3** — `tenantPackage/index.vue` 替换：
- "租户套餐"→"医院套餐"
- "分院权限"→"医院权限"

**Step 1.4** — API 注释文件批量替换：
- `tenant/index.ts`、`tenantPackage/index.ts`、`user/index.ts`、`login.ts`、`types.ts`、`workflow/task/index.ts`

**Step 1.5** — i18n 文件：
- `zh_CN.ts`（主项目 + Base 共享模块）中的"租户编号"→"医院编号"

**Step 1.6** — 其他组件：
- `Navbar.vue`、`register.vue` 中的注释替换

### Phase 2：Tree 接口改造

**Step 2.1** — 修改 `SysTenantServiceImpl`：

当前 `queryTenantTree()` 查所有 → 改为 `queryListByParent` 只查当前租户的下级：
```java
public List<SysTenantVo> queryListByParent(Long parentId) {
    return baseMapper.selectVoList(
        new LambdaQueryWrapper<SysTenant>()
            .eq(SysTenant::getParentTenantId, parentId)
            .orderByAsc(SysTenant::getId));
}
```
注意：当前登录用户的 tenant_id（String）不等于 sys_tenant 的 id（Long），需要先查出当前用户所属租户的 id，再用 id 查下级。

**Step 2.2** — 修改 `SysTenantController.tree()`：

```java
// 取消 SUPER_ADMIN 限制，改为可被普通医院管理员访问
// @SaCheckRole(TenantConstants.SUPER_ADMIN_ROLE_KEY)
@SaCheckPermission("system:tenant:list")
@GetMapping("/tree")
public R<List<SysTenantVo>> tree() {
    // 获取当前用户所属租户的 tenantId
    String tenantId = LoginHelper.getTenantId();
    // 通过 tenantId 查找到当前租户的 id
    SysTenantVo currentTenant = tenantService.queryByTenantId(tenantId);
    if (currentTenant == null) {
        return R.ok(List.of());
    }
    // 查下级医院（简化：返回 flat list）
    return R.ok(tenantService.queryListByParent(currentTenant.getId()));
}
```

**Step 2.3** — 更新 Service 接口 `ISysTenantService` 新增 `queryListByParent()` 方法（或直接复用已有 `queryByParentId()`）。

### Phase 3：表格加号按钮

**Step 3.1** — `tenant/index.vue` 操作栏改动：

- 增大 `width="150"` 为 `width="200"`（或 `width="220"`）
- 在 edit 按钮前新增（或放在最前面）一个加号按钮：

```vue
<el-tooltip v-if="!scope.row.parentTenantId" content="添加子医院" placement="top">
  <el-button
    v-hasPermi="['system:tenant:add']"
    link
    type="primary"
    icon="Plus"
    @click="handleAddChild(scope.row)"
  ></el-button>
</el-tooltip>
```

**Step 3.2** — 新增 `handleAddChild(row)` 函数：

```ts
/** 添加子医院按钮操作 */
const handleAddChild = async (row: TenantVO) => {
  reset();
  await getTenantPackage();
  await getTenantTree();
  form.value.parentTenantId = row.id;
  dialog.visible = true;
  dialog.title = '添加医院';
};
```

**注意**：之前 require03 已经实现了 `tenantTree` 和 `getTenantTree()`，这里直接复用。`form.parentTenantId` 预填为当前行的 `id`。

---

## 四、实施顺序

```
Phase 1 (文字替换) → 可并行 → Phase 2 (tree接口) + Phase 3 (加号按钮)
```

- Phase 1 中的 6 个步骤按优先级从前到后：用户可见 label > 注释
- Phase 2 和 Phase 3 无依赖关系，可同时实施

---

## 五、注意事项

| 注意点 | 说明 |
|--------|------|
| 不改代码标识符 | `tenantId`、`parentTenantId`、`getTenantId()` 等字段/方法名不改 |
| `tenant_id` 是 varchar(20) 字符串，`id` 是 bigint | tree 查询需要先通过 `tenantId(String)` 查 `id(Long)`，再用 `id` 查下级 |
| **按钮权限保留** | 加号按钮使用 `v-hasPermi="['system:tenant:add']"`，因为添加子医院本质也是「新增」操作 |
| *tree 接口去掉了 superadmin 限制* | 普通医院管理员需要能看到下级医院列表来操作 |
| *前端 `listTenantTree` 返回值仍叫 `TenantVO[]`* | API 格式不变，只是返回数据从全量树变为当前医院下级列表，不需要改前端类型定义 |
| 原有 `queryTenantTree()` 方法可保留 | 超管后台可能仍需要使用。不改原方法，新增 `queryListByParent` |
