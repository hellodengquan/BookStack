# BookStack 全文搜索：写入与查询路径详解

## 一、核心数据模型

### 1.1 SearchTerm（搜索词索引表）

文件：`app/Search/SearchTerm.php`

```php
class SearchTerm extends Model
{
    protected $fillable = ['term', 'entity_id', 'entity_type', 'score'];
    public $timestamps = false;

    public function entity()
    {
        return $this->morphTo('entity');
    }
}
```

- **term**: 分词后的关键词
- **entity_id / entity_type**: 多态关联到被索引的实体（Page / Chapter / Book / Bookshelf）
- **score**: 该词在该实体中的相关性评分，评分越高表示越相关

### 1.2 Entity 基类与搜索关联

文件：`app/Entities/Models/Entity.php:251-254`

```php
public function searchTerms(): MorphMany
{
    return $this->morphMany(SearchTerm::class, 'entity');
}
```

每个 Entity 子类通过多态关系 `searchTerms()` 关联到其所有搜索索引词。

---

## 二、写入路径：内容变更 → 同步进索引

### 2.1 触发入口

索引写入有 **5 个主要触发点**：

| 触发场景 | 调用位置 | 说明 |
|---------|---------|------|
| 新建实体 | `app/Entities/Repos/BaseRepo.php:65` | `create()` 方法中调用 `$entity->indexForSearch()` |
| 更新实体 | `app/Entities/Repos/BaseRepo.php:100` | `update()` 方法中调用 `$entity->indexForSearch()` |
| 恢复版本 | `app/Entities/Repos/PageRepo.php:249` | `restoreRevision()` 中单独调用 |
| 彻底删除 | `app/Entities/Tools/TrashCan.php:399` | `destroyCommonRelations()` 中调用 `$entity->searchTerms()->delete()` |
| 命令行重建 | `app/Console/Commands/RegenerateSearchCommand.php` | `bookstack:regenerate-search` 命令 |

#### 触发方法 `indexForSearch()`

文件：`app/Entities/Models/Entity.php:402-405`

```php
public function indexForSearch(): void
{
    app()->make(SearchIndex::class)->indexEntity(clone $this);
}
```

通过服务容器获取 `SearchIndex` 单例，对当前实体进行索引。

---

### 2.2 SearchIndex 核心流程

文件：`app/Search/SearchIndex.php`

#### 2.2.1 `indexEntity()` — 单个实体索引（第 35-40 行）

```php
public function indexEntity(Entity $entity): void
{
    $this->deleteEntityTerms($entity);      // 先删除旧索引
    $terms = $this->entityToTermDataArray($entity);  // 生成新的词-分数据
    $this->insertTerms($terms);              // 批量写入数据库
}
```

**策略**：先删后写，保证索引一致性。

#### 2.2.2 `indexAllEntities()` — 全量重建（第 68-95 行）

```php
public function indexAllEntities(?callable $progressCallback = null): void
{
    SearchTerm::query()->truncate();  // 清空整张表

    foreach ($this->entityProvider->all() as $entityModel) {
        // 按 250 条分批处理，避免内存溢出
        $entityModel->newQuery()
            ->select($selectFields)
            ->with(['tags:id,name,value,entity_id,entity_type'])
            ->chunk(250, $chunkCallback);
    }
}
```

#### 2.2.3 `entityToTermDataArray()` — 生成词-分数数据（第 248-274 行）

这是索引的核心逻辑，将实体内容转化为 `[term, score, entity_id, entity_type]` 数组。

```php
protected function entityToTermDataArray(Entity $entity): array
{
    // 名称：权重最高，×40 × searchFactor
    $nameTermsMap = $this->generateTermScoreMapFromText($entity->name, 40 * $entity->searchFactor);
    
    // 标签：名称 ×3，值 ×5
    $tagTermsMap = $this->generateTermScoreMapFromTags($entity->tags->all());

    // 正文：Page 解析 HTML 权重分级，其他实体解析 description
    if ($entity instanceof Page) {
        $bodyTermsMap = $this->generateTermScoreMapFromHtml($entity->html);
    } else {
        $bodyTermsMap = $this->generateTermScoreMapFromText($entity->getAttribute('description') ?? '', $entity->searchFactor);
    }

    // 合并所有词的分数
    $mergedScoreMap = $this->mergeTermScoreMaps($nameTermsMap, $bodyTermsMap, $tagTermsMap);
    // ...
}
```

**评分权重设计**：

| 来源 | 权重系数 | 说明 |
|------|---------|------|
| 实体名称 (name) | 40 × searchFactor | 标题匹配最重要 |
| 标签名 (tag name) | 3 | 次要匹配 |
| 标签值 (tag value) | 5 | 标签值比标签名更重要 |
| HTML H1 | 10 | 一级标题高权重 |
| HTML H2 | 5 | 二级标题 |
| HTML H3 | 4 | |
| HTML H4 | 3 | |
| HTML H5 | 2 | |
| HTML H6 | 1.5 | |
| 普通正文 | 1 | |
| 容器描述 (description) | 1 × searchFactor | Book/Chapter/Bookshelf 的描述 |

不同实体类型的 `searchFactor` 默认为 1.0。

---

### 2.3 分词算法

文件：`app/Search/SearchIndex.php:204-240`（`textToTermCountMap`）与 `app/Search/SearchTextTokenizer.php`

#### 2.3.1 分隔符定义

```php
public static string $delimiters = " \n\t.-,!?:;()[]{}<>`'\"«»";
public static string $softDelimiters = ".-";
```

- **硬分隔符**：直接切开词
- **软分隔符**（`.`、`-`）：既作为分隔符，也会保留组合词

#### 2.3.2 软分隔符处理示例

对文本 `user-friendly design`：

1. 按硬分隔符切出：`user`、`friendly`、`design`
2. 检测到 `user` 和 `friendly` 之间是软分隔符 `-`，额外保留组合词 `user-friendly`

最终索引：`user`、`friendly`、`design`、`user-friendly`，每个出现频次 1。

#### 2.3.3 SearchTextTokenizer

自定义分词器（替代 `strtok`），跟踪每一步的前一个分隔符，用于软分隔符逻辑判断。

---

### 2.4 HTML 正文解析

文件：`app/Search/SearchIndex.php:141-173`（`generateTermScoreMapFromHtml`）

1. 将 `<br>` 替换为换行符
2. 用 `HtmlDocument`（内部封装 DOMDocument）解析 HTML
3. 遍历 body 的直接子节点
4. 根据节点名称（h1-h6）应用不同权重系数
5. 每个节点的文本内容单独分词计分，分数累加

---

## 三、查询路径：用户输入 → 返回结果

### 3.1 路由入口

Web 路由（`routes/web.php:191-197`）：
```
GET /search                              → SearchController@search           全站搜索
GET /search/book/{bookId}                → SearchController@searchBook       书内搜索
GET /search/chapter/{chapterId}          → SearchController@searchChapter    章节内搜索
GET /search/entity-selector              → SearchController@searchForSelector  选择器搜索
GET /search/entity-selector-templates    → SearchController@templatesForSelector  模板搜索
GET /search/suggest                      → SearchController@searchSuggestions 搜索建议
```

API 路由（`routes/api.php:109`）：
```
GET /api/search                          → SearchApiController@all          API 搜索
```

---

### 3.2 SearchController 流程

文件：`app/Search/SearchController.php:23-45`

```php
public function search(Request $request, SearchResultsFormatter $formatter)
{
    // 1. 解析搜索参数
    $searchOpts = SearchOptions::fromRequest($request);
    
    // 2. 执行搜索
    $results = $this->searchRunner->searchEntities($searchOpts, 'all', $page, $count);
    
    // 3. 格式化结果（高亮关键词）
    $formatter->format($results['results']->all(), $searchOpts);
    
    // 4. 分页 + 渲染视图
    $paginator = new LengthAwarePaginator(...);
    return view('search.all', [...]);
}
```

---

### 3.3 SearchOptions 解析

文件：`app/Search/SearchOptions.php`

将搜索字符串解析为四类选项，存储在 `SearchOptionSet` 中：

| 类别 | 类 | 语法示例 | 说明 |
|------|----|---------|------|
| 普通搜索词 | `TermSearchOption` | `hello world` | 走倒排索引模糊匹配 |
| 精确匹配 | `ExactSearchOption` | `"hello world"` | 对 name/description/text 做 LIKE |
| 标签搜索 | `TagSearchOption` | `[priority=high]` | 关联 tags 表查询 |
| 过滤器 | `FilterSearchOption` | `{created_by:me}` | 各种附加过滤条件 |

#### 3.3.1 语法解析规则（第 108-149 行）

```
正则匹配：
- "..." 或 -"..."         → exacts（精确匹配，可否定）
- [...] 或 -[...]         → tags（标签搜索，可否定）
- {...} 或 -{...}         → filters（过滤器，可否定）
- 剩余空格分隔的普通词    → searches
```

如果普通词中含有硬分隔符（如 `hello,world`），则自动转为精确匹配。

#### 3.3.2 选项数量限制

为防止滥用，根据登录状态限制选项数量：
- 搜索词：游客 5 个 / 登录 10 个
- 精确匹配：游客 2 个 / 登录 4 个
- 标签：游客 4 个 / 登录 8 个
- 过滤器：游客 5 个 / 登录 10 个

---

### 3.4 SearchRunner 核心查询

文件：`app/Search/SearchRunner.php`

#### 3.4.1 `searchEntities()` — 主入口（第 42-62 行）

```php
public function searchEntities(SearchOptions $searchOpts, string $entityType = 'all', int $page = 1, int $count = 20): array
{
    // 确定搜索的实体类型（可被 filters.type 覆盖）
    $entityTypesToSearch = ...;
    
    // 构建查询
    $searchQuery = $this->buildQuery($searchOpts, $entityTypesToSearch);
    
    // 统计总数 + 获取分页数据
    $total = $searchQuery->count();
    $results = $this->getPageOfDataFromQuery($searchQuery, $page, $count);
    
    return ['total' => $total, 'results' => $results->values()];
}
```

#### 3.4.2 `buildQuery()` — 查询构建（第 110-150 行）

```php
protected function buildQuery(SearchOptions $searchOpts, array $entityTypes): EloquentBuilder
{
    // 基础可见性查询（权限过滤）
    $entityQuery = $this->entityQueries->visibleForList()
        ->whereIn('type', $entityTypes);

    // 1. 普通搜索词 → 走 search_terms 倒排索引 + 词频评分
    $this->applyTermSearch($entityQuery, $searchOpts, $entityTypes);

    // 2. 精确匹配 → 在 name/description/text 上做 LIKE
    foreach ($searchOpts->exacts->all() as $exact) { ... }

    // 3. 标签搜索 → whereHas('tags', ...)
    foreach ($searchOpts->tags->all() as $tagOption) {
        $this->applyTagSearch($entityQuery, $tagOption);
    }

    // 4. 过滤器 → 动态方法调用 filterXxx()
    foreach ($searchOpts->filters->all() as $filterOption) {
        $functionName = Str::camel('filter_' . $filterOption->getKey());
        if (method_exists($this, $functionName)) {
            $this->$functionName($entityQuery, ...);
        }
    }

    return $entityQuery;
}
```

---

### 3.5 `applyTermSearch()` — 倒排索引评分查询

文件：`app/Search/SearchRunner.php:155-186`

这是全文搜索最核心的 SQL 构建逻辑。

#### 3.5.1 整体思路

1. 对每个搜索词，计算其 **稀有度调整系数**（越稀有的词权重越高）
2. 在 `search_terms` 表中做前缀匹配 `term LIKE 'xxx%'`
3. 按实体分组，SUM 每个词的匹配分数
4. JOIN 到主实体查询，按总分降序排列

#### 3.5.2 子查询结构

```sql
SELECT 
    entity_id, 
    entity_type,
    SUM(
        IF(term like ?, score * 调整系数, 
        IF(term like ?, score * 调整系数, 0))
    ) as score
FROM search_terms
WHERE term LIKE '词1%' OR term LIKE '词2%' ...
GROUP BY entity_type, entity_id
```

#### 3.5.3 词频稀有度调整

文件：`app/Search/SearchRunner.php:222-275`

```php
protected function getTermAdjustments(SearchOptions $options): array
{
    // 1. 统计每个词在 search_terms 表中出现的次数（前缀匹配）
    // 2. 归一化：相对于出现次数最多的词
    // 3. 计算乘数：multiplier = 1.3 - (count / max_count)
    
    // 最稀有的词： multiplier ≈ 1.3
    // 最常见的词： multiplier ≈ 0.3
}
```

这是一种简化版的 **IDF（逆文档频率）** 实现：稀有词获得更高权重。

---

### 3.6 支持的过滤器

`SearchRunner` 中以 `filterXxx()` 方法形式提供：

| 过滤器键 | 方法 | 功能 |
|---------|------|------|
| `updated_after` | `filterUpdatedAfter` | 更新时间晚于 |
| `updated_before` | `filterUpdatedBefore` | 更新时间早于 |
| `created_after` | `filterCreatedAfter` | 创建时间晚于 |
| `created_before` | `filterCreatedBefore` | 创建时间早于 |
| `created_by` | `filterCreatedBy` | 创建者（slug 或 me） |
| `updated_by` | `filterUpdatedBy` | 更新者 |
| `owned_by` | `filterOwnedBy` | 拥有者 |
| `in_name` / `in_title` | `filterInName` | 仅搜索名称 |
| `in_body` | `filterInBody` | 仅搜索正文 |
| `is_restricted` | `filterIsRestricted` | 是否有权限限制 |
| `viewed_by_me` | `filterViewedByMe` | 我是否看过 |
| `not_viewed_by_me` | `filterNotViewedByMe` | 我是否没看过 |
| `is_template` | `filterIsTemplate` | 是否为模板 |
| `type` | （在 buildQuery 开头处理） | 实体类型过滤 |
| `sort_by` | `filterSortBy` → `sortByLastCommented` 等 | 特殊排序 |

所有过滤器都支持否定形式（前缀 `-`）。

---

### 3.7 SearchResultsFormatter — 结果高亮

文件：`app/Search/SearchResultsFormatter.php`

对搜索结果做以下处理：

1. **关键词定位**：在实体名称和正文中查找所有搜索词的出现位置（最多 25 处）
2. **位置合并**：对重叠或相邻的匹配区域进行合并
3. **摘要截取**：
   - 名称：完整显示，不截断（targetLength=0）
   - 正文：截取约 260 字符，以第一个匹配为中心，前后各留 32 字符上下文
4. **HTML 高亮**：将匹配词用 `<strong>` 标签包裹
5. **省略号**：截断处用 `...` 标示
6. **标签高亮**：如果标签名或值匹配搜索词，打上 `highlight_name` / `highlight_value` 属性

---

## 四、完整流程时序图

### 4.1 写入路径

```
用户创建/更新页面
        ↓
PageRepo::update() / BaseRepo::create() / BaseRepo::update()
        ↓
Entity::indexForSearch()
        ↓
SearchIndex::indexEntity($entity)
  ├─ deleteEntityTerms()          → DELETE FROM search_terms WHERE ...
  ├─ entityToTermDataArray()
  │    ├─ 名称分词 (×40 权重)
  │    ├─ 标签分词 (name×3, value×5)
  │    ├─ 正文分词 (HTML 按 h1-h6 分级权重)
  │    └─ mergeTermScoreMaps()    → 合并各来源分数
  └─ insertTerms()                → INSERT INTO search_terms (chunk 500)
```

### 4.2 查询路径

```
用户访问 /search?term=hello
        ↓
SearchController::search()
        ↓
SearchOptions::fromRequest()
  ├─ 解析 searches / exacts / tags / filters
  └─ 限制选项数量
        ↓
SearchRunner::searchEntities()
        ↓
SearchRunner::buildQuery()
  ├─ visibleForList()             → 基础权限过滤
  ├─ applyTermSearch()            → 倒排索引 + 稀有度评分 + ORDER BY score DESC
  ├─ exacts LIKE 查询
  ├─ applyTagSearch()             → whereHas tags
  └─ filterXxx()                  → 各过滤器
        ↓
COUNT 查询总数 + LIMIT/OFFSET 分页
        ↓
EntityHydrator::hydrate()         → 加载关联数据
        ↓
SearchResultsFormatter::format()  → 关键词高亮 + 摘要生成
        ↓
返回视图 / JSON
```

---

## 五、关键设计要点

1. **自建倒排索引**：不依赖数据库 FULLTEXT 或外部搜索引擎（ES/Solr），直接用 `search_terms` 表实现，部署简单
2. **评分加权**：名称权重远高于正文，标题匹配优先
3. **HTML 语义权重**：H1 > H2 > ... > 正文，利用 HTML 结构信息
4. **软分隔符**：`user-friendly` 同时索引为 `user`、`friendly`、`user-friendly`，兼顾精确和模糊
5. **稀有度调整**：查询时按词频动态调整，类似 IDF
6. **权限前置**：使用 `visibleForList()` scope，在评分前先做权限过滤
7. **批量处理**：索引写入按 500 条 chunk，全量重建按 250 实体 chunk，防止内存和 SQL 超限
8. **先删后写**：单实体索引更新采用 DELETE + INSERT 策略，简单可靠
