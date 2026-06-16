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

**④ Word 内容粘贴的格式清理 (`fixes.js:118`)**

粘贴 Word 内容时，TinyMCE 自带 `paste_word_valid_elements` 处理基础格式，但表格选区场景下可能残留大量 Word 私有样式。`handleTableCellRangeEvents` 会监听 `RemoveFormat` 命令，手动清理单元格上的冗余属性：

```javascript
const actionByCommand = {
    RemoveFormat: cell => {
        const attrsToRemove = ['class', 'style', 'width', 'height', 'align'];
        for (const attr of attrsToRemove) {
            cell.removeAttribute(attr);
        }
    },
    // ...
};
```

**⑤ SVG 与 MathML 公式：前端全放行，后端特殊净化**

TinyMCE 的 `extended_valid_elements` 配置 (`config.js:271`)：
```javascript
extended_valid_elements: 'pre[*],svg[*],div[drawio-diagram],details[*],summary[*],div[*],li[class|checked|style]',
```
- `svg[*]` 表示 SVG 标签及其所有属性在编辑器内全部放行（包括 MathML、VML 等 Word 粘贴的矢量图形）
- 保存时不做任何过滤，直接随 HTML 入库
- 真正的净化发生在**后端渲染阶段**（见 4.7 节 SVG 专项净化）

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

### 2.8 大图压缩与缩略图生成管线

图片上传后走 `ImageService → ImageResizer` 两级处理：

**① 上传时即时压缩 (`ImageService::saveNewFromUpload:30`)**

如果调用方传入了 `$resizeWidth`/`$resizeHeight`，保存原图前先压缩：
- `ImageResizer::resizeImageData()` → Intervention Image (GD驱动)
- 保持比例用 `scaleDown()`，裁剪用 `cover()`
- PNG 用 `PngEncoder`，其他用 `AutoEncoder`
- 如果压缩后体积反而更大，返回原图数据（不做负优化）
- 自动读取 EXIF Orientation 并纠正方向

**② 上传后生成两套缩略图 (`ImageRepo::saveNew:112`)**

保存原图后，`ImageResizer::loadGalleryThumbnailsForImage()` 生成两套：

| 缩略图名 | 尺寸 | 类型 | 用途 |
|---------|------|------|------|
| `gallery` | 150×150 | 裁剪 (cover) | 图库列表、编辑器图片选择 |
| `display` | 1680×null | 等比 (scaleDown) | 页面内展示（大图限制宽度） |

缩略图文件结构：
```
uploads/images/gallery/2024-01/
├── photo.jpg                  # 原图
├── thumbs-150-150/
│   └── photo.jpg              # 裁剪缩略图
└── scaled-1680-/
    └── photo.jpg              # 等比缩放图
```

**③ 缓存与懒生成**

缩略图不强制上传时生成，可按需延迟创建：
- 缓存键：`images::{id}::{thumbFilePath}`，缓存 1 周
- 先查缓存 → 再查磁盘 → 都没有才生成
- `shouldCreate=true` 时强制生成（上传时调用）
- 动图（GIF/APNG/AVIF）等比缩放宽高时跳过，返回原图

> **注意**：缩略图生成是 **同步阻塞** 的，不通过 hook 事件机制，直接在 `ImageRepo` 方法里链式调用。
> 主题系统没有图片处理相关的扩展点，只能通过替换 `ImageResizer` 服务实现扩展。

### 2.9 Code-Block：前后端语言识别规则对比

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

**④ 前后端语言识别规则对比**

| 维度 | 前端（CodeMirror 6） | 后端（无） |
|------|---------------------|------------|
| 高亮引擎 | CodeMirror 6 + `@codemirror/lang-*` + legacy mode | **不做语法高亮** |
| 语言识别 | 从 `<pre class="language-xxx">` 或 `<code class="language-xxx">` 提取 | 纯文本提取用 `toPlainText()` 剥所有标签 |
| 语言列表 | 60+ 种，含别名映射（`languages.js:20`） | N/A |
| PHP 特殊处理 | 根据是否含 `<?php` 切换 plain 模式 | N/A |
| 与 highlight.js 关系 | **不共用**，完全独立的 CodeMirror 6 体系 | N/A |

> **重要**：BookStack 的代码高亮是**纯前端行为**，服务端不做任何语法高亮处理。
> 服务端 `HtmlContentFilter` 只把 `<pre><code>` 当作普通 HTML 标签过滤，不识别 `language-*` 类名。
> 前后端**不共享**语言规则，也不使用 highlight.js。

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

### 4.6 SVG 与 Word 公式的专项净化路径

BookStack 对 SVG/MathML/VML 采用 **"前端全放行、后端重点防御"** 的策略：

**前端（TinyMCE）**：
- `extended_valid_elements: 'svg[*],...'` → SVG 及其所有属性全部允许进入编辑器
- Word 粘贴的 MathML 公式、VML 图形都以 SVG/XML 形式进入 DOM
- 保存时不做任何过滤，原样入库

**后端（HtmlContentFilter）**：
- 没有完整的 SVG 白名单，采用**风险点精准打击**策略：
  1. `//svg//@*[contains(., 'data:')]` → 删除 SVG 内所有含 `data:` URI 的属性值
  2. `//svg//@*[contains(., 'javascript:')]` → 删除 SVG 内所有含 `javascript:` 的属性值
  3. `//@*[contains(name(), 'xlink:href')]` → 删除所有 `xlink:href` 属性（SVG 废弃属性，常见 XSS 载体）
  4. `//@*[starts-with(name(), 'on')]` → 删除所有 `on*` 事件属性（包括 SVG 的 `onload`/`onclick` 等）

> **设计思路**：SVG 元素/属性众多，维护完整白名单成本高，因此采用"删已知风险"而非"保留已知安全"策略。
> 如果启用了 `useAllowListFilter`（HTMLPurifier），SVG 会被**整体删除**，因为 HTMLPurifier 默认白名单不包含 SVG。

### 4.7 Hook 扩展点：如何新增白名单标签

**BookStack 主题系统没有直接扩展 HTMLPurifier 白名单的事件。**

可用的扩展方式有三种：

**方式一：通过 `PAGE_CONTENT_PRE_STORE` 预处理**
```php
// themes/your-theme/functions.php
Theme::listen(ThemeEvents::PAGE_CONTENT_PRE_STORE, function($html, $page) {
    // 保存前自定义处理，可添加自定义属性/标签
    return $html;
});
```

**方式二：通过 `PAGE_CONTENT_POST_RENDER` 后处理**
```php
Theme::listen(ThemeEvents::PAGE_CONTENT_POST_RENDER, function($html, $page) {
    // 渲染后注入自定义内容（不受白名单限制，因为已经过净化）
    return $html . '<my-custom-component></my-custom-component>';
});
```

**方式三：替换 `ConfiguredHtmlPurifier` 服务（高级）**
在服务提供者中绑定自定义的 HTMLPurifier 配置类：
```php
$this->app->bind(ConfiguredHtmlPurifier::class, function() {
    return new MyCustomHtmlPurifier();
});
```

> **注意**：`PAGE_CONTENT_POST_RENDER` 钩子在**缓存之后**执行（见 5.9 节），因此通过此钩子添加的自定义标签不会被缓存，每次渲染都会重新执行。

### 4.8 净化触发时机汇总

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

### 5.9 长文章分屏滚动的前端性能预算

**结论：BookStack 没有虚拟滚动/分屏渲染机制，超长文章是全量 DOM 渲染。**

性能优化手段采用**保守型策略**，主要集中在减少滚动触发的重计算：

| 优化手段 | 位置 | 说明 |
|---------|------|------|
| `passive: true` 滚动监听 | `tri-layout.ts:27`, `page-display.js:116` | 滚动事件不阻塞主线程 |
| `IntersectionObserver` 目录高亮 | `page-display.js:78` | 异步判断元素可见性，不阻塞滚动 |
| `debounce` 防抖 | `util.ts:6` | 搜索、内容变化等场景使用，200~1000ms 不等 |
| `requestAnimationFrame` | 多处 | DOM 变更统一合并到下一帧 |
| 代码高亮异步加载 | `page-display.js:35` | `importVersioned('code')` 动态 import，不阻塞首屏 |
| 三栏布局响应式切换 | `tri-layout.ts:33` | 仅在 1000px / 1400px 断点切换，不做实时计算 |

**没有的性能机制**：
- ❌ 虚拟列表 / 窗口化（Virtual Scrolling）
- ❌ 内容分段懒加载
- ❌ 长文章分页
- ❌ 图片懒加载（页面内图片是全量加载的）
- ❌ 显式的性能预算配置

> **性能预期**：几千字的普通文章完全没问题。如果是包含大量代码块、图片的超长文章（数万字），首屏渲染可能会有明显延迟，主要瓶颈在 CodeMirror 6 代码高亮的 DOM 替换和重排。

### 5.10 缓存命中后 Hooks 触发确认

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
| `PAGE_CONTENT_PRE_STORE` | 保存前 | `string $html`, `Page $page` | 返回 string 替换 HTML |
| `PAGE_CONTENT_POST_RENDER` | 渲染后（含缓存命中） | `string $html`, `Page $page` | 返回 string 替换 HTML |
| `PAGE_INCLUDE_PARSE` | 页面引用解析时 | `string $tagReference`, `string $replacementHTML`, `Page $currentPage`, `?Page $referencedPage` | 返回 string 替换引用内容 |
| `COMMONMARK_ENVIRONMENT_CONFIGURE` | Markdown 转 HTML 前 | `Environment $environment` | 返回 Environment 替换 |

> **注意**：没有 `PAGE_CONTENT_HEAD` 这类头部渲染事件。页面头部自定义通过 `CustomHtmlHeadContentProvider` 机制实现（见 6.5 节）。

### 6.3 页面头部自定义机制 (`CustomHtmlHeadContentProvider`)

BookStack 没有 `PAGE_CONTENT_HEAD` 主题事件，头部自定义走独立的 **设置 + 主题模块** 双轨制：

**① 系统设置：`app-custom-head`**
- 后台「设置→自定义」里填写的 HTML 头部内容
- 通过 `setting('app-custom-head')` 读取

**② 主题模块：`head/` 目录**
- 激活主题下 `head/*.html` 文件会被自动拼接到页面头部
- 支持模块（modules）也可以提供 `head/` 目录

**③ 执行链路 (`CustomHtmlHeadContentProvider::forWeb():24`)**

```
getSourceContent()  [app-custom-head 设置]
  │
  ▼
+ getModuleHeadContent()  [主题模块 head/*.html]
  │
  ▼
HtmlNonceApplicator::prepare()  [预处理 nonce 占位符]
  │
  ▼
缓存 1 天 (md5(content) + modulesHash)
  │
  ▼
HtmlNonceApplicator::apply()  [注入实际 CSP nonce]
  │
  ▼
输出到页面 <head>
```

**④ 触发时机与顺序**

`custom-head.blade.php` 在布局模板的 `@stack('head')` **之后**注入：
```blade
<!-- base.blade.php -->
@stack('head')
@include('layouts.parts.custom-head')
```

执行顺序：
1. 各子视图 `@push('head')` 的内容（如页面特定 CSS/JS）
2. `CustomHtmlHeadContentProvider::forWeb()` 的内容（设置 + 主题模块）

**⑤ 导出模式下的头部净化 (`forExport():40`)**
导出 PDF/HTML 时，头部内容会走简化版过滤：
```php
$config = new HtmlContentFilterConfig(
    filterOutNonContentElements: false,
    useAllowListFilter: false
);
return (new HtmlContentFilter($config))->filterString($content);
```
只保留非内容元素过滤，跳过脚本/表单/白名单过滤（因为导出场景需要完整样式）。

### 6.4 主题系统加载流程 (`ThemeServiceProvider.php:25`)

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

### 7.4 三层递归 Include 的权限叠加机制

**递归解析循环 (`PageContent::render():327`)**：

```php
$doc = $this->getHtmlDocument();
$contentProvider = $this->getContentProviderClosure($blankIncludes);
$parser = new PageIncludeParser($doc, $contentProvider);

$nodesAdded = -1;
for ($includeDepth = 0; $includeDepth < 3 && $nodesAdded !== 0; $includeDepth++) {
    $nodesAdded = $parser->parse();
}

if ($includeDepth > 1) {
    $this->formatHtml($doc);  // 多层嵌套时重新规范所有锚点ID
}
```

**循环规则**：
- 上限：最多 3 层（`$includeDepth < 3`）
- 终止条件：某一轮没有新增节点（`$nodesAdded === 0`）
- 每层都调用同一个 `PageIncludeParser` 实例，在上一轮的结果上继续解析
- 超过 1 层嵌套时，最后统一重新规范锚点 ID，避免 ID 冲突

**每层的权限校验**：

| 层级 | 页面 | 权限校验方式 |
|------|------|-------------|
| 第 0 层（宿主） | 页面 A | `findVisibleBySlugsOrFail` → 四层权限 |
| 第 1 层（直接引用） | 页面 B | `findVisibleById(B)` → 独立四层权限 |
| 第 2 层（间接引用） | 页面 C | `findVisibleById(C)` → 独立四层权限 |

**权限叠加函数**：

**没有权限叠加**。每一层的页面都是**独立查询**，各自走完整的四层权限校验：
- 宿主 A 的权限不影响被引用 B 的权限判断
- 被引用 B 的权限不影响被引用 C 的权限判断
- 任意一层无权限 → 该层内容为空 → 更深层引用自然也不存在

> **安全边界**：递归 3 层上限 + 每层独立权限校验，既防止了循环引用死循环，
> 也确保了"能看到 A 不代表能看到 A 引用的 B"的权限隔离。

### 7.5 `findVisibleById` 实现 (`PageQueries.php:32`)

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

### 7.6 关键查询入口汇总

| 查询方法 | 场景 | 权限 scope |
|---------|------|------------|
| `findVisibleBySlugsOrFail` | 主页面展示 | visible |
| `findVisibleById` | 页面引用、附件/图片上传上下文 | visible |
| `visibleForList` | 页面列表 | visible |
| `visibleForContent` | 内容查询 | visible |
| `visibleWithContents` | 带 HTML 的列表 | visible |

### 7.7 权限校验时机

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
| HTML 文档包装器 | `app/Util/HtmlDocument.php` | 全文 |
| 编辑页数据装配 | `app/Entities/Tools/PageEditorData.php` | `build():37` |
| 编辑器类型枚举 | `app/Entities/Tools/PageEditorType.php` | 全文 |
| 主题服务（事件系统） | `app/Theming/ThemeService.php` | `listen():37`, `dispatch():54` |
| 主题事件常量 | `app/Theming/ThemeEvents.php` | `PAGE_CONTENT_POST_RENDER:125` |
| 主题服务提供者 | `app/App/Providers/ThemeServiceProvider.php` | `boot():25` |
| 自定义头部内容提供者 | `app/Theming/CustomHtmlHeadContentProvider.php` | `forWeb():24`, `forExport():40` |
| 权限应用器 | `app/Permissions/PermissionApplicator.php` | `restrictEntityQuery():99` |
| 实体基类（visible scope） | `app/Entities/Models/Entity.php` | `scopeVisible():150` |
| 页面查询类 | `app/Entities/Queries/PageQueries.php` | `findVisibleById():32` |
| 图片服务 | `app/Uploads/ImageService.php` | `saveNewFromUpload():30`, `saveNewFromBase64Uri():54` |
| 图片仓库 | `app/Uploads/ImageRepo.php` | `saveNewFromData():134` |
| 图片缩放器 | `app/Uploads/ImageResizer.php` | `resizeImageData():118`, `loadGalleryThumbnailsForImage():42` |
| 前端-页面编辑器 | `resources/js/components/page-editor.js` | `saveDraft():123` |
| 前端-WYSIWYG 编辑器 | `resources/js/components/wysiwyg-editor.js` | `getContent():65` |
| 前端-TinyMCE 编辑器 | `resources/js/components/wysiwyg-editor-tinymce.js` | `getContent():42` |
| 前端-TinyMCE 配置 | `resources/js/wysiwyg-tinymce/config.js` | `buildForEditor():241` |
| 前端-TinyMCE 粘贴处理 | `resources/js/wysiwyg-tinymce/drop-paste-handling.js` | `paste():35`, `uploadImageFile():15` |
| 前端-TinyMCE 修复 | `resources/js/wysiwyg-tinymce/fixes.js` | `handleTableCellRangeEvents():102` |
| 前端-TinyMCE 代码块插件 | `resources/js/wysiwyg-tinymce/plugin-codeeditor.js` | 全文 |
| 前端-TinyMCE 过滤器 | `resources/js/wysiwyg-tinymce/filters.js` | `setupFilters():40` |
| 前端-Markdown 编辑器 | `resources/js/components/markdown-editor.js` | `getContent():138` |
| 前端-page-display 组件 | `resources/js/components/page-display.js` | `setup():34` |
| 前端-三栏布局 | `resources/js/components/tri-layout.ts` | `updateLayout():33`, `setupDesktop():65` |
| 前端-代码高亮 | `resources/js/code/index.mjs` | `highlight():107`, `highlightElem():62` |
| 前端-代码语言映射 | `resources/js/code/languages.js` | `modeMap:20`, `getLanguageExtension():110` |
| 前端-代码编辑器视图 | `resources/js/code/views.js` | `createView():14`, `updateViewLanguage():45` |
| 前端-工具函数 | `resources/js/services/util.ts` | `debounce():8` |
| 只读展示模板 | `resources/views/pages/parts/page-display.blade.php` | 全文 |
| 展示页主模板 | `resources/views/pages/show.blade.php` | `@section('body'):9` |
| TinyMCE 编辑器模板 | `resources/views/pages/parts/wysiwyg-editor-tinymce.blade.php` | 全文 |
| 自定义头部模板 | `resources/views/layouts/parts/custom-head.blade.php` | 全文 |
| 主题系统文档 | `dev/docs/logical-theme-system.md` | 全文 |

---

## 九、代码级实现细节勘误

> 以下逐一对照代码核实用户提出的实现细节。部分概念在 BookStack 中**不存在对应实现**，如实记录。

### 9.1 本地无 MathJax 时的降级路径

**结论：BookStack 不内置 MathJax，不存在相关降级路径。**

搜索范围：
- 全项目 Grep `mathjax|katex|math.*render` → 命中仅 TinyMCE 压缩包内字符串，无业务代码
- `resources/js/` 无数学公式渲染模块
- `app/` 无 MathML 处理逻辑
- `composer.json` / `package.json` 无 math 相关依赖

**实际行为**：
- Word 粘贴的公式以 OMML（Office MathML）形式进入 TinyMCE
- TinyMCE 将其转为标准 HTML/SVG 片段（`svg[*]` 放行策略）
- 保存入库的是 SVG 标记，不是 MathML
- 只读视图中公式显示为静态 SVG，无 JavaScript 运行时渲染
- 如果用户需要 MathJax，只能通过「设置→自定义 HTML 头部」注入 CDN 脚本，不属于核心管线

### 9.2 SVG 外链 `<use href>` 的 SSRF 风险与防护

**结论：存在已知风险点，但 BookStack 的防护是间接的、不完整。**

`<svg><use href="http://internal-server/..."/></svg>` 是经典 SSRF 向量——浏览器会请求外部 URL 加载 SVG 片段。

**BookStack 已有的防护**：

| 防护层 | 代码位置 | 覆盖场景 |
|--------|---------|---------|
| `xlink:href` 全删 | `HtmlContentFilter.php:81` `//@*[contains(name(), 'xlink:href')]` | 覆盖 SVG 1.1 的 `xlink:href`（已废弃） |
| SVG 属性中 `data:`/`javascript:` 删除 | `HtmlContentFilter.php:76` | 覆盖属性值含 `data:`/`javascript:` 的情况 |
| `<iframe src>` 外链检查 | `ConfiguredHtmlPurifier.php` iframe 正则 `%^(http://|https://|//)%` | 仅限 iframe，不影响 SVG |
| HTMLPurifier URI 策略 | `ConfiguredHtmlPurifier.php` 允许 `http https mailto ...` | 仅限白名单过滤启用时 |

**未覆盖的缺口**：

```html
<!-- 这段不会被任何现有规则拦截： -->
<svg><use href="http://internal-server/admin"/></svg>
<!-- 1. href 属性不在 xlink:href 过滤范围 -->
<!-- 2. href 值不含 data: 或 javascript:，不触发属性值过滤 -->
<!-- 3. SVG use 不走 iframe 的 href 白名单 -->
<!-- 4. 若未启用 useAllowListFilter (a)，HTMLPurifier 不参与 -->
```

**风险评级**：中等。需要攻击者能在页面中插入任意 SVG（需要编辑权限），且服务端在内网有可达资源。默认配置 `jhfa` 中若启用 `a`（HTMLPurifier），SVG 整体会被删除从而消除此风险。

### 9.3 Intervention/Image 在 GD 与 Imagick 后端的兼容差异

**结论：BookStack 硬编码使用 GD 驱动，不支持 Imagick 后端。**

`ImageResizer.php:163-168`：
```php
if (!extension_loaded('gd')) {
    throw new ImageUploadException('The PHP "gd" extension is required to resize images, but is missing.');
}
$manager = new ImageManager(
    new Driver(),   // ← 硬编码 Gd\Driver
    autoOrientation: false,
);
```

| 维度 | GD（当前使用） | Imagick（未使用） |
|------|--------------|-----------------|
| 动画 GIF | `NativeObjectDecoder` + `imagecreatefromstring()` 特殊处理 | Imagick 原生支持 |
| EXIF 方向 | 手动 `exif_read_data()` + 旋转（`orientImageToOriginalExif`） | Intervention `autoOrientation` 已关闭，手动处理同 GD |
| 内存占用 | 较高（全量解码到内存） | 更高（Imagick 对象更大） |
| 格式支持 | `jpg jpeg png gif webp avif` | 更多格式 |
| APNG/AVIF 动画检测 | 手动二进制解析（`isApngData`/`isAnimatedAvifData`） | Imagick 可用 `getImageDelay()` |

**不存在运行时切换后端的逻辑**，也无配置项。若要切换需修改 `ImageResizer` 源码。

### 9.4 highlight.js 前后端版本不一致时的对账方案

**结论：BookStack 不使用 highlight.js，不存在前后端版本对账问题。**

搜索范围：
- 全项目 Grep `highlight\.js|hljs` → 仅命中 `debugbar.php` 配置（Laravel Debugbar 的可选功能）
- 代码高亮使用 **CodeMirror 6**（`resources/js/code/`），不是 highlight.js
- 服务端**不做语法高亮**（见 2.9 节）
- 前端 CodeMirror 6 语言包版本由 `package.json` 锁定，不存在前后端版本对账需求

**版本管理方式**：
- CodeMirror 及其语言扩展通过 `package.json` + `package-lock.json` 统一管理
- 构建时打包进 `dist/`，运行时加载单一 bundle
- 语言映射表 `languages.js` 是前端静态配置，不依赖服务端

### 9.5 IntersectionObserver 低端机降级到节流的阈值与判定

**结论：BookStack 不做 IntersectionObserver 降级，无阈值判定逻辑。**

`page-display.js:78` 的目录滚动高亮直接使用 `IntersectionObserver`：
```javascript
const pageNavObserver = new IntersectionObserver(headingVisibilityChange, {
    rootMargin: '0px 0px 0px 0px',
    threshold: 1.0,
});
```

**没有降级逻辑**：
- ❌ 无 `typeof IntersectionObserver === 'undefined'` 检测
- ❌ 无 `window.IntersectionObserver` polyfill
- ❌ 无备用的 `scroll` + `throttle` 方案
- ❌ 无低端机特征检测（UA/内存/CPU 核心）

**影响**：在不支持 `IntersectionObserver` 的浏览器（IE11、极老版 Android WebView）上，目录高亮功能静默失效，但不影响页面内容渲染——功能降级为"无高亮"，不存在崩溃风险。

### 9.6 ServiceProvider 注册扩展运行时禁用时的清理顺序

**结论：BookStack 的主题 ServiceProvider 不支持运行时动态禁用/清理。**

`ThemeServiceProvider::boot()` 的工作模式：
```
App boot
  ├─ 若无 APP_THEME → 直接 return（不注册任何东西）
  └─ 有主题配置 →
      ├─ loadModules() → 扫描目录
      ├─ readThemeActions() → require functions.php（监听器注册）
      ├─ dispatch(APP_BOOT) → 一次性通知
      ├─ 注册视图路径 → view()->addNamespace()
      └─ dispatch(THEME_REGISTER_VIEWS)
```

**关键特性**：
1. **单次启动**：`boot()` 只执行一次，注册的监听器、视图命名空间等无法撤销
2. **无 `register()` 方法**：不使用 Laravel 的 deferred provider 机制
3. **无 `provides()` 方法**：不声明可延迟加载的服务
4. **监听器不可取消**：`ThemeService::listen()` 只做 `$this->listeners[$event][] = $action`，没有 `forgetListener`/`removeListener`

**禁用主题的方式**：只能通过修改 `APP_THEME` 环境变量后重启应用。不支持运行时热切换。

### 9.7 PAGE_CONTENT_HEAD 与 PAGE_BEFORE_DISPLAY 的相对触发顺序

**结论：这两个事件在 BookStack 中都不存在。**

搜索范围：
- `ThemeEvents.php` 完整常量列表：无 `PAGE_CONTENT_HEAD`，无 `PAGE_BEFORE_DISPLAY`
- 全项目 Grep `PAGE_BEFORE_DISPLAY|PAGE_CONTENT_HEAD` → 仅命中 `page-render.md`（本文档之前的描述）

**BookStack 实际存在的事件**（完整列表）：

| 事件 | 触发时机 |
|------|---------|
| `APP_BOOT` | 应用启动后 |
| `COMMONMARK_ENVIRONMENT_CONFIGURE` | Markdown 转换前 |
| `PAGE_INCLUDE_PARSE` | 页面引用解析时 |
| `PAGE_CONTENT_PRE_STORE` | 内容保存前 |
| `PAGE_CONTENT_POST_RENDER` | 内容渲染后 |
| `WEB_MIDDLEWARE_BEFORE` | Web 中间件前 |
| `WEB_MIDDLEWARE_AFTER` | Web 中间件后 |
| `OIDC_ID_TOKEN_PRE_VALIDATE` | OIDC ID Token 验证前 |
| `ROUTES_REGISTER_WEB` | Web 路由注册时 |
| `ROUTES_REGISTER_WEB_AUTH` | 认证路由注册时 |
| `THEME_REGISTER_VIEWS` | 主题视图注册时 |

**页面头部自定义**通过 `CustomHtmlHeadContentProvider`（见 6.3 节）实现，不是通过事件钩子。

### 9.8 EntityPermissionEvaluator 权限链递归终止条件

**结论：权限链不是递归实现，而是有限长度的数组遍历，终止条件是链的固定结构。**

`EntityPermissionEvaluator::gatherEntityChainTypeIds()` (`EntityPermissionEvaluator.php:139`)：

```php
protected function gatherEntityChainTypeIds(SimpleEntityData $entity): array
{
    $chain = [$entity->type . ':' . $entity->id];

    if ($entity->type === 'page' && $entity->chapter_id) {
        $chain[] = 'chapter:' . $entity->chapter_id;
    }

    if ($entity->type === 'page' || $entity->type === 'chapter') {
        $chain[] = 'book:' . $entity->book_id;
    }

    return $chain;
}
```

**链的固定结构**（最多 3 级，无递归）：

| 实体类型 | 权限链 |
|---------|--------|
| Page（有章节） | `page:id` → `chapter:id` → `book:id` |
| Page（无章节） | `page:id` → `book:id` |
| Chapter | `chapter:id` → `book:id` |
| Book | `book:id` |
| Bookshelf | `bookshelf:id` |

**终止条件控制变量**：

```php
// collapseAndCategorisePermissions 中的提前终止
protected function collapseAndCategorisePermissions(array $typeIdChain, array $permissionMapByTypeId): array
{
    $permitsByType = ['fallback' => [], 'role' => []];

    foreach ($typeIdChain as $typeId) {
        // ...收集权限...
        
        // 关键终止条件：找到 fallback 权限即停止向上遍历
        if (isset($permitsByType['fallback'][0])) {
            break;   // ← 这是唯一的提前终止点
        }
    }

    return $permitsByType;
}
```

**终止机制总结**：
1. **结构性终止**：链长度由实体类型决定，最多 3 级，不会无限延伸
2. **语义性终止**：遍历链时如果找到 `fallback` 权限（`role_id = 0`），立即 `break`——上级实体的权限不再参考
3. **没有 `mergePermissions` 递归函数**：权限评估是迭代式链遍历，不是递归合并
4. **优先级**：链前端优先 → 页面自身权限 > 章节权限 > 书本权限；`role` 级权限 > `fallback` 级权限

### 9.9 边界条件源码级索引表

以下汇总 7 个边界问题的精确源码位置和关键参数：

| 边界问题 | 源码文件 | 行号 | 关键代码 | 备注 |
|---------|---------|------|---------|------|
| **MathJax 降级** | `app/Theming/CustomHtmlHeadContentProvider.php` | 24 | `forWeb()` | 无内置 MathJax，公式转静态 SVG。需自定义头部注入 CDN。 |
| **SVG `<use href>` SSRF 防护** | `app/Util/HtmlContentFilter.php` | 76, 81 | `//svg//@*[contains(., 'data:')]` / `//@*[contains(name(), 'xlink:href')]` | 防护不完整，默认 `jhfa` 中 `a` 选项会整体删除 SVG。 |
| **GD 与 Imagick 切换** | `app/Uploads/ImageResizer.php` | 163-168 | `new Driver()` (Gd\Driver 硬编码) | 不支持 Imagick，无配置项切换，需改源码。 |
| **highlight.js 版本对账** | N/A | N/A | N/A | 不使用 highlight.js，用 CodeMirror 6，服务端不做高亮，无对账需求。 |
| **IntersectionObserver 降级阈值** | `resources/js/components/page-display.js` | 78 | `threshold: 1.0` | 无降级逻辑。现有 `debounce()` 阈值由调用方传入，搜索用 200ms、内容变化用 500ms。 |
| **扩展运行时禁用清理** | `app/App/Providers/ThemeServiceProvider.php` | 25-40 | `boot()` | 不支持运行时禁用，监听器注册后无法注销，无清理顺序。 |
| **权限链终止关键变量** | `app/Permissions/EntityPermissionEvaluator.php` | 69-71 | `if (isset($permitsByType['fallback'][0])) break;` | 找到 `role_id = 0` 的 fallback 权限即停止向上遍历。 |
| **PermissionStatus 常量** | `app/Permissions/PermissionStatus.php` | 7-10 | `IMPLICIT_DENY=0, IMPLICIT_ALLOW=1, EXPLICIT_DENY=2, EXPLICIT_ALLOW=3` | 联合权限状态枚举，`max(status)` 用于 role 级冲突解决。 |

### 9.10 `debounce` 节流阈值在各场景的取值

BookStack 前端唯一的节流工具是 `util.ts:8` 的 `debounce()`，各调用场景的阈值取舍：

| 场景 | 文件 | 阈值 | 说明 |
|------|------|------|------|
| 搜索输入 | `components/search.js` | 200ms | 快速响应用户输入 |
| 编辑器内容变化 | `components/page-editor.js` | 500ms | 平衡响应速度与服务器压力 |
| 自动保存草稿 | `components/page-editor.js` | 30s | 定时保存，避免频繁提交 |
| 布局响应式切换 | `components/tri-layout.ts` | 100ms | 窗口 resize 防抖 |
| 目录滚动高亮 | N/A | N/A | 直接用 IntersectionObserver，不节流 |

> 如果要给 IntersectionObserver 做低端机降级，参考阈值可设为 `debounce(callback, 100ms)`，与布局切换一致。

### 9.11 联合权限状态合并逻辑

`JointPermissionBuilder::createJointPermissionData()` 中无 `mergePermissions` 递归函数，但有**角色权限 vs 实体权限**的合并逻辑：

```php
// EntityPermissionEvaluator.php:35-47
protected function evaluatePermitsByType(array $permitsByType): ?int
{
    // ① 角色级权限优先（显式设置）
    if (count($permitsByType['role']) > 0) {
        return max($permitsByType['role'])
            ? PermissionStatus::EXPLICIT_ALLOW
            : PermissionStatus::EXPLICIT_DENY;
    }

    // ② fallback 权限（role_id = 0，适用于所有角色）
    if (count($permitsByType['fallback']) > 0) {
        return $permitsByType['fallback'][0]
            ? PermissionStatus::IMPLICIT_ALLOW
            : PermissionStatus::IMPLICIT_DENY;
    }

    return null;  // ③ 无实体权限 → 回退到角色全局权限
}
```

**终止条件的控制变量**：
- **结构性**：`$typeIdChain` 数组长度（最多 3）
- **语义性**：`isset($permitsByType['fallback'][0])` → `break`
- **角色冲突**：`max($permitsByType['role'])` → 任意角色显式允许即允许（但通常同一实体同一角色只有一条权限记录，不会冲突）
