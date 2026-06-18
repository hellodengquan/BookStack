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

### 2.3 路径适配机制

`ImageStorageDisk::adjustPathForDisk()` at `app/Uploads/ImageStorageDisk.php:34`
- local_secure_images 磁盘：去掉 `uploads/images/` 前缀（因为 root 已经指向那里）
- 其他磁盘：保留 `uploads/images/` 前缀

`FileStorage::adjustPathForStorageDisk()` at `app/Uploads/FileStorage.php:121`
- 同理，对 local_secure_attachments 做路径裁剪

### 2.4 图片类型

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

---

## 四、孤儿资源清理

### 4.1 触发入口

| 入口 | 位置 | 说明 |
|---|---|---|
| Artisan 命令 | `app/Console/Commands/CleanupImagesCommand.php` | `php artisan bookstack:cleanup-images` |
| Web 管理后台 | `app/Settings/MaintenanceController.php:cleanupImages()` | 设置 → 维护 → 图片清理 |

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

## 五、三者串联关系

### 5.1 图片上传 → 存储 → 缩略图生成 链路

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

### 5.2 图片访问 → 缩略图懒加载 链路

```
ImageController::edit() / GalleryImageController::list()
  → ImageResizer::loadGalleryThumbnailsForMany() / loadGalleryThumbnailsForImage()
      → resizeToThumbnailUrl(..., shouldCreate=false)
          → Cache::get() → 命中则返回
          → disk->exists() → 存在则缓存并返回
          └─ 都没有 → 生成新缩略图
```

### 5.3 孤儿清理 → 存储 → 缩略图删除 链路

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

### 5.4 数据模型关系

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

## 六、关键设计模式

### 6.1 存储抽象层

`ImageStorage` 作为工厂 + 门面，`ImageStorageDisk` 作为具体磁盘封装。
好处：
- 统一路径适配逻辑
- 屏蔽不同磁盘（local/s3）的差异
- 提供图片特定的操作（如批量删除同名文件）

### 6.2 缩略图懒生成 + 缓存

- 不提前生成，按需创建
- 内存缓存（1周）避免每次都查磁盘
- 多级查找：缓存 → 磁盘 → 生成

### 6.3 孤儿清理策略

- 基于文件名模糊匹配（LIKE），简单但可能误判
- 只清理 gallery 和 drawio 类型（用户上传的内容图片）
- 支持 dry-run，安全第一
- 清理时同时删除所有缩略图（通过文件名匹配）

---

## 七、附件 vs 图片对比

| 特性 | 图片 | 附件 |
|---|---|---|
| 存储类 | `ImageStorage` + `ImageStorageDisk` | `FileStorage` |
| 服务类 | `ImageService` | `AttachmentService` |
| 缩略图 | 有（`ImageResizer`） | 无 |
| 孤儿清理 | 有（`deleteUnusedImages`） | 无 |
| 路径命名 | 语义化文件名（可能加随机前缀） | 纯随机 16 字符 |
| 安全模式 | local_secure / local_secure_restricted | 统一 local_secure_attachments |
| 数据库表 | images | attachments |

---

## 八、代码位置索引

| 功能 | 文件:行 |
|---|---|
| 图片存储工厂 | `app/Uploads/ImageStorage.php` |
| 图片磁盘封装 | `app/Uploads/ImageStorageDisk.php` |
| 附件存储 | `app/Uploads/FileStorage.php` |
| 图片服务（CRUD + 清理） | `app/Uploads/ImageService.php` |
| 附件服务 | `app/Uploads/AttachmentService.php` |
| 缩略图生成器 | `app/Uploads/ImageResizer.php` |
| 图片仓库（协调层） | `app/Uploads/ImageRepo.php` |
| 清理命令 | `app/Console/Commands/CleanupImagesCommand.php` |
| 维护页面控制器 | `app/Settings/MaintenanceController.php:36` |
| 文件系统配置 | `app/Config/filesystems.php` |
