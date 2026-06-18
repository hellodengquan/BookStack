# 图片与附件管理：存储驱动、缩略图生成与孤儿资源清理

## 一、整体架构概览

BookStack 的图片与附件管理分为两大独立体系：

- **图片体系**：`ImageService` + `ImageStorage` + `ImageStorageDisk` + `ImageResizer`
- **附件体系**：`AttachmentService` + `FileStorage`

两者共享相似的存储抽象模式，但图片体系多出缩略图生成和孤儿清理能力。

---

## 二、存储驱动

### 2.1 核心类与职责

| 类 | 文件 | 职责 |
|---|---|---|
| `ImageStorage` | `app/Uploads/ImageStorage.php` | 图片存储工厂，根据配置选择磁盘，提供 URL/路径转换 |
| `ImageStorageDisk` | `app/Uploads/ImageStorageDisk.php` | 图片磁盘封装，对 Laravel Filesystem 做路径适配和 CRUD |
| `FileStorage` | `app/Uploads/FileStorage.php` | 附件存储封装，类似 ImageStorageDisk 但服务于附件 |

### 2.2 支持的存储驱动

配置文件：`app/Config/filesystems.php`

```
STORAGE_TYPE / STORAGE_IMAGE_TYPE / STORAGE_ATTACHMENT_TYPE
可选值: local, local_secure, local_secure_restricted, s3
```

磁盘映射关系：

| 配置值 | 实际磁盘 | 根路径 |
|---|---|---|
| `local` (图片) | `local` | `public_path()` → 即 `public/uploads/images/` |
| `local_secure` / `local_secure_restricted` (图片) | `local_secure_images` | `storage_path('uploads/images/')` |
| `local` / `local_secure` / `local_secure_restricted` (附件) | `local_secure_attachments` | `storage_path('uploads/files/')` |
| `s3` | `s3` | S3 桶 |

> **注意**：system 类型图片（如 Logo）即使配置了 local_secure 也会使用 local 磁盘，确保公开可访问。
> 见 `ImageStorage::getDiskName()` at `app/Uploads/ImageStorage.php:67`

### 2.3 多 Disk 路径前缀适配详解

#### 2.3.1 图片存储路径的双重适配

路径适配发生在两层：

**第一层：`ImageStorage::getDiskName()` — 选择实际磁盘**
`app/Uploads/ImageStorage.php:67`
```
配置值 filesystems.images
  ├─ "local" + type="system" → disk = "local"
  ├─ "local"                 → disk = "local"
  ├─ "local_secure" / "local_secure_restricted" + type="system" → disk = "local"
  ├─ "local_secure" / "local_secure_restricted"                 → disk = "local_secure_images"
  └─ "s3"                    → disk = "s3"
```

**第二层：`ImageStorageDisk::adjustPathForDisk()` — 调整路径前缀**
`app/Uploads/ImageStorageDisk.php:34`
```
输入: /uploads/images/gallery/2024-01/photo.jpg

disk = "local_secure_images"
  → 去掉 "uploads/images/" 前缀
  → 结果: gallery/2024-01/photo.jpg
  → 因为 local_secure_images 的 root 已经是 storage_path('uploads/images/')

disk = "local" (即 public_path())
  → 保留 "uploads/images/" 前缀
  → 结果: uploads/images/gallery/2024-01/photo.jpg
  → 最终文件在 public/uploads/images/gallery/2024-01/photo.jpg

disk = "s3"
  → 保留 "uploads/images/" 前缀
  → 结果: uploads/images/gallery/2024-01/photo.jpg
  → S3 桶中的对象 key = uploads/images/gallery/2024-01/photo.jpg
```

#### 2.3.2 附件存储路径适配

`FileStorage::adjustPathForStorageDisk()` at `app/Uploads/FileStorage.php:121`
```
disk = "local_secure_attachments"
  → 去掉 "uploads/files/" 前缀
  → 因为 root 已经是 storage_path('uploads/files/')

其他 disk (如 s3)
  → 保留 "uploads/files/" 前缀
```

> **设计意图**：local_secure 系列磁盘的 Laravel root 配置已经限定到具体子目录，因此路径上不需要再写前缀，防止路径穿越到其他目录。而 local（public）和 S3 的 root 更顶层，必须保留前缀才能定位到正确目录。

### 2.4 URL 生成机制（无签名 URL）

BookStack **不使用 S3 签名 URL（presigned/temporary URL）**，而是以下两种模式：

#### 2.4.1 图片公共 URL 生成

`ImageStorage::getPublicUrl()` at `app/Uploads/ImageStorage.php:124`
→ 调用 `getPublicBaseUrl()` at `app/Uploads/ImageStorage.php:135`

```
优先级：
1. 配置项 filesystems.url (即 STORAGE_URL 环境变量)
   → 如果设置了，直接用它作为 base URL
   
2. 未配置 STORAGE_URL 且 filesystems.images = "s3"
   → 自动猜测 S3 公共 URL
     ├─ bucket 不含点号: https://{bucket}.s3.amazonaws.com
     └─ bucket 含点号:   https://s3-{region}.amazonaws.com/{bucket}
   
3. 其他情况 (默认)
   → url('/') 即 BookStack 应用自身的域名
```

最终 URL 格式：`{baseUrl}/uploads/images/{type}/{Y-m}/{filename}`

**代码挂载点**：
- 图片保存时：`ImageService::saveNew()` at `app/Uploads/ImageService.php:98`
  `'url' => $this->storage->getPublicUrl($fullPath)`
- 缩略图返回时：`ImageResizer::resizeToThumbnailUrl()` 中 5 处调用 getPublicUrl()
- 附件下载链接走应用路由：`Attachment::getUrl()` at `app/Uploads/Attachment.php:77`
  `url('/attachments/' . $this->id)` → 经控制器鉴权后流式返回

#### 2.4.2 为什么不使用签名 URL？

- 代码中完全没有 `temporaryUrl()`、`signedUrl()`、`getTemporaryUrl()` 的调用
- S3 图片默认通过桶的公共权限直接访问（配合 `setVisibility(PUBLIC)`）
- 私有图片（local_secure_restricted）走 BookStack 自己的鉴权路由 `ImageController::showImage()`，经权限校验后 `streamImageFromStorageResponse()` 返回
  见 `routes/web.php` 对应路由 + `app/Uploads/ImageService.php:251` pathAccessibleInLocalSecure()

### 2.5 图片类型

图片按 `type` 字段分类存储在不同子目录：
- `gallery` - 图库图片
- `drawio` - Draw.io 绘图
- `system` - 系统图片（Logo 等）
- `user` - 用户头像
- `cover_book` - 书籍封面
- `cover_bookshelf` - 书架封面

路径格式：`/uploads/images/{type}/{Y-m}/{filename}`

### 2.5 附件存储

路径格式：`uploads/files/{Y-m-M}/{random16char}-{ext}.{ext}`

文件名由 `Str::random(16)` 生成，避免冲突。
见 `FileStorage::uploadFile()` at `app/Uploads/FileStorage.php:49`

---

## 三、缩略图生成

### 3.1 核心类

`ImageResizer` at `app/Uploads/ImageResizer.php`

依赖：
- `ImageStorage` - 获取存储磁盘
- `Intervention/Image` - 图像处理库（GD 驱动）
- Laravel `Cache` - 缩略图路径缓存（1 周）

### 3.2 缩略图类型

| 目录命名模式 | 生成方式 | 用途 |
|---|---|---|
| `thumbs-{w}-{h}/` | `cover()` 裁剪 | 画廊缩略图（如 150x150） |
| `scaled-{w}-{h}/` | `scaleDown()` 等比缩放 | 展示图（如 1680 宽） |

### 3.3 生成流程（懒加载）

`ImageResizer::resizeToThumbnailUrl()` at `app/Uploads/ImageResizer.php:63`

```
1. 特殊情况处理
   ├─ GIF + keepRatio → 直接返回原图 URL（不生成缩略图）
   └─ 动态图片（APNG/动画 AVIF）+ keepRatio → 直接返回原图 URL

2. 计算缩略图路径
   thumbDirName = /{thumbs|scaled}-{w}-{h}/
   thumbFilePath = dirname(imagePath) + thumbDirName + basename(imagePath)

3. 缓存查询
   └─ 命中缓存 → 直接返回 URL

4. 磁盘查询
   └─ 文件已存在 → 写入缓存 → 返回 URL

5. 生成缩略图
   ├─ 从磁盘读取原图数据
   ├─ 调用 resizeImageData() 处理
   │   ├─ Intervention Image 解码
   │   ├─ EXIF 方向校正
   │   ├─ 缩放/裁剪
   │   └─ 编码输出
   ├─ 保存缩略图到磁盘（put with makePublic=true）
   └─ 写入缓存 → 返回 URL
```

> **优化**：keepRatio 模式下，如果缩略图体积比原图还大，直接返回原图数据。
> 见 `ImageResizer::resizeImageData()` at `app/Uploads/ImageResizer.php:149`

### 3.4 批量缩略图加载

`ImageResizer::loadGalleryThumbnailsForMany()` at `app/Uploads/ImageResizer.php:32`

为图库列表批量加载两种尺寸的缩略图：
- gallery: 150x150 裁剪
- display: 1680 宽等比缩放

### 3.5 缩略图失败回退与异常处理路径

BookStack **只使用 GD 驱动，完全没有使用 Imagick 驱动**。
证据：`composer.json` 只依赖 `intervention/image ^3.5`，代码中仅使用 `Intervention\Image\Drivers\Gd\Driver`。
`ImageResizer::interventionFromImageData()` at `app/Uploads/ImageResizer.php:161`
```php
$manager = new ImageManager(new Driver(), autoOrientation: false);
// Driver 即 Intervention\Image\Drivers\Gd\Driver
```

#### 3.5.1 异常路径总览

```
缩略图生成异常分为 4 个层级，每层有不同的回退策略：

层级 1: OutOfMemoryHandler (控制器外层)
  → 内存溢出保护，预留 4MB 内存用于返回友好错误
  → 位置: ImageController / GalleryImageController / FaviconHandler

层级 2: loadGalleryThumbnailsForImage 的 try-catch
  → 单张图片缩略图失败不影响整体流程，静默跳过
  → 位置: ImageResizer.php:46

层级 3: interventionFromImageData 中 GD 扩展缺失
  → 显式检查 extension_loaded('gd')，缺失则抛 ImageUploadException
  → 错误消息: "The PHP "gd" extension is required to resize images, but is missing."

层级 4: resizeImageData 中 Intervention 解码/处理异常
  → 捕获任意 Exception，记录 Log，转抛 ImageUploadException
  → 错误消息: errors.cannot_create_thumbs (多语言: "服务器无法创建缩略图，请检查您是否安装了GD PHP扩展")
```

#### 3.5.2 各层级详细代码路径

**层级 1 — OOM 内存保护（控制器层挂载）**

工具类：`app/Util/OutOfMemoryHandler.php`
原理：预先分配 4MB 内存作为储备，PHP 内存溢出触发时释放储备并执行回调。

挂载点（3 处）：
```
1. GalleryImageController::create()  上传图片时
   → OutOfMemoryHandler(fn() => jsonError(trans('errors.image_upload_memory_limit')))

2. GalleryImageController::list()    列出图库时
   → OutOfMemoryHandler(fn() => response()->view(..., ['warning' => trans('errors.image_gallery_thumbnail_memory_limit')]))

3. ImageController::edit()           编辑图片时
   → OutOfMemoryHandler(fn() => response()->view(..., ['warning' => trans('errors.image_thumbnail_memory_limit')]))

4. ImageController::updateFile()     替换图片文件时
   → OutOfMemoryHandler(fn() => jsonError(trans('errors.image_upload_memory_limit')))

5. ImageController::rebuildThumbnails() 重建缩略图时
   → OutOfMemoryHandler(fn() => jsonError(trans('errors.image_thumbnail_memory_limit')))
```

**层级 2 — 静默容错（缩略图批量加载）**

`ImageResizer::loadGalleryThumbnailsForImage()` at `app/Uploads/ImageResizer.php:42`
```php
try {
    $thumbs['gallery'] = $this->resizeToThumbnailUrl($image, 150, 150, false, $shouldCreate);
    $thumbs['display'] = $this->resizeToThumbnailUrl($image, 1680, null, true, $shouldCreate);
} catch (Exception $exception) {
    // Prevent thumbnail errors from stopping execution
    // 即便是单张图生成失败，也不会中断整个列表的渲染
}
$image->setAttribute('thumbs', $thumbs); // 可能为 null
```
视图层对 `thumbs` 为 null 做容错处理。

**层级 3 — GD 扩展显式检查**

`ImageResizer::interventionFromImageData()` at `app/Uploads/ImageResizer.php:163`
```php
if (!extension_loaded('gd')) {
    throw new ImageUploadException('The PHP "gd" extension is required to resize images, but is missing.');
}
```
这是硬检查，GD 不可用则立即失败，不做任何回退。

**层级 4 — Intervention 图像处理异常**

`ImageResizer::resizeImageData()` at `app/Uploads/ImageResizer.php:125`
```php
try {
    $thumb = $this->interventionFromImageData($imageData, $format);
} catch (Exception $e) {
    Log::error('Failed to resize image with error:' . $e->getMessage());
    throw new ImageUploadException(trans('errors.cannot_create_thumbs'));
}
```
捕获 Intervention 解码失败、内存不足、格式不支持等所有异常，写日志后转抛为用户友好错误。

#### 3.5.3 特殊格式的"软回退"（返回原图）

这些不是异常，而是策略性地跳过缩略图生成，直接返回原图 URL：

| 条件 | 位置 | 说明 |
|---|---|---|
| GIF + keepRatio=true | `ImageResizer.php:71` | GIF 动画缩略图后会丢失动画，索性返回原图 |
| APNG（动态 PNG）+ keepRatio=true | `ImageResizer.php:98` | 检测 PNG 数据中是否含 `acTL` chunk |
| 动画 AVIF + keepRatio=true | `ImageResizer.php:98` | 解析 `stsz` box 判断 frame 数 > 1 |
| keepRatio 且缩略图体积 > 原图 | `ImageResizer.php:149` | 缩放反而更大，无意义 |

---

## 四、孤儿资源清理

### 4.1 触发入口（无内置 Cron / Queue）

**重要结论：BookStack 没有为孤儿清理配置任何计划任务（cron）或队列（queue），清理完全是手动触发。**

证据：
- `app/Console/Kernel.php:17` 的 `schedule()` 方法为空
- 清理命令 `CleanupImagesCommand` 不实现 `ShouldQueue`
- 整个代码库中没有任何地方 dispatch 清理相关的 Job
- 唯一使用 Queue 的是 `DispatchWebhookJob`（Webhook 发送），与图片清理无关

#### 4.1.1 入口一：Artisan CLI

文件：`app/Console/Commands/CleanupImagesCommand.php`

```bash
# Dry run（默认）：只列出将要删除的图片，不执行
php artisan bookstack:cleanup-images
php artisan bookstack:cleanup-images -v  # 详细模式，列出具体路径

# 真正执行删除
php artisan bookstack:cleanup-images --force
php artisan bookstack:cleanup-images -f    # 简写
php artisan bookstack:cleanup-images --force --no-interaction  # 脚本化调用

# 同时删除仅存在于旧版本（revision）中的图片
php artisan bookstack:cleanup-images --force --all
php artisan bookstack:cleanup-images -f -a
```

命令执行流程：
```
CleanupImagesCommand::handle()
  ├─ 解析参数: --all → $checkRevisions=false, --force → $dryRun=false
  ├─ --force 模式: 输出警告 + confirm() 二次确认 (非交互模式跳过)
  ├─ ImageService::deleteUnusedImages($checkRevisions, $dryRun)
  ├─ dryRun: 显示统计 + "Run with -f or --force to perform deletions"
  └─ force: 显示统计 + 输出 "X image(s) deleted"
```

#### 4.1.2 入口二：Web 管理后台

路由：`routes/web.php:231`
```php
Route::delete('/settings/maintenance/cleanup-images', [MaintenanceController::class, 'cleanupImages']);
```

控制器：`app/Settings/MaintenanceController.php:36`

```
MaintenanceController::cleanupImages()
  ├─ 权限检查: Permission::SettingsManage
  ├─ 记录审计日志: ActivityType::MAINTENANCE_ACTION_RUN, 'cleanup-images'
  ├─ 解析表单:
  │   ├─ ignore_revisions=true → $checkRevisions=false
  │   └─ 存在 confirm 参数 → $dryRun=false
  ├─ ImageService::deleteUnusedImages($checkRevisions, $dryRun)
  │
  ├─ 结果为 0: showWarningNotification → "未找到需要清理的图片"
  │
  ├─ dryRun:
  │   └─ session()->flash('cleanup-images-warning', "将删除 X 张图片")
  │      （前端显示确认按钮，用户点击后带 confirm 参数再次提交）
  │
  └─ force:
      └─ showSuccessNotification → "已成功删除 X 张图片"
```

UI 流程：`resources/views/settings/maintenance.blade.php:34`
```
1. 用户进入 设置 → 维护 → 清理图片 区域
2. 可勾选"忽略旧版本中的图片引用"
3. 点击"扫描未使用的图片" → dry-run，显示警告+确认按钮
4. 点击确认（表单带 confirm 字段）→ 真正执行删除
```

#### 4.1.3 如何配置自动清理（需用户自行实现）

由于 BookStack 不提供内置 cron，管理员需要手动配置：

```bash
# crontab -e 示例：每周日凌晨 3 点清理
0 3 * * 0 cd /path/to/bookstack && php artisan bookstack:cleanup-images --force --no-interaction >> /var/log/bookstack-cleanup.log 2>&1
```

### 4.2 核心逻辑

`ImageService::deleteUnusedImages()` at `app/Uploads/ImageService.php:177`

```
清理范围：type ∈ ['gallery', 'drawio']

算法：
1. 分批查询图片（每批 1000 张）
2. 对每张图片：
   a. 取文件名（basename）
   b. 在 entity_page_data.html 中 LIKE 查询是否被引用
   c. （可选）在 page_revisions.html 中 LIKE 查询是否被引用
   d. 若两处都未引用 → 标记为孤儿
3. 返回所有孤儿图片路径（dryRun）或执行删除
```

### 4.3 删除操作

`ImageService::destroy()` at `app/Uploads/ImageService.php:155`

```
1. destroyFileAtPath(type, path)
   └─ ImageStorageDisk::destroyAllMatchingNameFromPath()
      ├─ 列出目录下所有文件
      ├─ 过滤出与原图同名的文件（包括各种尺寸的缩略图）
      ├─ 批量删除文件
      └─ 清理空文件夹
2. Image 模型 delete()
```

> **关键点**：`destroyAllMatchingNameFromPath()` 通过文件名匹配，一次删除原图 + 所有尺寸的缩略图。
> 见 `app/Uploads/ImageStorageDisk.php:100`

### 4.4 命令参数

```
bookstack:cleanup-images
  -a, --all    同时清理仅在旧版本中使用的图片（忽略 revisions）
  -f, --force  实际执行删除（默认 dry-run）
```

---

## 五、附件下载并发限流

### 5.1 限流总览

BookStack 使用 Laravel 内置的 `RateLimiter`（`Illuminate\Support\Facades\RateLimiter`）实现请求限流。

限流配置集中在 `RouteServiceProvider::configureRateLimiting()` at `app/App/Providers/RouteServiceProvider.php:79`

### 5.2 四种限流策略

| 限流名称 | 挂载位置 | 速率限制 | 限流 Key | 说明 |
|---|---|---|---|---|
| `api` | `api` middleware group 全局 | 60 次/分钟 | 登录用户 ID，未登录则 IP | 所有 API 路由统一限流 |
| `public` | 特定匿名路由（注册/密码重置） | 10 次/分钟 | IP | 只用于未登录公开操作 |
| `exports` | 导出控制器构造函数 | 访客 4 次/分钟, 登录 10 次/分钟 | 访客用 IP, 登录用 user.id | 控制 PDF/ZIP/HTML 等导出压力 |
| `ThrottleApiRequests` 中间件 | 覆盖 `api` 配置 | 读取 `api.requests_per_minute` 配置 | 同上 | 可通过配置项动态调整 API 限流 |

> **重要结论：附件下载路由 `/attachments/{id}` 本身没有挂载任何 throttle 中间件。**
> 路由定义：`routes/web.php:160` `Route::get('/attachments/{id}', ...)` — 只有 `web` middleware group，不含限流。
> 图片私有访问路由（`/uploads/images/{path}`）同样不限流。

### 5.3 限流代码挂载点详解

**限流定义（RouteServiceProvider）**
`app/App/Providers/RouteServiceProvider.php:79`
```php
RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});

RateLimiter::for('public', function (Request $request) {
    return Limit::perMinute(10)->by($request->ip());
});

RateLimiter::for('exports', function (Request $request) {
    $user = user();
    $attempts = $user->isGuest() ? 4 : 10;
    $key = $user->isGuest() ? $request->ip() : $user->id;
    return Limit::perMinute($attempts)->by($key);
});
```

**exports 限流挂载（3 个控制器构造函数）**
```php
// BookExportController::__construct  at app/Exports/Controllers/BookExportController.php:20
$this->middleware('throttle:exports');

// ChapterExportController::__construct  at app/Exports/Controllers/ChapterExportController.php:20
$this->middleware('throttle:exports');

// PageExportController::__construct  at app/Exports/Controllers/PageExportController.php:21
$this->middleware('throttle:exports');
```

**public 限流挂载（路由层）**
`routes/web.php`
```
Route::post('/register/confirm/accept', ...)->middleware('throttle:public');  // line 345
Route::post('/register', ...)->middleware('throttle:public');                  // line 346
Route::get('/register/invite/{token}', ...)->middleware('throttle:public');    // line 366
Route::post('/register/invite/{token}', ...)->middleware('throttle:public');   // line 367
Route::post('/password/email', ...)->middleware('throttle:public');            // line 371
Route::post('/password/reset', ...)->middleware('throttle:public');            // line 375
```

**api 限流（中间件组 + 可配置覆盖）**
`app/Http/Kernel.php:41` — api 组挂载 `ThrottleApiRequests`
```php
// ThrottleApiRequests 继承自 Laravel 默认 ThrottleRequests
// 但重写了 resolveMaxAttempts()，从配置读取
// app/Http/Middleware/ThrottleApiRequests.php:12
protected function resolveMaxAttempts($request, $maxAttempts): int
{
    return (int) config('api.requests_per_minute');
}
```
配置文件 `app/Config/api.php:13`：`'requests_per_minute' => env('API_REQUESTS_PER_MINUTE', 180),`
即默认 180 次/分钟，但 `RateLimiter::for('api')` 定义的 60 次/分钟仍然生效？——实际上两者并不冲突：`ThrottleApiRequests` 是 Laravel 原生 throttle 中间件的子类，在 api middleware 组中使用，而 `RateLimiter::for('api')` 是通过 `->middleware('throttle:api')` 显式挂载时使用。经检查路由，api.php 的 attachment 路由未单独加 throttle，只靠 `api` 组的 `ThrottleApiRequests`。

### 5.4 限流测试佐证

`tests/Exports/ZipExportTest.php:479` — ZIP 导出限流测试：
```php
// 前 4 次成功（访客 4 次/分钟）
for ($i = 0; $i < 4; $i++) {
    $this->get($page->getUrl("/export/zip"))->assertOk();
}
// 第 5 次触发 429 Too Many Attempts
$this->get($page->getUrl("/export/zip"))->assertTooManyRequests();

// 登录用户放宽到 10 次/分钟
for ($i = 0; $i < 10; $i++) { ...assertOk() }
$this->get(...)->assertTooManyRequests();
```

### 5.5 附件/图片下载为何不限流？

设计权衡：
1. 附件和图片下载前置了**权限校验**（`PageView` 权限），匿名用户本来就访问不了大部分内容
2. 实际文件读取走流式响应（`streamedDirectly`），不会一次性读入内存，并发压力相对可控
3. 如果需要限流，可自行在路由或控制器中追加 `->middleware('throttle:exports')` 或自定义限流器

---

## 六、多 Disk 切换时的旧资源迁移路径

### 6.1 核心结论：BookStack 没有内置的 storage 迁移工具

**证据链**：
1. Artisan 命令列表中（`app/Console/Commands/` 共 17 条命令）**没有任何 storage/disk/fs 迁移相关命令**
   - 现有关联命令：`bookstack:cleanup-images`、`bookstack:update-url`、`bookstack:regenerate-*`
   - 完全没有 `bookstack:storage-migrate` / `copy-storage` / `move-files` 之类的命令
2. 代码库中全局搜索 `copy.*disk|move.*disk|storage.*migrate|transfer`，没有相关逻辑
3. `app/Console/Kernel.php:17` 的 `schedule()` 方法为空，无迁移调度
4. **唯一跨 disk 读写路径是 ZIP 导出/导入流程**（见 6.2 节）

管理员从 local 切到 S3（或反向），必须手动用 `aws s3 sync` / `rclone` / 自写脚本迁移 `uploads/` 下的文件。

### 6.2 间接迁移路径：ZIP 导出 → ZIP 导入

虽然没有直接的存储迁移工具，但 ZIP 导出/导入机制提供了一个**跨实例跨 disk 的内容迁移通道**，且会自动将资源写入目标实例当前配置的磁盘。

#### 6.2.1 导出侧（从任意磁盘读文件 → 写入本地临时 ZIP）

`ZipExportBuilder::build()` at `app/Exports/ZipExports/ZipExportBuilder.php:67`
```
ZipExportFiles::extractEach(callback)
  ├─ 遍历所有引用的附件
  │   └─ AttachmentService::streamAttachmentFromStorage($attachment)
  │       → 不管原始文件在 local / local_secure / s3，都返回 PHP stream
  │       → 保存到 sys_get_temp_dir() 临时文件
  │
  └─ 遍历所有引用的图片
      └─ ImageService::getImageStream($image)
          → 同样从配置磁盘读取 stream
          → 保存到临时文件
  
  → 所有临时文件被 ZipArchive::addFile() 打包
  → 最终 ZIP 返回给用户下载
```

关键点：`streamAttachmentFromStorage()` 和 `getImageStream()` 内部调用的是 `FileStorage::getReadStream()` 或 `ImageStorageDisk::get()`，它们根据当前配置的磁盘适配路径，从正确位置读取。这意味着即便原始文件在 S3，也能被透明地读入 ZIP。

#### 6.2.2 导入侧（从 ZIP 解包 → 写入目标磁盘）

`ZipImportRunner::run()` at `app/Exports/ZipExports/ZipImportRunner.php:51`

**第一步：获取 ZIP 文件路径（本身就支持跨 disk）**
```php
// ZipImportRunner::getZipPath() at line 353
if (!$this->storage->isRemote()) {
    // 本地磁盘直接返回绝对路径
    return $this->storage->getSystemPath($import->path);
}

// 远程磁盘（S3）：先流式下载到本地临时文件
$tempFilePath = tempnam(sys_get_temp_dir(), 'bszip-import-');
$stream = $this->storage->getReadStream($import->path);
stream_copy_to_stream($stream, fopen($tempFilePath, 'wb'));
return $tempFilePath;
```

**第二步：逐个资源解包 → 存到目标磁盘**
```
ZipImportRunner::importAttachment() / importImage()
  → zipFileToUploadedFile()
      → ZipExportReader::streamFile() 从 ZIP 中读取文件流
      → 拷贝到 sys_get_temp_dir() 临时文件
      → 包装为 UploadedFile
  
  → saveNewUpload() / saveNewFromUpload()
      → FileStorage::uploadFile() / ImageStorageDisk::put()
          → 根据目标实例当前配置（STORAGE_TYPE / STORAGE_IMAGE_TYPE）
          → 写入对应磁盘（local / local_secure / s3）
          → 路径适配自动生效
```

**第三步：失败回滚（事务 + 文件清理）**
`ImportRepo::runImport()` at `app/Exports/ImportRepo.php:117`
```php
DB::beginTransaction();
try {
    $model = $this->importer->run($import, $parentModel);
} catch (ZipImportException $e) {
    DB::rollBack();
    $this->importer->revertStoredFiles();  // ← 删除已写入磁盘的图片和附件
    throw $e;
}
DB::commit();
```

`revertStoredFiles()` 遍历已导入的 Image/Attachment，调用各自的 Service 删除磁盘上的文件。

#### 6.2.3 导入文件存储路径

`ImportRepo::storeFromUpload()` at `app/Exports/ImportRepo.php:73`
```php
$path = $this->storage->uploadFile(
    $file,
    'uploads/files/imports/',   // 上传的 ZIP 文件存放在这个路径
    '',
    'zip'
);
```
导入 ZIP 本身走 `FileStorage`，即使用附件的存储配置（`STORAGE_ATTACHMENT_TYPE`）。

### 6.3 手动迁移参考方案（官方未提供，需自行实现）

由于 BookStack 不提供 storage 迁移命令，切换磁盘时需手动操作：

```bash
# 方案 A: local → s3
# 1. 切换配置
STORAGE_IMAGE_TYPE=s3
STORAGE_ATTACHMENT_TYPE=s3
# 2. 同步图片（从 public/uploads/images 或 storage/uploads/images）
aws s3 sync public/uploads/images/ s3://your-bucket/uploads/images/
# 3. 同步附件（从 storage/uploads/files/）
aws s3 sync storage/uploads/files/ s3://your-bucket/uploads/files/
# 4. 用 php artisan bookstack:update-url 更新数据库中的 URL（如有需要）

# 方案 B: s3 → local
# 反向操作即可，注意路径前缀匹配：
# local_secure_images 的 root 已是 storage/uploads/images/
# 因此 s3 对象 key uploads/images/gallery/... 需要对应到 gallery/...
```

路径前缀注意事项（与 2.3 节呼应）：
- 迁移到 **local_secure_images**：S3 中 `uploads/images/gallery/2024-01/a.jpg` → 本地 `storage/uploads/images/gallery/2024-01/a.jpg`（去掉前缀 `uploads/images/`，因为磁盘 root 已经是那个目录）
- 迁移到 **local（public）**：`public/uploads/images/gallery/2024-01/a.jpg`（保留完整前缀）
- 迁移到 **S3**：保留 `uploads/images/` 和 `uploads/files/` 前缀

### 6.4 URL 切换：`bookstack:update-url` 命令

虽然不能迁移文件，但有个配套命令用于更新数据库中存储的 URL：
`app/Console/Commands/UpdateUrlCommand.php`
```bash
php artisan bookstack:update-url https://old.example.com https://new.example.com
```
用途：当域名或 APP_URL 变更时，批量替换 `entity_page_data.html`、`page_revisions.html` 等字段中引用的旧 URL。与存储驱动无关，但切换 storage 类型后如果图片 URL 前缀变化，可能需要配合使用。

---

## 七、权限校验失败时下载早期中断

### 7.1 中断总览

附件和图片的下载链路都有"**权限前置校验**"设计，确保在任何字节被读取或发送给客户端之前，先完成所有权限验证。这是"早期中断"的核心机制 — 拒绝发生在磁盘 I/O、流式传输启动之前，避免资源浪费。

```
HTTP Request → [Middleware Pipeline] → [Controller Action]
    ↓ (权限校验失败)
  403/401 Response ← (无文件读取、无流打开)
    ↓ (权限校验通过)
  打开文件流 → StreamedResponse → 字节发送
```

### 7.2 附件下载的完整中断路径

**路由**：`routes/web.php:160` `Route::get('/attachments/{id}', [AttachmentController::class, 'get']);`
**Middleware 组**：`web`（不含 auth，权限校验在控制器内）

**代码路径**（`app/Uploads/Controllers/AttachmentController.php:212`）：

```
AttachmentController::get($request, $attachmentId)
  ↓ 步骤 1: 查数据库（无文件操作）
  $attachment = Attachment::query()->findOrFail($attachmentId);
    → 找不到 → 404（ModelNotFoundException）
  
  ↓ 步骤 2: 校验所属页面可见性（无文件操作）
  try {
      $page = $this->pageQueries->findVisibleByIdOrFail($attachment->uploaded_to);
  } catch (NotFoundException $e) {
      throw new NotFoundException(trans('errors.attachment_not_found'));
      → 返回 404，不打开文件
  }
  
  ↓ 步骤 3: 外链附件直接重定向（无文件操作）
  if ($attachment->external) {
      return redirect($attachment->path);
  }
  
  ↓ 步骤 4: (通过校验) 打开流并发送
  $fileName = $attachment->getFileName();
  $attachmentStream = $this->attachmentService->streamAttachmentFromStorage($attachment);
  $attachmentSize = $this->attachmentService->getAttachmentFileSize($attachment);
  → 这里才真正打开文件句柄
  
  return $this->download()->streamedDirectly(...) 或 streamedInline(...)
```

**关键点**：
- `findVisibleByIdOrFail()` 内部已包含 `PageView` 权限校验（查询 `visibleForList()` 作用域）
- 任何一步失败都不会调用 `streamAttachmentFromStorage()`，即不会打开文件流
- 完全不涉及磁盘 I/O，PHP 内存占用极低

### 7.3 图片私有访问的完整中断路径

**路由**：`routes/web.php` 中 `Route::get('/uploads/images/{path}', [ImageController::class, 'showImage']);`
（local_secure / local_secure_restricted 模式下才会走到这里）

**代码路径**（`app/Uploads/Controllers/ImageController.php:32`）：

```
ImageController::showImage($path)
  ↓ 步骤 1: 综合权限校验（无文件操作，可能查 DB）
  if (!$this->imageService->pathAccessibleInLocalSecure($path)) {
      throw new NotFoundException(...);
      → 返回 404，不打开文件
  }
  
  ↓ 步骤 2: (通过校验) 流式返回
  return $this->imageService->streamImageFromStorageResponse('gallery', $path);
```

`pathAccessibleInLocalSecure()` 的多层校验（`app/Uploads/ImageService.php:251`）：

```
pathAccessibleInLocalSecure($imagePath)
  → usingSecureImages() ? (local_secure 配置检查，无 I/O)
  
  → pathAccessible($imagePath)
      ├─ usingSecureRestrictedImages() ?
      │   └─ checkUserHasAccessToRelationOfImageAtPath($imagePath)
      │       ├─ 去掉 thumbs-/scaled- 目录前缀
      │       ├─ Image::query()->where('path', '=', $fullPath)->first()  ← 查 DB
      │       └─ 根据 type 校验可见性
      │           gallery/drawio → pages.visibleForList()->where('id', '=', uploaded_to)
      │           cover_book → books.visibleForList()
      │           cover_bookshelf → shelves.visibleForList()
      │           user/system → 直接允许
      │
      ├─ blockedBySecureImages() ?
      │   └─ secure_images + !app-public + guest → 阻止
      │
      └─ imageFileExists($imagePath, 'gallery')  ← 最后一步才查磁盘
```

**关键顺序**：DB 查询和权限校验在前，`imageFileExists()`（磁盘 `exists()` 检查）在最后。如果权限不过，磁盘碰都不碰。

### 7.4 权限校验的两种异常抛出方式

#### 方式 A：Middleware 层抛出（路由级）

`CheckUserHasPermission` 中间件（`app/Http/Middleware/CheckUserHasPermission.php:16`）：
```php
if (!user()->can($permission)) {
    return $this->errorResponse($request);
    // JSON → 403 {'error': '...'}
    // HTML → redirect('/') + session flash error
}
```
挂载在路由上如：`Route::get('/books', ...)->middleware('can:books-view-all')`

#### 方式 B：Controller 内抛出（业务级，用于权限依赖动态数据）

`Controller::checkPermission()` / `checkOwnablePermission()`（`app/Http/Controller.php:63`）：
```php
protected function checkPermission(string|Permission $permission): void
{
    if (!user()->can($permission)) {
        $this->showPermissionError();
    }
}

protected function showPermissionError(string $redirectLocation = '/'): never
{
    $message = request()->wantsJson() ? trans('errors.permissionJson') : trans('errors.permission');
    throw new NotifyException($message, $redirectLocation, 403);
}
```
`NotifyException` 是 BookStack 自定义异常，由异常处理器渲染为用户友好的通知页面。

### 7.5 流式响应的中断时机（Range 请求）

如果权限通过，进入流式响应阶段，`DownloadResponseFactory::streamedDirectly()`（`app/Http/DownloadResponseFactory.php:27`）会：
1. 包装 `RangeSupportedStream`（支持断点续传）
2. 解析 `Range` 请求头，返回 206 Partial Content
3. `response()->stream()` 包装闭包，在闭包内才真正读取流并输出

**即使到了这一步，如果客户端断开连接，PHP 会在 `stream_copy_to_stream()` 调用时检测到并终止，底层 `fclose()` 正常执行。**

`RangeSupportedStream::outputAndClose()` at `app/Http/RangeSupportedStream.php:46`：
```php
public function outputAndClose(): void
{
    $outStream = fopen('php://output', 'w');
    stream_copy_to_stream($this->stream, $outStream, $bytesToWrite);
    // ← 如果客户端断开，这里会中断并进入 PHP 请求清理
    fclose($this->stream);
    fclose($outStream);
}
```

**临时文件自动删除**：`streamedFileDirectly()` 中 `$deleteAfter=true` 时，注册了双重删除钩子：
```php
app()->terminating($callback);       // Laravel 正常结束
register_shutdown_function($callback); // PHP 异常/中断结束
```
确保无论请求正常结束、异常抛出、用户断开，临时 ZIP 文件都会被删除。

### 7.6 权限失败 vs 流式中断的对比

| 场景 | 阶段 | 磁盘 I/O | 状态码 | 资源清理 |
|---|---|---|---|---|---|
| 无权限访问附件 | 控制器入口（findVisibleByIdOrFail 前） | ❌ 无 | 404 | 无需清理 |
| 页面被删除/权限被撤 | 控制器入口（findVisibleByIdOrFail） | ❌ 无 | 404 | 无需清理 |
| 图片 local_secure_restricted 无页面权限 | pathAccessible 校验 | ❌ 无（可能有 DB 查询） | 404 | 无需清理 |
| 非 guest 但 secure_images + 非公开应用 | blockedBySecureImages 校验 | ❌ 无 | 404 | 无需清理 |
| 图片/附件不存在 | imageFileExists / getReadStream 中 | ✅ 有（exists 或 fopen 失败） | 500 / 404 | 无需清理 |
| 流式传输中客户端断开 | outputAndClose 的 stream_copy_to_stream | ✅ 有（已部分传输） | N/A | PHP 清理 + shutdown 函数删临时文件 |

---

## 八、迁移中途断电的恢复机制

### 8.1 核心结论：无断点续传，仅靠幂等性兜底

BookStack 的 ZIP 导入流程**没有断点续传（resume）** 能力。如果中途断电/进程被杀/网络断开，只能重新运行导入。但是系统通过以下设计提供了"**幂等安全 + 失败清理 + 可手动重试**"的恢复能力。

### 8.2 导入流程的三个阶段与事务边界

```
阶段 1: 上传并验证 ZIP (ImportRepo::storeFromUpload)
  → 解压 ZIP 读取 metadata
  → 验证完整性 (ZipExportValidator)
  → 保存 ZIP 到磁盘（FileStorage::uploadFile）
  → 写入 imports 表一条记录（pending 状态）
  → 事务外，提交后持久化
  ← 断电后果：imports 表无记录，文件可能残留（可手动删 / 自动被下次覆盖）

阶段 2: 确认导入 (用户点击"开始导入")
  ← 此时断电无影响，imports 表有记录，重新进入 /import/{id} 可继续

阶段 3: 执行导入 (ImportRepo::runImport)
  → DB::beginTransaction()
  → ZipImportRunner::run()
      ├─ 解压到临时文件
      ├─ 逐资源写入目标磁盘（图片 → ImageService，附件 → AttachmentService）
      │   每成功一个就记录到 ZipImportReferences
      ├─ replaceReferences() 更新 HTML 中的资源引用
      └─ 写入 pages/books/chapters 等表
  → DB::commit()
  → deleteImport() 删除导入 ZIP 文件和 imports 表记录
  → 成功重定向到新页面
  
  ← 断电后果：
     1. DB 事务回滚（DB 无残留）
     2. 但磁盘上已写入的图片/附件文件已持久化（不会回滚）
     3. 需手动触发清理（见 8.3）
```

> 关键：**磁盘写入不在 DB 事务内**。DB 事务只能回滚数据库记录，不能回滚磁盘文件。这是 Laravel/Life Cycle 的通用限制。

### 8.3 失败时的清理机制

#### 8.3.1 异常回滚（代码内捕获）

`ImportRepo::runImport()` at `app/Exports/ImportRepo.php:117`
```php
DB::beginTransaction();
try {
    $model = $this->importer->run($import, $parentModel);
} catch (ZipImportException $e) {
    DB::rollBack();
    $this->importer->revertStoredFiles();  // ← 手动回滚磁盘文件
    throw $e;
}
DB::commit();
```

`ZipImportRunner::revertStoredFiles()` at `app/Exports/ZipExports/ZipImportRunner.php:103`
```php
public function revertStoredFiles(): void
{
    foreach ($this->references->images() as $image) {
        $this->imageService->destroyFileAtPath($image->type, $image->path);
    }
    foreach ($this->references->attachments() as $attachment) {
        if (!$attachment->external) {
            $this->attachmentService->deleteFileInStorage($attachment);
        }
    }
    $this->cleanup(); // 删除所有临时文件
}
```

**关键点**：
- `ZipImportReferences` 在导入过程中**实时记录**每一个成功写入的 Image/Attachment 模型
- 回滚时遍历这些记录，调用各自的 Service 删除磁盘文件
- 缩略图会被 `destroyFileAtPath()` 自动连带删除（`destroyAllMatchingNameFromPath` 文件名匹配）

#### 8.3.2 断电/进程被杀时的残留清理（代码外）

如果 PHP 进程在 `catch` 块执行前被中断（断电、`kill -9`、服务器重启），上述清理逻辑不会运行。此时：

**数据库层面**：
- 事务未 commit，导入的 page/book/chapter/image/attachment 记录全部回滚
- imports 表中有导入记录（因为阶段 1 已提交），状态仍为"pending"

**磁盘层面**：
- 已写入的图片和附件文件作为**孤儿文件**遗留在磁盘上
- 导入 ZIP 文件本身留在 `uploads/files/imports/` 目录

**恢复步骤**：
1. 重新访问 `/import` 页面，看到 pending 的导入记录
2. 可以选择**重新运行导入**（会再写一遍文件，但 DB 记录会覆盖）
3. 或者**删除导入**（`ImportController::delete()` → `ImportRepo::deleteImport()` → 仅删除 ZIP 文件和 imports 记录，不清理孤儿文件）
4. 孤儿文件需要运行孤儿清理：
   ```bash
   php artisan bookstack:cleanup-images --force
   ```
   这会扫描 gallery/drawio 类型图片，未在 `entity_page_data.html` 中引用的会被删除（包括缩略图）
5. 附件孤儿文件**无法自动清理** — 因为附件没有孤儿清理机制，需要手动对比 `attachments` 表和磁盘文件列表后手动删除

### 8.4 临时文件的自动清理

导入过程中产生的各种临时文件通过以下机制清理：

| 临时文件 | 位置 | 清理时机 | 清理方式 |
|---|---|---|---|
| ZIP 解压的单个资源临时文件 | `sys_get_temp_dir()/bszipextract-*` | 导入完成 / 异常回滚 | `ZipImportRunner::cleanup()` 遍历 `$tempFilesToCleanup` unlink |
| S3 下载的 ZIP 临时副本 | `sys_get_temp_dir()/bszip-import-*` | 同上 | 同上 |
| 导出用临时文件 | `sys_get_temp_dir()/bszipfile-*` / `bszipimage-*` | 导出完成后立即 | `ZipExportFiles::extractEach()` 回调中 unlink |
| 最终导出 ZIP 文件 | `sys_get_temp_dir()/bszip-*` | 流式下载完成 / 中断 | `streamedFileDirectly($deleteAfter=true)` 注册 `app()->terminating()` + `register_shutdown_function()` 双保险 |
| 用户上传的导入 ZIP | `uploads/files/imports/` | 导入成功 / 用户手动删除 | `ImportRepo::deleteImport()` → `FileStorage::delete()` |

### 8.5 导入幂等性分析

**不支持断点续传，但支持安全重试**：
- 导入失败后重试，磁盘上可能存在上一次遗留下的同名文件
- `FileStorage::uploadFile()` 会生成 `Str::random(16)` 文件名，每次都不同，不会冲突
- 图片上传如果指定了原始文件名，会在文件名后追加随机后缀避免冲突
- 重试产生的额外文件会成为孤儿，可通过 `bookstack:cleanup-images` 清理
- 数据库层面因为每次事务独立，重试时会分配新的 page/image/attachment ID，不会有主键冲突

### 8.6 导出过程中的断电恢复

导出比导入简单得多：
- 导出时不写任何数据库记录
- 所有中间文件都在 `sys_get_temp_dir()`，PHP 进程被杀后由操作系统的临时目录清理机制自动清理（通常 reboot 或定期 `tmpwatch`）
- `streamedFileDirectly($deleteAfter=true)` 的双保险删除钩子在进程被杀时可能来不及运行，但文件在 tmp 目录，无需担心
- 用户看到下载中断，重新点击导出即可，无任何状态需要恢复

---

## 九、附件版本控制

### 9.1 核心结论：附件没有版本控制

**与页面（Page）不同，附件（Attachment）完全没有版本控制机制。**

证据链：
1. `attachments` 表没有 `revision_id`、`version` 等版本相关字段
2. 没有 `attachment_revisions` 表
3. `page_revisions` 表只存 `html` 和 `text`（页面内容），不存附件信息
4. 附件更新时直接覆盖旧文件，不保留历史

### 9.2 attachments 表完整字段

`database/migrations/2016_10_09_142037_create_attachments_table.php:16`

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | increments | 主键 |
| `name` | string | 显示名称（文件名） |
| `path` | string | 存储路径（随机16字符文件名）或外链URL |
| `extension` | string(20) | 文件扩展名（外链时为空） |
| `uploaded_to` | integer | 所属页面 ID（有索引） |
| `external` | boolean | 是否为外链附件 |
| `order` | integer | 排序序号 |
| `created_by` | integer | 创建者用户 ID |
| `updated_by` | integer | 更新者用户 ID |
| `created_at` | timestamp | 创建时间 |
| `updated_at` | timestamp | 更新时间 |

**对比页面版本**：`page_revisions` 表存 `page_id`、`name`、`html`、`text`、`created_by`，每页可保留 100 个历史版本（`revision_limit` 配置，默认 100）。附件没有类似机制。

### 9.3 附件更新是覆盖式的

`AttachmentService::saveUpdatedUpload()` at `app/Uploads/AttachmentService.php:65`

```php
public function saveUpdatedUpload(UploadedFile $uploadedFile, Attachment $attachment): Attachment
{
    if (!$attachment->external) {
        $this->deleteFileInStorage($attachment);  // ← 先删旧文件
    }

    $attachmentName = $uploadedFile->getClientOriginalName();
    $attachmentPath = $this->putFileInStorage($uploadedFile);  // ← 存新文件（新随机名）

    $attachment->name = $attachmentName;
    $attachment->path = $attachmentPath;  // ← path 也变了
    $attachment->external = false;
    $attachment->extension = $uploadedFile->getClientOriginalExtension();
    $attachment->save();  // ← 直接 update，不创建新记录

    return $attachment;
}
```

**关键点**：
- 旧文件立即被删除（不可恢复）
- 新文件生成全新的随机 16 字符文件名（不复用旧 path）
- 数据库记录原地更新，ID 不变
- 没有任何历史快照

### 9.4 与页面修订的关系

页面保存时创建的 `page_revisions` 只包含 HTML 和纯文本内容，**不包含附件列表的快照**。这意味着：
- 回滚页面版本不会回滚附件
- 无法通过页面修订历史查看某版本时附件是什么状态
- 附件删除了就是删了，跟页面版本没关系

**间接关联**：页面 HTML 中可能包含指向附件的链接（`/attachments/{id}`），回滚页面 HTML 时这些链接 ID 仍然有效，只要附件没被删除。

### 9.5 外链附件的"版本"行为

`AttachmentService::updateFile()` at `app/Uploads/AttachmentService.php:117`
- 修改外链 URL：直接更新 `path` 字段，旧 URL 不保留
- 上传文件替换外链：删除外链状态，`external=false`，写入磁盘文件
- 将文件改为外链：删除磁盘文件，`external=true`，`path` 存 URL

全部都是**直接覆盖**，无历史记录。

---

## 十、上传配额限制

### 10.1 核心配置

**单一全局配置，图片和附件共用**。

`app/Config/app.php:38`
```php
'upload_limit' => env('FILE_UPLOAD_SIZE_LIMIT', 50),  // 单位：MB
```

环境变量：`FILE_UPLOAD_SIZE_LIMIT`
默认值：50 MB
作用范围：图片上传 + 附件上传 + ZIP 导入文件大小检查 + 嵌入 base64 图片

> 注意：没有针对不同类型（图片/附件/头像）的单独配额，也没有按用户/角色的配额。

### 10.2 单位换算的差异

代码中有**两种不同的换算方式**，需要注意：

| 位置 | 乘法因子 | 换算基准 | 50MB 结果 |
|---|---|---|---|
| Laravel `max:` 验证规则 | × 1000 | KB（千字节，1000 字节） | 50,000 KB |
| ZIP 导入 size 检查 | × 1000000 | 字节（1000×1000） | 50,000,000 字节 |
| 嵌入 base64 图片检查 | × 1000000 | 字节 | 50,000,000 字节 |

**差异原因**：Laravel 的 `max` 验证规则的"max:50000"单位是 KB（kilobytes），但 Laravel 用的是 1000 字节 = 1KB（十进制），不是 1024。而代码中其他地方直接乘 1000000 也是十进制兆字节。两者实际上是一致的（都是十进制 MB），但实现路径不同。

验证：`config('app.upload_limit') * 1000` KB = `50 * 1000 * 1000` 字节 = `50 * 1000000` 字节 ✓

### 10.3 六个挂载点

#### 挂载点 1：Web 附件上传（新建）

`app/Uploads/Controllers/AttachmentController.php:37`
```php
$this->validate($request, [
    'file' => array_merge(['required'], $this->attachmentService->getFileValidationRules()),
]);
```
调用 `AttachmentService::getFileValidationRules()` → `['file', 'max:' . (config('app.upload_limit') * 1000)]`

#### 挂载点 2：Web 附件上传（更新）

`app/Uploads/Controllers/AttachmentController.php:66`
```php
$this->validate($request, [
    'file' => array_merge(['required'], $this->attachmentService->getFileValidationRules()),
]);
```
与新建相同的验证规则。

#### 挂载点 3：API 附件上传

`app/Uploads/Controllers/AttachmentApiController.php:51` + `rules()` 方法
```php
// create 规则
'file' => array_merge(['required_without:link'], $this->attachmentService->getFileValidationRules()),

// update 规则（可选文件）
'file' => $this->attachmentService->getFileValidationRules(),
```
API 创建时 file 和 link 二选一；更新时 file 可选。

#### 挂载点 4：图片上传（Web + API 通用）

`app/Http/Controller.php:163` — 基类方法
```php
protected function getImageValidationRules(): array
{
    return ['image_extension', 'mimes:jpeg,png,gif,webp,avif', 'max:' . (config('app.upload_limit') * 1000)];
}
```
被以下位置调用：
- `ImageGalleryApiController` 的 `create` 和 `readDataForUrl` 规则
- `GalleryImageController::create()` at line 61
- `DrawioImageController::create()` at line 59
- `ImageController::updateFile()` at line 48
- `FaviconHandler`（间接，通过 ImageService）

#### 挂载点 5：ZIP 导入文件大小检查

**两处检查**：

a) data.json 大小检查（`ZipExportReader::readData()` at line 66）
```php
$maxSize = max(intval(config()->get('app.upload_limit')), 1) * 1000000;
if ($info['size'] > $maxSize) {
    throw new ZipExportException(trans('errors.import_zip_data_too_large'));
}
```
防止 data.json 过大导致内存溢出。

b) 单个附件/图片文件检查（`ZipFileReferenceRule` at line 25）
```php
if (!$this->context->zipReader->fileWithinSizeLimit($value)) {
    $fail('validation.zip_file_size')->translate([
        'attribute' => $value,
        'size' => config('app.upload_limit'),  // 显示 MB 数
    ]);
}
```
`fileWithinSizeLimit()` 内部也是用 `upload_limit * 1000000` 字节比较。

> 注意：ZIP 文件本身的大小不受 `upload_limit` 限制，只限制 ZIP 内的单个文件和 data.json。

#### 挂载点 6：页面 HTML 中嵌入的 base64 图片

`PageContent::extractTagsAndSaveImages()` at `app/Entities/Tools/PageContent.php:160`
```php
// Validate that the content is not over our upload limit
$uploadLimitBytes = (config('app.upload_limit') * 1000000);
if (strlen($imageInfo['data']) > $uploadLimitBytes) {
    return '';  // 超过大小直接丢弃，不保存
}
```
用户在编辑器里粘贴 base64 图片时的检查。超过限制的图片不会被转为正式图片上传，直接忽略。

### 10.4 PHP 层面的额外限制

除了 BookStack 自身的 `upload_limit`，还受到 PHP.ini 配置限制：
- `upload_max_filesize` — 单个上传文件上限
- `post_max_size` — POST 请求体总大小上限
- `memory_limit` — 内存上限（图像处理时会用到）

这些是 PHP 层面的硬限制，优先级高于 BookStack 的 `upload_limit`。
BookStack 的 `upload_limit` 不能超过 PHP 的 `upload_max_filesize`，否则验证通过但 PHP 层面会先报错。

### 10.5 错误消息

| 场景 | 错误消息 key | 位置 |
|---|---|---|
| 附件/图片上传超大 | `validation.max`（Laravel 默认） | 由 `max:` 验证规则抛出 |
| ZIP 导入 data.json 超大 | `errors.import_zip_data_too_large` | `ZipExportReader::readData()` |
| ZIP 内单个文件超大 | `validation.zip_file_size` | `ZipFileReferenceRule` |
| 嵌入 base64 图片超大 | 静默返回空字符串 | `PageContent` |

### 10.6 配额限制与多 disk 的关系

`upload_limit` 与存储驱动无关：
- local / local_secure / local_secure_restricted / s3 都用同一个限制
- 校验发生在控制器层（验证规则），写入磁盘之前
- 切换磁盘不会改变配额限制

---

## 十一、三者串联关系

### 11.1 图片上传 → 存储 → 缩略图生成 链路

```
GalleryImageController::create()
  → ImageRepo::saveNew()
      → ImageService::saveNewFromUpload()
      │   → ImageStorage::getDisk(type)
      │   → ImageStorageDisk::put(path, data, makePublic=true)
      │   └─ 写入数据库 Image 模型
      → ImageResizer::loadGalleryThumbnailsForImage($image, shouldCreate=true)
          → resizeToThumbnailUrl(150, 150, false, true)   // gallery thumb
          └─ resizeToThumbnailUrl(1680, null, true, true) // display scaled
```

**关键交互点**：
- `ImageRepo` 是协调者，串联 `ImageService`（存储）和 `ImageResizer`（缩略图）
- 上传时 `shouldCreate=true` 强制生成缩略图
- 缩略图与原图存在同一目录下的 `thumbs-/scaled-` 子目录中

### 11.2 图片访问 → 缩略图懒加载 链路

```
ImageController::edit() / GalleryImageController::list()
  → ImageResizer::loadGalleryThumbnailsForMany() / loadGalleryThumbnailsForImage()
      → resizeToThumbnailUrl(..., shouldCreate=false)
          → Cache::get() → 命中则返回
          → disk->exists() → 存在则缓存并返回
          └─ 都没有 → 生成新缩略图
```

### 11.3 孤儿清理 → 存储 → 缩略图删除 链路

```
CleanupImagesCommand::handle()
  → ImageService::deleteUnusedImages(checkRevisions, dryRun)
      → 遍历 gallery/drawio 图片
      → 检查 page_html / page_revisions 中是否引用
      → 未引用 → ImageService::destroy(image)
          → ImageService::destroyFileAtPath(type, path)
          │   → ImageStorage::getDisk(type)
          │   → ImageStorageDisk::destroyAllMatchingNameFromPath(path)
          │       ├─ 查找目录中同名文件（含所有缩略图）
          │       ├─ 批量删除文件
          │       └─ 清理空目录
          └─ Image::delete()
```

### 11.4 数据模型关系

```
Image 模型 (images 表)
├─ id, name, path, url, type, uploaded_to
├─ created_by, updated_by, created_at, updated_at
└─ 通过 uploaded_to 关联到 Page/Book/Shelf

Attachment 模型 (attachments 表)
├─ id, name, path, extension, external
├─ uploaded_to, order, created_by, updated_by
└─ 通过 uploaded_to 关联到 Page
```

---

## 十二、关键设计模式

### 12.1 存储抽象层

`ImageStorage` 作为工厂 + 门面，`ImageStorageDisk` 作为具体磁盘封装。
好处：
- 统一路径适配逻辑
- 屏蔽不同磁盘（local/s3）的差异
- 提供图片特定的操作（如批量删除同名文件）

### 12.2 缩略图懒生成 + 缓存

- 不提前生成，按需创建
- 内存缓存（1周）避免每次都查磁盘
- 多级查找：缓存 → 磁盘 → 生成

### 12.3 孤儿清理策略

- 基于文件名模糊匹配（LIKE），简单但可能误判
- 只清理 gallery 和 drawio 类型（用户上传的内容图片）
- 支持 dry-run，安全第一
- 清理时同时删除所有缩略图（通过文件名匹配）

---

## 十三、附件 vs 图片对比

| 特性 | 图片 | 附件 |
|---|---|---|
| 存储类 | `ImageStorage` + `ImageStorageDisk` | `FileStorage` |
| 服务类 | `ImageService` | `AttachmentService` |
| 缩略图 | 有（`ImageResizer`，仅 GD 驱动） | 无 |
| 孤儿清理 | 有（`deleteUnusedImages`，手动触发） | 无 |
| 路径命名 | 语义化文件名（可能加随机前缀） | 纯随机 16 字符 |
| 安全模式 | local_secure / local_secure_restricted | 统一 local_secure_attachments |
| 数据库表 | images | attachments |
| URL 模式 | 公共 URL（S3/应用域名）或鉴权路由 | 始终走鉴权路由 `/attachments/{id}` |
| 图像处理驱动 | Intervention Image (GD only) | 不处理 |

---

## 十四、补充发现

### 14.1 关于签名 URL（Presigned URL）

BookStack **完全不使用 S3 签名 URL**。策略如下：
- S3 图片：`put()` 时设置 `Visibility::PUBLIC`，桶公开读权限，直接通过 S3 公共 URL 访问
- 本地私有图片：走应用路由 `/uploads/images/{path}`，经 `ImageService::pathAccessibleInLocalSecure()` 校验权限后 `streamImageFromStorageResponse()` 流式返回
- 附件：始终走 `/attachments/{id}` 路由，经 `AttachmentController::get()` 校验后下载

### 14.2 关于 Queue 队列

整个上传/缩略图/清理流程**全部同步执行**，不使用队列。
代码库中唯一使用 `ShouldQueue` 的是 `DispatchWebhookJob`（Webhook 回调），与附件图片系统无关。
原因：图片上传和缩略图生成通常在用户交互时完成，需要即时反馈。

### 14.3 关于 GD vs Imagick

BookStack 明确只支持 GD：
- composer.json 未安装 `intervention/image-imagick`
- `ImageResizer` 中硬编码 `new Driver()` 即 `Gd\Driver`
- `interventionFromImageData()` 显式检查 `extension_loaded('gd')`
- 代码中没有任何 Imagick 相关的引用或回退逻辑

---

## 十五、代码位置索引

| 功能 | 文件:行 |
|---|---|
| 图片存储工厂 | `app/Uploads/ImageStorage.php` |
| 图片磁盘封装（路径适配+CRUD） | `app/Uploads/ImageStorageDisk.php` |
| 附件存储 | `app/Uploads/FileStorage.php` |
| 图片服务（CRUD + 清理 + 权限） | `app/Uploads/ImageService.php` |
| 附件服务 | `app/Uploads/AttachmentService.php` |
| 缩略图生成器（GD 驱动 + 异常处理） | `app/Uploads/ImageResizer.php` |
| 图片仓库（协调存储与缩略图） | `app/Uploads/ImageRepo.php` |
| OOM 内存保护工具 | `app/Util/OutOfMemoryHandler.php` |
| 清理 Artisan 命令 | `app/Console/Commands/CleanupImagesCommand.php` |
| 调度 Kernel（schedule 为空） | `app/Console/Kernel.php:17` |
| 维护页面控制器（清理入口） | `app/Settings/MaintenanceController.php:36` |
| Webhook Job（队列唯一使用者） | `app/Activity/DispatchWebhookJob.php` |
| Favicon 处理（也用 ImageResizer） | `app/Uploads/FaviconHandler.php` |
| 文件系统配置（disks 定义） | `app/Config/filesystems.php` |
| 清理 Web 路由 | `routes/web.php:231` |
| 清理维护页面视图 | `resources/views/settings/maintenance.blade.php:34` |
| 限流配置（RateLimiter 定义） | `app/App/Providers/RouteServiceProvider.php:79` |
| API 限流中间件（可配置覆盖） | `app/Http/Middleware/ThrottleApiRequests.php` |
| HTTP Kernel（middleware groups） | `app/Http/Kernel.php` |
| 限流 API 配置 | `app/Config/api.php` |
| 附件下载路由（不限流） | `routes/web.php:160` |
| 导出控制器（挂载 `throttle:exports`） | `app/Exports/Controllers/BookExportController.php:20` |
| ZIP 导出构建器（跨 disk 读取文件） | `app/Exports/ZipExports/ZipExportBuilder.php` |
| ZIP 文件引用管理（stream 读取） | `app/Exports/ZipExports/ZipExportFiles.php` |
| ZIP 导入执行器（跨 disk 写入） | `app/Exports/ZipExports/ZipImportRunner.php` |
| ZIP 导入仓库（事务+回滚） | `app/Exports/ImportRepo.php` |
| 导入控制器（Web 入口） | `app/Exports/Controllers/ImportController.php` |
| URL 更新命令（非文件迁移） | `app/Console/Commands/UpdateUrlCommand.php` |
| 附件下载控制器（权限前置校验） | `app/Uploads/Controllers/AttachmentController.php:212` |
| 图片私有访问控制器（权限前置校验） | `app/Uploads/Controllers/ImageController.php:32` |
| 图片路径权限校验（多层） | `app/Uploads/ImageService.php:251` |
| 图片关联权限检查（按类型） | `app/Uploads/ImageService.php:314` |
| Controller 基类权限检查方法 | `app/Http/Controller.php:63` |
| 权限中间件（路由级） | `app/Http/Middleware/CheckUserHasPermission.php` |
| 下载响应工厂（流式响应） | `app/Http/DownloadResponseFactory.php` |
| Range 支持流（断点续传） | `app/Http/RangeSupportedStream.php` |
| 导入模型（imports 表） | `app/Exports/Import.php` |
| 导入引用追踪（回滚用） | `app/Exports/ZipExports/ZipImportReferences.php` |
| 导入失败回滚磁盘文件 | `app/Exports/ZipExports/ZipImportRunner.php:103` |
| 附件表迁移（字段定义） | `database/migrations/2016_10_09_142037_create_attachments_table.php` |
| 页面修订表迁移（无附件字段） | `database/migrations/2015_08_09_093534_create_page_revisions_table.php` |
| 附件更新（覆盖式，无版本） | `app/Uploads/AttachmentService.php:65` |
| 附件详情更新 | `app/Uploads/AttachmentService.php:117` |
| 全局上传配额配置 | `app/Config/app.php:38` |
| 附件上传验证规则 | `app/Uploads/AttachmentService.php:183` |
| 图片上传验证规则（基类） | `app/Http/Controller.php:163` |
| ZIP 导入 data.json 大小检查 | `app/Exports/ZipExports/ZipExportReader.php:66` |
| ZIP 导入单文件大小检查 | `app/Exports/ZipExports/ZipExportReader.php:86` |
| ZIP 导入文件大小验证规则 | `app/Exports/ZipExports/ZipFileReferenceRule.php:25` |
| 嵌入 base64 图片大小检查 | `app/Entities/Tools/PageContent.php:160` |
