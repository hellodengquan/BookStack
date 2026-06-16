# 页面渲染管线分析（WYSIWYG 编辑 → 只读视图）

本文档梳理 BookStack 中从所见即所得编辑器输出内容，经过后端模板拼装、净化策略，最终渲染到只读视图的完整管线。

---

## 一、整体管线总览

```
┌───────────────────────────────────────────────────────────────────────┐
│                        前端：编辑阶段                                    │
│  ┌──────────────┐   getContent()   ┌──────────────┐   form.submit    │
│  │ WYSIWYG/MD   │ ──────────────► │ PageEditor   │ ────────────────  │
│  │   Editor     │   {html, md?}   │  (协调者)    │   PUT/POST       │
│  └──────────────┘                  └──────────────┘                  │
└──────────────────────────────────────┬────────────────────────────────┘
                                       │ name, html, markdown, editor
                                       ▼
┌───────────────────────────────────────────────────────────────────────┐
│                     后端：保存/拼装阶段                                  │
│  ┌──────────────┐  updateContent  ┌──────────────┐  formatHtml+img   │
│  │  PageRepo    │ ──────────────► │ PageContent  │ ────────────────  │
│  │ (路由入口)   │                 │ (内容加工)    │  ID规范化/提取    │
│  └──────────────┘                 └──────────────┘                  │
│                                       │                               │
│                                       ▼                               │
│                              pages 表 (html/markdown/text 字段)       │
└──────────────────────────────────────┬────────────────────────────────┘
                                       │
                                       ▼
┌───────────────────────────────────────────────────────────────────────┐
│                     后端：渲染/净化阶段                                  │
│  ┌──────────────┐  PageContent   ┌──────────────────┐  cache         │
│  │PageController│ ::render()     │ PageIncludeParser│ 命中直接返回   │
│  │   ::show     │ ──────────────►│  (解析{{@id}})   │ ───────┐       │
│  └──────────────┘                └────────┬─────────┘        │       │
│                                           │ 未命中            │       │
│                                           ▼                   │       │
│                                ┌──────────────────┐          │       │
│                                │ HtmlContentFilter│          │       │
│                                │ (5层净化管道)     │          │       │
│                                │ + HTMLPurifier   │          │       │
│                                └────────┬─────────┘          │       │
│                                         │ 写入缓存          │       │
│                                         ▼                   ▼       │
│                                    返回净化后的 HTML ◄────────┘       │
└──────────────────────────────────────┬────────────────────────────────┘
                                       │ {!! $page->html !!}
                                       ▼
┌───────────────────────────────────────────────────────────────────────┐
│                     前端：只读视图阶段                                    │
│  pages/show.blade.php                                                 │
│    └─ pages/parts/page-display.blade.php                              │
│         └─ {!! isset($page->renderedHTML) ? $page->renderedHTML       │
│              : $page->html !!}                                        │
│              (component="page-display" 负责前端交互增强)                │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 二、编辑器输出（前端阶段）

### 2.1 三种编辑器类型

由 `PageEditorType` 枚举 (`app/Entities/Tools/PageEditorType.php:7`) 定义：

| 枚举值 | value | 输出格式 | 说明 |
|--------|-------|----------|------|
| `WysiwygLexical` | `wysiwyg2024` | HTML | 新版 Lexical 编辑器（默认方向） |
| `WysiwygTinymce` | `wysiwyg` | HTML | 旧版 TinyMCE（已弃用但仍兼容） |
| `Markdown` | `markdown` | Markdown + HTML | Markdown 编辑器，双格式输出 |

### 2.2 编辑器组件的内容输出

#### WYSIWYG 编辑器 (`resources/js/components/wysiwyg-editor.js:65`)

```javascript
// 返回 Promise<{html: String}>
async getContent() {
    return {
        html: await this.editor.getContentAsHtml(),
    };
}
```

- 表单提交时拦截 `submit` 事件，先通过 `getContentAsHtml()` 拉取内容
- 写入隐藏的 `<textarea name="html">` 再触发真正的提交
- 模板文件：`resources/views/pages/parts/wysiwyg-editor.blade.php:14`

```blade
<textarea refs="wysiwyg-editor@input" id="html-editor" hidden="hidden"
          name="html" rows="5">{{ (old('html') ?? $model->html ?? '') ?: '<p></p>' }}</textarea>
```

#### Markdown 编辑器 (`resources/js/components/markdown-editor.js:138`)

```javascript
// 返回 Promise<{html: String, markdown: String}>
async getContent() {
    return this.editor.actions.getContent();
}
```

### 2.3 PageEditor 协调者 (`resources/js/components/page-editor.js:6`)

- 监听 `editor-html-change` / `editor-markdown-change` 事件感知内容变化
- 每 30 秒自动保存草稿：调用 `saveDraft()` 向 `/ajax/page/{id}/save-draft` 发送 PUT 请求
- 手动保存页面：直接调用 `form.requestSubmit()`（走常规表单提交）

草稿保存时的数据结构 (`page-editor.js:124`)：
```javascript
const data = {name: this.titleElem.value.trim()};
const editorContent = await this.getEditorComponent().getContent();
Object.assign(data, editorContent);
// => { name, html, [markdown] }
```

### 2.4 编辑器加载时的预处理 (`app/Entities/Tools/PageEditorData.php:72`)

如果**不是原作者**打开已有页面，加载时就先对 HTML 过一次净化过滤器，防止把别人的恶意脚本带进编辑器：

```php
if ($editorType->isHtmlBased() && !old('html') && $lastEditorId !== user()->id) {
    $filterConfig = HtmlContentFilterConfig::fromConfigString(config('app.content_filtering'));
    $filter = new HtmlContentFilter($filterConfig);
    $page->html = $filter->filterString($page->html);
}
```

---

## 三、模板拼装与内容保存（后端阶段）

### 3.1 保存入口路由

| 场景 | 方法 | 核心调用 |
|------|------|---------|
| 发布新页面 | `PageController::store` → `PageRepo::publishDraft` | `updateTemplateStatusAndContentFromInput` |
| 更新页面 | `PageController::update` → `PageRepo::update` | `updateTemplateStatusAndContentFromInput` |
| 保存草稿 | `PageController::saveDraft` → `PageRepo::updatePageDraft` | 草稿页面直接更新，否则存到 `PageRevision` |
| 恢复版本 | `PageRepo::restoreRevision` | `PageContent::setNewHTML/setNewMarkdown` |

### 3.2 内容分发逻辑 (`PageRepo.php:151`)

```php
protected function updateTemplateStatusAndContentFromInput(Page $page, array $input): void
{
    $pageContent = new PageContent($page);
    // ... 根据 input 判断编辑器类型 ...

    $haveInput = isset($input['markdown']) || isset($input['html']);
    $inputEmpty = empty($input['markdown']) && empty($input['html']);

    if ($haveInput && $inputEmpty) {
        $pageContent->setNewHTML('', user());           // 清空
    } elseif (!empty($input['markdown']) && is_string($input['markdown'])) {
        $pageContent->setNewMarkdown($input['markdown'], user());  // Markdown 路径
    } elseif (isset($input['html'])) {
        $pageContent->setNewHTML($input['html'], user());         // HTML 路径 (WYSIWYG)
    }
}
```

### 3.3 HTML 路径：`PageContent::setNewHTML` (`PageContent.php:40`)

```
输入 HTML
  │
  ▼
① extractBase64ImagesFromHtml()
  │  扫描 <img src="data:image/xxx;base64,...">
  │  调用 ImageRepo::saveNewFromData() 存为文件并替换 src 为 URL
  │
  ▼
② formatHtml()
  │  ├─ updateIdsRecursively()
  │  │    顶层块级元素、h1~h6、有id的元素分配 bkmrk-* ID
  │  │    用于页面内锚点导航
  │  └─ updateLinks()
  │       更新锚点 href 指向重新生成的 id
  │
  ▼
③ Theme::dispatch(PAGE_CONTENT_PRE_STORE)
  │  主题系统可在此钩子修改 HTML
  │
  ▼
④ 写入 $page->html
  ▼
⑤ toPlainText() → 写入 $page->text (搜索索引用)
  ▼
⑥ $page->markdown = '' (清空标记为 HTML 源)
```

### 3.4 Markdown 路径：`PageContent::setNewMarkdown` (`PageContent.php:58`)

```
输入 Markdown
  │
  ▼
① extractBase64ImagesFromMarkdown()
  │  正则 + 手动循环定位 data:image URI
  │  存为文件并替换 Markdown 中的 URL
  │
  ▼
② 原始 Markdown → $page->markdown
  ▼
③ MarkdownToHtml() 转换为 HTML
  │  (League CommonMark + 自定义扩展)
  ▼
④ formatHtml()  [与HTML路径一致：ID 规范化]
  ▼
⑤ Theme::dispatch(PAGE_CONTENT_PRE_STORE)
  ▼
⑥ 写入 $page->html + $page->text (同 HTML 路径)
```

### 3.5 默认模板拼装 (`PageRepo.php:63`)

创建草稿时如果父级（书/章节）设置了 `default_template_id`，会自动加载模板内容：

```php
$defaultTemplate = $page->chapter?->defaultTemplate() ?? $page->book->defaultTemplate();
if ($defaultTemplate) {
    $page->forceFill([
        'html'  => $defaultTemplate->html,
        'markdown' => $defaultTemplate->markdown,
    ]);
    $page->text = (new PageContent($page))->toPlainText();
}
```

### 3.6 Base64 图片安全检查 (`PageContent.php:137`)

提取内嵌图片时做了多层防护：

1. 权限：`$updater->can(Permission::ImageCreateAll)`
2. 扩展名：`ImageService::isExtensionSupported()`
3. MIME 嗅探：`WebSafeMimeSniffer` 校验必须是 `image/*`
4. 大小限制：`upload_limit` (MB)

---

## 四、净化策略

BookStack 采用 **多层防御 + 按需组合** 的净化架构，由 `HtmlContentFilter` + 可选的 `HTMLPurifier` 白名单实现。

### 4.1 配置驱动 (`app/Config/app.php:48`)

```php
'content_filtering' => env('APP_CONTENT_FILTERING',
    env('ALLOW_CONTENT_SCRIPTS', false) === true ? '' : 'jhfa'
);
```

配置字符含义（`HtmlContentFilterConfig::fromConfigString`）：

| 字符 | 开关 | 说明 |
|------|------|------|
| `j` | `filterOutJavaScript` | 过滤脚本/二进制 |
| `h` | `filterOutBadHtmlElements` + `filterOutNonContentElements` | 过滤异常/非内容元素 |
| `f` | `filterOutFormElements` | 过滤表单元素 |
| `a` | `useAllowListFilter` | 启用 HTMLPurifier 白名单 |

默认 `jhfa`（全开）；`ALLOW_CONTENT_SCRIPTS=true` 时全部关闭（不推荐）。

### 4.2 HtmlContentFilter 五层管道 (`app/Util/HtmlContentFilter.php:17`)

```php
public function filterDocument(HtmlDocument $doc): string
{
    if ($this->config->filterOutJavaScript)        $this->filterOutScriptsFromDocument($doc);        // ①
    if ($this->config->filterOutFormElements)      $this->filterOutFormElementsFromDocument($doc);    // ②
    if ($this->config->filterOutBadHtmlElements)   $this->filterOutBadHtmlElementsFromDocument($doc); // ③
    if ($this->config->filterOutNonContentElements)$this->filterOutNonContentElementsFromDocument($doc);// ④

    $filtered = $doc->getBodyInnerHtml();
    if ($this->config->useAllowListFilter)          $filtered = $this->applyAllowListFiltering($filtered); // ⑤

    return $filtered;
}
```

#### ① JavaScript/XSS 过滤 (`filterOutScriptsFromDocument`)

| 操作 | XPath 示例 |
|------|-----------|
| 删除 `<script>` 标签 | `//script` |
| 删除 `javascript:` 链接 | `//*[contains(@href, 'javascript:')]` |
| 删除危险表单属性 | `action` / `formaction` 中的 `javascript:` |
| 删除 data:/javascript: iframe/embed | `//*[contains(@src, 'data:')]` 等 |
| 删除 SVG 危险属性值 | SVG 下属性值含 data:/javascript: |
| 删除 `xlink:href` 属性 | SVG 废弃属性，防止 XSS |
| 删除所有 `on*` 属性 | `//@*[starts-with(name(), 'on')]` |

#### ② 表单元素过滤 (`filterOutFormElementsFromDocument`)

- 全删：`<form>` `<fieldset>` `<button>` `<textarea>` `<select>`
- 非 checkbox 的 `<input>` 也删
- 删属性：`form` `formaction` `formmethod` `formtarget`

#### ③ 异常 HTML 元素 (`filterOutBadHtmlElementsFromDocument`)

- 删除带 URL 跳转的 `<meta http-equiv="refresh">`

#### ④ 非内容元素 (`filterOutNonContentElementsFromDocument`)

- 删除：`<link>` `<style>` `<meta>` `<title>` `<template>`

#### ⑤ HTMLPurifier 白名单 (`ConfiguredHtmlPurifier`)

基于 `ezyang/htmlpurifier` + `xemlock/htmlpurifier-html5`，额外扩展支持：

```php
// 额外允许的元素
- <object data type width height>
- <embed src type width height>
- <input type=checkbox checked disabled readonly>

// 额外允许的属性
- div[drawio-diagram]         —— Draw.io 图表标识
- a[target=_blank]            —— 新窗口链接
- a[data-mention-user-id]     —— 用户@提及
```

URI 策略：
- 允许协议：`http https mailto ftp nntp news tel file`
- `file://` 协议限制为只能指向锚点（`UriLimitFileProtocolToAnchors`）
- Iframe src 正则：`%^(http://|https://|//)%`
- 启用：`CSS.AllowTricky`、`Attr.EnableID`

### 4.3 净化触发时机汇总

| 时机 | 位置 | 说明 |
|------|------|------|
| 进入编辑页 | `PageEditorData.php:72` | **非原作者**时提前净化，防止污染编辑器 DOM |
| 只读渲染 | `PageContent.php:343` | 必经之路，每次 `render()` 必净化（可被缓存跳过） |
| Ajax 取页 | `PageController::getPageAjax:188` | 独立接口单独走一次过滤 |

> 注意：**保存到数据库前不净化**，存储原始输入；净化只在读取/展示时进行。这一设计支持保留编辑器原始结构，同时可通过更改配置立即对历史内容生效。

---

## 五、只读视图渲染

### 5.1 控制器入口 (`PageController::show`, `PageController.php:139`)

```php
public function show(string $bookSlug, string $pageSlug)
{
    $page = $this->queries->findVisibleBySlugsOrFail($bookSlug, $pageSlug);

    $pageContent = (new PageContent($page));
    $page->html = $pageContent->render();          // ← 核心：拼装+净化
    $pageNav  = $pageContent->getNavigation($page->html); // 提取 h1~h6 生成侧边目录

    return view('pages.show', [
        'page' => $page,
        'pageNav' => $pageNav,
        // ...
    ]);
}
```

### 5.2 `PageContent::render()` 详解 (`PageContent.php:314`)

```
$page->html (DB原始内容)
  │
  ▼
① 空内容短路 → handlePostRender('')
  │
  ▼
② PageIncludeParser::parse()  (最多3层嵌套)
  │  扫描 {{@page_id}} 语法 → 递归拉取被引用页面 html 嵌入
  │  若嵌套>1层，重新规范所有锚点ID
  │
  ▼
③ 计算缓存键：content_filtering + appVersion + pageId + updatedAt + md5(html)
  │
  ▼
④ 命中 cache → 直接跳转 ⑧
  │  未命中 → 继续
  │
  ▼
⑤ HtmlContentFilter 净化 (见第四章)
  │
  ▼
⑥ 写入缓存 → 1周 (86400 * 7)
  │
  ▼
⑦ Theme::dispatch(PAGE_CONTENT_POST_RENDER)
  │
  ▼
⑧ 返回最终 HTML
```

### 5.3 页面引用解析 (`PageIncludeParser.php:34`)

用户在编辑器中输入 `{{@123}}` 即可引用 id=123 的页面内容：

1. **定位**：XPath `//*[text()[contains(., '{{@')]]` → 切分 Text 节点，将每个标签隔离为独立 DOM 节点
2. **块级处理**：非内联内容会把父 `<p>` 拆开，引用内容作为兄弟节点插入
3. **递归上限**：最多嵌套 3 层防止循环引用死循环
4. **主题钩子**：`PAGE_INCLUDE_PARSE` 可替换被引用内容

### 5.4 视图模板链路

```
pages/show.blade.php
  └─ @include('pages.parts.page-display')
       └─ page-display.blade.php:10

          @if (isset($diff) && $diff)
              {!! $diff !!}
          @else
              {!! isset($page->renderedHTML) ? $page->renderedHTML : $page->html !!}
          @endif
```

- `$page->html` 在控制器里已被覆写为渲染+净化后的结果
- `$page->renderedHTML` 是预留字段（差异对比视图用）
- 外层包裹在 `<div component="page-display">` 中，前端 `page-display.js` 组件负责：
  - 代码高亮
  - 详情块折叠
  - 点击附件/图片交互

### 5.5 内容缓存键设计 (`PageContent.php:359`)

```php
return "page-content-cache::{$filterConfig}::{$appVersion}::{$contentId}::{$contentTime}::{$contentHash}";
```

包含的维度：
- `filterConfig`：净化配置变更 → 强制重算
- `appVersion`：程序版本升级 → 强制重算
- `contentId`：页面ID
- `contentTime`：`updated_at` 时间戳 → 编辑后自动失效
- `contentHash`：HTML 内容 MD5 → 内容变化即失效

---

## 六、关键文件索引

| 职责 | 文件 | 核心行 |
|------|------|--------|
| 页面展示控制器 | `app/Entities/Controllers/PageController.php` | `show():139` |
| 页面保存仓库 | `app/Entities/Repos/PageRepo.php` | `updateTemplateStatusAndContentFromInput():151` |
| 内容加工/渲染核心 | `app/Entities/Tools/PageContent.php` | `setNewHTML():40`, `render():314` |
| 页面引用解析器 | `app/Entities/Tools/PageIncludeParser.php` | `parse():34` |
| HTML 过滤器主类 | `app/Util/HtmlContentFilter.php` | `filterDocument():17` |
| 过滤配置 | `app/Util/HtmlContentFilterConfig.php` | `fromConfigString():20` |
| HTMLPurifier 封装 | `app/Util/HtmlPurifier/ConfiguredHtmlPurifier.php` | 全文 |
| 编辑页数据装配 | `app/Entities/Tools/PageEditorData.php` | `build():37` |
| 编辑器类型枚举 | `app/Entities/Tools/PageEditorType.php` | 全文 |
| 前端-页面编辑器 | `resources/js/components/page-editor.js` | `saveDraft():123` |
| 前端-WYSIWYG 编辑器 | `resources/js/components/wysiwyg-editor.js` | `getContent():65` |
| 前端-Markdown 编辑器 | `resources/js/components/markdown-editor.js` | `getContent():138` |
| 只读展示模板 | `resources/views/pages/parts/page-display.blade.php` | 全文 |
| 展示页主模板 | `resources/views/pages/show.blade.php` | `@section('body'):9` |
