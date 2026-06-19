# Activity Log 与通知的代码流转

## 全局概览

BookStack 的 Activity Log 与通知系统围绕一个核心入口 `ActivityLogger::add()` 展开，一次活动事件会同时触发 **三条并行链路**：

```
用户操作 → ActivityLogger::add(type, detail)
              ├─ 链路1: 写入 activities 表（Activity Log 记录）
              ├─ 链路2: Session Flash 提示（操作者即时反馈）
              ├─ 链路3: NotificationManager → Handler → 邮件通知（订阅者可见提醒）
              ├─ 链路4: Webhook 分发（外部系统通知）
              └─ 链路5: Theme 事件派发（插件扩展点）
```

---

## 链路 1：事件记录 — Activity 持久化

### 入口

`ActivityLogger::add()` (`app/Activity/Tools/ActivityLogger.php:27`)

```php
public function add(string $type, string|Loggable $detail = ''): void
```

### 调用方式

业务代码通过 Facade 调用：

```php
Activity::add(ActivityType::PAGE_CREATE, $page);
```

- `Activity` Facade (`app/Facades/Activity.php`) → 绑定到容器单例 `activity`
- 容器注册 (`app/App/Providers/AppServiceProvider.php:38`)：`'activity' => ActivityLogger::class`
- `ActivityLogger` 以 **singleton** 注册，构造时注入 `NotificationManager` 并加载默认 Handler

### 写入逻辑

`ActivityLogger::newActivityForUser()` (`ActivityLogger.php:50`) 创建 Activity 模型实例：

| 字段 | 来源 |
|------|------|
| `type` | 传入的活动类型，如 `page_create` |
| `user_id` | 当前登录用户 `user()->id` |
| `ip` | 当前请求 IP，经 `IpFormatter` 格式化 |
| `detail` | 若 `$detail` 是 `Loggable` 则取 `logDescriptor()`，否则直接字符串 |
| `loggable_id` / `loggable_type` | 若 `$detail` 是 `Entity` 则记录多态关联 |

随后 `$activity->save()` 写入 `activities` 表。

### ActivityType 常量

`app/Activity/ActivityType.php` 定义了全部活动类型常量，涵盖：

- 页面操作：`PAGE_CREATE` / `PAGE_UPDATE` / `PAGE_DELETE` / `PAGE_RESTORE` / `PAGE_MOVE`
- 章节操作：`CHAPTER_CREATE` / `CHAPTER_UPDATE` / `CHAPTER_DELETE` / `CHAPTER_MOVE`
- 图书 / 书架操作：`BOOK_CREATE` / `BOOK_UPDATE` / `BOOK_DELETE` / `BOOK_SORT` / `BOOKSHELF_*`
- 评论操作：`COMMENT_CREATE` / `COMMENT_UPDATE` / `COMMENT_DELETE` / `COMMENTED_ON`
- 权限 / 设置 / 回收站 / 用户 / 角色 / Webhook / 导入 / 排序规则 等

### 调用方（典型）

| 调用位置 | 活动类型 |
|----------|---------|
| `PageRepo::create()` | `PAGE_CREATE` |
| `PageRepo::update()` | `PAGE_UPDATE` |
| `PageRepo::destroy()` | `PAGE_DELETE` |
| `PageRepo::restore()` | `PAGE_RESTORE` + `REVISION_RESTORE` |
| `PageRepo::move()` | `PAGE_MOVE` |
| `ChapterRepo` / `BookRepo` / `BookshelfRepo` | 对应 CREATE / UPDATE / DELETE |
| `CommentController` | `COMMENT_CREATE` / `COMMENT_UPDATE` / `COMMENT_DELETE` |
| `Controller::logActivity()` 基类方法 | 各类操作 |

### Activity 读取与展示

**ActivityQueries** (`app/Activity/ActivityQueries.php`) 负责查询：

| 方法 | 用途 |
|------|------|
| `latest($count, $page)` | 首页侧边栏 "Recent Activity" |
| `entityActivity($entity, $count, $page)` | 实体详情页的活动列表 |
| `userActivity($user, $count, $page)` | 用户个人页的活动列表 |

查询均经过 `PermissionApplicator::restrictEntityRelationQuery()` 过滤，确保用户只能看到有权查看的实体的活动。结果经 `filterSimilar()` 去重，合并连续相同 type + loggable 的记录。

**AuditLogController** (`app/Activity/Controllers/AuditLogController.php`) 提供管理员的审计日志页面，需要 `SettingsManage` + `UsersManage` 权限，支持按事件类型 / 日期 / 用户 / IP 过滤。

**前端视图**：`resources/views/common/activity-list.blade.php` + `activity-item.blade.php` 渲染活动列表，出现在首页侧边栏、图书详情页、用户 Profile 页等处。

---

## 链路 2：Session Flash 提示 — 操作者即时反馈

`ActivityLogger::setNotification()` (`ActivityLogger.php:76`)：

```php
protected function setNotification(string $type): void
{
    $notificationTextKey = 'activities.' . $type . '_notification';
    if (trans()->has($notificationTextKey)) {
        $message = trans($notificationTextKey);
        session()->flash('success', $message);
    }
}
```

- 仅当语言文件中存在对应 `activities.{type}_notification` 翻译键时才闪现
- 消息类型固定为 `success`
- 仅对当前操作者可见，是一次性提示

---

## 链路 3：通知分发 — 订阅者可见提醒（核心）

### 3.1 NotificationManager 注册与分发

`NotificationManager` (`app/Activity/Notifications/NotificationManager.php`) 在 `ActivityLogger` 构造时被注入并调用 `loadDefaultHandlers()`：

```php
public function loadDefaultHandlers(): void
{
    $this->registerHandler(ActivityType::PAGE_CREATE, PageCreationNotificationHandler::class);
    $this->registerHandler(ActivityType::PAGE_UPDATE, PageUpdateNotificationHandler::class);
    $this->registerHandler(ActivityType::COMMENT_CREATE, CommentCreationNotificationHandler::class);
    $this->registerHandler(ActivityType::COMMENT_CREATE, CommentMentionNotificationHandler::class);
    $this->registerHandler(ActivityType::COMMENT_UPDATE, CommentMentionNotificationHandler::class);
}
```

**注册表结构**：`$handlersByActivity` 数组，键为活动类型，值为 Handler 类名数组。一种活动类型可注册多个 Handler（如 `COMMENT_CREATE` 同时触发评论通知和 @提及通知）。

**分发逻辑** (`NotificationManager::handle()`):

```php
public function handle(Activity $activity, string|Loggable $detail, User $user): void
{
    $activityType = $activity->type;
    $handlersToRun = $this->handlersByActivity[$activityType] ?? [];
    foreach ($handlersToRun as $handlerClass) {
        $handler = new $handlerClass();
        $handler->handle($activity, $detail, $user);
    }
}
```

在 `ActivityLogger::add()` 中，`$this->notifications->handle($activity, $detail, user())` 在 Activity 保存之后被调用。

### 3.2 Watch（订阅）机制 — 决定谁收到通知

#### 数据模型

`Watch` 模型 (`app/Activity/Models/Watch.php`)，对应 `watches` 表：

| 字段 | 说明 |
|------|------|
| `user_id` | 订阅用户 |
| `watchable_type` | 订阅目标实体类型（book / chapter / page） |
| `watchable_id` | 订阅目标实体 ID |
| `level` | 订阅等级 |

#### 订阅等级（WatchLevels）

`app/Activity/WatchLevels.php`:

| 常量 | 值 | 含义 |
|------|----|------|
| `DEFAULT` | -1 | 默认，未设置 |
| `IGNORE` | 0 | 忽略所有通知 |
| `NEW` | 1 | 关注新内容 |
| `UPDATES` | 2 | 关注更新和新内容 |
| `COMMENTS` | 3 | 关注评论、更新和新内容 |

等级数值越大，覆盖的通知类型越多。`COMMENTS` 包含 `UPDATES` 包含 `NEW`。

#### 订阅设置入口

`WatchController::update()` (`app/Activity/Controllers/WatchController.php`)：
- 需要 `ReceiveNotifications` 权限
- 调用 `UserEntityWatchOptions::updateLevelByName()` 更新或删除 Watch 记录

`UserEntityWatchOptions` (`app/Activity/Tools/UserEntityWatchOptions.php`)：
- 管理单个用户对单个实体的订阅设置
- `updateLevelByValue()`: level < 0 删除记录，>= 0 则 `updateOrCreate`
- `getWatchMap()`: 查询当前用户在目标实体及其父实体上的所有 Watch 记录

#### 订阅触发 — EntityWatchers

`EntityWatchers` (`app/Activity/Tools/EntityWatchers.php`) 是通知 Handler 的核心工具，负责从 Watch 记录中计算出哪些用户应该收到通知。

**构造参数**：`EntityWatchers($entity, $watchLevel)` — 目标实体 + 所需最低等级

**计算逻辑** (`build()` 方法):

1. `getRelevantWatches()`: 查询与目标实体及其父实体关联的所有 Watch 记录
   - 对 Page：查 Page 自身 + 所属 Chapter + 所属 Book
   - 对 Chapter：查 Chapter 自身 + 所属 Book
   - 对 Book：查 Book 自身
2. 按 `watchable_type` 排序（book → chapter → page），确保同一用户取到最具体层级的设置
3. 按 `user_id` 去重，保留最具体层级的 level
4. 筛选出 `level >= $watchLevel` 的用户 ID 列为 `watchers`
5. 筛选出 `level === IGNORE(0)` 的用户 ID 列为 `ignorers`

**继承 + 覆盖语义**：用户在 Book 上设 `COMMENTS` 等级，则该 Book 下所有 Page/Chapter 的评论都会通知他；但如果他在某个 Chapter 上设 `IGNORE`，则该 Chapter 不再通知。`ignorers` 列表在 Handler 中用于排除特定用户（如页面所有者如果主动忽略则不通知）。

### 3.3 各 Handler 的通知逻辑

#### PageCreationNotificationHandler

`app/Activity/Notifications/Handlers/PageCreationNotificationHandler.php`

```
PAGE_CREATE 活动 → EntityWatchers($page, WatchLevels::NEW)
                 → 获取关注新内容的 watcher IDs
                 → sendNotificationToUserIds(PageCreationNotification, ...)
```

#### PageUpdateNotificationHandler

`app/Activity/Notifications/Handlers/PageUpdateNotificationHandler.php`

```
PAGE_UPDATE 活动 → 15 分钟去重：同一用户对同一页面 15 分钟内的连续更新只通知一次
                → EntityWatchers($page, WatchLevels::UPDATES) 获取 watcher IDs
                → 若页面有 owner 且不在 ignorers 中：
                    查 UserNotificationPreferences::notifyOnOwnPageChanges()
                    若开启则加入通知列表
                → sendNotificationToUserIds(PageUpdateNotification, ...)
```

#### CommentCreationNotificationHandler

`app/Activity/Notifications/Handlers/CommentCreationNotificationHandler.php`

```
COMMENT_CREATE 活动 → EntityWatchers($page, WatchLevels::COMMENTS) 获取 watcher IDs
                    → 页面 owner：若不在 ignorers 且 notifyOnOwnPageComments 开启 → 加入
                    → 父评论创建者：若不在 ignorers 且 notifyOnCommentReplies 开启 → 加入
                    → sendNotificationToUserIds(CommentCreationNotification, ...)
```

#### CommentMentionNotificationHandler

`app/Activity/Notifications/Handlers/CommentMentionNotificationHandler.php`

```
COMMENT_CREATE / COMMENT_UPDATE 活动 → MentionParser::parseUserIdsFromHtml() 解析评论 HTML 中的 @提及
                                    → 从 <a data-mention-user-id="..."> 提取被提及用户 ID
                                    → 过滤出 notifyOnCommentMentions 开启的用户
                                    → 若是 COMMENT_UPDATE：查 MentionHistory 排除已通知过的用户，避免重复
                                    → 记录到 MentionHistory 表
                                    → sendNotificationToUserIds(CommentMentionNotification, ...)
```

### 3.4 BaseNotificationHandler::sendNotificationToUserIds() — 统一发送逻辑

`app/Activity/Notifications/Handlers/BaseNotificationHandler.php:19`

对每个目标用户依次检查：

| 检查项 | 说明 |
|--------|------|
| 发起者排除 | `$user->id === $initiator->id` 则跳过，不给自己发通知 |
| 权限检查 | `$user->can(Permission::ReceiveNotifications)` |
| 内容可见性 | `PermissionApplicator::checkOwnableUserAccess($relatedModel, 'view')` 确保用户有权查看关联实体 |

全部通过后：`$user->notify(new $notification($detail, $initiator))`

利用 Laravel Notification 机制发送，异常时 Log 记录但不中断。

### 3.5 通知消息类

所有通知消息类继承 `BaseActivityNotification` (`app/Activity/Notifications/Messages/BaseActivityNotification.php`)，而 `BaseActivityNotification` 继承 `MailNotification` (`app/App/MailNotification.php`)。

`MailNotification` 继承 Laravel `Notification` 并实现 `ShouldQueue`，通知通过队列异步发送邮件。

**via 通道**：`['mail']` — 仅邮件通道，无数据库通知通道。

| 通知消息类 | 触发 Handler | 邮件内容 |
|-----------|-------------|---------|
| `PageCreationNotification` | PageCreationNotificationHandler | 新页面名称 + 路径 + 创建者 |
| `PageUpdateNotification` | PageUpdateNotificationHandler | 页面名称 + 路径 + 更新者 + 防抖说明 |
| `CommentCreationNotification` | CommentCreationNotificationHandler | 页面名称 + 路径 + 评论者 + 评论内容 |
| `CommentMentionNotification` | CommentMentionNotificationHandler | 页面名称 + 路径 + 评论者 + 评论内容 |

每封邮件底部附带 "管理通知偏好" 链接，指向 `/my-account/notifications`。

### 3.6 用户通知偏好

`UserNotificationPreferences` (`app/Settings/UserNotificationPreferences.php`) 管理 4 项个人偏好：

| 偏好键 | 含义 | 使用者 |
|--------|------|--------|
| `own-page-changes` | 自己拥有的页面被修改时通知 | PageUpdateNotificationHandler |
| `own-page-comments` | 自己拥有的页面收到评论时通知 | CommentCreationNotificationHandler |
| `comment-replies` | 自己的评论收到回复时通知 | CommentCreationNotificationHandler |
| `comment-mentions` | 在评论中被 @提及时通知 | CommentMentionNotificationHandler |

存储方式：`setting()->putUser($user, 'notifications#' . $key, $value)`，即用户设置表中的 `notifications#own-page-changes` 等键。

**设置页面**：`UserAccountController::showNotifications()` (`app/Users/Controllers/UserAccountController.php:124`)，路由 `/my-account/notifications`，同时展示当前用户的所有 Watch 订阅列表。

---

## 链路 4：Webhook 分发

`ActivityLogger::dispatchWebhooks()` (`ActivityLogger.php:85`)：

1. 查询所有活跃 Webhook (`Webhook::where('active', true)`)，且 `trackedEvents` 包含当前事件类型或 `all`
2. 对每个 Webhook 调度 `DispatchWebhookJob`（队列异步执行）
3. `DispatchWebhookJob` (`app/Activity/DispatchWebhookJob.php`) 构造请求体（经 `WebhookFormatter` 格式化或被 `ThemeEvents::WEBHOOK_CALL_BEFORE` 主题事件覆盖），POST 到 Webhook 端点

---

## 链路 5：Theme 事件派发

`ActivityLogger::add()` 末尾调用：

```php
Theme::dispatch(ThemeEvents::ACTIVITY_LOGGED, $type, $detail);
```

供插件系统监听所有活动事件做自定义处理。

---

## 完整流转图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户操作（创建页面 / 更新页面 / 评论等）         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Activity::add(type, detail)                                         │
│  → ActivityLogger::add()                                             │
│  (app/Activity/Tools/ActivityLogger.php:27)                         │
└─────────────┬────────────┬──────────────┬──────────────┬────────────┘
              │            │              │              │
     ┌────────▼───┐  ┌─────▼──────┐  ┌───▼──────────┐  │
     │ 保存到      │  │ Session    │  │ Notification │  │
     │ activities │  │ Flash      │  │ Manager      │  │
     │ 表         │  │ 提示       │  │ ::handle()   │  │
     │            │  │            │  │              │  │
     │ type       │  │ success    │  │ 按 type 查   │  │
     │ user_id    │  │ flash      │  │ 找已注册     │  │
     │ ip         │  │ 消息       │  │ Handler 列表  │  │
     │ detail     │  │            │  │              │  │
     │ loggable_* │  │            │  │              │  │
     └────────────┘  └────────────┘  └──┬───────────┘  │
                                      │               │
                    ┌─────────────────┼────────┐      │
                    │                 │        │      │
          ┌─────────▼──────┐ ┌───────▼──────┐ │      │
          │ PageCreation   │ │ PageUpdate   │ │      │
          │ Handler        │ │ Handler      │ │      │
          │                │ │              │ │      │
          │ EntityWatchers │ │ EntityWatchers│ │      │
          │ (NEW)          │ │ (UPDATES)    │ │      │
          │                │ │ + 页面Owner  │ │      │
          │                │ │ + 15min去重  │ │      │
          └───────┬────────┘ └──────┬───────┘ │      │
                  │                 │         │      │
          ┌───────▼────────┐ ┌─────▼─────────┐│      │
          │ CommentCreation│ │ CommentMention ││      │
          │ Handler        │ │ Handler        ││      │
          │                │ │                ││      │
          │ EntityWatchers │ │ MentionParser  ││      │
          │ (COMMENTS)     │ │ 解析 @提及     ││      │
          │ + 页面Owner    │ │ + 偏好过滤     ││      │
          │ + 父评论者     │ │ + 去重历史     ││      │
          └───────┬────────┘ └──────┬────────┘│      │
                  │                 │         │      │
                  └────────┬────────┘         │      │
                           │                  │      │
                  ┌────────▼────────┐         │      │
                  │ BaseNotification│         │      │
                  │ Handler        │         │      │
                  │ ::sendToUserIds│         │      │
                  │                │         │      │
                  │ 逐用户检查:     │         │      │
                  │ 1.非发起者      │         │      │
                  │ 2.有通知权限    │         │      │
                  │ 3.有内容查看权  │         │      │
                  │                │         │      │
                  │ $user->notify()│         │      │
                  │ → 队列发邮件   │         │      │
                  └────────────────┘         │      │
                                              │      │
                              ┌───────────────▼──┐   │
                              │ Webhook 分发      │   │
                              │ (DispatchWebhook │   │
                              │  Job, 队列异步)   │   │
                              └──────────────────┘   │
                                                      │
                              ┌───────────────────────▼──┐
                              │ Theme::dispatch          │
                              │ (ACTIVITY_LOGGED)        │
                              └─────────────────────────-┘
```

---

## 关键数据表

| 表 | 用途 |
|----|------|
| `activities` | 活动日志记录，存储所有用户操作 |
| `watches` | 用户订阅关系，记录用户对实体的关注等级 |
| `mention_history` | @提及通知历史，用于编辑评论时避免重复通知 |
| `notifications`（Laravel 内置） | 邮件通知队列记录（因仅使用 mail 通道，此表可能为空） |

---

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `app/Activity/Tools/ActivityLogger.php` | 核心入口，活动记录 + 通知触发 + Webhook |
| `app/Activity/Models/Activity.php` | Activity Eloquent 模型 |
| `app/Activity/ActivityType.php` | 活动类型常量定义 |
| `app/Activity/ActivityQueries.php` | Activity 查询（首页 / 实体 / 用户） |
| `app/Facades/Activity.php` | Activity Facade |
| `app/Activity/Notifications/NotificationManager.php` | 通知分发管理器，Handler 注册表 |
| `app/Activity/Notifications/Handlers/NotificationHandler.php` | Handler 接口 |
| `app/Activity/Notifications/Handlers/BaseNotificationHandler.php` | Handler 基类，统一发送逻辑 |
| `app/Activity/Notifications/Handlers/PageCreationNotificationHandler.php` | 新页面通知 |
| `app/Activity/Notifications/Handlers/PageUpdateNotificationHandler.php` | 页面更新通知 |
| `app/Activity/Notifications/Handlers/CommentCreationNotificationHandler.php` | 新评论通知 |
| `app/Activity/Notifications/Handlers/CommentMentionNotificationHandler.php` | @提及通知 |
| `app/Activity/Notifications/Messages/BaseActivityNotification.php` | 通知消息基类 |
| `app/Activity/Notifications/Messages/PageCreationNotification.php` | 新页面邮件 |
| `app/Activity/Notifications/Messages/PageUpdateNotification.php` | 页面更新邮件 |
| `app/Activity/Notifications/Messages/CommentCreationNotification.php` | 新评论邮件 |
| `app/Activity/Notifications/Messages/CommentMentionNotification.php` | @提及邮件 |
| `app/Activity/Models/Watch.php` | Watch Eloquent 模型 |
| `app/Activity/WatchLevels.php` | 订阅等级常量 |
| `app/Activity/Tools/EntityWatchers.php` | 从 Watch 记录计算通知目标用户 |
| `app/Activity/Tools/UserEntityWatchOptions.php` | 用户订阅设置工具 |
| `app/Activity/Controllers/WatchController.php` | 订阅设置 API |
| `app/Activity/Tools/MentionParser.php` | 从 HTML 解析 @提及用户 ID |
| `app/Activity/Models/MentionHistory.php` | 提及通知历史模型 |
| `app/Settings/UserNotificationPreferences.php` | 用户通知偏好读写 |
| `app/App/MailNotification.php` | 邮件通知基类（ShouldQueue） |
| `app/Activity/Controllers/AuditLogController.php` | 管理员审计日志页面 |
| `app/Permissions/Permission.php` | `ReceiveNotifications` 权限枚举 |
