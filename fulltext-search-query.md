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

### 2.5 索引重建与增量更新的差异化策略

BookStack 提供了三种索引更新方式，分别适用于不同场景：

#### 2.5.1 三种更新方式对比

| 方式 | 入口方法 | 适用场景 | 策略 | 性能特点 |
|------|---------|---------|------|---------|
| **单实体增量更新** | `indexEntity()` | 单个实体创建/更新/恢复版本时 | DELETE + INSERT（先删后写） | 实时、同步执行 |
| **多实体批量索引** | `indexEntities()` | 批量导入、批量操作 | 先收集所有词数据，再统一批量 INSERT（chunk 500） | 减少数据库交互次数 |
| **全量重建** | `indexAllEntities()` | 首次部署、数据迁移、索引损坏 | TRUNCATE 全表 + 逐实体遍历（chunk 250） + 批量 INSERT | 耗时长、一次性清空 |

#### 2.5.2 单实体增量更新（`indexEntity`）

文件：`app/Search/SearchIndex.php:35-40`

```php
public function indexEntity(Entity $entity): void
{
    $this->deleteEntityTerms($entity);      // DELETE FROM search_terms WHERE entity_id = ? AND entity_type = ?
    $terms = $this->entityToTermDataArray($entity);
    $this->insertTerms($terms);              // INSERT 多条
}
```

**设计权衡**：
- ✅ 实现简单，无需判断哪些词新增/删除
- ✅ 保证索引与内容完全一致
- ❌ 每次更新都有 DELETE + 两次 SELECT（加载 tags）+ INSERT 的开销
- ❌ 更新过程中存在短暂的"索引空窗期"（但在事务内不可见）

#### 2.5.3 多实体批量索引（`indexEntities`）

文件：`app/Search/SearchIndex.php:47-56`

```php
public function indexEntities(array $entities): void
{
    $terms = [];
    foreach ($entities as $entity) {
        $entityTerms = $this->entityToTermDataArray($entity);
        array_push($terms, ...$entityTerms);
    }
    $this->insertTerms($terms);
}
```

**注意**：批量索引方法**不会自动删除旧索引**，调用方需要自己确保实体是新的或已手动清理。这在 `indexAllEntities()` 中通过先 TRUNCATE 来保证。

#### 2.5.4 全量重建（`indexAllEntities`）

文件：`app/Search/SearchIndex.php:68-95`

```php
public function indexAllEntities(?callable $progressCallback = null): void
{
    SearchTerm::query()->truncate();          // 清空整张表

    foreach ($this->entityProvider->all() as $entityModel) {
        // Page 使用 html 字段，其他使用 description 字段
        $indexContentField = $entityModel instanceof Page ? 'html' : 'description';
        
        // 预加载 tags 关联，避免 N+1 查询
        $entityModel->newQuery()
            ->select($selectFields)
            ->with(['tags:id,name,value,entity_id,entity_type'])
            ->withTrashed()                    // 包含已软删除的实体
            ->chunk(250, function (Collection $entities) use (...$chunkCallback) {
                $this->indexEntities($entities->all());  // 批量写入
                // ... 进度回调
            });
    }
}
```

**关键设计细节**：
- **TRUNCATE 而非逐行 DELETE**：速度快，重置自增 ID
- **250 实体 / chunk**：平衡内存占用与数据库交互频率
- **预加载 tags**：使用 `with(['tags'])` 避免 N+1 查询
- **包含软删除实体**：`withTrashed()` 确保回收站中的内容也能被搜索到（但权限过滤后用户可能看不到）
- **字段差异化**：Page 索引 `html`，其他实体索引 `description`
- **回调报告进度**：支持传入 `$progressCallback`，命令行场景下用于输出进度

#### 2.5.5 写入批处理（`insertTerms`）

文件：`app/Search/SearchIndex.php:110-116`

```php
protected function insertTerms(array $terms): void
{
    $chunkedTerms = array_chunk($terms, 500);
    foreach ($chunkedTerms as $termChunk) {
        SearchTerm::query()->insert($termChunk);
    }
}
```

**为什么是 500 条？**
- MySQL 的 `max_allowed_packet` 默认 4MB，500 条 INSERT 语句的数据量远低于此限制
- 单条 INSERT 多条数据比多条 INSERT 单条数据快 5-10 倍
- 500 是经验值，兼顾性能与数据库锁持有时间

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

#### 3.5.4 `selectForScoredTerms()` — IF 链构建原理

文件：`app/Search/SearchRunner.php:196-213`

这个方法负责构建评分计算的 SQL 表达式，使用了**反向构建 IF 链**的巧妙设计。

```php
protected function selectForScoredTerms(array $scoredTerms): array
{
    $ifChain = '0';
    $bindings = [];
    foreach ($scoredTerms as $term => $score) {
        $ifChain = 'IF(term like ?, score * ' . (float) $score . ', ' . $ifChain . ')';
        $bindings[] = $term . '%';
    }

    return [
        'statement' => 'SUM(' . $ifChain . ') as score',
        'bindings'  => array_reverse($bindings),
    ];
}
```

**构建过程示例**（搜索词：`cat`、`dog`，稀有度系数分别为 1.2、0.8）：

初始：`$ifChain = '0'`

第 1 轮（term=cat, score=1.2）：
```
$ifChain = 'IF(term like "cat%", score * 1.2, 0)'
```

第 2 轮（term=dog, score=0.8）：
```
$ifChain = 'IF(term like "dog%", score * 0.8, IF(term like "cat%", score * 1.2, 0))'
```

最终 SQL：
```sql
SUM(
    IF(term like "dog%", score * 0.8, 
    IF(term like "cat%", score * 1.2, 0))
) as score
```

**为什么反向构建？**
- 每个 term 行只会匹配第一个符合条件的 IF 分支（因为 IF 是短路求值的）
- 但实际上每个 term 只匹配一个搜索词前缀，所以顺序不影响结果
- 使用 IF 链而不是 CASE WHEN 的原因是：MySQL 中 IF 嵌套在某些场景下性能略好，且代码构建更简洁

**绑定参数反转**：
- 构建时是正向遍历（cat → dog），但最内层 IF 先绑定 cat
- 所以 `$bindings` 需要 `array_reverse`，使绑定顺序与 SQL 中 `?` 出现顺序一致（dog → cat）

#### 3.5.5 相关度评分公式详解

最终相关度分数由三部分相乘得到：

```
最终得分 = Σ(基础分 × 位置权重 × 词频稀有度系数)
```

**1. 基础分（索引时计算，存入 search_terms.score）**

| 来源 | 计算公式 | 示例 |
|------|---------|------|
| 名称 | 词出现次数 × 40 × searchFactor | "Cat cat" 在 name 中 → 2 × 40 = 80 |
| 标签名 | 词出现次数 × 3 | 标签名 "Category" → 1 × 3 = 3 |
| 标签值 | 词出现次数 × 5 | 标签值 "cats" → 1 × 5 = 5 |
| H1 标题 | 词出现次数 × 10 | `<h1>Cat page</h1>` → 1 × 10 = 10 |
| H2 标题 | 词出现次数 × 5 | `<h2>Cat info</h2>` → 1 × 5 = 5 |
| 普通正文 | 词出现次数 × 1 | `<p>cat cat cat</p>` → 3 × 1 = 3 |
| 容器描述 | 词出现次数 × 1 × searchFactor | Book 的 description |

**2. 词频稀有度系数（查询时计算，动态调整）**

```
系数 = 1.3 - (该词匹配的 term 行数 / 最大匹配行数)
```

- 最稀有词：系数 ≈ **1.3**（几乎只出现一次）
- 最常见词：系数 ≈ **0.3**（出现次数最多）
- 作用范围：0.3 ~ 1.3，相差约 4.3 倍

**3. 匹配方式**

采用**前缀匹配**（`term LIKE 'xxx%'`）而非精确匹配：
- 搜索 `cat` 可以匹配到 `cat`、`cats`、`category`、`cat-like` 等
- 这也是软分隔符组合词（如 `user-friendly`）能被搜索到的原因
- 前缀匹配可以利用数据库索引，速度快于 `%xxx%` 的中缀匹配

#### 3.5.6 排序规则

**默认排序**：相关度降序（`ORDER BY score DESC`）

文件：`app/Search/SearchRunner.php:185`
```php
$entityQuery->orderBy('score', 'desc');
```

**自定义排序**：通过 `{sort_by:xxx}` 过滤器指定

目前仅支持一种自定义排序：
- `{sort_by:last_commented}` — 按最后评论时间排序

文件：`app/Search/SearchRunner.php:431-440`
```php
protected function sortByLastCommented(EloquentBuilder $query, bool $negated)
{
    // 自连接找出每个实体的最新评论
    $commentQuery = DB::raw('(SELECT c1.commentable_id, ...) as comments');
    $query->join($commentQuery, ...)
          ->orderBy('last_commented', $negated ? 'asc' : 'desc');
}
```

> **注意**：当使用 `sort_by` 时，会覆盖默认的相关度排序。目前没有"相关度 + 时间"的组合排序。

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

### 3.7 SearchResultsFormatter — 结果高亮与片段提取

文件：`app/Search/SearchResultsFormatter.php`

#### 3.7.1 整体处理流程

```
format($results, $options)
        ↓
setSearchPreview($entity, $options)
  ├─ getMatchPositions()           → 找出所有匹配位置
  ├─ sortAndMergeMatchPositions()  → 排序并合并重叠/相邻的匹配
  └─ formatTextUsingMatchPositions() → 生成高亮摘要
        ↓
highlightTagsContainingTerms()     → 标签高亮标记
```

#### 3.7.2 `getMatchPositions()` — 关键词定位

文件：`app/Search/SearchResultsFormatter.php:85-103`

```php
protected function getMatchPositions(string $text, array $terms): array
{
    $matchRefs = [];
    $text = mb_strtolower($text);

    foreach ($terms as $term) {
        $offset = 0;
        $term = mb_strtolower($term);
        $pos = mb_strpos($text, $term, $offset);
        while ($pos !== false && count($matchRefs) < 25) {
            $end = $pos + mb_strlen($term);
            $matchRefs[$pos] = $end;
            $offset = $end;
            $pos = mb_strpos($text, $term, $offset);
        }
    }

    return $matchRefs;
}
```

**核心要点**：
- **大小写不敏感**：先将文本和搜索词都转为小写
- **多字节安全**：全程使用 `mb_strpos`、`mb_strlen`，支持中文等多字节字符
- **位置数组格式**：`[startIndex => endIndex]`，键是起始位置，值是结束位置
- **最多 25 个匹配**：防止长文本中匹配过多导致性能问题
- **遍历所有搜索词**：精确匹配词（exacts）和普通搜索词（searches）都会被查找

#### 3.7.3 `sortAndMergeMatchPositions()` — 位置合并

文件：`app/Search/SearchResultsFormatter.php:113-132`

```php
protected function sortAndMergeMatchPositions(array $matchPositions): array
{
    ksort($matchPositions);
    $mergedRefs = [];
    $lastStart = 0;
    $lastEnd = 0;

    foreach ($matchPositions as $start => $end) {
        if ($start > $lastEnd) {
            $mergedRefs[$start] = $end;       // 不重叠，新增
            $lastStart = $start;
            $lastEnd = $end;
        } elseif ($end > $lastEnd) {
            $mergedRefs[$lastStart] = $end;   // 重叠/相邻，合并（扩展结束位置）
            $lastEnd = $end;
        }
    }

    return $mergedRefs;
}
```

**合并逻辑示例**：

假设搜索词为 `cat` 和 `category`，在文本中的匹配位置：
```
"the cat category"
      ↑   ↑
      4-7 (cat)
          8-16 (category)
```

合并前：`[4 => 7, 8 => 16]`
合并后：`[4 => 16]`（相邻，合并为一个高亮区域）

**三种情况处理**：
1. `start > lastEnd` → 不重叠，新增区域
2. `start <= lastEnd && end > lastEnd` → 部分重叠或相邻，扩展当前区域
3. `end <= lastEnd` → 完全包含，忽略

#### 3.7.4 `formatTextUsingMatchPositions()` — 核心片段提取与高亮

文件：`app/Search/SearchResultsFormatter.php:143-236`

这是整个格式化最复杂的函数，单次遍历所有匹配位置，时间复杂度 O(n)。

**关键参数**：

| 参数 | 名称 | 值（名称） | 值（正文） | 说明 |
|------|------|-----------|-----------|------|
| `$targetLength` | 目标长度 | 0（完整显示） | 260 | 0 表示不截断 |
| `$contextLength` | 上下文长度 | 0 | 32 | 每个匹配前后保留的字符数 |
| `$fetchAll` | 是否获取全部 | true | false | 由 targetLength === 0 决定 |

**执行流程**：

```
1. 初始化变量
   ├─ maxEnd = 文本总长度
   ├─ fetchAll = (targetLength === 0)
   └─ contextLength = fetchAll ? 0 : 32

2. 遍历每个匹配位置：
   对每个 [start => end]：
   ├─ 计算上下文范围：
   │   contextStart = max(start - contextLength, 0, lastEnd)
   │   contextEnd   = min(end + contextLength, maxEnd)
   │
   ├─ 处理重叠（如果当前匹配与上一个重叠）：
   │   回退已生成的内容，避免重复
   │
   ├─ 添加省略号或填充间隙：
   │   - fetchAll=false 且有间隙 → 添加 " ..."
   │   - fetchAll=true → 填充间隙文本（完整保留）
   │
   ├─ 拼接内容：
   │   上下文前缀（普通文本） + <strong>匹配文本</strong> + 上下文后缀（普通文本）
   │   所有普通文本都经过 e() 转义，防止 XSS
   │
   ├─ 更新 lastEnd = contextEnd
   │
   └─ 达到目标长度（targetLength - 10）时 break

3. 后处理：
   ├─ 如果没有匹配 → 截取前 targetLength 字符
   ├─ 末尾长度不足 → 向后补充（padEndLength）
   ├─ 长度仍不足且不是开头 → 向前补充（padStart），前面加 "..."
   └─ 不是末尾 → 结尾加 "..."
```

**前后补全策略**：

当高亮内容总长度不足 260 字符时，按以下优先级补全：
1. 优先**向后补全**（向文本末尾方向扩展）
2. 仍然不足时**向前补全**（向文本开头方向扩展）
3. 向前补全时，如果不是从文本开头开始，前面加 `...`

**示例**（目标长度 260，实际高亮内容 200 字符）：
```
向后补 40 字符 → 还缺 20 字符 → 向前补 20 字符
结果："...[向前20字符]...[高亮200字符][向后40字符]..."
```

#### 3.7.5 XSS 防护细节

所有普通文本输出都经过 Laravel 的 `e()` 函数（即 `htmlspecialchars`）转义：

```php
$content .= e(mb_substr($originalText, $contextStart, $start - $contextStart));
$content .= '<strong>' . e(mb_substr($originalText, $start, $end - $start)) . '</strong>';
```

**注意**：`<strong>` 标签是硬编码的，不经过转义，这是有意的设计。最终返回的字符串用 `HtmlString` 包装，告知 Blade 模板不要再次转义：

```php
$entity->setAttribute($attributeName, new HtmlString($formatted));
```

#### 3.7.6 标签高亮

文件：`app/Search/SearchResultsFormatter.php:58-76`

```php
protected function highlightTagsContainingTerms(array $tags, array $terms): void
{
    foreach ($tags as $tag) {
        $tagName = mb_strtolower($tag->name);
        $tagValue = mb_strtolower($tag->value);

        foreach ($terms as $term) {
            $termLower = mb_strtolower($term);
            if (mb_strpos($tagName, $termLower) !== false) {
                $tag->setAttribute('highlight_name', true);
            }
            if (mb_strpos($tagValue, $termLower) !== false) {
                $tag->setAttribute('highlight_value', true);
            }
        }
    }
}
```

- 检查标签名和标签值是否包含搜索词（任意位置，非前缀）
- 设置 `highlight_name` / `highlight_value` 属性供前端样式使用
- 使用 `mb_strpos` 支持中文匹配

---

### 3.8 分页实现与游标稳定性

#### 3.8.1 分页参数解析

文件：`app/Search/SearchController.php:23-45`

```php
public function search(Request $request, SearchResultsFormatter $formatter)
{
    $page = intval($request->input('page', '0')) ?: 1;
    $count = setting()->getInteger('lists-page-count-search', 18, 1, 1000);
    
    $results = $this->searchRunner->searchEntities($searchOpts, 'all', $page, $count);
    // ...
}
```

**参数说明**：
- `page`：从 query string 读取，默认 1（0 会被转为 1）
- `count`：从系统设置读取，默认 18，范围 1-1000，可在后台配置
- 总数统计独立查询，用于生成分页器

#### 3.8.2 SQL 分页实现

文件：`app/Search/SearchRunner.php:94-104`

```php
protected function getPageOfDataFromQuery(EloquentBuilder $query, int $page, int $count): Collection
{
    $entities = $query->clone()
        ->skip(($page - 1) * $count)
        ->take($count)
        ->get();

    $hydrated = $this->entityHydrator->hydrate($entities->all(), true, true);
    return collect($hydrated);
}
```

**生成的 SQL**：
```sql
SELECT ... ORDER BY score DESC LIMIT 18 OFFSET 0;    -- 第1页
SELECT ... ORDER BY score DESC LIMIT 18 OFFSET 18;   -- 第2页
SELECT ... ORDER BY score DESC LIMIT 18 OFFSET 36;   -- 第3页
```

这是标准的 **LIMIT + OFFSET** 分页，不是游标（cursor）分页。

#### 3.8.3 分页器构建

文件：`app/Search/SearchController.php:32-34`

```php
$paginator = new LengthAwarePaginator($results['results'], $results['total'], $count, $page);
$paginator->setPath(url('/search'));
$paginator->appends($request->except('page'));  // 保留其他查询参数
```

使用 Laravel 的 `LengthAwarePaginator`，需要提前知道总数，所以会执行两次查询：
1. `SELECT COUNT(*) ...` — 统计总数
2. `SELECT ... LIMIT ... OFFSET ...` — 获取当前页数据

#### 3.8.4 游标稳定性问题

**游标稳定性**（Cursor Stability）是指：当用户在分页浏览时，如果数据库中的数据发生变化（新增、删除、修改分数），后续页面的结果是否会出现**重复**或**遗漏**。

BookStack 的分页**存在游标稳定性问题**，具体表现：

| 场景 | 问题表现 | 原因 |
|------|---------|------|
| **新增高相关度实体** | 翻页时第2页开头可能重复第1页末尾的条目 | 新实体挤入第1页，原有条目整体后移一位 |
| **删除已浏览的实体** | 翻页时第2页跳过一条本该出现的条目 | 被删条目消失，后续条目整体前移一位 |
| **实体分数变化** | 条目可能从第1页跳到第3页，或反之 | ORDER BY score 的排序顺序变化 |
| **并发索引更新** | 同一实体可能在两页重复出现，或完全消失 | 先删后写的索引更新过程中，评分排序不稳定 |

**为什么没有稳定排序？**

为了游标稳定，通常需要增加**二级排序键**（如 `ORDER BY score DESC, id ASC`），保证相同分数的条目顺序固定。但 BookStack 目前只有：

```php
$entityQuery->orderBy('score', 'desc');  // app/Search/SearchRunner.php:185
```

**缺少二级排序键**。如果两条结果分数相同，数据库返回顺序是不确定的（取决于查询计划、索引、数据物理存储顺序等），可能在不同分页查询中返回不同顺序。

#### 3.8.5 各接口的分页差异

| 接口 | 分页方式 | 分页大小 | 说明 |
|------|---------|---------|------|
| `/search`（全站） | LIMIT/OFFSET | 配置值（默认18） | 支持翻页 |
| `/search/book/{id}`（书内） | LIMIT/OFFSET | 固定 20 | 取第1页，不支持翻页 |
| `/search/chapter/{id}`（章节内） | LIMIT/OFFSET | 固定 20 | 取第1页，不支持翻页 |
| `/search/suggest`（搜索建议） | LIMIT/OFFSET | 固定 5 | 取第1页，然后在 PHP 中再 `slice(0, 5)` |
| `/api/search`（API） | LIMIT/OFFSET | 配置值（默认20） | 支持翻页 |

#### 3.8.6 游标分页 vs 偏移分页对比

| 维度 | BookStack 当前方案（OFFSET） | 游标分页（WHERE id > ?） |
|------|---------------------------|----------------------|
| **实现复杂度** | 简单 | 需要稳定排序 + 游标编码 |
| **深翻页性能** | 差（OFFSET 10000 需要扫描 10000 行） | 好（直接从游标位置开始） |
| **游标稳定性** | 差（并发更新时重复/丢失） | 好（基于上次看到的最后一条） |
| **跳页支持** | 支持（直接跳第N页） | 不支持（只能上一页/下一页） |
| **总数获取** | 需要（LengthAwarePaginator） | 不需要（只需知道是否有下一页） |

**设计权衡**：BookStack 选择偏移分页，主要因为实现简单、支持跳页，且知识库搜索场景下深翻页需求不多（用户通常只看前几页）。

---

### 3.9 搜索词频调整缓存与失效策略

#### 3.9.1 唯一的缓存：词频调整系数缓存

BookStack 全文搜索中**只有一处缓存**：词频稀有度调整系数的请求内缓存。

文件：`app/Search/SearchRunner.php:24-35`

```php
/**
 * Retain a cache of score-adjusted terms for specific search options.
 */
protected WeakMap $termAdjustmentCache;

public function __construct(
    protected EntityProvider $entityProvider,
    protected EntityQueries $entityQueries,
    protected EntityHydrator $entityHydrator,
) {
    $this->termAdjustmentCache = new WeakMap();
}
```

缓存使用的是 PHP 原生的 `WeakMap`，以 `SearchOptions` 对象作为 key。

#### 3.9.2 缓存读取与写入

文件：`app/Search/SearchRunner.php:222-251`

```php
protected function getTermAdjustments(SearchOptions $options): array
{
    if (isset($this->termAdjustmentCache[$options])) {
        return $this->termAdjustmentCache[$options];
    }

    $termQuery = SearchTerm::query()->toBase();
    // ... 组装 WHERE 条件 ...
    $termCounts = $termQuery->pluck('count', 'term')->toArray();
    $adjusted = $this->rawTermCountsToAdjustments($termCounts);

    $this->termAdjustmentCache[$options] = $adjusted;

    return $this->termAdjustmentCache[$options];
}
```

**缓存内容**：每个搜索词对应的稀有度调整系数（float 值）

**缓存 key**：`SearchOptions` 对象实例

#### 3.9.3 缓存的作用范围

| 维度 | 说明 |
|------|------|
| **生命周期** | 请求级（单次 HTTP 请求内有效） |
| **存储位置** | PHP 内存（`WeakMap` 对象） |
| **缓存粒度** | 以 `SearchOptions` 对象为粒度 |
| **缓存命中** | 同一请求中相同搜索选项重复调用时命中 |

**为什么只有请求级缓存？**
- `SearchRunner` 是单例（或服务容器中的共享实例）
- 但 PHP 是共享-nothing 架构，请求结束后所有内存释放
- 没有使用 Redis / Memcached / 文件缓存等跨请求缓存

#### 3.9.4 WeakMap 的特性与失效机制

`WeakMap` 是 PHP 8.0+ 引入的弱引用 Map：

- **弱引用 key**：当 `SearchOptions` 对象没有其他引用时，会被垃圾回收，对应条目自动从 Map 中移除
- **内存自动管理**：不需要手动清理缓存
- **对象相等性**：使用对象身份（`===`）作为 key，不是值相等

**潜在问题**：
- 相同的搜索字符串，如果创建了两个不同的 `SearchOptions` 对象，缓存不会命中
- 但 `SearchOptions::fromString()` / `fromRequest()` 每次调用都返回新对象
- 实际上在 `searchEntities()` 方法中，`buildQuery()` 会调用一次 `getTermAdjustments()`，如果没有其他地方重复调用，缓存命中率可能很低

#### 3.9.5 没有缓存的部分

以下内容**完全没有缓存**，每次搜索都实时计算：

| 组件 | 是否缓存 | 计算成本 |
|------|---------|---------|
| 词频调整系数 | ⚠️ 请求内 WeakMap | 中等（search_terms 表 GROUP BY 查询） |
| 搜索结果总数 | ❌ 无 | 高（需要 COUNT 全表扫描 JOIN） |
| 搜索结果列表 | ❌ 无 | 高（倒排索引 JOIN + 排序 + 分页） |
| 结果高亮与摘要 | ❌ 无 | 低（PHP 字符串处理） |
| 搜索建议 | ❌ 无 | 中（走正常搜索逻辑） |

#### 3.9.6 缓存失效策略

由于只有请求级缓存，**没有专门的失效策略**：

- **请求结束**：PHP 进程释放内存，缓存自然失效
- **对象销毁**：`SearchOptions` 对象被 GC 时，WeakMap 条目自动移除
- **索引更新**：不需要考虑，因为每次请求都是独立的，新请求会重新查询

**缺失的缓存场景**：
- 热门搜索词结果缓存（如首页展示"热门搜索"）
- 相同搜索词的结果缓存（多人搜索相同内容）
- 搜索建议缓存
- 标签/作者聚合数据缓存

#### 3.9.7 设计权衡

| 优点 | 缺点 |
|------|------|
| 实现简单，零外部依赖 | 缓存效率低（仅同请求内有效） |
| 无需考虑缓存失效逻辑 | 热门搜索词无法复用结果 |
| 数据始终最新，无一致性问题 | 高并发场景下数据库压力大 |
| WeakMap 自动内存管理 | 相同搜索字符串但不同对象不命中 |

> 这是**极简缓存策略**，符合 BookStack "部署简便、零依赖"的设计哲学。对于中小规模知识库，数据库查询通常不是瓶颈。

---

### 3.10 搜索结果聚合（分面搜索）能力分析

#### 3.10.1 核心事实：无动态分面聚合

BookStack 的搜索结果页面**没有分面搜索（Faceted Search）**功能，即不会根据当前搜索结果动态统计并展示：
- 按标签聚合（每个标签有多少结果）
- 按作者聚合（每个作者有多少结果）
- 按实体类型聚合（每个类型有多少结果）
- 按日期聚合（时间分布）

侧边栏的高级搜索选项是**静态过滤条件**，不是**动态聚合结果**。

#### 3.10.2 侧边栏高级搜索 vs 分面搜索

| 特性 | 侧边栏高级搜索 | 分面搜索（Faceted Search） |
|------|-------------|------------------------|
| **数据来源** | 静态表单选项 | 从当前搜索结果动态统计 |
| **计数显示** | 无（只显示选项名） | 有（每个选项后显示结果数） |
| **动态更新** | 否（选项固定） | 是（每次搜索后重新统计） |
| **用户交互** | 选择条件 → 搜索 | 搜索 → 看到各维度分布 → 点击筛选 |
| **示例** | 类型：页面 书籍 章节 书架 | 类型：页面(120) 书籍(45) 章节(38) 书架(12) |

搜索页面侧边栏（`resources/views/search/all.blade.php`）提供的选项：

```
高级搜索
├─ 搜索词 [输入框]
├─ 内容类型 [x] 页面 [x] 章节 [x] 书籍 [x] 书架
├─ 精确匹配 [添加]
├─ 标签 [添加]
├─ 搜索选项
│   ├─ [ ] 我看过的
│   ├─ [ ] 我没看过的
│   ├─ [ ] 有权限限制的
│   ├─ [ ] 我创建的
│   ├─ [ ] 我更新的
│   └─ [ ] 我拥有的
└─ 日期选项
    ├─ 更新于之后
    ├─ 更新于之前
    ├─ 创建于之后
    └─ 创建于之前
```

这些都是**输入控件**，不是**结果聚合统计**。

#### 3.10.3 已有的标签聚合能力（非搜索场景）

虽然搜索结果没有聚合，但 `TagRepo` 提供了标签使用统计功能（用于标签列表页）：

文件：`app/Activity/TagRepo.php:24-79`

```php
public function queryWithTotalsForList(SimpleListOptions $listOptions, string $nameFilter): Builder
{
    $query = $this->baseQueryWithTotals($nameFilter, $searchTerm)
        ->orderBy($sort, $listOptions->getOrder());

    return $this->permissions->restrictEntityRelationQuery($query, 'tags', 'entity_id', 'entity_type');
}
```

统计维度包括：
- `usages`：总使用次数
- `page_count`：页面使用次数
- `chapter_count`：章节使用次数
- `book_count`：书籍使用次数
- `shelf_count`：书架使用次数
- `values`：不同值的数量

**但这些是全局统计，不是针对当前搜索结果的聚合。**

#### 3.10.4 标签建议功能

`TagRepo` 还提供标签名称和值的建议（用于标签输入时的自动补全）：

文件：`app/Activity/TagRepo.php:85-127`

```php
public function getNameSuggestions(string $searchTerm): Collection
{
    $query = Tag::query()
        ->select('*', DB::raw('count(*) as count'))
        ->groupBy('name');

    if ($searchTerm) {
        $query = $query->where('name', 'LIKE', $searchTerm . '%')->orderBy('name', 'asc');
    } else {
        $query = $query->orderBy('count', 'desc')->take(50);
    }
    // ... 权限过滤
    return $query->pluck('name');
}
```

- 有搜索词时：前缀匹配 + 按名称排序
- 无搜索词时：按使用次数降序取前 50（热门标签）
- 都经过权限过滤（只统计用户可见实体的标签）

#### 3.10.5 缺失的聚合能力清单

| 聚合维度 | 实现状态 | 业务价值 |
|---------|---------|---------|
| 按实体类型聚合 | ❌ 未实现 | 中（快速了解结果分布） |
| 按标签聚合 | ❌ 未实现 | 高（标签云、标签筛选） |
| 按作者聚合 | ❌ 未实现 | 中（找到领域专家） |
| 按创建/更新时间聚合 | ❌ 未实现 | 低（时间分布） |
| 按书架/书籍聚合 | ❌ 未实现 | 中（定位到具体书籍） |

#### 3.10.6 设计权衡与扩展思路

**为什么不做分面搜索？**
- 实现复杂：每个维度都需要额外的 GROUP BY 查询
- 性能开销大：搜索本来就慢，再加多个聚合查询更慢
- 需求不强：知识库搜索通常关键词足够精准
- 已有替代：侧边栏的静态过滤器 + 标签搜索语法 `[tag=value]`

**如果需要实现分面搜索，扩展思路**：

```php
// 伪代码：在 searchEntities() 中增加聚合查询
$facets = [
    'types'  => $searchQuery->clone()->groupBy('type')->select('type', DB::raw('count(*) as count'))->pluck('count', 'type'),
    'tags'   => $searchQuery->clone()->with('tags')->groupBy('tags.name')->...  // 更复杂
    'authors' => $searchQuery->clone()->groupBy('created_by')->select('created_by', DB::raw('count(*) as count'))->...
];
```

更推荐的方案是集成专用搜索引擎（ES/Meilisearch），它们原生支持分面聚合。

---

### 3.11 权限过滤与结果裁剪的协同作用

#### 3.11.1 权限过滤的位置：前置过滤

BookStack 的搜索权限过滤是**前置的**（在评分和排序之前），而不是后置的（查出结果后再过滤）。

```
搜索执行顺序：
1. 基础权限过滤（visibleForList）   ← 先过滤掉无权查看的实体
2. 倒排索引评分（applyTermSearch）    ← 只对有权限的实体评分
3. 其他过滤条件（exacts / tags / filters）
4. 排序（ORDER BY score DESC）
5. 分页（LIMIT / OFFSET）
6. 结果高亮与格式化
```

**入口点**：

文件：`app/Search/SearchRunner.php:110-113`

```php
protected function buildQuery(SearchOptions $searchOpts, array $entityTypes): EloquentBuilder
{
    $entityQuery = $this->entityQueries->visibleForList()
        ->whereIn('type', $entityTypes);
    // ...
}
```

`visibleForList()` 返回的是已经应用了 `visible` scope 的查询。

#### 3.11.2 `visible` scope 的实现

文件：`app/Entities/Models/EntityTable.php:29-32`

```php
public function scopeVisible(Builder $query): Builder
{
    return app()->make(PermissionApplicator::class)->restrictEntityQuery($query);
}
```

#### 3.11.3 `restrictEntityQuery()` 权限过滤逻辑

文件：`app/Permissions/PermissionApplicator.php:99-111`

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

**核心逻辑**：
- 通过 `joint_permissions` 表（预计算的联合权限表）进行权限过滤
- 只返回用户角色有权限查看的实体
- 考虑了"所有者"权限（自己创建的内容即使角色无权限也能看到）

#### 3.11.4 联合权限表（joint_permissions）

`joint_permissions` 是预计算的权限表，将角色权限、继承权限、所有者权限等预先计算好，避免每次查询都递归计算。

**权限继承链**：
```
书架 → 书籍 → 章节 → 页面
```

页面的权限受以下层级影响（优先级从高到低）：
1. 页面自身的显式权限
2. 章节的权限（继承）
3. 书籍的权限（继承）
4. 书架的权限（继承）
5. 角色的默认权限

`EntityPermissionEvaluator` 负责单实体的权限评估，而 `joint_permissions` 表是预计算结果，用于列表查询时的快速过滤。

#### 3.11.5 草稿页面的特殊处理

除了通用的实体权限过滤，页面还有草稿状态的额外过滤：

文件：`app/Permissions/PermissionApplicator.php:117-126`

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

规则：
- 非草稿页面：所有人可见（受通用权限约束）
- 草稿页面：只有所有者可见

#### 3.11.6 软删除的过滤

Entity 模型使用了 `SoftDeletes` trait，默认查询会自动过滤已软删除的实体：

文件：`app/Entities/Models/EntityTable.php:22`

```php
use SoftDeletes;
```

这意味着：
- 回收站中的实体默认不出现在搜索结果中
- 除非显式调用 `withTrashed()`

但在索引重建时，`indexAllEntities()` 使用了 `withTrashed()`，也就是说**回收站中的实体也在索引中**。但搜索查询时 `SoftDeletes` 全局 scope 会自动过滤掉，所以用户看不到。

#### 3.11.7 权限过滤对结果的影响

**权限过滤与搜索结果裁剪的协同作用**体现在以下方面：

| 维度 | 影响 |
|------|------|
| **总数统计** | `COUNT(*)` 是权限过滤后的数量，用户看到的是"我能看到的结果数"，不是系统总结果数 |
| **评分排序** | 只对有权限的实体评分排序，不会出现"高相关度但无权限"的结果排在前面 |
| **分页** | 分页也是基于有权限的结果集，不会出现整页无权限的空页 |
| **索引大小** | 索引包含所有实体（包括无权和已删除的），但查询时过滤 |
| **性能** | 权限 JOIN 增加了查询复杂度，但比分页后再过滤高效得多 |

#### 3.11.8 前置过滤 vs 后置过滤对比

| 方案 | BookStack 当前（前置） | 后置过滤（先查后过滤） |
|------|---------------------|-------------------|
| **性能** | 较好（数据库层面过滤，可利用索引） | 差（查出很多无权限结果后丢弃） |
| **分页准确性** | 好（基于过滤后的结果分页） | 差（可能某一页过滤后为空） |
| **总数准确性** | 好（COUNT 就是过滤后的数量） | 差（总数可能远大于实际可见数） |
| **评分公平性** | 好（只在有权限的结果中评分排序） | 差（高相关但无权限的条目会占用排名） |
| **实现复杂度** | 较高（需要权限 JOIN） | 低（查出后循环判断） |
| **深翻页性能** | 较好 | 很差（可能需要翻很多页才能凑够一页有效结果） |

BookStack 选择前置过滤是正确的设计，保证了搜索结果的可用性和性能。

#### 3.11.9 特殊过滤器与权限的交互

一些过滤器与权限系统有关联：

- **`{is_restricted}`**：筛选有权限限制的实体（即显式设置过权限的）
- **`{created_by:me}` / `{owned_by:me}`**：筛选自己创建/拥有的实体
- **`{viewed_by_me}`**：筛选自己看过的实体

这些过滤器与权限过滤是 **AND 关系**，在权限过滤的基础上进一步缩小范围。

例如搜索 `{is_restricted} 文档` 的逻辑是：
```
我有权限查看的  AND  设置了权限限制的  AND  包含"文档"关键词
```

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

---

## 六、中文分词与英文 token 化的混合处理

### 6.1 核心事实：无专用中文分词器

BookStack 的搜索系统**没有集成任何中文分词库**（如 Jieba、IK Analyzer、SCWS 等），采用的是**基于分隔符的统一 token 化方案**，对中英文混合内容做相同处理。

文件：`app/Search/SearchTextTokenizer.php:22`
```php
$this->length = strlen($this->text);  // 使用字节长度 strlen，而非 mb_strlen
```

### 6.2 英文处理：标准空格分词

对于英文等空格分隔语言，分词效果良好：

**输入**：`The quick brown fox jumps over the lazy dog`

**索引结果**：
```
the (2次)、quick (1次)、brown (1次)、fox (1次)、jumps (1次)、over (1次)、lazy (1次)、dog (1次)
```

**软分隔符处理**（`user-friendly design`）：
- 拆出：`user`、`friendly`、`design`
- 额外保留：`user-friendly`
- 兼顾前缀匹配的模糊搜索和组合词的精确搜索

### 6.3 中文处理：按字符边界，实为"整段索引"

对于中文（以及日文、韩文等无空格语言），由于没有空格作为分隔符，分词器会将**一整段连续中文视为一个超长的"词"**。

**输入**：`这是一段中文测试文本`

**索引结果**：
```
这是一段中文测试文本 (1次)  ← 整段作为一个词
```

**原因**：
- 分隔符列表中没有中文字符
- `SearchTextTokenizer` 逐字符遍历，遇到分隔符才切分
- 连续中文字符之间没有分隔符，所以不会被切开

### 6.4 中文搜索的实际效果

#### 6.4.1 前缀匹配机制

由于查询使用 `term LIKE '关键词%'` 前缀匹配，中文搜索仍然**可用**，但粒度很粗：

**搜索**：`中文`

**匹配逻辑**：
```sql
WHERE term LIKE '中文%'
```

可以匹配到：
- `中文测试文本` ✓（整段以"中文"开头）
- `中文文档` ✓
- `中文字符` ✓

无法匹配到：
- `测试中文` ✗（"中文"不在开头）
- `这是中文的测试` ✗（"中文"在中间）

#### 6.4.2 实际可用性分析

| 场景 | 效果 | 说明 |
|------|------|------|
| 搜索标题中的中文词 | 较好 | 标题通常较短，且以目标词开头的概率较高 |
| 搜索正文中的中文词 | 较差 | 正文段落长，词通常不在段落开头 |
| 精确短语匹配 | 可以用 `"..."` | 使用精确匹配语法，走 LIKE '%...%' |
| 多词组合搜索 | 很差 | 无法按词切分，多个中文词会被当作一个整体 |

### 6.5 中英文混合内容的处理

**输入**：`BookStack 是一个开源的文档管理系统`

**索引结果**：
```
BookStack (1次)         ← 英文部分，空格切分
是一个开源的文档管理系统 (1次)  ← 中文部分，整段作为一个词
```

**搜索测试**：

| 搜索词 | 能否匹配 | 原因 |
|-------|---------|------|
| `BookStack` | ✓ | 精确匹配英文词 |
| `Book` | ✓ | 前缀匹配 BookStack |
| `是一个` | ✓ | 前缀匹配中文整段 |
| `文档管理` | ✗ | 不在中文段开头 |
| `"文档管理"` | ✓ | 精确匹配语法走 LIKE '%文档管理%' |

### 6.6 结果高亮与多字节处理

虽然分词对中文不友好，但**结果高亮和摘要功能对中文支持良好**，因为使用了 `mb_*` 多字节函数：

文件：`app/Search/SearchResultsFormatter.php`
```php
$pos = mb_strpos($text, $term, $offset);      // 多字节字符串查找
$end = $pos + mb_strlen($term);                // 多字节长度计算
$content = mb_substr($content, 0, ...);        // 多字节截取
```

这意味着：
- 即使中文词是通过精确匹配（`LIKE '%...%'`）找到的
- 结果页面仍然可以正确高亮显示匹配的中文字符
- 摘要截取不会出现乱码或半个汉字的问题

### 6.7 提高中文搜索效果的替代方案

如果需要更好的中文搜索体验，可以使用以下方式绕过分词限制：

1. **使用精确匹配语法**：用双引号包裹搜索词 `"<中文关键词>"`，走 `LIKE '%...%'` 全匹配
   - 缺点：无法利用倒排索引，性能较差，且没有相关度评分

2. **使用标签辅助搜索**：给文档打上中文标签，通过 `[标签名]` 语法搜索
   - 标签名索引权重为 ×3，标签值权重为 ×5，比正文更容易搜到

3. **在名称中包含关键词**：名称权重最高（×40），且通常较短，前缀匹配效果较好

4. **部署时集成外部搜索引擎**：如 Elasticsearch + 中文分词插件（IK Analysis）
   - 需要二次开发，替换 `SearchIndex` 和 `SearchRunner` 的实现

### 6.8 设计权衡

BookStack 选择"无专用分词器"的方案，主要基于以下考虑：

| 优点 | 缺点 |
|------|------|
| 实现简单，零依赖 | 中文/日文等无空格语言搜索效果差 |
| 数据库直接支持，部署门槛低 | 多词组合搜索在中文场景下不可用 |
| 对英文/拼音等空格分隔语言效果良好 | 搜索粒度粗，查准率和查全率都较低 |
| 性能可预测（SQL LIKE + 索引） | 长文本中文搜索基本只能靠精确匹配 |

> 这是一种典型的**"为部署简便性牺牲多语言搜索质量"**的设计取舍。对于以英文内容为主的知识库，这套方案足够高效；对于中文为主的场景，可能需要考虑扩展方案。

---

## 七、同义词、拼写纠正与模糊匹配功能

### 7.1 核心事实：三项功能均未内置

经过代码全面排查，BookStack 的全文搜索系统**没有内置**以下三项高级查询功能：

| 功能 | 英文名 | 实现状态 | 常见实现方式 |
|------|--------|---------|-------------|
| **同义词扩展** | Synonym Expansion | ❌ 未实现 | 同义词表映射、查询重写 |
| **拼写纠正** | Spell Correction | ❌ 未实现 | 编辑距离、N-gram、查询日志 |
| **模糊匹配** | Fuzzy Matching | ⚠️ 部分实现 | 只有前缀匹配，无编辑距离匹配 |

相关代码中**完全没有**出现：
- 同义词映射表或配置
- `levenshtein()`、`similar_text()`、`metaphone()`、`soundex()` 等模糊匹配函数
- "Did you mean"、"你是不是想找" 等拼写建议逻辑
- N-gram 索引或查询

### 7.2 已有的"模糊匹配"能力

BookStack 仅有的模糊匹配能力是**前缀匹配**和**精确匹配语法的中缀匹配**：

#### 7.2.1 前缀匹配（默认行为）

```sql
WHERE term LIKE 'cat%'
```

可以匹配：
- `cat` ✓（完全匹配）
- `cats` ✓（前缀匹配）
- `category` ✓（前缀匹配）
- `cat-like` ✓（前缀匹配）

**不可以**匹配：
- `wildcat` ✗（中缀）
- `bobcat` ✗（中缀）
- `cut` ✗（字符差异）
- `cta` ✗（拼写错误）

这本质上是**右模糊**匹配，利用数据库 B-tree 索引的有序性实现。

#### 7.2.2 精确匹配语法的中缀匹配

使用 `"..."` 语法时，走 `LIKE '%...%'` 中缀匹配：

```sql
WHERE name LIKE '%document%' 
   OR description LIKE '%document%'
   OR text LIKE '%document%'
```

- 可以匹配任意位置的出现
- 但**不使用倒排索引**，直接在实体表上做全表扫描
- 性能较差，且没有相关度评分

### 7.3 同义词扩展缺失分析

#### 7.3.1 实际搜索行为

假设索引了文档：
- 文档 A："The domestic cat is a small furry animal"
- 文档 B："Felines are obligate carnivores"

用户搜索 `cat`：
- ✅ 匹配文档 A（`cat` 在索引中）
- ❌ **不匹配**文档 B（`felines` 不在索引中，也不会自动扩展为 `cat`）

用户必须手动输入所有同义词：
```
cat feline kitty
```

#### 7.3.2 多搜索词的逻辑关系

需要注意：BookStack 多个搜索词之间是 **AND 关系**，不是 OR 关系。

搜索 `cat feline kitty` 的实际逻辑是：
```
实体必须同时包含 cat AND feline AND kitty
```

这意味着用户**不能**通过输入多个同义词来达到同义词扩展的效果，反而会导致搜索结果为 0。

> **隐含设计**：从代码看，多个搜索词生成的是多个 `term LIKE 'xxx%'` 条件，它们在 WHERE 子句中是 AND 关系。没有 OR 逻辑的搜索语法。

#### 7.3.3 扩展方案

如果需要同义词功能，需要二次开发：

```php
// 伪代码：在 SearchOptions 解析后扩展同义词
$synonymMap = [
    'cat' => ['cat', 'feline', 'kitty'],
    'vm'  => ['vm', 'virtual', 'virtual machine'],
];

foreach ($searchOpts->searches as $term) {
    if (isset($synonymMap[$term->value])) {
        foreach ($synonymMap[$term->value] as $synonym) {
            $searchOpts->searches->add(new TermSearchOption($synonym, false));
        }
    }
}
```

但需要修改查询逻辑，将同一概念的同义词改为 OR 关系。

### 7.4 拼写纠正缺失分析

#### 7.4.1 实际搜索行为

用户搜索 `dpcument`（拼写错误）：
- ❌ 不返回任何结果（`dpcument%` 没有匹配的 term）
- ❌ 不会提示"你是不是想找 document？"
- ❌ 不会自动重试正确拼写的查询

#### 7.4.2 缺失的实现组件

完整的拼写纠正系统需要：

1. **词典**：所有已索引词的集合（`search_terms` 表的 `term` 字段去重）
2. **候选生成**：找出与输入词编辑距离最小的候选词
3. **候选排序**：结合词频、上下文等因素排序候选
4. **用户体验**：展示建议，但不强制修改查询

#### 7.4.3 潜在的低成本实现

不引入外部服务的前提下，可以基于现有 `search_terms` 表实现基础拼写纠正：

```php
// 伪代码：查询无结果时尝试拼写纠正
if ($total === 0 && $searchOpts->searches->count() > 0) {
    foreach ($searchOpts->searches as $term) {
        $suggestions = DB::select("
            SELECT term 
            FROM (
                SELECT DISTINCT term 
                FROM search_terms 
                WHERE term LIKE ? OR term LIKE ?
            ) candidates
            WHERE levenshtein(term, ?) <= 2
            ORDER BY levenshtein(term, ?) ASC, term ASC
            LIMIT 5
        ", [
            $term[0] . '%',
            '%' . substr($term, -1),
            $term, $term
        ]);
    }
}
```

> **注意**：MySQL 5.7+ 内置 `levenshtein()` 函数，但需要开启；PostgreSQL 需要安装 fuzzystrmatch 扩展。

### 7.5 搜索建议 vs 拼写纠正

前端全局搜索框有一个"搜索建议"功能，但它**不是拼写纠正**：

文件：`resources/js/components/global-search.js`

```javascript
async updateSuggestions(search) {
    const {data: results} = await window.$http.get('/search/suggest', {term: search});
    // ... 显示前5条匹配实体
}
```

后端接口 `searchSuggestions`：

文件：`app/Search/SearchController.php:120-132`

```php
public function searchSuggestions(Request $request)
{
    $searchTerm = $request->input('term', '');
    $entities = $this->searchRunner->searchEntities(
        SearchOptions::fromString($searchTerm), 
        'all', 1, 5
    )['results'];
    
    // 清空预览内容，只返回标题列表
    foreach ($entities as $entity) {
        $entity->setAttribute('preview_content', '');
    }
    
    return view('search.parts.entity-suggestion-list', [
        'entities' => $entities->slice(0, 5)
    ]);
}
```

**两者区别**：

| 特性 | 搜索建议（Suggestions） | 拼写纠正（Did You Mean） |
|------|------------------------|-------------------------|
| **触发时机** | 输入时实时触发（200ms debounce） | 查询无结果或低相关度时 |
| **返回内容** | 匹配的实体列表（标题+类型） | 推荐的搜索词修正建议 |
| **依赖** | 搜索索引（正常搜索逻辑） | 词典 + 编辑距离算法 |
| **目的** | 快速到达目标文档 | 帮助修正拼写错误 |
| **当前状态** | ✅ 已实现 | ❌ 未实现 |

前端 debounce 200ms，避免频繁请求。

### 7.6 设计权衡

BookStack 选择不内置这些高级功能，主要基于以下考虑：

**不做同义词**：
- 同义词维护成本高（不同领域同义词不同）
- 多语言同义词更加复杂
- 误扩展可能引入不相关结果，降低查准率
- 知识库搜索场景下，用户通常知道精确术语

**不做拼写纠正**：
- 实现复杂（需要词典、候选生成、排序）
- 依赖数据库函数（levenshtein）可能不可用
- 性能开销大（每个无结果查询需要额外的词典扫描）
- 用户复制粘贴搜索占比高，拼写错误场景相对较少

**不做高级模糊匹配**：
- 前缀匹配 + 精确匹配语法已覆盖多数场景
- 编辑距离匹配性能差
- 增加复杂性但收益有限

**扩展建议**：
如果确实需要这些功能，推荐方案是**替换搜索引擎层**，而不是在现有代码上修补：

1. **Meilisearch** / **Typesense**：轻量级、易部署、内置同义词、拼写纠正、模糊匹配
2. **Elasticsearch** / **OpenSearch**：功能强大但运维复杂
3. 集成后只需替换 `SearchIndex`（写入）和 `SearchRunner`（查询）两个类的实现

### 7.7 现有搜索语法能力清单

作为对比，总结 BookStack 已有的搜索语法：

| 语法 | 示例 | 功能 |
|------|------|------|
| 普通词 | `cat dog` | AND 关系，前缀匹配倒排索引 |
| 精确匹配 | `"cat dog"` | LIKE '%cat dog%'，走实体表扫描 |
| 标签搜索 | `[priority=high]` | 关联 tags 表查询 |
| 过滤器 | `{created_by:me}` | 17 种内置过滤条件 |
| 否定前缀 | `-cat` | NOT 逻辑，适用于所有语法 |
| 类型过滤 | `{type:page}` | 限制实体类型 |
| 自定义排序 | `{sort_by:last_commented}` | 按最后评论时间排序 |
