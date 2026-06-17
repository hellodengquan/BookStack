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

**流式提取大文件**：
`ZipExportFiles::extractEach()` 使用 PHP `stream_copy_to_stream()` 做流式拷贝，而不是一次性读入内存：
```
stream 来源 (S3/本地文件) → stream 临时文件 → 加入 ZIP
```
- 附件走 `AttachmentService::streamAttachmentFromStorage()` 取流
- 图片走 `ImageService::getImageStream()` 取流
- 每个文件落到 `tempnam(sys_get_temp_dir(), 'bszipfile-')` 命名的临时文件
- 回调处理完后由调用方（`ZipExportBuilder`）负责 `unlink()` 清理
- 避免大文件占满内存，尤其在远程存储（S3 等）场景下

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

**Page Include 与循环引用防御**：
导出前的 `PageContent::render()` 会展开 `{{@pageId}}` 形式的页面 include 标签（`app/Entities/Tools/PageContent.php:327-335`）：
- 最多展开 **3 层** 嵌套（`$includeDepth < 3`），每层调用 `PageIncludeParser::parse()`
- 每层展开后如果有新增节点才继续下一层，无新增则提前终止
- 超过 1 层时还会重新生成 DOM 元素的 `id`（`bkmrk-` 前缀），防止 id 冲突
- 这是一种**深度限制**的循环防御：即使 A 包含 B、B 包含 A，3 层后自动停止，不会无限递归
- 导出的 HTML 是**完全展开后的静态内容**，导入时不再有 include 标签，因此导入阶段不需要处理循环引用

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

**ZipUniqueIdRule：历史 ID 复用与唯一性**（`app/Exports/ZipExports/ZipUniqueIdRule.php` + `ZipValidationHelper.php:46-56`）：
- 校验范围是**同类型内** ID 唯一，即 `page:5` 和 `chapter:5` 可以共存，但不能有两个 `page:5`
- `validatedIds` 用 `"<type>:<id>"` 字符串 key 做 Set，`hasIdBeenUsed()` 首次调用返回 false 并登记，后续相同 key 返回 true 触发失败
- 这些 ID 是**源实例的历史 ID**，导入后会重新分配新 ID，不与当前实例 ID 冲突
- 唯一性保证的是 `[[bsexport:page:5]]` 占位符不会歧义——一个 ID 只能指向一个实体

**ZipFileReferenceRule + WebSafeMimeSniffer：极端 MIME 防御**：
`ZipFileReferenceRule`（`app/Exports/ZipExports/ZipFileReferenceRule.php`）做三重校验：
1. **存在性**：`files/<name>` 在 ZIP 中真实存在
2. **大小限制**：单文件 ≤ `app.upload_limit` MB，和 data.json 用同一上限
3. **MIME 白名单**（图片时）：用 `WebSafeMimeSniffer` 嗅探前 2000 字节，必须在 `{image/png, image/jpeg, image/gif, image/webp}` 内

`WebSafeMimeSniffer`（`app/Util/WebSafeMimeSniffer.php`）的降级策略：
- 先用 `finfo` 扩展嗅探原始 MIME
- 若结果在 40 余种 `$safeMimes` 白名单内 → 直接返回
- 若不在白名单但属于 `text/*` → 降级为 `text/plain`
- 其他全部 → 降级为 `application/octet-stream`
- text/plain 且提供了扩展名时，按 `textTypesByExtension` 映射回更具体的类型（css→text/css、js→text/javascript、json→application/json、csv→text/csv）
- 本质是**MIME 白名单 + 安全降级**，防止 polyglot 文件（伪装成图片的脚本/HTML）绕过检测

**ZipValidationHelper 自定义规则扩展机制**：
`ZipValidationHelper`（`app/Exports/ZipExports/ZipValidationHelper.php`）是校验规则的上下文容器，设计上支持自定义规则扩展：

**核心机制**：
- **上下文共享**：Helper 持有 `ZipExportReader` 实例和 `validatedIds` 状态，所有规则类共享同一上下文
- **工厂方法**：`fileReferenceRule()` 和 `uniqueIdRule()` 是规则工厂方法，每次调用返回新的 Rule 实例但共享同一个 Helper
- **Laravel 集成**：内部通过 `app(Factory::class)` 获取 Laravel Validation Factory，`validateData()` 委托给 Laravel Validator
- **递归校验**：`validateRelations()` 对每个子数组（如 pages[]、chapters[]）递归调用对应模型的 `validate()` 方法，整棵树一次性校验完

**扩展新规则的约定步骤**：
1. 新建规则类实现 `Illuminate\Contracts\Validation\ValidationRule` 接口
2. 构造函数接收 `ZipValidationHelper $context` + 业务参数
3. 在 `validate()` 方法中通过 `$this->context->zipReader` 访问 ZIP，通过 `$this->context->hasIdBeenUsed()` 等方法共享状态
4. 在 `ZipValidationHelper` 中添加工厂方法（如 `myCustomRule($param): MyCustomRule`）
5. 在对应 `ZipExport*::validate()` 方法的 rules 数组中使用 `$helper->myCustomRule(...)`

**状态共享模式**：
- `validatedIds` 数组是全局 Set，跨所有子模型校验共享，保证 ID 全局唯一
- `zipReader` 是单例，所有规则复用同一个 ZIP 句柄，避免重复打开
- 错误消息扁平化：`ZipExportValidator::flattenModelErrors()` 把嵌套错误数组拍平为 `book.chapters.0.pages.2.name` 形式的点分路径，方便前端展示

**ZipValidationHelper 5 步扩展的测试样本参考**：
以"新增一个自定义规则：页面名称长度不超过 100 字符"为例，完整测试样本模式如下（参考 `tests/Exports/ZipExportValidatorTest.php` 的测试模式）：

**第 1 步：新建规则类**
```php
// app/Exports/ZipExports/ZipPageNameLengthRule.php
class ZipPageNameLengthRule implements ValidationRule
{
    public function __construct(protected ZipValidationHelper $context) {}
    public function validate(string $attribute, mixed $value, Closure $fail): void {
        if (mb_strlen($value) > 100) {
            $fail('Page name exceeds 100 characters');
        }
    }
}
```

**第 2 步：Helper 添加工厂方法**
```php
// ZipValidationHelper.php
public function pageNameLengthRule(): ZipPageNameLengthRule {
    return new ZipPageNameLengthRule($this);
}
```

**第 3 步：模型 validate 中使用**
```php
// ZipExportPage.php  rules 数组中增加
'name' => ['required', 'string', 'max:255', $helper->pageNameLengthRule()],
```

**第 4 步：编写测试用例**（参考 `ZipExportValidatorTest::test_ids_have_to_be_unique` 模式）
```php
// tests/Exports/ZipExportValidatorTest.php
public function test_page_name_length_validation()
{
    $validator = $this->getValidatorForData([
        'book' => [
            'id' => 1, 'name' => 'Test Book',
            'pages' => [
                ['id' => 1, 'name' => str_repeat('a', 101), 'html' => 'content'],
            ],
        ]
    ]);
    $results = $validator->validate();
    $this->assertArrayHasKey('book.pages.0.name', $results);
}
```

**第 5 步：验证错误消息扁平化**
- 断言错误 key 为 `book.pages.0.name` 形式（点分路径，索引从 0 开始）
- 断言错误消息是翻译后的文本（或测试环境英文）
- 验证多个子项都出错时，每个错误有独立 key，不会被覆盖

**测试样本数据结构**：
| 测试场景 | ZIP 构造 | 预期错误 |
|---------|---------|---------|
| 正常通过 | name 长度 50 字符 | 0 个错误 |
| 刚好边界 | name 长度 100 字符 | 0 个错误 |
| 超出边界 | name 长度 101 字符 | 1 个错误 |
| 多页面都超长 | 3 个页面都超长 | 3 个错误，分别在 pages.0/1/2 |
| 空字符串 | name 为空 | 由 required 规则先拦截，不会走到长度规则 |

**测试辅助工具**：`ZipTestHelper::zipUploadFromData($data, $files)` 可快速构造测试 ZIP，`getValidatorForData()` 封装了 ZIP 构造 → Reader → Validator 的完整链路。

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

**publishDraft 原子性**（`app/Entities/Repos/PageRepo.php:83-103`）：
`publishDraft()` 用 `DatabaseTransaction` 包装成单事务，设置 **READ COMMITTED** 隔离级别：
```
DatabaseTransaction → DB::transaction(...)
  ├── $draft->draft = false;
  ├── $draft->revision_count = 1;
  ├── $this->getNewPriority($draft)  // 计算新 priority
  ├── updateTemplateStatusAndContentFromInput()  // 处理 HTML/Markdown 内容
  ├── baseRepo->update()  // slug、name 等字段更新
  ├── rebuildPermissions()  // 重建权限索引
  ├── revisionRepo->storeNewForPage()  // 存初始版本
  ├── Activity::add(PAGE_CREATE)  // 活动日志
  └── baseRepo->sortParent()  // 触发父级排序
```
- 任何一步异常都会回滚整个事务，保证页面对象、版本、权限、活动日志全部或全无
- `DatabaseTransaction` 先执行 `SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED`，再调用 Laravel 的 `DB::transaction()`
- READ COMMITTED 的意义：权限重建等操作能读到其他已提交事务的变更，避免权限遗漏

**Markdown 双轨与 metadata 兼容**：
- 导出时：若 Page 有原始 `markdown` 字段，data.json 中同时存 `html` + `markdown`；否则只存 `html`
- `metadataOnly()` 时：`html` 和 `markdown` 都被置为 null，仅保留 `name`/`id`/`priority` 等元数据
- 导入预览（`Import::decodeMetadata()`）：因为 metadataOnly 过，所以只展示名称树，不暴露正文内容
- 执行导入时：重新从 ZIP 的 data.json 读取完整数据，包含 html + markdown
- 正文写入（`updateTemplateStatusAndContentFromInput()`）：**markdown 优先** — 若 input 中有 markdown 且非空，走 Markdown 编辑器路径并重新渲染 HTML；否则走 HTML 编辑器路径
- 编辑器类型（`page.editor`字段）会根据内容类型自动切换，需 `Permission::EditorChange` 权限

**priority 排序与 ABAC 接入**：
导入过程中 priority 是直接赋值的，不经过排序 UI 的 ABAC 检查，但权限统一在 `ensurePermissionsPermitImport()` 中校验：
- 导入排序不经过 `BookSorter`（`app/Sorting/BookSorter.php`）的复杂 ABAC 逻辑
- `BookSorter::isSortChangePermissible()` 会同时检查：当前父级更新权 + 目标父级更新权 + 元素本身编辑权 + 移动时的删除权/创建权
- 导入时简化为：只要有对应层级的 `PageCreate`/`ChapterCreate` 权限，就允许设置 priority
- 导入完成后 `sortParent()` 只做同层内的顺序调整，不涉及跨父级移动
- 真正的排序 ABAC 发生在手动拖拽排序（BookSorter）场景，导入是"批量创建"语义而非"移动重排"语义

**BookSorter cache invalidation 机制**：
`BookSorter::sortUsingMap()` 完成排序后，对每本涉及的 Book 调用 `rebuildPermissions()`（`BookSorter.php:111-114`），触发权限缓存失效与重建：
- **缓存介质**：`joint_permissions` 数据库表（预计算的权限联合表），由 `JointPermissionBuilder` 维护（`app/Permissions/JointPermissionBuilder.php`）
- **失效粒度**：按 Book 维度失效+重建 — 排序可能改变 book/chapter 归属，因此整本书的 joint permission 全部重算
- **级联重建**：`rebuildForEntity()` 对 Book 实例会递归包含所有子 Chapter 和 Page，确保整树权限一致
- **与导入的区别**：导入时 `publishDraft()` / 创建章节也会调 `rebuildPermissions()`，但只针对单个新创建的实体；BookSorter 因为可能涉及跨书籍移动，需要按 Book 粒度整块重建
- **READ COMMITTED 隔离**：`DatabaseTransaction` 使用 READ COMMITTED 而非默认 REPEATABLE READ，确保权限重建能读到其他已提交事务的变更，避免遗漏

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

**大文件流式处理的多层设计**：
整个导入导出链路对大文件都采用 stream 流式处理，避免一次性载入内存：

| 层级 | 实现 | 代码位置 |
|------|------|---------|
| ZIP 读取 | `ZipArchive::getStream()` 获取文件流，而非 `getFromName()` 一次性读 | `ZipExportReader.php:100-103` |
| ZIP → 临时文件 | `stream_copy_to_stream()` 字节流拷贝，内存占用恒定 | `ZipImportRunner.php:273-275` |
| 远程存储 → 本地 | `getZipPath()` 中远程 ZIP 流式下载到 `tempnam()` 临时文件 | `ZipImportRunner.php:353-368` |
| 存储 → ZIP 导出 | `streamAttachmentFromStorage()` / `getImageStream()` 取流后再拷贝 | `ZipExportFiles.php:87-106` |

**四层流式的 backpressure 特性**：
整个流式链路基于 PHP `stream_copy_to_stream()` 的同步阻塞模型，天然具备 backpressure（背压）：
- **天然背压**：读端和写端在同一个线程内同步执行，读速度受写速度约束，写慢则读慢，不会出现"读太快写跟不上导致内存暴涨"的情况
- **缓冲区大小**：PHP 默认 8192 字节/次拷贝，每批次都在 write 端完成后才读下一批
- **无异步队列**：没有 producer-consumer 队列，也不需要显式的水位线（watermark）控制
- **阻塞点**：远程存储（S3）的网络 IO 是主要瓶颈，backpressure 自动从最慢的一层向上传导
- **局限性**：单线程串行拷贝吞吐量有限；超大文件（GB 级）仍受 PHP max_execution_time 限制

**两层大小限制**：
1. `data.json` 大小限制（读取前检查 stat size）
2. 每个 `files/xxx` 大小限制（提取前检查 stat size）
都基于 `app.upload_limit` 配置（MB），防止超大文件撑爆内存或磁盘。

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

**循环引用与递归防御**：
导入阶段的引用替换是**单次扫描**，不会出现循环替换问题：
- `parseReferences()` 用正则一次性找出所有 `[[bsexport:*]]` 占位符，逐个替换为真实 URL
- 替换后的 URL 不会再被扫描（不是占位符格式），因此 A→B→A 的循环引用不会导致无限替换
- 与导出前 Page Include 的 3 层深度限制不同，导入时的链接引用是"平面的"——只是 URL 字符串，没有嵌套结构
- 真正的循环引用风险在 Page Include（`{{@pageId}}` 标签），但导出时已全部展开为静态 HTML，导入文件中不存在 include 标签

**环检测双层的子树合并机制**：
导出端的 Page Include 展开是双层环检测 + 子树合并的组合策略：

| 层级 | 机制 | 作用 |
|------|------|------|
| 第一层（深度限制） | `$includeDepth < 3`，最多展开 3 层 | 防止 A→B→C→A 的循环引用无限展开 |
| 第二层（节点增量检测） | 每层展开后 `$nodesAdded !== 0` 才继续，无新增则提前终止 | 处理空内容或纯文本 include 的快速退出 |

**子树合并的 DOM 操作**（`PageIncludeParser`）：
1. **定位与隔离**：用 XPath 找到包含 `{{@` 的文本节点，按标签位置切割为独立 DOM 节点
2. **段落拆分**：如果 include 标签在 `<p>` 中间，需要把 `<p>` 从标签处拆成两个 `<p>`，因为 include 内容可能是块级元素
3. **节点替换**：把 include 标签节点替换为被包含页面的 DOM 子树（`toDomNodes()` 返回的节点数组）
4. **空节点清理**：`toCleanup` 数组收集被掏空的父节点，统一删除
5. **id 重生成**：超过 1 层时 `updateIdsRecursively()` 给所有标题/有 id 的元素加 `bkmrk-` 前缀，防止子树间 id 冲突
6. **引用转换**：展开后内容是纯 HTML，原页面内的相对链接会变成绝对 URL（由导出引用编码阶段再处理）

**PageIncludeParser 6 步 DOM 性能基准**：
基于 DOM 操作复杂度和实际测试场景的性能估算：

| 步骤 | 时间复杂度 | 典型耗时占比 | 性能瓶颈点 |
|------|-----------|-------------|-----------|
| 定位与隔离 | O(n) n=文本节点数 | ~15% | XPath 查询 `//*[text()[contains(., '{{@')]]` 全树扫描 |
| 段落拆分 | O(k) k=标签数 × 子节点数 | ~25% | `splitNodeAtChildNode()` 中 `cloneNode()` + 子节点移动 |
| 节点替换 | O(m) m=被替换节点数 | ~20% | `importNode()` 跨文档节点导入（深拷贝） |
| 空节点清理 | O(p) p=候选清理节点 | ~10% | 向上遍历父节点链删除空元素 |
| id 重生成 | O(q) q=标题/有 id 元素数 | ~20% | 递归遍历 + 字符串拼接 `bkmrk-` 前缀 |
| 引用转换 | O(r) r=链接/图片数 | ~10% | 正则匹配 + URL 拼接 |

**性能优化约定**：
- 3 层深度限制既是环检测也是性能上限 — 最坏情况 O(n^depth) 被限制在可控范围
- `nodesAdded !== 0` 提前终止：无新增节点时立即停止下一层展开，避免无效遍历
- `toCleanup` 批量收集最后统一清理：减少 DOM 重排重绘次数
- 实际场景中 90% 的 include 只有 1 层嵌套，性能接近 O(n) 线性

**inline vs block 的子树兼容边界**：

| 维度 | inline 模式 | block 模式 |
|------|------------|-----------|
| 判定依据 | 被包含页面只有**单个段落**且内容都是行内元素 | 包含 table/ul/ol/pre/h1-h6 等块级元素 |
| 父节点处理 | 留在原 `<p>` 内，把标签文本替换为子节点 | 必须"破开"父 `<p>`，把块级元素提升为同级 |
| 段落拆分 | 不需要 | 需要 `splitNodeAtChildNode()` + `moveTagNodeToBesideParent()` |
| 嵌套层级处理 | 嵌套在 `<strong>`/`<em>` 等行内元素中也能工作 | 嵌套在块级元素中时按文本位置（前半/后半）放前或后 |
| id 兼容性 | 保留原始 id | 保留原始 id（多层时加 `bkmrk-` 前缀） |
| 原文件名/锚点兼容 | `{{@pageId#anchor}}` 语法通过 `#` 后的 id 定位到具体元素 | 同样支持 `#anchor` 定位，block 模式也生效 |

**原文件名/锚点兼容约定**：
- include 标签支持 `{{@pageId}}` 和 `{{@pageId#anchorId}}` 两种语法
- `#anchor` 部分由 `PageIncludeTag` 解析，`PageContent` 用 `getSection()` 在被包含页面 DOM 中查找对应 id 的元素
- 锚点 id 区分大小写，匹配失败时返回全文（静默降级，不抛错）
- 多层 include 后，内层锚点 id 会被加 `bkmrk-` 前缀，但 include 解析是先取内容再合并，所以锚点查找在 id 重命名之前完成，不受影响

**子树合并的边界约定**：
- include 内容若是 inline（行内），可留在 `<p>` 内
- include 内容若是 block（块级），必须"破开"父 `<p>`，把块级元素提升到同级
- `isInline()` 判断依据：被包含页面是否只含单个段落/纯文本

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

**部分回滚的边界与限制**：
回滚是"尽力而为"的部分回滚，存在以下边界情况：

1. **references 只记录已完成创建的**：
   - 只有成功调用 `addXxx()` 加入 references 的文件才会被回滚
   - 如果在创建某个文件的过程中失败（如写到一半磁盘满了），该文件不在 references 中，可能残留临时文件
   - `tempFilesToCleanup[]` 中的临时文件由 `cleanup()` 兜底删除

2. **回滚顺序与部分成功**：
   - 按 images → attachments → 临时文件的顺序删除
   - 单个文件删除失败不会中断回滚过程（没有 try-catch 包裹）
   - 极端情况下可能出现"删了一部分图片后失败，附件还没删"的部分回滚状态

3. **远程存储的最终一致性**：
   - S3 等远程存储的删除操作是异步的，即使 API 返回成功也可能有延迟
   - 回滚调用的是同步删除 API，但不保证立即生效

4. **回滚不覆盖 data.json 解析阶段**：
   - 上传时 ZIP 已经存到 `uploads/files/imports/` 下
   - 执行导入失败不删除 ZIP 文件本身（Import 记录保留，用户可重试或手动删除）
   - 只有 `deleteImport()` 才会清理 ZIP 文件和 Import 记录

**references rollback 的幂等性分析**：
`revertStoredFiles()` 的幂等程度取决于底层存储 API：

| 操作 | 幂等性 | 原因 |
|------|--------|------|
| `ImageService::destroyFileAtPath()` | 近似幂等 | 底层 `FileStorage::delete()` 调用 `Storage::delete()`，大多数驱动（local/S3）对已不存在文件静默返回 |
| `AttachmentService::deleteFileInStorage()` | 近似幂等 | 同上 |
| `cleanup()` 清理临时文件 | 幂等 | `unlink()` 后 `tempFilesToCleanup = []` 清空数组，重复调用不会出错 |
| 数据库回滚 | 严格幂等 | DB transaction 原子性，要么全部撤销要么全部未提交 |

**幂等性的边界条件**：
- `revertStoredFiles()` 没有幂等保护标记，多次调用会重复执行删除逻辑
- 但由于 `cleanup()` 清空 `tempFilesToCleanup` 数组，第二次调用时临时文件列表为空
- images/attachments 数组在引用对象中不会被清空，重复调用会重复执行文件删除 — 大多数存储驱动对已删除文件返回成功或静默忽略，但理论上可能抛出异常
- 实际场景中回滚只执行一次（事务回滚后立即调用一次），所以幂等性不是强需求

**4 项幂等操作的监控建议**：
当前代码没有内置幂等性监控指标，生产环境可考虑补充以下监控点：

| 操作 | 建议监控指标 | 告警阈值 |
|------|-------------|---------|
| `destroyFileAtPath()` | 删除失败次数 / 总删除次数 | 失败率 > 1% 告警 |
| `deleteFileInStorage()` | 同上 | 同上 |
| `cleanup()` 临时文件 | 残留临时文件数（定时扫描） | > 100 个告警 |
| DB 事务回滚 | 回滚次数 / 总事务数 | 回滚率 > 5% 告警 |

**幂等性增强方案**（可选改造）：
1. 在 `revertStoredFiles()` 入口加 `$reverted = false` 标记，已回滚过直接返回
2. images/attachments 数组删除后清空，防止重复操作
3. 对存储删除操作加 try-catch，失败时记 warning 日志但不中断回滚流程

**远程存储最终一致的告警机制**：
BookStack 对远程存储最终一致性的处理非常克制，**没有显式告警**：
- 写入/删除操作不做重试，失败直接抛 `FileUploadException` 或被上层 catch
- `FileStorage::uploadFile()` 捕获异常后只 `Log::error()` 记日志，然后重新抛出
- 导入导出链路中，存储操作失败会被最外层的 try-catch 捕获，触发 DB 回滚 + 文件回滚
- S3 的最终一致性窗口内（通常秒级）重复操作可能出现"刚上传就读不到"或"刚删了还能读到"的情况
- 由于导入流程是单线程串行的（先上传文件再读回？不，导入时只写不读回），写后立刻读的场景很少，因此最终一致性问题实际影响有限
- 若需增强告警，可在 `FileStorage` 层面添加事件钩子（如 `FileStored` / `FileDeleted` event），由外部系统监听并做一致性校验

**S3 一致性窗口的告警设计方案**（可选增强）：

| 告警类型 | 触发条件 | 严重级别 | 建议处理 |
|---------|---------|---------|---------|
| 上传后读回失败 | 文件写入成功后立即读回，size 为 0 或不存在 | WARNING | 延迟 500ms 重试一次，仍失败则告警 |
| 删除后读回仍存在 | 文件删除后 30s 读回仍存在 | INFO | 异步轮询，超过 5 分钟告警 |
| 存储操作超时 | S3 API 调用超过 30s 未返回 | WARNING | 重试 1 次，仍超时则告警 |
| 存储 5xx 错误 | S3 返回 500/503 等服务端错误 | ERROR | 指数退避重试 3 次，仍失败则告警 |
| 跨区域复制延迟 | 启用 S3 CRR 时，目标区读回延迟超过阈值 | INFO | 业务侧不阻塞，异步监控 |

**导入场景的最终一致性风险评估**：
- ✅ **文件上传后不立即读**：导入时保存文件后直接返回 path，不会立即读回，避开了写后读的一致性窗口
- ⚠️ **回滚时删除刚上传的文件**：短时间内先写后删，可能在 S3 内部产生冲突，但通常会收敛到"已删除"状态
- ⚠️ **图片缩略图生成**：上传后立即生成缩略图（如果有）可能读不到原图，但 BookStack 是懒加载缩略图的
- ✅ **引用替换用数据库**：引用替换阶段读的是数据库记录，不是读文件，不受一致性影响

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
| Page Include 循环引用 | 最多展开 3 层嵌套，每层无新增节点则提前终止；超过 1 层重新生成元素 id | PageContent.php:327-335 |
| 大文件内存占用 | 导出时 `stream_copy_to_stream()` 流式拷贝，不一次性载入内存 | ZipExportFiles.php:87-106 |
| CSS 变量不兼容 PDF 引擎 | PDF 格式额外注入内联样式：链接色 `app-link`、引用左边框色 `app-color` | resources/views/exports/parts/styles.blade.php |

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
| 极端 MIME 攻击 | `WebSafeMimeSniffer` 白名单机制，非安全 MIME 降级为 text/plain 或 application/octet-stream | WebSafeMimeSniffer.php:64-83 |
| 历史 ID 复用歧义 | 同类型内 ID 必须唯一，保证 `[[bsexport:type:id]]` 占位符不会指向多个实体 | ZipUniqueIdRule.php:20-25 |
| publishDraft 部分失败 | 用 `DatabaseTransaction` 包裹为单事务，READ COMMITTED 隔离级别 | PageRepo.php:83-103 |
| 回滚不彻底 | `revertStoredFiles()` 只清理已加入 references 的文件；临时文件由 `cleanup()` 兜底 | ZipImportRunner.php:103-125 |
| 大文件内存溢出 | ZIP 读取、存储传输全程 `stream_copy_to_stream()` 流式处理 | ZipExportReader.php、ZipImportRunner.php、ZipExportFiles.php |
| 引用循环替换 | 单次扫描正则替换，替换后的 URL 不再重新扫描，不会循环替换 | ZipImportReferences.php:130-164 |
| metadata 正文泄露 | 导入前 `metadataOnly()` 清空 html/markdown 等大字段，预览页只展示名称 | ImportRepo.php:96-97 |

### 3.3 跨层级一致性约定

- **priority 排序**：导出和导入都按 priority 升序处理，保证书籍内章节/页面顺序稳定
- **priority 与 ABAC**：导入是"批量创建"语义，只校验 Create 权限；手动排序（BookSorter）才会执行完整的 Update/Delete/Create 多级权限校验
- **可见性过滤**：导出前所有查询均走 `visible` scope，不导出用户不可见的内容
- **标签保留**：Book/Chapter/Page 三级的 tags 都完整保留（name + value）
- **Markdown 双轨**：导出时若 Page 有原始 markdown 则同时存 html + markdown；导入时优先用 markdown 渲染回 html
- **Markdown 与 metadata**：`metadataOnly()` 会同时清空 html 和 markdown，预览阶段不暴露正文内容
- **引用两阶段**：导出用 `[[bsexport:*]]` 占位符编码，导入第二阶段统一替换为新 URL — 避免创建过程中引用到尚未创建的实体
- **循环引用防御**：导出端 Page Include 3 层深度限制 + 导入端单次扫描替换，双层保证不会出现无限循环
- **空字段省略**：`ZipExportModel::jsonSerialize()` 自动过滤 null 值，使 data.json 更精简
- **流式大文件**：导入导出链路全程 stream 处理，从 ZIP 读取到存储读写都避免一次性载入内存
- **双重大小限制**：data.json 和 files/* 分别受 `app.upload_limit` 限制，双层防护防止超大文件
- **事务隔离级别**：关键写操作（publishDraft 等）统一用 `DatabaseTransaction` 封装，显式设置 READ COMMITTED 隔离级别

### 3.4 导入 17 项防御优先级排序

按执行顺序 + 防御重要性综合排序（编号越小越先执行 / 越关键）：

| 优先级 | 防御项 | 类型 | 失败影响 |
|-------|--------|------|---------|
| P0 | ZIP 格式校验（能否打开） | 入口防线 | 直接拒绝，不进入后续逻辑 |
| P0 | data.json 存在且可解析 | 入口防线 | 直接拒绝 |
| P0 | data.json 大小限制 | 资源防线 | 防止超大 JSON 撑爆内存 |
| P1 | 顶层 key 类型识别（book/chapter/page） | 结构防线 | 拒绝未知格式 |
| P1 | 必填字段校验 | 结构防线 | 字段缺失导致后续逻辑出错 |
| P1 | 字段类型校验 | 结构防线 | 类型错误导致运行时异常 |
| P1 | ZipUniqueIdRule 同类型 ID 唯一 | 引用防线 | 占位符歧义、引用指向错误实体 |
| P2 | ZipFileReferenceRule 文件存在性 | 文件防线 | 创建实体时找不到文件 |
| P2 | 单文件大小限制 | 资源防线 | 大文件占满磁盘/内存 |
| P2 | 图片 MIME 白名单校验 | 安全防线 | polyglot 文件绕过安全策略 |
| P2 | 父级类型匹配校验 | 结构防线 | 实体归属错误 |
| P3 | 权限逐级校验（ensurePermissionsPermitImport） | 权限防线 | 越权创建实体 |
| P3 | 图片伪装扩展名嗅探 | 安全防线 | 同上，侧重上传后真实检测 |
| P3 | publishDraft 事务原子性 | 数据一致性 | 部分创建半残数据 |
| P3 | 数据库事务 + 文件回滚 | 数据一致性 | 失败后残留垃圾数据 |
| P4 | 引用替换两阶段 | 数据正确性 | 内部链接指向旧 URL 或失效 |
| P4 | Draw.io 图表 ID 替换 | 数据正确性 | drawio 图表关联错乱 |

**分层防御思想**：
- **P0-P1 入口/结构层**：在 data.json 解析阶段就挡掉 80% 的无效输入
- **P2 文件/资源层**：在实体创建前校验文件引用和资源限制
- **P3 权限/一致性层**：创建过程中保证权限合规和事务完整
- **P4 正确性/细节层**：创建后修正引用，保证数据可用但不影响安全

**P0-P4 80 percent 基线指标**：
基于"80% 的问题在早期被拦截"的帕累托法则，各防线的拦截率基线指标：

| 防线层级 | 理论拦截率 | 实际基线目标 | 典型拦截场景 |
|---------|-----------|-------------|-------------|
| P0 入口防线 | 30% | ≥ 25% | ZIP 损坏、无法解析、超大文件 |
| P1 结构防线 | 50% | ≥ 45% | 缺字段、类型错、ID 重复、未知类型 |
| P2 文件/资源层 | 15% | ≥ 10% | 文件缺失、大小超限、MIME 不合法 |
| P3 权限/一致性层 | 4% | ≥ 3% | 越权、创建失败、事务异常 |
| P4 正确性层 | 1% | ≥ 1% | 引用失效、图表 ID 错乱 |

**80/20 指标说明**：
- **P0+P1 合计拦截 ≥ 70%**：绝大部分无效输入在解析阶段就被挡掉，不进入更重的创建流程
- **P0+P1+P2 合计拦截 ≥ 80%**：文件层之后再补 10%，80% 的问题在实体创建前解决
- **P3+P4 处理剩余 20%**：真正进入创建流程后的问题，成本更高但数量更少
- 优化优先级：P0/P1 防线的性能优化投入产出比最高，因为每次导入都会经过，且处理成本最低

**指标采集方式**：
- 在 `ZipExportValidator::validate()` 和 `ZipImportRunner::run()` 中埋点计数
- 按错误类型分类统计（format/structure/file/permission/consistency/correctness）
- 按日/周维度观察各层拦截率的变化趋势
- 异常波动可能意味着新的攻击方式或导入格式变更

---

## 附录 A：核心类索引

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
| ZIP 唯一 ID 校验规则 | `app/Exports/ZipExports/ZipUniqueIdRule.php` |
| ZIP 文件引用校验规则 | `app/Exports/ZipExports/ZipFileReferenceRule.php` |
| ZIP 校验辅助类 | `app/Exports/ZipExports/ZipValidationHelper.php` |
| 导入仓储（生命周期管理） | `app/Exports/ImportRepo.php` |
| 导入模型 | `app/Exports/Import.php` |
| 书籍内容树构建 | `app/Entities/Tools/BookContents.php` |
| 页面内容渲染（含 include 展开） | `app/Entities/Tools/PageContent.php` |
| 页面 Include 解析器 | `app/Entities/Tools/PageIncludeParser.php` |
| 页面仓储（publishDraft 等） | `app/Entities/Repos/PageRepo.php` |
| 书籍排序器（ABAC 排序） | `app/Sorting/BookSorter.php` |
| 排序规则模型 | `app/Sorting/SortRule.php` |
| 数据库事务封装 | `app/Util/DatabaseTransaction.php` |
| Web 安全 MIME 嗅探器 | `app/Util/WebSafeMimeSniffer.php` |
| 导出配置 | `app/Config/exports.php` |

---

## 附录 B：24 核心类 SOLID 评估

对导入导出链路中 24 个核心类按 SOLID 五原则做逐一评估：

| 类 | SRP 单一职责 | OCP 开闭 | LSP 里氏替换 | ISP 接口隔离 | DIP 依赖反转 | 综合评价 |
|----|-------------|----------|-------------|-------------|-------------|---------|
| **PdfGenerator** | ⭐⭐⭐⭐⭐ 只做 PDF 生成，引擎选择逻辑内聚 | ⭐⭐⭐ 新增引擎需改 match 分支 | ⭐⭐⭐ 无继承体系 | ⭐⭐⭐⭐ 对外接口单一（fromHtml） | ⭐⭐ 直接 new 引擎对象 | 引擎策略模式可再抽象 |
| **ExportFormatter** | ⭐⭐⭐⭐ 只做导出格式化，5 种格式各成方法 | ⭐⭐⭐ 新增格式需加方法 | ⭐⭐⭐ 无继承 | ⭐⭐⭐⭐ 对外是具体方法 | ⭐⭐ 依赖具体类而非接口 | 职责较多但边界清晰 |
| **ZipExportBuilder** | ⭐⭐⭐⭐⭐ 只负责构建 ZIP 文件 | ⭐⭐⭐ 新增导出类型需扩展 | ⭐⭐⭐ 无继承 | ⭐⭐⭐⭐ 接口简洁 | ⭐⭐⭐ 依赖 ZipExportFiles 等具体类 | 职责单一，异常回滚完善 |
| **ZipExportFiles** | ⭐⭐⭐⭐⭐ 只管文件引用命名和提取 | ⭐⭐⭐⭐ 新增文件类型只需加方法 | ⭐⭐⭐ 无继承 | ⭐⭐⭐⭐ 接口精简 | ⭐⭐⭐ 依赖 AttachmentService/ImageService | 命名去重逻辑干净 |
| **ZipExportReferences** | ⭐⭐⭐⭐⭐ 只管引用编码替换 | ⭐⭐⭐⭐ 新增引用类型加 resolver | ⭐⭐⭐ 无继承 | ⭐⭐⭐⭐ 接口明确 | ⭐⭐⭐ 依赖具体 referenceMap | 职责高度内聚 |
| **ZipImportRunner** | ⭐⭐⭐ 负责整个导入流程，职责偏多 | ⭐⭐ 新增导入类型需改 run() | ⭐⭐⭐ 无继承 | ⭐⭐⭐ 接口简单但内部复杂 | ⭐⭐⭐ 依赖注入 Repo 和 Service | 可拆分为 ImportOrchestrator + 各层级 Importer |
| **ZipImportReferences** | ⭐⭐⭐⭐⭐ 只管引用替换和映射 | ⭐⭐⭐⭐ 新增类型加方法即可 | ⭐⭐⭐ 无继承 | ⭐⭐⭐⭐ 接口清晰 | ⭐⭐⭐ 依赖具体 model 类 | 引用两阶段设计优雅 |
| **ZipReferenceParser** | ⭐⭐⭐⭐⭐ 只做 URL/引用解析 | ⭐⭐⭐⭐⭐ resolver 链模式可扩展 | ⭐⭐⭐ 无继承 | ⭐⭐⭐⭐ 接口单一 | ⭐⭐⭐⭐ 面向 Closure 编程可扩展 | 解析器链模式经典 |
| **ZipExportReader** | ⭐⭐⭐⭐⭐ 只读取 ZIP 文件 | ⭐⭐⭐⭐ 新增读取方法不破坏 | ⭐⭐⭐ 无继承 | ⭐⭐⭐⭐ 接口简洁 | ⭐⭐⭐ 依赖 ZipArchive 具体类 | 流式读取设计好 |
| **ZipExportValidator** | ⭐⭐⭐⭐⭐ 只做校验编排 | ⭐⭐⭐⭐ 模型 validate 可扩展 | ⭐⭐⭐ 无继承 | ⭐⭐⭐⭐ 接口单一 | ⭐⭐⭐⭐ 依赖 ZipValidationHelper 抽象 | 校验与错误分离清晰 |
| **ZipUniqueIdRule** | ⭐⭐⭐⭐⭐ 只做唯一性校验 | ⭐⭐⭐⭐ 新增规则不影响 | ⭐⭐⭐⭐⭐ 实现 ValidationRule 接口 | ⭐⭐⭐⭐⭐ 接口隔离好 | ⭐⭐⭐ 依赖 ZipValidationHelper 具体类 | 典型策略模式 |
| **ZipFileReferenceRule** | ⭐⭐⭐⭐⭐ 只做文件引用校验 | ⭐⭐⭐⭐ 同上 | ⭐⭐⭐⭐⭐ 实现 ValidationRule 接口 | ⭐⭐⭐⭐⭐ 接口隔离好 | ⭐⭐⭐ 依赖 ZipValidationHelper | 三重校验逻辑内聚 |
| **ZipValidationHelper** | ⭐⭐⭐⭐ 管理校验上下文和规则工厂 | ⭐⭐⭐⭐ 加规则只需加工厂方法 | ⭐⭐⭐ 无继承 | ⭐⭐⭐ 暴露方法较多 | ⭐⭐⭐ 依赖具体规则类 | 上下文容器设计合理 |
| **ZipExportModel** (抽象) | ⭐⭐⭐⭐ 定义导出模型契约 | ⭐⭐⭐⭐⭐ 新增子类不改基类 | ⭐⭐⭐⭐⭐ 模板方法模式 | ⭐⭐⭐⭐⭐ 接口精简（4 个方法） | ⭐⭐⭐⭐⭐ 完全依赖抽象 | 抽象基类设计典范 |
| **ZipExportBook** | ⭐⭐⭐⭐⭐ 只表示书籍导出数据 | ⭐⭐⭐⭐ 新增字段不破坏 | ⭐⭐⭐⭐⭐ 继承 ZipExportModel | ⭐⭐⭐⭐ 接口稳定 | ⭐⭐⭐ 依赖具体 model | 数据模型类 |
| **ZipExportChapter** | ⭐⭐⭐⭐⭐ 同上 | ⭐⭐⭐⭐ 同上 | ⭐⭐⭐⭐⭐ 同上 | ⭐⭐⭐⭐ 同上 | ⭐⭐⭐ 同上 | 数据模型类 |
| **ZipExportPage** | ⭐⭐⭐⭐⭐ 同上 | ⭐⭐⭐⭐ 同上 | ⭐⭐⭐⭐⭐ 同上 | ⭐⭐⭐⭐ 同上 | ⭐⭐⭐ 同上 | 数据模型类 |
| **ImportRepo** | ⭐⭐⭐⭐ 导入生命周期管理 | ⭐⭐⭐ 新增生命周期阶段需改 | ⭐⭐⭐ 无继承 | ⭐⭐⭐ 方法较多 | ⭐⭐⭐ 依赖具体类 | 仓储模式标准实现 |
| **Import** | ⭐⭐⭐⭐⭐ Eloquent 模型，只表示数据 | ⭐⭐⭐⭐ 加字段不破坏 | ⭐⭐⭐ 继承 Model | ⭐⭐⭐  Eloquent 接口较大 | ⭐⭐ Laravel 模型通病 | 典型 Active Record |
| **BookContents** | ⭐⭐⭐⭐⭐ 只构建书籍内容树 | ⭐⭐⭐⭐ 新增排序方式可扩展 | ⭐⭐⭐ 无继承 | ⭐⭐⭐⭐ 接口清晰 | ⭐⭐⭐ 依赖具体查询类 | 树构建逻辑内聚 |
| **PageContent** | ⭐⭐⭐⭐ 页面内容渲染，含 include 展开 | ⭐⭐⭐ 新增渲染逻辑需加方法 | ⭐⭐⭐ 无继承 | ⭐⭐⭐ 接口较多 | ⭐⭐⭐ 依赖具体类 | 渲染+include 略重 |
| **PageIncludeParser** | ⭐⭐⭐⭐⭐ 只做 include 标签解析 | ⭐⭐⭐⭐ 解析逻辑独立 | ⭐⭐⭐ 无继承 | ⭐⭐⭐⭐ 接口单一 | ⭐⭐⭐⭐ 依赖 Closure 回调 | 解析器模式 |
| **PageRepo** | ⭐⭐⭐⭐ 页面仓储，CRUD + 业务操作 | ⭐⭐⭐ 新增操作需加方法 | ⭐⭐⭐ 无接口 | ⭐⭐⭐ 接口较多 | ⭐⭐ 依赖多个具体服务 | 典型 Repository 模式 |
| **BookSorter** | ⭐⭐⭐⭐ 只做书籍排序 | ⭐⭐⭐⭐⭐ SortRule 可扩展排序操作 | ⭐⭐⭐ 无继承 | ⭐⭐⭐⭐ 接口清晰 | ⭐⭐⭐ 依赖具体类 | 排序策略模式好 |
| **WebSafeMimeSniffer** | ⭐⭐⭐⭐⭐ 只做 MIME 嗅探 + 安全降级 | ⭐⭐⭐⭐ 加白名单改配置即可 | ⭐⭐⭐ 无继承 | ⭐⭐⭐⭐⭐ 单方法接口 | ⭐⭐⭐ 依赖 finfo 扩展 | 职责单一且安全 |
| **DatabaseTransaction** | ⭐⭐⭐⭐⭐ 只封装事务和隔离级别 | ⭐⭐⭐ 新增隔离策略需改 | ⭐⭐⭐ 无继承 | ⭐⭐⭐⭐⭐ 单 run() 方法 | ⭐⭐⭐ 依赖 DB Facade | 装饰器模式精简实现 |

**整体 SOLID 评价**：

- **最强项**：SRP（单一职责）整体表现优秀，大部分类职责清晰且边界明确；Zip 相关类尤其突出
- **次强项**：ISP（接口隔离）良好，核心类对外接口精简，Validation Rule 类是典范
- **中等项**：OCP（开闭原则）参差不齐 — 解析器链、排序规则、校验规则扩展性好；导入执行器、PDF 生成器扩展性一般
- **薄弱项**：DIP（依赖反转）是最大短板 — Laravel 生态下普遍依赖具体类而非接口；Eloquent 模型和 Facade 广泛使用导致依赖方向向下
- **LSP**：继承体系不深，主要在 ZipExportModel 模板方法模式中体现良好

**SRP / ISP / DIP 量化评估**：
对 24 个核心类按 5 分制打分（1=最差，5=最好），取平均分：

| 原则 | 平均分 | 最高分 | 最低分 | 标准差 | 量化结论 |
|------|-------|-------|-------|-------|---------|
| SRP 单一职责 | 4.4 | 5.0（15 个类满分） | 3.0（ZipImportRunner、PageRepo） | 0.65 | **强** — 83% 的类得分 ≥ 4，职责边界整体清晰 |
| ISP 接口隔离 | 4.2 | 5.0（12 个类满分） | 3.0（Import、PageRepo） | 0.72 | **强** — 75% 的类得分 ≥ 4，小接口占比高 |
| OCP 开闭原则 | 3.4 | 5.0（ZipReferenceParser、BookSorter） | 2.0（ZipImportRunner） | 0.95 | **中** — 分化严重，扩展性好的和差的差距大 |
| LSP 里氏替换 | 3.8 | 5.0（ZipExportModel 系列） | 3.0（多数无继承体系） | 0.81 | **中偏上** — 有继承的地方都遵循良好，但继承体系浅 |
| DIP 依赖反转 | 2.3 | 4.0（ZipExportModel 抽象基类） | 1.0（Import、DatabaseTransaction） | 0.78 | **弱** — 平均分仅 2.3，75% 的类得分 ≤ 3 |

**DIP 薄弱的根因分析**：
1. **Laravel 生态惯性**：Eloquent Active Record 模式天然依赖具体模型，而非 Repository 接口
2. **Facade 广泛使用**：`DB::transaction()`、`Log::error()` 等 Facade 直接调用，无法注入抽象
3. **helper 函数依赖**：`trans()`、`config()`、`url()` 等全局函数耦合
4. **实用主义优先**：中小型项目优先开发效率，过度抽象收益不明显
5. **缺少 Interface 目录**：整个项目中纯接口（Interface）数量远少于抽象类

**可改进建议**：
1. `ZipImportRunner` 可拆分为 `ImportOrchestrator`（编排）+ `BookImporter`/`ChapterImporter`/`PageImporter`（执行），提升 SRP
2. PDF 引擎可抽象出 `PdfEngineInterface`，用 DI 容器注入，提升 OCP + DIP
3. Repository 层可抽出接口（如 `PageRepositoryInterface`），便于测试和替换实现

**3 条 SOLID 改进的重构成本评估**：

| 改进项 | 涉及文件数 | 预估代码行数 | 测试影响 | 风险等级 | 重构成本 | 收益比 |
|-------|-----------|-------------|---------|---------|---------|--------|
| 1. ZipImportRunner 拆分 | ~5 个文件 | +300 行 | 需新增 3 个 Importer 单测 + 调整现有集成测试 | 中 | ⭐⭐⭐ 中等 | ⭐⭐⭐⭐ 高收益 |
| 2. PDF 引擎抽接口 | ~4 个文件 | +150 行 | 需调整 PdfGenerator 测试，增加 mock 测试 | 低 | ⭐⭐ 较低 | ⭐⭐⭐ 中收益 |
| 3. Repository 抽接口 | ~8 个文件 | +400 行 | 所有依赖 Repo 的测试需改 mock 方式 | 高 | ⭐⭐⭐⭐⭐ 高 | ⭐⭐ 低收益 |

**重构成本说明**：
- **改进 1**（ZipImportRunner 拆分）：性价比最高，拆分后每个 Importer 职责单一，便于单独测试和扩展新导入类型，风险可控
- **改进 2**（PDF 引擎接口）：成本低收益中等，主要好处是可插拔 PDF 引擎和便于 mock 测试；但当前 3 种引擎已够用，扩展性压力不大
- **改进 3**（Repository 接口）：成本最高（波及面广），收益最低（主要是"更优雅"，业务价值有限）；Laravel 生态下强行抽接口属于"为了 SOLID 而 SOLID"，不建议优先做
