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

## 十、设计要点总结

1. **自底向上就近优先**：继承链从自身开始向上查找，离得越近优先级越高
2. **角色显式覆盖父级**：同一角色的权限，子级设置覆盖父级设置
3. **Fallback 阻断机制**：遇到"其他所有人"设置后，停止向上查找
4. **多角色或运算**：用户拥有的多个角色中，任一角色显式允许即可通过
5. **预计算缓存加速**：JointPermission 表将继承计算物化，查询只需 JOIN + GROUP BY
6. **管理员例外**：系统管理员角色跳过所有权限检查
7. **Owner 权限独立**：xxx-view-own 权限在非显式拒绝情况下对所有者生效
8. **书架独立于继承链**：书架和书是多对多关系，不自动参与继承，需手动下发

