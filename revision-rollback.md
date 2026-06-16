# BookStack 页面修订与回滚代码链路分析

## 一、编辑落库策略

### 1.1 双轨草稿机制

BookStack 采用**两级草稿体系**来保障编辑安全：

#### 1.1.1 页面级草稿（Page.draft = true）
- **创建时机**：用户点击"创建新页面"时立即生成
- **存储位置**：`pages` 表，`draft` 字段标记为 `true`
- **核心代码**：`PageRepo::getNewDraftPage()` [app/Entities/Repos/PageRepo.php:42-78](app/Entities/Repos/PageRepo.php#L42-L78)

```php
public function getNewDraftPage(Entity $parent): Page
{
    $page = (new Page())->forceFill([
        'name'       => trans('entities.pages_initial_name'),
        'created_by' => user()->id,
        'owned_by'   => user()->id,
        'updated_by' => user()->id,
        'draft'      => true,  // 标记为草稿
        'editor'     => PageEditorType::getSystemDefault()->value,
        'html'       => '',
        'markdown'   => '',
        'text'       => '',
    ]);
    // ... 保存并重建权限
    return $page;
}
```

#### 1.1.2 修订级草稿（PageRevision.type = 'update_draft'）
- **创建时机**：编辑已有页面时，自动保存触发
- **存储位置**：`page_revisions` 表，`type` 字段为 `'update_draft'`
- **核心代码**：`RevisionRepo::getNewDraftForCurrentUser()` [app/Entities/Repos/RevisionRepo.php:28-44](app/Entities/Repos/RevisionRepo.php#L28-L44)

```php
public function getNewDraftForCurrentUser(Page $page): PageRevision
{
    $draft = $this->queries->findLatestCurrentUserDraftsForPageId($page->id);
    if ($draft) {
        return $draft;  // 已有草稿则复用
    }
    $draft = new PageRevision();
    $draft->page_id = $page->id;
    $draft->type = 'update_draft';  // 标记为更新草稿
    // ...
    return $draft;
}
```

### 1.2 前端自动保存策略

- **触发频率**：30秒间隔，仅在内容有变更时执行
- **兜底机制**：AJAX保存失败时写入LocalStorage
- **核心代码**：`PageEditor::runAutoSave()` [resources/js/components/page-editor.js:109-117](resources/js/components/page-editor.js#L109-L117)

```javascript
runAutoSave() {
    const savedRecently = (Date.now() - this.autoSave.last < this.autoSave.frequency / 2);
    if (savedRecently || !this.autoSave.pendingChange) {
        return;
    }
    this.saveDraft();
}
```

### 1.3 正式保存流程

#### 1.3.1 草稿发布（首次发布）
- **入口**：`PageController::store()` → `PageRepo::publishDraft()`
- **关键操作**：
  1. `draft` 标记改为 `false`
  2. `revision_count` 初始化为 `1`
  3. 创建第一条正式修订记录（type = 'version'）
- **核心代码**：`PageRepo::publishDraft()` [app/Entities/Repos/PageRepo.php:83-103](app/Entities/Repos/PageRepo.php#L83-L103)

#### 1.3.2 页面更新
- **入口**：`PageController::update()` → `PageRepo::update()`
- **关键操作**：
  1. 保存旧值用于变更检测
  2. 更新页面内容
  3. `revision_count++`
  4. 删除当前用户的更新草稿
  5. 检测内容变化，有变化则创建新修订
- **核心代码**：`PageRepo::update()` [app/Entities/Repos/PageRepo.php:119-149](app/Entities/Repos/PageRepo.php#L119-L149)

```php
public function update(Page $page, array $input): Page
{
    $oldHtml = $page->html;
    $oldName = $page->name;
    $oldMarkdown = $page->markdown;
    
    // 更新内容...
    
    $page->revision_count++;  // 版本号递增
    $page->save();
    
    $this->revisionRepo->deleteDraftsForCurrentUser($page);
    
    // 变更检测
    $htmlChanged = isset($input['html']) && $input['html'] !== $oldHtml;
    $nameChanged = isset($input['name']) && $input['name'] !== $oldName;
    $markdownChanged = isset($input['markdown']) && $input['markdown'] !== $oldMarkdown;
    
    if ($htmlChanged || $nameChanged || $markdownChanged || $summary) {
        $this->revisionRepo->storeNewForPage($page, $summary);
    }
    // ...
}
```

### 1.4 updatePage 事务回滚链

#### 1.4.1 事务封装机制

BookStack 使用自定义的 `DatabaseTransaction` 类封装数据库事务，设置 `READ COMMITTED` 隔离级别：

- **核心类**：`DatabaseTransaction` [app/Util/DatabaseTransaction.php](app/Util/DatabaseTransaction.php)
- **隔离级别**：`READ COMMITTED` — 事务内能读到其他已提交事务的修改
- **设计原因**：权限生成等场景需要考虑其他已提交事务的变更

```php
class DatabaseTransaction
{
    public function run(): mixed
    {
        DB::statement('SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED');
        return DB::transaction($this->callback);
    }
}
```

#### 1.4.2 新页面创建事务链

`publishDraft()` 使用事务包裹以下操作，任一环节失败全部回滚：

1. `draft = false` + `revision_count = 1`
2. `BaseRepo::update()` — 更新页面基本信息
3. `rebuildPermissions()` — 重建权限
4. `storeNewForPage()` — 创建第一条修订
5. `Activity::add()` — 记录活动日志
6. `sortParent()` — 父级排序

- **核心代码**：`PageRepo::publishDraft()` [app/Entities/Repos/PageRepo.php:85-102](app/Entities/Repos/PageRepo.php#L85-L102)

```php
public function publishDraft(Page $draft, array $input): Page
{
    return (new DatabaseTransaction(function () use ($draft, $input) {
        $draft->draft = false;
        $draft->revision_count = 1;
        // ... 更新内容 ...
        $draft = $this->baseRepo->update($draft, $input);
        $draft->rebuildPermissions();
        $this->revisionRepo->storeNewForPage($draft, $summary);
        // ... 活动日志 ...
        return $draft;
    }))->run();
}
```

#### 1.4.3 页面更新的非事务特性

**注意**：常规 `PageRepo::update()` **没有显式事务包裹**，以下操作分步执行：

| 步骤 | 操作 | 失败影响 |
|------|------|----------|
| 1 | `BaseRepo::update()` 保存页面 | 页面数据不更新 |
| 2 | `revision_count++` 保存 | 版本号不递增 |
| 3 | 删除用户草稿 | 草稿可能残留 |
| 4 | `storeNewForPage()` 创建修订 | 修订不创建，但页面已更新 |

> **风险**：第4步（创建修订）失败时，页面内容已更新但版本历史缺失，出现"幽灵更新"。

#### 1.4.4 回滚操作的非事务特性

`restoreRevision()` 同样**没有事务包裹**，分步执行：

1. `revision_count++` + 填充旧数据
2. 重新解析内容（setNewMarkdown/HTML）
3. `refreshSlug()` + `save()`
4. `indexForSearch()` — 搜索索引
5. `referenceStore->updateForEntity()` — 引用存储
6. `storeNewForPage()` — 创建恢复版本的修订
7. URL 变化时更新引用
8. 活动日志

> **风险点**：步骤 6 之前失败会导致页面已回滚但无修订记录；步骤 7 失败会导致引用链接失效。

#### 1.4.5 DB transaction savepoint 嵌套能力

**实际状态**：BookStack 没有使用数据库 `SAVEPOINT` 进行嵌套事务控制。

- **代码证据**：`DatabaseTransaction` 类只有一层 `DB::transaction()` 包装
- **Laravel 底层**：Laravel 的 `DB::transaction()` 会检测嵌套调用，对于 MySQL 会自动用 savepoint 处理（见 `Illuminate/Database/Concerns/ManagesTransactions`）
- **BookStack 实际使用**：所有事务都是单层的，没有嵌套事务场景

```php
// DatabaseTransaction.php:37-41 - 只有单层事务，无 savepoint 操作
public function run(): mixed
{
    DB::statement('SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED');
    return DB::transaction($this->callback);  // Laravel 自动处理嵌套
}
```

**唯一的嵌套事务场景**：`TrashCan::destroyEntity()` 内调用 `destroyBook()` 又遍历调用 `destroyPage()`，但 `destroyEntity()` 本身已经用 `DatabaseTransaction` 包裹，内层的删除操作都在同一个外层事务内执行。

```php
// TrashCan.php:370-386
public function destroyEntity(Entity $entity): int
{
    return (new DatabaseTransaction(function () use ($entity) {
        if ($entity instanceof Book) {
            return $this->destroyBook($entity);  // 内部遍历调用 destroyPage()
        }
        // ...
    }))->run();
}
```

> **注意**：虽然 Laravel 底层会用 savepoint 支持嵌套事务，但 BookStack 业务代码没有主动使用 savepoint，也没有针对 savepoint 的回滚逻辑。

### 1.5 修订存储与清理

- **存储字段**：完整快照（name、html、markdown、text）
- **版本限制**：通过 `config('app.revision_limit')` 配置，默认保留最近版本
- **清理时机**：每次创建新修订时触发
- **核心代码**：`RevisionRepo::deleteOldRevisions()` [app/Entities/Repos/RevisionRepo.php:75-92](app/Entities/Repos/RevisionRepo.php#L75-L92)

```php
protected function deleteOldRevisions(Page $page): void
{
    $revisionLimit = config('app.revision_limit');
    if ($revisionLimit === false) return;
    
    $revisionsToDelete = PageRevision::query()
        ->where('page_id', '=', $page->id)
        ->orderBy('created_at', 'desc')
        ->skip(intval($revisionLimit))
        ->take(10)
        ->get(['id']);
    // 删除超出限制的修订
}
```

---

## 二、版本号生成机制

### 2.1 双字段版本体系

| 字段 | 所在表 | 含义 | 更新时机 |
|------|--------|------|----------|
| `revision_count` | `pages` | 页面当前总修订数 | 每次更新/回滚时 `++` |
| `revision_number` | `page_revisions` | 修订对应的页面版本号 | 创建修订时取自 `page.revision_count` |

### 2.2 版本号演进路径

#### 2.2.1 新页面创建
```
publishDraft():
  page.revision_count = 1
  create revision with revision_number = 1
```

#### 2.2.2 页面更新
```
update():
  page.revision_count++  // 假设原为1，变为2
  save()
  create revision with revision_number = 2  // 存入当前值
```

#### 2.2.3 版本回滚
```
restoreRevision():
  page.revision_count++  // 回滚也产生新版本
  save()
  create revision with revision_number = N+1
```

### 2.3 revision_number 并发插入冲突分析

#### 2.3.1 并发场景与风险

**典型并发时序**（两个用户同时保存同一页面）：

```
User A: read page.revision_count = 5
User B: read page.revision_count = 5
User A: page.revision_count++ → 6, save()
User A: create revision with revision_number = 6 ✓
User B: page.revision_count++ → 7, save()
User B: create revision with revision_number = 7 ✓
```

看似没有问题，但存在**丢失更新风险**：

| 时序 | 用户A | 用户B | page.revision_count |
|------|-------|-------|---------------------|
| T1 | 读取=5 | - | 5 |
| T2 | - | 读取=5 | 5 |
| T3 | ++ → 6，save | - | 6 |
| T4 | - | ++ → 6，save | 6 (覆盖!) |
| T5 | 创建修订#6 | - | 6 |
| T6 | - | 创建修订#6 | 6 |

> **问题**：PHP 层 `++` 操作不是原子的，两个请求都读到 5，都改成 6，产生两个 `revision_number = 6` 的修订，且 `revision_count` 只递增了一次。

#### 2.3.2 现有防御机制

**实际上 BookStack 并没有显式的乐观锁或悲观锁机制**，依赖以下因素降低冲突概率：

1. **草稿分散写入**：自动保存写入 `update_draft` 类型的修订，不更新 `revision_count`
2. **正式保存低频**：用户主动点击保存才触发 `revision_count++`，频率远低于自动保存
3. **并发警告**：通过 `PageEditActivity` 提前告知用户有人在编辑

#### 2.3.3 为什么用自增ID而非revision_number定位

`getPreviousRevision()` 使用自增 ID 定位前一版本 [app/Entities/Models/PageRevision.php:66-77](app/Entities/Models/PageRevision.php#L66-L77)，而非 `revision_number`：

```php
$id = static::newQuery()->where('page_id', '=', $this->page_id)
    ->where('id', '<', $this->id)  // 用自增ID比较
    ->max('id');
```

**原因**：
- `revision_number` 可能重复（并发冲突）
- 自增 ID 全局唯一，不会重复
- 即使版本号冲突，历史记录仍然可通过 ID 正确遍历

#### 2.3.4 UNIQUE constraint 冲突重试

**Slug 生成的冲突检测与重试**：

`SlugGenerator` 在生成 URL slug 时，先查询再生成，遇到冲突追加随机后缀：

- **核心代码**：`SlugGenerator::generate()` [app/Entities/Tools/SlugGenerator.php:18-26](app/Entities/Tools/SlugGenerator.php#L18-L26)

```php
public function generate(SluggableInterface&Model $model, string $slugSource): string
{
    $slug = $this->formatNameAsSlug($slugSource);
    while ($this->slugInUse($slug, $model)) {
        $slug .= '-' . Str::random(3);  // 冲突则追加随机3位随机字符串
    }
    return $slug;
}
```

**缺陷**：
1. **TOCTOU 风险**：先检查后生成，并发创建同书名页面时，查询显示可用但插入时可能冲突
2. **无数据库层重试**：没有 catch `PDOException` 进行数据库层的冲突重试
3. **无限循环风险**：极端高并发下可能多次追加后缀仍冲突

**页面修订表无唯一约束**：`page_revisions` 表没有针对 `(page_id, revision_number)` 的唯一约束，理论上允许重复的 `revision_number`。

```sql
-- page_revisions 表结构（无唯一约束）
CREATE TABLE page_revisions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    page_id INT NOT NULL,
    revision_number INT NULL,
    -- ... 其他字段
    -- 没有 UNIQUE KEY (page_id, revision_number)
)
```

**对比**：
- `pages` 表有 `UNIQUE KEY (book_id, slug)` — 有唯一约束，但生成器用"检查+重试"
- `page_revisions` 表无复合唯一约束 — 允许版本号重复
- 无任何一处使用 `try/catch PDOException` 重试机制

#### 2.3.5 并发冲突的实际影响

即使发生并发覆盖，后果相对可控：

- **revision_count 不准确**：显示的修订总数偏少，但修订记录本身存在
- **重复的 revision_number**：两个修订版本号相同，但内容是两份快照
- **不影响回滚**：回滚通过 `id` 而非 `revision_number` 定位
- **不影响差异对比**：diff 通过 ID 顺序遍历

> **结论**：并发冲突主要影响"版本号展示"的准确性，不影响核心功能可用性。这是一个已知的权衡设计，优先保障写入性能和用户体验。

### 2.4 数据库设计溯源

- `pages.revision_count` 字段：[database/migrations/2017_04_20_185112_add_revision_counts.php:15-17](database/migrations/2017_04_20_185112_add_revision_counts.php#L15-L17)
- `page_revisions.revision_number` 字段：[database/migrations/2017_04_20_185112_add_revision_counts.php:18-21](database/migrations/2017_04_20_185112_add_revision_counts.php#L18-L21)

---

## 三、差异对比实现

### 3.1 第三方依赖

使用 `ssddanbrown/htmldiff` 库进行HTML级别的差异对比：
- 版本：`^2.0.0`
- 引入位置：[composer.json:42](composer.json#L42)

### 3.2 差异对比流程

- **入口**：`PageRevisionController::changes()` [app/Entities/Controllers/PageRevisionController.php:98-128](app/Entities/Controllers/PageRevisionController.php#L98-L128)
- **对比对象**：当前修订 vs 前一修订（通过 `getPreviousRevision()` 获取）
- **核心代码**：

```php
public function changes(string $bookSlug, string $pageSlug, int $revisionId)
{
    // ...
    $prev = $revision->getPreviousRevision();
    $prevContent = $prev->html ?? '';
    
    // 执行差异对比
    $rawDiff = Diff::excecute($prevContent, $revision->html);
    
    // 内容过滤（XSS防护）
    $filterConfig = HtmlContentFilterConfig::fromConfigString(config('app.content_filtering'));
    $filter = new HtmlContentFilter($filterConfig);
    $diff = $filter->filterString($rawDiff);
    // ...
}
```

### 3.3 前一版本定位算法

- **核心代码**：`PageRevision::getPreviousRevision()` [app/Entities/Models/PageRevision.php:66-77](app/Entities/Models/PageRevision.php#L66-L77)

```php
public function getPreviousRevision(): ?PageRevision
{
    $id = static::newQuery()->where('page_id', '=', $this->page_id)
        ->where('id', '<', $this->id)
        ->max('id');  // 取比当前ID小的最大ID
    
    return $id ? static::query()->find($id) : null;
}
```

> **注意**：这里使用的是自增ID而非 `revision_number` 来定位前一版本，避免版本号不连续导致的定位错误。

### 3.4 html_diff 大文档性能分析

#### 3.4.1 算法复杂度

`ssddanbrown/htmldiff` 基于经典的 HTML diff 算法：

- **核心思想**：将 HTML 拆分为词元（words），使用 LCS（最长公共子序列）算法找差异
- **时间复杂度**：O(n*m)，n 和 m 分别为两个版本的词元数
- **空间复杂度**：O(n*m)，需构建二维动态规划表

#### 3.4.2 性能瓶颈点

| 瓶颈 | 原因 | 影响 |
|------|------|------|
| **大文档比对** | LCS 算法平方级复杂度 | 文档越长，耗时指数级增长 |
| **HTML 结构复杂** | 嵌套标签多，词元数量膨胀 | 内存占用高，计算慢 |
| **大段内容替换** | 几乎无公共子序列 | 退化为 O(n*m) 最坏情况 |

#### 3.4.3 BookStack 中的性能保障

BookStack 对 diff 做了以下间接优化：

1. **仅按需加载**：`changes()` 路由独立，用户点击"查看变化"才执行 diff
2. **分页展示**：修订列表分页（50条/页），diff 只针对单个修订
3. **内容过滤后再对比**：先过滤再 diff，减少无效标签干扰
   ```php
   $rawDiff = Diff::excecute($prevContent, $revision->html);
   $diff = $filter->filterString($rawDiff);  // 注意：过滤在diff之后
   ```

> **注意**：`HtmlContentFilter` 在 diff 之后执行，意味着 diff 过程处理的是原始 HTML，可能包含大量可过滤标签，增加了计算量。

#### 3.4.4 patience diff 复杂度分析

**ssddanbrown/htmldiff 算法实现**：

该库实现的是标准的 HTML Diff 算法，不是 git 风格的 patience diff。核心流程：

1. **HTML 解析**：将 HTML 拆分为词元（words），保留标签边界信息
2. **LCS 计算**：使用动态规划计算两个词元序列的最长公共子序列
3. **差异标记**：根据 LCS 标记插入（`<ins>`）和删除（`<del>`）

**复杂度对比**：

| 算法类型 | 时间复杂度 | 空间复杂度 | 适用场景 |
|---------|-----------|-----------|----------|
| **标准 LCS diff** (当前使用) | O(n*m) | O(n*m) | 通用HTML比较 |
| **Patience Diff** (git) | O(n log n) 平均 | O(n) | 代码行比较，长文档 |

**Patience Diff 优势**：
1. 对长文档性能更好（从平方级降到近线性）
2. 对代码类内容匹配更准确
3. 内存占用更低

**BookStack 未使用 Patience Diff 的原因**：
- 依赖的库 `ssddanbrown/htmldiff` 本身不实现 patience diff
- HTML 结构比纯文本更复杂，标签需要特殊处理
- 短文档场景下 LCS 算法足够快

#### 3.4.5 大文档场景风险

- **内存溢出**：特别长的页面（几万字+复杂HTML）可能导致 PHP 内存超限
- **超时风险**：diff 计算可能超过 PHP `max_execution_time` 限制
- **无缓存机制**：每次访问重新计算，不缓存 diff 结果

---

## 四、并发覆盖兜底机制

### 4.1 活跃编辑检测

#### 4.1.1 检测逻辑
- **核心类**：`PageEditActivity` [app/Entities/Tools/PageEditActivity.php](app/Entities/Tools/PageEditActivity.php)
- **检测窗口**：最近60分钟内的更新草稿
- **排除条件**：排除当前用户自己的草稿
- **核心代码**：`PageEditActivity::activePageEditingQuery()` [app/Entities/Tools/PageEditActivity.php:92-104](app/Entities/Tools/PageEditActivity.php#L92-L104)

```php
protected function activePageEditingQuery(int $withinMinutes): Builder
{
    $checkTime = Carbon::now()->subMinutes($withinMinutes);
    return PageRevision::query()
        ->where('type', '=', 'update_draft')
        ->where('page_id', '=', $this->page->id)
        ->where('updated_at', '>', $this->page->updated_at)  // 草稿新于页面最后更新
        ->where('created_by', '!=', user()->id)  // 排除自己
        ->where('updated_at', '>=', $checkTime)  // 时间窗口内
        ->with('createdBy');
}
```

#### 4.1.2 警告触发时机

1. **进入编辑页时**：`PageEditorData::build()` [app/Entities/Tools/PageEditorData.php:54-56](app/Entities/Tools/PageEditorData.php#L54-L56)
   ```php
   if ($editActivity->hasActiveEditing()) {
       $this->warnings[] = $editActivity->activeEditingMessage();
   }
   ```

2. **自动保存时**：`PageController::saveDraft()` [app/Entities/Controllers/PageController.php:249](app/Entities/Controllers/PageController.php#L249)
   ```php
   $warnings = (new PageEditActivity($page))->getWarningMessagesForDraft($draft);
   ```

### 4.2 页面已更新检测

- **检测逻辑**：草稿创建时间 < 页面最后更新时间
- **核心代码**：`PageEditActivity::hasPageBeenUpdatedSinceDraftCreated()` [app/Entities/Tools/PageEditActivity.php:69-72](app/Entities/Tools/PageEditActivity.php#L69-L72)

```php
protected function hasPageBeenUpdatedSinceDraftCreated(PageRevision $draft): bool
{
    return $draft->page->updated_at->timestamp > $draft->created_at->timestamp;
}
```

### 4.3 多层级警告体系

| 警告类型 | 触发场景 | 消息示例 |
|----------|----------|----------|
| 他人活跃编辑 | 60分钟内有其他用户的更新草稿 | "Admin has started editing this page in the last 60 minutes..." |
| 页面已更新 | 草稿创建后页面被他人修改 | "The page content has been updated since this draft was created." |
| 编辑已有草稿 | 用户继续编辑自己之前的草稿 | "You are currently editing a draft that was last saved..." |

### 4.4 前端警告展示

- **缓存机制**：`shownWarningsCache` Set 存储已展示过的警告，避免重复提示
- **触发方式**：通过 `window.$events.emit('warning', message)` 事件系统
- **核心代码**：[resources/js/components/page-editor.js:139-142](resources/js/components/page-editor.js#L139-L142)

```javascript
if (resp.data.warning && !this.shownWarningsCache.has(resp.data.warning)) {
    window.$events.emit('warning', resp.data.warning);
    this.shownWarningsCache.add(resp.data.warning);
}
```

> **设计思想**：BookStack 采用**警告而非阻塞**的策略，用户可以选择继续保存，但会被明确告知风险。

### 4.5 乐观锁冲突解决 UI

#### 4.5.1 无真正的乐观锁机制

BookStack **没有实现基于版本号的乐观锁**（即没有 `version` 字段用于 `WHERE version = ?` 的原子更新检查）。

当前的"冲突检测"本质是：
- **检测**：查询是否有他人的 `update_draft` 草稿
- **提示**：给出警告消息
- **不阻止**：用户仍然可以保存，不会因为冲突而拒绝写入

#### 4.5.2 冲突UI呈现方式

警告通过两种方式传递给用户：

**方式一：页面级通知（进入编辑页时）**

```php
// PageEditorData::build()
if ($editActivity->hasActiveEditing()) {
    $this->warnings[] = $editActivity->activeEditingMessage();
}
// 在控制器中通过 showWarningNotification() 展示
if ($editorData->getWarnings()) {
    $this->showWarningNotification(implode("\n", $editorData->getWarnings()));
}
```

**方式二：AJAX 返回的 warning 字段（自动保存时）**

```javascript
// page-editor.js
if (resp.data.warning && !this.shownWarningsCache.has(resp.data.warning)) {
    window.$events.emit('warning', resp.data.warning);
    this.shownWarningsCache.add(resp.data.warning);
}
```

#### 4.5.3 缺少的冲突解决能力

与真正的协作系统相比，BookStack 缺少：

| 能力 | 状态 | 说明 |
|------|------|------|
| **实时看到他人编辑** | ❌ | 仅知道有人在编辑，看不到编辑内容 |
| **冲突差异对比** | ❌ | 不会对比你的版本和最新版本的差异 |
| **选择合并** | ❌ | 没有"接受我的版本/接受他人版本/合并"选项 |
| **保存被拒绝** | ❌ | 后保存的会直接覆盖先保存的 |

#### 4.5.4 ConflictResolutionModal 移动端适配

**实际状态**：BookStack 没有名为 `ConflictResolutionModal` 的冲突解决弹窗组件。

**现有的警告通知机制**：

1. **页面级通知**（进入编辑页时）：
   - 通过 `showWarningNotification()` 发送到 session flash
   - 由前端全局通知系统展示，位于页面顶部
   - 无专门的移动端适配代码，依赖全局通知组件的响应式样式

2. **AJAX 警告**（自动保存时）：
   - 通过 `window.$events.emit('warning', message)` 事件系统
   - 同样由全局通知组件展示

**全局弹窗组件的移动端适配**：

编辑器内部弹窗使用 `EditorFormModal` 类，样式定义在 `_editor.scss`：

- **核心样式**：`editor-modal` [resources/sass/_editor.scss:293-300](resources/sass/_editor.scss#L293-L300)

```scss
.editor-modal {
  @include mixins.lightDark(background-color, #FFF, #222);
  border-radius: 4px;
  overflow: hidden;
  box-shadow: 0 0 15px 0 rgba(0, 0, 0, 0.3);
  margin: vars.$xs;      // 小屏外边距
  max-height: 100%;      // 不超出视口
  overflow-y: auto;      // 内容可滚动
}
```

**响应式断点**：
- 断点定义在 `vars.$bp-xs`（默认约 576px）
- 小屏下 `margin: vars.$xs` 确保弹窗不贴边
- `max-height: 100% + overflow-y: auto` 确保长内容可滚动

```scss
// 小屏断点示例：_editor.scss:560
@include mixins.smaller-than(vars.$bp-xs) {
  // 小屏特定样式
}
```

> **结论**：虽然没有专门的冲突解决弹窗，但现有弹窗组件已做了基本的移动端适配，包括小屏边距、高度限制和内容滚动。

### 4.6 分支与合并机制

#### 4.6.1 无版本分支概念

BookStack 的修订历史是**线性的**，没有分支（branch）概念：

- 只有一条时间线，按 `created_at` / `id` 排序
- 每个修订都只有一个"前一版本"和（可能的）"后一版本"
- 回滚 = 创建新版本，不是切回旧分支

#### 4.6.2 Yjs 实时协作（实验性？）

代码中存在 Yjs 协同编辑相关实现，位于 `resources/js/wysiwyg/lexical/yjs/` 目录下：

- **核心文件**：
  - `SyncEditorStates.ts` — 编辑器状态同步
  - `SyncCursors.ts` — 光标位置同步
  - `Bindings.ts` — Yjs 与 Lexical 绑定
  - `CollabElementNode.ts` / `CollabTextNode.ts` — 可协作节点类型

- **冲突解决方式**：CRDT（无冲突复制数据类型）
  - Yjs 自动处理并发编辑的合并
  - 不需要用户手动解决冲突
  - 基于操作转换（OT）思想的 CRDT 实现

```typescript
// SyncEditorStates.ts - 核心同步逻辑
export function syncYjsChangesToLexical(
  binding: Binding,
  provider: Provider,
  events: Array<YEvent<YText>>,
  isFromUndoManger: boolean,
): void {
  editor.update(() => {
    for (let i = 0; i < events.length; i++) {
      $syncEvent(binding, events[i]);
    }
    // ... 光标同步
  });
}
```

#### 4.6.3 实时协作的现状

根据代码分析：
- 仅存在于 Lexical 编辑器（下一代 WYSIWYG 编辑器）
- 有完整的 Yjs 绑定实现
- 但**需要 Provider**（如 WebSocket 后端）才能真正启用
- 默认安装中可能未启用实时协作功能

> **注意**：Yjs 协作是前端层面的实时合并，最终保存时仍然走常规修订流程，产生 `revision_count++` 的新版本快照。

### 4.7 PageDraft 长期分支管理

**实际状态**：BookStack 没有真正意义上的"长期分支"概念，所有草稿都是临时的。

#### 4.7.1 两级草稿的生命周期

| 草稿类型 | 创建时机 | 生命周期 | 清理时机 |
|---------|---------|---------|---------|
| **Page.draft** (页面级草稿) | 用户点击"创建新页面" | 直到正式发布前一直存在 | `publishDraft()` 时设置 `draft=false` |
| **PageRevision.update_draft** (修订级草稿) | 编辑已有页面并触发自动保存 | 同一用户同一页面仅保留一份 | 1. 正式保存时删除<br>2. 用户手动删除<br>3. 命令行 `bookstack:clear-revisions --all` |

#### 4.7.2 无过期自动清理机制

`update_draft` 草稿**没有过期自动清理**，除非：
1. 用户正式保存（`update()` 内调用 `deleteDraftsForCurrentUser()`）
2. 管理员执行 `bookstack:clear-revisions --all` 命令
3. 用户手动点击"删除草稿"按钮

- **核心代码**：`ClearRevisionsCommand` [app/Console/Commands/ClearRevisionsCommand.php](app/Console/Commands/ClearRevisionsCommand.php)

```php
public function handle(): int
{
    $deleteTypes = $this->option('all') ? ['version', 'update_draft'] : ['version'];
    PageRevision::query()->whereIn('type', $deleteTypes)->delete();
    return 0;
}
```

#### 4.7.3 "长期分支"的缺陷

由于没有真正的分支管理，长期编辑草稿存在以下问题：

1. **草稿冲突风险**：两个用户各自编辑草稿，后保存的会覆盖先保存的
2. **草稿丢失风险**：用户换设备/浏览器后无法继续编辑之前的草稿
3. **无草稿对比**：无法对比草稿与正式页面的差异（除非手动调用 changes 路由）
4. **无草稿合并**：无法将草稿与页面最新版本合并

#### 4.7.4 草稿复用机制

同一用户编辑同一页面时复用草稿，不会创建多份：

- **核心代码**：`RevisionRepo::getNewDraftForCurrentUser()` [app/Entities/Repos/RevisionRepo.php:28-44](app/Entities/Repos/RevisionRepo.php#L28-L44)

```php
public function getNewDraftForCurrentUser(Page $page): PageRevision
{
    $draft = $this->queries->findLatestCurrentUserDraftsForPageId($page->id);
    if ($draft) {
        return $draft;  // 已有草稿则复用
    }
    // 创建新草稿
}
```

---

## 五、版本审计与权限

### 5.1 修订权限体系

#### 5.1.1 权限定义

专门的修订权限只有一个：

```php
// Permission.php
case RevisionViewAll = 'revision-view-all';
```

权限粒度：
- **查看**：`revision-view-all` — 全局查看所有修订历史
- **删除**：受 `page-delete` 权限控制（删除修订需要页面删除权限）
- **恢复**：受 `page-update` 权限控制（回滚需要页面更新权限）

- **核心代码**：`Permission::RevisionViewAll` [app/Permissions/Permission.php:121](app/Permissions/Permission.php#L121)

#### 5.1.2 权限检查点

| 操作 | 权限检查 | 代码位置 |
|------|----------|----------|
| 查看修订列表 | `RevisionViewAll` | `PageRevisionController::index()` |
| 查看修订详情 | `RevisionViewAll` | `PageRevisionController::show()` |
| 查看修订差异 | `RevisionViewAll` | `PageRevisionController::changes()` |
| 执行回滚 | `PageUpdate` + `RevisionViewAll` | `PageRevisionController::restore()` |
| 删除修订 | `PageDelete` + `RevisionViewAll` | `PageRevisionController::destroy()` |

```php
// PageRevisionController::restore()
$this->checkPermission(Permission::RevisionViewAll);
$page = $this->pageQueries->findVisibleBySlugsOrFail($bookSlug, $pageSlug);
$this->checkOwnablePermission(Permission::PageUpdate, $page);
```

### 5.2 审计日志

#### 5.2.1 修订相关活动类型

```php
// ActivityType.php
const REVISION_RESTORE = 'revision_restore';  // 修订被恢复
const REVISION_DELETE = 'revision_delete';    // 修订被删除
```

- **核心代码**：`ActivityType` [app/Activity/ActivityType.php:36-37](app/Activity/ActivityType.php#L36-L37)

#### 5.2.2 审计日志查询

- **入口**：`AuditLogController::index()` [app/Activity/Controllers/AuditLogController.php:15-72](app/Activity/Controllers/AuditLogController.php#L15-L72)
- **权限要求**：`SettingsManage` + `UsersManage`（管理员级权限）
- **筛选维度**：事件类型、日期范围、用户、IP

#### 5.2.3 审计日志与修订记录的区别

| 维度 | 修订记录 (page_revisions) | 审计日志 (activities) |
|------|---------------------------|----------------------|
| **内容** | 包含完整页面内容快照 | 仅记录操作事件元数据 |
| **目的** | 支持回滚、差异对比 | 操作审计、安全追溯 |
| **保留** | 受 revision_limit 限制 | 通常永久保留（需手动清理） |
| **权限** | revision-view-all | settings-manage + users-manage |
| **粒度** | 按页面 | 全系统 |

#### 5.2.4 audit log 完整性保障分析

**ActivityLogger 实现**：

- **核心类**：`ActivityLogger` [app/Activity/Tools/ActivityLogger.php](app/Activity/Tools/ActivityLogger.php)
- **调用流程**：主业务逻辑 → `Activity::add()` → 日志记录

```php
// ActivityLogger.php:27-45
public function add(string $type, string|Loggable $detail = ''): void
{
    $detailToStore = ($detail instanceof Loggable) ? $detail->logDescriptor() : $detail;
    
    $activity = $this->newActivityForUser($type);
    $activity->detail = $detailToStore;
    
    if ($detail instanceof Entity) {
        $activity->loggable_id = $detail->id;
        $activity->loggable_type = $detail->getMorphClass();
    }
    
    $activity->save();  // ① 保存活动日志
    
    $this->setNotification($type);                // ② 设置通知
    $this->dispatchWebhooks($type, $detail);       // ③ 发送 Webhook
    $this->notifications->handle($activity, $detail, user());  // ④ 通知处理
    Theme::dispatch(ThemeEvents::ACTIVITY_LOGGED, $type, $detail);  // ⑤ 主题事件
}
```

**完整性风险**：

1. **事务外写入**：日志记录在主事务提交之后执行，主事务回滚不影响日志（但也意味着如果主事务成功但日志写入失败，会出现"操作已执行但无日志"）
2. **无事务包裹**：`$activity->save()` 独立于主事务
3. **分步执行**：步骤 ①-⑤ 分步执行，中间任意一步失败不会回滚已写入的日志
4. **无补偿机制**：日志写入失败不会重试，也不会回滚主事务

**日志失败的影响范围**：

| 步骤 | 失败后果 | 是否影响主业务 |
|------|---------|--------------|
| ① `$activity->save()` | 操作无审计记录 | 不影响（业务已完成） |
| ② `setNotification()` | 用户看不到成功通知 | 不影响 |
| ③ `dispatchWebhooks()` | Webhook 未发送 | 不影响 |
| ④ `notifications->handle()` | 通知未发送 | 不影响 |
| ⑤ `Theme::dispatch()` | 主题事件未触发 | 不影响 |

**审计完整性的薄弱点**：
- 活动日志与业务操作不绑定在同一事务
- 日志写入失败静默失败，没有告警
- 没有日志完整性校验机制（如哈希链、签名）
- `ClearActivityCommand` 可清空日志，无二次确认

---

## 六、存储优化与回滚副作用

### 6.1 存储优化：仅存 diff 的可能性

#### 6.1.1 当前策略：完整快照

BookStack 采用**全量快照**存储策略，每次修订保存完整的 HTML/Markdown/文本：

```php
// RevisionRepo::storeNewForPage()
$revision->name = $page->name;
$revision->html = $page->html;      // 完整HTML
$revision->markdown = $page->markdown;  // 完整Markdown
$revision->text = $page->text;      // 完整纯文本
```

#### 6.1.2 为什么不用增量 diff 存储

**优点（当前快照策略）**：
1. **回滚简单**：直接读取某条修订记录即可恢复
2. **diff 灵活**：可以任意两个版本对比，不限于相邻版本
3. **可靠性高**：单条记录损坏不影响其他版本
4. **实现简单**：逻辑直接，bug 少

**缺点**：
1. **存储空间大**：每个版本都是完整拷贝，冗余度高
2. **增量效率低**：小修改也产生完整快照

#### 6.1.3 现有的存储优化手段

BookStack 通过以下方式控制存储体积，而非使用 diff 存储：

1. **版本数量限制**：`config('app.revision_limit')`，超出自动清理
   ```php
   protected function deleteOldRevisions(Page $page): void {
       $revisionLimit = config('app.revision_limit');
       // ... 跳过前 N 条，删除剩余的
   }
   ```

2. **草稿定期失效**：`update_draft` 草稿只检测最近 60 分钟的
3. **用户草稿复用**：同一用户对同一页面只有一个 update_draft（更新而非新增）

#### 6.1.4 DiffStorage 回滚成本对比

**实际状态**：BookStack 没有 `DiffStorage` 类，也不使用增量 diff 存储。以下是两种存储策略的回滚成本对比：

| 维度 | 完整快照存储 (当前) | 增量 Diff 存储 (假设) |
|------|-------------------|---------------------|
| **存储成本** | 高（每份完整拷贝） | 低（仅存储差异） |
| **回滚步骤** | 1步：直接读取目标版本 | N步：从基线版本依次应用 N 个 diff |
| **回滚时间** | O(1) — 恒定时间 | O(N) — 与版本差成正比 |
| **回滚复杂度** | 简单直接 | 复杂，需处理 diff 冲突 |
| **中间版本损坏影响** | 仅损坏版本不可用 | 该版本之后的所有版本都无法回滚 |
| **任意版本对比** | O(1) — 直接对比两个快照 | O(N) — 需先重建两个版本 |

**为什么 BookStack 选择快照存储**：

1. **回滚可靠性**：单条记录损坏不影响其他版本
2. **实现简单**：回滚 = `fill($revision->toArray())` + `save()`
3. **diff 灵活**：可以任意两个版本对比，无需重建
4. **存储成本可接受**：文本数据压缩比高，配合 `revision_limit` 可控

**假设的 DiffStorage 实现复杂度**：

```php
// 伪代码：如果使用增量存储，回滚需要这样做
class DiffStorage {
    public function rollbackTo(Page $page, int $revisionId): Page {
        $revisions = $this->getRevisionsBetween($page->currentRevisionId, $revisionId);
        $content = $page->content;
        
        // 反向应用 diff，N 次操作
        foreach (array_reverse($revisions) as $rev) {
            $content = $this->applyDiffReverse($content, $rev->diff);
        }
        
        $page->content = $content;
        return $page;
    }
}
```

> **结论**：BookStack 用存储成本换取回滚的简单性和可靠性。

### 6.2 回滚副作用分析

#### 6.2.1 直接副作用

执行 `restoreRevision()` 会触发以下连锁反应：

| 副作用 | 触发条件 | 说明 |
|--------|----------|------|
| **搜索索引重建** | 总是触发 | `$page->indexForSearch()` |
| **引用关系更新** | 总是触发 | `referenceStore->updateForEntity()` |
| **Slug 重新生成** | 总是触发 | `refreshSlug()` — 可能导致URL变化 |
| **引用链接批量更新** | URL 变化时触发 | 更新所有引用该页面的链接 |
| **父级重排序** | 总是触发 | `sortParent()` — 可能影响同级页面顺序 |
| **新修订产生** | 总是触发 | 回滚本身产生新版本，revision_count++ |
| **活动日志** | 总是触发 | PAGE_RESTORE + REVISION_RESTORE 两条日志 |

#### 6.2.2 URL 变化的连锁反应

当回滚导致页面标题（slug）变化时，`ReferenceUpdater` 会批量更新所有引用：

- **核心代码**：`ReferenceUpdater::updateEntityReferences()` [app/References/ReferenceUpdater.php:20-30](app/References/ReferenceUpdater.php#L20-L30)

```php
public function updateEntityReferences(Entity $entity, string $oldLink): void
{
    $references = $this->getReferencesToUpdate($entity);
    foreach ($references as $reference) {
        $this->updateReferencesWithinEntity($reference->from, $oldLink, $newLink);
    }
}
```

**影响范围**：
- 更新所有引用该页面的其他页面的 HTML/Markdown
- 更新书籍、章节的描述中的链接
- 每个被修改的页面都会 `revision_count++`，产生新修订
- 新修订的摘要为 "Updated references to page"

> **潜在风险**：回滚一个热门页面（被很多其他页面引用）可能触发大量页面的修订记录增长。

#### 6.2.3 搜索索引回滚一致性

回滚后调用 `indexForSearch()`，确保搜索结果与回滚后内容一致：

- 更新搜索索引中的页面标题、文本内容
- 回滚是立即生效的，搜索结果同步更新

#### 6.2.4 回滚的"不可逆"性

虽然回滚操作本身被记录为新版本（可以"回滚回滚"），但有几点不可逆：

1. **修订记录的创建**：回滚产生的新版本不会消失
2. **活动日志**：审计日志永久记录回滚操作
3. **引用更新**：如果 URL 变化后又变回来，引用页面也会产生两次修订记录
4. **已删除的修订**：如果回滚到的版本之后的修订被手动删除了，回滚后那些内容就找不回来了

### 6.3 cascade soft delete 级联深度

**实际状态**：BookStack 的软删除是**手动递归遍历**实现的，不是数据库层面的 `ON DELETE CASCADE`。

#### 6.3.1 级联关系层级

```
Bookshelf (书架)
    ↓ (无软删除级联，只有 forceDelete 时才级联)
    Book (书籍)
        ↓ (软删除自动级联)
        ├─ Chapter (章节)
        │     ↓ (软删除自动级联)
        │     └─ Page (页面)
        └─ Page (页面)
```

#### 6.3.2 软删除级联实现

- **核心类**：`TrashCan` [app/Entities/Tools/TrashCan.php](app/Entities/Tools/TrashCan.php)
- **最大级联深度**：**4 层**（Bookshelf → Book → Chapter → Page）
- **实现方式**：PHP 代码手动遍历，不是数据库外键约束

```php
// TrashCan.php:49-63 - softDestroyBook 级联删除章节和页面
public function softDestroyBook(Book $book)
{
    $this->ensureDeletable($book);
    Deletion::createForEntity($book);
    
    foreach ($book->pages as $page) {
        $this->softDestroyPage($page, false);  // 级联: Book → Page
    }
    
    foreach ($book->chapters as $chapter) {
        $this->softDestroyChapter($chapter, false);  // 级联: Book → Chapter
    }
    
    $book->delete();
}
```

```php
// TrashCan.php:70-84 - softDestroyChapter 级联删除页面
public function softDestroyChapter(Chapter $chapter, bool $recordDelete = true)
{
    if (count($chapter->pages) > 0) {
        foreach ($chapter->pages as $page) {
            $this->softDestroyPage($page, false);  // 级联: Chapter → Page
        }
    }
    $chapter->delete();
}
```

#### 6.3.3 Bookshelf 的特殊情况

**删除 Bookshelf 不会级联软删除关联的 Books**：

```php
// TrashCan.php:37-42 - softDestroyShelf 不级联
public function softDestroyShelf(Bookshelf $shelf)
{
    $this->ensureDeletable($shelf);
    Deletion::createForEntity($shelf);
    $shelf->delete();  // 只删除书架本身，不级联删除书籍
}
```

**原因**：Bookshelf 和 Book 是多对多关系（一个书籍可以属于多个书架），删除书架不应删除书籍本身。

#### 6.3.4 永久删除（forceDelete）的级联

永久删除时，级联深度更深，包括所有关联数据：

| 级联项 | 处理方式 | 代码位置 |
|--------|---------|---------|
| **活动日志** | 解除关联，保留日志 | `destroyCommonRelations()`:393 |
| **浏览记录** | 物理删除 | `destroyCommonRelations()`:394 |
| **权限** | 物理删除 | `destroyCommonRelations()`:395-398 |
| **标签** | 物理删除 | `destroyCommonRelations()`:396 |
| **评论** | 物理删除 | `destroyCommonRelations()`:397 |
| **联合权限** | 物理删除 | `destroyCommonRelations()`:398 |
| **搜索索引** | 物理删除 | `destroyCommonRelations()`:399 |
| **收藏/关注** | 物理删除 | `destroyCommonRelations()`:401-402 |
| **引用关系** | 物理删除 | `destroyCommonRelations()`:403-404 |
| **Slug历史** | 物理删除 | `destroyCommonRelations()`:405 |
| **封面图片** | 物理删除图片文件 | `destroyCommonRelations()`:407-410 |
| **页面修订** | 物理删除（仅页面） | `destroyPage()`:208 |
| **附件** | 物理删除文件（仅页面） | `destroyPage()`:211-214 |

#### 6.3.5 数据库层级联

数据库迁移中也定义了部分 `ON DELETE CASCADE` 外键约束，主要用于：
- 角色-用户关联表
- 书架-书籍关联表
- 权限表

```sql
-- 示例: database/migrations/2015_08_29_105422_add_roles_and_permissions.php:41
FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
```

> **注意**：数据库层的 CASCADE 仅用于关联表数据清理，核心实体的级联软删除由 PHP 代码控制。

---

## 七、回滚完整流程

### 7.1 回滚入口

- **路由**：`POST /books/{bookSlug}/page/{pageSlug}/revisions/{revisionId}/restore`
- **控制器**：`PageRevisionController::restore()` [app/Entities/Controllers/PageRevisionController.php:135-144](app/Entities/Controllers/PageRevisionController.php#L135-L144)

```php
public function restore(string $bookSlug, string $pageSlug, int $revisionId)
{
    $page = $this->pageQueries->findVisibleBySlugsOrFail($bookSlug, $pageSlug);
    $this->checkOwnablePermission(Permission::PageUpdate, $page);
    $page = $this->pageRepo->restoreRevision($page, $revisionId);
    return redirect($page->getUrl());
}
```

### 7.2 回滚核心实现

- **核心方法**：`PageRepo::restoreRevision()` [app/Entities/Repos/PageRepo.php:229-265](app/Entities/Repos/PageRepo.php#L229-L265)

```php
public function restoreRevision(Page $page, int $revisionId): Page
{
    $oldUrl = $page->getUrl();
    $page->revision_count++;  // 回滚也产生新版本号
    
    /** @var PageRevision $revision */
    $revision = $page->revisions()->where('id', '=', $revisionId)->first();
    
    // 用修订数据填充页面
    $page->fill($revision->toArray());
    $content = new PageContent($page);
    
    // 根据内容类型重新解析
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
    
    // 创建新的修订记录，标记为恢复操作
    $summary = trans('entities.pages_revision_restored_from', [
        'id' => strval($revisionId), 
        'summary' => $revision->summary
    ]);
    $this->revisionRepo->storeNewForPage($page, $summary);
    
    // URL变更时更新引用
    if ($oldUrl !== $page->getUrl()) {
        $this->referenceUpdater->updateEntityReferences($page, $oldUrl);
    }
    
    Activity::add(ActivityType::PAGE_RESTORE, $page);
    Activity::add(ActivityType::REVISION_RESTORE, $revision);
    
    return $page;
}
```

### 5.3 回滚的关键设计决策

#### 5.3.1 回滚 = 新版本
- **设计**：回滚操作不会删除历史，而是创建一个内容等于旧版本的新版本
- **原因**：
  1. 保留完整操作审计轨迹
  2. 回滚操作本身可被追溯和撤销
  3. `revision_count` 持续递增，版本号永不回退

#### 5.3.2 内容重解析
- **设计**：回滚时会重新调用 `setNewMarkdown()` 或 `setNewHTML()`
- **原因**：
  1. 确保 text 字段（纯文本索引）正确生成
  2. 触发HTML净化和安全过滤
  3. 处理可能的格式升级

#### 7.3.3 引用更新
- **设计**：检测URL变化，更新系统内所有引用
- **涉及**：`ReferenceUpdater::updateEntityReferences()`

### 7.4 修订删除限制

- **禁止删除最新修订**：`PageRevisionController::destroy()` [app/Entities/Controllers/PageRevisionController.php:163-167](app/Entities/Controllers/PageRevisionController.php#L163-L167)

```php
if (intval($page->currentRevision->id ?? null) === intval($revId)) {
    $this->showErrorNotification(trans('entities.revision_cannot_delete_latest'));
    return redirect($page->getUrl('/revisions'));
}
```

---

## 八、完整数据链路图

```
用户编辑页面
    ↓
[进入编辑页] PageEditorData::build()
    ├─ 检测并发编辑 → PageEditActivity::hasActiveEditing()
    ├─ 加载用户草稿 → 填充编辑内容
    └─ 生成警告消息
    ↓
[自动保存] saveDraft() (每30秒)
    ├─ PageRepo::updatePageDraft()
    │   ├─ 新页面 → 更新Page.draft=true
    │   └─ 旧页面 → 创建/更新PageRevision.type='update_draft'
    └─ 返回并发警告
    ↓
[正式保存] update()
    ├─ 检测内容变化 (html/name/markdown)
    ├─ page.revision_count++
    ├─ BaseRepo::update() → 保存pages表
    ├─ 删除当前用户的update_draft
    └─ 有变化 → RevisionRepo::storeNewForPage()
        ├─ 创建PageRevision.type='version'
        ├─ revision_number = page.revision_count
        └─ deleteOldRevisions() → 清理超出版本限制的记录
    ↓
[查看历史] revisions.index()
    ↓
[查看差异] changes()
    ├─ getPreviousRevision() → 定位前一版本
    └─ Diff::execute() → HTML差异对比
    ↓
[执行回滚] restore()
    └─ PageRepo::restoreRevision()
        ├─ page.revision_count++
        ├─ 用旧修订数据填充页面
        ├─ 重新解析内容 (setNewMarkdown/HTML)
        ├─ 刷新slug、重建索引、更新引用
        └─ 创建新修订 (标记为恢复操作)
```

---

## 九、关键设计总结

| 设计点 | 实现方式 | 优势/权衡 |
|--------|----------|----------|
| **版本永不回退** | 回滚创建新版本，revision_count单调递增 | 审计完整，操作可追溯 |
| **双轨草稿制** | 页面级草稿 + 修订级草稿 | 新页面和已有页面编辑都有安全网 |
| **完整快照存储** | 每次修订存储完整html/markdown/text | 回滚简单直接，diff灵活，存储空间换可靠性 |
| **警告而非阻塞** | 并发编辑只提示不阻止 | 用户体验好，极端场景不丢数据 |
| **LocalStorage兜底** | AJAX失败时存本地 | 网络异常时保障用户输入 |
| **版本数量限制** | 可配置revision_limit，自动清理旧版本 | 控制数据库体积 |
| **线性历史无分支** | 单时间线，回滚=创建新版本 | 概念简单，用户易理解 |
| **READ COMMITTED事务** | 自定义事务隔离级别 | 权限生成等场景能看到其他已提交变更 |
| **自增ID定位前序** | diff用ID而非revision_number | 避免版本号冲突导致历史断裂 |
| **CRDT实时协作** | Lexical+Yjs（前端层） | 多用户实时编辑无冲突合并 |
| **引用自动更新** | URL变化时批量更新引用页 | 保持链接有效性 |
| **审计日志分离** | activities表独立于revisions表 | 安全审计与内容回滚职责分离 |
| **无savepoint嵌套事务** | 仅单层DatabaseTransaction | 实现简单，依赖Laravel底层自动处理 |
| **Slug先查后生成** | while循环追加随机后缀 | 高并发下有TOCTOU风险，实现简单 |
| **无版本号唯一约束** | page_revisions无复合唯一键 | 允许版本号重复，不影响核心功能 |
| **LCS标准diff算法** | ssddanbrown/htmldiff | 通用HTML比较，短文档足够快，O(n*m)复杂度 |
| **手动遍历级联删除** | TrashCan PHP递归实现 | 灵活可控，最大级联深度4层，无数据库外键约束 |
| **日志事务外写入** | Activity::add()在主事务后 | 主业务不受日志失败影响，但存在审计缺口风险 |
| **无草稿过期清理** | update_draft永久保留除非手动删除 | 用户数据不丢失，但可能产生垃圾数据 |
| **Bookshelf不级联软删** | 多对多关系仅删除关联 | 书籍可属于多个书架，删除书架不影响书籍 |
| **forceDelete深度清理** | 13项关联数据级联清除 | 永久删除彻底，不留残留 |
| **单用户草稿复用** | 同一用户同一页面仅一份草稿 | 减少存储，避免多版本混乱 |
| **无乐观锁版本检查** | 无WHERE version=?更新 | 实现简单，高并发下有丢失更新风险 |
