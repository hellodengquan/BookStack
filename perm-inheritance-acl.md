# 书架、章节、页面的权限继承与显式列表覆盖机制解析

## 一、概述

BookStack 采用 **"自底向上、就近优先、显式阻断"** 的权限继承模型。权限评估从目标实体自身开始，沿着父子层级链向上遍历，一旦遇到显式定义的权限就停止向上查找。同时存在两套权限数据：**EntityPermission（用户配置的原始权限）** 和 **JointPermission（预计算的缓存权限表）**。

---

## 二、核心数据模型

### 2.1 EntityPermission（原始权限表）

**表结构**（`app/Permissions/Models/EntityPermission.php:19`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `entity_type` | string | 实体类型：`page` / `chapter` / `book` / `bookshelf` |
| `entity_id` | int | 实体 ID |
| `role_id` | int | 角色 ID。**特殊值 `0` 表示 "其他所有人"（Fallback/默认权限）** |
| `view` | bool | 查看权限 |
| `create` | bool | 创建权限 |
| `update` | bool | 更新权限 |
| `delete` | bool | 删除权限 |

**关键特性**：
- `role_id = 0` 的记录称为 **"Fallback 权限"** 或 **"其他所有人"权限**
- `role_id != 0` 的记录称为 **"角色显式权限"**
- 同一个实体可以有多条记录（每个角色一条，外加可能的一条 fallback）

### 2.2 JointPermission（预计算缓存表）

**表结构**（`app/Permissions/Models/JointPermission.php:11`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `entity_type` | string | 实体类型 |
| `entity_id` | int | 实体 ID |
| `role_id` | int | 角色 ID |
| `status` | int | 权限状态（见 2.3） |
| `owner_id` | int | 非空时表示：仅当用户是该实体的所有者且状态不是显式拒绝时生效 |

**作用**：这是一张 **"预计算缓存表"**，将继承链计算后的权限结果提前物化，用于查询时的高效过滤。每次实体权限更新后都会重新生成。

### 2.3 PermissionStatus 权限状态常量

定义于 `app/Permissions/PermissionStatus.php:5`

| 常量 | 值 | 含义 |
|------|-----|------|
| `IMPLICIT_DENY` | 0 | 隐式拒绝（继承自父级 fallback 或角色默认无权限） |
| `IMPLICIT_ALLOW` | 1 | 隐式允许（继承自父级 fallback 或角色默认有权限） |
| `EXPLICIT_DENY` | 2 | 显式拒绝（实体自身为某角色明确设置了拒绝） |
| `EXPLICIT_ALLOW` | 3 | 显式允许（实体自身为某角色明确设置了允许） |

优先级：`EXPLICIT_DENY/ALLOW` > `IMPLICIT_DENY/ALLOW`

---

## 三、实体层级与继承链构建

### 3.1 实体层级关系

```
Bookshelf（书架）
    └── Book（书） ←── 注意：书架和书是多对多，不参与继承链！
            ├── Chapter（章节）
            │       └── Page（页面）
            └── Page（直接挂在书下的页面）
```

### 3.2 继承链的构建算法

**核心方法**：`EntityPermissionEvaluator::gatherEntityChainTypeIds()`
位置：`app/Permissions/EntityPermissionEvaluator.php:139`

```php
protected function gatherEntityChainTypeIds(SimpleEntityData $entity): array
{
    $chain = [$entity->type . ':' . $entity->id];  // 1. 自身始终在最前

    if ($entity->type === 'page' && $entity->chapter_id) {
        $chain[] = 'chapter:' . $entity->chapter_id;  // 2. 页面有章节则加章节
    }

    if ($entity->type === 'page' || $entity->type === 'chapter') {
        $chain[] = 'book:' . $entity->book_id;  // 3. 页面/章节都加所属书
    }

    return $chain;
}
```

**重要：Bookshelf（书架）不在继承链中！**

不同实体类型的继承链：

| 实体类型 | 继承链（优先级从高到低） | 示例 |
|----------|--------------------------|------|
| **Page（有章节）** | [自身 Page → 所属 Chapter → 所属 Book] | `[page:42, chapter:7, book:3]` |
| **Page（无章节）** | [自身 Page → 所属 Book] | `[page:43, book:3]` |
| **Chapter** | [自身 Chapter → 所属 Book] | `[chapter:7, book:3]` |
| **Book** | [自身 Book] | `[book:3]` |
| **Bookshelf** | [自身 Bookshelf] | `[bookshelf:1]` |

**链中顺序 = 优先级顺序**：越靠前优先级越高。

---

## 四、权限评估算法（核心）

### 4.1 入口方法

单实体权限检查：`EntityPermissionEvaluator::evaluateEntityForUser()`
位置：`app/Permissions/EntityPermissionEvaluator.php:17`

```
evaluateEntityForUser(entity, userRoleIds)
        │
        ├─► 步骤 1：系统管理员直接放行（返回 true）
        │
        ├─► 步骤 2：构建继承链 typeIdChain
        │
        ├─► 步骤 3：查询链上所有实体的相关权限（EntityPermission）
        │           （筛选条件：role_id ∈ 用户角色IDs ∪ {0}）
        │
        ├─► 步骤 4：遍历继承链，收集权限并分类（collapseAndCategorisePermissions）
        │
        └─► 步骤 5：根据分类结果评估最终状态（evaluatePermitsByType）
```

### 4.2 步骤 4：权限收集与分类 —— collapseAndCategorisePermissions

位置：`app/Permissions/EntityPermissionEvaluator.php:55`

这是**显式覆盖机制**的核心。

```php
protected function collapseAndCategorisePermissions(array $typeIdChain, array $permissionMapByTypeId): array
{
    $permitsByType = ['fallback' => [], 'role' => []];

    foreach ($typeIdChain as $typeId) {           // 按优先级从高到低遍历链
        $permissions = $permissionMapByTypeId[$typeId] ?? [];
        foreach ($permissions as $permission) {
            $roleId = $permission->role_id;
            $type = $roleId === 0 ? 'fallback' : 'role';
            if (!isset($permitsByType[$type][$roleId])) {
                // ★ 关键：同一 role 只保留第一次（最优先）出现的值
                $permitsByType[$type][$roleId] = $permission->{$this->action};
            }
        }

        if (isset($permitsByType['fallback'][0])) {
            // ★★ 关键阻断点：遇到 fallback（其他所有人）就停止向上查找！
            break;
        }
    }

    return $permitsByType;
}
```

**两条核心规则**：

1. **角色显式权限就近覆盖**：同一 `role_id`，只保留链中**最靠前**（最近）的设置，后面父级的被忽略。
2. **Fallback 阻断继承**：一旦在某层级遇到了 `role_id=0`（其他所有人）的设置，**立即停止遍历，不再向上查找父级**。

### 4.3 步骤 5：结果评估 —— evaluatePermitsByType

位置：`app/Permissions/EntityPermissionEvaluator.php:35`

```php
protected function evaluatePermitsByType(array $permitsByType): ?int
{
    // 规则 A：角色显式权限优先，且遵循"一票否决 + 至少一通过"
    if (count($permitsByType['role']) > 0) {
        // max() 妙用：只要有一个角色是 true（3/1），结果就是 EXPLICIT_ALLOW
        // 如果全部是 false（0），结果就是 EXPLICIT_DENY
        return max($permitsByType['role']) 
            ? PermissionStatus::EXPLICIT_ALLOW 
            : PermissionStatus::EXPLICIT_DENY;
    }

    // 规则 B：无角色显式权限时，看 fallback（其他所有人）
    if (count($permitsByType['fallback']) > 0) {
        return $permitsByType['fallback'][0] 
            ? PermissionStatus::IMPLICIT_ALLOW 
            : PermissionStatus::IMPLICIT_DENY;
    }

    // 规则 C：链上没有任何权限设置 → 返回 null，交给角色系统权限兜底
    return null;
}
```

**注意**：`$permitsByType['role']` 是一个 `[roleId => bool]` 数组，涵盖用户所有角色。
- 用户**只要有一个角色**在该层被显式设为允许，就 `EXPLICIT_ALLOW`
- 用户**所有角色**在该层都被显式设为拒绝，才 `EXPLICIT_DENY`

### 4.4 返回值处理

回到入口 `evaluateEntityForUser()`：

```php
$status = $this->evaluatePermitsByType($permitsByType);
// null → 链上无任何 EntityPermission 设置，交给上层（角色系统权限）判断
return is_null($status) 
    ? null 
    : $status === PermissionStatus::IMPLICIT_ALLOW || $status === PermissionStatus::EXPLICIT_ALLOW;
```

**完整决策树总结**：

```
用户访问某实体（Page/Chapter/Book）
        │
        ├─► 是系统管理员？ ──是──► 允许
        │
        ├─► 构建继承链（自身→父级→祖父级...）
        │
        ├─► 按优先级遍历链：
        │     │
        │     ├─► 遇到角色显式权限 ──► 收集（同角色只留最近的）
        │     │
        │     └─► 遇到 fallback（其他所有人） ──► 停止遍历
        │
        ├─► 有收集到角色显式权限？
        │     ├─► 是 ──► 任一角色允许？ ──是──► EXPLICIT_ALLOW ──► 允许
        │     │                       └─否──► EXPLICIT_DENY  ──► 拒绝
        │     │
        │     └─► 否 ──► 有 fallback？
        │                 ├─► 是 ──► fallback=true？ ──是──► IMPLICIT_ALLOW ──► 允许
        │                 │                     └─否──► IMPLICIT_DENY  ──► 拒绝
        │                 │
        │                 └─► 否 ──► 返回 null，由角色系统权限兜底
        │
        └─► 角色系统权限兜底判断（JointPermissionBuilder::createJointPermissionData）
              ├─► 有 {entityType}-view-all 权限？ ──是──► IMPLICIT_ALLOW
              ├─► 否则有 {entityType}-view-own 且是所有者？ ──是──► 允许（owner_id）
              └─► 否则 IMPLICIT_DENY
```

---

## 五、显式列表覆盖机制 —— 详细示例

### 示例场景

```
Book id=3（《技术文档》）
  └── Chapter id=7（《API 参考》）
        └── Page id=42（《认证接口》）
```

用户拥有角色：`role_id=5`（编辑）和 `role_id=8`（访客）

### 场景 1：页面自身设置了 fallback → 完全阻断

**EntityPermission 表数据**：

| entity | entity_id | role_id | view | 说明 |
|--------|-----------|---------|------|------|
| page | 42 | 0 | true | 页面 fallback：其他所有人可看 |
| book | 3 | 5 | false | 书级别：编辑角色禁止看 |

**评估过程**：

```
继承链：[page:42, chapter:7, book:3]
         │
         └─► page:42 有权限
              ├─► role 5：（无设置）
              ├─► role 8：（无设置）
              └─► fallback (role 0)：true ★ 遇到 fallback，停止遍历！

结果：无角色显式权限 → fallback=true → IMPLICIT_ALLOW → ✅ 允许
```

**结论**：即使父级 Book 对 role=5 设了拒绝，Page 自己的 fallback 阻断了向上查找，用户仍可访问。

---

### 场景 2：章节设置了角色显式权限 → 覆盖书的设置

**EntityPermission 表数据**：

| entity | entity_id | role_id | view |
|--------|-----------|---------|------|
| chapter | 7 | 5 | true | 章节允许编辑查看 |
| book | 3 | 5 | false | 书禁止编辑查看 |
| book | 3 | 0 | true | 书 fallback：允许 |

**评估过程**：

```
继承链：[page:42, chapter:7, book:3]
         │
         ├─► page:42：无任何权限
         │
         ├─► chapter:7：有设置
         │    └─► role 5 = true（收集起来）
         │    └─► 无 fallback → 继续向上
         │
         └─► book:3：有设置
              ├─► role 5 = false（★ 已收集过 role5= true，跳过此值！）
              └─► fallback = true（★ role5 已在 role 桶中，fallback 无效）

结果：roles 桶中有 [5=>true]
      max([true]) = true → EXPLICIT_ALLOW → ✅ 允许
```

**结论**：章节的 role=5 显式允许，**覆盖**了书级的 role=5 显式拒绝。

---

### 场景 3：书的 fallback 阻断，但页面角色显式拒绝优先

**EntityPermission 表数据**：

| entity | entity_id | role_id | view |
|--------|-----------|---------|------|
| page | 42 | 8 | false | 页面禁止访客角色 |
| chapter | 7 | 0 | true | 章节 fallback：所有人可看 |
| book | 3 | 5 | true | 书允许编辑角色 |

**评估过程**：

```
继承链：[page:42, chapter:7, book:3]
         │
         ├─► page:42：
         │    ├─► role 5：（无设置）
         │    ├─► role 8：= false（收集）
         │    └─► 无 fallback → 继续向上
         │
         └─► chapter:7：
              ├─► role 5/8：（无设置）
              └─► fallback = true ★ 遇到 fallback，停止！

结果：roles 桶中有 [8=>false]
      注意：role 5 没有任何 EntityPermission 设置！
      
      对于 role 5：不在 role 桶中 → fallback=true → IMPLICIT_ALLOW ✅
      对于 role 8：在 role 桶中 → max([false])=false → EXPLICIT_DENY ❌
      
      用户同时有 role 5 和 8，任一角色允许即可 → ✅ 允许
```

**结论**：多角色用户中，任一角色被显式允许即可通过。

---

### 场景 4：完全无 EntityPermission → 角色系统权限兜底

**EntityPermission 表数据**：链上所有实体均无 EntityPermission 记录。

**评估过程**：

```
继承链遍历完毕，permitsByType = ['fallback'=>[], 'role'=>[]]
→ evaluatePermitsByType() 返回 null
→ 交给 JointPermissionBuilder::createJointPermissionData() 兜底

兜底逻辑：
  ├─► 用户角色有 page-view-all 权限？ → IMPLICIT_ALLOW ✅
  ├─► 否则有 page-view-own 且 owned_by == 当前用户？ → owner_id 生效 ✅
  └─► 否则 → IMPLICIT_DENY ❌
```

---

## 六、JointPermission 预计算机制

### 6.1 为什么需要预计算？

查询时如果每个实体都实时计算继承链，性能会很差。因此系统将**继承链计算后的最终权限状态**物化到 `joint_permissions` 表中。

**核心类**：`JointPermissionBuilder`
位置：`app/Permissions/JointPermissionBuilder.php:21`

### 6.2 预计算流程（createJointPermissionData）

位置：`app/Permissions/JointPermissionBuilder.php:257`

```php
protected function createJointPermissionData(
    SimpleEntityData $entity, int $roleId, 
    MassEntityPermissionEvaluator $permissionMap, 
    array $rolePermissionMap, bool $isAdminRole
): array {
    // 1. 系统管理员直接全部允许
    if ($isAdminRole) {
        return $this->createJointPermissionDataArray($entity, $roleId, EXPLICIT_ALLOW, true);
    }

    // 2. 用上面第四章的算法评估 EntityPermission 链
    $entityPermissionStatus = $permissionMap->evaluateEntityForRole($entity, $roleId);
    if ($entityPermissionStatus !== null) {
        // 链上有设置 → 使用评估结果
        return $this->createJointPermissionDataArray($entity, $roleId, $entityPermissionStatus, false);
    }

    // 3. 链上无设置 → 角色系统权限兜底
    $permissionPrefix = $entity->type . '-view';
    $roleHasPermission = isset($rolePermissionMap["$roleId:$permissionPrefix-all"]);
    $roleHasPermissionOwn = isset($rolePermissionMap["$roleId:$permissionPrefix-own"]);
    $status = $roleHasPermission ? IMPLICIT_ALLOW : IMPLICIT_DENY;
    
    return $this->createJointPermissionDataArray($entity, $roleId, $status, $roleHasPermissionOwn);
}
```

### 6.3 Owner 权限（owner_id）

```php
protected function createJointPermissionDataArray(...): array
{
    // owner_id 生效条件：
    // 1. 角色有 xxx-view-own 权限（$hasPermissionOwn）
    // 2. 状态不是 EXPLICIT_DENY
    // 3. 实体确实有所有者（$entity->owned_by）
    $ownPermissionActive = ($hasPermissionOwn 
                          && $permissionStatus !== EXPLICIT_DENY 
                          && $entity->owned_by);

    return [
        'entity_id'   => $entity->id,
        'entity_type' => $entity->type,
        'role_id'     => $roleId,
        'status'      => $permissionStatus,
        'owner_id'    => $ownPermissionActive ? $entity->owned_by : null,
    ];
}
```

### 6.4 查询时的过滤

**核心方法**：`PermissionApplicator::restrictEntityQuery()`
位置：`app/Permissions/PermissionApplicator.php:99`

```sql
SELECT * FROM entities
WHERE EXISTS (
    SELECT 1 FROM joint_permissions jp
    WHERE jp.entity_id   = entities.id
      AND jp.entity_type = entities.type
      AND jp.role_id     IN (?, ?, ...)   -- 用户的所有角色ID
    GROUP BY jp.entity_type, jp.entity_id
    HAVING (
        -- 条件 A：状态是 IMPLICIT_ALLOW(1) 或 EXPLICIT_ALLOW(3)
        MAX(jp.status) IN (1, 3)
        OR
        -- 条件 B：owner_id 匹配且状态不是 EXPLICIT_DENY(2)
        (MAX(jp.owner_id) = ? AND MAX(jp.status) != 2)
        -- ↑ 当前用户ID
    )
)
```

巧妙之处：
- `MAX(status)`：只要任一角色是允许（1/3）就满足条件 A
- `MAX(owner_id)`：owner_id 非空表示该角色可按所有者放行
- 条件 A 和 B 是"或"的关系，任一成立即可

### 6.5 何时触发重新计算？

触发点：`Entity::rebuildPermissions()`
位置：`app/Entities/Models/Entity.php:394`

调用场景：
1. `PermissionsUpdater::updateFromPermissionsForm()` —— 权限表单提交后
2. `PermissionsUpdater::updateFromApiRequestData()` —— API 更新权限后
3. `PermissionsUpdater::updateBookPermissionsFromShelf()` —— 书架权限下发到书后
4. `RegeneratePermissionsCommand` —— 手动执行全量重建命令

重建范围（`JointPermissionBuilder::rebuildForEntity()`）：
- **Book**：该书 + 其下所有 Chapter + 所有 Page
- **Chapter**：该章节 + 其下所有 Page + 所属 Book
- **Page**：该页面 + 所属 Chapter（如有）+ 所属 Book
- **Bookshelf**：该书架自身

---

## 七、Bookshelf（书架）的特殊性

### 7.1 书架不参与继承链

注意 `gatherEntityChainTypeIds()` 方法中**没有 Bookshelf**！

- 书架下的 Book 的权限不会自动继承书架的 EntityPermission
- 书架的权限只控制**书架本身**是否可被查看/编辑

### 7.2 如何让书架的权限影响其下的书？

使用 `PermissionsUpdater::updateBookPermissionsFromShelf()`
位置：`app/Entities/Tools/PermissionsUpdater.php:145`

```php
public function updateBookPermissionsFromShelf(Bookshelf $shelf, $checkUserPermissions = true): int
{
    $shelfPermissions = $shelf->permissions()->get(...)->toArray();
    foreach ($shelf->books as $book) {
        if ($checkUserPermissions && !userCan('restrictions-manage', $book)) {
            continue;
        }
        $book->permissions()->delete();                   // 清空书的旧权限
        $book->permissions()->createMany($shelfPermissions); // 复制书架的权限
        $book->rebuildPermissions();                       // 重新生成 JointPermission
    }
}
```

这是**"复制/下发"**而非**"继承"**：
- 书架权限改变后，不会自动同步到已下发的书
- 必须主动触发该操作才能下发
- 下发后书拥有自己独立的 EntityPermission，和书架不再有关联

---

## 八、权限更新 API 流程

### 8.1 表单提交方式

`PermissionsUpdater::updateFromPermissionsForm()`
位置：`app/Entities/Tools/PermissionsUpdater.php:21`

```php
public function updateFromPermissionsForm(Entity $entity, Request $request): void
{
    $permissions = $request->input('permissions', null);
    $ownerId     = $request->input('owned_by', null);

    $entity->permissions()->delete();  // ★ 先全部清空

    if (!is_null($permissions)) {
        // 重新创建：每个角色一行，外加可能的一行 role_id=0（fallback）
        $entityPermissionData = $this->formatPermissionsFromRequestToEntityPermissions($permissions);
        $entity->permissions()->createMany($entityPermissionData);
    }
    // 更新 owner
    if (!is_null($ownerId)) { $this->updateOwnerFromId($entity, intval($ownerId)); }

    $entity->save();
    $entity->rebuildPermissions();  // ★ 重建 JointPermission
}
```

### 8.2 API 方式

`PermissionsUpdater::updateFromApiRequestData()`
位置：`app/Entities/Tools/PermissionsUpdater.php:46`

支持分别设置：
- `role_permissions`：角色显式权限列表（`role_id != 0`）
- `fallback_permissions.inheriting`：若为 `false`，则设置 fallback（`role_id = 0`），否则清除 fallback 表示继续继承
- `owner_id`：更新所有者

---

## 九、关键类与文件速查

| 职责 | 文件路径 | 核心类/方法 |
|------|----------|-------------|
| 单实体权限评估 | `app/Permissions/EntityPermissionEvaluator.php` | `evaluateEntityForUser()` `gatherEntityChainTypeIds()` `collapseAndCategorisePermissions()` |
| 批量权限评估 | `app/Permissions/MassEntityPermissionEvaluator.php` | 继承自上，带缓存 |
| JointPermission 生成 | `app/Permissions/JointPermissionBuilder.php` | `rebuildForAll()` `rebuildForEntity()` `createJointPermissionData()` |
| 查询时权限过滤 | `app/Permissions/PermissionApplicator.php` | `restrictEntityQuery()` `checkOwnableUserAccess()` |
| EntityPermission 模型 | `app/Permissions/Models/EntityPermission.php` | 原始权限存储 |
| JointPermission 模型 | `app/Permissions/Models/JointPermission.php` | 预计算缓存 |
| 权限状态常量 | `app/Permissions/PermissionStatus.php` | `EXPLICIT_ALLOW` 等常量 |
| 权限枚举 | `app/Permissions/Permission.php` | `view/create/update/delete` 等 |
| 权限更新操作 | `app/Entities/Tools/PermissionsUpdater.php` | `updateFromPermissionsForm()` |
| 实体基类 | `app/Entities/Models/Entity.php` | `rebuildPermissions()` `permissions()` |
| 简单实体数据 | `app/Permissions/SimpleEntityData.php` | 评估时的轻量对象 |

---

## 十一、权限缓存刷新机制

### 11.1 JointPermission 缓存模型

`joint_permissions` 是一张**写时更新（write-through）缓存表**，不存在 TTL 过期概念。它始终与 `entity_permissions` 表保持一致，但这种一致性不是自动的，而是由代码在每次权限变更后**显式触发重建**来保证的。

### 11.2 三种重建策略

| 策略 | 方法 | 触发场景 | 作用范围 |
|------|------|----------|----------|
| **全量重建** | `rebuildForAll()` | 手动命令 `bookstack:regenerate-permissions`；数据库迁移后 | TRUNCATE 整张表 → 重建所有实体×所有角色 |
| **按实体增量重建** | `rebuildForEntity($entity)` | 权限表单提交、API 更新、实体创建/移动/删除/排序、书架下发 | 只重建该实体及其关联实体的 JointPermission |
| **按角色增量重建** | `rebuildForRole($role)` | 角色创建、角色更新、角色权限变更 | 只重建该角色对所有实体的 JointPermission |

### 11.3 增量重建的细节

#### 按实体增量（`rebuildForEntity`）

位置：`app/Permissions/JointPermissionBuilder.php:54`

**重建范围取决于实体类型**：

| 触发实体 | 重建范围 | 原因 |
|----------|----------|------|
| Book | 该 Book + 其下所有 Chapter + 所有 Page | Book 是继承链根节点，影响所有子实体 |
| Chapter | 该 Chapter + 其下所有 Page + 所属 Book | Chapter 变更可能影响 Page 继承，同时需要 Book 的上下文 |
| Page | 该 Page + 所属 Chapter（如有）+ 所属 Book | 需要完整继承链上下文 |
| Bookshelf | 仅该书架自身 | Bookshelf 不参与子实体继承链 |

**实现逻辑**：

```
rebuildForEntity(entity)
        │
        ├─► Book: 单独走 bookFetchQuery() 路径
        │     1. 查询 Book + 预加载 chapters + pages
        │     2. 先 deleteOld=true 删除旧的 JointPermission
        │     3. 对 (Book, chapters, pages) × (所有角色) 重新计算并插入
        │
        ├─► Chapter: 
        │     1. entities = [Chapter, Book, Chapter.pages...]
        │     2. 删除这些实体的旧 JointPermission
        │     3. 对所有角色重新计算并插入
        │
        ├─► Page:
        │     1. entities = [Page, Book, Chapter?(如有)]
        │     2. 删除这些实体的旧 JointPermission
        │     3. 对所有角色重新计算并插入
        │
        └─► Bookshelf: 仅重建书架自身
```

#### 按角色增量（`rebuildForRole`）

位置：`app/Permissions/JointPermissionBuilder.php:84`

```
rebuildForRole(role)
        │
        ├─► 1. 删除该角色的所有 JointPermission
        │     role->jointPermissions()->delete()
        │
        ├─► 2. 分块遍历所有 Book（每块 10 本）
        │     对每本书及其子实体 × 该角色 重新计算
        │
        └─► 3. 分块遍历所有 Bookshelf（每块 50 个）
              对每个书架 × 该角色 重新计算
```

### 11.4 全量重建流程

位置：`app/Permissions/JointPermissionBuilder.php:32`

```
rebuildForAll()
        │
        ├─► 1. TRUNCATE joint_permissions 表（完全清空）
        │
        ├─► 2. 加载所有角色（含关联的 role_permissions）
        │
        ├─► 3. 分块遍历所有 Book（每块 5 本）
        │     buildJointPermissionsForBooks(books, roles)
        │     → 对 (Book + chapters + pages) × (所有角色) 计算并批量插入
        │
        └─► 4. 分块遍历所有 Bookshelf（每块 50 个）
              createManyJointPermissions(shelves, roles)
              → 对 Bookshelf × (所有角色) 计算并批量插入
```

**注意**：全量重建使用 `TRUNCATE`，这是 DDL 操作，在 MySQL 中会隐式提交，**无法回滚**。

### 11.5 所有触发 rebuildPermissions 的场景

通过代码搜索，以下操作会触发 `Entity::rebuildPermissions()`：

| 触发位置 | 操作 | 重建范围 |
|----------|------|----------|
| `PermissionsUpdater::updateFromPermissionsForm()` | 表单提交权限更新 | 当前实体 + 关联实体 |
| `PermissionsUpdater::updateFromApiRequestData()` | API 更新权限 | 当前实体 + 关联实体 |
| `PermissionsUpdater::updateBookPermissionsFromShelf()` | 书架权限下发 | 每个 Book 独立触发 |
| `BaseRepo::create()` | 创建新实体 | 新实体 + 关联实体 |
| `PageRepo::updatePage()` | 更新页面 | 页面 + 关联实体 |
| `PageRepo::updateDraft()` | 更新草稿 | 草稿 + 关联实体 |
| `PageRepo::changePageParent()` | 移动页面父级 | 页面 + 关联实体 |
| `ChapterRepo::updateChapter()` | 更新章节 | 章节 + 关联实体 |
| `ChapterRepo::changeChapterParent()` | 移动章节父级 | 章节 + 关联实体 |
| `BookSorter::runBookSort()` | 书籍排序 | 涉及的所有 Book |
| `Cloner::copyEntityPermissions()` | 实体克隆后复制权限 | 目标实体 + 关联实体 |
| `TrashCan::destroyEntity()` | 销毁实体 | 通过 `permissions()->delete()` + `jointPermissions()->delete()` 清理 |
| `RegeneratePermissionsCommand` | 命令行手动全量重建 | 所有实体 |

---

## 十二、多用户并发更新的冲突处理

### 12.1 事务隔离级别

BookStack 自定义了 `DatabaseTransaction` 工具类来统一管理权限相关操作的事务。

位置：`app/Util/DatabaseTransaction.php:24`

```php
class DatabaseTransaction
{
    public function run(): mixed
    {
        // ★ 设置事务隔离级别为 READ COMMITTED
        DB::statement('SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED');
        return DB::transaction($this->callback);
    }
}
```

**为什么选择 READ COMMITTED？**

代码注释明确说明了原因：

> "READ COMMITTED ensures that changes from other transactions can be read within a transaction, even if started afterward (and for example, it was blocked by the initial transaction). This is quite important for things like permission generation, where we would want to consider the changes made by other committed transactions by the time we come to regenerate permission access."

默认 MySQL InnoDB 使用 `REPEATABLE READ`，这会导致一个问题：

```
时间线：
  T1: BEGIN → 修改 Book 权限 → (此时 T2 已提交了 Chapter 权限修改) → 重建 JointPermission
  T2: BEGIN → 修改 Chapter 权限 → 重建 JointPermission → COMMIT

在 REPEATABLE READ 下，T1 的重建读取 Chapter 权限时会看到 T2 开始前的旧数据，
导致 T1 重建的 JointPermission 丢失 T2 的修改。
```

`READ COMMITTED` 确保了在 T1 执行重建时，能看到 T2 已经提交的最新数据。

### 12.2 无显式行锁机制

BookStack **不使用** `SELECT ... FOR UPDATE` 或其他悲观锁来防止并发修改同一实体的权限。

**冲突处理策略**：**"最后写入者胜出"（Last Writer Wins）**

```
用户 A: 读取 Book id=3 的权限 → 设置 role=5 view=true → delete all → create → rebuild
用户 B: 读取 Book id=3 的权限 → 设置 role=5 view=false → delete all → create → rebuild

结果：谁后完成，谁的权限生效。先完成者的修改被覆盖。
```

**这是有意的设计选择**：
- 权限管理通常是低频操作，并发冲突极少
- `delete() → createMany() → rebuildPermissions()` 在事务中保证原子性
- 即使被覆盖，也只是权限回退到上一个操作者的设置，不会导致数据不一致

### 12.3 事务包裹的完整链路

所有权限更新操作都在 `DatabaseTransaction` 中执行：

| Controller 方法 | 事务范围 |
|-----------------|----------|
| `PermissionsController::updateForPage()` | `DatabaseTransaction(updateFromPermissionsForm)` |
| `PermissionsController::updateForChapter()` | `DatabaseTransaction(updateFromPermissionsForm)` |
| `PermissionsController::updateForBook()` | `DatabaseTransaction(updateFromPermissionsForm)` |
| `PermissionsController::updateForShelf()` | `DatabaseTransaction(updateFromPermissionsForm)` |
| `PermissionsController::copyShelfPermissionsToBooks()` | `DatabaseTransaction(updateBookPermissionsFromShelf)` |
| `ContentPermissionApiController::update()` | ⚠️ **未包裹事务**（见 12.4 分析） |
| `PermissionsRepo::saveNewRole()` | `DatabaseTransaction(role save + permissions sync + rebuildForRole)` |
| `PermissionsRepo::updateRole()` | `DatabaseTransaction(permissions sync + role save + rebuildForRole)` |
| `PermissionsRepo::deleteRole()` | `DatabaseTransaction(user migration + entityPermission delete + jointPermission delete + role delete)` |

### 12.4 API 路径的事务缺口

`ContentPermissionApiController::update()` **没有**包裹 `DatabaseTransaction`：

```php
// app/Permissions/ContentPermissionApiController.php:69
public function update(Request $request, string $contentType, string $contentId)
{
    $entity = $this->entities->get($contentType)->newQuery()->scopes(['visible'])->findOrFail($contentId);
    $this->checkOwnablePermission(Permission::RestrictionsManage, $entity);
    $data = $this->validate($request, $this->rules()['update']);
    $this->permissionsUpdater->updateFromApiRequestData($entity, $data);  // 无事务包裹
    return response()->json($this->formattedPermissionDataForEntity($entity));
}
```

而 Web 表单路径的 `PermissionsController` 则正确使用了事务。

**潜在风险**：`updateFromApiRequestData()` 内部分为：delete role_permissions → createMany → delete fallback → createMany → save → rebuildPermissions。如果中间步骤失败，可能导致部分权限被删除但新权限未创建的中间状态。不过由于 `updateFromApiRequestData()` 自身内部操作不多且不会抛出业务异常，实际风险较低。

### 12.5 delete + create 的原子性分析

`PermissionsUpdater::updateFromPermissionsForm()` 中的核心操作：

```php
$entity->permissions()->delete();   // 步骤 1：删除所有旧的 EntityPermission
$entity->permissions()->createMany($entityPermissionData);  // 步骤 2：批量创建新的
$entity->save();                    // 步骤 3：保存实体
$entity->rebuildPermissions();      // 步骤 4：重建 JointPermission
```

这四步在 `DatabaseTransaction` 中的保证：
- 如果步骤 2 失败（如数据校验错误），步骤 1 的删除会被回滚
- 如果步骤 4 失败，步骤 1-3 会被回滚
- **关键**：`rebuildPermissions()` 内部的 `deleteManyJointPermissionsForEntities()` + `createManyJointPermissions()` 也在同一事务中

### 12.6 全量重建的不可回滚性

`rebuildForAll()` 开头执行 `JointPermission::query()->truncate()`，这是 DDL 语句，在 MySQL 中会触发隐式提交。这意味着：

- TRUNCATE 之前的任何未提交事务会被强制提交
- TRUNCATE 本身**无法回滚**
- 如果 TRUNCATE 后的重建过程失败，`joint_permissions` 表将处于**部分空**的状态
- 此时需要再次运行 `bookstack:regenerate-permissions` 命令来修复

---

## 十三、列表查询中权限校验的 N+1 问题规避

### 13.1 核心设计：预计算表替代实时评估

如果没有 `joint_permissions` 预计算表，查询用户可见的实体列表需要：

```sql
-- ❌ N+1 方式：对每个实体实时评估继承链
SELECT * FROM entities WHERE type = 'page';
-- 然后对每一行：
--   → 查询 page 的 EntityPermission
--   → 查询 chapter 的 EntityPermission
--   → 查询 book 的 EntityPermission
--   → 运行 collapseAndCategorisePermissions 逻辑
```

对于 N 个实体，这将产生 3N+1 次查询。

**JointPermission 的解决方案**：将权限评估结果物化，查询时只需一次子查询。

### 13.2 restrictEntityQuery 的 SQL 生成

位置：`app/Permissions/PermissionApplicator.php:99`

```php
public function restrictEntityQuery(Builder $query): Builder
{
    return $query->where(function (Builder $parentQuery) {
        $parentQuery->whereHas('jointPermissions', function (Builder $permissionQuery) {
            $permissionQuery->select(['entity_id', 'entity_type'])
                ->selectRaw('max(owner_id) as owner_id')
                ->selectRaw('max(status) as status')
                ->whereIn('role_id', $this->getCurrentUserRoleIds())
                ->groupBy(['entity_type', 'entity_id'])
                ->havingRaw('(status IN (1, 3) or (owner_id = ? and status != 2))', [$this->currentUser()->id]);
        });
    });
}
```

**生成的实际 SQL**（简化版）：

```sql
SELECT * FROM entities
WHERE EXISTS (
    SELECT entity_id, entity_type,
           MAX(owner_id) AS owner_id,
           MAX(status)   AS status
    FROM joint_permissions
    WHERE entity_id   = entities.id
      AND entity_type = entities.type
      AND role_id     IN (2, 5)        -- 当前用户的角色列表
    GROUP BY entity_type, entity_id
    HAVING (status IN (1, 3) OR (owner_id = 42 AND status != 2))
)
```

**这是一条 SQL，不存在 N+1 问题。**

### 13.3 whereHas 的实现方式

Laravel 的 `whereHas` 使用 `WHERE EXISTS` 子查询：

```
主查询 → WHERE EXISTS (子查询 joint_permissions)
```

- 子查询中的 `role_id IN (...)` 通过索引快速定位
- `GROUP BY entity_type, entity_id` 将多角色记录聚合为一条
- `HAVING` 子句完成最终的权限判断逻辑
- 整个过滤在数据库层完成，PHP 层不参与权限计算

### 13.4 joint_permissions 表的索引策略

根据迁移文件，`joint_permissions` 表有以下索引：

| 索引 | 字段 | 用途 |
|------|------|------|
| `idx (entity_id, entity_type)` | 联合索引 | WHERE 条件中按实体定位 |
| `idx (status)` | 单列索引 | HAVING 中按状态过滤 |
| `idx (owner_id)` | 单列索引 | HAVING 中按所有者过滤 |
| `idx (role_id)` | 单列索引 | WHERE 条件中按角色过滤 |

**查询命中路径**：
1. 通过 `(entity_id, entity_type)` 关联子查询 → 缩小到具体实体的记录
2. 通过 `role_id` 进一步过滤 → 只剩当前用户角色的记录
3. GROUP BY + HAVING 完成聚合判断

### 13.5 其他查询方法的 N+1 规避

除了 `restrictEntityQuery()`，还有三个场景的权限过滤方法：

| 方法 | 场景 | N+1 规避方式 |
|------|------|-------------|
| `restrictEntityRelationQuery()` | 过滤多态关联（如标签、评论） | `restrictEntityQuery()` + `WHERE EXISTS` 子查询排除草稿页 |
| `restrictPageRelationQuery()` | 过滤页面的一对多关联 | `restrictEntityQuery()` + `WHERE EXISTS` 检查页面草稿状态 |
| `restrictDraftsOnPageQuery()` | 页面列表排除他人草稿 | 简单 WHERE 条件，不涉及子查询 |

这些方法都**不加载实体模型到内存**进行权限判断，而是在 SQL 层完成过滤。

### 13.6 单实体权限检查路径（不使用预计算表）

位置：`app/Permissions/PermissionApplicator.php:64`

```php
protected function hasEntityPermission(Entity $entity, array $userRoleIds, string $action): ?bool
{
    return (new EntityPermissionEvaluator($action))->evaluateEntityForUser($entity, $userRoleIds);
}
```

**单实体检查不走 `joint_permissions`**，而是实时评估 `entity_permissions` 继承链。

**原因**：
- 单实体检查频率低（如编辑/删除操作前的校验）
- 需要检查 `view/create/update/delete` 四种操作，而 `joint_permissions` 只缓存了 `view`
- 实时评估一次的成本可接受（1~3 次 SQL 查询）

**查询效率**：
- `EntityPermissionEvaluator::getPermissionsMapByTypeId()` 一次查询获取继承链上所有相关权限
- 查询条件：`entity_type IN (...) AND entity_id IN (...) AND role_id IN (...)`
- 由于继承链最多 3 个节点，查询量很小

### 13.7 MassEntityPermissionEvaluator 的批量优化

位置：`app/Permissions/MassEntityPermissionEvaluator.php:8`

在 `JointPermissionBuilder` 的全量/增量重建中使用，避免对每个实体重复查询：

```
1. 预先收集所有涉及的 SimpleEntityData
2. 一次性查询所有相关的 EntityPermission
3. 在内存中按 (typeId, roleId) 分组缓存
4. 每次评估直接从内存读取，不再查数据库
```

这是重建过程中避免 N+1 的关键优化。

---

## 十四、批量授权与撤销的事务边界及回滚链路

### 14.1 权限更新的事务边界

BookStack 的权限更新遵循 **"先删后建"** 策略，事务边界由 `DatabaseTransaction` 统一管控。

#### 表单路径的事务链路

```
PermissionsController::updateForBook()
        │
        └─► DatabaseTransaction::run()
                │
                ├─► SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED
                │
                └─► DB::transaction(callback)
                        │
                        ├─► 1. PermissionsUpdater::updateFromPermissionsForm()
                        │       ├─► entity->permissions()->delete()       -- 删除旧权限
                        │       ├─► entity->permissions()->createMany()    -- 创建新权限
                        │       ├─► entity->save()                         -- 保存实体
                        │       └─► entity->rebuildPermissions()           -- 重建 JointPermission
                        │               ├─► deleteManyJointPermissionsForEntities()
                        │               └─► createManyJointPermissions()
                        │                       ├─► MassEntityPermissionEvaluator (批量评估)
                        │                       └─► DB::table('joint_permissions')->insert() (分块1000)
                        │
                        └─► 2. Activity::add(PERMISSIONS_UPDATE)          -- 记录活动日志
```

**事务保证**：以上所有步骤要么全部成功，要么全部回滚。

#### API 路径的事务链路

```
ContentPermissionApiController::update()
        │
        └─► PermissionsUpdater::updateFromApiRequestData()  -- ⚠️ 无外层事务
                │
                ├─► 1. entity->permissions()->where('role_id', '!=', 0)->delete()
                ├─► 2. entity->permissions()->createMany(rolePermissionData)
                ├─► 3. entity->permissions()->where('role_id', '=', 0)->delete()
                ├─► 4. (条件) entity->permissions()->createMany(fallbackData)
                ├─► 5. entity->save()
                └─► 6. entity->rebuildPermissions()
```

**API 路径的分步删除策略**：

与表单路径的 `delete all → create all` 不同，API 路径支持**部分更新**：
- 只传 `role_permissions` → 只替换角色权限，保留 fallback
- 只传 `fallback_permissions` → 只替换 fallback，保留角色权限
- 不传的字段不会被修改

### 14.2 书架权限下发的事务边界

`PermissionsController::copyShelfPermissionsToBooks()` 包裹在 `DatabaseTransaction` 中：

```php
$updateCount = (new DatabaseTransaction(function () use ($shelf) {
    return $this->permissionsUpdater->updateBookPermissionsFromShelf($shelf);
}))->run();
```

但 `updateBookPermissionsFromShelf()` 内部对**每个 Book** 独立执行 `delete → createMany → rebuildPermissions`：

```
updateBookPermissionsFromShelf(shelf)
        │
        ├─► 获取 shelf 的权限快照
        ├─► 获取 shelf 下的所有 books
        │
        └─► foreach book:
                ├─► 权限检查（userCan）
                ├─► book->permissions()->delete()
                ├─► book->permissions()->createMany(shelfPermissions)
                └─► book->rebuildPermissions()
                    → 重建该书 + 其下所有 Chapter + Page 的 JointPermission
```

**事务边界**：整个循环在同一个事务中，如果第 3 本书处理失败，前 2 本书的权限修改也会回滚。

### 14.3 角色权限变更的事务边界

`PermissionsRepo::updateRole()` 的事务链路：

```
PermissionsRepo::updateRole()
        │
        └─► DatabaseTransaction::run()
                │
                ├─► 1. assignRolePermissions(role, permissionNames)
                │       ├─► 查询 RolePermission IDs
                │       └─► role->permissions()->sync($permissionIds)  -- 替换角色权限
                │
                ├─► 2. role->fill(data)->save()                        -- 保存角色信息
                │
                └─► 3. permissionBuilder->rebuildForRole(role)         -- 重建该角色的 JointPermission
                        │
                        ├─► role->jointPermissions()->delete()          -- 删除旧记录
                        │
                        ├─► 分块遍历所有 Book (每块10本)
                        │     → (Book + chapters + pages) × role 计算并插入
                        │
                        └─► 分块遍历所有 Bookshelf (每块50个)
                              → Bookshelf × role 计算并插入
```

**关键风险**：`rebuildForRole()` 的 `delete → insert` 循环使用分块插入，每块 1000 条。如果中途失败：
- 由于在 `DatabaseTransaction` 中，所有插入会被回滚
- 但 `role->jointPermissions()->delete()` 的删除也会回滚
- 最终状态：回到事务开始前的数据

### 14.4 角色删除的事务边界

```
PermissionsRepo::deleteRole()
        │
        └─► DatabaseTransaction::run()
                │
                ├─► 1. (条件) 用户迁移到新角色
                │       newRole->users()->sync(userIds)
                │
                ├─► 2. role->entityPermissions()->delete()    -- 删除该角色的所有 EntityPermission
                │
                ├─► 3. role->jointPermissions()->delete()     -- 删除该角色的所有 JointPermission
                │
                ├─► 4. Activity::add(ROLE_DELETE)             -- 记录活动日志
                │
                └─► 5. role->delete()                          -- 删除角色
```

**注意**：角色删除后，**不会触发** `rebuildForAll()` 或其他重建操作。这意味着：
- 该角色的 `EntityPermission` 被直接删除
- 该角色的 `JointPermission` 被直接删除
- 其他角色的 `JointPermission` 不受影响
- 无需重建：因为其他角色的权限评估不依赖被删除角色的数据

### 14.5 回滚链路总结

| 操作失败点 | 回滚行为 | 数据一致性影响 |
|-----------|---------|---------------|
| `permissions()->delete()` 失败 | 事务回滚，无影响 | 无 |
| `permissions()->createMany()` 失败 | 事务回滚，delete 被撤销 | 无 |
| `entity->save()` 失败 | 事务回滚，delete + createMany 被撤销 | 无 |
| `rebuildPermissions()` 中 delete 失败 | 事务回滚，权限表 + 实体修改被撤销 | 无 |
| `rebuildPermissions()` 中 insert 失败 | 事务回滚，delete + 之前的 insert 被撤销 | 无 |
| `rebuildForAll()` 中 TRUNCATE 后失败 | ⚠️ **不可回滚**，TRUNCATE 是 DDL | joint_permissions 表部分空，需手动重建 |
| API 路径无事务包裹时失败 | ⚠️ **部分提交**，delete 已执行 | 可能丢失部分权限配置 |

### 14.6 实体销毁时的权限清理

`TrashCan::destroyEntity()` 在 `DatabaseTransaction` 中执行：

```php
// app/Entities/Tools/TrashCan.php:391
protected function destroyCommonRelations(Entity $entity): void
{
    $entity->permissions()->delete();      // 删除 EntityPermission
    $entity->jointPermissions()->delete(); // 删除 JointPermission
    // ... 其他关联数据清理
}
```

实体销毁时**不触发 `rebuildPermissions()`**，而是直接删除该实体的权限记录。这是正确的，因为：
- 实体已被删除，无需再维护其 JointPermission
- 不影响其他实体的权限（其他实体的继承链中不包含已删除实体）

---

## 十五、补充设计要点

9. **预计算缓存的写时更新**：JointPermission 不设 TTL，由业务代码在权限变更后显式触发重建
10. **READ COMMITTED 隔离级别**：避免权限重建时读取到过时的快照数据，保证并发场景下的数据新鲜度
11. **Last Writer Wins 并发策略**：无悲观锁，依赖事务的原子性保证单次操作的完整性
12. **全量重建的不可回滚风险**：TRUNCATE 是 DDL 操作，一旦执行无法回滚，需谨慎使用
13. **API 路径的事务缺口**：`ContentPermissionApiController::update()` 未包裹事务，存在部分提交风险
14. **单实体 vs 列表查询的分流**：列表查询走预计算表（一次 SQL），单实体检查走实时评估（灵活但多查询）
15. **MassEntityPermissionEvaluator 的批量优化**：重建时预加载所有权限到内存，避免逐实体查询的 N+1 问题

---

## 十六、跨租户权限隔离与共享边界

### 16.1 BookStack 的"多租户"定位

BookStack **不是传统意义上的多租户 SaaS 系统**，没有独立的租户隔离沙箱。它采用的是**单实例、多用户、基于角色的权限隔离**模型。所有实体共享同一张表，通过 `joint_permissions` 表进行行级权限过滤。

**没有的概念**：
- ❌ 没有独立的租户 Schema/数据库
- ❌ 没有 `tenant_id` 字段
- ❌ 没有租户级管理员（只有全局管理员）
- ❌ 没有数据物理隔离

**有的隔离方式**：
- ✅ 基于角色的访问控制（RBAC）
- ✅ 实体级 EntityPermission 精细化权限
- ✅ Public 角色实现公开/私密内容区分
- ✅ Owner 权限实现"我的文档"隔离

### 16.2 Public 角色与公开访问边界

**Public 角色**是 BookStack 实现"访客共享"的核心机制。

#### Public 系统角色

位置：`app/Users/Models/Role.php:114`

```php
public static function getSystemRole(string $systemName): ?self
{
    static $cache = [];
    if (!isset($cache[$systemName])) {
        $cache[$systemName] = static::query()->where('system_name', '=', $systemName)->first();
    }
    return $cache[$systemName];
}
```

系统内置两个特殊角色：
- `system_name = 'public'` —— 公开访客角色
- `system_name = 'admin'` —— 管理员角色

#### Public 用户

位置：`app/Users/Models/User.php:103`

```php
public function isGuest(): bool
{
    return $this->system_name === 'public';
}
```

- Public 用户（访客）自动拥有 Public 角色
- 当 `setting('app-public')` 为 true 时，未登录用户以 Public 用户身份访问
- Public 用户**不能分配其他角色**（迁移 `2023_06_10_071823_remove_guest_user_secondary_roles.php` 移除了该能力）

#### 公开访问的全局开关

```php
// app/Users/Models/User.php:111
public function hasAppAccess(): bool
{
    return !$this->isGuest() || setting('app-public');
}
```

- `app-public = true`：未登录用户可以访问系统，以 Public 角色参与权限评估
- `app-public = false`：未登录用户直接被拒绝，连登录页外的内容都看不到

#### Public 角色在权限继承中的行为

Public 角色和其他角色**完全一样**参与 EntityPermission 继承链评估，没有任何特殊待遇：

```
未登录用户访问页面：
  用户角色 = [public_role_id]
  评估过程与普通用户完全一致
  → 遍历继承链
  → collapseAndCategorisePermissions
  → evaluatePermitsByType
```

**实现"公开文档"的标准方式**：
在 Book 级别设置 `role_id = 0`（fallback / 其他所有人）的 view = true，那么所有用户（包括 Public）都能看到这本书及其子内容。

### 16.3 共享的层级边界

实体权限的共享是**向下继承**的，遵循以下边界：

| 操作 | 共享范围 | 影响深度 |
|------|----------|----------|
| 设置 Bookshelf 权限 | 仅书架本身 | 不影响书架下的 Book |
| 书架权限下发到 Book | 所有选中的 Book | 触发每个 Book 独立重建，影响其下所有章节和页面 |
| 设置 Book 权限 | 该书 + 章节 + 页面 | 完整继承链，影响最大 |
| 设置 Chapter 权限 | 该章节 + 其下页面 | 部分影响，Book 级别不受影响 |
| 设置 Page 权限 | 仅该页面 | 影响最小 |
| 设置 Fallback（其他所有人） | 当前层级及以下 | 阻断向上继承，所有未单独设置角色的用户都受影响 |

### 16.4 Owner 权限的"准租户"隔离

`xxx-view-own` 权限 + `owned_by` 字段提供了一种**"我的文档"**级别的软隔离：

```sql
-- JointPermission 中 owner_id 非空表示：
-- 当用户是所有者且状态不是 EXPLICIT_DENY 时生效
HAVING (status IN (1, 3) OR (owner_id = ? AND status != 2))
```

典型应用场景：
- 普通用户有 `page-view-own` 但没有 `page-view-all`
- 只能看到自己创建的页面和**显式开放给所有人**的页面
- 形成一种"个人工作区 + 共享内容区"的软隔离效果

### 16.5 跨"租户"共享的实践模式

虽然没有原生多租户，但 BookStack 社区常用以下模式实现类似效果：

| 模式 | 实现方式 | 隔离程度 |
|------|----------|----------|
| **按书架分区** | 每个团队一个书架，通过书架权限控制可见性 | 弱（书架不参与继承，需手动下发到 Book） |
| **按书分区** | 每个团队一本书/一组书，设置 Book 级权限 | 中（继承链完整，能有效隔离） |
| **角色+Owner** | 创建团队角色，配合 owned_by 实现"我的团队内容" | 中（依赖内容创建时的归属设置） |
| **Public 角色** | 公开内容通过 fallback 开放给 Public 角色 | 弱（只有公开/私密二元选择） |

### 16.6 管理员权限的穿透性

系统管理员（Admin 角色）**完全绕过**所有实体权限检查：

```php
// app/Permissions/EntityPermissionEvaluator.php:24
if (in_array(0, $userRoleIds)) {
    return true;  // 系统管理员直接放行
}
```

```php
// app/Permissions/JointPermissionBuilder.php:265
if ($isAdminRole) {
    // 管理员所有实体所有操作都是 EXPLICIT_ALLOW
    return $this->createJointPermissionDataArray($entity, $roleId, PermissionStatus::EXPLICIT_ALLOW, true);
}
```

**管理员的权限是全局的**，不存在"只能管理某些书"的部分管理员概念。这也是 BookStack 不是多租户系统的重要标志。

---

## 十七、权限模板与角色继承路径

### 17.1 两套权限体系

BookStack 的权限系统由**两个正交的层级**组成：

```
┌─────────────────────────────────────────────────────┐
│  层级一：系统级权限（RolePermission）                 │
│  - 定义角色"能做什么类型的操作"                       │
│  - 如：page-view-all, book-create-own               │
│  - 存储在 role_permissions + permission_role 表     │
└────────────────────┬────────────────────────────────┘
                     │ 组合
                     ▼
┌─────────────────────────────────────────────────────┐
│  层级二：实体级权限（EntityPermission）              │
│  - 定义"对具体实体的访问限制"                        │
│  - 如：Book id=3 对 role_id=5 禁止 view              │
│  - 存储在 entity_permissions 表                      │
│  - 预计算结果缓存在 joint_permissions 表             │
└─────────────────────────────────────────────────────┘
```

两套权限共同决定最终访问结果：

```
用户能访问实体 X 吗？
    ├─► 有系统级权限 xxx-view-all 吗？ ──是──► 允许
    │
    ├─► 是实体所有者且有 xxx-view-own 吗？ ──是──► 允许
    │
    └─► EntityPermission 继承链评估
          ├─► 有角色显式允许？ ──是──► 允许
          ├─► 有角色显式拒绝？ ──是──► 拒绝
          ├─► 有 fallback 允许？ ──是──► 允许
          └─► 其他情况 → 拒绝
```

### 17.2 RolePermission（系统权限）详解

定义位置：`app/Permissions/Permission.php`（枚举）
存储位置：`role_permissions` 表 + `permission_role` 关联表

#### 权限命名规则

```
{实体类型}-{动作}-{范围}
```

| 部分 | 可选值 | 说明 |
|------|--------|------|
| 实体类型 | `page`, `chapter`, `book`, `bookshelf`, `image`, `attachment`, `comment` | 操作对象类型 |
| 动作 | `view`, `create`, `update`, `delete` | 操作类型 |
| 范围 | `all`, `own` | **all** = 所有实体；**own** = 仅自己所有的实体 |

**特殊权限**（没有 all/own 后缀）：
- `access-api` —— 访问 API
- `content-export` —— 导出内容
- `content-import` —— 导入内容
- `editor-change` —— 切换编辑器
- `receive-notifications` —— 接收通知
- `restrictions-manage` —— 管理权限（实体级）
- `restrictions-manage-all` / `restrictions-manage-own`
- `settings-manage` —— 管理系统设置
- `templates-manage` —— 管理模板
- `user-roles-manage` —— 管理角色
- `users-manage` —— 管理用户

#### 权限的层级语义

以 `page-view` 为例：

| 权限 | 含义 |
|------|------|
| `page-view-all` | 可以查看所有页面（不受 EntityPermission 限制？不，还是受限制的） |
| `page-view-own` | 可以查看**自己拥有的**页面（配合 owned_by 字段） |

**注意**：`xxx-view-all` 只是系统权限层面的"全局查看许可"，仍然会被 EntityPermission 的显式拒绝覆盖。完整逻辑见 `checkOwnableUserAccess()`。

### 17.3 角色之间的关系：无继承，只有叠加

BookStack 的角色**没有父子继承关系**。用户可以拥有多个角色，权限是**"或"叠加**的。

```
用户拥有角色 A + 角色 B：
  系统权限 = (角色 A 的权限) ∪ (角色 B 的权限)
  实体权限 = 任一角色允许即允许
```

```php
// app/Permissions/EntityPermissionEvaluator.php:38
// max() 实现多角色或运算
return max($permitsByType['role']) 
    ? PermissionStatus::EXPLICIT_ALLOW 
    : PermissionStatus::EXPLICIT_DENY;
```

```php
// app/Users/Models/User.php:165
// 系统权限也是合并去重
$this->permissions = $this->newQuery()->getConnection()->table('role_user', 'ru')
    ->select('role_permissions.name as name')->distinct()
    // ...
    ->where('ru.user_id', '=', $this->id)
    ->pluck('name');
```

### 17.4 系统角色的"模板"作用

虽然没有"权限模板"的概念，但 Role 本身起到了**权限集合模板**的作用：

| "模板"类型 | 实现方式 | 特点 |
|-----------|----------|------|
| **系统内置角色** | `system_name` 字段标记的 admin / public | 不可删除，有特殊逻辑 |
| **默认注册角色** | `registration-role` 设置 | 新用户注册时自动分配 |
| **普通角色** | 后台创建的角色 | 可自由编辑权限，分配给用户 |
| **外部认证映射** | `external_auth_id` 字段 | LDAP/SAML 登录时自动匹配角色 |

#### 角色创建的"复制"功能

位置：`tests/User/RoleManagementTest.php:224`（测试验证），前端 UI 提供"复制角色"按钮

创建角色时可以通过 `?copy_from=<role_id>` 参数预填充另一个角色的权限，这是一种**手动模板复制**机制，而非动态继承。

### 17.5 checkOwnableUserAccess 完整决策流

位置：`app/Permissions/PermissionApplicator.php:27`

这是**单实体权限检查**的总入口，综合了系统权限 + 实体权限：

```php
public function checkOwnableUserAccess(Model&OwnableInterface $ownable, string|Permission $permission): bool
{
    // 1. 计算权限全称（如 page-view-all）
    $allRolePermission = $user->can($fullPermission . '-all');
    $ownRolePermission = $user->can($fullPermission . '-own');
    $isOwner = $user->id === $ownableFieldVal;
    $hasRolePermission = $allRolePermission || ($isOwner && $ownRolePermission);

    // 2. 非实体类权限（image/attachment/comment/restrictions）
    //    只看系统权限，不走 JointPermission
    if (in_array($explodedPermission[0], $nonJointPermissions)) {
        return $hasRolePermission;
    }

    // 3. 系统权限 + EntityPermission 联合判断
    if ($hasRolePermission) {
        // 有系统权限 → 再检查 EntityPermission 是否显式拒绝
        $entityPermissionResult = $this->hasEntityPermission($entity, $userRoleIds, $action);
        if (is_null($entityPermissionResult)) {
            return true;  // 无 EntityPermission 设置 → 系统权限说了算
        }
        return $entityPermissionResult;  // 有设置 → EntityPermission 说了算
    }

    // 4. 没有系统权限 → 检查 EntityPermission 是否显式允许
    $entityPermissionResult = $this->hasEntityPermission($entity, $userRoleIds, $action);
    if (is_null($entityPermissionResult)) {
        return false;  // 无设置 → 默认拒绝
    }
    return $entityPermissionResult;  // 有设置 → EntityPermission 说了算
}
```

**核心规则总结**：

| 系统权限 | EntityPermission 结果 | 最终结论 |
|----------|----------------------|----------|
| ✅ 有 | null（无设置） | ✅ 允许（系统权限生效） |
| ✅ 有 | EXPLICIT_ALLOW | ✅ 允许 |
| ✅ 有 | EXPLICIT_DENY | ❌ 拒绝（显式拒绝覆盖系统权限） |
| ✅ 有 | IMPLICIT_ALLOW | ✅ 允许 |
| ✅ 有 | IMPLICIT_DENY | ❌ 拒绝 |
| ❌ 无 | null（无设置） | ❌ 拒绝（默认拒绝） |
| ❌ 无 | EXPLICIT_ALLOW | ✅ 允许（实体级授权覆盖系统级拒绝） |
| ❌ 无 | EXPLICIT_DENY | ❌ 拒绝 |
| ❌ 无 | IMPLICIT_ALLOW | ✅ 允许 |
| ❌ 无 | IMPLICIT_DENY | ❌ 拒绝 |

> 💡 **关键点**：EntityPermission 的显式设置（EXPLICIT_*）可以**正反双向**覆盖系统权限。系统权限只是"默认值"，EntityPermission 是"精细化调整"。

### 17.6 权限层级总览图

```
┌─────────────────────────────────────────────────────────────┐
│                    系统管理员角色                             │
│              （完全绕过所有权限检查）                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   系统角色权限 (RolePermission)              │
│  page-view-all  page-view-own  page-create-all  ...         │
│  （决定用户"能不能做某类事"的基础许可）                      │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                  实体权限继承链 (EntityPermission)            │
│  Book → Chapter → Page                                       │
│  + 角色显式权限（就近覆盖）                                   │
│  + Fallback 阻断（遇到即停）                                  │
│  （对具体实体的精细化授权/拒绝）                              │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    Owner 权限 (owned_by)                     │
│  -xxx-view-own + 是所有者 → 允许                             │
│  - 显式拒绝时 owner 权限无效                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 十八、权限变更的通知与同步机制

### 18.1 权限变更的活动日志记录

每次权限变更都会记录 `PERMISSIONS_UPDATE` 活动日志。

触发位置：`app/Entities/Tools/PermissionsUpdater.php:40` 和 `:72`

```php
Activity::add(ActivityType::PERMISSIONS_UPDATE, $entity);
```

#### ActivityLogger 流水线

位置：`app/Activity/Tools/ActivityLogger.php:27`

```
Activity::add(type, detail)
        │
        ├─► 1. 写入 activities 表
        │     ├─► type（如 permissions_update）
        │     ├─► user_id（操作者）
        │     ├─► ip（IP 地址）
        │     ├─► detail（实体描述）
        │     └─► loggable_id / loggable_type（关联实体，多态）
        │
        ├─► 2. setNotification() —— 前端 Flash 提示
        │     └─► session()->flash('success', '权限更新成功')
        │
        ├─► 3. dispatchWebhooks() —— Webhook 分发
        │     └─► 异步 DispatchWebhookJob
        │
        ├─► 4. NotificationManager::handle() —— 内部通知
        │     └─► 根据活动类型查找对应的 Handler
        │
        └─► 5. Theme::dispatch(ACTIVITY_LOGGED) —— 主题事件钩子
```

### 18.2 权限变更的通知：没有内置通知

**重要发现**：`NotificationManager` 的默认 Handler 列表中，**没有** `PERMISSIONS_UPDATE` 对应的通知处理器。

位置：`app/Activity/Notifications/NotificationManager.php:47`

```php
public function loadDefaultHandlers(): void
{
    $this->registerHandler(ActivityType::PAGE_CREATE, PageCreationNotificationHandler::class);
    $this->registerHandler(ActivityType::PAGE_UPDATE, PageUpdateNotificationHandler::class);
    $this->registerHandler(ActivityType::COMMENT_CREATE, CommentCreationNotificationHandler::class);
    $this->registerHandler(ActivityType::COMMENT_CREATE, CommentMentionNotificationHandler::class);
    $this->registerHandler(ActivityType::COMMENT_UPDATE, CommentMentionNotificationHandler::class);
    // ❌ 没有 PERMISSIONS_UPDATE 的 Handler！
}
```

这意味着：
- ✅ 权限变更会写入活动日志（可在审计日志中查看）
- ✅ 权限变更会触发 Webhook（如果配置了 `all` 或 `permissions_update` 事件）
- ❌ 权限变更**不会**发送邮件通知给关注者或相关用户
- ❌ 权限变更**不会**产生站内通知

**为什么不通知？**
权限变更通常是低频的管理操作，且影响范围可能很大（如修改 Book 权限影响所有子页面），如果给每个受影响用户都发通知，可能造成通知风暴。

### 18.3 Watch（关注）系统的继承机制

虽然权限变更没有通知，但 Watch 系统本身也有一套类似的**继承机制**，值得对照理解。

#### Watch 级别

位置：`app/Activity/WatchLevels.php:9`

| 级别 | 值 | 含义 |
|------|-----|------|
| `DEFAULT` | -1 | 默认（未设置，不存储） |
| `IGNORE` | 0 | 忽略所有通知（不接收） |
| `NEW` | 1 | 仅新内容通知 |
| `UPDATES` | 2 | 新内容 + 更新通知 |
| `COMMENTS` | 3 | 新内容 + 更新 + 评论通知 |

#### Watch 的继承链评估

位置：`app/Activity/Tools/EntityWatchers.php:64`

```php
protected function getRelevantWatches(): array
{
    $entitiesInvolved = array_filter([
        $this->entity,                    // 自身
        $this->entity instanceof BookChild ? $this->entity->book : null,    // 所属 Book
        $this->entity instanceof Page ? $this->entity->chapter : null,     // 所属 Chapter
    ]);
    // 查询这些实体上的所有 Watch 记录
}
```

和权限继承链完全一致：`Page → Chapter → Book`

#### Watch 的"就近覆盖"规则

位置：`app/Activity/Tools/EntityWatchers.php:44`

```php
// 按 entity_type 排序：book -> chapter -> page
// （因为类型名字母序：book < chapter < page）
usort($watches, function (Watch $watchA, Watch $watchB) {
    $entityTypeDiff = $watchA->watchable_type <=> $watchB->watchable_type;
    // ...
});

// De-dupe by user id → 后面的覆盖前面的
// （因为 page 在排序后最后，所以 page 级别的 watch 会覆盖 book/chapter 级别的）
$levelByUserId = [];
foreach ($watches as $watch) {
    $levelByUserId[$watch->user_id] = $watch->level;
}
```

**和权限继承的异同**：

| 特性 | 权限继承 | Watch 继承 |
|------|----------|------------|
| 继承链 | Page → Chapter → Book | 相同 |
| 方向 | 自身优先（子覆盖父） | 相同（后遍历的覆盖先遍历的，page 最后） |
| 多角色 | 或运算 | 不涉及（每个用户一条 watch 记录） |
| Fallback 阻断 | 有（遇到 role_id=0 就停） | 无（总是查完整链） |
| IGNORE 级别 | 无对应概念 | 有（显式忽略，优先级最高） |

#### Watch 与权限的交互

位置：`app/Activity/Notifications/Handlers/BaseNotificationHandler.php:19`

发送通知前会做权限检查：

```php
// 防止发送给没有内容访问权限的用户
$permissions = new PermissionApplicator($user);
if (!$permissions->checkOwnableUserAccess($relatedModel, 'view')) {
    continue;  // 没有权限 → 不发通知
}
```

**保证**：用户不会收到自己无权查看的内容的通知。这是权限系统和通知系统的关键交汇点。

### 18.4 Webhook 同步机制

权限变更可以通过 Webhook 实现**实时外部同步**。

位置：`app/Activity/Tools/ActivityLogger.php:85`

```php
protected function dispatchWebhooks(string $type, string|Loggable $detail): void
{
    $webhooks = Webhook::query()
        ->whereHas('trackedEvents', function (Builder $query) use ($type) {
            $query->where('event', '=', $type)->orWhere('event', '=', 'all');
        })
        ->where('active', '=', true)
        ->get();

    foreach ($webhooks as $webhook) {
        dispatch(new DispatchWebhookJob($webhook, $type, $detail));
    }
}
```

#### Webhook 事件类型

`ActivityType` 中所有常量都可以作为 Webhook 事件，包括：
- `permissions_update` —— 实体权限变更
- `role_create` / `role_update` / `role_delete` —— 角色变更
- `user_create` / `user_update` / `user_delete` —— 用户变更

#### DispatchWebhookJob

位置：`app/Activity/DispatchWebhookJob.php`

- 异步队列执行（不阻塞主请求）
- 失败后自动重试（默认 Laravel 队列配置）
- 包含实体完整数据（含权限变更内容）

**这是权限变更实时同步到外部系统的唯一官方通道**。

### 18.5 权限缓存与数据一致性

权限变更后的"同步"主要是指 **JointPermission 缓存表的重建**，这在第十一章已有详述。此处补充几个关键的一致性保证：

#### 事务内一致性

权限更新操作在 `DatabaseTransaction` 中执行时：
- EntityPermission 的 delete + create
- JointPermission 的 delete + insert
- Activity 日志写入

以上操作在**同一个数据库事务**中，保证原子性。

#### 最终一致性场景

以下场景可能出现短暂的不一致窗口：

| 场景 | 不一致窗口 | 原因 |
|------|-----------|------|
| 全量重建 | 几秒~几分钟 | TRUNCATE 后逐块插入，中间状态表不完整 |
| 实体移动（如 Page 换 Chapter） | 毫秒级 | 先更新 parent_id，再重建权限，中间有时间差 |
| 批量书架下发 | 每个 Book 之间 | 循环处理，前一个已提交，后一个还未开始 |
| 角色删除 | 毫秒级 | 先删 role_user，再删 jointPermissions |

这些窗口都很小（除了全量重建），且都是"最终一致"的。

### 18.6 权限变更的审计追踪

所有权限相关的操作都有活动日志记录，可用于审计：

| 操作 | 活动类型 | 关联实体 |
|------|----------|----------|
| 实体权限变更 | `permissions_update` | 该实体 |
| 角色创建 | `role_create` | - |
| 角色更新 | `role_update` | - |
| 角色删除 | `role_delete` | - |
| 用户创建 | `user_create` | - |
| 用户更新 | `user_update` | - |
| 用户删除 | `user_delete` | - |
| 系统设置更新 | `settings_update` | - |

这些日志包含操作者、IP 地址、时间戳，满足基本的审计需求。

---

## 十九、最终设计要点总览

1. **自底向上就近优先**：继承链从自身开始向上查找，离得越近优先级越高
2. **角色显式覆盖父级**：同一角色的权限，子级设置覆盖父级设置
3. **Fallback 阻断机制**：遇到"其他所有人"设置后，停止向上查找
4. **多角色或运算**：用户拥有的多个角色中，任一角色显式允许即可通过
5. **预计算缓存加速**：JointPermission 表将继承计算物化，查询只需 JOIN + GROUP BY
6. **管理员例外**：系统管理员角色跳过所有权限检查
7. **Owner 权限独立**：xxx-view-own 权限在非显式拒绝情况下对所有者生效
8. **书架独立于继承链**：书架和书是多对多关系，不自动参与继承，需手动下发
9. **预计算缓存的写时更新**：JointPermission 不设 TTL，由业务代码在权限变更后显式触发重建
10. **READ COMMITTED 隔离级别**：避免权限重建时读取到过时的快照数据，保证并发场景下的数据新鲜度
11. **Last Writer Wins 并发策略**：无悲观锁，依赖事务的原子性保证单次操作的完整性
12. **全量重建的不可回滚风险**：TRUNCATE 是 DDL 操作，一旦执行无法回滚，需谨慎使用
13. **API 路径的事务缺口**：`ContentPermissionApiController::update()` 未包裹事务，存在部分提交风险
14. **单实体 vs 列表查询的分流**：列表查询走预计算表（一次 SQL），单实体检查走实时评估（灵活但多查询）
15. **MassEntityPermissionEvaluator 的批量优化**：重建时预加载所有权限到内存，避免逐实体查询的 N+1 问题
16. **非多租户架构**：无 tenant_id，通过角色 + EntityPermission + Owner 实现软隔离
17. **Public 角色的特殊地位**：实现公开/私密内容区分，是未登录用户的唯一身份
18. **系统权限与实体权限正交**：两套体系独立运作，EntityPermission 可双向覆盖系统权限
19. **角色无继承关系**：多角色权限是或叠加，没有父子角色模板继承
20. **权限变更无内置通知**：只有活动日志和 Webhook，没有邮件/站内通知
21. **Watch 系统的类似继承**：关注关系也遵循 Page→Chapter→Book 的就近覆盖规则
22. **通知的权限校验**：发送通知前检查接收者是否有内容访问权限，防止信息泄露
23. **Webhook 是外部同步的主通道**：异步队列执行，支持所有活动类型事件

