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

---

## 专项深挖 1：通知去重与合并 — 避免重复推送的完整判定

系统在 **4 个层级** 上做了去重 / 合并逻辑，分布在数据存储、Handler 业务、Watch 计算、查询展示四个阶段。

### 层级 A：Activity 展示层 — `filterSimilar` 相似度折叠

`ActivityQueries::filterSimilar()` (`app/Activity/ActivityQueries.php:102`)

```php
protected function filterSimilar(iterable $activities): array
{
    $newActivity = [];
    $previousItem = null;
    foreach ($activities as $activityItem) {
        if (!$previousItem || !$activityItem->isSimilarTo($previousItem)) {
            $newActivity[] = $activityItem;
        }
        $previousItem = $activityItem;
    }
    return $newActivity;
}
```

`Activity::isSimilarTo()` (`app/Activity/Models/Activity.php:76`) 的判定条件是 **完全相等** 的三元组：

```php
[$this->type, $this->loggable_type, $this->loggable_id] === [$activityB->type, $activityB->loggable_type, $activityB->loggable_id]
```

| 判定维度 | 说明 |
|----------|------|
| 去重时机 | 查询结果返回前（`latest()` / `entityActivity()` / `userActivity()` 都会调用） |
| 判定条件 | 连续两条记录的 type + loggable_type + loggable_id 完全相同 |
| 作用范围 | 仅对 UI 展示生效，不影响通知发送；通知发送走的是独立 Handler 路径 |
| 典型场景 | 同一用户对同一页面连续保存多次，在活动列表里只展示一条 |

注意：该过滤只对"相邻"的相似记录生效，必须先按 `created_at desc` 排序才能保证效果。

### 层级 B：Watch 计算层 — `EntityWatchers` 用户级去重

`EntityWatchers::build()` (`app/Activity/Tools/EntityWatchers.php:40`)

**问题背景**：同一用户可能同时订阅了 Page + Chapter + Book，三层 Watch 记录都存在。如果不过滤，同一个用户会被重复加入通知列表。

**处理逻辑**：

1. `getRelevantWatches()` 拉取目标实体及其所有父实体（Page → Chapter → Book）的全部 Watch 记录
2. 按 `watchable_type` 排序（`book` → `chapter` → `page`），确保层级由粗到细
3. 按 `user_id` 去重：`$levelByUserId[$watch->user_id] = $watch->level`
   - 后赋值覆盖先赋值 → 最终每个 user_id 保留的是 **最具体层级**（Page 层）的 level 值
4. 按目标 `watchLevel` 筛选：`level >= watchLevel` 为 watcher，`level === IGNORE(0)` 为 ignorer

**效果**：无论用户在多少层级上订阅了同一棵实体树，最终只会在通知列表里出现一次，且取的是最具体（优先级最高）的订阅等级。

### 层级 C：Handler 业务层 — 针对每种通知的定制去重

#### C-1 Page Update：15 分钟时间窗防抖

`PageUpdateNotificationHandler::handle()` (`app/Activity/Notifications/Handlers/PageUpdateNotificationHandler.php:25`)

```php
// 查询该页面的上一条 PAGE_UPDATE 活动（排除当前这条）
$lastUpdate = $detail->activity()
    ->where('type', '=', ActivityType::PAGE_UPDATE)
    ->where('id', '!=', $activity->id)
    ->latest('created_at')
    ->first();

// 同一用户在 15 分钟内对同一页面的连续更新 → 跳过通知
if ($lastUpdate && $lastUpdate->user_id === $user->id) {
    if ($lastUpdate->created_at->gt(now()->subMinutes(15))) {
        return;
    }
}
```

| 判定维度 | 值 |
|----------|----|
| 防抖时长 | 15 分钟（硬编码） |
| 判定范围 | 同一页面 + 同一用户 + 活动类型为 PAGE_UPDATE |
| 时间基准 | 上一条该页面 PAGE_UPDATE 活动的 `created_at` |
| 行为 | 满足条件时整个 Handler `return`，不发任何通知 |

**边界情况**：
- A 用户更新 → 通知正常发出 → B 用户 10 分钟后更新 → **正常通知**（user_id 不同）
- A 用户更新 → 10 分钟后 A 再次更新 → **跳过**
- A 用户更新 → 20 分钟后 A 再次更新 → **正常通知**

#### C-2 Comment Mention：编辑评论时避免重复 @通知

`CommentMentionNotificationHandler::handle()` (`app/Activity/Notifications/Handlers/CommentMentionNotificationHandler.php:43`)

```php
// 仅在 COMMENT_UPDATE 时检查历史
if ($activity->type === ActivityType::COMMENT_UPDATE) {
    $previouslyNotifiedUserIds = $this->getPreviouslyNotifiedUserIds($detail);
    $receivingNotificationsUserIds = array_values(
        array_diff($receivingNotificationsUserIds, $previouslyNotifiedUserIds)
    );
}
```

**判定依据**：`mention_history` 表，记录每次 @提及通知的发送历史。

`MentionHistory` (`app/Activity/Models/MentionHistory.php`) 结构：

| 字段 | 含义 |
|------|------|
| `mentionable_type` + `mentionable_id` | 被提及所在的 Comment（多态） |
| `from_user_id` | 提及者 |
| `to_user_id` | 被提及者 |
| `created_at` / `updated_at` | 时间戳 |

发送前写入历史：`CommentMentionNotificationHandler::logMentions()`（批量 insert）

**去重逻辑**：
- COMMENT_CREATE：首次创建评论，直接通知所有 @用户，同时写入历史
- COMMENT_UPDATE：编辑评论时，先从 mention_history 查出该评论曾通知过谁，用 `array_diff` 扣除已通知用户，只给**本次新增的 @用户**发通知

### 层级 D：发送层 — `BaseNotificationHandler` 的发起者排除

`BaseNotificationHandler::sendNotificationToUserIds()` (`app/Activity/Notifications/Handlers/BaseNotificationHandler.php:26`)

```php
// Prevent sending to the user that initiated the activity
if ($user->id === $initiator->id) {
    continue;
}
```

这是最后一道过滤，防止"自己操作了自己订阅的页面给自己发通知"。配合 Handler 内的 array_unique userIds，保证最终每个用户只收到一封邮件。

### 去重机制汇总

| 层级 | 实现位置 | 去重对象 | 判定条件 |
|------|---------|---------|---------|
| D. 发送层 | `BaseNotificationHandler` | 通知目标用户 | 排除发起者 + `array_unique` userIds |
| C-2. Handler 业务 | `CommentMentionNotificationHandler` | @提及通知 | `mention_history` 排除已通知用户 |
| C-1. Handler 业务 | `PageUpdateNotificationHandler` | 页面更新通知 | 同一用户同一页面 15 分钟内只发一次 |
| B. Watch 计算 | `EntityWatchers` | Watch 订阅用户 | 按 user_id 取最具体层级的 level，去重 |
| A. 展示层 | `ActivityQueries::filterSimilar` | Activity 列表展示 | 相邻记录 type+loggable 相同则折叠 |

---

## 专项深挖 2：推送渠道分流 — 站内 / 邮件 / Mobile Push

### 结论先行：BookStack 只有邮件通道

代码中 **不存在** 站内通知（database channel）、Mobile Push、Slack、Pusher/Broadcast 等推送渠道。

### 证据链

#### 唯一通知基类 `MailNotification` 的 `via()` 硬编码 mail

`app/App/MailNotification.php:28`

```php
public function via($notifiable)
{
    return ['mail'];
}
```

所有通知类都继承 `MailNotification`：

| 通知类 | 父类 | via() |
|--------|------|-------|
| `PageCreationNotification` | `BaseActivityNotification` → `MailNotification` | `['mail']` |
| `PageUpdateNotification` | `BaseActivityNotification` → `MailNotification` | `['mail']` |
| `CommentCreationNotification` | `BaseActivityNotification` → `MailNotification` | `['mail']` |
| `CommentMentionNotification` | `BaseActivityNotification` → `MailNotification` | `['mail']` |
| `UserInviteNotification` | `MailNotification` | `['mail']` |
| `ResetPasswordNotification` | `MailNotification` | `['mail']` |

没有任何子类覆盖 `via()` 增加其他通道。

#### User 模型不具备 DatabaseNotification 能力

`app/Users/Models/User.php` 使用了 `Illuminate\Notifications\Notifiable` trait，这个 trait 自带 `routeNotificationForMail()`（返回 email）等默认路由，但代码中：

- **不存在** `routeNotificationForDatabase()` / `routeNotificationForNexmo()` / `routeNotificationForSlack()` / `routeNotificationForBroadcast()` 等自定义路由方法
- **没有任何地方** 调用 `$user->notifications` / `$user->unreadNotifications` / `DatabaseNotification` 模型
- 搜索 `HasDatabaseNotifications` 无结果 → 说明 Laravel 的 database notification channel 未被使用

#### 没有 Broadcast / Pusher / WebSocket 基础设施

- 项目配置中没有 `broadcasting.php`
- `app/Config/services.php` 中无 pusher / ably / redis broadcast 配置
- 搜索 `BroadcastChannel`、`ShouldBroadcast`、`pusher` 无结果
- 前端代码中无 WebSocket / Pusher 客户端初始化

#### 邮件传输层配置

`app/Config/mail.php` 提供的 mailer 选项：

| mailer | transport |
|--------|-----------|
| `smtp` | SMTP（默认） |
| `sendmail` | sendmail 命令 |
| `log` | 写入日志 |
| `array` | 内存数组（测试用） |
| `failover` | smtp 失败后降级写入 log |

用户可以通过 `MAIL_DRIVER` 环境变量切换，但都还是 **邮件通道内部的传输方式切换**，不是不同推送渠道。

### 系统里"通知偏好"的真实含义

`UserNotificationPreferences` 的 4 项偏好（`own-page-changes` / `own-page-comments` / `comment-replies` / `comment-mentions`）控制的是 **邮件是否发送**，而不是在多个渠道间做选择。代码中没有"渠道选择偏好"的概念。

### 渠道扩展可能性

如果未来要加站内通知或 Push，理论上的改造点：

1. 覆盖 `MailNotification::via()` 返回 `['mail', 'database']` 等多通道
2. 实现 `toDatabase()` / `toBroadcast()` 方法
3. User 模型增加 `routeNotificationFor*` 方法做 Push 设备注册
4. 增加 UI 展示 `notifications` 表中的站内消息

但当前代码完全没有这些。

---

## 专项深挖 3：队列堆积时的削峰、丢弃、降级策略

### 结论先行：几乎没有业务层的削峰策略，完全依赖 Laravel 队列框架的默认行为

BookStack 在通知和 Webhook 的 Job 类上没有自定义任何重试次数、超时、延迟、唯一键、中间件。所有行为都是 Laravel 默认。

### 队列配置

`app/Config/queue.php`

| 配置项 | 值 | 说明 |
|--------|----|------|
| `default` driver | `env('QUEUE_CONNECTION', 'sync')` | **默认 sync（同步执行）**，即默认情况下根本不经过队列 |
| database 连接 `retry_after` | 90 秒 | 超过 90 秒未完成的 Job 会被认为已失败，可以被其他 worker 重新执行 |
| redis 连接 `retry_after` | 90 秒 | 同上 |
| `failed.driver` | `database-uuids` | 失败 Job 写入 `failed_jobs` 表 |

**关键发现**：默认 `QUEUE_CONNECTION=sync`，这意味着默认部署下，邮件通知和 Webhook 调用是在请求线程里同步执行的，没有异步削峰能力。只有管理员显式配置了 `database` 或 `redis` 队列连接，异步机制才会生效。

### 两个具体 Job 的实现

#### MailNotification（邮件通知 Job）

`app/App/MailNotification.php`：

```php
abstract class MailNotification extends Notification implements ShouldQueue
{
    use Queueable;
    // 没有 public $tries / public $timeout / backoff() / retryUntil() / uniqueId()
}
```

Laravel 默认行为：

- **重试次数**：无上限（除非 worker 命令行参数 `--tries` 指定）
- **超时**：无限制（除非 `--timeout` 指定或 `pcntl` 扩展支持）
- **退避延迟**：0（无 backoff）
- **任务唯一**：未实现 `ShouldBeUnique` 接口，不保证唯一
- **失败处理**：未实现 `failed()` 方法，异常直接写入 `failed_jobs` 表

**Handler 内的 try/catch**：`BaseNotificationHandler::sendNotificationToUserIds()` 对 `$user->notify()` 包了一层 try/catch：

```php
try {
    $user->notify(new $notification($detail, $initiator));
} catch (\Exception $exception) {
    Log::error("Failed to send email notification to user [id:{$user->id}] with error: {$exception->getMessage()}");
}
```

这个 try/catch 只在 **sync 驱动或通知进入队列前** 的异常生效。如果是异步队列，Job 序列化后投递成功，实际发送时的异常不会被这个 try/catch 捕获，而是走 Laravel 的失败 Job 流程。

#### DispatchWebhookJob（Webhook Job）

`app/Activity/DispatchWebhookJob.php`：

```php
class DispatchWebhookJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    // 同样没有 $tries / $timeout / backoff() 等配置
}
```

额外特性：`use InteractsWithQueue`，但 Job 内部未调用 `$this->release()` / `$this->fail()` 等方法，全靠默认行为。

Webhook 调用失败时：
- 记录 `last_errored_at` 和 `last_error` 到 `webhooks` 表
- 记录 `last_called_at`
- 不会触发重试，不会延迟重放

### Laravel 默认重试策略的实际效果

当队列堆积时：

| 场景 | Laravel 默认行为 | 业务代码额外动作 |
|------|-----------------|-----------------|
| 邮件 SMTP 超时 / 拒绝连接 | 无限重试直到 failed_jobs（除非 worker 设了 --tries） | 仅 Log 记录错误 |
| Webhook 端点 4xx / 5xx | Job 视为成功（因为 HTTP 异常被 catch 了），不重试 | 仅记录 last_error 到 webhook 表 |
| 队列 worker 挂掉重启 | retry_after（90s）后 Job 可被重新消费，可能重复发送 | 无去重保护 |
| 瞬时大流量 | sync 驱动下阻塞请求线程；database/redis 驱动下排队但无优先级 | 无优先级、无熔断、无批量丢弃 |

### 没有的策略

代码中完全**不存在**以下机制：

| 策略类型 | 是否存在 | 搜索证据 |
|----------|---------|---------|
| 任务唯一键（ShouldBeUnique / uniqueId） | ❌ | `ShouldBeUnique` 仅 DispatchWebhookJob 用了 `InteractsWithQueue` 但未实现接口 |
| 限流 / 频控（RateLimiter） | ❌ | 仅 API 层有 `ThrottleApiRequests`（基于 `api.requests_per_minute` 配置），通知 / Webhook Job 中无 |
| 指数退避（backoff） | ❌ | 无 `backoff()` 属性或方法 |
| 最大重试次数（tries） | ❌ | 无 `public $tries` 定义 |
| 超时控制（timeout） | ❌ | 无 `public $timeout`；Webhook HTTP 客户端 `connect_timeout=10`，但总 timeout 由 webhook.timeout 列控制 |
| 任务优先级（onQueue） | ❌ | 所有 Job 都在 `default` 队列，无高低优先级分离 |
| 批量合并（batch） | ❌ | 配置中有 `job_batches` 表，但 Activity 通知无批量逻辑 |
| 调度器清理（Scheduled） | ❌ | `app/Console/Kernel.php` 的 `schedule()` 方法为空，无自动清理 failed_jobs 或过期 activities |
| 熔断 / 降级 | ❌ | Webhook 失败不影响后续调用；邮件失败不降级为站内通知（因为没有站内通道） |

### 唯二的"降级"相关代码

1. **MAIL_DRIVER=failover** (`app/Config/mail.php:60`)：当 SMTP 发送失败时，Laravel 的 `failover` transport 会自动回退到 `log` mailer，把邮件写入日志而不是报错。这是 Laravel 框架层的降级，非业务代码实现。

2. **bookstack:clear-activity** (`app/Console/Commands/ClearActivityCommand.php`)：提供了手动清空 `activities` 表的命令（`Activity::query()->truncate()`），但这是运维手动操作，非自动削峰。Console `schedule()` 为空，也没有 cron 自动清理。

---

## 专项深挖补充：三层全景总览

```
┌───────────────────────────────────────────────────────────────────┐
│                      推送渠道（实际仅有 mail）                      │
│                                                                   │
│  MailNotification::via() → ['mail']                               │
│     └─ Mailer 传输方式可切换（MAIL_DRIVER）                        │
│         ├─ smtp    （默认）                                        │
│         ├─ sendmail                                               │
│         ├─ log     （写入日志）                                     │
│         ├─ array   （测试）                                        │
│         └─ failover（smtp→log 降级）                               │
│                                                                   │
│  其他渠道（database/broadcast/push/slack）: ❌ 不存在               │
└───────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                       去重 / 合并（4 层叠加）                        │
│                                                                   │
│  D. 发送层（BaseNotificationHandler）                              │
│     ├─ 排除发起者                                                  │
│     └─ array_unique(userIds)                                      │
│                                                                   │
│  C. Handler 业务层                                                 │
│     ├─ PageUpdate: 15 分钟同用户防抖                               │
│     └─ CommentMention: mention_history 排除已通知                  │
│                                                                   │
│  B. Watch 计算层（EntityWatchers）                                 │
│     └─ 按 user_id 去重，取最具体层级 level                          │
│                                                                   │
│  A. 展示层（ActivityQueries::filterSimilar）                       │
│     └─ 相邻记录 type+loggable 相同折叠（仅 UI）                    │
└───────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                   队列削峰 / 降级（几乎无业务实现）                   │
│                                                                   │
│  默认: QUEUE_CONNECTION=sync → 同步执行，无队列                    │
│                                                                   │
│  配置 database/redis 后:                                           │
│     ├─ retry_after=90s（超时后可被重新消费）                        │
│     ├─ Job 无 tries / timeout / backoff / unique → 全靠默认        │
│     ├─ 失败写入 failed_jobs 表                                     │
│     └─ 无优先级 / 限流 / 熔断 / 批量合并                            │
│                                                                   │
│  运维手段:                                                         │
│     ├─ bookstack:clear-activity（手动清空 activities）             │
│     └─ failover mailer（smtp→log 自动降级）                        │
└───────────────────────────────────────────────────────────────────┘
```

---

## 专项深挖 4：通知中心未读 / 已读状态的存储模式与索引

### 结论先行：BookStack **没有** 通知中心

系统不存在"站内通知中心"、"未读消息红点"、"已读 / 未读状态"等功能。以下是完整证据链。

### 4.1 数据库层：不存在 notifications 表的迁移

`database/migrations/` 目录下的所有迁移中搜索 `notifications` 表相关迁移，唯一匹配的是：

- `2023_07_25_124945_add_receive_notifications_role_permissions.php` — 只是加了一个角色权限常量，与数据表无关

**Laravel 默认的 `notifications` 表（DatabaseNotification）从未在 BookStack 中被创建或使用。** 代码中搜索 `HasDatabaseNotifications`、`unreadNotifications`、`readNotifications`、`markAsRead`、`read_at` 均无结果。

### 4.2 模型层：User 只 `use Notifiable`，无 DatabaseNotification 能力

`app/Users/Models/User.php:53`：

```php
use Notifiable;
```

`Illuminate\Notifications\Notifiable` trait 提供的是 `routeNotificationForMail()` 等通用路由方法，`DatabaseNotification` 能力需要额外 `use HasDatabaseNotifications` trait，BookStack 的 User 模型没有使用。

### 4.3 通知消息层：`via()` 只返回 `['mail']`

所有通知消息类继承 `MailNotification` (`app/App/MailNotification.php:28`)：

```php
public function via($notifiable)
{
    return ['mail'];
}
```

没有任何子类覆盖 `via()` 加入 `'database'` 通道，也没有实现 `toDatabase()` 方法。

### 4.4 前端层：无通知铃铛 / 未读计数 / 标记已读 UI

搜索 `notification` 相关的 Blade 视图，只有 `users/account/notifications.blade.php` —— 这是**用户偏好设置页面**，用于开关邮件通知、管理 Watch 订阅，不是"通知中心"列表页。

Header 中 (`layouts/parts/header.blade.php`) 无铃铛图标、无未读计数。搜索 `bell` / `notification-badge` / `unread-count` 无结果。

### 4.5 对照：哪些表是真实存在且有索引的

虽然没有站内通知中心，但系统有三张与活动 / 通知相关的核心表，其索引如下：

#### `activities` 表 — 活动日志

| 字段 | 类型 | 索引 | 迁移 |
|------|------|------|------|
| `id` | int | PK | `2015_08_16_142133_create_activities_table` |
| `type` (原 `key`) | string | INDEX | `2020_09_19_094251_add_activity_indexes` |
| `created_at` | timestamp | INDEX | `2020_09_19_094251_add_activity_indexes` |
| `ip` | string(45) | INDEX (`activities_ip_index`) | `2021_11_26_070438_add_index_for_user_ip` |
| `loggable_id` (原 `entity_id`) | int nullable | 无单独索引 | `2024_05_04_154409_rename_activity_relation_columns` |
| `loggable_type` (原 `entity_type`) | string nullable | 无单独索引 | 同上 |
| `user_id` | int | 无单独索引（由 Eloquent 查询负责） | — |
| `detail` (原 `extra`) | text | 无索引 | — |

**重要缺失**：`(loggable_type, loggable_id)` 组合列无索引。`ActivityQueries::entityActivity()` 和 `EntityWatchers::getRelevantWatches()` 中频繁使用 `WHERE loggable_type = ? AND loggable_id = ?` 查询，数据库端需走全表扫描。

#### `watches` 表 — 用户订阅

迁移 `2023_07_31_104430_create_watches_table`：

| 字段 | 类型 | 索引 |
|------|------|------|
| `id` | int | PK |
| `user_id` | int | INDEX |
| `level` | tinyint unsigned | INDEX |
| `(watchable_id, watchable_type)` | — | COMPOSITE INDEX `watchable_index` |

索引设计合理：
- `user_id` 索引用于查询"某用户的所有 Watch"（`/my-account/notifications` 页面列表）
- `(watchable_id, watchable_type)` 复合索引用于 `EntityWatchers::getRelevantWatches()` 查询某实体的所有订阅者
- `level` 索引可辅助按等级筛选

#### `mention_history` 表 — @提及通知历史

迁移 `2025_12_15_140219_create_mention_history_table`：

| 字段 | 类型 | 索引 |
|------|------|------|
| `id` | int | PK |
| `mentionable_type` | string(50) | INDEX |
| `mentionable_id` | unsignedBigInt | INDEX |
| `from_user_id` | unsignedInt | 无索引 |
| `to_user_id` | unsignedInt | 无索引 |
| `created_at` / `updated_at` | timestamp | 无索引 |

**注意**：`(mentionable_type, mentionable_id)` 是两个独立单列索引，不是复合索引。`CommentMentionNotificationHandler::getPreviouslyNotifiedUserIds()` 查询 `WHERE mentionable_id = ? AND mentionable_type = ?`，数据库可能只能命中一个索引再过滤。

#### `settings` 表 — 用户偏好（通知偏好存在这里）

迁移 `2015_08_30_125859_create_settings_table` + `2021_01_30_225441_add_settings_type_column`：

| 字段 | 类型 | 索引 |
|------|------|------|
| `setting_key` | string | PK（主键即索引） |
| `value` | text | 无索引 |
| `type` | string(50) | 无索引 |
| `created_at` / `updated_at` | nullable timestamp | 无索引 |

用户级偏好通过 `user:{userId}:{key}` 前缀存储，主键索引可直接命中。

---

## 专项深挖 5：移动端通知点击回跳的 Deep Link 路由与参数传递

### 结论先行：BookStack **没有** 原生移动端 App，也没有 Deep Link / Universal Link / App Link 机制

所谓"通知点击回跳"就是邮件里的普通 HTTP 链接直接打开浏览器，路由完全是标准 Web 路由。

### 5.1 通知邮件里的链接构建方式

所有 4 种活动通知邮件都通过 `->action()` 方法或 MessageParts 构建链接，**全部使用 `Entity::getUrl()` 生成标准 Web URL**。

#### Page 通知 — `Page::getUrl()`

`app/Entities/Models/Page.php:114`

```php
public function getUrl(string $path = ''): string
{
    $parts = [
        'books',
        urlencode($this->book_slug ?? $this->book->slug),
        $this->draft ? 'draft' : 'page',
        $this->draft ? $this->id : urlencode($this->slug),
        trim($path, '/'),
    ];
    return url('/' . implode('/', $parts));
}
```

示例输出：`https://bookstack.example.com/books/my-book/page/my-page-slug`

Page 还有 `getPermalink()` 方法生成 ID 链接：`https://bookstack.example.com/link/{id}`，但通知邮件中未使用。

#### Comment 通知 — 带锚点

```php
// CommentCreationNotification / CommentMentionNotification
->action(
    $locale->trans('notifications.action_view_comment'),
    $page->getUrl('#comment' . $comment->local_id)
)
```

生成 URL 示例：`https://bookstack.example.com/books/book-slug/page/page-slug#comment12`

`local_id` 是评论在页面内的局部 ID，锚点滚动到具体评论。

#### 其他通知邮件中的链接

| 链接组件 | 类 | 构建方式 |
|---------|----|---------|
| 实体名称超链接 | `EntityLinkMessageLine` | `$entity->getUrl()` |
| Book > Chapter 路径 | `EntityPathMessageLine` | 多个 `EntityLinkMessageLine` 用 ` > ` 拼接 |
| 底部"管理通知偏好"链接 | `BaseActivityNotification::buildReasonFooterLine()` | `url('/my-account/notifications')` |
| 邀请 / 重置密码链接 | `UserInviteNotification` / `ResetPasswordNotification` | `url('/register/invite/{token}')` / `url('password/reset/{token}')` |

### 5.2 Web 路由（标准 HTTP）

所有通知链接指向的路由定义在 `routes/web.php`，由 `web` middleware group 处理（Session / CSRF / Localization / CSP 等）：

| URL 模式 | 路由处理 |
|---------|---------|
| `/books/{bookSlug}/page/{pageSlug}` | Page 详情页（由实体路由注册） |
| `/books/{bookSlug}/chapter/{chapterSlug}` | Chapter 详情页 |
| `/books/{bookSlug}` | Book 详情页 |
| `/link/{id}` | 永久链接（ID 跳转） |
| `/my-account/notifications` | `UserAccountController::showNotifications()` |
| `/register/invite/{token}` | 邀请注册 |
| `/password/reset/{token}` | 重置密码 |

### 5.3 无任何 App 级 Deep Link 基础设施

搜索关键字全部无结果：

| 搜索项 | 结果 |
|--------|------|
| `deeplink` / `deep-link` / `deepLink` | ❌ |
| `app://` schema | ❌ |
| `intent://` | ❌ |
| `universal link` / `App Link` / `association` | ❌ |
| `apple-app-site-association` / `assetlinks.json` | ❌ |
| `notification_click` / `pendingIntent` / `intent-filter` | ❌ |

邮件模板也没有条件分支判断是否在 App 内打开（例如 `@if(app->runningInConsole())` 之类的逻辑）。如果后续要做移动端 App，需要：

1. 在邮件模板中增加 App Schema 或 Universal Link 分支
2. 服务端提供 `.well-known/apple-app-site-association` 和 `.well-known/assetlinks.json` 路由
3. 路由层增加 App Deep Link 参数解析与自动跳转

### 5.4 参数传递路径总结

```
通知构建（Message 类）
    │
    ├─ PageCreation / PageUpdate
    │     └─ $page->getUrl()
    │           └─ /books/{bookSlug}/page/{pageSlug}
    │
    ├─ CommentCreation / CommentMention
    │     └─ $page->getUrl('#comment' . $comment->local_id)
    │           └─ /books/{bookSlug}/page/{pageSlug}#comment{local_id}
    │
    ├─ EntityLinkMessageLine（邮件正文内联）
    │     └─ $entity->getUrl()
    │
    └─ 底部"管理通知偏好"链接
          └─ url('/my-account/notifications')
                └─ /my-account/notifications
    │
    ▼
Laravel MailMessage::action() / ->line() 渲染
    │
    ▼
邮件发送（mail channel）
    │
    ▼
用户点击链接
    │
    ▼
标准 Web 路由（routes/web.php）→ Controller → View
    ├─ 无额外跳转层
    ├─ 无 App Deep Link 检测
    └─ 无通知来源 / 来源追踪参数（如 utm_source / notification_id）
```

**注意**：通知链接中没有附带 `notification_id` / `activity_id` 等元信息参数，点击打开后服务端无法追踪哪封通知被点击了，也没有自动标记"已读"的逻辑（因为根本没有站内通知中心）。

---

## 专项深挖 6：用户偏好 — Mute / 频道选择 / 频率限制的落库与读取逻辑

### 结论先行

BookStack 的用户通知偏好系统 **极简**：

- **Mute（全局静音）**：❌ 不存在。唯一接近的是 `WatchLevels::IGNORE` 针对**单个实体**静音，无法全局关邮件
- **频道选择（邮件 / 站内 / Push）**：❌ 不存在。只有邮件一个通道，没得选
- **频率限制（即时 / 每日摘要 / 每周摘要 / 免打扰时段）**：❌ 不存在。所有通知即时发送

实际存在的只有 **4 项布尔开关**，全部落在 `settings` 表的 KV 存储里。

### 6.1 4 项通知偏好定义

`app/Settings/UserNotificationPreferences.php`

| 方法 | 对应设置键 | 含义 |
|------|-----------|------|
| `notifyOnOwnPageChanges()` | `notifications#own-page-changes` | 自己拥有的页面被他人修改时通知 |
| `notifyOnOwnPageComments()` | `notifications#own-page-comments` | 自己拥有的页面收到评论时通知 |
| `notifyOnCommentReplies()` | `notifications#comment-replies` | 自己的评论收到回复时通知 |
| `notifyOnCommentMentions()` | `notifications#comment-mentions` | 在评论中被 @提及时通知 |

### 6.2 存储方式：`settings` 表的前缀 KV

`SettingService::putUser()` (`app/Settings/SettingService.php:222`)：

```php
public function putUser(User $user, string $key, string $value): bool
{
    if ($user->isGuest()) {
        session()->put($key, $value);
        return true;
    }
    return $this->put($this->userKey($user->id, $key), $value);
}

protected function userKey(string $userId, string $key = ''): string
{
    return 'user:' . $userId . ':' . $key;
}
```

实际存入 `settings` 表的 `setting_key` 为：`user:123:notifications#own-page-changes`

存储值是字符串 `'true'` 或 `'false'`，读取时由 `SettingService::formatValue()` 自动转为 PHP 布尔值。

### 6.3 写入逻辑 — 白名单校验

`UserNotificationPreferences::updateFromSettingsArray()` (`app/Settings/UserNotificationPreferences.php:34`)

```php
public function updateFromSettingsArray(array $settings)
{
    $allowList = ['own-page-changes', 'own-page-comments', 'comment-replies', 'comment-mentions'];
    foreach ($settings as $setting => $status) {
        if (!in_array($setting, $allowList)) {
            continue;
        }
        $value = $status === 'true' ? 'true' : 'false';
        setting()->putUser($this->user, 'notifications#' . $setting, $value);
    }
}
```

- 白名单机制：只有 4 个允许的键能被修改，其他键被静默丢弃
- 非 `'true'` 的值一律存为 `'false'`（输入归一化）

### 6.4 读取逻辑 — 带默认值

`UserNotificationPreferences::getNotificationSetting()` (`app/Settings/UserNotificationPreferences.php:47`)

```php
protected function getNotificationSetting(string $key): bool
{
    return setting()->getUser($this->user, 'notifications#' . $key);
}
```

`SettingService::getUser()` 会自动 fallback 到 `config('setting-defaults.user.' . $key)`。

`app/Config/setting-defaults.php:44` 中只配置了一个默认值：

```php
'user' => [
    // ...
    'notifications#comment-mentions' => true,  // 唯一有默认值的
],
```

**其他 3 项偏好默认值**：`config('setting-defaults.user.notifications#own-page-changes')` 不存在，`SettingService::get()` 会取 `false` 作为兜底（`get($key, $default = null)` → `$default = null` → 取 `config('setting-defaults.' . $key, false)` → 最终 `false`）。

总结默认值：

| 偏好 | 默认值 |
|------|--------|
| `own-page-changes` | `false` |
| `own-page-comments` | `false` |
| `comment-replies` | `false` |
| `comment-mentions` | `true`（显式配置） |

### 6.5 读取缓存策略

`SettingService` 有一个请求级内存缓存 `$localCache`，按分类分组：

- 应用级设置 → 缓存键 `'app'`，用 `WHERE setting_key NOT LIKE 'user:%'` 批量加载
- 用户级设置 → 缓存键 `'user:{userId}'`，用 `WHERE setting_key LIKE 'user:{userId}:%'` 批量加载

首次读取用户的任一偏好时，会把该用户所有 `user:{userId}:%` 开头的设置全部查出来缓存。后续读取同用户其他偏好直接走数组键查找，无 DB 查询。

### 6.6 HTTP 入口 — Controller 与路由

`app/Users/Controllers/UserAccountController.php`：

| 方法 | 路由 | 职责 |
|------|------|------|
| `showNotifications()` | `GET /my-account/notifications` | 渲染偏好表单 + 用户 Watch 订阅列表 |
| `updateNotifications()` | `PUT /my-account/notifications` | 接收表单提交 → `updateFromSettingsArray()` 写入 |

前置权限：`$this->checkPermission(Permission::ReceiveNotifications)`，无此角色权限的用户无法访问或修改通知偏好。

### 6.7 UI 表单结构

`resources/views/users/account/notifications.blade.php` 用 4 个 `form.toggle-switch` 组件渲染，`name` 对应 `preferences[own-page-changes]` 等数组格式，`$preferences->notifyOnXxx()` 填充当前值。

表单下方还展示该用户的所有 Watch 订阅列表（分页 20 条/页），每条显示实体图标 + 订阅等级名称。

### 6.8 不存在的功能与替代设计

| 用户诉求 | 是否存在 | 替代 / 近似方案 |
|---------|---------|----------------|
| **全局 Mute（关闭所有邮件通知）** | ❌ | 可移除角色权限 `ReceiveNotifications`，`BaseNotificationHandler` 会检查 `$user->can(Permission::ReceiveNotifications)` 直接跳过发送 |
| **单实体 Mute** | ✅ | `WatchLevels::IGNORE (level=0)`，设置后 `EntityWatchers::isUserIgnoring()` 返回 true，被 Handler 用于排除（如页面 Owner 的通知） |
| **按频道选择（邮件 vs 站内 vs Push）** | ❌ | 只有邮件通道，无选择余地 |
| **发送频率（即时 / 摘要）** | ❌ | 全部即时发送，无 digest / batch 机制 |
| **免打扰时段（Do Not Disturb）** | ❌ | 无任何时间窗口判断 |
| **按活动类型细分（只关 PageUpdate 不关 Comment）** | ❌ | 4 项偏好是按场景聚合的，无法精细到 `PAGE_CREATE` vs `PAGE_UPDATE` |

全局 Mute 的替代路径值得注意：角色权限检查在 `BaseNotificationHandler` 中作为**发送层总开关**存在，如果管理员把用户的 `ReceiveNotifications` 权限拿掉，该用户不会收到任何活动通知邮件，这实际上就是"全局静音"。只是用户自己无法在 UI 上切换，需要管理员介入角色管理。

---

## 专项深挖补充：第 2 轮三层全景总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                  通知中心未读/已读 — ❌ 不存在                        │
│                                                                     │
│  · 无 notifications 表迁移（Laravel DatabaseNotification 未启用）     │
│  · User 模型无 HasDatabaseNotifications                              │
│  · Notification::via() 只返回 ['mail']                               │
│  · 无 markAsRead / unreadNotifications / read_at 代码               │
│  · Header 无通知铃铛 / 未读计数 UI                                   │
│                                                                     │
│  实际存在的 3 张核心表：                                              │
│  ┌────────────┬────────────────────────────────────────────┐       │
│  │ activities │ PK(id), INDEX(type), INDEX(created_at),    │       │
│  │            │ INDEX(ip), (loggable_*) 无复合索引          │       │
│  ├────────────┼────────────────────────────────────────────┤       │
│  │ watches    │ INDEX(user_id), INDEX(level),               │       │
│  │            │ COMPOSITE(watchable_id, watchable_type)     │       │
│  ├────────────┼────────────────────────────────────────────┤       │
│  │ mention_   │ INDEX(mentionable_type),                   │       │
│  │ history    │ INDEX(mentionable_id)  [两单列非复合]       │       │
│  └────────────┴────────────────────────────────────────────┘       │
└────────────────────────────────────┬────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│          Deep Link / 移动端通知点击回跳 — ❌ 无原生 App              │
│                                                                     │
│  · 无 app:// / intent:// 等 Schema                                   │
│  · 无 apple-app-site-association / assetlinks.json                  │
│  · 无 notification_id / utm_source 等追踪参数                         │
│                                                                     │
│  链接全部为标准 Web URL，由 Entity::getUrl() 构建：                    │
│  ┌──────────────────────┬───────────────────────────────────────┐   │
│  │ PageCreation/Update  │ /books/{slug}/page/{slug}            │   │
│  ├──────────────────────┼───────────────────────────────────────┤   │
│  │ CommentCreation/     │ /books/{slug}/page/{slug}#comment{id} │   │
│  │ CommentMention       │                                       │   │
│  ├──────────────────────┼───────────────────────────────────────┤   │
│  │ 底部"管理偏好"链接     │ /my-account/notifications            │   │
│  └──────────────────────┴───────────────────────────────────────┘   │
│                                                                     │
│  点击后：标准 Web 路由（routes/web.php）→ Controller → View          │
│  无自动标记已读（因为没有"未读"概念）                                   │
└────────────────────────────────────┬────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│      用户偏好 Mute / 频道 / 频率 — 极简（仅 4 个布尔开关）            │
│                                                                     │
│  存储：settings 表 KV                                                │
│    setting_key = "user:{uid}:notifications#{type}"                   │
│    value = "true" / "false" (string)                                 │
│  读取经 SettingService 请求级批量缓存                                  │
│                                                                     │
│  ┌────────────────────────┬───────────┬──────────────────────────┐  │
│  │ 偏好项                 │ 默认值    │ 控制的通知场景            │  │
│  ├────────────────────────┼───────────┼──────────────────────────┤  │
│  │ own-page-changes       │ false     │ 自己的页面被修改（UPD）  │  │
│  │ own-page-comments      │ false     │ 自己的页面被评论（CRE）  │  │
│  │ comment-replies        │ false     │ 自己的评论被回复         │  │
│  │ comment-mentions       │ true      │ 评论中被 @提及           │  │
│  └────────────────────────┴───────────┴──────────────────────────┘  │
│                                                                     │
│  ❌ 不存在的功能：                                                    │
│  · 全局 Mute → 替代：移除 ReceiveNotifications 角色权限              │
│  · 单实体 Mute → 有：WatchLevels::IGNORE (level=0)                  │
│  · 频道选择 → 只有 mail 单通道                                       │
│  · 频率限制（即时/摘要）→ 全部即时发送                                │
│  · 免打扰时段 → 无任何时间判断                                       │
└─────────────────────────────────────────────────────────────────────┘
```

