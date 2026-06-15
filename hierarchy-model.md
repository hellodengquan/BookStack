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
