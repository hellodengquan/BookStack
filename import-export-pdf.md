# BookStack 导入导出引擎代码路径详解

本文档系统梳理 BookStack 中页面、章节、整本书转 PDF 对外发布，以及外部 Markdown/Word/HTML 文档反向导回知识库的完整代码处理路径，包括各阶段约定与边角处理。

---

## 第一部分：导出（Export）流程

### 1.1 导出入口：Controller 层

三个层级（Page/Chapter/Book）分别对应三个 Controller，结构完全对称：

| 层级 | Controller 文件 |
|------|----------------|
| 页面 | `app/Exports/Controllers/PageExportController.php` |
| 章节 | `app/Exports/Controllers/ChapterExportController.php` |
| 书籍 | `app/Exports/Controllers/BookExportController.php` |

**共同约定**（所有 Controller 构造函数中）：
- 应用 `Permission::ContentExport` 中间件做权限校验
- 应用 `throttle:exports` 限流中间件

每个 Controller 提供 5 种导出格式路由：
- `pdf()` — PDF 文件
- `html()` — 自包含 HTML（内嵌 base64 图片）
- `plainText()` — 纯文本 TXT
- `markdown()` — Markdown
- `zip()` — ZIP 归档（含附件、图片、结构化 JSON）

以书籍导出 PDF 为例，调用链路：
```
BookExportController::pdf($bookSlug)
  ↓
$this->queries->findVisibleBySlugOrFail($bookSlug)   // 权限可见性查询
  ↓
$this->exportFormatter->bookToPdf($book)              // 核心格式化
  ↓
return $this->download()->directly($pdfContent, $bookSlug . '.pdf')
```

> 代码位置：`app/Exports/Controllers/BookExportController.php:28-34`

---

### 1.2 内容格式化核心：ExportFormatter

**文件**：`app/Exports/ExportFormatter.php`

这是所有导出格式的统一入口，负责把 Entity 模型序列化为目标格式。

#### 1.2.1 内容树构建约定

| 导出层级 | 内容树构建方式 | 代码位置 |
|---------|---------------|---------|
| Page | `(new PageContent($page))->render()` 渲染单页 HTML | `ExportFormatter.php:36` |
| Chapter | `$chapter->getVisiblePages()` 获取可见页面，逐个渲染 | `ExportFormatter.php:54-57` |
| Book | `(new BookContents($book))->getTree(false, true)` 构建整树 | `ExportFormatter.php:76` |

**BookContents 构建规则**（`app/Entities/Tools/BookContents.php:41-71`）：
- 递归收集所有 visible scope 的 Chapter 和 Page
- 按 `priority` 字段升序排序；draft 页面 priority=-100 排在最前（但导出时 `$showDrafts=false` 会被过滤掉）
- `$renderPages=true` 时对每个 Page 调用 `PageContent::render()` 预渲染 HTML
- 章节通过 `chapter_id` 关联页面，无章节归属的页面作为 "lone pages" 直接挂在 Book 下

#### 1.2.2 三种 PDF 生成引擎

`PdfGenerator::fromHtml()` 会按优先级选择引擎（`app/Exports/PdfGenerator.php:23-30`）：

```php
return match ($this->getActiveEngine()) {
    self::ENGINE_COMMAND => $this->renderUsingCommand($html),   // 优先级1: 自定义命令
    self::ENGINE_WKHTML  => $this->renderUsingWkhtml($html),    // 优先级2: WKHTMLtoPDF
    default              => $this->renderUsingDomPdf($html)     // 默认: DOMPDF
};
```

**引擎选择判定**（`PdfGenerator.php:36-47`）：
1. 若 `exports.pdf_command` 配置非空 → COMMAND
2. 若有 WKHTMLTOPDF 二进制路径 **且** `app.allow_untrusted_server_fetching=true` → WKHTML
3. 否则 → DOMPDF

**各引擎细节**：

| 引擎 | 配置 | 特殊处理 |
|------|------|---------|
| DOMPDF | `exports.dompdf.*` | 自动扫描 `storage/fonts/dompdf/*.ttf` 并生成 .ufm 字体度量文件；chroot 限制在 `public_path()` 内；支持 CSS 变量颜色补丁（链接色、引用左边框色） |
| WKHTML | `exports.snappy.*` | 2024-04 标记为 deprecated；启用 print-media-type 和 outline 书签 |
| COMMAND | `exports.pdf_command` + `exports.pdf_command_timeout` (默认15s) | 用 `{input_html_path}` 和 `{output_pdf_path}` 占位符；超时或非 0 退出码抛 `PdfExportException`；始终清理临时文件 |

#### 1.2.3 HTML→PDF 必经的预处理

在交给 PDF 引擎前，`ExportFormatter::htmlToPdf()` 做了两个关键 DOM 变换（`ExportFormatter.php:153-164`）：

1. **展开 `<details>` 折叠块**：`openDetailElements()` 遍历所有 `<details>` 标签，强制加 `open="open"` 属性，避免 PDF 中折叠内容不可见。
2. **替换 `<iframe>` 为纯链接**：`replaceIframesWithLinks()` 将 iframe 替换为 `<p><a href="...">url</a></p>`，因为 PDF 引擎无法渲染 iframe。协议相对 URL（`//` 开头）会补全为 `https:`。

#### 1.2.4 containHtml：资源自包含化

`ExportFormatter::containHtml()`（`ExportFormatter.php:206-242`）做两件事：

**① 图片转 base64 内嵌**
```
正则匹配所有 <img src="...">
  ↓
ImageService::imageUrlToBase64($srcString)
  ↓ 失败（如外链）则保留原 URL
替换为 data:image/xxx;base64,...
```

**② 相对链接绝对化**
```
正则匹配所有 <a href="...">
  ↓
非 http 开头的 → url($srcString) 补全为 APP_URL 绝对地址
```

这样导出的 HTML/PDF 不依赖服务器即可离线查看。

---

### 1.3 视图层：导出模板结构

Blade 模板位于 `resources/views/exports/`：

```
resources/views/exports/
├── book.blade.php          # 整本书导出布局
├── chapter.blade.php       # 单章节导出布局
├── page.blade.php          # 单页面导出布局
├── import.blade.php        # 导入上传页
├── import-show.blade.php   # 待确认导入预览页
└── parts/
    ├── styles.blade.php            # 内联样式（含 CSS 变量补丁）
    ├── meta.blade.php              # 元信息（创建/更新时间、作者）
    ├── book-contents-menu.blade.php # 书籍目录页 TOC
    ├── chapter-contents-menu.blade.php
    ├── chapter-item.blade.php      # 章节内容项
    ├── page-item.blade.php         # 页面内容项
    ├── custom-head.blade.php       # 主题系统自定义 head
    └── import*.blade.php
```

**书籍导出结构**（`book.blade.php`）：
```
<h1> 书名
书籍描述 HTML
@include('book-contents-menu')   ← 目录页（TOC），锚点指向 #chapter-id / #page-id
@foreach($bookChildren)
  若是 chapter → @include(chapter-item)  # 章节标题 + 描述 + 遍历 pages
  若是 page    → @include(page-item)     # 页面标题 + 渲染后的 HTML
@endforeach
```

**样式注入约定**（`parts/styles.blade.php`）：
- 读 `public/dist/export-styles.css` 内联到 `<style>` 标签
- PDF 格式额外注入：`a { color: {{setting('app-link')}} }` 和 `blockquote { border-left-color: {{setting('app-color')}} }`，因为 DOMPDF/WKHTML 对 CSS 变量支持有限

---

### 1.4 ZIP 导出：结构化打包

ZIP 导出是唯一保留附件、图片、标签、内部引用关系的导出格式，用于跨实例迁移。

#### 1.4.1 ZIP 内部结构约定

由 `ZipExportBuilder::build()` 生成（`app/Exports/ZipExports/ZipExportBuilder.php:67-117`）：

```
export.zip
├── data.json          # 结构化元数据（书/章/页树 + 附件/图片引用）
└── files/
    ├── <random20>.<ext>    # 附件文件（随机名避免冲突）
    └── <random20>.<ext>    # 图片文件
```

**data.json 顶层结构**：
```json
{
  "exported_at": "2024-01-01T00:00:00+00:00",
  "instance": { "id": "...", "version": "24.xx" },
  "book|chapter|page": { /* 对应层级的 ZipExport* 模型序列化结果 */ }
}
```

#### 1.4.2 文件引用与随机命名

`ZipExportFiles`（`app/Exports/ZipExports/ZipExportFiles.php`）管理文件命名：
- 每个附件/图片生成 `Str::random(20) + '.' . extension` 随机文件名
- 循环检测确保不与已分配的文件名冲突
- 按 `attachmentId → ref` 和 `imageId → ref` 两张 Map 去重，同个文件多次引用只存一份

#### 1.4.3 内部引用转换：[[bsexport:type:id]]

`ZipExportReferences::buildReferences()`（`ZipExportReferences.php:80-112`）把内容中的绝对 URL 替换为占位符：

```
page.html 中的链接/图片 → ZipReferenceParser::parseLinks()
  ↓
匹配 APP_URL 和图片 CDN URL 正则
  ↓
通过 ModelResolvers 链解析为对应 Model（PagePermalink→PageLink→Chapter→Book→Image→Attachment）
  ↓
若 Model 在本次导出范围内 → 替换为 [[bsexport:page:123]] 占位符
  ↓ 占位符格式
[[bsexport:book:<id>]] / [[bsexport:chapter:<id>]] / [[bsexport:page:<id>]]
[[bsexport:image:<id>]] （仅 gallery 和 drawio 类型）
[[bsexport:attachment:<id>]]
```

**图片处理的特殊约定**：
- 仅 `gallery` 和 `drawio` 类型图片被纳入（用户头像、系统图等排除）
- 需额外通过 `imageAccessible()` 权限检查
- 图片会被挂到其所属 Page 的 `images[]` 数组中

**附件处理约定**：
- 外部链接（`external=true`）→ 存 `link` 字段（URL 字符串）
- 内部文件 → 存 `file` 字段（指向 files/ 下的随机名引用）

#### 1.4.4 各层级数据模型

所有模型继承 `ZipExportModel` 抽象类（`app/Exports/ZipExports/Models/ZipExportModel.php`），约定了三个核心方法：
- `validate()` — 导入时校验
- `fromArray()` — 从 data.json 反序列化
- `metadataOnly()` — 清空正文内容只保留 id/name（用于 Import 列表预览）
- `jsonSerialize()` — 序列化时自动过滤 null 字段

| 模型 | 关键字段 |
|------|---------|
| ZipExportBook | name, description_html, cover (图片引用), chapters[], pages[], tags[] |
| ZipExportChapter | name, description_html, priority, pages[], tags[] |
| ZipExportPage | name, html, markdown, priority, attachments[], images[], tags[] |
| ZipExportAttachment | name, link (外链) / file (文件引用) — 二选一 |
| ZipExportImage | name, file, type ∈ {gallery, drawio} |
| ZipExportTag | name, value |

#### 1.4.5 ZIP 构建的异常清理

`ZipExportBuilder::build()` 的 try-catch 中：
- 出错时遍历已添加到 ZIP 的 entry 名逐个 `deleteName()`
- 遍历已写入的临时文件逐个 `unlink()`
- 最后关闭 ZIP 并删除 ZIP 文件本身
- 抛 `ZipExportException` 附带上游错误信息

---

## 第二部分：导入（Import）流程

### 2.1 导入生命周期：三阶段

```
上传 → 校验存储 → 预览确认 → 执行导入 → 清理
  │        │           │          │        │
  │   storeFromUpload  │      runImport   deleteImport
  │        │           │          │        │
  ▼        ▼           ▼          ▼        ▼
 ImportController   ImportRepo   ZipImportRunner
```

#### 2.1.1 阶段一：上传与校验

`ImportRepo::storeFromUpload()`（`app/Exports/ImportRepo.php:73-112`）：
```
接收 UploadedFile
  ↓
new ZipExportReader($zipPath)
  ↓
(new ZipExportValidator($reader))->validate()
  ├── ZipExportReader::readData() 读取 data.json
  │     └── data.json 大小受 app.upload_limit 限制（MB）
  └── 根据顶层 key 分发到 ZipExportBook/Chapter/Page::validate()
        ├── Laravel Validation 规则校验字段类型/必填
        ├── ZipUniqueIdRule：同类型 id 全局唯一（如不能有两个 page:5）
        └── ZipFileReferenceRule：引用的 files/xxx 存在且（图片）MIME 合法
  ↓ 校验失败抛 ZipValidationException，返回给前端显示
decodeDataToExportModel() → 决定 import.type ∈ {book, chapter, page}
  ↓
$exportModel->metadataOnly() → 只保留名称/id 存到 import.metadata
  ↓
文件存到 uploads/files/imports/ 目录
  ↓
写入 imports 表记录（type/name/size/path/metadata/created_by）
```

**Import 表模型**（`app/Exports/Import.php`）：
- `decodeMetadata()` 可从 JSON metadata 反序列化回 ZipExport* 模型（但已 metadataOnly，不含正文）
- 非管理员只能看到自己创建的 Import（`ImportRepo.php:46-55`）

#### 2.1.2 阶段二：预览与确认

`ImportController::show()`（`ImportController.php:62-72`）：
- 读取 Import 记录
- `decodeMetadata()` 得到精简结构（仅 id/name）
- 传给 `exports.import-show` 视图展示层级树
- Page/Chapter 导入会额外显示父级选择下拉框

#### 2.1.3 阶段三：执行导入

`ImportRepo::runImport()`（`ImportRepo.php:117-138`）：
```
解析 parent（Page/Chapter 需要，Book 不需要）
  ↓
DB::beginTransaction()
  ↓
ZipImportRunner::run($import, $parentModel)
  ↓ 异常 → DB::rollBack() + revertStoredFiles() 清理已存文件
DB::commit()
  ↓
deleteImport() 删除 ZIP 文件和 imports 表记录
  ↓
返回新建的顶级 Entity URL 跳转
```

---

### 2.2 ZipImportRunner：实体树重建

**文件**：`app/Exports/ZipExports/ZipImportRunner.php`

#### 2.2.1 前置检查

`ZipImportRunner::run()` 开头做四重检查：
1. **重新校验 ZIP**（防止上传后文件被篡改）
2. **父级类型匹配**：
   - Book 导入 → parent 必须 null
   - Chapter 导入 → parent 必须是 Book
   - Page 导入 → parent 必须是 Book 或 Chapter
3. **权限检查** `ensurePermissionsPermitImport()`：
   - 依内容递归检查 BookCreateAll / ChapterCreate(-All) / PageCreate(-All/-Own) / ImageCreateAll / AttachmentCreateAll
   - 任一不满足抛 `ZipImportException` 附本地化错误消息

#### 2.2.2 树重建顺序

严格按 **自上而下、先附件图片后正文** 的顺序创建：

```
importBook()
  ├── BookRepo::create() 建书籍（含封面上传、标签）
  ├── 封面图加入 references
  ├── 按 priority 合并 chapters[] + pages[]
  └── 遍历子节点
        ├─ importChapter()
        │    ├── ChapterRepo::create() 建章节
        │    ├── 按 priority 排序 pages[]
        │    └── 遍历 pages → importPage()
        │         ├── PageRepo::getNewDraftPage() 建草稿页
        │         ├── importAttachment() 逐个存附件
        │         ├── importImage() 逐个存图片
        │         ├── PageRepo::publishDraft() 发布（写入 html/markdown/name/tags）
        │         └── 页面对象加入 references
        │    └── 章节对象加入 references
        └─ importPage()（同上，挂在 Book 下）
  └── Book 对象加入 references
```

**关键细节**：
- Page 先创建为 **草稿**，附件和图片都挂到草稿页，最后 `publishDraft()` 一次性提交正文
- 所有子节点按 `priority` 升序创建，保持导出时的排序
- 每个创建的实体/附件/图片都立即调用 `ZipImportReferences::addXxx()` 建立旧 id → 新对象的映射

#### 2.2.3 文件提取与上传

`ZipImportRunner::zipFileToUploadedFile()`（`ZipImportRunner.php:264-281`）：
```
检查 files/<name> 大小 ≤ app.upload_limit
  ↓ 超出抛 ZipImportException
从 ZIP stream 拷贝到临时文件 tempnam(sys_get_temp_dir(), 'bszipextract')
  ↓
包装为 Symfony UploadedFile 对象传给各 Repo 的 saveNewFromUpload
  ↓
临时文件路径加入 tempFilesToCleanup[]，全部流程结束后统一 unlink
```

**图片 MIME 嗅探**：`importImage()` 先用 `ZipExportReader::sniffFileMime()` 读文件前 2000 字节通过 `WebSafeMimeSniffer` 检测真实 MIME，再从 MIME 推导出扩展名用于保存。

#### 2.2.4 引用替换：两阶段修复

所有实体创建完毕后，调用 `ZipImportReferences::replaceReferences()`（`ZipImportReferences.php:130-164`）做第二遍内容扫描，把导出时写入的 `[[bsexport:type:id]]` 占位符还原为新实例的 URL：

```
遍历所有已创建的 Book → 更新 description_html
遍历所有已创建的 Chapter → 更新 description_html
遍历所有已创建的 Page → 更新 html 或 markdown（按 contentType）
  ↓
ZipReferenceParser::parseReferences() 正则匹配 /\[\[bsexport:([a-z]+):(\d+)]]/
  ↓
handleReference($type, $id) 在 referenceMap 中查找
  ├── Entity → $model->getUrl() （新的永久链接）
  ├── Image  → gallery 类型先加载缩略图，返回 $model->thumbs['display'] 或 $model->url
  └── Attachment → $model->getUrl(false)
  ↓
额外处理：replaceDrawingIdReferences() 替换 drawio-diagram="<oldid>" 属性为新 image id
  ↓
BaseRepo::update() / PageRepo::setContentFromInput() 写回数据库
```

**referenceMap 的构建时机**：每个 `addXxx($newModel, $exportModel)` 被调用时，以 `"<type>:<oldid>"` 为 key 存入 `$this->referenceMap`，因此替换阶段能查到映射。

---

### 2.3 导入失败的回滚策略

两层防御：

**第一层：数据库事务**（`ImportRepo.php:124-133`）
- 所有实体创建在 `DB::beginTransaction()` 内
- 任何异常触发 `DB::rollBack()`，实体表记录全部撤销

**第二层：文件级回滚**（`ZipImportRunner::revertStoredFiles()`，`ZipImportRunner.php:103-116`）
- 事务回滚后显式调用
- 遍历 `references->images()` 调 `ImageService::destroyFileAtPath()` 删除磁盘上的图片
- 遍历 `references->attachments()` 中 `external=false` 的调 `AttachmentService::deleteFileInStorage()`
- 清理临时文件

> 注意：文件回滚独立于事务，因为存储（尤其 S3 等远程）不受 DB 事务控制。

---

### 2.4 Markdown / HTML / Word 导入说明

BookStack 的「反向导入」设计是 **ZIP 结构导向** 的。原始 Markdown、HTML、Word 文档需先转换为 ZIP 归档（含 data.json + files/）才能被导入引擎识别。

系统未提供直接的 `.md` / `.html` / `.docx` 单文件上传导入接口。若需支持，需要在 `ImportRepo::storeFromUpload()` 之前增加一步 **格式转换**：

| 原始格式 | 转换为 ZIP 所需步骤 |
|---------|-------------------|
| Markdown (.md) | 解析 H1 标题切分为 Page 树；提取 `![](...)` 图片引用打包；按 ZipExportPage 结构组装 data.json |
| HTML (.html) | 用 DOM 解析器提取 `<h1>~<h6>` 构建章节层级；提取 `<img src>` 和 `<a href>` 做引用转换 |
| Word (.docx) | PHPWord 解析 docx XML；将段落/样式映射到 BookStack HTML 标签；嵌入图片提取到 files/ |

---

## 第三部分：边角处理与约定清单

### 3.1 导出阶段的防御性处理

| 场景 | 处理方式 | 代码位置 |
|------|---------|---------|
| 草稿页导出 | Book/Chapter 导出时显式过滤 `!$page->draft`；整树 getTree 也传 `$showDrafts=false` | ZipExportBook.php:73, ZipExportChapter.php:47, BookContents.php:100 |
| 图片 base64 转换失败 | `ImageService::imageUrlToBase64()` 返回 null 时保留原始 URL | ExportFormatter.php:217-219 |
| `<details>` 折叠内容 | PDF 导出前强制加 `open="open"` | ExportFormatter.php:169-176 |
| `<iframe>` 嵌入内容 | PDF 导出前替换为文本链接；协议相对 URL 补全 `https:` | ExportFormatter.php:182-199 |
| 特殊字符（€/£） | DOMPDF 渲染前做 HTML 实体转换 | PdfGenerator.php:192-203 |
| 字体文件不可写 | 抛 `PdfExportException` 带明确路径提示 | PdfGenerator.php:71-73, 102-104 |
| PDF 命令超时 | 默认 15 秒后 `ProcessTimedOutException` 捕获并抛友好异常 | PdfGenerator.php:158-161 |
| ZIP 导出中途异常 | 回滚已添加到 ZIP 的 entry + 删除临时文件 + 关闭并删除 ZIP | ZipExportBuilder.php:97-108 |
| 文件名冲突 | 附件/图片用 `Str::random(20)` 循环检测生成唯一名 | ZipExportFiles.php:43-45, 65-67 |

### 3.2 导入阶段的防御性处理

| 场景 | 处理方式 | 代码位置 |
|------|---------|---------|
| data.json 过大 | 读取前检查 size ≤ app.upload_limit MB | ZipExportReader.php:66-69 |
| 单文件过大 | 提取前检查 files/xxx size ≤ app.upload_limit | ZipImportRunner.php:266-270 |
| 图片伪装扩展名 | 提取文件前 2000 字节用 `WebSafeMimeSniffer` 嗅探真实 MIME | ZipImportRunner.php:230-231 |
| 重复 ID 冲突 | `ZipUniqueIdRule` 保证同类型 ID 全局唯一；用 `validatedIds` Set 去重 | ZipValidationHelper.php:46-56 |
| 无效文件引用 | `ZipFileReferenceRule` 检查 files/xxx 存在；图片额外校验 MIME ∈ {png/jpeg/gif/webp} | ZipFileReferenceRule.php |
| 权限逐级校验 | `ensurePermissionsPermitImport()` 递归展开 Book→Chapter→Page，按实体类型、图片、附件分别查权限 | ZipImportRunner.php:286-351 |
| 父级类型不匹配 | Book 不能有 parent；Chapter 必须 Book 父；Page 必须 Book/Chapter 父 | ZipImportRunner.php:71-77 |
| 数据库异常 | 外层事务回滚 + `revertStoredFiles()` 清理已落盘的图片/附件 | ImportRepo.php:127-130 |
| 远程存储文件 | `getZipPath()` 先把远程 ZIP 流式下载到本地临时文件再操作 | ZipImportRunner.php:353-368 |
| Draw.io 图表 ID | `replaceDrawingIdReferences()` 单独处理 `drawio-diagram="<id>"` 属性替换 | ZipImportReferences.php:114-128 |

### 3.3 跨层级一致性约定

- **priority 排序**：导出和导入都按 priority 升序处理，保证书籍内章节/页面顺序稳定
- **可见性过滤**：导出前所有查询均走 `visible` scope，不导出用户不可见的内容
- **标签保留**：Book/Chapter/Page 三级的 tags 都完整保留（name + value）
- **Markdown 双轨**：导出时若 Page 有原始 markdown 则同时存 html + markdown；导入时优先用 markdown 渲染回 html
- **引用两阶段**：导出用 `[[bsexport:*]]` 占位符编码，导入第二阶段统一替换为新 URL — 避免创建过程中引用到尚未创建的实体
- **空字段省略**：`ZipExportModel::jsonSerialize()` 自动过滤 null 值，使 data.json 更精简

---

## 附录：核心类索引

| 功能 | 类文件路径 |
|------|-----------|
| PDF 生成引擎 | `app/Exports/PdfGenerator.php` |
| 导出格式化（HTML/PDF/TXT/MD） | `app/Exports/ExportFormatter.php` |
| ZIP 导出构建器 | `app/Exports/ZipExports/ZipExportBuilder.php` |
| ZIP 文件引用管理 | `app/Exports/ZipExports/ZipExportFiles.php` |
| 导出时引用编码 | `app/Exports/ZipExports/ZipExportReferences.php` |
| 导入执行器（实体重建） | `app/Exports/ZipExports/ZipImportRunner.php` |
| 导入时引用替换 | `app/Exports/ZipExports/ZipImportReferences.php` |
| 链接/引用解析器 | `app/Exports/ZipExports/ZipReferenceParser.php` |
| ZIP 读取器 | `app/Exports/ZipExports/ZipExportReader.php` |
| ZIP 校验器 | `app/Exports/ZipExports/ZipExportValidator.php` |
| 导入仓储（生命周期管理） | `app/Exports/ImportRepo.php` |
| 书籍内容树构建 | `app/Entities/Tools/BookContents.php` |
| 页面内容渲染（含 include 展开） | `app/Entities/Tools/PageContent.php` |
| 导出配置 | `app/Config/exports.php` |
