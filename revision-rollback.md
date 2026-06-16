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

### 1.4 修订存储与清理

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

### 2.3 数据库设计溯源

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

---

## 五、回滚完整流程

### 5.1 回滚入口

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

### 5.2 回滚核心实现

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

#### 5.3.3 引用更新
- **设计**：检测URL变化，更新系统内所有引用
- **涉及**：`ReferenceUpdater::updateEntityReferences()`

### 5.4 修订删除限制

- **禁止删除最新修订**：`PageRevisionController::destroy()` [app/Entities/Controllers/PageRevisionController.php:163-167](app/Entities/Controllers/PageRevisionController.php#L163-L167)

```php
if (intval($page->currentRevision->id ?? null) === intval($revId)) {
    $this->showErrorNotification(trans('entities.revision_cannot_delete_latest'));
    return redirect($page->getUrl('/revisions'));
}
```

---

## 六、完整数据链路图

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

## 七、关键设计总结

| 设计点 | 实现方式 | 优势 |
|--------|----------|------|
| **版本永不回退** | 回滚创建新版本，revision_count单调递增 | 审计完整，操作可追溯 |
| **双轨草稿制** | 页面级草稿 + 修订级草稿 | 新页面和已有页面编辑都有安全网 |
| **完整快照存储** | 每次修订存储完整html/markdown/text | 回滚简单直接，diff灵活 |
| **警告而非阻塞** | 并发编辑只提示不阻止 | 用户体验好，极端场景不丢数据 |
| **LocalStorage兜底** | AJAX失败时存本地 | 网络异常时保障用户输入 |
| **版本数量限制** | 可配置revision_limit，自动清理旧版本 | 控制数据库体积 |
