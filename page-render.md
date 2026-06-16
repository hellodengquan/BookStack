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

### 2.5 TinyMCE 编辑器：Paste 净化 (`resources/js/wysiwyg-tinymce/config.js:296`)

TinyMCE 配置了三层粘贴防御：

**① `paste_preprocess` 钩子（TinyMCE 原生事件）**
```javascript
paste_preprocess(plugin, args) {
    const {content} = args;
    if (content.indexOf('<img src="file://') !== -1) {
        args.content = '';  // 直接清空本地文件引用
    }
}
```

**② `paste_data_images: false`（全局配置）**
```javascript
paste_data_images: false,  // 禁止 TinyMCE 自动把剪贴板图片转成 base64
```

**③ 自定义 `paste` 事件监听器 (`drop-paste-handling.js:35`)**
```javascript
export function listenForDragAndPaste(editor, options) {
    editor.on('paste', event => paste(editor, options, event));
    editor.on('drop', event => drop(editor, options, event));
    // ...
}
```

自定义 `paste()` 函数处理流程：
1. 用 `Clipboard` 服务检查剪贴板条目
2. 如果包含表格数据 → 交给 TinyMCE 默认处理
3. 有图片文件 → 拦截默认行为，插入 loading 占位图后异步上传
4. 无图片 → 不拦截，走默认粘贴

### 2.6 TinyMCE 编辑器：Image Upload Base64 转外链 (`drop-paste-handling.js:15`)

当粘贴/拖拽图片到编辑器时：

```javascript
async function uploadImageFile(file, pageId) {
    const formData = new FormData();
    formData.append('file', file, file.name);
    formData.append('uploaded_to', pageId);
    const resp = await window.$http.post(window.baseUrl('/images/gallery'), formData);
    return resp.data;  // 返回 {url, thumbs: {display}, name}
}

function paste(editor, options, event) {
    const images = clipboard.getImages();
    for (const imageFile of images) {
        // 1. 先插入 loading 占位图
        editor.insertContent(`<p><img src="/loading.gif" id="${id}"></p>`);
        // 2. 异步上传
        uploadImageFile(imageFile, options.pageId).then(resp => {
            // 3. 上传成功，替换为外链 + a 标签包裹
            const newImageHtml = `<img src="${resp.thumbs.display}" alt="${resp.name}"/>`;
            const newEl = editor.dom.create('a', {target: '_blank', href: resp.url}, newImageHtml);
            editor.dom.replace(newEl, id);
        }).catch(err => {
            // 4. 上传失败，移除占位图
            editor.dom.remove(id);
        });
    }
}
```

**后端上传安全检查**（`ImageRepo.php:87`）：
```php
$contextPage = $this->pageQueries->findVisibleByIdOrFail($uploadedTo);
// 校验当前用户对该 page 有 ImageCreateAll 权限
```

### 2.7 TinyMCE 编辑器：Code-Block 语言识别 (`plugin-codeeditor.js`)

TinyMCE 使用自定义 `<code-block>` Web Component 作为代码块包装器，实现语法高亮的所见即所得：

**① 解析阶段：`<pre>` → `<code-block>` 包装**
```javascript
editor.parser.addNodeFilter('pre', elms => {
    for (const el of elms) {
        const wrapper = window.tinymce.html.Node.create('code-block', {
            contenteditable: 'false',
        });
        // 剥除 <pre> 内的高亮 span（防止粘贴带格式的代码）
        const spans = el.getAll('span');
        for (const span of spans) span.unwrap();
        el.attr('style', null);
        el.wrap(wrapper);
    }
});
```

**② 语言识别：`<code class="language-xxx">`**
```javascript
getLanguage() {
    const getLanguageFromClassList = classes => {
        const langClasses = classes.split(' ').filter(c => c.startsWith('language-'));
        return (langClasses[0] || '').replace('language-', '');
    };
    const code = this.querySelector('code');
    const pre = this.querySelector('pre');
    return getLanguageFromClassList(pre.className) || (code && getLanguageFromClassList(code.className)) || '';
}
```

**③ 序列化阶段：`<code-block>` → 解包**
```javascript
editor.serializer.addNodeFilter('code-block', elms => {
    for (const el of elms) {
        // 把 dir 转到内部 <pre> 上
        const direction = el.attr('dir');
        if (direction && el.firstChild) el.firstChild.attr('dir', direction);
        el.unwrap();  // 移除外层 <code-block>，保留内部 <pre><code>
    }
});
```

最终输出到数据库的格式：
```html
<pre dir="ltr"><code class="language-javascript">const a = 1;</code></pre>
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

### 4.4 HtmlContentFilter 自定义过滤的完整 XPath 枚举

`filterOutScriptsFromDocument` 方法中的完整 XPath 规则：

| 目标 | XPath 表达式 | 处理方式 |
|------|-------------|----------|
| `<script>` 标签 | `//script` | 删除节点 |
| `<iframe src="data:">` | `//iframe[contains(@src, 'data:')]` | 删除节点 |
| `<embed src="data:">` | `//*[self::embed or self::object][contains(@src, 'data:')]` | 删除节点 |
| `<iframe src="javascript:">` | `//*[self::iframe or self::embed or self::object][contains(@src, 'javascript:')]` | 删除节点 |
| `href="javascript:"` | `//*[contains(@href, 'javascript:')]` | 删除节点 |
| `action="javascript:"` | `//*[contains(@action, 'javascript:')]` | 删除节点 |
| `formaction="javascript:"` | `//*[contains(@formaction, 'javascript:')]` | 删除节点 |
| `xlink:href` SVG 属性 | `//@*[name() = 'xlink:href']` | 删除属性 |
| SVG 危险属性值 | `//@*[namespace-uri() = 'http://www.w3.org/2000/svg' and contains(., 'javascript:')]` | 删除属性 |
| SVG data: 属性值 | `//@*[namespace-uri() = 'http://www.w3.org/2000/svg' and contains(., 'data:')]` | 删除属性 |
| `on*` 事件属性 | `//@*[starts-with(name(), 'on')]` | 删除属性 |

`filterOutFormElementsFromDocument` 方法：

| 目标 | XPath 表达式 | 处理方式 |
|------|-------------|----------|
| 表单标签 | `//*[self::form or self::fieldset or self::button or self::textarea or self::select]` | 删除节点 |
| 非 checkbox 的 `<input>` | `//input[not(@type) or @type != 'checkbox']` | 删除节点 |
| `form*` 属性 | `//@*[name() = 'form' or name() = 'formaction' or name() = 'formmethod' or name() = 'formtarget']` | 删除属性 |

`filterOutBadHtmlElementsFromDocument` 方法：

| 目标 | XPath 表达式 | 处理方式 |
|------|-------------|----------|
| meta refresh | `//meta[@http-equiv = 'refresh' and contains(@content, 'url=')]` | 删除节点 |

`filterOutNonContentElementsFromDocument` 方法：

| 目标 | XPath 表达式 | 处理方式 |
|------|-------------|----------|
| 非内容标签 | `//*[self::link or self::style or self::meta or self::title or self::template]` | 删除节点 |

### 4.5 HTMLPurifier 白名单标签与属性完整枚举

`HTMLPurifier` 默认 HTML5 配置 + BookStack 自定义扩展后的允许清单：

**块级元素**：
```
address, article, aside, blockquote, body, br, caption, cite, code, col, colgroup, dd, del, details, dfn, div, dl, dt, em, figcaption, figure, footer, h1, h2, h3, h4, h5, h6, header, hr, html, i, img, ins, kbd, li, main, mark, nav, noscript, ol, p, pre, q, rp, rt, ruby, s, samp, section, small, span, strike, strong, sub, summary, sup, table, tbody, td, tfoot, th, thead, time, tr, u, ul, var, wbr, object, embed, input
```

**自定义扩展元素**：
```php
// <object data type width height>
$object = $config->getHTMLDefinition(true)->addElement('object', 'Block', 'Flow', 'Common', [
    'data' => 'URI#embedded',
    'type' => 'Text',
    'width' => 'Length',
    'height' => 'Length',
]);
// <embed src type width height>
$embed = $config->getHTMLDefinition(true)->addElement('embed', 'Inline', 'Empty', 'Common', [
    'src' => 'URI#embedded',
    'type' => 'Text',
    'width' => 'Length',
    'height' => 'Length',
]);
// <input type=checkbox checked disabled readonly>
$input = $config->getHTMLDefinition(true)->addElement('input', 'Inline', 'Empty', 'Common', [
    'type' => new HTMLPurifier_AttrDef_Enum(['checkbox']),
    'checked' => 'Bool',
    'disabled' => 'Bool',
    'readonly' => 'Bool',
]);
```

**自定义扩展属性**：
```php
// div[drawio-diagram]
$div = $config->getHTMLDefinition(true)->addBlankElement('div');
$div->attr['drawio-diagram'] = new HTMLPurifier_AttrDef_Text();

// a[target=_blank|data-mention-user-id]
$a = $config->getHTMLDefinition(true)->addBlankElement('a');
$a->attr['target'] = new HTMLPurifier_AttrDef_Enum(['_blank']);
$a->attr['data-mention-user-id'] = new HTMLPurifier_AttrDef_Text();
```

### 4.6 净化触发时机汇总

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

### 5.6 `page-display` 前端组件：Mounted 阶段 (`page-display.js:34`)

`PageDisplay 组件在 DOM 就绪后触发 `setup()` 方法（等同于 mounted）：

```javascript
setup() {
    this.container = this.$el;
    this.pageId = this.$opts.pageId;

    // ① 语法高亮
    window.importVersioned('code').then(Code => Code.highlight());

    // ② 目录滚动高亮
    this.setupNavHighlighting();

    // ③ URL hash 跳转到指定内容
    if (window.location.hash) {
        const text = window.location.hash.replace(/%20/g, ' ').substring(1);
        this.goToText(text);
    }

    // ④ 侧边导航点击跳转
    const sidebarPageNav = document.querySelector('.sidebar-page-nav');
    if (sidebarPageNav) {
        DOM.onChildEvent(sidebarPageNav, 'a', 'click', (event, child) => {
            event.preventDefault();
            window.$components.first('tri-layout').showContent();
            const contentId = child.getAttribute('href').substr(1);
            this.goToText(contentId);
            window.history.pushState(null, null, `#${contentId}`);
        });
    }
}
```

### 5.7 语法高亮实现 (`resources/js/code/index.mjs:62`)

后端输出的 `<pre><code class="language-xxx">` 在前端被 CodeMirror 6 替换：

```javascript
function highlightElem(elem) {
    const innerCodeElem = elem.querySelector('code[class^=language-]');
    elem.innerHTML = elem.innerHTML.replace(/<br\s*\/?>/gi, '\n');
    const content = elem.textContent.trimEnd();

    // 从 class 提取语言名
    let langName = '';
    if (innerCodeElem !== null) {
        langName = innerCodeElem.className.replace('language-', '');
    }

    // 创建包装器 + CodeMirror 替换原始 <pre>
    const wrapper = document.createElement('div');
    elem.parentNode.insertBefore(wrapper, elem);

    const ev = createView('content-code-block', {
        parent: wrapper,
        doc: content,
        extensions: viewerExtensions(wrapper),
    });

    const editor = new SimpleEditorInterface(ev);
    editor.setMode(langName, content);  // 根据语言名加载对应模式

    elem.remove();
    addCopyIcon(ev);  // 添加复制按钮
}
```

**语言识别映射表** (`languages.js:20`)：

支持 60+ 种语言别名映射到 CodeMirror 6 模式：
```javascript
const modeMap = {
    // 原生 CM6 扩展：css, json, javascript, html, markdown, php, xml, twig
    // 动态加载 legacy mode：bash, c, c++, c#, java, python, ruby, rust, sql, ...
    // 特殊处理：php 根据是否包含 <?php 决定 plain 模式
    php: async code => {
        const hasTags = code.includes('<?php');
        return php({plain: !hasTags});
    },
};
```

### 5.8 目录滚动高亮 (`page-display.js:78`)

使用 `IntersectionObserver` 实现滚动时自动高亮当前阅读位置：

```javascript
function addNavObserver(headings) {
    const intersectOpts = {
        rootMargin: '0px 0px 0px 0px',
        threshold: 1.0,  // 100% 可见才触发
    };
    const pageNavObserver = new IntersectionObserver(headingVisibilityChange, intersectOpts);

    for (const heading of headings) {
        pageNavObserver.observe(heading);
    }
}

function headingVisibilityChange(entries) {
    for (const entry of entries) {
        const isVisible = (entry.intersectionRatio === 1);
        toggleAnchorHighlighting(entry.target.id, isVisible);
    }
}

function toggleAnchorHighlighting(elementId, shouldHighlight) {
    DOM.forEach(`#page-navigation a[href="#${elementId}"]`, anchor => {
        anchor.closest('li').classList.toggle('current-heading', shouldHighlight);
    });
}
```

**滚动跳转** (`page-display.js:60`)：
```javascript
goToText(text) {
    const idElem = document.getElementById(text);
    if (idElem !== null) {
        scrollAndHighlightElement(idElem);  // 平滑滚动 + 闪烁高亮
    } else {
        const textElem = DOM.findText('.page-content > div > *', text);
        if (textElem) scrollAndHighlightElement(textElem);
    }
}
```

### 5.9 缓存命中后 Hooks 触发确认

**结论：缓存命中后 `PAGE_CONTENT_POST_RENDER` 钩子仍然会触发。**

看 `PageContent::render()` 流程：

```php
public function render(bool $blankIncludes = false): string
{
    // ... include 解析 ...

    $cacheKey = $this->getContentCacheKey($doc->getBodyInnerHtml());
    $cached = cache()->get($cacheKey, null);
    
    if ($cached !== null) {
        return $this->handlePostRender($cached);  // ← 缓存命中，仍然调用 handlePostRender
    }

    // ... 未命中，净化 + 写缓存 ...
    $filtered = $filter->filterDocument($doc);
    cache()->put($cacheKey, $filtered, $cacheTime);

    return $this->handlePostRender($filtered);  // ← 未命中也调用
}

protected function handlePostRender(string $html): string
{
    $themeResult = Theme::dispatch(ThemeEvents::PAGE_CONTENT_POST_RENDER, $html, $this->page);
    return is_string($themeResult) ? $themeResult : $html;
}
```

**设计意图**：
- 缓存只存"净化后的 HTML"，不包含主题定制结果
- 允许主题动态修改内容而不受缓存限制
- 主题钩子必须是纯函数、幂等，重复调用不产生副作用

---

## 六、主题系统扩展点与监听器机制

### 6.1 `Theme` 门面 API (`app/Theming/ThemeService.php`)

BookStack 的主题事件系统是典型的观察者模式：

```php
class ThemeService
{
    protected array $listeners = [];

    // 注册监听器
    public function listen(string $event, callable $action): void
    {
        $this->listeners[$event][] = $action;
    }

    // 触发事件（第一个非 null 返回值终止链）
    public function dispatch(string $event, ...$args): mixed
    {
        foreach ($this->listeners[$event] ?? [] as $action) {
            $result = call_user_func_array($action, $args);
            if (!is_null($result)) {
                return $result;  // 第一个非 null 返回作为结果
            }
        }
        return null;
    }
}
```

**执行规则：
1. 允许多个监听器监听同一事件
2. 按注册顺序执行
3. 任何监听器返回非 `null` 时立即终止后续监听器
4. 返回值被系统使用（取决于具体事件）

### 6.2 页面渲染相关的主题事件

| 事件常量 | 触发时机 | 参数 | 返回值用途 |
|----------|---------|------|----------|
| `PAGE_CONTENT_PRE_STORE | 保存前 | `string $html`, `Page $page` | 返回 string 替换 HTML |
| `PAGE_CONTENT_POST_RENDER | 渲染后（含缓存命中） | `string $html`, `Page $page` | 返回 string 替换 HTML |
| `PAGE_INCLUDE_PARSE | 页面引用解析时 | `string $tagReference`, `string $replacementHTML`, `Page $currentPage`, `?Page $referencedPage` | 返回 string 替换引用内容 |
| `COMMONMARK_ENVIRONMENT_CONFIGURE` | Markdown 转 HTML 前 | `Environment $environment` | 返回 Environment 替换 |

### 6.3 主题系统加载流程 (`ThemeServiceProvider.php:25`)

```
App boot
  │
  ▼
ThemeServiceProvider::boot()
  │
  ├─ 注册自定义 Blade @include 指令（支持视图注入）
  │
  ├─ 若无 APP_THEME 配置 → 直接返回
  │
  └─ 有主题配置：
     ├─ loadModules() → 扫描 themes/xxx/modules
     ├─ readThemeActions() → require theme/xxx/functions.php
     ├─ dispatch(APP_BOOT) → 通知主题启动
     ├─ 注册主题视图路径
     └─ dispatch(THEME_REGISTER_VIEWS) → 允许注入自定义视图
```

### 6.4 BookStack 自带监听器情况

**BookStack 核心代码**不注册任何页面渲染相关的默认监听器。
所有监听器都由用户在主题的 `functions.php` 中自行注册。

使用示例（主题 `functions.php`）：
```php
<?php
use BookStack\Facades\Theme;
use BookStack\Theming\ThemeEvents;

// 给所有页面内容末尾追加版权信息
Theme::listen(ThemeEvents::PAGE_CONTENT_POST_RENDER, function($html, $page) {
    return $html . '<p class="copyright">© 2024 My Company</p>';
});

// 修改页面引用的内容
Theme::listen(ThemeEvents::PAGE_INCLUDE_PARSE, function($tag, $html, $current, $referenced) {
    if (!$referenced) return null;
    return '<div class="included-page">' . $html . '</div>';
});
```

### 6.5 其它相关公共事件（扩展点）

**前端公共事件**：
- `library-cm6::pre-init` / `library-cm6::post-init` — 代码编辑器创建前后
- `editor-tinymce::pre-init` / `editor-tinymce::setup` — TinyMCE 初始化前后
- `editor-markdown-cm6::pre-init` — Markdown 编辑器初始化前
- `editor-html-change` / `editor-markdown-change` — 编辑器内容变化

---

## 七、权限校验层级与跨页 Include 权限继承

### 7.1 页面权限校验四层架构

```
┌─────────────────────────────────────────────────────┐
│  ① 角色级权限（Role Permissions）           │
│  page-view-all / page-view-own                │
│  检查当前用户角色是否有全局查看权限             │
└──────────────────┬───────────────────────────────┘
                 │
┌────────────────▼───────────────────────────────┐
│  ② 实体级权限（Entity Permissions）       │
│  通过 joint_permissions 表继承：            │
│  status IN (1,3) OR (owner=me AND status !=2)  │
│  由 PermissionApplicator::restrictEntityQuery │
└──────────────────┬───────────────────────────────┘
                 │
┌────────────────▼───────────────────────────────┐
│  ③ 草稿限制（Draft Restriction）         │
│  draft=false OR (draft=true AND owned_by=me)  │
│  只有作者能看自己的草稿              │
└──────────────────┬───────────────────────────────┘
                 │
┌────────────────▼───────────────────────────────┐
│  ④ 软删除过滤（Soft Deleted Filter）        │
│  deleted_at IS NULL                     │
└───────────────────────────────────────────────┘
```

### 7.2 主页面权限校验实现 (`Entity.php:150`)

```php
public function scopeVisible(Builder $query): Builder
{
    return app()->make(PermissionApplicator::class)->restrictEntityQuery($query);
}
```

`PermissionApplicator::restrictEntityQuery` 核心 SQL 逻辑：

```sql
WHERE EXISTS (
    SELECT 1 FROM joint_permissions
    WHERE joint_permissions.entity_id = entities.id
      AND joint_permissions.entity_type = entities.type
      AND joint_permissions.role_id IN (用户角色ID列表)
    GROUP BY entity_type, entity_id
    HAVING (
        status IN (1, 3)              -- 显式允许，或继承允许
        OR (owner_id = 当前用户ID AND status != 2)  -- 是所有者且未被显式拒绝
    )
)
```

### 7.3 跨页 Include 权限继承规则

**结论：不继承宿主页面权限，每个被引用页面独立做权限校验**

在 `PageContent::getContentProviderClosure` 中：

```php
protected function getContentProviderClosure(bool $blankIncludes): Closure
{
    $contextPage = $this->page;
    $queries = $this->pageQueries;

    return function (PageIncludeTag $tag) use ($blankIncludes, $contextPage, $queries): PageIncludeContent {
        if ($blankIncludes) {
            return PageIncludeContent::fromHtmlAndTag('', $tag);
        }

        // 关键点：用独立的 visible scope 查询被引用页面
        $matchedPage = $queries->findVisibleById($tag->getPageId());
        // ↑ 这里走完整的四层权限校验，与宿主页面无关

        $content = PageIncludeContent::fromHtmlAndTag($matchedPage->html ?? '', $tag);

        // 主题钩子可进一步控制
        if (Theme::hasListeners(ThemeEvents::PAGE_INCLUDE_PARSE)) {
            $themeReplacement = Theme::dispatch(
                ThemeEvents::PAGE_INCLUDE_PARSE,
                $tag->tagContent,
                $content->toHtml(),
                $contextPage,
                $matchedPage
            );
            if (is_string($themeReplacement)) {
                $content = PageIncludeContent::fromHtmlAndTag($themeReplacement, $tag);
            }
        }

        return $content;
    };
}
```

**权限规则**：
1. **宿主页面权限不传递给被引用页面
2. 被引用页面用 `findVisibleById` 做独立权限校验
3. 无权限时返回空字符串（静默失败，不报错）
4. 被引用页面内容也会经过完整净化流程（在宿主页面净化阶段）

### 7.4 `findVisibleById` 实现 (`PageQueries.php:32`)

```php
public function findVisibleById(int $id): ?Page
{
    return $this->start()->scopes('visible')->find($id);
}
```

`visible` scope 叠加了：
- ① 角色权限 + ② 实体权限（通过 `restrictEntityQuery`）
- ③ 草稿过滤（通过 `restrictDraftsOnPageQuery`）
- ④ 软删除过滤（通过 `SoftDeletes` scope）

### 7.5 关键查询入口汇总

| 查询方法 | 场景 | 权限 scope |
|---------|------|------------|
| `findVisibleBySlugsOrFail` | 主页面展示 | visible |
| `findVisibleById` | 页面引用、附件/图片上传上下文 | visible |
| `visibleForList` | 页面列表 | visible |
| `visibleForContent` | 内容查询 | visible |
| `visibleWithContents` | 带 HTML 的列表 | visible |

### 7.6 权限校验时机

| 操作 | 权限检查点 |
|------|----------|
| 查看页面 | `PageController::show` → `findVisibleBySlugsOrFail` |
| 编辑页面 | `PageController::edit` → `findVisibleByIdOrFail` + `userCan('page-update')` |
| 上传图片 | `ImageRepo::saveNewFromData` → `findVisibleByIdOrFail` + `can('image-create-all')` |
| 页面引用 | `PageIncludeParser` → 每个 `{{@id}}` 独立 `findVisibleById` |
| Ajax 取页 | `PageController::getPageAjax` → `findVisibleByIdOrFail` |

---

## 八、关键文件索引

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
| 主题服务（事件系统） | `app/Theming/ThemeService.php` | `listen():37`, `dispatch():54` |
| 主题事件常量 | `app/Theming/ThemeEvents.php` | `PAGE_CONTENT_POST_RENDER:125` |
| 主题服务提供者 | `app/App/Providers/ThemeServiceProvider.php` | `boot():25` |
| 权限应用器 | `app/Permissions/PermissionApplicator.php` | `restrictEntityQuery():99` |
| 实体基类（visible scope） | `app/Entities/Models/Entity.php` | `scopeVisible():150` |
| 页面查询类 | `app/Entities/Queries/PageQueries.php` | `findVisibleById():32` |
| HTML 文档包装器 | `app/Util/HtmlDocument.php` | 全文 |
| 前端-页面编辑器 | `resources/js/components/page-editor.js` | `saveDraft():123` |
| 前端-WYSIWYG 编辑器 | `resources/js/components/wysiwyg-editor.js` | `getContent():65` |
| 前端-TinyMCE 编辑器 | `resources/js/components/wysiwyg-editor-tinymce.js` | `getContent():42` |
| 前端-TinyMCE 配置 | `resources/js/wysiwyg-tinymce/config.js` | `buildForEditor():241` |
| 前端-TinyMCE 粘贴处理 | `resources/js/wysiwyg-tinymce/drop-paste-handling.js` | `paste():35`, `uploadImageFile():15` |
| 前端-TinyMCE 代码块插件 | `resources/js/wysiwyg-tinymce/plugin-codeeditor.js` | 全文 |
| 前端-TinyMCE 过滤器 | `resources/js/wysiwyg-tinymce/filters.js` | `setupFilters():40` |
| 前端-Markdown 编辑器 | `resources/js/components/markdown-editor.js` | `getContent():138` |
| 前端-page-display 组件 | `resources/js/components/page-display.js` | `setup():34` |
| 前端-代码高亮 | `resources/js/code/index.mjs` | `highlight():107`, `highlightElem():62` |
| 前端-代码语言映射 | `resources/js/code/languages.js` | `modeMap:20`, `getLanguageExtension():110` |
| 前端-代码编辑器视图 | `resources/js/code/views.js` | `createView():14`, `updateViewLanguage():45` |
| 只读展示模板 | `resources/views/pages/parts/page-display.blade.php` | 全文 |
| 展示页主模板 | `resources/views/pages/show.blade.php` | `@section('body'):9` |
| TinyMCE 编辑器模板 | `resources/views/pages/parts/wysiwyg-editor-tinymce.blade.php` | 全文 |
| 主题系统文档 | `dev/docs/logical-theme-system.md` | 全文 |
