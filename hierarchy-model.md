# 书架、册、章、页 四级实体模型关联与加载策略分析

## 一、类继承结构

```
Entity (抽象基类)
├── BookChild (抽象类)
│   ├── Chapter (章)
│   └── Page (页)
├── Book (册)
└── Bookshelf (书架)
```

### 1.1 Entity 基类 (`app/Entities/Models/Entity.php`)

所有实体的公共基类，定义了：
- 公共属性：`id`, `type`, `name`, `slug`, `book_id`, `chapter_id`, `priority` 等
- 关联方法：`tags()`, `permissions()`, `jointPermissions()` 等
- 作用域：`scopeVisible()` - 权限过滤
- 抽象方法：`getUrl()`, `relatedData()`, `getParent()`

### 1.2 BookChild 抽象类 (`app/Entities/Models/BookChild.php`)

Book 和 Chapter 的公共父类，仅定义了：
- `book(): BelongsTo` - 关联到所属 Book，支持软删除

### 1.3 各实体类位置

| 实体 | 类文件 |
|------|--------|
| 书架 | `app/Entities/Models/Bookshelf.php` |
| 册 | `app/Entities/Models/Book.php` |
| 章 | `app/Entities/Models/Chapter.php` |
| 页 | `app/Entities/Models/Page.php` |

---

## 二、数据表结构

采用**单表继承（STI）+ 分表存储内容

### 2.1 entities 主表

```
┌─────────────────────────────────────────────────────────┐
│                      entities                       │
├─────────────┬───────────────────────────────────┤
│ id            │ BIGINT 主键                       │
│ type          │ VARCHAR(10) 类型区分          │
│ name          │ 名称                             │
│ slug          │ URL 标识                        │
│ book_id       │ 所属册ID (NULL  │
│ chapter_id    │ 所属章ID (NULL  │
│ priority      │ 排序优先级                       │
│ created_at    │ 创建时间                         │
│ updated_at    │ 更新时间                         │
│ deleted_at    │ 删除时间 (软删除)              │
│ created_by    │ 创建者ID                       │
│ updated_by    │ 更新者ID                       │
│ owned_by      │ 拥有者ID                       │
└─────────────┴───────────────────────────────────┘
```

### 2.2 entity_container_data 表

存储 Bookshelf、Book、Chapter 的描述性内容：

```
┌─────────────────────────────────────────────────────┐
│              entity_container_data              │
├─────────────────────┬───────────────────────────┤
│ entity_id          │ 实体ID                    │
│ entity_type        │ 实体类型                  │
│ description       │ 描述文本                 │
│ description_html   │ 描述HTML                 │
│ default_template_id │ 默认模板ID            │
│ image_id          │ 封面图ID                  │
│ sort_rule_id      │ 排序规则ID                │
└─────────────────────┴───────────────────────────┘
```

### 2.3 entity_page_data 表

存储 Page 的内容数据：

```
┌─────────────────────────────────────────────────────┐
│                entity_page_data                │
├──────────────────┬──────────────────────────────┤
│ page_id          │ 页面ID (主键)               │
│ draft            │ 是否草稿                     │
│ template         │ 是否模板                     │
│ revision_count   │ 修订次数                     │
│ editor           │ 编辑器类型                   │
│ html             │ HTML 内容                     │
│ text             │ 纯文本内容                 │
│ markdown         │ Markdown 内容               │
└──────────────────┴──────────────────────────────┘
```

### 2.4 bookshelves_books 中间表

书架与册的多对多关联表：

```
┌─────────────────────────────────────────────────────┐
│                bookshelves_books                │
├──────────────────┬──────────────────────────────┤
│ bookshelf_id     │ 书架ID                     │
│ book_id          │ 册ID                        │
│ order            │ 排序顺序                    │
└──────────────────┴──────────────────────────────┘
```

---

## 三、模型关联关系

### 3.1 关联关系总图

```
┌─────────────┐        ┌─────────────┐        ┌─────────────┐        ┌─────────────┐
│ Bookshelf  │───┐    │    Book     │───┐    │   Chapter   │───┐    │    Page     │
│  (书架)    │   │    │    (册)      │   │    │    (章)      │   │    │    (页)      │
└─────────────┘   │    └─────────────┘   │    └─────────────┘   │    └─────────────┘
       │            │           │            │           │            │           │
       │            │           │            │           │            │           │
       └─BelongsToMany─┘           └─hasMany──────┘           └─hasMany──────┘
       (多对多)              (一对多)              (一对多)
            │            │           │            │           │            │           │
            │            │           │            │           │            │           │
            └───────────────┘           └───────────────┘           └───────────────┘
             belongsToMany             belongsTo               belongsTo
            (多对多)              (多对一)              (多对一)
```

### 3.2 各实体关联详情

#### Bookshelf (书架)

```php
// 多对多关联到 Book
public function books(): BelongsToMany
{
    return $this->belongsToMany(Book::class, 'bookshelves_books', 'bookshelf_id', 'book_id')
        ->select(['entities.*', 'entity_container_data.*'])
        ->withPivot('order')
        ->orderBy('order', 'asc');
}

// 权限过滤后的可见书籍
public function visibleBooks(): BelongsToMany
{
    return $this->books()->scopes('visible');
}
```

**关键点**：
- 通过中间表 `bookshelves_books` 关联
- 关联时同时查询 `entity_container_data` 表获取描述信息
- 按 `order` 字段排序

#### Book (册)

```php
// 一对多关联到 Chapter
public function chapters(): HasMany
{
    return $this->hasMany(Chapter::class);
}

// 一对多关联到 Page (所有页)
public function pages(): HasMany
{
    return $this->hasMany(Page::class);
}

// 直接隶属于册的页（不属于任何章）
public function directPages(): HasMany
{
    return $this->pages()->whereNull('chapter_id');
}

// 多对多关联到 Bookshelf
public function shelves(): BelongsToMany
{
    return $this->belongsToMany(Bookshelf::class, 'bookshelves_books', 'book_id', 'bookshelf_id');
}

// 获取册下可见的直接子项（章 + 直接页）
public function getDirectVisibleChildren(): Collection
{
    $pages = $this->directPages()->scopes('visible')->get();
    $chapters = $this->chapters()->scopes('visible')->get();
    return $pages->concat($chapters)->sortBy('priority')->sortByDesc('draft');
}
```

**关键点**：
- Page 可以直接隶属于 Book（`chapter_id` 为 NULL）
- Page 也可以隶属于 Chapter（`chapter_id` 非 NULL）
- `getDirectVisibleChildren()` 返回章和直接页的混合集合

#### Chapter (章)

```php
// 继承自 BookChild，自动拥有 book() 关联

// 一对多关联到 Page
public function pages(string $dir = 'ASC'): HasMany
{
    return $this->hasMany(Page::class)->orderBy('priority', $dir);
}

// 获取章下可见的页
public function getVisiblePages(): Collection
{
    return $this->pages()
        ->scopes('visible')
        ->orderBy('draft', 'desc')
        ->orderBy('priority', 'asc')
        ->get();
}
```

#### Page (页)

```php
// 继承自 BookChild，自动拥有 book() 关联

// 多对一关联到 Chapter（可选）
public function chapter(): BelongsTo
{
    return $this->belongsTo(Chapter::class);
}

// 检查是否属于某个章
public function hasChapter(): bool
{
    return $this->chapter()->count() > 0;
}
```

**关键点**：
- Page 始终有 `book_id`（通过 BookChild）
- Page 可以有 `chapter_id`（可选）

---

## 四、查询与加载策略

### 4.1 查询层架构

采用 **Repository + Queries** 双层架构：

```
Controllers (控制器)
      │
      ▼
EntityQueries (统一查询入口)
      │
      ├─ BookshelfQueries
      ├─ BookQueries
      ├─ ChapterQueries
      └─ PageQueries
```

#### EntityQueries (`app/Entities/Queries/EntityQueries.php`)

统一的查询入口，聚合各实体的查询类：

```php
public function __construct(
    public BookshelfQueries $shelves,
    public BookQueries $books,
    public ChapterQueries $chapters,
    public PageQueries $pages,
    ...
) {}
```

#### 各实体 Queries 类通用方法：

| 方法 | 说明 |
|------|------|
| `start()` | 启动查询构造器 |
| `findVisibleById()` | 按 ID 查询可见实体 |
| `findVisibleBySlugOrFail()` | 按 slug 查询可见实体 |
| `visibleForList()` | 列表查询（选择特定字段） |
| `visibleForContent()` | 内容查询（选择全部字段） |
| `visibleForListWithCover()` | 列表查询并加载封面 |

### 4.2 权限过滤机制

所有查询都通过 `scopeVisible()` 进行权限过滤：

```php
// Entity.php:150-153
public function scopeVisible(Builder $query): Builder
{
    return app()->make(PermissionApplicator::class)->restrictEntityQuery($query);
}
```

### 4.3 关键加载策略

#### 4.3.1 MixedEntityListLoader (`app/Entities/Tools/MixedEntityListLoader.php`)

**批量加载混合类型实体**，用于活动日志、搜索结果等场景：

```php
protected function idsByTypeToModelMap(array $idsByType, bool $eagerLoadParents, bool $withContents): array
{
    foreach ($idsByType as $type => $ids) {
        $base = $withContents ? 
            $this->queries->visibleForContentForType($type) : 
            $this->queries->visibleForListForType($type);
        
        $models = $base->whereIn('id', $ids)
            ->with($eagerLoadParents ? $this->getRelationsToEagerLoad($type) : [])
            ->get();
        ...
    }
}
```

**Eager Loading 策略 (`getRelationsToEagerLoad`)：

```php
protected function getRelationsToEagerLoad(string $type): array
{
    $toLoad = [];
    $loadVisible = fn (Relation $query) => $query->scopes('visible');

    if ($type === 'chapter' || $type === 'page') {
        $toLoad['book'] = $loadVisible;
    }

    if ($type === 'page') {
        $toLoad['chapter'] = $loadVisible;
    }

    return $toLoad;
}
```

**加载规则**：

| 实体类型 | 预加载关联 |
|---------|------------|
| page | book, chapter |
| chapter | book |
| book | - |
| bookshelf | - |

#### 4.3.2 BookContents (`app/Entities/Tools/BookContents.php`)

**构建书籍内容树**：

```php
public function getTree(bool $showDrafts = false, bool $renderPages = false): Collection
{
    $pages = $this->getPages($showDrafts, $renderPages);
    $chapters = $this->book->chapters()->scopes('visible')->get();
    
    // 按 chapter_id 分组页面
    $pages->groupBy('chapter_id')->each(function ($pages, $chapter_id) use ($chapterMap, &$lonePages) {
        $chapter = $chapterMap->get($chapter_id);
        if ($chapter) {
            $chapter->setAttribute('visible_pages', collect($pages)->sortBy($this->bookChildSortFunc()));
        } else {
            $lonePages = $lonePages->concat($pages);
        }
    });
    
    // 为所有实体设置 book 关联，避免 N+1
    $all->each(function (Entity $entity) use ($renderPages) {
        $entity->setRelation('book', $this->book);
        ...
    });
    
    return collect($chapters)->concat($lonePages)->sortBy($this->bookChildSortFunc());
}
```

**加载流程**：
1. 一次查询获取所有页
2. 一次查询获取所有章
3. 在内存中组装树形结构
4. 手动设置 `book` 关联（避免 N+1）

#### 4.3.3 子查询优化

在 ChapterQueries 和 PageQueries 中使用**子查询获取 book_slug 避免 JOIN：

```php
// ChapterQueries.php:62-71
public function visibleForList(): Builder
{
    return $this->start()
        ->scopes('visible')
        ->select(array_merge(static::$listAttributes, ['book_slug' => function ($builder) {
            $builder->select('slug')
                ->from('entities as books')
                ->where('type', '=', 'book')
                ->whereColumn('books.id', '=', 'entities.book_id');
        }]));
}
```

### 4.4 实际查询走法示例

#### 示例1：查询章详情页的侧边栏树

```
PageController::show()
└─ BookContents::getTree()
   ├─ 查询所有页 (1次查询)
   ├─ 查询所有章 (1次查询)
   └─ 内存组装树结构
```

#### 示例2：查询页详情（带父级预加载）

```
PageQueries::findVisibleBySlugsOrFail()
└─ with('book')  // 预加载 book
   └─ whereHas('book', ...)  // 通过 book slug 过滤
```

#### 示例3：混合实体列表加载

```
MixedEntityListLoader::loadIntoRelations()
└─ 按类型分组 ID
   └─ 每种类型一次查询
      └─ 根据类型预加载父级
         └─ 内存设置关联
```

---

## 五、特殊字段说明

### 5.1 book_slug 字段

- 存储在 `entities` 表中，实际通过子查询动态获取
- 用于 URL 生成时使用，避免 JOIN 查询
- Chapter 和 Page 列表查询时自动包含

### 5.2 priority 字段

- 同级实体的排序优先级
- 数值越大越靠后
- 新建时自动计算为当前最大 priority + 1

### 5.3 draft 字段（仅 Page）

- 标识是否为草稿
- 草稿在列表中默认不显示（除非 `showDrafts = true`
- 排序时草稿优先（`orderBy('draft', 'desc')）

---

## 六、关键代码位置速查

| 功能 | 文件位置 |
|------|----------|
| 实体基类 | `app/Entities/Models/Entity.php` |
| BookChild 抽象类 | `app/Entities/Models/BookChild.php` |
| 书架模型 | `app/Entities/Models/Bookshelf.php` |
| 册模型 | `app/Entities/Models/Book.php` |
| 章模型 | `app/Entities/Models/Chapter.php` |
| 页模型 | `app/Entities/Models/Page.php` |
| 内容数据表 | `app/Entities/Models/EntityPageData.php` |
| 容器数据表 | `app/Entities/Models/EntityContainerData.php` |
| 统一查询入口 | `app/Entities/Queries/EntityQueries.php` |
| 书架查询 | `app/Entities/Queries/BookshelfQueries.php` |
| 册查询 | `app/Entities/Queries/BookQueries.php` |
| 章查询 | `app/Entities/Queries/ChapterQueries.php` |
| 页查询 | `app/Entities/Queries/PageQueries.php` |
| 书籍内容树 | `app/Entities/Tools/BookContents.php` |
| 混合实体加载器 | `app/Entities/Tools/MixedEntityListLoader.php` |
| 书架 Repository | `app/Entities/Repos/BookshelfRepo.php` |
| 册 Repository | `app/Entities/Repos/BookRepo.php` |
| 章 Repository | `app/Entities/Repos/ChapterRepo.php` |
| 页 Repository | `app/Entities/Repos/PageRepo.php` |
| 数据库迁移 | `database/migrations/2025_09_15_132850_create_entities_table.php` |

---

## 七、权限校验链路（Permission Policy）

### 7.1 权限系统架构

```
Controllers / Models
      │
      ├─ scopeVisible() 作用域
      │     └─ PermissionApplicator::restrictEntityQuery()
      │
      ├─ userCan() 辅助函数
      │     └─ PermissionApplicator::checkOwnableUserAccess()
      │
      └─ checkOwnablePermission() 控制器方法
            └─ PermissionApplicator::checkOwnableUserAccess()
                  │
                  ├─ 角色权限检查（-all / -own）
                  └─ 实体权限检查（EntityPermissionEvaluator）
```

### 7.2 核心类与职责

#### 7.2.1 PermissionApplicator (`app/Permissions/PermissionApplicator.php`)

权限校验的核心入口类，提供三个主要校验方法：

**方法1：`scopeVisible()` → `restrictEntityQuery()`**

查询时的权限过滤，通过 `joint_permissions` 表预计算结果进行过滤：

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

**权限状态值说明**：
- `1` - EXPLICIT_ALLOW（显式允许）
- `2` - EXPLICIT_DENY（显式拒绝）
- `3` - IMPLICIT_ALLOW（隐式允许）
- `4` - IMPLICIT_DENY（隐式拒绝）

**方法2：`checkOwnableUserAccess()` - 单个实体权限检查**

```php
public function checkOwnableUserAccess(Model&OwnableInterface $ownable, string|Permission $permission): bool
{
    // 1. 解析权限名称（如 page-view → action=view）
    $permissionName = is_string($permission) ? $permission : $permission->value;
    $explodedPermission = explode('-', $permissionName);
    $action = $explodedPermission[1] ?? $explodedPermission[0];
    
    // 2. 检查角色级权限
    $fullPermission = count($explodedPermission) > 1 ? $permissionName : $ownable->getMorphClass() . '-' . $permissionName;
    $allRolePermission = $user->can($fullPermission . '-all');
    $ownRolePermission = $user->can($fullPermission . '-own');
    
    // 3. 检查是否为拥有者
    $isOwner = $user->id === $ownableFieldVal;
    $hasRolePermission = $allRolePermission || ($isOwner && $ownRolePermission);
    
    // 4. 检查实体级权限（如果设置了的话）
    $hasApplicableEntityPermissions = $this->hasEntityPermission($ownable, $userRoleIds, $action);
    
    // 5. 返回结果：实体权限优先，否则使用角色权限
    return is_null($hasApplicableEntityPermissions) ? $hasRolePermission : $hasApplicableEntityPermissions;
}
```

**方法3：`restrictDraftsOnPageQuery()` - 草稿过滤**

```php
public function restrictDraftsOnPageQuery(Builder $query): Builder
{
    return $query->where(function (Builder $query) {
        $query->where('draft', '=', false)
            ->orWhere(function (Builder $query) {
                $query->where('draft', '=', true)
                    ->where('owned_by', '=', $this->currentUser()->id);
            });
    });
}
```

#### 7.2.2 JointPermissionBuilder (`app/Permissions/JointPermissionBuilder.php`)

**预计算权限表**，将角色权限和实体权限预先计算到 `joint_permissions` 表，避免查询时复杂计算。

**触发时机**：
- 实体创建/更新时调用 `$entity->rebuildPermissions()`
- 角色权限变更时调用 `rebuildForRole()`
- 系统初始化时调用 `rebuildForAll()`

**重建单个实体的权限**：

```php
public function rebuildForEntity(Entity $entity): void
{
    $entities = [$entity];
    
    // Book 需要同时重建其下所有章节和页面
    if ($entity instanceof Book) {
        $books = $this->bookFetchQuery()->where('id', '=', $entity->id)->get();
        $this->buildJointPermissionsForBooks($books, $roles, true);
        return;
    }
    
    // BookChild (Chapter/Page) 需要同时重建所属 Book
    if ($entity instanceof BookChild) {
        $entities[] = $entity->book;
    }
    
    // Page 如果有 chapter_id，需要同时重建所属 Chapter
    if ($entity instanceof Page && $entity->chapter_id) {
        $entities[] = $entity->chapter;
    }
    
    // Chapter 需要同时重建其下所有 Page
    if ($entity instanceof Chapter) {
        foreach ($entity->pages as $page) {
            $entities[] = $page;
        }
    }
    
    $this->buildJointPermissionsForEntities($entities);
}
```

**权限计算逻辑 `createJointPermissionData()`**：

```php
protected function createJointPermissionData(SimpleEntityData $entity, int $roleId, ...): array
{
    // 1. 系统管理员角色直接全部允许
    if ($isAdminRole) {
        return $this->createJointPermissionDataArray($entity, $roleId, PermissionStatus::EXPLICIT_ALLOW, true);
    }
    
    // 2. 检查实体级权限（如果有设置）
    $entityPermissionStatus = $permissionMap->evaluateEntityForRole($entity, $roleId);
    if ($entityPermissionStatus !== null) {
        return $this->createJointPermissionDataArray($entity, $roleId, $entityPermissionStatus, false);
    }
    
    // 3. 默认使用角色级权限
    $permissionPrefix = $entity->type . '-view';
    $roleHasPermission = isset($rolePermissionMap[$roleId . ':' . $permissionPrefix . '-all']);
    $roleHasPermissionOwn = isset($rolePermissionMap[$roleId . ':' . $permissionPrefix . '-own']);
    $status = $roleHasPermission ? PermissionStatus::IMPLICIT_ALLOW : PermissionStatus::IMPLICIT_DENY;
    
    return $this->createJointPermissionDataArray($entity, $roleId, $status, $roleHasPermissionOwn);
}
```

### 7.3 权限校验接入点

#### 7.3.1 查询时接入 - `scopeVisible()`

```php
// Entity.php:150-153
public function scopeVisible(Builder $query): Builder
{
    return app()->make(PermissionApplicator::class)->restrictEntityQuery($query);
}

// Page 重载了 scopeVisible 以包含草稿过滤
public function scopeVisible(Builder $query): Builder
{
    $query = app()->make(PermissionApplicator::class)->restrictDraftsOnPageQuery($query);
    return parent::scopeVisible($query);
}
```

#### 7.3.2 操作时接入 - `userCan()` 辅助函数

在 Controller 或业务逻辑中使用：

```php
// 检查是否有更新权限
if (!userCan(Permission::PageUpdate, $page)) {
    abort(403);
}

// 检查是否有删除权限
$this->checkOwnablePermission(Permission::BookDelete, $book);
```

### 7.4 joint_permissions 表结构

```
┌─────────────────────────────────────────────────────┐
│                joint_permissions                │
├──────────────────┬──────────────────────────────┤
│ entity_id        │ 实体ID                       │
│ entity_type      │ 实体类型                     │
│ role_id          │ 角色ID                       │
│ status           │ 权限状态 (1-4)              │
│ owner_id         │ 拥有者ID (仅 own 权限有效) │
└──────────────────┴──────────────────────────────┘
```

**复合主键**：`(entity_id, entity_type, role_id)`

---

## 八、软删除回收链路（Soft Delete）

### 8.1 软删除架构

```
删除操作
    │
    ▼
Repo::destroy()
    │
    └─ TrashCan::softDestroyXxx()
          │
          ├─ ensureDeletable()  // 检查是否可删除
          ├─ Deletion::createForEntity()  // 记录删除
          └─ $entity->delete()  // 软删除（设置 deleted_at）
                │
                └─ SoftDeletes trait

回收站操作
    │
    ├─ 恢复：TrashCan::restoreFromDeletion()
    │     └─ $entity->restore() + 级联恢复子项
    │
    └─ 永久删除：TrashCan::destroyFromDeletion()
          ├─ destroyCommonRelations()  // 清理关联数据
          └─ $entity->forceDelete()  // 物理删除
```

### 8.2 核心类与职责

#### 8.2.1 TrashCan (`app/Entities/Tools/TrashCan.php`)

软删除操作的核心工具类，所有删除操作都通过此类执行。

**软删除方法**：

```php
// 书架软删除 - 无子项需要级联
public function softDestroyShelf(Bookshelf $shelf)
{
    $this->ensureDeletable($shelf);
    Deletion::createForEntity($shelf);
    $shelf->delete();
}

// 册软删除 - 需要级联删除章节和页面
public function softDestroyBook(Book $book)
{
    $this->ensureDeletable($book);
    Deletion::createForEntity($book);

    // 级联软删除所有页面（recordDelete=false 表示不单独记录删除）
    foreach ($book->pages as $page) {
        $this->softDestroyPage($page, false);
    }

    // 级联软删除所有章节
    foreach ($book->chapters as $chapter) {
        $this->softDestroyChapter($chapter, false);
    }

    $book->delete();
}

// 章软删除 - 需要级联删除页面
public function softDestroyChapter(Chapter $chapter, bool $recordDelete = true)
{
    if ($recordDelete) {
        $this->ensureDeletable($chapter);
        Deletion::createForEntity($chapter);
    }

    // 级联软删除所有页面
    if (count($chapter->pages) > 0) {
        foreach ($chapter->pages as $page) {
            $this->softDestroyPage($page, false);
        }
    }

    $chapter->delete();
}

// 页软删除 - 无子项
public function softDestroyPage(Page $page, bool $recordDelete = true)
{
    if ($recordDelete) {
        $this->ensureDeletable($page);
        Deletion::createForEntity($page);
    }

    $page->delete();
}
```

**删除前检查 `ensureDeletable()`**：

```php
protected function ensureDeletable(Entity $entity): void
{
    // 检查是否被设置为自定义首页
    $customHomeId = intval(explode(':', setting('app-homepage', '0:'))[0]);
    $customHomeActive = setting('app-homepage-type') === 'page';
    
    // Page 本身是首页
    if ($entity instanceof Page && $entity->id === $customHomeId) {
        if ($customHomeActive) {
            throw new NotifyException(trans('errors.page_custom_home_deletion'), $entity->getUrl());
        }
        $removeCustomHome = true;
    }
    
    // Chapter 或 Book 包含首页
    if ($entity instanceof Chapter || $entity instanceof Book) {
        if ($entity->pages()->where('id', '=', $customHomeId)->exists()) {
            if ($customHomeActive) {
                throw new NotifyException(trans('errors.page_custom_home_deletion'), $entity->getUrl());
            }
            $removeCustomHome = true;
        }
    }
    
    if ($removeCustomHome) {
        setting()->remove('app-homepage');
    }
}
```

**恢复方法 `restoreEntity()`**：

```php
protected function restoreEntity(Entity $entity): int
{
    $count = 1;
    $entity->restore();

    $restoreAction = function ($entity) use (&$count) {
        if ($entity->deletions_count > 0) {
            $entity->deletions()->delete();
        }
        $entity->restore();
        $count++;
    };

    // 恢复章节/册时级联恢复其下的页面
    if ($entity instanceof Chapter || $entity instanceof Book) {
        $entity->pages()->withTrashed()->withCount('deletions')->get()->each($restoreAction);
    }

    // 恢复册时级联恢复其下的章节
    if ($entity instanceof Book) {
        $entity->chapters()->withTrashed()->withCount('deletions')->get()->each($restoreAction);
    }

    return $count;
}
```

**永久删除方法 `destroyEntity()`**：

```php
public function destroyEntity(Entity $entity): int
{
    if ($entity instanceof Page) {
        return $this->destroyPage($entity);
    } else if ($entity instanceof Chapter) {
        return $this->destroyChapter($entity);
    } else if ($entity instanceof Book) {
        return $this->destroyBook($entity);
    } else if ($entity instanceof Bookshelf) {
        return $this->destroyShelf($entity);
    }
}
```

**永久删除 Page 时的清理工作**：

```php
protected function destroyPage(Page $page): int
{
    $this->destroyCommonRelations($page);
    $page->allRevisions()->delete();  // 删除所有修订

    // 删除附件文件
    $attachmentService = app()->make(AttachmentService::class);
    foreach ($page->attachments as $attachment) {
        $attachmentService->deleteFile($attachment);
    }

    // 移除作为默认模板的引用
    EntityContainerData::query()
        ->where('default_template_id', '=', $page->id)
        ->update(['default_template_id' => null]);

    // 清空上传图片的关联
    Image::query()
        ->whereIn('type', ['gallery', 'drawio'])
        ->where('uploaded_to', '=', $page->id)
        ->update(['uploaded_to' => null]);

    $page->forceDelete();
    return 1;
}
```

**通用关联清理 `destroyCommonRelations()`**：

```php
protected function destroyCommonRelations(Entity $entity): void
{
    Activity::removeEntity($entity);
    $entity->views()->delete();
    $entity->permissions()->delete();
    $entity->tags()->delete();
    $entity->comments()->delete();
    $entity->jointPermissions()->delete();
    $entity->searchTerms()->delete();
    $entity->deletions()->delete();
    $entity->favourites()->delete();
    $entity->watches()->delete();
    $entity->referencesTo()->delete();
    $entity->referencesFrom()->delete();
    $entity->slugHistory()->delete();

    if ($entity instanceof HasCoverInterface && $entity->coverInfo()->exists()) {
        $imageService = app()->make(ImageService::class);
        $imageService->destroy($entity->coverInfo()->getImage());
    }

    $entity->relatedData()->delete();  // 删除 entity_container_data 或 entity_page_data
}
```

#### 8.2.2 Deletion (`app/Entities/Models/Deletion.php`)

删除记录模型，用于回收站列表展示。

```php
class Deletion extends Model implements Loggable
{
    // 多态关联到被删除的实体
    public function deletable(): MorphTo
    {
        return $this->morphTo('deletable')->withTrashed();
    }

    // 关联到删除执行者
    public function deleter(): BelongsTo
    {
        return $this->belongsTo(User::class, 'deleted_by');
    }

    // 创建删除记录
    public static function createForEntity(Entity $entity): self
    {
        $record = (new self())->forceFill([
            'deleted_by'     => user()->id,
            'deletable_type' => $entity->getMorphClass(),
            'deletable_id'   => $entity->id,
        ]);
        $record->save();
        return $record;
    }
}
```

#### 8.2.3 Deletion 表结构

```
┌─────────────────────────────────────────────────────┐
│                   deletions                      │
├──────────────────┬──────────────────────────────┤
│ id               │ 主键                         │
│ deleted_by       │ 删除者ID                     │
│ deletable_type   │ 被删除实体类型               │
│ deletable_id     │ 被删除实体ID                 │
│ created_at       │ 删除时间                     │
└──────────────────┴──────────────────────────────┘
```

### 8.3 自动清理机制

```php
public function autoClearOld(): int
{
    $lifetime = intval(config('app.recycle_bin_lifetime'));
    if ($lifetime < 0) {
        return 0;  // 负数表示不自动清理
    }

    $clearBeforeDate = Carbon::now()->addSeconds(10)->subDays($lifetime);
    $deleteCount = 0;

    // 删除超过生命周期的记录
    $deletionsToRemove = Deletion::query()->where('created_at', '<', $clearBeforeDate)->get();
    foreach ($deletionsToRemove as $deletion) {
        $deleteCount += $this->destroyFromDeletion($deletion);
    }

    return $deleteCount;
}
```

**调用时机**：每次软删除后调用 `$this->trashCan->autoClearOld()`

### 8.4 接入点示例

#### Repository 层调用

```php
// BookshelfRepo.php:98-103
public function destroy(Bookshelf $shelf): void
{
    $this->trashCan->softDestroyShelf($shelf);
    Activity::add(ActivityType::BOOKSHELF_DELETE, $shelf);
    $this->trashCan->autoClearOld();  // 自动清理旧记录
}
```

#### 控制器层调用

```php
// RecycleBinController.php:79-86
public function restore(DeletionRepo $deletionRepo, string $id)
{
    $restoreCount = $deletionRepo->restore((int) $id);
    $this->showSuccessNotification(trans('settings.recycle_bin_restore_notification', ['count' => $restoreCount]));
    return redirect($this->recycleBinBaseUrl);
}
```

---

## 九、版本管理链路（Revision）

### 9.1 版本管理架构

```
页面编辑流程
    │
    ├─ 保存草稿（自动保存）
    │   └─ RevisionRepo::updatePageDraft()
    │         └─ 创建 update_draft 类型的 PageRevision
    │
    └─ 发布/更新
         ├─ PageRepo::publishDraft() / update()
         │    └─ RevisionRepo::storeNewForPage()
         │          ├─ 创建 version 类型的 PageRevision
         │          ├─ revision_number = page->revision_count
         │          └─ deleteOldRevisions()  // 清理超出版本限制的旧版本
         │
         └─ revision_count++

版本操作
    ├─ 列表：PageRevisionController::index()
    ├─ 预览：PageRevisionController::show()
    ├─ 对比：PageRevisionController::changes()
    └─ 恢复：PageRepo::restoreRevision()
```

### 9.2 核心类与职责

#### 9.2.1 PageRevision (`app/Entities/Models/PageRevision.php`)

修订记录模型，存储页面的历史版本。

```php
class PageRevision extends Model implements Loggable
{
    protected $fillable = ['name', 'text', 'summary'];
    protected $hidden = ['html', 'markdown', 'text'];

    // 关联到所属页面
    public function page(): BelongsTo
    {
        return $this->belongsTo(Page::class);
    }

    // 关联到创建者
    public function createdBy(): BelongsTo
    {
        return $this->belongsTo(User::class, 'created_by');
    }

    // 获取前一个修订
    public function getPreviousRevision(): ?PageRevision
    {
        $id = static::newQuery()->where('page_id', '=', $this->page_id)
            ->where('id', '<', $this->id)
            ->max('id');

        return $id ? static::query()->find($id) : null;
    }
}
```

#### 9.2.2 PageRevision 表结构

```
┌─────────────────────────────────────────────────────┐
│                page_revisions                   │
├──────────────────┬──────────────────────────────┤
│ id               │ 主键                         │
│ page_id          │ 页面ID                       │
│ name             │ 页面名称（快照）             │
│ slug             │ 页面 slug（快照）            │
│ book_slug        │ 册 slug（快照）              │
│ html             │ HTML 内容快照                │
│ markdown         │ Markdown 内容快照            │
│ text             │ 纯文本内容快照               │
│ type             │ 类型：version / update_draft │
│ revision_number  │ 版本号（仅 version 类型）    │
│ summary          │ 修订摘要                     │
│ created_by       │ 创建者ID                     │
│ created_at       │ 创建时间                     │
│ updated_at       │ 更新时间                     │
└──────────────────┴──────────────────────────────┘
```

**类型说明**：
- `version` - 正式发布的版本，有 `revision_number`
- `update_draft` - 编辑过程中的自动保存草稿，无版本号

#### 9.2.3 RevisionRepo (`app/Entities/Repos/RevisionRepo.php`)

修订管理的核心服务类。

**创建正式版本 `storeNewForPage()`**：

```php
public function storeNewForPage(Page $page, ?string $summary = null): PageRevision
{
    $revision = new PageRevision();

    // 保存当前页面内容快照
    $revision->name = $page->name;
    $revision->html = $page->html;
    $revision->markdown = $page->markdown;
    $revision->text = $page->text;
    $revision->page_id = $page->id;
    $revision->slug = $page->slug;
    $revision->book_slug = $page->book->slug;
    $revision->created_by = user()->id;
    $revision->created_at = $page->updated_at;
    $revision->type = 'version';
    $revision->summary = $summary;
    $revision->revision_number = $page->revision_count;  // 使用当前 revision_count
    $revision->save();

    $this->deleteOldRevisions($page);  // 清理旧版本

    return $revision;
}
```

**旧版本清理 `deleteOldRevisions()`**：

```php
protected function deleteOldRevisions(Page $page): void
{
    $revisionLimit = config('app.revision_limit');
    if ($revisionLimit === false) {
        return;  // false 表示不限制
    }

    // 保留最近 N 个版本，删除其余的
    $revisionsToDelete = PageRevision::query()
        ->where('page_id', '=', $page->id)
        ->orderBy('created_at', 'desc')
        ->skip(intval($revisionLimit))
        ->take(10)
        ->get(['id']);

    if ($revisionsToDelete->count() > 0) {
        PageRevision::query()->whereIn('id', $revisionsToDelete->pluck('id'))->delete();
    }
}
```

**获取/创建编辑草稿 `getNewDraftForCurrentUser()`**：

```php
public function getNewDraftForCurrentUser(Page $page): PageRevision
{
    // 查找现有草稿
    $draft = $this->queries->findLatestCurrentUserDraftsForPageId($page->id);

    if ($draft) {
        return $draft;
    }

    // 创建新草稿
    $draft = new PageRevision();
    $draft->page_id = $page->id;
    $draft->slug = $page->slug;
    $draft->book_slug = $page->book->slug;
    $draft->created_by = user()->id;
    $draft->type = 'update_draft';

    return $draft;
}
```

#### 9.2.4 PageRevisionQueries (`app/Entities/Queries/PageRevisionQueries.php`)

修订查询类。

```php
class PageRevisionQueries
{
    // 查找用户最新的编辑草稿
    public function findLatestCurrentUserDraftsForPageId(int $pageId): ?PageRevision
    {
        return $this->latestCurrentUserDraftsForPageId($pageId)->first();
    }

    public function latestCurrentUserDraftsForPageId(int $pageId): Builder
    {
        return $this->start()
            ->where('created_by', '=', user()->id)
            ->where('type', 'update_draft')
            ->where('page_id', '=', $pageId)
            ->orderBy('created_at', 'desc');
    }
}
```

### 9.3 Page 模型中的接入点

```php
// Page.php:75-81
// 仅返回正式版本（排除草稿）
public function revisions(): HasMany
{
    return $this->allRevisions()
        ->where('type', '=', 'version')
        ->orderBy('created_at', 'desc')
        ->orderBy('id', 'desc');
}

// 返回所有修订（包括草稿）
public function allRevisions(): HasMany
{
    return $this->hasMany(PageRevision::class);
}

// 获取当前最新版本
public function currentRevision(): HasOne
{
    return $this->hasOne(PageRevision::class)
        ->where('type', '=', 'version')
        ->orderBy('created_at', 'desc')
        ->orderBy('id', 'desc');
}
```

### 9.4 修订恢复流程

```php
// PageRepo.php:229-265
public function restoreRevision(Page $page, int $revisionId): Page
{
    $oldUrl = $page->getUrl();
    $page->revision_count++;  // 恢复操作也会增加版本号

    // 查找要恢复的修订
    $revision = $page->revisions()->where('id', '=', $revisionId)->first();

    // 用修订内容填充页面
    $page->fill($revision->toArray());
    $content = new PageContent($page);

    if (!empty($revision->markdown)) {
        $content->setNewMarkdown($revision->markdown, user());
    } else {
        $content->setNewHTML($revision->html, user());
    }

    $page->updated_by = user()->id;
    $this->baseRepo->refreshSlug($page);
    $page->save();
    $page->indexForSearch();
    $this->referenceStore->updateForEntity($page);

    // 创建新的修订记录（记录恢复操作）
    $summary = trans('entities.pages_revision_restored_from', ['id' => strval($revisionId), 'summary' => $revision->summary]);
    $this->revisionRepo->storeNewForPage($page, $summary);

    // 如果 URL 变化，更新引用
    if ($oldUrl !== $page->getUrl()) {
        $this->referenceUpdater->updateEntityReferences($page, $oldUrl);
    }

    Activity::add(ActivityType::PAGE_RESTORE, $page);
    Activity::add(ActivityType::REVISION_RESTORE, $revision);

    $this->baseRepo->sortParent($page);

    return $page;
}
```

### 9.5 修订对比流程

```php
// PageRevisionController.php:98-128
public function changes(string $bookSlug, string $pageSlug, int $revisionId)
{
    $page = $this->pageQueries->findVisibleBySlugsOrFail($bookSlug, $pageSlug);
    $revision = $page->revisions()->where('id', '=', $revisionId)->first();

    // 获取前一个版本
    $prev = $revision->getPreviousRevision();
    $prevContent = $prev->html ?? '';

    // 使用 HtmlDiff 库计算差异
    $rawDiff = Diff::excecute($prevContent, $revision->html);
    
    // 内容过滤（安全处理）
    $filterConfig = HtmlContentFilterConfig::fromConfigString(config('app.content_filtering'));
    $filter = new HtmlContentFilter($filterConfig);
    $diff = $filter->filterString($rawDiff);

    return view('pages.revision', [
        'page'     => $page,
        'book'     => $page->book,
        'diff'     => $diff,
        'revision' => $revision,
    ]);
}
```

### 9.6 版本号管理机制

1. **初始发布**：`publishDraft()` 时设置 `revision_count = 1`
2. **每次更新**：`update()` 时 `revision_count++`
3. **恢复版本**：`restoreRevision()` 时 `revision_count++`（恢复操作本身也算一个新版本）
4. **保存修订**：`storeNewForPage()` 时 `revision_number = page->revision_count`

---

## 十一、全文搜索与索引链路（Search）

### 11.1 搜索系统架构

```
实体变更（创建/更新/删除）
    │
    └─ Entity::indexForSearch()
          │
          └─ SearchIndex::indexEntity()
                │
                ├─ deleteEntityTerms()  // 先清除旧索引
                ├─ entityToTermDataArray()  // 生成词项-分数映射
                │     │
                │     ├─ generateTermScoreMapFromText(name, ×40)
                │     ├─ generateTermScoreMapFromHtml(description/html)
                │     │     ├─ h1: ×10
                │     │     ├─ h2: ×5
                │     │     ├─ h3: ×4
                │     │     ├─ h4: ×3
                │     │     ├─ h5: ×2
                │     │     ├─ h6: ×1.5
                │     │     └─ 其他: ×1
                │     └─ generateTermScoreMapFromTags()
                │           ├─ tag name: ×3
                │           └─ tag value: ×5
                │
                └─ insertTerms()  // 批量写入 search_terms 表

用户搜索请求
    │
    └─ SearchController::search()
          │
          └─ SearchRunner::searchEntities()
                │
                ├─ buildQuery()
                │     │
                │     ├─ applyTermSearch()  // 词项搜索（TF-IDF 风格）
                │     │     │
                │     │     ├─ getTermAdjustments()  // 稀有词加权（越罕见分越高）
                │     │     │     └─ multiplier = 1.3 - (term_count / max_count)
                │     │     └─ selectForScoredTerms()  // 构建 SUM(IF(...)) 计算总分
                │     │
                │     ├─ applyTagSearch()  // 标签搜索
                │     └─ 过滤器：exact / filters（updated_after 等）
                │
                └─ EntityHydrator::hydrate()  // 权限检查 + 预加载父级
```

### 11.2 核心类与职责

#### 11.2.1 SearchIndex (`app/Search/SearchIndex.php`)

**索引管理核心类**，负责实体的索引创建、更新、删除。

**索引单个实体 `indexEntity()`**：

```php
public function indexEntity(Entity $entity): void
{
    $this->deleteEntityTerms($entity);  // 先删旧索引
    $terms = $this->entityToTermDataArray($entity);  // 生成新词项
    $this->insertTerms($terms);  // 写入数据库
}
```

**词项分词规则 `textToTermCountMap()`**：

```php
// 硬分隔符：空格、换行、标点等
public static string $delimiters = " \n\t.-,!?:;()[]{}<>`'\"«»";

// 软分隔符：既保留完整词，也拆分后的词
public static string $softDelimiters = ".-";

// 例：输入 "my-app.v2.0"
// 分词结果：my, my-app, my-app.v2, my-app.v2.0, app, app.v2, app.v2.0, v2, v2.0, 0
```

**各字段权重系数**：

| 字段来源 | 权重系数 | 说明 |
|---------|---------|------|
| 实体名称（name） | `×40 × searchFactor` | 最高权重 |
| 标签名（tag name） | `×3` | |
| 标签值（tag value） | `×5` | |
| H1 标题 | `×10` | |
| H2 标题 | `×5` | |
| H3 标题 | `×4` | |
| H4 标题 | `×3` | |
| H5 标题 | `×2` | |
| H6 标题 | `×1.5` | |
| 普通内容 / 描述 | `×1 × searchFactor` | 基础权重 |

**全量重建索引 `indexAllEntities()`**：

```php
public function indexAllEntities(?callable $progressCallback = null): void
{
    SearchTerm::query()->truncate();  // 清空所有索引

    foreach ($this->entityProvider->all() as $entityModel) {
        // 分批处理，每批 250 条
        $entityModel->newQuery()
            ->select($selectFields)
            ->with(['tags:id,name,value,entity_id,entity_type'])
            ->chunk(250, $chunkCallback);
    }
}
```

#### 11.2.2 SearchTerm (`app/Search/SearchTerm.php`)

搜索词项存储模型。

**表结构**：

```
┌─────────────────────────────────────────────────────┐
│                  search_terms                    │
├──────────────────┬──────────────────────────────┤
│ id               │ 主键（自增）                 │
│ term             │ 词项文本                     │
│ entity_id        │ 实体ID                       │
│ entity_type      │ 实体类型                     │
│ score            │ 权重分数                     │
└──────────────────┴──────────────────────────────┘
```

**索引**：`(term, entity_type, entity_id)` 复合索引用于快速查找

#### 11.2.3 SearchRunner (`app/Search/SearchRunner.php`)

**搜索查询执行器**，执行实际的搜索查询并返回结果。

**词项搜索核心逻辑 `applyTermSearch()`**：

```php
protected function applyTermSearch(EloquentBuilder $entityQuery, SearchOptions $options, array $entityTypes): void
{
    // 1. 计算每个词的稀有度权重（稀有词得分更高）
    $scoredTerms = $this->getTermAdjustments($options);
    // multiplier = 1.3 - (term_count / max_count)
    // 例：最常见词 ×0.3，最罕见词 ×1.3

    // 2. 构建子查询：从 search_terms 计算每个实体的总分
    $subQuery = DB::table('search_terms')->select([
        'entity_id',
        'entity_type',
        DB::raw('SUM(IF(term like ?, score * ?, IF(...))) as score'),
    ]);
    $subQuery->groupBy('entity_type', 'entity_id');

    // 3. 与 entities 表 JOIN，并按分数排序
    $entityQuery->joinSub($subQuery, 's', function (JoinClause $join) {
        $join->on('s.entity_id', '=', 'entities.id')
            ->on('s.entity_type', '=', 'entities.type');
    });
    $entityQuery->orderBy('score', 'desc');
}
```

**支持的搜索过滤器**：

| 过滤器 | 说明 |
|-------|------|
| `{type}` | 按实体类型（page/chapter/book/bookshelf） |
| `updated_after` | 更新时间晚于 |
| `updated_before` | 更新时间早于 |
| `created_after` | 创建时间晚于 |
| `created_before` | 创建时间早于 |
| `created_by` | 创建者 |
| `updated_by` | 更新者 |
| `owned_by` | 拥有者 |
| `in_name` / `in_title` | 仅在名称中搜索 |
| `in_body` | 仅在正文中搜索 |
| `is_restricted` | 是否有自定义权限 |
| `viewed_by_me` | 我看过的 |
| `not_viewed_by_me` | 我没看过的 |
| `is_template` | 是否为模板页 |
| `sort_by_last_commented` | 按最后评论时间排序 |
| `[tag=value]` | 标签搜索 |
| `"exact phrase"` | 精确短语匹配 |

#### 11.2.4 EntityHydrator (`app/Entities/Tools/EntityHydrator.php`)

搜索结果后处理器：
- 对结果执行权限检查
- 预加载父级关联（book、chapter）
- 避免 N+1 查询

### 11.3 索引更新接入点

索引通过 `Entity::indexForSearch()` 方法触发：

```php
// Entity.php:402-405
public function indexForSearch(): void
{
    app()->make(SearchIndex::class)->indexEntity($this);
}
```

**调用时机**：

| 操作 | 触发位置 |
|------|---------|
| 创建实体 | `BaseRepo::create()`:65 |
| 更新实体 | `BaseRepo::update()`:100 |
| 恢复修订 | `PageRepo::restoreRevision()`:249 |
| 删除实体 | 不直接调用，而是在永久删除时 `destroyCommonRelations()` 中 `$entity->searchTerms()->delete()` |

**搜索接入点（查询）**：

| 功能 | 位置 |
|------|------|
| 全局搜索 | `SearchController::search()` |
| 册内搜索 | `SearchController::searchBook()` |
| 章内搜索 | `SearchController::searchChapter()` |
| 实体选择器搜索 | `SearchController::searchForSelector()` |
| 模板选择器搜索 | `SearchController::templatesForSelector()` |
| 搜索建议 | `SearchController::searchSuggestions()` |

### 11.4 search_terms 表数据量估算

假设：
- 平均每个页提取 200 个唯一词项
- 系统有 10,000 页 + 1,000 章 + 500 册 + 100 书架
- 总词项记录约：10,600 × 200 = **2,120,000 条**

---

## 十二、导出转换链路（Export）

### 12.1 导出系统架构

```
用户请求导出
    │
    └─ {Entity}ExportController::{format}()
          │
          ├─ Permission::ContentExport middleware  // 导出权限校验
          ├─ throttle:exports middleware  // 限流
          │
          └─ ExportFormatter::{entity}To{Format}()
                │
                ├─ PageContent::render()  // 渲染页面内容（短代码、引用等）
                │
                ├─ 格式为 PDF：
                │   └─ view('exports.{entity}') → containHtml() → PdfGenerator::fromHtml()
                │         │
                │         ├─ Engine: DomPDF（默认，PHP 实现）
                │         ├─ Engine: WkHtml（需配置二进制路径）
                │         └─ Engine: Command（自定义 shell 命令）
                │
                ├─ 格式为 HTML：
                │   └─ view('exports.{entity}') → containHtml()
                │         │
                │         ├─ 图片转 base64 内嵌
                │         └─ 相对链接转绝对 URL
                │
                ├─ 格式为 Markdown：
                │   └─ 有 markdown → 直接使用
                │   └─ 无 markdown → HtmlToMarkdown::convert()
                │
                ├─ 格式为 PlainText：
                │   └─ HtmlToPlainText::convert()
                │
                └─ 格式为 ZIP：
                      └─ ZipExportBuilder::buildFor{Entity}()
                            │
                            ├─ ZipExportFiles::generateFor{Entity}()
                            ├─ 导出 Markdown 文件
                            ├─ 导出图片（转本地文件）
                            ├─ 导出附件
                            ├─ 导出标签 JSON
                            └─ 打包 index.json 元数据
```

### 12.2 核心类与职责

#### 12.2.1 ExportFormatter (`app/Exports/ExportFormatter.php`)

**导出格式化核心类**，提供各实体到各格式的转换方法。

**支持的导出矩阵**：

| | Page | Chapter | Book |
|---|---|---|---|
| PDF | `pageToPdf()` | `chapterToPdf()` | `bookToPdf()` |
| HTML | `pageToContainedHtml()` | `chapterToContainedHtml()` | `bookToContainedHtml()` |
| Markdown | `pageToMarkdown()` | `chapterToMarkdown()` | `bookToMarkdown()` |
| PlainText | `pageToPlainText()` | `chapterToPlainText()` | `bookToPlainText()` |
| ZIP | 独立构建器 | 独立构建器 | 独立构建器 |

**HTML 自包含处理 `containHtml()`**：

```php
protected function containHtml(string $htmlContent): string
{
    // 1. 图片：src URL → base64 内嵌
    // <img src="/uploads/xxx.png"> → <img src="data:image/png;base64,...">
    foreach ($imageTagsOutput[0] as $index => $imgMatch) {
        $imageEncoded = $this->imageService->imageUrlToBase64($srcString);
        $htmlContent = str_replace($srcString, $imageEncoded, $htmlContent);
    }

    // 2. 链接：相对路径 → 绝对 URL
    // <a href="/books/xxx"> → <a href="https://example.com/books/xxx">
    foreach ($linksOutput[0] as $index => $linkMatch) {
        if (!str_starts_with(trim($srcString), 'http')) {
            $newSrcString = url($srcString);
            $htmlContent = str_replace($oldLinkString, $newLinkString, $htmlContent);
        }
    }

    return $htmlContent;
}
```

**PDF 特殊处理**（在 `htmlToPdf()` 中）：

```php
protected function htmlToPdf(string $html): string
{
    $html = $this->containHtml($html);
    $doc = new HtmlDocument();
    $doc->loadCompleteHtml($html);

    // 1. 将 iframe 替换为文本链接（PDF 不支持 iframe）
    $this->replaceIframesWithLinks($doc);
    // <iframe src="..."> → <p><a href="...">https://...</a></p>

    // 2. 展开所有 <details> 元素（PDF 不支持折叠交互）
    $this->openDetailElements($doc);
    // <details> → <details open="open">

    $cleanedHtml = $doc->getHtml();
    return $this->pdfGenerator->fromHtml($cleanedHtml);
}
```

**Markdown 转换规则**：

```php
public function pageToMarkdown(Page $page): string
{
    // 如果页面原生存储了 Markdown（用 Markdown 编辑器编辑的），直接使用
    if ($page->markdown) {
        return '# ' . $page->name . "\n\n" . $page->markdown;
    }

    // 否则从 HTML 转换
    return '# ' . $page->name . "\n\n" . (new HtmlToMarkdown($page->html))->convert();
}
```

**批量导出（Book/Chapter）**：

```php
// 导出册为 Markdown
public function bookToMarkdown(Book $book): string
{
    $bookTree = (new BookContents($book))->getTree(false, true);
    $text = '# ' . $book->name . "\n\n";

    // 描述信息
    $description = (new HtmlToMarkdown($book->descriptionInfo()->getHtml()))->convert();
    if ($description) {
        $text .= $description . "\n\n";
    }

    // 遍历章和页
    foreach ($bookTree as $bookChild) {
        if ($bookChild instanceof Chapter) {
            $text .= $this->chapterToMarkdown($bookChild) . "\n\n";
        } else {
            $text .= $this->pageToMarkdown($bookChild) . "\n\n";
        }
    }

    return trim($text);
}
```

#### 12.2.2 PdfGenerator (`app/Exports/PdfGenerator.php`)

**PDF 生成引擎选择器**，支持三种后端。

**引擎选择优先级**：

```php
public function getActiveEngine(): string
{
    // 1. 配置了自定义命令 → 使用命令引擎
    if (config('exports.pdf_command')) {
        return self::ENGINE_COMMAND;
    }

    // 2. 配置了 wkhtmltopdf 且允许非信任服务端获取 → 使用 WkHtml
    if ($this->getWkhtmlBinaryPath() && config('app.allow_untrusted_server_fetching') === true) {
        return self::ENGINE_WKHTML;
    }

    // 3. 默认使用 DomPDF（纯 PHP 实现）
    return self::ENGINE_DOMPDF;
}
```

**三种引擎对比**：

| 引擎 | 实现方式 | 优点 | 缺点 |
|-----|---------|------|------|
| DomPDF | PHP 库（默认） | 无需额外依赖、跨平台 | 复杂 CSS 支持有限、速度慢 |
| WkHtml | wkhtmltopdf 二进制 | 渲染效果好、支持现代 CSS | 需要安装二进制文件 |
| Command | 自定义 Shell 命令 | 完全灵活（可接 WeasyPrint 等） | 需要自行配置 |

**DomPDF 自定义字体支持**：

```php
// 从 storage/fonts/dompdf/*.ttf 加载自定义字体
// 字体文件命名规范：{FamilyName}-{Variation}.ttf
// 例：NotoSansSC-Regular.ttf → family: noto sans sc, variation: normal
protected function getUserDomPdfFontFamilies(): array
{
    $fontStore = storage_path('fonts/dompdf');
    $fontFiles = glob($fontStore . DIRECTORY_SEPARATOR . '*.ttf');
    // ... 自动生成 .ufm 字体度量文件并注册
}
```

#### 12.2.3 导出控制器

每个实体类型各有 Web 控制器和 API 控制器：

| 实体 | Web 控制器 | API 控制器 |
|------|-----------|-----------|
| Page | `PageExportController` | `PageExportApiController` |
| Chapter | `ChapterExportController` | `ChapterExportApiController` |
| Book | `BookExportController` | `BookExportApiController` |

**控制器通用模式**（以 Page 为例）：

```php
class PageExportController extends Controller
{
    public function __construct(
        protected PageQueries $queries,
        protected ExportFormatter $exportFormatter,
    ) {
        $this->middleware(Permission::ContentExport->middleware());  // 权限校验
        $this->middleware('throttle:exports');  // 限流保护
    }

    public function pdf(string $bookSlug, string $pageSlug)
    {
        $page = $this->queries->findVisibleBySlugsOrFail($bookSlug, $pageSlug);
        $page->html = (new PageContent($page))->render();  // 先渲染内容
        $pdfContent = $this->exportFormatter->pageToPdf($page);
        return $this->download()->directly($pdfContent, $pageSlug . '.pdf');
    }

    // html(), markdown(), plainText(), zip() 模式相同
}
```

### 12.3 ZIP 导出结构

```
export.zip
├── index.json          // 元数据（导出版本、实体类型、创建时间）
├── book.json           // 册/章/页信息 JSON
├── page-001.md         // 页面 Markdown
├── page-002.md
├── chapter-001/
│   └── page-003.md
├── images/
│   ├── image-001.png
│   └── image-002.jpg
├── attachments/
│   ├── file.pdf
│   └── doc.xlsx
└── tags.json           // 标签信息
```

---

## 十三、Activity 审计链路

### 13.1 审计系统架构

```
业务操作（创建/更新/删除/移动等）
    │
    └─ Activity::add(ActivityType::XXX, $entity)
          │
          └─ ActivityLogger::add()
                │
                ├─ 创建 Activity 记录
                │     ├─ type: 活动类型
                │     ├─ user_id: 当前用户
                │     ├─ ip: 客户端 IP
                │     ├─ detail: 描述信息
                │     └─ loggable_id/loggable_type: 关联实体
                │
                ├─ setNotification()  // 闪存成功提示
                ├─ dispatchWebhooks()  // 触发 Webhook 队列任务
                ├─ NotificationManager::handle()  // 触发站内/邮件通知
                └─ Theme::dispatch(ACTIVITY_LOGGED)  // 触发主题事件钩子
```

### 13.2 核心类与职责

#### 13.2.1 ActivityType (`app/Activity/ActivityType.php`)

**活动类型常量定义**，涵盖所有可审计操作。

**四级实体相关的活动类型**：

| 实体 | 创建 | 更新 | 删除 | 其他 |
|-----|------|------|------|------|
| Page | `PAGE_CREATE` | `PAGE_UPDATE` | `PAGE_DELETE` | `PAGE_RESTORE`, `PAGE_MOVE` |
| Chapter | `CHAPTER_CREATE` | `CHAPTER_UPDATE` | `CHAPTER_DELETE` | `CHAPTER_MOVE` |
| Book | `BOOK_CREATE`, `BOOK_CREATE_FROM_CHAPTER` | `BOOK_UPDATE` | `BOOK_DELETE` | `BOOK_SORT` |
| Bookshelf | `BOOKSHELF_CREATE`, `BOOKSHELF_CREATE_FROM_BOOK` | `BOOKSHELF_UPDATE` | `BOOKSHELF_DELETE` | - |

**其他活动类型**（节选）：

- 评论：`COMMENTED_ON`, `COMMENT_CREATE`, `COMMENT_UPDATE`, `COMMENT_DELETE`
- 权限：`PERMISSIONS_UPDATE`
- 版本：`REVISION_RESTORE`, `REVISION_DELETE`
- 回收站：`RECYCLE_BIN_EMPTY`, `RECYCLE_BIN_RESTORE`, `RECYCLE_BIN_DESTROY`
- 认证：`AUTH_LOGIN`, `AUTH_REGISTER`, `AUTH_PASSWORD_RESET_REQUEST`
- 用户/角色/API Token：`USER_CREATE`, `ROLE_UPDATE`, `API_TOKEN_DELETE` 等
- 设置：`SETTINGS_UPDATE`, `MAINTENANCE_ACTION_RUN`
- Webhook：`WEBHOOK_CREATE`, `WEBHOOK_UPDATE`, `WEBHOOK_DELETE`
- 导入：`IMPORT_CREATE`, `IMPORT_RUN`, `IMPORT_DELETE`
- 排序规则：`SORT_RULE_CREATE`, `SORT_RULE_UPDATE`, `SORT_RULE_DELETE`

#### 13.2.2 ActivityLogger (`app/Activity/Tools/ActivityLogger.php`)

**审计日志核心写入器**。

**主入口方法 `add()`**：

```php
public function add(string $type, string|Loggable $detail = ''): void
{
    // 1. 处理 detail 参数
    $detailToStore = ($detail instanceof Loggable) ? $detail->logDescriptor() : $detail;

    // 2. 创建活动记录
    $activity = $this->newActivityForUser($type);
    $activity->detail = $detailToStore;

    // 3. 如果传入的是 Entity，关联到该实体
    if ($detail instanceof Entity) {
        $activity->loggable_id = $detail->id;
        $activity->loggable_type = $detail->getMorphClass();
    }

    $activity->save();

    // 4. 触发副作用
    $this->setNotification($type);  // 前端 flash 消息
    $this->dispatchWebhooks($type, $detail);  // Webhook（异步队列）
    $this->notifications->handle($activity, $detail, user());  // 通知系统
    Theme::dispatch(ThemeEvents::ACTIVITY_LOGGED, $type, $detail);  // 主题事件
}
```

**创建活动实例 `newActivityForUser()`**：

```php
protected function newActivityForUser(string $type): Activity
{
    return (new Activity())->forceFill([
        'type'     => strtolower($type),
        'user_id'  => user()->id,
        'ip'       => IpFormatter::fromCurrentRequest()->format(),  // IP 脱敏处理
    ]);
}
```

**实体删除后的日志清理 `removeEntity()`**：

```php
// 当实体被永久删除时，将其活动日志与实体解绑
// 保留日志记录但不再关联到已删除的实体
public function removeEntity(Entity $entity): void
{
    $entity->activity()->update([
        'detail'         => $entity->name,    // 保留实体名作为文本描述
        'loggable_id'    => null,             // 清除关联 ID
        'loggable_type'  => null,             // 清除关联类型
    ]);
}
```

#### 13.2.3 Activity (`app/Activity/Models/Activity.php`)

**活动记录模型**。

**表结构**：

```
┌─────────────────────────────────────────────────────┐
│                   activities                     │
├──────────────────┬──────────────────────────────┤
│ id               │ 主键                         │
│ type             │ 活动类型（page_create 等）   │
│ detail           │ 描述文本                     │
│ user_id          │ 操作用户ID                   │
│ ip               │ 客户端 IP 地址               │
│ loggable_id      │ 关联实体ID（多态）           │
│ loggable_type    │ 关联实体类型（多态）         │
│ created_at       │ 操作时间                     │
│ updated_at       │ 更新时间                     │
└──────────────────┴──────────────────────────────┘
```

**关联方法**：

```php
// 多态关联到被操作的实体
public function loggable(): MorphTo
{
    return $this->morphTo('loggable');
}

// 关联到操作执行者
public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}

// 权限过滤用：通过 joint_permissions 关联
public function jointPermissions(): HasMany
{
    return $this->hasMany(JointPermission::class, 'entity_id', 'loggable_id')
        ->whereColumn('activities.loggable_type', '=', 'joint_permissions.entity_type');
}
```

**辅助方法**：

```php
// 获取活动的文本描述（从翻译文件查找）
public function getText(): string
{
    return trans('activities.' . $this->type);
}

// 判断是否为实体相关活动
public function isForEntity(): bool
{
    return Str::startsWith($this->type, [
        'page_', 'chapter_', 'book_', 'bookshelf_',
    ]);
}
```

#### 13.2.4 NotificationManager (`app/Activity/Notifications/NotificationManager.php`)

基于 Activity 的通知分发系统，支持：

- **CommentCreationNotificationHandler** - 新评论通知（给页面作者）
- **CommentMentionNotificationHandler** - 评论中 @ 提及通知（给被提及用户）
- **PageCreationNotificationHandler** - 新页面创建通知（给关注者）
- **PageUpdateNotificationHandler** - 页面更新通知（给关注者）

#### 13.2.5 Webhook 分发

```php
protected function dispatchWebhooks(string $type, string|Loggable $detail): void
{
    // 查询所有跟踪该事件类型的活跃 Webhook
    $webhooks = Webhook::query()
        ->whereHas('trackedEvents', function (Builder $query) use ($type) {
            $query->where('event', '=', $type)
                ->orWhere('event', '=', 'all');  // 'all' 表示跟踪所有事件
        })
        ->where('active', '=', true)
        ->get();

    // 异步分发
    foreach ($webhooks as $webhook) {
        dispatch(new DispatchWebhookJob($webhook, $type, $detail));
    }
}
```

### 13.3 Activity 调用栈（以 Page 操作为例）

#### 13.3.1 创建页（发布草稿）

```
PageController::store()
└─ PageRepo::publishDraft()
      ├─ BaseRepo::update()
      │    ├─ 保存实体
      │    ├─ rebuildPermissions()
      │    └─ indexForSearch()
      ├─ RevisionRepo::storeNewForPage()  // 保存版本
      └─ Activity::add(PAGE_CREATE, $draft)
            └─ ActivityLogger::add()
                  ├─ Activity::save()
                  ├─ setNotification()
                  ├─ dispatchWebhooks()
                  ├─ NotificationManager::handle()
                  └─ Theme::dispatch(ACTIVITY_LOGGED)
```

#### 13.3.2 更新页

```
PageController::update()
└─ PageRepo::update()
      ├─ BaseRepo::update()
      │    ├─ 保存实体
      │    └─ indexForSearch()
      ├─ RevisionRepo::storeNewForPage()  // 保存新版本
      └─ Activity::add(PAGE_UPDATE, $page)
```

#### 13.3.3 删除页（软删除）

```
PageController::destroy()
└─ PageRepo::destroy()
      ├─ TrashCan::softDestroyPage()
      │    ├─ ensureDeletable()
      │    ├─ Deletion::createForEntity()
      │    └─ $page->delete()  // 软删除
      ├─ Activity::add(PAGE_DELETE, $page)
      └─ TrashCan::autoClearOld()
```

#### 13.3.4 移动页

```
PageController::move()
└─ PageRepo::move()
      ├─ ParentChanger::changeBook()
      ├─ rebuildPermissions()
      └─ Activity::add(PAGE_MOVE, $page)
```

#### 13.3.5 恢复修订

```
PageRepo::restoreRevision()
├─ 恢复内容到页面
├─ indexForSearch()
├─ RevisionRepo::storeNewForPage()  // 保存恢复操作的新版本
├─ Activity::add(PAGE_RESTORE, $page)
└─ Activity::add(REVISION_RESTORE, $revision)
```

### 13.4 Entity 模型中的 Activity 接入点

```php
// Entity.php:214-217
public function activity(): MorphMany
{
    return $this->morphMany(Activity::class, 'loggable')
        ->orderBy('created_at', 'desc');
}
```

所有四级实体都继承此方法，可以直接查询某个实体的活动历史：

```php
$activities = $page->activity()->take(10)->get();
```

### 13.5 审计日志查询入口

| 功能 | 控制器 |
|------|--------|
| 审计日志页面 | `AuditLogController::index()` |
| 审计日志 API | `AuditLogApiController::index()` |

---

## 十四、关键代码位置速查（完整）

### 14.1 四级实体模型

| 功能 | 文件位置 |
|------|----------|
| 实体基类 | `app/Entities/Models/Entity.php` |
| BookChild 抽象类 | `app/Entities/Models/BookChild.php` |
| 书架模型 | `app/Entities/Models/Bookshelf.php` |
| 册模型 | `app/Entities/Models/Book.php` |
| 章模型 | `app/Entities/Models/Chapter.php` |
| 页模型 | `app/Entities/Models/Page.php` |
| 内容数据表 | `app/Entities/Models/EntityPageData.php` |
| 容器数据表 | `app/Entities/Models/EntityContainerData.php` |

### 14.2 查询层

| 功能 | 文件位置 |
|------|----------|
| 统一查询入口 | `app/Entities/Queries/EntityQueries.php` |
| 书架查询 | `app/Entities/Queries/BookshelfQueries.php` |
| 册查询 | `app/Entities/Queries/BookQueries.php` |
| 章查询 | `app/Entities/Queries/ChapterQueries.php` |
| 页查询 | `app/Entities/Queries/PageQueries.php` |

### 14.3 Repository 层

| 功能 | 文件位置 |
|------|----------|
| 基础 Repository | `app/Entities/Repos/BaseRepo.php` |
| 书架 Repository | `app/Entities/Repos/BookshelfRepo.php` |
| 册 Repository | `app/Entities/Repos/BookRepo.php` |
| 章 Repository | `app/Entities/Repos/ChapterRepo.php` |
| 页 Repository | `app/Entities/Repos/PageRepo.php` |
| 修订 Repository | `app/Entities/Repos/RevisionRepo.php` |
| 删除 Repository | `app/Entities/Repos/DeletionRepo.php` |

### 14.4 辅助工具类

| 功能 | 文件位置 |
|------|----------|
| 书籍内容树 | `app/Entities/Tools/BookContents.php` |
| 混合实体加载器 | `app/Entities/Tools/MixedEntityListLoader.php` |
| 垃圾桶工具类 | `app/Entities/Tools/TrashCan.php` |

### 14.5 权限相关

| 功能 | 文件位置 |
|------|----------|
| 权限应用器 | `app/Permissions/PermissionApplicator.php` |
| 联合权限构建器 | `app/Permissions/JointPermissionBuilder.php` |
| 实体权限评估器 | `app/Permissions/EntityPermissionEvaluator.php` |
| 批量权限评估器 | `app/Permissions/MassEntityPermissionEvaluator.php` |
| 权限枚举 | `app/Permissions/Permission.php` |
| 权限状态枚举 | `app/Permissions/PermissionStatus.php` |
| 实体权限模型 | `app/Permissions/Models/EntityPermission.php` |
| 联合权限模型 | `app/Permissions/Models/JointPermission.php` |

### 14.6 软删除相关

| 功能 | 文件位置 |
|------|----------|
| 删除记录模型 | `app/Entities/Models/Deletion.php` |
| 回收站控制器 | `app/Entities/Controllers/RecycleBinController.php` |
| 回收站 API 控制器 | `app/Entities/Controllers/RecycleBinApiController.php` |

### 14.7 版本管理相关

| 功能 | 文件位置 |
|------|----------|
| 页面修订模型 | `app/Entities/Models/PageRevision.php` |
| 修订查询类 | `app/Entities/Queries/PageRevisionQueries.php` |
| 修订控制器 | `app/Entities/Controllers/PageRevisionController.php` |

### 14.8 全文搜索相关

| 功能 | 文件位置 |
|------|----------|
| 搜索索引管理 | `app/Search/SearchIndex.php` |
| 搜索词项模型 | `app/Search/SearchTerm.php` |
| 搜索执行器 | `app/Search/SearchRunner.php` |
| 搜索控制器 | `app/Search/SearchController.php` |
| 搜索 API 控制器 | `app/Search/SearchApiController.php` |
| 搜索选项 | `app/Search/SearchOptions.php` |
| 文本分词器 | `app/Search/SearchTextTokenizer.php` |
| 搜索结果格式化 | `app/Search/SearchResultsFormatter.php` |

### 14.9 导出转换相关

| 功能 | 文件位置 |
|------|----------|
| 导出格式化器 | `app/Exports/ExportFormatter.php` |
| PDF 生成器 | `app/Exports/PdfGenerator.php` |
| 页面导出控制器 | `app/Exports/Controllers/PageExportController.php` |
| 章导出控制器 | `app/Exports/Controllers/ChapterExportController.php` |
| 册导出控制器 | `app/Exports/Controllers/BookExportController.php` |
| ZIP 导出构建器 | `app/Exports/ZipExports/ZipExportBuilder.php` |
| ZIP 导出文件处理 | `app/Exports/ZipExports/ZipExportFiles.php` |
| HTML→Markdown 转换 | `app/Entities/Tools/Markdown/HtmlToMarkdown.php` |
| HTML→纯文本转换 | `app/Util/HtmlToPlainText.php` |

### 14.10 Activity 审计相关

| 功能 | 文件位置 |
|------|----------|
| Activity 模型 | `app/Activity/Models/Activity.php` |
| Activity 类型枚举 | `app/Activity/ActivityType.php` |
| Activity 日志写入器 | `app/Activity/Tools/ActivityLogger.php` |
| Activity 查询类 | `app/Activity/ActivityQueries.php` |
| IP 格式化器 | `app/Activity/Tools/IpFormatter.php` |
| 通知管理器 | `app/Activity/Notifications/NotificationManager.php` |
| Webhook 分发任务 | `app/Activity/DispatchWebhookJob.php` |
| 审计日志控制器 | `app/Activity/Controllers/AuditLogController.php` |
| 审计日志 API | `app/Activity/Controllers/AuditLogApiController.php` |

---

## 十五、API 限流与 Throttle 链路

### 15.1 限流系统架构

```
HTTP 请求进入
    │
    ├─ Kernel $middlewareAliases 注册 throttle 别名
    │     └─ ThrottleRequests::class
    │
    ├─ RouteServiceProvider::configureRateLimiting()
    │     │
    │     ├─ RateLimiter::for('api', ...)        // API 路由
    │     ├─ RateLimiter::for('public', ...)     // 公开路由
    │     └─ RateLimiter::for('exports', ...)    // 导出路由
    │
    ├─ 路由 / 控制器 middleware 声明
    │     ├─ 'api' middlewareGroup → ThrottleApiRequests::class
    │     └─ 手动指定 → 'throttle:exports'
    │
    └─ 业务场景内的手动限流（trait）
          ├─ ThrottlesLogins trait（登录限流）
          └─ MfaVerificationLimiter（MFA 验证限流）
```

### 15.2 中间件注册与配置

#### 15.2.1 Kernel 中间件注册 (`app/Http/Kernel.php`)

```php
// 路由中间件组
protected $middlewareGroups = [
    'web' => [
        // ... CSP、Cookie、Session、CSRF、EmailCheck、Theme、Localization
    ],
    'api' => [
        \BookStack\Http\Middleware\ThrottleApiRequests::class,  // ① API 限流
        \BookStack\Http\Middleware\EncryptCookies::class,
        \BookStack\Http\Middleware\StartSessionIfCookieExists::class,
        \BookStack\Http\Middleware\ApiAuthenticate::class,
        \BookStack\Http\Middleware\CheckEmailConfirmed::class,
    ],
];

// 中间件别名（可在路由中通过字符串引用）
protected $middlewareAliases = [
    'auth'       => Authenticate::class,
    'can'        => CheckUserHasPermission::class,
    'throttle'   => \Illuminate\Routing\Middleware\ThrottleRequests::class,  // ② 通用限流
    'mfa-setup'  => AuthenticatedOrPendingMfa::class,
];
```

#### 15.2.2 RouteServiceProvider 限流配置

```php
// app/App/Providers/RouteServiceProvider.php:79-95
protected function configureRateLimiting(): void
{
    // API 路由限流：登录用户按 user_id，访客按 IP
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });

    // 公开路由限流：按 IP
    RateLimiter::for('public', function (Request $request) {
        return Limit::perMinute(10)->by($request->ip());
    });

    // 导出限流：访客 4次/分，登录用户 10次/分
    RateLimiter::for('exports', function (Request $request) {
        $user = user();
        $attempts = $user->isGuest() ? 4 : 10;
        $key = $user->isGuest() ? $request->ip() : $user->id;
        return Limit::perMinute($attempts)->by($key);
    });
}
```

### 15.3 三种限流配置对比

| 配置名 | 适用范围 | 速率限制 | 标识 Key | 应用方式 |
|-------|---------|---------|---------|---------|
| `api` | API 接口 | 60 次/分 | user_id 或 IP | api middlewareGroup |
| `public` | 公开路由 | 10 次/分 | IP | 路由手动指定 |
| `exports` | 导出功能 | 访客 4/登录 10 次/分 | user_id 或 IP | 控制器 middleware |

### 15.4 ThrottleApiRequests 自定义中间件

```php
// app/Http/Middleware/ThrottleApiRequests.php
class ThrottleApiRequests extends ThrottleRequests
{
    /**
     * 覆盖 resolveMaxAttempts，从配置文件 api.requests_per_minute 读取
     * 而不是从路由参数中解析，允许管理员自定义 API 限流速率
     */
    protected function resolveMaxAttempts($request, $maxAttempts): int
    {
        return (int) config('api.requests_per_minute');
    }
}
```

### 15.5 导出限流接入点

在三个导出控制器的构造函数中统一声明：

```php
class PageExportController extends Controller
{
    public function __construct(...)
    {
        $this->middleware(Permission::ContentExport->middleware());  // 权限
        $this->middleware('throttle:exports');  // ← 限流
    }
}
```

**应用范围**：
- `PageExportController` - 页面导出
- `ChapterExportController` - 章节导出
- `BookExportController` - 册导出

### 15.6 业务场景内手动限流

#### 15.6.1 登录限流 - ThrottlesLogins trait

```php
// app/Access/Controllers/ThrottlesLogins.php
trait ThrottlesLogins
{
    // 5 次尝试后锁定 1 分钟
    public function maxAttempts(): int { return 5; }
    public function decayMinutes(): int { return 1; }

    // 限流 Key = 小写用户名 | IP
    protected function throttleKey(Request $request): string
    {
        return Str::transliterate(Str::lower($request->input($this->username())) . '|' . $request->ip());
    }

    // 超过限制抛出 ValidationException (429)
    protected function sendLockoutResponse(Request $request): Response
    {
        throw ValidationException::withMessages([
            $this->username() => [trans('auth.throttle', [
                'seconds' => $seconds,
                'minutes' => ceil($seconds / 60),
            ])],
        ])->status(Response::HTTP_TOO_MANY_REQUESTS);
    }
}
```

#### 15.6.2 MFA 验证限流 - MfaVerificationLimiter

```php
// app/Access/Mfa/MfaVerificationLimiter.php
class MfaVerificationLimiter
{
    // 双层限流：按用户（严）+ 按 IP（宽）
    protected int $maxUserAttemptsPerMinute = 5;    // 单用户 5 次/分
    protected int $maxIpAttemptsPerMinute = 60;     // 单 IP 60 次/分

    public function hasHitLimit(User $user, Request $request): bool
    {
        return $this->rateLimiter->tooManyAttempts(
                   $this->getUserKey($user), $this->maxUserAttemptsPerMinute + 1
               )
            || $this->rateLimiter->tooManyAttempts(
                   $this->getRequestKey($request), $this->maxIpAttemptsPerMinute + 1
               );
    }

    // Key 命名规范
    protected function getUserKey(User $user): string
    {
        return "mfa-attempt::user::{$user->id}";
    }

    protected function getRequestKey(Request $request): string
    {
        return "mfa-attempt::request::{$request->ip()}";
    }
}
```

---

## 十六、Webhook 外部通知链路

### 16.1 Webhook 系统架构

```
Activity::add(type, entity)
    │
    └─ ActivityLogger::add()
          │
          └─ dispatchWebhooks(type, detail)
                │
                ├─ 查询匹配的 Webhook（trackedEvents 包含当前事件或 'all'）
                │
                └─ 循环 dispatch(new DispatchWebhookJob(...))  // 异步队列
                      │
                      └─ DispatchWebhookJob::handle()
                            │
                            ├─ SSRF 防护：SsrUrlValidator::ensureAllowed()
                            ├─ 主题事件钩子：ThemeEvents::WEBHOOK_CALL_BEFORE
                            ├─ WebhookFormatter 构造 payload
                            ├─ HttpClient::sendRequest() POST
                            └─ 更新 webhook 状态（last_called_at / last_error）
```

### 16.2 核心数据模型

#### 16.2.1 Webhook 模型 (`app/Activity/Models/Webhook.php`)

```php
class Webhook extends Model implements Loggable
{
    protected $fillable = ['name', 'endpoint', 'timeout'];

    // 一对多关联到跟踪的事件
    public function trackedEvents(): HasMany
    {
        return $this->hasMany(WebhookTrackedEvent::class);
    }

    // 更新跟踪事件（全量替换）
    public function updateTrackedEvents(array $events): void
    {
        $this->trackedEvents()->delete();
        $eventsToStore = array_intersect($events, array_values(ActivityType::all()));
        if (in_array('all', $events)) {
            $eventsToStore = ['all'];  // 'all' 跟踪所有事件
        }
        ...
    }

    // 检查是否跟踪某事件
    public function tracksEvent(string $event): bool
    {
        return $this->trackedEvents->pluck('event')->contains($event);
    }
}
```

**Webhook 表结构**：

```
┌─────────────────────────────────────────────────────┐
│                    webhooks                      │
├──────────────────┬──────────────────────────────┤
│ id               │ 主键                         │
│ name             │ Webhook 名称                 │
│ endpoint         │ 回调 URL                     │
│ active           │ 是否启用                     │
│ timeout          │ 超时时间（秒）               │
│ last_called_at   │ 最后调用时间                 │
│ last_errored_at  │ 最后错误时间                 │
│ last_error       │ 最后错误消息                 │
│ created_at       │ 创建时间                     │
│ updated_at       │ 更新时间                     │
└──────────────────┴──────────────────────────────┘
```

**WebhookTrackedEvent 表结构**：

```
┌─────────────────────────────────────────────────────┐
│              webhook_tracked_events             │
├──────────────────┬──────────────────────────────┤
│ id               │ 主键                         │
│ webhook_id       │ 所属 Webhook ID              │
│ event            │ 跟踪的事件类型               │
└──────────────────┴──────────────────────────────┘
```

### 16.3 Webhook 查找与分发

```php
// ActivityLogger.php:85-98
protected function dispatchWebhooks(string $type, string|Loggable $detail): void
{
    // 查询所有跟踪此事件的活跃 Webhook
    $webhooks = Webhook::query()
        ->whereHas('trackedEvents', function (Builder $query) use ($type) {
            $query->where('event', '=', $type)
                ->orWhere('event', '=', 'all');  // 'all' 表示跟踪所有
        })
        ->where('active', '=', true)
        ->get();

    // 每个 Webhook 分发一个异步队列任务
    foreach ($webhooks as $webhook) {
        dispatch(new DispatchWebhookJob($webhook, $type, $detail));
    }
}
```

### 16.4 DispatchWebhookJob 异步任务

```php
// app/Activity/DispatchWebhookJob.php
class DispatchWebhookJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    // 构造时即构造 payload（队列序列化前完成）
    public function __construct(Webhook $webhook, string $event, Loggable|string $detail)
    {
        $this->webhook = $webhook;
        $this->initiator = user();
        $this->initiatedTime = time();

        // 允许主题系统通过 WEBHOOK_CALL_BEFORE 事件修改 payload
        $themeResponse = Theme::dispatch(
            ThemeEvents::WEBHOOK_CALL_BEFORE,
            $event, $this->webhook, $detail,
            $this->initiator, $this->initiatedTime
        );

        $this->webhookData = $themeResponse ??
            WebhookFormatter::getDefault($event, $this->webhook, $detail, $this->initiator, $this->initiatedTime)
                ->format();
    }

    public function handle(HttpRequestService $http)
    {
        try {
            // ① SSRF 防护：禁止访问内网地址（127.0.0.1、192.168.x.x 等）
            (new SsrUrlValidator())->ensureAllowed($this->webhook->endpoint);

            // ② 构建 HTTP 客户端
            $client = $http->buildClient($this->webhook->timeout, [
                'connect_timeout' => 10,
                'allow_redirects' => ['strict' => true],
            ]);

            // ③ 发送 POST JSON 请求
            $response = $client->sendRequest(
                $http->jsonRequest('POST', $this->webhook->endpoint, $this->webhookData)
            );
            $statusCode = $response->getStatusCode();

            // ④ 4xx/5xx 记录为错误
            if ($statusCode >= 400) {
                $lastError = "Response status from endpoint was {$statusCode}";
                Log::error("Webhook call failed with status {$statusCode}");
            }
        } catch (\Exception $error) {
            $lastError = $error->getMessage();
            Log::error("Webhook call failed with error \"{$lastError}\"");
        }

        // ⑤ 更新 Webhook 状态（成功/失败时间、错误信息）
        $this->webhook->last_called_at = now();
        if ($lastError) {
            $this->webhook->last_errored_at = now();
            $this->webhook->last_error = $lastError;
        }
        $this->webhook->save();
    }
}
```

### 16.5 WebhookFormatter - Payload 构造

**默认 Payload 结构**：

```php
// app/Activity/Tools/WebhookFormatter.php
public function format(): array
{
    $data = [
        'event'                    => 'page_create',          // 事件类型
        'text'                     => 'Admin 创建了 "My Page"',// 人类可读描述
        'triggered_at'             => '2025-01-15T10:00:00Z',// ISO 时间
        'triggered_by'             => [...],                  // 触发者用户对象
        'triggered_by_profile_url' => '/user/1/admin',       // 用户主页
        'webhook_id'               => 1,                     // Webhook ID
        'webhook_name'             => 'My Hook',             // Webhook 名称
        'url'                      => '/books/1/page/2',     // 相关实体 URL
        'related_item'            => [...],                  // 相关实体详情
    ];
}
```

**Model 智能格式化器（条件化预加载）**：

```php
public function addDefaultModelFormatters(): void
{
    // 对所有 Entity 类型，预加载 用户信息（创建者/更新者/拥有者）
    $this->addModelFormatter(
        fn ($event, $model) => ($model instanceof Entity),
        fn ($model) => $model->load(['ownedBy', 'createdBy', 'updatedBy'])
    );

    // 对 Page 创建/更新事件，预加载当前版本详情
    $this->addModelFormatter(
        fn ($event, $model) => ($model instanceof Page && in_array($event, [
            ActivityType::PAGE_CREATE, ActivityType::PAGE_UPDATE
        ])),
        fn ($model) => $model->load('currentRevision')
    );
}
```

### 16.6 Webhook 安全机制

| 机制 | 实现 | 作用 |
|-----|------|------|
| SSRF 防护 | `SsrUrlValidator::ensureAllowed()` | 禁止回调到内网/本地 IP |
| 严格重定向 | `allow_redirects: ['strict' => true]` | 避免开放重定向滥用 |
| 超时控制 | `timeout` + `connect_timeout: 10` | 防止长连接阻塞队列 |
| 配置项 | 管理员可设置 active=false | 快速停用有问题的 Webhook |
| 错误追踪 | `last_error`, `last_errored_at` | 便于排查问题 |

---

## 十七、批量任务调度与 Job 队列链路

### 17.1 队列系统架构

```
┌───────────────────────────────────────────────────────┐
│                    任务调度体系                       │
├───────────────────────────────────────────────────────┤
│                                                       │
│  ① Console 命令（手工/CRON 触发）                     │
│     php artisan bookstack:xxx                         │
│                                                       │
│  ② Queue 队列任务（ShouldQueue 接口）                 │
│     dispatch(new SomeJob()) → 队列 → queue:work       │
│                                                       │
│  ③ Schedule 调度器（Console Kernel::schedule）        │
│     cron → php artisan schedule:run                   │
│                                                       │
└───────────────────────────────────────────────────────┘
```

### 17.2 Console 命令清单（21 个 Artisan 命令）

所有命令位于 `app/Console/Commands/`，命名空间 `bookstack:`：

| 命令 | 功能分类 | 说明 |
|------|---------|------|
| `bookstack:regenerate-search` | 搜索 | 重建全文搜索索引（每批 250 条，带进度回调） |
| `bookstack:regenerate-permissions` | 权限 | 重建所有联合权限（joint_permissions） |
| `bookstack:regenerate-references` | 引用 | 重建所有实体间的交叉引用 |
| `bookstack:clear-activity` | 审计 | **清空**所有活动日志（破坏性操作） |
| `bookstack:clear-revisions` | 版本 | 清理旧版本修订 |
| `bookstack:cleanup-images` | 存储 | 清理未使用图片（支持 --all --force） |
| `bookstack:clear-views` | 其他 | 清理访问记录 |
| `bookstack:update-url` | 配置 | 批量替换数据库中的旧 URL |
| `bookstack:upgrade-database-encoding` | 数据库 | 升级数据库字符编码 |
| `bookstack:create-admin` | 用户 | 创建管理员账号 |
| `bookstack:delete-users` | 用户 | 批量删除用户 |
| `bookstack:reset-mfa` | 用户 | 重置用户 MFA 设置 |
| `bookstack:refresh-avatar` | 用户 | 刷新用户头像 |
| `bookstack:copy-shelf-permissions` | 权限 | 复制书架权限到其下所有册 |
| `bookstack:assign-sort-rule` | 排序 | 分配排序规则到指定册 |
| `bookstack:install-module` | 扩展 | 安装主题模块 |
| `bookstack:copy-shelf-permissions` | 权限 | 复制书架权限 |

**典型命令实现 - RegenerateSearchCommand**：

```php
class RegenerateSearchCommand extends Command
{
    protected $signature = 'bookstack:regenerate-search
                            {--database= : 使用的数据库连接}';

    public function handle(SearchIndex $searchIndex): int
    {
        // 支持切换数据库连接
        if ($this->option('database') !== null) {
            DB::setDefaultConnection($this->option('database'));
        }

        // 调用 SearchIndex，传入进度报告回调
        $searchIndex->indexAllEntities(function (Entity $model, int $processed, int $total): void {
            $this->info('Indexed ' . class_basename($model) . " entries ({$processed}/{$total})");
        });

        $this->line('Search index regenerated!');
        return static::SUCCESS;
    }
}
```

### 17.3 Schedule 调度器

```php
// app/Console/Kernel.php
class Kernel extends ConsoleKernel
{
    protected function schedule(Schedule $schedule)
    {
        // 当前为空，未配置定时任务
        // 典型配置示例（可扩展）：
        // $schedule->command('bookstack:cleanup-images --force')->daily();
        // $schedule->command('bookstack:clear-views')->weekly();
    }

    protected function commands()
    {
        $this->load(__DIR__ . '/Commands');  // 自动加载所有 Commands
    }
}
```

### 17.4 Queue 队列任务

#### 17.4.1 唯一队列任务：DispatchWebhookJob

```php
class DispatchWebhookJob implements ShouldQueue
{
    // 核心 Trait 组合
    use Dispatchable;       // 支持 dispatch() 静态调用
    use InteractsWithQueue; // 可与队列交互（release()、delete() 等）
    use Queueable;          // 队列配置（onQueue、delay 等）
    use SerializesModels;   // 自动序列化/反序列化 Eloquent 模型

    // 注意：构造函数在 dispatch 时同步执行，handle 在队列 worker 中异步执行
    public function __construct(Webhook $webhook, string $event, ...) { ... }
    public function handle(HttpRequestService $http) { ... }
}
```

#### 17.4.2 其他异步通知（Notification）

```
User::notify() / Notification::send()
    │
    ├─ 站内通知：DatabaseChannel（写入 notifications 表）
    │
    ├─ 邮件通知：MailChannel
    │     └─ MailNotification → 支持 locale 切换
    │
    └─ 支持 ShouldQueue 接口时：异步队列执行
```

### 17.5 队列配置说明

通过 Laravel 标准 `config/queue.php` 配置，支持驱动：
- `sync`（默认，同步执行，开发环境）
- `database`（数据库队列）
- `redis`（Redis 队列）
- `beanstalkd`、`sqs` 等

**DispatchWebhookJob 是项目中唯一显式实现 ShouldQueue 的 Job 类**，其他异步处理通过 Notification 系统完成。

---

## 十八、Locale 多语言链路

### 18.1 多语言系统架构

```
HTTP 请求进入
    │
    └─ web middlewareGroup: Localization::handle()
          │
          ├─ ① LocaleManager::getForUser(user())
          │     │
          │     ├─ 登录用户 → setting()->getUser(user, 'language', default)
          │     └─ 访客用户
          │           ├─ auto_detect_locale=true → Accept-Language header 匹配
          │           └─ auto_detect_locale=false → config('app.default_locale')
          │
          ├─ ② LocaleDefinition 封装（appName + isoName + isRtl）
          │
          ├─ ③ view()->share('locale', $userLocale)   // 视图共享
          │
          └─ ④ app()->setLocale($userLocale->appLocale())  // 设置 Laravel 翻译器
                │
                └─ Translator 使用 FileLoader 加载翻译
                      │
                      ├─ ① 原始翻译（resources/lang/）
                      ├─ ② 模块翻译（Modules/*/lang/）
                      └─ ③ 主题覆盖（theme_path('lang')/）
```

### 18.2 Localization 中间件

```php
// app/Http/Middleware/Localization.php
class Localization
{
    public function handle($request, Closure $next)
    {
        // ① 获取用户 Locale 定义
        $userLocale = $this->localeManager->getForUser(user());

        // ② 共享到所有视图（用于 HTML lang/dir 属性等）
        view()->share('locale', $userLocale);

        // ③ 设置应用 Locale（Laravel 翻译器、Carbon 等）
        app()->setLocale($userLocale->appLocale());

        return $next($request);
    }
}
```

### 18.3 LocaleManager - 核心管理器

```php
// app/Translation/LocaleManager.php
class LocaleManager
{
    // RTL（右到左）语言列表
    protected array $rtlLocales = ['ar', 'fa', 'he'];

    // BookStack Locale → ISO Locale 映射（76 种语言）
    protected array $localeMap = [
        'en'          => 'en_GB',
        'zh_CN'       => 'zh_CN',
        'zh_TW'       => 'zh_TW',
        'de_informal' => 'de_DE',     // 特殊：非正式德语也用 de_DE
        'ar'          => 'ar',
        // ... 约 70 种语言映射
    ];

    /**
     * 为用户解析 Locale 字符串
     */
    protected function getLocaleForUser(User $user): string
    {
        $default = config('app.default_locale');

        // 访客 + 开启自动检测 → 从 Accept-Language 解析
        if ($user->isGuest() && config('app.auto_detect_locale')) {
            return $this->autoDetectLocale(request(), $default);
        }

        // 其他：用户设置 > 系统默认
        return setting()->getUser($user, 'language', $default);
    }

    /**
     * 从 HTTP Accept-Language 头自动匹配支持的语言
     */
    protected function autoDetectLocale(Request $request, string $default): string
    {
        $availableLocales = $this->getAllAppLocales();

        // 按浏览器偏好顺序逐一匹配
        foreach ($request->getLanguages() as $lang) {
            if (in_array($lang, $availableLocales)) {
                return $lang;
            }
        }

        return $default;
    }

    /**
     * 返回完整 Locale 定义（三重命名空间）
     */
    public function getForUser(User $user): LocaleDefinition
    {
        $localeString = $this->getLocaleForUser($user);

        return new LocaleDefinition(
            $localeString,                           // BookStack 内部名：zh_CN
            $this->localeMap[$localeString] ?? $localeString,  // ISO 名：zh_CN
            in_array($localeString, $this->rtlLocales),  // 是否 RTL：false
        );
    }
}
```

### 18.4 LocaleDefinition - 封装对象

```php
// app/Translation/LocaleDefinition.php
class LocaleDefinition
{
    public function __construct(
        protected string $appName,   // 内部名（如 zh_CN、de_informal）
        protected string $isoName,   // ISO 标准名（如 zh_CN、de_DE）
        protected bool $isRtl        // 是否为 RTL 语言
    ) {}

    // Laravel 翻译器使用
    public function appLocale(): string { return $this->appName; }

    // 系统级操作使用（如 setlocale()）
    public function isoLocale(): string { return $this->isoName; }

    // HTML lang 属性（zh-CN）
    public function htmlLang(): string { return str_replace('_', '-', $this->isoName); }

    // HTML dir 属性（ltr / rtl）
    public function htmlDirection(): string { return $this->isRtl ? 'rtl' : 'ltr'; }

    // 按此 Locale 翻译（临时切换）
    public function trans(string $key, array $replace = []): string
    {
        return trans($key, $replace, $this->appLocale());
    }
}
```

### 18.5 翻译文件加载器 - FileLoader

```php
// app/Translation/FileLoader.php
class FileLoader extends BaseLoader
{
    /**
     * 覆盖 Laravel 默认加载器，支持三层翻译覆盖
     */
    public function load($locale, $group, $namespace = null): array
    {
        if (is_null($namespace) || $namespace === '*') {
            // ① 主题翻译（优先级最高，可覆盖系统默认）
            $themePath = theme_path('lang');
            $themeTranslations = $themePath ?
                $this->loadPaths([$themePath], $locale, $group) : [];

            // ② 模块翻译（安装的扩展模块）
            $modules = Theme::getModules();
            $moduleTranslations = [];
            foreach ($modules as $module) {
                $modulePath = $module->path('lang');
                if (file_exists($modulePath)) {
                    $moduleTranslations = array_merge(
                        $moduleTranslations,
                        $this->loadPaths([$modulePath], $locale, $group)
                    );
                }
            }

            // ③ 原始系统翻译（resources/lang/）
            $originalTranslations = $this->loadPaths($this->paths, $locale, $group);

            // 合并（后面的覆盖前面的）
            return array_merge($originalTranslations, $moduleTranslations, $themeTranslations);
        }

        return $this->loadNamespaced($locale, $group, $namespace);
    }
}
```

**三层翻译覆盖优先级**：

```
主题目录 lang/  (theme_path)
    ↑ 覆盖
模块目录 lang/  (Modules/Xxx/lang)
    ↑ 覆盖
系统默认 lang/  (resources/lang)
```

### 18.6 MessageSelector - 复数支持扩展

```php
// app/Translation/MessageSelector.php
/**
 * 解决非标准 Locale（如 de_informal）的复数匹配问题
 * 取 Locale 的第一部分（下划线前）作为复数判断依据
 */
class MessageSelector extends BaseClass
{
    public function getPluralIndex($locale, $number)
    {
        $locale = explode('_', $locale)[0];  // de_informal → de
        return parent::getPluralIndex($locale, $number);
    }
}
```

### 18.7 多语言接入点汇总

| 场景 | 使用方式 | 位置 |
|------|---------|------|
| Web 请求自动设置 | Localization middleware | `app/Http/Kernel.php:38` |
| 视图中使用 | `$locale->htmlLang()`, `$locale->htmlDirection()` | Blade 模板 |
| PHP 代码翻译 | `trans('key')` / `__('key')` | 全局 |
| 邮件通知翻译 | `$notifiable->getLocale()` | 各类 Notification |
| 导出 PDF/HTML | `user()->getLocale()` | ExportFormatter:41,63,82... |
| 用户模型便捷获取 | `User::getLocale(): LocaleDefinition` | User.php:335 |

### 18.8 特殊场景：邮件通知中的 Locale 处理

邮件通知可能在队列中异步发送（请求上下文不存在），需要在通知构造时锁定接收者的 Locale：

```php
// 通知构造方法示例
public function __construct(...)
{
    // 构造时即保存接收者的 Locale，防止队列中 user() 上下文丢失
    $locale = $notifiable->getLocale();
    ...
}
```

---

## 十九、完整代码位置速查表（最终版）

### 19.1 API 限流与 Throttle

| 功能 | 文件位置 |
|------|----------|
| Kernel 中间件注册 | `app/Http/Kernel.php` |
| 路由服务提供商 | `app/App/Providers/RouteServiceProvider.php` |
| API 请求限流中间件 | `app/Http/Middleware/ThrottleApiRequests.php` |
| 登录限流 Trait | `app/Access/Controllers/ThrottlesLogins.php` |
| MFA 验证限流 | `app/Access/Mfa/MfaVerificationLimiter.php` |

### 19.2 Webhook 外部通知

| 功能 | 文件位置 |
|------|----------|
| Webhook 模型 | `app/Activity/Models/Webhook.php` |
| 跟踪事件模型 | `app/Activity/Models/WebhookTrackedEvent.php` |
| Webhook 分发任务 | `app/Activity/DispatchWebhookJob.php` |
| Webhook Payload 格式化 | `app/Activity/Tools/WebhookFormatter.php` |
| SSRF URL 验证器 | `app/Util/SsrUrlValidator.php` |
| Webhook 控制器 | `app/Activity/Controllers/WebhookController.php` |

### 19.3 批量任务与队列调度

| 功能 | 文件位置 |
|------|----------|
| Console 调度内核 | `app/Console/Kernel.php` |
| 重建搜索索引命令 | `app/Console/Commands/RegenerateSearchCommand.php` |
| 重建权限命令 | `app/Console/Commands/RegeneratePermissionsCommand.php` |
| 重建引用命令 | `app/Console/Commands/RegenerateReferencesCommand.php` |
| 清理活动日志命令 | `app/Console/Commands/ClearActivityCommand.php` |
| 清理旧版本命令 | `app/Console/Commands/ClearRevisionsCommand.php` |
| 清理未使用图片命令 | `app/Console/Commands/CleanupImagesCommand.php` |
| URL 批量更新命令 | `app/Console/Commands/UpdateUrlCommand.php` |
| 创建管理员命令 | `app/Console/Commands/CreateAdminCommand.php` |
| 删除用户命令 | `app/Console/Commands/DeleteUsersCommand.php` |
| 重置 MFA 命令 | `app/Console/Commands/ResetMfaCommand.php` |
| 队列 HTTP 服务 | `app/Http/HttpRequestService.php` |

### 19.4 Locale 多语言

| 功能 | 文件位置 |
|------|----------|
| Locale 中间件 | `app/Http/Middleware/Localization.php` |
| Locale 管理器 | `app/Translation/LocaleManager.php` |
| Locale 定义对象 | `app/Translation/LocaleDefinition.php` |
| 翻译文件加载器（三层覆盖） | `app/Translation/FileLoader.php` |
| 复数选择器（非标准 Locale） | `app/Translation/MessageSelector.php` |
| 翻译服务提供商 | `app/App/Providers/TranslationServiceProvider.php` |
| Locale 配置 | `config/app.php` |

---

## 二十、SAML 与 OIDC 外部认证集成

### 20.1 外部认证架构总览

```
┌───────────────────────────────────────────────────────────┐
│              外部认证体系                                   │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  ① SocialAuth（OAuth 社交登录）                           │
│     Google / GitHub / Facebook / Twitter / Azure          │
│     Okta / GitLab / Slack / Twitch / Discord              │
│                                                           │
│  ② SAML 2.0 企业认证                                      │
│     基于 onelogin/php-saml 库                             │
│                                                           │
│  ③ OpenID Connect (OIDC)                                  │
│     基于 league/oauth2-client + 自定义 JWT 验证           │
│                                                           │
└───────────────────────────────────────────────────────────┘
      │
      ▼
  RegistrationService::findOrRegister()  // 查找或创建用户
      │
      └─ LoginService::login()  // 统一登录入口
            │
            ├─ MFA 检查（needsMfaVerification）
            ├─ 邮箱确认检查（awaitingEmailConfirmation）
            └─ Activity::add(AUTH_LOGIN)
```

### 20.2 SocialAuth 社交认证

#### 20.2.1 SocialDriverManager - 驱动管理器

```php
// app/Access/SocialDriverManager.php
class SocialDriverManager
{
    // 内置支持 10 种驱动
    protected array $validDrivers = [
        'google', 'github', 'facebook', 'slack', 'twitter',
        'azure', 'okta', 'gitlab', 'twitch', 'discord',
    ];

    // 检查驱动是否配置（client_id + client_secret + callback_url）
    protected function checkDriverConfigured(string $driver): bool { ... }

    // 每个驱动的配置选项
    public function isAutoRegisterEnabled(string $driver): bool { ... }
    public function isAutoConfirmEmailEnabled(string $driver): bool { ... }

    // 支持通过 addSocialDriver() 动态添加自定义驱动
    // 常用于主题/模块扩展
    public function addSocialDriver(string $driverName, array $config, ...) { ... }
}
```

#### 20.2.2 SocialController - 社交认证控制器

```php
// app/Access/Controllers/SocialController.php

// 登录入口 → 重定向到认证服务
public function login(string $socialDriver)
{
    session()->put('social-callback', 'login');
    return $this->socialAuthService->startLogIn($socialDriver);
}

// 注册入口 → 重定向到认证服务
public function register(string $socialDriver)
{
    $this->registrationService->ensureRegistrationAllowed();
    session()->put('social-callback', 'register');
    return $this->socialAuthService->startRegister($socialDriver);
}

// 回调处理
public function callback(Request $request, string $socialDriver)
{
    $action = session()->pull('social-callback');
    $socialUser = $this->socialAuthService->getSocialUser($socialDriver);

    if ($action === 'login') {
        try {
            return $this->socialAuthService->handleLoginCallback($socialDriver, $socialUser);
        } catch (SocialSignInAccountNotUsed $exception) {
            // 如果开启自动注册，失败后自动走注册流程
            if ($this->socialAuthService->drivers()->isAutoRegisterEnabled($socialDriver)) {
                return $this->socialRegisterCallback($socialDriver, $socialUser);
            }
            throw $exception;
        }
    }
}
```

#### 20.2.3 SocialAuthService - 核心服务

**登录回调的 5 种场景处理**：

| 场景 | 登录状态 | SocialAccount 存在 | 用户匹配 | 处理 |
|------|---------|-------------------|---------|------|
| ① | 未登录 | ✅ | - | 直接登录 |
| ② | 已登录 | ❌ | - | 绑定到当前用户 |
| ③ | 已登录 | ✅ | 当前用户 | 提示已绑定 |
| ④ | 已登录 | ✅ | 其他用户 | 报错：已被其他账号绑定 |
| ⑤ | 未登录 | ❌ | - | 报错：无对应账号（可配置自动注册） |

```php
// app/Access/SocialAuthService.php
public function handleLoginCallback(string $socialDriver, SocialUser $socialUser)
{
    $socialAccount = SocialAccount::where('driver_id', '=', $socialUser->getId())->first();
    $isLoggedIn = auth()->check();

    // 场景 ①：未登录 + 账号存在 → 直接登录
    if (!$isLoggedIn && $socialAccount !== null) {
        $this->loginService->login($socialAccount->user, $socialDriver);
        return redirect()->intended('/');
    }

    // 场景 ②：已登录 + 账号不存在 → 绑定
    if ($isLoggedIn && $socialAccount === null) {
        $account = $this->newSocialAccount($socialDriver, $socialUser);
        $currentUser->socialAccounts()->save($account);
        return redirect('/my-account/auth#social_accounts');
    }

    // ... 其他场景
}
```

### 20.3 SAML 2.0 认证

#### 20.3.1 Saml2Service - SAML 服务

```php
// app/Access/Saml2Service.php
class Saml2Service
{
    // 基于 onelogin/php-saml 库实现
    // 支持：SP 元数据、SSO、SLO、ACS、SLS

    /**
     * 发起登录：返回 IdP 登录 URL
     */
    public function login(): array
    {
        $toolKit = $this->getToolkit();
        return [
            'url' => $toolKit->login($returnRoute, [], false, false, true),
            'id'  => $toolKit->getLastRequestID(),
        ];
    }

    /**
     * 处理 ACS 响应（IdP 回调）
     */
    public function processAcsResponse(?string $requestId, string $samlResponse): ?User
    {
        $_POST['SAMLResponse'] = $samlResponse;
        $toolkit = $this->getToolkit();
        $toolkit->processResponse($requestId);

        if (!$toolkit->isAuthenticated()) {
            return null;
        }

        $attrs = $toolkit->getAttributes();
        $id = $toolkit->getNameId();
        session()->put('saml2_session_index', $toolkit->getSessionIndex());

        return $this->processLoginCallback($id, $attrs);
    }

    /**
     * 处理登出（支持 SLO 单点登出）
     */
    public function logout(User $user): array { ... }

    /**
     * 生成 SP 元数据 XML
     */
    public function metadata(): string { ... }
}
```

#### 20.3.2 SAML 用户属性映射

```php
// 从 SAML 响应提取用户信息
protected function getUserDetails(string $samlID, $samlAttributes): array
{
    // external_id_attribute 配置项 → 外部唯一 ID
    // display_name_attributes 配置项 → 显示名（可多属性拼接）
    // email_attribute 配置项 → 邮箱
    // group_attribute 配置项 → 用户组（用于角色同步）

    return [
        'external_id' => $externalId,
        'name'        => $displayName,
        'email'       => $email,
        'saml_id'     => $samlID,
    ];
}
```

#### 20.3.3 SAML 配置项

| 配置项 | 说明 |
|-------|------|
| `saml2.onelogin.*` | onelogin/php-saml 库的原生配置 |
| `saml2.autoload_from_metadata` | 自动从 IdP 元数据 URL 加载配置 |
| `saml2.external_id_attribute` | 外部 ID 字段映射 |
| `saml2.display_name_attributes` | 显示名字段（数组，可拼接） |
| `saml2.email_attribute` | 邮箱字段映射 |
| `saml2.group_attribute` | 用户组字段映射 |
| `saml2.user_to_groups` | 是否启用组同步 |
| `saml2.remove_from_groups` | 是否将用户从不存在的组中移除 |
| `saml2.dump_user_details` | 调试模式：输出用户详情 |

### 20.4 OpenID Connect (OIDC) 认证

#### 20.4.1 OidcService - OIDC 服务

```php
// app/Access/Oidc/OidcService.php
class OidcService
{
    /**
     * 发起授权请求
     * @return array{url: string, state: string}
     */
    public function login(): array
    {
        $settings = $this->getProviderSettings();
        $provider = $this->getProvider($settings);

        $url = $provider->getAuthorizationUrl();
        session()->put('oidc_pkce_code', $provider->getPkceCode() ?? '');

        // 主题钩子：允许修改重定向 URL
        $returnUrl = Theme::dispatch(ThemeEvents::OIDC_AUTH_PRE_REDIRECT, $url);

        return ['url' => $url, 'state' => $provider->getState()];
    }

    /**
     * 处理授权回调
     */
    public function processAuthorizeResponse(?string $authorizationCode): User
    {
        $settings = $this->getProviderSettings();
        $provider = $this->getProvider($settings);

        // 验证 PKCE
        $pkceCode = session()->pull('oidc_pkce_code', '');
        $provider->setPkceCode($pkceCode);

        // 用授权码换 Access Token
        $accessToken = $provider->getAccessToken('authorization_code', [
            'code' => $authorizationCode,
        ]);

        return $this->processAccessTokenCallback($accessToken, $settings);
    }
}
```

#### 20.4.2 ID Token 验证流程

```php
protected function processAccessTokenCallback(OidcAccessToken $accessToken, ...): User
{
    // 1. 解析 ID Token（JWT）
    $idToken = new OidcIdToken($idTokenText, $settings->issuer, $settings->keys);

    // 2. 主题钩子：允许修改 claims
    $returnClaims = Theme::dispatch(ThemeEvents::OIDC_ID_TOKEN_PRE_VALIDATE, ...);
    if (!is_null($returnClaims)) {
        $idToken->replaceClaims($returnClaims);
    }

    // 3. 调试模式
    if ($this->config()['dump_user_details']) {
        throw new JsonDebugException($idToken->getAllClaims());
    }

    // 4. 验证 Token（issuer、audience、签名、过期时间等）
    $idToken->validate($settings->clientId);

    // 5. 提取用户详情（从 ID Token + userinfo endpoint）
    $userDetails = $this->getUserDetailsFromToken($idToken, $accessToken, $settings);

    // 6. 查找或注册用户
    $user = $this->registrationService->findOrRegister(
        $userDetails->name, $userDetails->email, $userDetails->externalId
    );

    // 7. 可选：头像同步
    if ($this->config()['fetch_avatar'] && !$user->avatar()->exists() && $userDetails->picture) {
        $this->userAvatars->assignToUserFromUrl($user, $userDetails->picture);
    }

    // 8. 可选：组同步
    if ($this->shouldSyncGroups()) {
        $this->groupService->syncUserWithFoundGroups($user, $userDetails->groups ?? [], ...);
    }

    // 9. 登录
    $this->loginService->login($user, 'oidc');

    return $user;
}
```

#### 20.4.3 OIDC 配置项

| 配置项 | 说明 |
|-------|------|
| `oidc.client_id` | 客户端 ID |
| `oidc.client_secret` | 客户端密钥 |
| `oidc.issuer` | Issuer URL |
| `oidc.discover` | 是否启用 OIDC Discovery 自动发现 |
| `oidc.authorization_endpoint` | 授权端点 |
| `oidc.token_endpoint` | Token 端点 |
| `oidc.userinfo_endpoint` | Userinfo 端点 |
| `oidc.end_session_endpoint` | 登出端点（RP-initiated logout） |
| `oidc.jwt_public_key` | JWT 签名公钥 |
| `oidc.additional_scopes` | 额外 scope（逗号分隔） |
| `oidc.external_id_claim` | 外部 ID claim 名 |
| `oidc.display_name_claims` | 显示名 claim |
| `oidc.groups_claim` | 用户组 claim |
| `oidc.user_to_groups` | 是否启用组同步 |
| `oidc.remove_from_groups` | 是否移除不在组中的用户角色 |
| `oidc.fetch_avatar` | 是否同步头像 |
| `oidc.dump_user_details` | 调试模式 |

### 20.5 统一用户注册流程

```php
// RegistrationService::findOrRegister()
public function findOrRegister(string $name, string $email, string $externalId): User
{
    // 1. 按 external_auth_id 查找用户
    $user = User::query()->where('external_auth_id', '=', $externalId)->first();

    if (is_null($user)) {
        $userData = [
            'name'             => $name,
            'email'            => $email,
            'password'         => Str::random(32),  // 随机密码，外部认证用户不使用
            'external_auth_id' => $externalId,
        ];
        $user = $this->registerUser($userData, null, false);
    }
    return $user;
}
```

### 20.6 GroupSyncService - 组同步

SAML/OIDC 都支持用户组与系统角色的同步：

```php
// app/Access/GroupSyncService.php
class GroupSyncService
{
    /**
     * 将外部组映射到内部角色
     */
    public function syncUserWithFoundGroups(User $user, array $externalGroups, bool $detachExisting): void
    {
        $matchedRoleIds = $this->matchGroupsToRoles($externalGroups);

        if ($detachExisting) {
            // 全量同步：只保留匹配的角色
            $user->roles()->sync($matchedRoleIds);
        } else {
            // 增量同步：添加匹配的角色
            $user->roles()->attach($matchedRoleIds);
        }

        // 同步默认角色（如果用户没有任何角色）
        if ($user->roles()->count() === 0) {
            $user->attachDefaultRole();
        }
    }
}
```

---

## 二十一、API Token 生命周期

### 21.1 API Token 架构

```
API 请求进入
    │
    └─ api middlewareGroup: ApiAuthenticate::handle()
          │
          ├─ 有 Session 会话？
          │     ├─ 是 → 检查权限 + 只允许 GET 请求（便捷浏览）
          │     └─ 否 → 切换到 api guard → ApiTokenGuard::authenticate()
          │
          └─ ApiTokenGuard
                │
                ├─ ① 解析 Authorization: Token {id}:{secret}
                ├─ ② 按 token_id 查询 ApiToken
                ├─ ③ Hash::check(secret, token.secret)
                ├─ ④ 检查 expires_at 是否已过期
                ├─ ⑤ 检查用户是否有 AccessApi 权限
                └─ ⑥ 检查邮箱是否已确认
```

### 21.2 ApiToken 模型

```php
// app/Api/ApiToken.php
class ApiToken extends Model implements Loggable
{
    protected $fillable = ['name', 'expires_at'];
    protected $casts = ['expires_at' => 'date:Y-m-d'];

    // 所属用户
    public function user(): BelongsTo { ... }

    // 默认过期时间：100 年后（即永不过期）
    public static function defaultExpiry(): string
    {
        return Carbon::now()->addYears(100)->format('Y-m-d');
    }
}
```

**api_tokens 表结构**：

```
┌─────────────────────────────────────────────────────┐
│                  api_tokens                      │
├──────────────────┬──────────────────────────────┤
│ id               │ 主键                         │
│ user_id          │ 所属用户ID                   │
│ token_id         │ 公开 ID（用于查询）         │
│ secret           │ 密钥哈希（Hash::make）       │
│ name             │ Token 名称（用户自定义）     │
│ expires_at       │ 过期时间                     │
│ created_at       │ 创建时间                     │
│ updated_at       │ 更新时间                     │
└──────────────────┴──────────────────────────────┘
```

### 21.3 Token 创建流程

```php
// UserApiTokenController::store()
public function store(Request $request, int $userId)
{
    // 1. 验证权限
    $this->checkPermission(Permission::AccessApi);

    // 2. 生成随机 ID 和密钥
    $secret = Str::random(32);
    $token = (new ApiToken())->forceFill([
        'name'       => $request->input('name'),
        'token_id'   => Str::random(32),  // 32 字符 ID
        'secret'     => Hash::make($secret),  // 哈希存储
        'user_id'    => $user->id,
        'expires_at' => $request->input('expires_at') ?: ApiToken::defaultExpiry(),
    ]);

    // 3. 确保 token_id 唯一（极小概率冲突）
    while (ApiToken::query()->where('token_id', '=', $token->token_id)->exists()) {
        $token->token_id = Str::random(32);
    }

    $token->save();

    // 4. secret 仅在创建后通过 session flash 显示一次
    session()->flash('api-token-secret:' . $token->id, $secret);

    return redirect($token->getUrl());
}
```

> **安全要点**：
> - `token_id` 是公开的（用于查询）
> - `secret` 只在创建时显示一次，数据库中以哈希存储
> - 请求格式：`Authorization: Token {token_id}:{secret}`

### 21.4 Token 验证流程

```php
// ApiTokenGuard::getAuthorisedUserFromRequest()
protected function getAuthorisedUserFromRequest(): Authenticatable
{
    // ① 解析请求头
    $authToken = trim($this->request->headers->get('Authorization', ''));
    $this->validateTokenHeaderValue($authToken);

    // ② 拆分 id:secret
    [$id, $secret] = explode(':', str_replace('Token ', '', $authToken));

    // ③ 查询 Token（预加载 user）
    $token = ApiToken::query()
        ->where('token_id', '=', $id)
        ->with(['user'])->first();

    // ④ 验证（4 关）
    $this->validateToken($token, $secret);
    //   1. token 是否存在
    //   2. secret 是否匹配（Hash::check）
    //   3. 是否已过期（expires_at <= now）
    //   4. 用户是否有 AccessApi 权限

    // ⑤ 邮箱确认检查
    if ($this->loginService->awaitingEmailConfirmation($token->user)) {
        throw new ApiAuthException(trans('errors.email_confirmation_awaiting'));
    }

    return $token->user;
}
```

### 21.5 ApiAuthenticate 中间件

```php
// app/Http/Middleware/ApiAuthenticate.php
class ApiAuthenticate
{
    public function handle(Request $request, Closure $next)
    {
        $this->ensureAuthorizedBySessionOrToken($request);
        return $next($request);
    }

    protected function ensureAuthorizedBySessionOrToken(Request $request): void
    {
        // ① 如果已有会话（Cookie 认证）
        if (session()->isStarted()) {
            // 有会话 → 只允许 GET 请求（方便在浏览器中直接浏览 API）
            if ($request->method() !== 'GET') {
                throw new ApiAuthException(trans('errors.api_cookie_auth_only_get'), 403);
            }
            if (!$this->sessionUserHasApiAccess()) {
                throw new ApiAuthException(trans('errors.api_user_no_api_permission'), 403);
            }
            return;
        }

        // ② 无会话 → 切换到 api guard 用 Token 认证
        auth()->shouldUse('api');
        auth()->authenticate();
    }
}
```

### 21.6 Token 生命周期总结

| 阶段 | 操作 | 位置 |
|-----|------|------|
| 创建 | 生成 token_id + secret + Hash 存储 | `UserApiTokenController::store()` |
| 查看 | 仅显示基本信息，secret 不返回 | `UserApiTokenController::edit()` |
| 更新 | 仅可修改 name 和 expires_at | `UserApiTokenController::update()` |
| 删除 | 直接 delete | `UserApiTokenController::destroy()` |
| 验证 | Authorization header → 4 步校验 | `ApiTokenGuard::validateToken()` |
| 过期 | expires_at 字段自动判断 | `ApiTokenGuard::validateToken()` |

---

## 二十二、Notification 与邮件分发

### 22.1 Notification 系统架构

```
Activity::add(type, entity)
    │
    └─ ActivityLogger::add()
          │
          └─ NotificationManager::handle()
                │
                └─ handlersByActivity[type]  // 按活动类型分发
                      │
                      ├─ PageCreationNotificationHandler  → 页面创建通知
                      ├─ PageUpdateNotificationHandler    → 页面更新通知
                      ├─ CommentCreationNotificationHandler → 评论创建通知
                      └─ CommentMentionNotificationHandler  → 评论 @ 提及通知
                            │
                            └─ sendNotificationToUserIds()
                                  │
                                  ├─ 排除触发者本人
                                  ├─ 检查 ReceiveNotifications 权限
                                  ├─ 检查内容可见权限
                                  └─ $user->notify(new XxxNotification(...))
                                        │
                                        └─ BaseActivityNotification
                                              ├─ toArray() → 站内通知（database）
                                              └─ toMail()  → 邮件通知
                                                    │
                                                    └─ MailNotification + Queueable
                                                          └─ locale 切换
```

### 22.2 NotificationManager - 通知管理器

```php
// app/Activity/Notifications/NotificationManager.php
class NotificationManager
{
    // 活动类型 → 处理器映射
    protected array $handlersByActivity = [];

    // 注册默认处理器
    public function loadDefaultHandlers(): void
    {
        $this->registerHandler(ActivityType::PAGE_CREATE, PageCreationNotificationHandler::class);
        $this->registerHandler(ActivityType::PAGE_UPDATE, PageUpdateNotificationHandler::class);
        $this->registerHandler(ActivityType::COMMENT_CREATE, CommentCreationNotificationHandler::class);
        $this->registerHandler(ActivityType::COMMENT_CREATE, CommentMentionNotificationHandler::class);
        $this->registerHandler(ActivityType::COMMENT_UPDATE, CommentMentionNotificationHandler::class);
    }

    // 执行分发
    public function handle(Activity $activity, string|Loggable $detail, User $user): void
    {
        $activityType = $activity->type;
        $handlersToRun = $this->handlersByActivity[$activityType] ?? [];

        foreach ($handlersToRun as $handlerClass) {
            $handler = new $handlerClass();
            $handler->handle($activity, $detail, $user);
        }
    }
}
```

### 22.3 四类通知处理器

#### 22.3.1 PageCreationNotificationHandler - 页面创建

通知范围：关注该书籍的用户

```php
class PageCreationNotificationHandler extends BaseNotificationHandler
{
    public function handle(Activity $activity, ..., User $user): void
    {
        if (!($detail instanceof Page)) {
            throw new \InvalidArgumentException(...);
        }

        // 获取所有关注者（关注 Book 或 Chapter 或 Page 的）
        $watchers = new EntityWatchers($detail, WatchLevels::UPDATES);
        $watcherIds = $watchers->getWatcherUserIds();

        $this->sendNotificationToUserIds(
            PageCreationNotification::class,
            $watcherIds, $user, $detail, $detail
        );
    }
}
```

#### 22.3.2 PageUpdateNotificationHandler - 页面更新

通知范围：关注者 + 页面拥有者（根据偏好）

```php
class PageUpdateNotificationHandler extends BaseNotificationHandler
{
    public function handle(Activity $activity, ...): void
    {
        // 防抖：同一用户 15 分钟内的多次更新只发一次通知
        $lastUpdate = $detail->activity()
            ->where('type', '=', ActivityType::PAGE_UPDATE)
            ->where('id', '!=', $activity->id)
            ->latest('created_at')
            ->first();

        if ($lastUpdate && $lastUpdate->user_id === $user->id) {
            if ($lastUpdate->created_at->gt(now()->subMinutes(15))) {
                return;  // 15 分钟内同一用户更新 → 跳过
            }
        }

        // 关注者
        $watchers = new EntityWatchers($detail, WatchLevels::UPDATES);
        $watcherIds = $watchers->getWatcherUserIds();

        // 页面拥有者（根据通知偏好）
        if ($detail->owned_by && !$watchers->isUserIgnoring($detail->owned_by)) {
            $userNotificationPrefs = new UserNotificationPreferences($detail->ownedBy);
            if ($userNotificationPrefs->notifyOnOwnPageChanges()) {
                $watcherIds[] = $detail->owned_by;
            }
        }

        $this->sendNotificationToUserIds(
            PageUpdateNotification::class, $watcherIds, $user, $detail, $detail
        );
    }
}
```

#### 22.3.3 CommentCreationNotificationHandler - 评论创建

通知范围：页面作者（收到新评论通知）

#### 22.3.4 CommentMentionNotificationHandler - @ 提及

通知范围：被 @ 提及的用户

### 22.4 通知发送过滤（BaseNotificationHandler）

```php
abstract class BaseNotificationHandler implements NotificationHandler
{
    protected function sendNotificationToUserIds(
        string $notification, array $userIds,
        User $initiator, string|Loggable $detail, Entity $relatedModel
    ): void {
        $users = User::query()->whereIn('id', array_unique($userIds))->get();

        foreach ($users as $user) {
            // ① 排除触发者自己
            if ($user->id === $initiator->id) {
                continue;
            }

            // ② 检查接收通知权限
            if (!$user->can(Permission::ReceiveNotifications)) {
                continue;
            }

            // ③ 检查内容可见权限（看不到内容就不通知）
            $permissions = new PermissionApplicator($user);
            if (!$permissions->checkOwnableUserAccess($relatedModel, 'view')) {
                continue;
            }

            // ④ 发送通知
            try {
                $user->notify(new $notification($detail, $initiator));
            } catch (\Exception $exception) {
                Log::error("Failed to send email notification ...");
            }
        }
    }
}
```

### 22.5 BaseActivityNotification - 通知基类

```php
// app/Activity/Notifications/Messages/BaseActivityNotification.php
abstract class BaseActivityNotification extends MailNotification
{
    use Queueable;  // 支持队列异步发送

    public function __construct(
        protected Loggable|string $detail,
        protected User $user,  // 触发通知的用户
    ) {}

    // 站内通知数据
    public function toArray($notifiable): array
    {
        return [
            'activity_detail' => $this->detail,
            'activity_creator' => $this->user,
        ];
    }

    // 辅助：构建页面路径（Book > Chapter），考虑权限可见性
    protected function buildPagePathLine(Page $page, User $notifiable): ?EntityPathMessageLine
    {
        $permissions = new PermissionApplicator($notifiable);
        $path = array_filter([$page->book, $page->chapter], function (?Entity $entity) use ($permissions) {
            return !is_null($entity) && $permissions->checkOwnableUserAccess($entity, 'view');
        });
        return empty($path) ? null : new EntityPathMessageLine($path);
    }
}
```

### 22.6 MailNotification - 邮件基类

```php
// app/App/MailNotification.php
abstract class MailNotification extends Notification
{
    public function via($notifiable)
    {
        return ['mail', 'database'];  // 同时发邮件 + 站内通知
    }

    public function toMail($notifiable)
    {
        // 切换到接收者的语言环境
        $locale = $notifiable->getLocale();
        ...
    }
}
```

### 22.7 关注体系（Watch）

```
用户关注实体（Book / Chapter / Page）
    │
    ├─ 级别：不关注 / 关注更新 / 关注评论
    │
    └─ EntityWatchers - 计算关注者
          │
          ├─ 直接关注当前实体的用户
          ├─ 关注父级（章 → 册）的用户（继承）
          └─ 排除设置了忽略的用户
```

### 22.8 通知偏好设置

```
用户通知偏好（UserNotificationPreferences）
    ├─ notifyOnOwnPageChanges()  // 自己的页面有更新时是否通知自己
    └─ ... 其他偏好
```

---

## 二十三、Attachment 附件资源接入

### 23.1 Attachment 系统架构

```
页面附件管理
    │
    ├─ 上传文件 → AttachmentService::saveNewUpload()
    │     │
    │     ├─ 验证文件大小（upload_limit 配置）
    │     ├─ FileStorage::uploadFile() 存储到磁盘
    │     │     └─ 路径：uploads/files/YYYY-MM/{hash}.{ext}
    │     └─ 创建 Attachment 记录
    │
    ├─ 上传链接 → AttachmentService::saveNewFromLink()
    │     └─ external=true，path 存 URL
    │
    ├─ 访问附件 → AttachmentController::get()
    │     ├─ 检查页面可见权限
    │     ├─ external → 302 重定向
    │     └─ 本地文件 → 流式下载 / 内联打开
    │
    └─ 删除附件 → AttachmentService::deleteFile()
          ├─ external=false → FileStorage::delete()
          └─ Attachment::delete()
```

### 23.2 Attachment 模型

```php
// app/Uploads/Attachment.php
class Attachment extends Model implements OwnableInterface
{
    use HasCreatorAndUpdater;

    protected $fillable = ['name', 'order'];
    protected $hidden = ['path', 'page'];
    protected $casts = ['external' => 'bool'];

    // 所属页面
    public function page(): BelongsTo
    {
        return $this->belongsTo(Page::class, 'uploaded_to');
    }

    // 通过 uploaded_to 关联页面权限（visibility 继承自页面）
    public function jointPermissions(): HasMany
    {
        return $this->hasMany(JointPermission::class, 'entity_id', 'uploaded_to')
            ->where('joint_permissions.entity_type', '=', 'page');
    }

    // 权限过滤作用域（继承页面权限）
    public function scopeVisible(): Builder
    {
        $permissions = app()->make(PermissionApplicator::class);
        return $permissions->restrictPageRelationQuery(
            self::query(), 'attachments', 'uploaded_to'
        );
    }

    // 下载文件名（如果 name 没扩展名就加上）
    public function getFileName(): string { ... }

    // 访问 URL（通过控制器中转，不暴露真实路径）
    public function getUrl($openInline = false): string { ... }

    // 编辑器插入内容（视频自动用 <video> 标签）
    public function editorContent(): array {
        $videoExtensions = ['mp4', 'webm', 'mkv', 'ogg', 'avi'];
        if (in_array(strtolower($this->extension), $videoExtensions)) {
            return ['text/html' => '<video src="..." controls>...</video>'];
        }
        return ['text/html' => $this->htmlLink(), 'text/plain' => $this->markdownLink()];
    }
}
```

**attachments 表结构**：

```
┌─────────────────────────────────────────────────────┐
│                  attachments                     │
├──────────────────┬──────────────────────────────┤
│ id               │ 主键                         │
│ name             │ 显示名称                     │
│ path             │ 存储路径（hidden）           │
│ extension        │ 文件扩展名                   │
│ uploaded_to      │ 所属页面 ID                 │
│ external         │ 是否为外部链接               │
│ order            │ 排序                         │
│ created_by       │ 创建者ID                     │
│ updated_by       │ 更新者ID                     │
│ created_at       │ 创建时间                     │
│ updated_at       │ 更新时间                     │
└──────────────────┴──────────────────────────────┘
```

### 23.3 AttachmentService - 核心服务

```php
// app/Uploads/AttachmentService.php
class AttachmentService
{
    public function __construct(protected FileStorage $storage) {}

    /**
     * 上传新文件
     */
    public function saveNewUpload(UploadedFile $uploadedFile, int $pageId): Attachment
    {
        $attachmentName = $uploadedFile->getClientOriginalName();
        $attachmentPath = $this->putFileInStorage($uploadedFile);  // 存储到文件系统
        $largestExistingOrder = Attachment::where('uploaded_to', '=', $pageId)->max('order');

        return Attachment::forceCreate([
            'name'        => $attachmentName,
            'path'        => $attachmentPath,
            'extension'   => $uploadedFile->getClientOriginalExtension(),
            'uploaded_to' => $pageId,
            'order'       => $largestExistingOrder + 1,
            'created_by'  => user()->id,
            'updated_by'  => user()->id,
        ]);
    }

    /**
     * 更新文件（重新上传）
     */
    public function saveUpdatedUpload(UploadedFile $uploadedFile, Attachment $attachment): Attachment
    {
        if (!$attachment->external) {
            $this->deleteFileInStorage($attachment);  // 删除旧文件
        }
        // ... 保存新文件
    }

    /**
     * 添加链接类型附件
     */
    public function saveNewFromLink(string $name, string $link, int $page_id): Attachment
    {
        return Attachment::forceCreate([
            'name'        => $name,
            'path'        => $link,
            'external'    => true,
            'extension'   => '',
            'uploaded_to' => $page_id,
            'order'       => $largestExistingOrder + 1,
        ]);
    }

    /**
     * 更新附件信息（名称/链接）
     */
    public function updateFile(Attachment $attachment, array $requestData): Attachment { ... }

    /**
     * 删除附件
     */
    public function deleteFile(Attachment $attachment)
    {
        if (!$attachment->external) {
            $this->deleteFileInStorage($attachment);
        }
        $attachment->delete();
    }

    /**
     * 更新排序
     */
    public function updateFileOrderWithinPage(array $attachmentOrder, string $pageId) { ... }

    /**
     * 流式读取
     * @return resource|null
     */
    public function streamAttachmentFromStorage(Attachment $attachment)
    {
        return $this->storage->getReadStream($attachment->path);
    }

    /**
     * 文件大小
     */
    public function getAttachmentFileSize(Attachment $attachment): int
    {
        return $this->storage->getSize($attachment->path);
    }

    // 文件验证规则：受 upload_limit 配置限制
    public static function getFileValidationRules(): array
    {
        return ['file', 'max:' . (config('app.upload_limit') * 1000)];
    }

    // 存储路径：uploads/files/YYYY-MM-M/{hash}.{ext}
    protected function putFileInStorage(UploadedFile $uploadedFile): string
    {
        $basePath = 'uploads/files/' . date('Y-m-M') . '/';
        return $this->storage->uploadFile($uploadedFile, $basePath, $uploadedFile->getClientOriginalExtension(), '');
    }
}
```

### 23.4 AttachmentController - 控制器

**路由与权限对应**：

| 操作 | 权限 | 说明 |
|-----|------|------|
| upload | AttachmentCreateAll + PageUpdate | 上传新文件 |
| uploadUpdate | PageUpdate + AttachmentUpdate | 更新文件 |
| get | PageView（继承） | 查看/下载 |
| listForPage | PageView | 附件列表 |
| sortForPage | PageUpdate | 排序 |
| delete | AttachmentDelete | 删除 |

### 23.5 权限继承机制

Attachment 不直接设置权限，而是**继承所属 Page 的权限**：

```php
// Attachment::scopeVisible()
public function scopeVisible(): Builder
{
    $permissions = app()->make(PermissionApplicator::class);
    return $permissions->restrictPageRelationQuery(
        self::query(), 'attachments', 'uploaded_to'
    );
}
```

即：**能看到页面就能看到附件**。

### 23.6 附件与页面删除的联动

在 `TrashCan::destroyPage()` 中，永久删除页面时会清理附件：

```php
// TrashCan.php:destroyPage()
protected function destroyPage(Page $page): int
{
    $this->destroyCommonRelations($page);

    // 删除附件文件
    $attachmentService = app()->make(AttachmentService::class);
    foreach ($page->attachments as $attachment) {
        $attachmentService->deleteFile($attachment);
    }

    // ... 其他清理
    $page->forceDelete();
    return 1;
}
```

---

## 二十四、完整代码位置速查表（终极版）

### 24.1 SAML / OIDC / Social 认证

| 功能 | 文件位置 |
|------|----------|
| 社交认证服务 | `app/Access/SocialAuthService.php` |
| 社交驱动管理器 | `app/Access/SocialDriverManager.php` |
| 社交认证控制器 | `app/Access/Controllers/SocialController.php` |
| SAML 服务 | `app/Access/Saml2Service.php` |
| SAML 控制器 | `app/Access/Controllers/Saml2Controller.php` |
| OIDC 服务 | `app/Access/Oidc/OidcService.php` |
| OIDC OAuth Provider | `app/Access/Oidc/OidcOAuthProvider.php` |
| OIDC ID Token | `app/Access/Oidc/OidcIdToken.php` |
| OIDC 用户详情 | `app/Access/Oidc/OidcUserDetails.php` |
| OIDC 配置 | `app/Access/Oidc/OidcProviderSettings.php` |
| OIDC 控制器 | `app/Access/Controllers/OidcController.php` |
| 登录服务 | `app/Access/LoginService.php` |
| 注册服务 | `app/Access/RegistrationService.php` |
| 组同步服务 | `app/Access/GroupSyncService.php` |
| 社交账号模型 | `app/Access/SocialAccount.php` |

### 24.2 API Token

| 功能 | 文件位置 |
|------|----------|
| ApiToken 模型 | `app/Api/ApiToken.php` |
| API Token Guard | `app/Api/ApiTokenGuard.php` |
| API 认证中间件 | `app/Http/Middleware/ApiAuthenticate.php` |
| API Token 控制器 | `app/Api/UserApiTokenController.php` |
| API 认证异常 | `app/Exceptions/ApiAuthException.php` |

### 24.3 Notification 通知

| 功能 | 文件位置 |
|------|----------|
| 通知管理器 | `app/Activity/Notifications/NotificationManager.php` |
| 通知处理器接口 | `app/Activity/Notifications/Handlers/NotificationHandler.php` |
| 基础通知处理器 | `app/Activity/Notifications/Handlers/BaseNotificationHandler.php` |
| 页面创建通知 | `app/Activity/Notifications/Handlers/PageCreationNotificationHandler.php` |
| 页面更新通知 | `app/Activity/Notifications/Handlers/PageUpdateNotificationHandler.php` |
| 评论创建通知 | `app/Activity/Notifications/Handlers/CommentCreationNotificationHandler.php` |
| @ 提及通知 | `app/Activity/Notifications/Handlers/CommentMentionNotificationHandler.php` |
| 基础活动通知 | `app/Activity/Notifications/Messages/BaseActivityNotification.php` |
| 邮件通知基类 | `app/App/MailNotification.php` |
| 用户通知偏好 | `app/Settings/UserNotificationPreferences.php` |
| 实体关注器 | `app/Activity/Tools/EntityWatchers.php` |
| 关注级别 | `app/Activity/WatchLevels.php` |

### 24.4 Attachment 附件

| 功能 | 文件位置 |
|------|----------|
| Attachment 模型 | `app/Uploads/Attachment.php` |
| Attachment 服务 | `app/Uploads/AttachmentService.php` |
| 附件控制器 | `app/Uploads/Controllers/AttachmentController.php` |
| 文件存储抽象 | `app/Uploads/FileStorage.php` |
