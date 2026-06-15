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
