# BookStack API Token 校验、请求限流与权限判断协作流程

## 一、整体架构概览

API 请求进入系统后，依次经过以下三层校验：

```
请求 → [1. 速率限制] → [2. Token 认证] → [3. 权限判断] → 控制器处理
```

中间件注册顺序定义在 `app/Http/Kernel.php:40-46`：

```php
'api' => [
    \BookStack\Http\Middleware\ThrottleApiRequests::class,  // 1. 速率限制
    \BookStack\Http\Middleware\EncryptCookies::class,
    \BookStack\Http\Middleware\StartSessionIfCookieExists::class,
    \BookStack\Http\Middleware\ApiAuthenticate::class,       // 2. Token 认证
    \BookStack\Http\Middleware\CheckEmailConfirmed::class,   // 邮箱确认检查
],
```

---

## 二、速率限制层（ThrottleApiRequests）

### 2.1 核心实现

**文件**：`app/Http/Middleware/ThrottleApiRequests.php`

```php
class ThrottleApiRequests extends Middleware
{
    protected function resolveMaxAttempts($request, $maxAttempts): int
    {
        return (int) config('api.requests_per_minute');
    }
}
```

继承自 Laravel 的 `Illuminate\Routing\Middleware\ThrottleRequests`，重写了 `resolveMaxAttempts` 方法，从配置中读取每分钟请求数限制。

### 2.2 配置来源

**文件**：`app/Config/api.php:21`

```php
'requests_per_minute' => env('API_REQUESTS_PER_MIN', 180),
```

默认限制 180 次/分钟，可通过环境变量 `API_REQUESTS_PER_MIN` 覆盖。

### 2.3 限流粒度

在 `app/App/Providers/RouteServiceProvider.php:79-95` 中还定义了更细粒度的限流器：

```php
RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});
```

限流标识逻辑：**已认证用户按用户 ID，未认证用户按 IP 地址**。

> **注意**：虽然 `RouteServiceProvider` 定义了名为 `api` 的限流器（60次/分钟），但实际 API 路由组使用的是 `ThrottleApiRequests` 中间件，直接读取 `api.requests_per_minute` 配置（默认180次/分钟），优先级更高。

---

## 三、Token 认证层（ApiAuthenticate + ApiTokenGuard）

### 3.1 入口中间件：ApiAuthenticate

**文件**：`app/Http/Middleware/ApiAuthenticate.php`

支持两种认证方式，优先使用 Session 认证（便于在 UI 中探索 API），否则使用 Token 认证：

```
┌─────────────────────────────────────────────────────┐
│ ensureAuthorizedBySessionOrToken()                  │
│                                                     │
│  1. Session 已启动?                                  │
│     ├─ 是 → 检查用户是否有 AccessApi 权限             │
│     │       ├─ 无权限 → 抛出 403 ApiAuthException   │
│     │       └─ 有权限 → 仅允许 GET 请求               │
│     │                ├─ 非GET → 抛出 403 异常        │
│     │                └─ GET → 通过，继续后续流程     │
│     └─ 否 → 切换到 'api' guard，执行 Token 认证       │
│              auth()->shouldUse('api')                │
│              auth()->authenticate()                  │
└─────────────────────────────────────────────────────┘
```

关键代码片段（`app/Http/Middleware/ApiAuthenticate.php:31-64`）：

```php
protected function ensureAuthorizedBySessionOrToken(Request $request): void
{
    if (session()->isStarted()) {
        if (!$this->sessionUserHasApiAccess()) {
            throw new ApiAuthException(trans('errors.api_user_no_api_permission'), 403);
        }
        if ($request->method() !== 'GET') {
            throw new ApiAuthException(trans('errors.api_cookie_auth_only_get'), 403);
        }
        return;
    }

    auth()->shouldUse('api');
    auth()->authenticate();
}
```

### 3.2 Token 认证守卫：ApiTokenGuard

**文件**：`app/Api/ApiTokenGuard.php`

认证配置在 `app/Config/auth.php:50-52`：

```php
'api' => [
    'driver' => 'api-token',
],
```

完整的 Token 校验流程：

```
┌──────────────────────────────────────────────────────────────────┐
│ getAuthorisedUserFromRequest()                                   │
│                                                                  │
│  1. 读取 Authorization 请求头                                      │
│     $authToken = $request->headers->get('Authorization')         │
│                                                                  │
│  2. validateTokenHeaderValue() — 格式校验                          │
│     ├─ 空值 → "No authorization found"                           │
│     ├─ 格式错误（不含":"或不以"Token "开头）                        │
│     │     → "Bad authorization format"                           │
│     └─ 通过 → 解析 token_id:secret                               │
│                                                                  │
│  3. 数据库查询 ApiToken                                           │
│     ApiToken::where('token_id', $id)->with('user')->first()      │
│                                                                  │
│  4. validateToken() — Token 有效性校验                              │
│     ├─ Token 不存在 → "Token not found"                          │
│     ├─ Secret 哈希校验失败 → "Incorrect token secret"            │
│     ├─ Token 已过期 ($token->expires_at <= now)                  │
│     │     → "Token expired" (403)                                │
│     └─ 用户无 AccessApi 权限 → "No API permission" (403)         │
│                                                                  │
│  5. 检查邮箱确认状态                                               │
│     loginService->awaitingEmailConfirmation($user)               │
│     → 未确认邮箱 → "Email confirmation awaiting"                 │
│                                                                  │
│  6. 返回认证用户 $token->user                                      │
└──────────────────────────────────────────────────────────────────┘
```

### 3.3 Token 模型

**文件**：`app/Api/ApiToken.php`

```php
class ApiToken extends Model
{
    protected $fillable = ['name', 'expires_at'];
    protected $casts = ['expires_at' => 'date:Y-m-d'];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

Token 由两部分组成：`token_id:secret`，其中 `secret` 使用 Hash 存储（`app/Api/ApiTokenGuard.php:126`）：

```php
if (!Hash::check($secret, $token->secret)) {
    throw new ApiAuthException(trans('errors.api_incorrect_token_secret'));
}
```

默认过期时间为 100 年（`app/Api/ApiToken.php:43-46`）：

```php
public static function defaultExpiry(): string
{
    return Carbon::now()->addYears(100)->format('Y-m-d');
}
```

---

## 四、权限判断层

权限判断分为三个层级，层层递进：

### 4.1 系统级权限（在 Token 认证时已校验）

Token 认证阶段已校验用户是否有 `Permission::AccessApi` 权限（`app/Api/ApiTokenGuard.php:135-137`）：

```php
if (!$token->user->can(Permission::AccessApi)) {
    throw new ApiAuthException(trans('errors.api_user_no_api_permission'), 403);
}
```

### 4.2 路由/中间件级权限

**文件**：`app/Http/Middleware/CheckUserHasPermission.php`

通过 `can` 中间件别名注册（`app/Http/Kernel.php:56`）：

```php
'can' => \BookStack\Http\Middleware\CheckUserHasPermission::class,
```

使用方式（在路由或控制器构造函数中）：

```php
Route::get('/example', [Controller::class, 'action'])->middleware('can:access-api');
```

Permission 枚举提供了 `middleware()` 辅助方法（`app/Permissions/Permission.php:140-143`）：

```php
public function middleware(): string
{
    return 'can:' . $this->value;
}
```

### 4.3 控制器/实体级权限（最常用）

**核心方法**：`checkOwnablePermission` — 定义在 `app/Http/Controller.php:83-88`

```php
protected function checkOwnablePermission(string|Permission $permission, Model $ownable, string $redirectLocation = '/'): void
{
    if (!userCan($permission, $ownable)) {
        $this->showPermissionError($redirectLocation);
    }
}
```

**辅助函数**：`userCan` — 定义在 `app/App/helpers.php:43-53`

```php
function userCan(string|Permission $permission, ?Model $ownable = null): bool
{
    if (is_null($ownable)) {
        return user()->can($permission);
    }
    $permissions = app()->make(PermissionApplicator::class);
    return $permissions->checkOwnableUserAccess($ownable, $permission);
}
```

实体权限检查示例（`app/Entities/Controllers/PageApiController.php:72-87`）：

```php
public function create(Request $request)
{
    $this->validate($request, $this->rules['create']);

    if ($request->has('chapter_id')) {
        $parent = $this->entityQueries->chapters->findVisibleByIdOrFail(intval($request->input('chapter_id')));
    } else {
        $parent = $this->entityQueries->books->findVisibleByIdOrFail(intval($request->input('book_id')));
    }
    $this->checkOwnablePermission(Permission::PageCreate, $parent);

    $draft = $this->pageRepo->getNewDraftPage($parent);
    $this->pageRepo->publishDraft($draft, $request->only(array_keys($this->rules['create'])));

    return response()->json($draft->forJsonDisplay());
}
```

### 4.4 Permission 枚举

**文件**：`app/Permissions/Permission.php`

定义了系统中所有权限类型，主要分类：

| 分类 | 示例 |
|------|------|
| 系统权限 | `AccessApi`, `ContentExport`, `SettingsManage`, `UsersManage` |
| 实体权限 | `BookView/Create/Update/Delete`, `PageView/Create/Update/Delete` 等 |
| 非实体内容权限 | `AttachmentCreate/Update/Delete`, `ImageCreate/Update/Delete`, `CommentCreate/Update/Delete` |

权限支持 `All/Own` 后缀（如 `PageCreateAll`, `PageCreateOwn`），用于区分"创建所有"和"创建自己的"。

---

## 五、完整请求处理协作流程

### 5.1 时序图

```
客户端                   ThrottleApiRequests          ApiAuthenticate              ApiTokenGuard              Controller
   │                           │                            │                          │                         │
   │─── API 请求 ──────────────>│                            │                          │                         │
   │                           │ 1. 检查速率限制              │                          │                         │
   │                           │    (按 user_id 或 IP)       │                          │                         │
   │                           │                            │                          │                         │
   │                           │──── 超过限制 ───────────────│                          │                         │
   │<── 429 Too Many Requests ──┤                            │                          │                         │
   │                           │                            │                          │                         │
   │                           │──── 未超限 ────────────────>│                          │                         │
   │                           │                            │ 2. Session 认证?         │                         │
   │                           │                            │    ├─ 是 → 检查权限+方法  │                         │
   │                           │                            │    └─ 否 → Token 认证 ──>│                         │
   │                           │                            │                          │ 3. 解析 Token          │
   │                           │                            │                          │    ├─ 格式校验           │
   │                           │                            │                          │    ├─ 哈希校验           │
   │                           │                            │                          │    ├─ 过期校验           │
   │                           │                            │                          │    ├─ 邮箱确认检查       │
   │                           │                            │                          │    └─ AccessApi 权限    │
   │                           │                            │                          │                         │
   │                           │                            │<── 认证失败 ─────────────│                         │
   │<── 401/403 Unauthorized ───┤<───────────────────────────┤                          │                         │
   │                           │                            │                          │                         │
   │                           │                            │──── 认证通过 ────────────>│                         │
   │                           │                            │                          │                         │
   │                           │                            │                          │ 4. 控制器级权限检查     │
   │                           │                            │                          │    checkOwnablePermission│
   │                           │                            │                          │    (实体级权限)         │
   │                           │                            │                          │                         │
   │                           │                            │                          │<── 权限不足 ────────────│
   │<── 403 Forbidden ──────────┤<───────────────────────────┤<─────────────────────────┤                         │
   │                           │                            │                          │                         │
   │                           │                            │                          │──── 通过 ──────────────>│
   │                           │                            │                          │                         │ 5. 业务处理
   │<── 200 JSON Response ──────┤<───────────────────────────┤<─────────────────────────┤<────────────────────────│
```

### 5.2 协作关键点总结

1. **速率限制最先执行**：在任何认证逻辑之前，避免恶意请求消耗认证资源。

2. **认证方式双通道**：
   - Session 认证（Cookie）：仅允许 GET 请求，便于浏览器直接访问 API
   - Token 认证（Authorization Header）：支持所有 HTTP 方法

3. **权限分层校验**：
   - **第一层**（认证阶段）：检查 `AccessApi` 系统权限 — 确保用户能访问 API
   - **第二层**（路由阶段）：通过 `can` 中间件做粗粒度权限过滤
   - **第三层**（控制器阶段）：通过 `checkOwnablePermission` 做实体级细粒度权限校验

4. **限流标识联动**：速率限制中间件优先使用认证后的 `user()->id` 作为限流 key，未认证则回退到 IP 地址。这意味着：
   - 未认证请求先按 IP 限流（此时用户信息未知）
   - 认证通过后，如果后续有限流器使用 `user()->id`，则按用户 ID 限流

5. **异常处理**：
   - 速率限制：返回 `429 Too Many Requests`（Laravel 框架默认）
   - Token 认证失败：返回 `401 Unauthorized` 或带原因的 `403 Forbidden`
   - 权限不足：返回 `403 Forbidden`，JSON 请求返回 `{'error': '...'}`

---

## 六、关键文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| API Token Guard | `app/Api/ApiTokenGuard.php` |
| API Token 模型 | `app/Api/ApiToken.php` |
| API 认证中间件 | `app/Http/Middleware/ApiAuthenticate.php` |
| API 限流中间件 | `app/Http/Middleware/ThrottleApiRequests.php` |
| 权限检查中间件 | `app/Http/Middleware/CheckUserHasPermission.php` |
| 邮箱确认检查 | `app/Http/Middleware/CheckEmailConfirmed.php` |
| HTTP 中间件注册 | `app/Http/Kernel.php` |
| 控制器基类（权限方法） | `app/Http/Controller.php` |
| Permission 枚举 | `app/Permissions/Permission.php` |
| 权限应用器 | `app/Permissions/PermissionApplicator.php` |
| 路由服务提供者（限流配置） | `app/App/Providers/RouteServiceProvider.php` |
| 认证配置 | `app/Config/auth.php` |
| API 配置 | `app/Config/api.php` |
| 辅助函数（userCan） | `app/App/helpers.php` |
| API 路由 | `routes/api.php` |
